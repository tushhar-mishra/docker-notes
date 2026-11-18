# 17 · Multi-Stage Builds

> **Level:** ★★ intermediate
>
> **You will learn:** how to build your app in one stage (with compilers and build tools) and ship only the result in a small final stage. You'll see `FROM ... AS name`, `COPY --from`, `--target`, how BuildKit runs stages in parallel and skips unused ones, and complete examples for Go, Java, Node and Python.
>
> **You need:** topics 14–16.

---

## What it is

A **multi-stage build** is a Dockerfile with **several `FROM` lines**. Each `FROM` starts a new **stage**: a separate, temporary image. Later stages can **copy files out of earlier ones**. Only the **last stage** (or the one you pick with `--target`) becomes the final image.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25 AS build              # stage 1: big image with the compiler
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM gcr.io/distroless/static-debian12  # stage 2: tiny image, no compiler, no shell
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

The final image contains **only** the compiled `/app` binary on a tiny base. The Go compiler, the source code and the downloaded modules stay behind in stage 1, which is thrown away.

---

## Why it exists

Building an app needs a lot: compilers, SDKs, package managers, dev headers, test tools, source code. **Running** it needs much less. Shipping the build tools in production images means:
- **Size**: a Go app's image could be 900 MB (golang base) instead of 10 MB.
- **Security**: compilers, shells and package managers are tools an attacker can use, and every extra package is another set of possible vulnerabilities.
- **Leaks**: source code and build-time files end up in the image.

Before multi-stage builds (Docker 17.05), people kept two Dockerfiles and a shell script to copy files between them (the "builder pattern"). Multi-stage builds made this one file.

---

## Analogy

A **car factory and a car showroom**. The factory (build stage) has huge machines, spare parts and tools. The showroom (final stage) has only the finished car. You don't ship the factory to the customer.

---

## How it works inside

### Naming and copying

- `FROM image AS name` names a stage. Without a name, stages are numbered from 0 (`COPY --from=0`), but names are much clearer.
- `COPY --from=name /path/in/stage /path/in/this/stage` copies files from another stage's filesystem.
- `COPY --from=nginx:1.27-alpine /etc/nginx/nginx.conf /tmp/` also works: `--from` can be **any image**, not only a stage.
- A stage can start `FROM` an earlier stage: `FROM base AS test` reuses everything from `base`.

### What BuildKit does with stages

BuildKit builds a **dependency graph** of stages (topic 16):

```
          ┌──────── deps ────────┐
          ▼                      ▼
   base ─► build ─► final     test (only if --target test)
          ▲
      assets (independent, runs in parallel with deps)
```

- **Only stages the target needs are built.** A `test` stage that `final` doesn't use is **skipped** unless you ask for it with `--target test`.
- **Independent stages run in parallel**, e.g. building frontend assets while compiling the backend.
- Each stage has its own cache.

The legacy builder (deprecated) built **every** stage in order, even unused ones. That's a big reason to use BuildKit.

### `--target`: stop at a specific stage

```bash
docker build --target build -t myapp:builder .
docker build --target test .
```

Useful for: running tests in CI, debugging the build stage, and making a dev image with tools from the same Dockerfile.

### Static vs dynamic binaries (why some final images can be tiny)

- A **statically linked** binary contains all the libraries it needs. It can run on `scratch` or `distroless/static`, with no C library present. Go with `CGO_ENABLED=0` and Rust with the musl target produce these.
- A **dynamically linked** binary needs shared libraries at runtime (like `libc.so.6` from **glibc**, the GNU C Library). It must run on a base that has those exact libraries. If you build on Debian (glibc) and run on Alpine (which uses **musl**, a different C library), you get confusing errors like `no such file or directory` for a file that clearly exists (the missing file is the library loader). Topic 19 covers this.

Check with `ldd` or `file`:

```bash
file app
```

```
app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=…, stripped
```

- **ELF** (Executable and Linkable Format) = the standard file format of Linux programs and libraries. `statically linked` is the part that matters here.

---

## Commands / files

### Example 1: Go (static binary → distroless)

`main.go`:

```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		h, _ := os.Hostname()
		fmt.Fprintf(w, "hello from %s\n", h)
	})
	http.ListenAndServe(":8080", nil)
}
```

`go.mod`:

```
module example.com/hello

go 1.25
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/hello .

FROM gcr.io/distroless/static-debian12:nonroot AS final
COPY --from=build /out/hello /hello
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/hello"]
```

- `CGO_ENABLED=0` = no C code, so a fully static binary.
- `-trimpath` = remove local folder paths from the binary (reproducibility, no path leaks).
- `-ldflags="-s -w"` = strip debug symbols (smaller binary).
- `distroless/static-debian12:nonroot` = a base with only CA certificates, time zone data and `/etc/passwd`, and a `nonroot` user (UID 65532). No shell, no package manager (topic 19).

```bash
docker build -t hello-go:1.0 .
docker images hello-go golang --format 'table {{.Repository}}:{{.Tag}}\t{{.Size}}'
```

```
REPOSITORY:TAG     SIZE
hello-go:1.0       9.8MB
golang:1.25        874MB
```

**~10 MB** instead of ~880 MB.

### Example 2: Java (Maven → JRE only)

```dockerfile
# syntax=docker/dockerfile:1

FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -q dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -q -DskipTests package \
 && cp target/*.jar /app.jar

FROM eclipse-temurin:21-jre AS final
RUN useradd --system --uid 10001 appuser
COPY --from=build /app.jar /app/app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

- Build stage: JDK (Java Development Kit, the compiler) + Maven. Final: **JRE** (Java Runtime Environment) only.
- `-XX:MaxRAMPercentage=75` = the JVM sizes its heap from the container's **cgroup memory limit** (modern JVMs read it) instead of the host's RAM. Without this kind of setting, Java apps often get OOM-killed (topic 31).

### Example 3: Node (install with dev deps, ship production deps only)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM deps AS build
COPY . .
RUN npm run build                         # e.g. TypeScript → dist/

FROM node:22-slim AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:22-slim AS final
ENV NODE_ENV=production
WORKDIR /app
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

- `npm ci` = clean install exactly from the lock file (reproducible).
- `build` and `prod-deps` don't depend on each other, so BuildKit runs them **in parallel**.
- `--omit=dev` = skip development dependencies (test frameworks, TypeScript compiler).
- `USER node` = the official Node image includes a non-root user called `node`.

### Example 4: Python (build wheels / venv, copy the venv)

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.13-slim AS build
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

FROM python:3.13-slim AS final
COPY --from=build /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
WORKDIR /app
RUN useradd --system --uid 10001 app
COPY . .
USER 10001
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

- The build stage has `build-essential` (gcc and friends), needed to compile some packages. The final stage doesn't.
- A **venv** (virtual environment) is a self-contained folder of installed packages, which is easy to copy as one unit.
- Both stages use the **same base** (`python:3.13-slim`), so compiled packages match the final system libraries.
- `PYTHONUNBUFFERED=1` = print logs immediately (no output buffering), so `docker logs` is live.

### Example 5: a test stage and `--target`

```dockerfile
FROM build AS test
RUN --mount=type=cache,target=/root/.cache/pip pip install pytest
COPY tests ./tests
RUN pytest -q
```

```bash
docker build --target test .        # runs the tests; fails the build if tests fail
docker build -t myapp:1.0 .         # default target = last stage; the test stage is SKIPPED
```

(If `test` isn't the last stage, it won't run unless targeted. Put `final` last.)

### See what got built

```bash
docker build --progress=plain -t hello-go:1.0 . 2>&1 | grep -E '^#[0-9]+ \[' | sort -u
```

You'll see `[build 1/6]`, `[final 1/2]` style step names: the stage name is shown with each step.

---

## Hands-on lab

1. Build the Go example. Compare its size with the golang base image.
2. Prove there's nothing else in it:
   ```bash
   docker run --rm -d --name hg -p 8080:8080 hello-go:1.0
   curl -s localhost:8080
   docker exec hg sh          # fails: no shell in distroless
   docker export hg | tar -t | head -30
   docker rm -f hg
   ```
   `docker export` writes a container's whole filesystem as a tar. The list shows only `/hello`, a few `/etc` files, certificates and zone info.
3. **Single-stage comparison:** write a one-stage version (`FROM golang:1.25`, build, `CMD ["/out/hello"]`), build it, and compare sizes with `docker images`.
4. **Parallel stages:** in the Node example, add `RUN sleep 5` at the start of both `build` and `prod-deps`. Build with `--progress=plain` and watch both sleeps run at the same time (total ~5 s, not 10 s).
5. **glibc vs musl trap:** build the Go binary **with** `CGO_ENABLED=1` (and the Debian golang image), copy it into `alpine:3.22`, and run it. You get `exec /hello: no such file or directory`. Run `file` on the binary in the build stage: `dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2`. That loader doesn't exist on Alpine.

---

## Where you'll use it

- **Every compiled-language service** (Go, Rust, Java, .NET, C++): build stage + minimal runtime stage.
- **Frontend apps:** build with Node, serve the static files with nginx (`COPY --from=build /app/dist /usr/share/nginx/html`).
- **CI:** `--target test` and `--target lint` stages make the Dockerfile the single definition of how to build and test.
- **Security reviews:** "does the production image contain compilers, shells, source code?" Multi-stage is the main answer.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `COPY --from=build /out/app: not found` | The path doesn't exist in that stage (wrong `-o` path or `WORKDIR`) | Run `docker build --target build` and look inside: `docker run --rm -it <img> sh` |
| `exec /app: no such file or directory` (but the file exists) | A dynamically linked binary is missing its loader/libraries in the final image (e.g. glibc binary on Alpine or scratch) | Build static (`CGO_ENABLED=0`), or use a final base with the same libc as the build stage |
| `error while loading shared libraries: libxyz.so.1: cannot open shared object file` | Final image lacks a runtime library the app links to | Install the runtime package (not the `-dev` one) in the final stage |
| Test stage never runs | It's not needed by the default target, so BuildKit skips it | `docker build --target test` in CI |
| `x509: certificate signed by unknown authority` from an app in `scratch` | No CA certificates in the image, so HTTPS calls fail | Use `distroless/static` (has certs) or copy `/etc/ssl/certs/ca-certificates.crt` from the build stage |
| Time zone or locale wrong in tiny images | No `tzdata` | Copy zoneinfo or use a base that includes it (topic 45) |
| Final image still big | You copied the whole build folder (`COPY --from=build /src /app`) | Copy only the artifacts you need |

---

## Quick recap

- Several `FROM` lines = several stages. `COPY --from=<stage or image>` moves files between them. Only the target stage (default: the last) becomes the image.
- Build tools, source code and caches stay in throwaway stages. That means smaller, safer images.
- BuildKit builds only the stages the target needs, and runs independent stages in parallel.
- `--target <stage>` builds a specific stage (tests, debug images).
- Match the runtime base to how the binary was built: static binaries can use scratch/distroless; glibc binaries need a glibc base.

**Interview questions**

1. **Q: What is a multi-stage build, and why use it?**
   A: A Dockerfile with several FROM stages, where later stages copy only the needed artifacts from earlier ones. The final image excludes compilers, build dependencies and source, so it's much smaller, has fewer vulnerabilities, and leaks less.

2. **Q: How do you run tests as part of a Docker build without shipping test tools?**
   A: Add a `test` stage that builds on the build stage and runs the tests. Build it with `--target test` in CI. The production target doesn't depend on it, so it's skipped and nothing from it reaches the final image.

3. **Q: Why might a binary fail with "no such file or directory" in a scratch or Alpine image even though it's there?**
   A: It's dynamically linked against glibc, and the loader (e.g. `/lib64/ld-linux-x86-64.so.2`) or libraries it needs aren't in the image. Build it statically, or use a base with matching libraries.
