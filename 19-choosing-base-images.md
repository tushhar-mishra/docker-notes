# 19 · Choosing a Base Image

> **Level:** ★★ intermediate
>
> **You will learn:** the main families of base images (full Debian/Ubuntu, `-slim`, Alpine, distroless, `scratch`, Red Hat UBI, and hardened images such as Chainguard and Docker Hardened Images), what's actually inside each one, the glibc vs musl difference that causes real production bugs, and how to choose based on size, security, compatibility and debuggability.
>
> **You need:** topics 07 and 17 (layers, static vs dynamic binaries).

---

## What it is

The **base image** is what your `FROM` line points to: the starting filesystem your app is added to. It decides:
- which **C library** your programs run against (glibc or musl),
- which **package manager** you have (`apt`, `apk`, `dnf`/`microdnf`, or none),
- whether there's a **shell** and basic tools,
- how many **packages**, and so how many potential **vulnerabilities**, come with it,
- how **big** every image built on it is.

---

## Why it exists (why the choice matters)

- **Security:** scanners report CVEs (Common Vulnerabilities and Exposures, the public IDs for known security bugs) in **every package in the image**, used or not. A full Debian base can bring 100+ findings before you add a line of code. Fewer packages = fewer CVEs to patch and explain to auditors.
- **Size:** smaller images pull faster (faster deploys and autoscaling), cost less to store, and start faster on new nodes.
- **Compatibility:** some software only works (or works well) with glibc.
- **Debuggability:** an image with no shell is harder to debug in an emergency (topic 51 shows the tools for that).

---

## Analogy

Choosing a **flat to move into**:
- **Full image** = a fully furnished house. Everything's there, including a garage full of tools you'll never use (and have to maintain and insure).
- **Slim** = a furnished flat with only the essentials.
- **Alpine** = a tiny, well-designed studio. Very light, but the plugs are a different standard (musl), so some of your appliances need adapters.
- **Distroless** = an empty flat with electricity and water connected (runtime + certificates). No furniture, no toolbox. You bring only your app.
- **scratch** = an empty plot of land. You bring everything.

---

## How it works inside

### The families

| Family | Example tags | C library | Package manager | Shell | Typical size (compressed) | Good for |
|---|---|---|---|---|---|---|
| **Full** (Debian/Ubuntu-based) | `python:3.13`, `node:22`, `ubuntu:24.04` | glibc | apt | yes | 350–450 MB (language images) | Build stages; when you need many system tools |
| **Slim** | `python:3.13-slim`, `node:22-slim`, `debian:bookworm-slim` | glibc | apt | yes | 40–60 MB | **Default choice for most apps** |
| **Alpine** | `alpine:3.22`, `python:3.13-alpine`, `nginx:1.27-alpine` | **musl** | apk | yes (busybox `sh`) | 3–25 MB | Small images; tools; static binaries |
| **Distroless** (Google) | `gcr.io/distroless/static-debian12`, `/base-debian12`, `/cc-debian12`, `/python3-debian12`, `/java21-debian12`, `:nonroot`, `:debug` | glibc (except `static`) | **none** | **no** (except `:debug`) | 2–100 MB | Production runtime for compiled apps, Java, Python |
| **scratch** | `FROM scratch` | none | none | no | 0 MB | Fully static binaries |
| **Red Hat UBI** (Universal Base Image) | `registry.access.redhat.com/ubi9/ubi`, `ubi9/ubi-minimal`, `ubi9/ubi-micro` | glibc | dnf / microdnf / none | yes / yes / no | 30–80 MB | RHEL/OpenShift shops; freely redistributable; RHEL-supported packages |
| **Hardened, minimal-CVE** | Chainguard (`cgr.dev/chainguard/...`, based on the **Wolfi** distribution), Docker Hardened Images | glibc (Wolfi) or Debian/Alpine based | apk (dev variants) or none | dev variants only | small | Regulated environments; teams that want near-zero CVE reports |

(Sizes change with every release. Check current numbers with `docker images`.)

### What "distroless" really contains

`gcr.io/distroless/static-debian12`:
- `/etc/passwd` and `/etc/group` with `root`, `nonroot` (65532) and `nobody`,
- CA certificates (`/etc/ssl/certs/ca-certificates.crt`) so HTTPS works,
- time zone data (`/usr/share/zoneinfo`),
- `/tmp`,
- **nothing else**: no shell, no `ls`, no libc.

`base-debian12` adds glibc and OpenSSL. `cc-debian12` adds libgcc/libstdc++ (for C++ or Rust binaries). The language ones add the runtime (Python, Java).

`:nonroot` tags run as UID 65532 by default. `:debug` tags add a **busybox** shell for emergencies (don't run them in production by default).

### glibc vs musl: the difference that bites

Both are **C standard libraries**: the layer between programs and the kernel that provides basic functions (`printf`, `malloc` for memory, `getaddrinfo` for DNS lookups, threads, locale). Almost every program on Linux uses one.

| | glibc (GNU C Library) | musl |
|---|---|---|
| Used by | Debian, Ubuntu, RHEL, distroless (base) | Alpine |
| Size | Larger | Very small |
| Binary compatibility | Programs built on glibc **won't run** on musl (and the reverse), except static ones | |
| Python packages | Prebuilt "manylinux" wheels install instantly | Needs "musllinux" wheels. If none exist, pip **compiles from source** (slow builds, needs gcc in the image) |
| DNS resolution | Full-featured (`/etc/nsswitch.conf`, ordered nameserver retry) | Simpler: queries nameservers **in parallel**, historically no TCP fallback for large DNS answers (fixed in musl 1.2.4), different `search`/`ndots` behaviour |
| Memory allocator | Fast for multi-threaded apps | Slower `malloc` under heavy multi-threading (some apps swap in jemalloc/mimalloc) |
| Locales | Full | Minimal |

Real production symptoms of musl issues:
- `exec /app: no such file or directory` for a glibc binary (topic 17),
- `Error relocating ...: symbol not found`,
- Python builds suddenly taking 15 minutes (compiling numpy, cryptography),
- odd DNS behaviour in Kubernetes with many `search` domains,
- noticeably lower throughput on some multi-threaded services.

**Rule of thumb:** Alpine is great for small tools, static binaries and the official images built for it (nginx, redis). For language runtimes (Python, Node, Java, Ruby), prefer **`-slim`** (glibc) or **distroless**, unless you've tested Alpine well.

### Tag pinning for bases

| Tag style | Example | What you get |
|---|---|---|
| Floating major | `python:3` | Changes on every minor release. Too loose |
| Minor | `python:3.13-slim` | Patch updates and OS security fixes. **Common choice** |
| Exact | `python:3.13.9-slim-bookworm` | Only this patch version and Debian release |
| Digest | `python:3.13-slim@sha256:…` | Exactly these bytes. Combine with an automated updater (Renovate, Dependabot) so you still get fixes (topic 20) |

Name the OS release too (`-bookworm`, `-trixie`) so a new Debian major release doesn't arrive by surprise when the default changes.

### Debuggability trade-off

No shell = smaller attack surface, but `docker exec -it x sh` won't work. Modern answers (topic 51):
- `docker debug <container>` (attaches a toolbox container to it),
- a sidecar debug container sharing its PID/network namespace,
- `nsenter` from the host (topic 11),
- in Kubernetes, `kubectl debug` with ephemeral containers.

So "we need a shell for debugging" is no longer a strong reason to ship one.

---

## Commands / files

### Compare sizes yourself

```bash
for img in python:3.13 python:3.13-slim python:3.13-alpine gcr.io/distroless/python3-debian12; do
  docker pull -q $img > /dev/null
done
docker images --format 'table {{.Repository}}:{{.Tag}}\t{{.Size}}' | grep -E 'python'
```

```
REPOSITORY:TAG                                  SIZE
python:3.13                                     1.48GB
python:3.13-slim                                 211MB
python:3.13-alpine                               69.1MB
gcr.io/distroless/python3-debian12              77.6MB
```

(Disk size, unpacked. The compressed download is roughly a third.)

### Count packages (a proxy for attack surface)

```bash
docker run --rm python:3.13 sh -c 'dpkg -l | grep -c ^ii'
docker run --rm python:3.13-slim sh -c 'dpkg -l | grep -c ^ii'
docker run --rm python:3.13-alpine sh -c 'apk info | wc -l'
```

```
452
92
41
```

### Scan for vulnerabilities (preview of topic 43)

```bash
docker scout quickview python:3.13
docker scout quickview python:3.13-slim
```

```
  Target     │  python:3.13          │    0C     3H    12M   140L     4?
  Target     │  python:3.13-slim     │    0C     0H     2M    24L
```

- `C/H/M/L` = Critical / High / Medium / Low severity counts. Your numbers will differ: they change daily as fixes and new CVEs appear.

### Which libc does a binary need?

```bash
docker run --rm python:3.13-slim sh -c 'ldd /usr/local/bin/python3.13 | head -3'
docker run --rm python:3.13-alpine sh -c 'ldd /usr/local/bin/python3.13 | head -3'
```

```
        linux-vdso.so.1 (0x00007ffd…)
        libpython3.13.so.1.0 => /usr/local/bin/../lib/libpython3.13.so.1.0 (0x00007f…)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f…)          ← glibc
        /lib/ld-musl-x86_64.so.1 (0x7f…)
        libpython3.13.so.1.0 => /usr/local/lib/libpython3.13.so.1.0 (0x7f…)
        libc.musl-x86_64.so.1 => /lib/ld-musl-x86_64.so.1 (0x7f…)          ← musl
```

### OS release inside an image

```bash
docker run --rm python:3.13-slim cat /etc/os-release | grep PRETTY
docker run --rm --entrypoint "" registry.access.redhat.com/ubi9/ubi-minimal cat /etc/os-release | grep PRETTY
```

```
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
PRETTY_NAME="Red Hat Enterprise Linux 9.6 (Plow)"
```

(This is why you pin `-bookworm` / `-trixie` if your app depends on a specific Debian release: the default for `python:3.13-slim` moves to new Debian releases.)

### A distroless Python app

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim-bookworm AS build
RUN python -m venv /venv
COPY requirements.txt .
RUN /venv/bin/pip install --no-cache-dir -r requirements.txt

FROM gcr.io/distroless/python3-debian12:nonroot
COPY --from=build /venv /venv
COPY app.py /app/app.py
ENV PYTHONPATH=/venv/lib/python3.11/site-packages
WORKDIR /app
ENTRYPOINT ["python3", "app.py"]
```

- distroless/python3-debian12 ships **Debian 12's Python (3.11)**, so the build stage must use **the same Python minor version** and the same Debian release. Otherwise compiled packages won't load.

---

## Hands-on lab

1. Run the size comparison and the package counts above.
2. **musl vs glibc DNS and wheels:**
   ```bash
   time docker run --rm python:3.13-slim pip install --quiet pandas
   time docker run --rm python:3.13-alpine pip install --quiet pandas
   ```
   If musllinux wheels exist for your versions, both are quick. If not, Alpine compiles from source (and may fail without gcc). Note the difference.
3. **No shell in distroless:**
   ```bash
   docker run --rm gcr.io/distroless/static-debian12 sh
   docker run --rm --entrypoint sh gcr.io/distroless/static-debian12:debug -c 'ls /; cat /etc/passwd'
   ```
   First: `exec: "sh": executable file not found`. Second: the `:debug` variant has busybox.
4. **A static binary on scratch:** take `hello-go` from topic 17, change the final stage to `FROM scratch`, and add `COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`. Build, run, compare sizes with distroless.
5. Write down, for an app you know, which base you'd pick and why (one sentence each for security, size, compatibility, debuggability).

---

## Where you'll use it

- **Platform/security standards:** many companies mandate an approved list of base images (often slim, distroless, UBI or hardened images) and scan for anything else.
- **CVE reduction projects:** moving from full → slim/distroless often removes most findings with no code change.
- **Performance incidents:** a service slower on Alpine than on Debian (allocator, DNS) is a classic case.
- **OpenShift/RHEL environments:** UBI images are the expected base and are supported.
- **Interviews:** "How do you make an image smaller and more secure?" → multi-stage + minimal base + non-root.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `exec /app: no such file or directory` on Alpine/scratch | glibc-linked binary, no glibc loader | Static build, or a glibc base (slim/distroless base) |
| `Error relocating /usr/lib/...: __xyz: symbol not found` | glibc-specific code on musl | Use a glibc base |
| pip build takes ages / `error: command 'gcc' failed` on Alpine | No musllinux wheel, so pip compiles | Use `-slim`, or install build deps in a build stage |
| `x509: certificate signed by unknown authority` | No CA certificates (scratch, some minimal bases) | Use distroless/static or copy CA certs |
| `unknown time zone Asia/Kolkata` | No tzdata | Install `tzdata`, or use a base with zoneinfo |
| `exec: "sh": executable file not found` during debugging | Distroless/scratch has no shell | `docker debug`, sidecar container, `nsenter` (topics 11, 51) |
| Image suddenly changed OS release and broke | Tag without a release name moved to a new Debian/Alpine release | Pin `-bookworm` / `alpine3.22` style tags or digests |
| Scanner reports hundreds of CVEs | Full base image with many unused packages | Switch to slim/distroless/hardened; multi-stage |

---

## Quick recap

- The base image decides libc, package manager, shell, size and CVE count.
- **slim** (glibc) is a safe default. **distroless** for production runtimes. **scratch** for static binaries. **Alpine** for tools/static binaries, but test carefully: musl differs from glibc. **UBI** for Red Hat shops. **Hardened** images for near-zero CVEs.
- Binaries built against glibc don't run on musl. Match the runtime base to the build.
- Pin minor version + OS release (or a digest with automated updates).
- No shell is fine. Debug with `docker debug`, sidecars or `nsenter`.

**Interview questions**

1. **Q: Alpine or Debian-slim for a Python service?**
   A: Usually slim. Alpine uses musl, so many Python packages need compiling from source, and there are behaviour differences (DNS, allocator performance). Slim is still small and fully glibc-compatible. Distroless is even leaner for production.

2. **Q: What is a distroless image?**
   A: An image with only the app's runtime dependencies (for example CA certs, tzdata, and maybe libc or a language runtime), with no shell or package manager. It means a smaller attack surface and fewer CVEs.

3. **Q: Why can't you run a binary built on Ubuntu inside an Alpine container?**
   A: If it's dynamically linked, it needs glibc's loader and libraries. Alpine uses musl, so the loader path doesn't exist and the libraries aren't ABI-compatible (ABI = Application Binary Interface: the low-level rules for how compiled code calls library functions). Build it statically, or use a glibc-based image.
