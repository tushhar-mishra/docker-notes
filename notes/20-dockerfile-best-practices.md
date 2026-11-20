# 20 · Dockerfile Best Practices

> **Level:** ★★ intermediate
>
> **You will learn:** how to write production-grade Dockerfiles: run as a non-root user, pin versions and digests (and keep them updated), use `COPY --chown`/`--chmod`/`--link`, keep layers small and clean, aim for reproducible builds, and lint automatically with `docker build --check` and hadolint. Every rule comes with the reason behind it.
>
> **You need:** topics 14–19.

---

## What it is

A set of rules that make images **smaller, safer, faster to build and predictable**. Most are about one of four things:

| Goal | Main practices |
|---|---|
| **Security** | Non-root `USER`, minimal base, no secrets in layers, pinned and scanned dependencies |
| **Size** | Multi-stage, slim/distroless base, clean up in the same `RUN`, `--no-install-recommends`, `.dockerignore` |
| **Speed** | Cache-friendly order, cache mounts, small context, `COPY --link` |
| **Predictability** | Pinned versions, lock files, exec-form CMD, labels, linting |

---

## Why it exists

Most container security findings and slow pipelines come from a short list of Dockerfile mistakes: running as root, `latest` tags, unpinned packages, leftover caches, build tools in the final image, secrets in layers. Each one on its own is small. Together they mean slow, fragile, vulnerable images.

---

## Analogy

A **building code** for construction. You could build a house without it, and it might stand. The code exists because people learned the hard way where fires start and walls fall down. Each rule has a story behind it.

---

## How it works inside: the practices

### 1. Run as a non-root user

By default the container's process runs as **root (UID 0)**, the same UID 0 as on the host (topic 11). If an attacker breaks into the app, they're root inside the container. Combined with a mistake (a mounted Docker socket, `--privileged`, a writable host path, or a kernel bug), that becomes root on the host.

```dockerfile
# Debian/Ubuntu base
RUN groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app
USER 10001:10001

# Alpine base
RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app -H -s /sbin/nologin app
USER 10001:10001
```

- Use a **fixed numeric UID/GID** (like 10001). Numbers are what the kernel checks, and what volume permissions and Kubernetes `runAsNonRoot`/`runAsUser` settings work with. `runAsNonRoot` can only verify a **numeric** `USER`.
- Use a high UID (≥ 10000) so it's unlikely to clash with real users on the host.
- Put `USER` **after** steps that need root (installing packages), and **before** `CMD`.
- Many official images already include a user: `node` (Node), `nginx` (nginx, for workers), `nonroot` (distroless `:nonroot`).
- Ports below 1024 used to need root. Docker (since 20.10) sets the sysctl `net.ipv4.ip_unprivileged_port_start=0` inside each container's own network namespace, so a non-root user **can** bind port 80 in a normal Docker container. But that's not true with `--network host`, on hosts/runtimes that don't set it, or in some Kubernetes setups. The portable habit is still: listen on 8080 and map the port outside (topic 28).

### 2. Give files the right owner without extra layers

```dockerfile
COPY --chown=10001:10001 --chmod=0755 entrypoint.sh /app/entrypoint.sh
COPY --chown=10001:10001 . /app
```

- `--chown` and `--chmod` set ownership and permissions **while copying**.
- The wrong way: `COPY . /app` then `RUN chown -R app /app`. This **duplicates every file** into a new layer (copy-up, topic 07): your image grows by the size of the app.
- Keep code **read-only to the app user** where possible (owned by root, readable by all), and give the app user write access only to the folders it must write (`/app/tmp`, a volume).

### 3. Pin versions and digests, and automate updates

| What | Unpinned (bad) | Pinned |
|---|---|---|
| Base image | `FROM python` / `:latest` | `FROM python:3.13-slim-bookworm@sha256:…` |
| OS packages | `apt-get install curl` | `apt-get install curl=8.14.1-2` (where practical) |
| Language deps | `pip install flask` | `requirements.txt` with `==` versions + a lock file (`package-lock.json`, `poetry.lock`, `uv.lock`, `go.sum`) |
| Downloaded tools | `curl …/latest/tool` | Specific version + `sha256sum -c` check (or `ADD --checksum=sha256:…`) |

Why: unpinned builds produce **different images from the same Dockerfile** on different days. A broken upstream release breaks your build or, worse, your production.

The catch: pinned images **don't get security fixes on their own**. So pin, and use an automated updater, such as **Renovate** or **Dependabot**, that opens pull requests when a new digest/version is out. Your CI then builds, tests and scans the update before you merge it.

### 4. Clean up in the same layer

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*
```

- `--no-install-recommends` = install only required dependencies, not "recommended" extras (often saves 30–50%).
- `rm -rf /var/lib/apt/lists/*` = delete the downloaded package lists (tens of MB), **in the same `RUN`**, so they never reach a layer.
- Alpine: `apk add --no-cache curl` (no cache is written at all).
- pip: `--no-cache-dir`, or a cache mount (topic 16).
- Official Debian/Ubuntu images already auto-clean `/var/cache/apt/archives` after install.

### 5. Fewer, meaningful layers (but don't over-optimise)

- Each `RUN`/`COPY` is a layer. Combine **related** commands (install + cleanup) into one `RUN`.
- Don't squash unrelated things together just to reduce the count. That hurts cache reuse. Cache-friendliness matters more than the exact number of layers.

### 6. `COPY --link`: independent layers

```dockerfile
COPY --link --from=build /out/app /app
```

- Normally a `COPY` layer depends on the layers below it. With `--link`, BuildKit creates the copied files as an **independent layer** that doesn't depend on what's underneath.
- Benefit: when the **base image** changes (e.g. a security update), BuildKit can **reuse** this layer and just stack it on the new base ("rebase") instead of redoing the copy. Pushes can also skip layers the registry already has.
- Limitation: the copy can't rely on files below it (e.g. it can't follow a symlink in the destination path that exists only in the base). It's safe for copying build artifacts into a final stage.

### 7. One main process per container

- One container = one service (one PID 1, topic 12). Don't run nginx + app + cron + sshd together with a supervisor.
- Why: separate scaling, separate restarts, separate logs, clear health, clean signal handling.
- Never run `sshd` in a container. Use `docker exec` / `kubectl exec`.

### 8. Exec form, `exec "$@"`, and a sensible stop signal

From topics 12 and 15: `CMD ["app"]`, entrypoint scripts end with `exec "$@"`, and `STOPSIGNAL` if your app needs a different graceful signal.

### 9. Declare how to check health

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD ["/app/healthcheck"]
```

- Lets Docker (and Compose) know whether the app actually **works**, not just whether the process exists. Full details in topic 32. (Kubernetes ignores `HEALTHCHECK` and uses its own probes.)
- Use a command that exists in the image. A tiny static healthcheck binary works even in distroless.

### 10. Reproducible builds

Same source + same Dockerfile → ideally the **same image digest**. Helps with verification, caching and supply-chain security (topic 43).
- Pin everything (practice 3).
- Use lock files and `npm ci` / `pip install --require-hashes` / `go mod download`.
- Avoid writing timestamps or random values into files. BuildKit supports **`SOURCE_DATE_EPOCH`** (a standard variable holding a fixed timestamp). On its own it sets the image **config and history timestamps** (the "created" dates). To also rewrite the **mtimes of files inside the layers**, add the exporter option `rewrite-timestamp=true` (BuildKit ≥ 0.13):
  ```bash
  SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) \
    docker buildx build --build-arg SOURCE_DATE_EPOCH \
    --output type=image,name=myapp:1.0,rewrite-timestamp=true .
  ```
  Check reproducibility by building twice and comparing digests (or with `diffoci`).

### 11. Labels, `.dockerignore`, and small contexts

- OCI labels for source, revision and version (topic 18).
- `.dockerignore` for `.git`, `node_modules`, `.env`, build output (topic 14).

### 12. Deprecated instructions to avoid

| Deprecated | Use instead |
|---|---|
| `MAINTAINER name` | `LABEL org.opencontainers.image.authors="name"` |
| Legacy builder (`DOCKER_BUILDKIT=0`) | BuildKit (default) |
| `ADD` for local files that aren't tarballs | `COPY` |

### 13. Lint automatically

**`docker build --check`** runs BuildKit's built-in rules without building. Examples of rules:

| Rule | Catches |
|---|---|
| `JSONArgsRecommended` | Shell-form `CMD`/`ENTRYPOINT` (signal problems) |
| `SecretsUsedInArgOrEnv` | `ARG PASSWORD`, `ENV API_KEY` |
| `UndefinedVar` | Using `$VAR` that was never declared |
| `FromAsCasing` | `FROM x as y` (mixed case) |
| `MaintainerDeprecated` | `MAINTAINER` |
| `WorkdirRelativePath` | `WORKDIR app` without a leading `/` |
| `CopyIgnoredFile` | `COPY` of a file excluded by `.dockerignore` |

Make warnings fail the build with a directive at the top of the Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true
```

**hadolint** is a popular extra linter (it also checks shell commands inside `RUN` with ShellCheck): unpinned apt packages, missing `--no-install-recommends`, `cd` instead of `WORKDIR`, and more.

---

## Commands / files

### Before: a typical first Dockerfile

```dockerfile
FROM python:latest
MAINTAINER dev@acme.com
COPY . /app
RUN cd /app && pip install -r requirements.txt
RUN apt-get update
RUN apt-get install -y curl
ENV DB_PASSWORD=supersecret
EXPOSE 80
CMD python /app/app.py
```

Problems: `latest`; deprecated `MAINTAINER`; copy before install (no caching); `cd` in `RUN`; stale `apt-get update`; no cleanup; recommended packages; secret in `ENV`; root user; port 80 (needs root); shell-form CMD; full base image.

### After

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true

ARG PYTHON_IMAGE=python:3.13-slim-bookworm@sha256:<digest-pinned-by-renovate>

FROM ${PYTHON_IMAGE} AS build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --require-hashes -r requirements.txt

FROM ${PYTHON_IMAGE} AS final
LABEL org.opencontainers.image.authors="platform@acme.com" \
      org.opencontainers.image.source="https://github.com/acme/hello"
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/* \
 && groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --no-create-home app
ENV PATH="/opt/venv/bin:$PATH" PYTHONUNBUFFERED=1 PORT=8000
COPY --link --from=build /opt/venv /opt/venv
WORKDIR /app
COPY --chown=root:root --chmod=0644 app.py .
USER 10001:10001
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD ["curl", "-fsS", "http://127.0.0.1:8000/health"]
CMD ["python", "app.py"]
```

(`DB_PASSWORD` is gone: it's provided at runtime from a secret store; topic 44. `--require-hashes` needs a requirements file with hashes, e.g. made by `pip-compile --generate-hashes`.)

### Run `--check`

```bash
docker build --check -f Dockerfile.before .
```

```
Check complete, 4 warnings have been found!

WARNING: MaintainerDeprecated - https://docs.docker.com/go/dockerfile/rule/maintainer-deprecated/
Maintainer instruction is deprecated in favor of using label
Dockerfile.before:2
--------------------
   1 |     FROM python:latest
   2 | >>> MAINTAINER dev@acme.com
--------------------

WARNING: SecretsUsedInArgOrEnv - ...
Do not use ARG or ENV instructions for sensitive data (ENV "DB_PASSWORD")
Dockerfile.before:7

WARNING: JSONArgsRecommended - ...
JSON arguments recommended for CMD to prevent unintended behavior related to OS signals
Dockerfile.before:9
...
```

### Run hadolint

```bash
docker run --rm -i hadolint/hadolint < Dockerfile.before
```

```
-:1 DL3007 warning: Using latest is prone to errors if the image will ever update. Pin the version explicitly to a release tag
-:2 DL4000 error: MAINTAINER is deprecated
-:4 DL3003 warning: Use WORKDIR to switch to a directory
-:4 DL3042 warning: Avoid use of cache directory with pip. Use `pip install --no-cache-dir <package>`
-:5 DL3009 info: Delete the apt-get lists after installing something
-:6 DL3008 warning: Pin versions in apt get install. Instead of `apt-get install <package>` use `apt-get install <package>=<version>`
-:6 DL3015 info: Avoid additional packages by specifying `--no-install-recommends`
-:9 DL3025 warning: Use arguments JSON notation for CMD and ENTRYPOINT arguments
```

- `DLxxxx` = hadolint's rule IDs. You can ignore specific rules in a `.hadolint.yaml` file, or with a `# hadolint ignore=DL3008` comment above a line.

### Find wasted space layer by layer: `dive`

`docker history` (topic 07) shows each layer's size. **dive** shows **what's inside** each layer, and how much space is wasted by files that later layers overwrite or delete (the "`RUN rm` doesn't shrink the image" problem):

```bash
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:<version> myapp:1.0
```

- Left pane: layers. Right pane: the filesystem of the selected layer, with added/changed/removed files highlighted. Tab switches panes.
- The bottom shows **image efficiency** (e.g. `95 %`) and **wasted bytes**, with the files responsible.
- In CI, it can fail a build that wastes too much:
  ```bash
  docker run --rm -e CI=true -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:<version> myapp:1.0 \
    --lowestEfficiency=0.95 --highestUserWastedPercent=0.05
  ```
- Pin the version (look it up on the project's releases page), as with every tool that gets the Docker socket (topics 40, 43).

### Check an image's user

```bash
docker image inspect myapp:1.0 -f 'User={{.Config.User}}'
docker run --rm myapp:1.0 id
```

```
User=10001:10001
uid=10001 gid=10001 groups=10001
```

- An empty `User=` means root.

---

## Hands-on lab

1. Save the "before" Dockerfile and a minimal `app.py` + `requirements.txt`. Run `docker build --check` and hadolint. List every finding.
2. Rewrite it step by step into the "after" version. After each change, rebuild and record: image size (`docker images`), build time on a code-only change, and `docker run --rm <img> id`.
3. **The chown trap:**
   ```dockerfile
   # syntax=docker/dockerfile:1
   FROM alpine:3.22
   RUN head -c 50000000 /dev/urandom > /data.bin
   RUN adduser -D -u 10001 app && chown app /data.bin
   ```
   Check `docker history`: the `chown` layer is also ~50 MB (copy-up). Rewrite so the file is created with the right owner in one step (create and chown in the same `RUN`, or `COPY --chown`).
4. Add `# check=error=true` to a Dockerfile with a shell-form `CMD` and confirm the build fails.
5. Clean up the images you built.

---

## Where you'll use it

- **Code review checklists** for every Dockerfile change.
- **CI gates:** `docker build --check` / hadolint as a required step (topic 46).
- **Security compliance:** non-root and minimal-base rules are in the CIS Docker Benchmark (topic 40) and Kubernetes Pod Security Standards (`runAsNonRoot`).
- **Dependency hygiene:** Renovate/Dependabot PRs for base image digests.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `PermissionError: [Errno 13] Permission denied: '/app/cache'` after adding `USER` | The non-root user can't write where the app writes | Create those folders with the right owner, or write to `/tmp` / a volume |
| `bind: permission denied` on port 80 | Non-root process with no unprivileged-port sysctl (host network, other runtimes, some K8s setups) | Listen on 8080+ and map it with `-p 80:8080` (or grant `CAP_NET_BIND_SERVICE` via file capabilities; topic 41) |
| Image doubled in size after adding `chown -R` | Copy-up of every file into a new layer | `COPY --chown` instead |
| Kubernetes: `container has runAsNonRoot and image has non-numeric user (app), cannot verify user is non-root` | `USER app` (name) instead of a number | `USER 10001` |
| Build fails after pinning an apt package version | That exact version is gone from the mirror (Debian replaces versions with security updates) | Pin less tightly for OS packages, or rely on base digest pinning + scanning |
| Pinned digest never updates, and the scanner shows old CVEs | Pinning without an updater | Set up Renovate/Dependabot |
| `--check` fails the build after adding `check=error=true` | Lint findings are now errors | Fix them, or skip a rule with `# check=skip=RuleName` |

---

## Quick recap

- **Non-root** with a fixed numeric UID. Use `COPY --chown/--chmod`, not `RUN chown -R`.
- **Pin** base images (tag + digest), dependencies (lock files) and downloads (checksums), then **automate updates**.
- Install with `--no-install-recommends` and clean up **in the same `RUN`**. Multi-stage + a minimal base.
- One process per container, exec-form CMD, `HEALTHCHECK` where Docker/Compose run it, OCI labels, `.dockerignore`.
- Lint with `docker build --check` (`# check=error=true`) and hadolint in CI.

**Interview questions**

1. **Q: Why should containers not run as root, and how do you do it?**
   A: If the app is compromised, root inside the container is UID 0 on the host kernel. Any misconfiguration or kernel bug can then escalate to host root. Create a user with a fixed numeric UID in the Dockerfile and set `USER 10001`. Set file ownership at COPY time, and prefer ports above 1024 for portability.

2. **Q: How do you keep pinned base images secure?**
   A: Pin by tag and digest for reproducibility, and use an automated dependency updater (Renovate or Dependabot) to propose new digests. CI rebuilds, tests and scans each update before merge.

3. **Q: What does `COPY --link` do?**
   A: It creates the copied files as an independent layer that doesn't depend on the layers beneath. When the base image changes, BuildKit can reuse the layer and rebase it instead of re-copying, which also lets registries skip existing layers on push.
