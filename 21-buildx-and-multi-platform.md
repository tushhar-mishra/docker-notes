# 21 · buildx and Multi-Platform Images

> **Level:** ★★★ advanced
>
> **You will learn:** what `docker buildx` is, what builders and drivers are, how to build one image for both amd64 (Intel/AMD) and arm64 (AWS Graviton, Apple Silicon, Raspberry Pi) servers, how QEMU emulation and cross-compilation work underneath (`binfmt_misc`), how the result is stored as an image index, build outputs (`--load`, `--push`, `--output`), and how `docker buildx bake` defines many builds in one file.
>
> **You need:** topic 06 (image index / manifest list), topics 16–18 (BuildKit, stages, `TARGETARCH`).

---

## What it is

- **`docker buildx`** is the CLI plugin for BuildKit's full feature set. `docker build` is actually an alias for `docker buildx build` with the default builder.
- A **builder** is a BuildKit instance that runs builds. A **driver** decides **where** it runs.
- A **multi-platform image** is one image name (tag) that points to an **image index** listing one image per CPU architecture. Each machine pulls the variant it can run (topic 06).

```
myapp:1.0 → index
             ├── linux/amd64  → manifest → layers built for x86-64
             └── linux/arm64  → manifest → layers built for ARM 64-bit
```

---

## Why it exists

- **ARM servers are mainstream:** AWS Graviton, Google Axion, Azure Cobalt and Ampere instances are often 20–40% cheaper for the same performance.
- **Developer laptops are ARM:** Apple Silicon Macs, Windows on ARM.
- **Edge devices:** Raspberry Pi and IoT boards.

A container shares the host kernel **and the host CPU**. An image built for amd64 contains amd64 machine code. Running it on arm64 fails, unless emulated (slowly). So you need images built **for each CPU architecture**, ideally under one name.

---

## Analogy

Publishing a book in **several languages under one title**. The bookshop (registry) has one entry, "myapp 1.0". When a reader asks for it, the shop hands over the edition in **their** language (CPU architecture). The translation can be done by a translator who reads one language and writes another as they go (**emulation**, QEMU), or by an author who writes each edition directly (**cross-compilation**, faster).

---

## How it works inside

### Builders and drivers

| Driver | Where BuildKit runs | Multi-platform | Notes |
|---|---|---|---|
| `docker` (default) | Inside dockerd | **Yes with the containerd image store** (Engine 29 default). With the old overlay2 store: build-only, can't load multi-platform results | Simplest. Shares the image store with `docker run` |
| `docker-container` | A BuildKit container (`moby/buildkit`) started by buildx | Yes | Full features (all cache exports), isolated. Results must be `--load`ed or `--push`ed |
| `kubernetes` | BuildKit pods in a cluster | Yes | Scale-out builds; native nodes per arch |
| `remote` | A BuildKit daemon you run elsewhere | Yes | Connect over TCP/TLS |
| Docker Build Cloud | Docker's hosted builders | Yes, native amd64 + arm64 | Paid feature; shared cache for teams |

### Two ways to build for another CPU

**1. Emulation with QEMU (easy, slow)**

- **QEMU** = a program that can run machine code for one CPU on another CPU, by translating instructions.
- **`binfmt_misc`** = a Linux kernel feature that says "when someone tries to run a file with *this* format (e.g. an ARM64 ELF binary), run it through *this* interpreter (e.g. `qemu-aarch64`)". It's registered in `/proc/sys/fs/binfmt_misc/`.
- With both set up, `RUN apt-get install …` in an arm64 stage on an amd64 host just works: every ARM program the build runs is transparently passed through QEMU.
- Cost: often **5–20× slower** for CPU-heavy steps (compiling).

**2. Cross-compilation (fast, needs a bit of Dockerfile work)**

Run the **compiler natively** on the build machine's architecture, and tell it to produce machine code for the target:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25 AS build      # runs natively (amd64)
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot             # this stage is per-target platform
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

- `FROM --platform=$BUILDPLATFORM` = "this stage runs on the **build** machine's architecture", so no emulation.
- `TARGETOS`/`TARGETARCH` (topic 18) tell Go what to produce.
- The final stage has no `RUN`, so it needs no emulation either: BuildKit just pulls the arm64 distroless image and copies the file in.
- Go, Rust, Zig, .NET and Java (JARs are platform-independent) cross-build easily. C/C++ and Python native extensions are harder: use emulation or native builders for those.

**3. Native builders (fastest, needs machines)**

Add an arm64 machine as a second node to a builder (`docker buildx create --append`), or use a cloud builder. Each platform is built on real hardware.

### What a multi-platform build produces

1. BuildKit builds the graph **once per platform** (in parallel).
2. Each platform's result = config + layers + a manifest.
3. An **image index** is created listing those manifests with their `platform` fields.
4. With `--push`, all of it goes to the registry under one tag. With the containerd image store, `--load` keeps the whole index locally too.

### Outputs

| Flag | Result |
|---|---|
| (default with `docker` driver) | Image in the local image store |
| `--load` | Load the result into the local image store (needed with `docker-container` driver) |
| `--push` | Push to the registry (= `--output type=registry`) |
| `--output type=local,dest=./out` | Write the **final filesystem** to a folder (great for extracting build artifacts, no image) |
| `--output type=tar,dest=out.tar` | The filesystem as a tar |
| `--output type=oci,dest=img.tar` | An OCI image layout tarball |

---

## Commands / files

### See builders

```bash
docker buildx ls
```

```
NAME/NODE     DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default*      docker
 \_ default    \_ default       running   v0.25.1    linux/amd64 (+3), linux/386
```

- `*` = the current builder.
- `PLATFORMS` = what this builder can build **natively or via installed emulators**. `(+3)` = amd64 micro-architecture levels (v2, v3, v4).
- No `linux/arm64` here, so emulation isn't installed yet.

### Install QEMU emulators (binfmt)

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

```
installing: arm64 OK
installing: arm OK
installing: riscv64 OK
installing: ppc64le OK
installing: s390x OK
...
{
  "supported": ["linux/amd64", "linux/arm64", "linux/riscv64", "linux/ppc64le", "linux/s390x", "linux/386", "linux/arm/v7", "linux/arm/v6"],
  "emulators": ["qemu-aarch64", "qemu-arm", "qemu-ppc64le", "qemu-riscv64", "qemu-s390x"]
}
```

- `--privileged` is needed because it registers handlers in the **host kernel's** `binfmt_misc` (topics 40 and 41a explain `--privileged`).
- Registrations don't survive a reboot unless you set them up permanently (e.g. the `qemu-user-static`/`binfmt-support` packages, or rerun this at boot).
- **Docker Desktop includes emulation already.**

Look at what the kernel now knows:

```bash
ls /proc/sys/fs/binfmt_misc/ | grep qemu
cat /proc/sys/fs/binfmt_misc/qemu-aarch64 | head -3
```

```
qemu-aarch64
qemu-arm
...
enabled
interpreter /usr/bin/qemu-aarch64
flags: POCF
```

- `interpreter` = the program the kernel runs for ARM64 binaries.
- `flags: F` = "fix binary": the kernel opens the interpreter **at registration time**, so it works inside containers that don't have QEMU in their filesystem. That's why emulated containers just work.

### Run an image for another architecture

```bash
docker run --rm --platform linux/arm64 alpine uname -m
docker run --rm alpine uname -m
```

```
aarch64
x86_64
```

### Create a `docker-container` builder

```bash
docker buildx create --name multi --driver docker-container --use --bootstrap
docker buildx ls
```

```
NAME/NODE     DRIVER/ENDPOINT              STATUS    BUILDKIT   PLATFORMS
multi*        docker-container
 \_ multi0     \_ unix:///var/run/docker.sock running v0.25.1    linux/amd64 (+3), linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
default       docker
 \_ default    \_ default                  running   v0.25.1    linux/amd64 (+3), linux/arm64, ...
```

- `--use` = make it the current builder. `--bootstrap` = start its BuildKit container now.
- `docker ps` shows a `buildx_buildkit_multi0` container: that's the builder.

### Build and push a multi-platform image

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/<you>/hello-go:1.0 --push .
```

```
[+] Building 38.1s (17/17) FINISHED                                    docker-container:multi
 => [linux/amd64 internal] load metadata for gcr.io/distroless/static-debian12:nonroot  1.2s
 => [linux/arm64 internal] load metadata for gcr.io/distroless/static-debian12:nonroot  1.2s
 => [linux/amd64 build 1/4] FROM docker.io/library/golang:1.25@sha256:…                12.0s
 => [linux/amd64→arm64 build 4/4] RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build…   4.1s
 => [linux/amd64 build 4/4] RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build…         3.9s
 => [linux/arm64 final 1/1] COPY --from=build /out/app /app                             0.1s
 => [linux/amd64 final 1/1] COPY --from=build /out/app /app                             0.1s
 => exporting to image                                                                  4.2s
 => => exporting manifest list sha256:9f1e…                                             0.0s
 => => pushing layers                                                                   3.1s
```

- `[linux/amd64→arm64 build 4/4]` = the build stage runs on amd64 (`$BUILDPLATFORM`) and produces arm64 output: cross-compilation, no QEMU.
- `exporting manifest list` = the index tying both platforms to one tag.

(To push to GHCR you need `docker login ghcr.io` first; topic 22. To practise without a registry, use `--output type=oci,dest=hello.tar`, or with the containerd image store and the default builder, `--load`.)

### Inspect the result

```bash
docker buildx imagetools inspect ghcr.io/<you>/hello-go:1.0
```

```
Name:      ghcr.io/<you>/hello-go:1.0
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:9f1e…

Manifests:
  Name:        ghcr.io/<you>/hello-go:1.0@sha256:aa11…
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64

  Name:        ghcr.io/<you>/hello-go:1.0@sha256:bb22…
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64

  Name:        ghcr.io/<you>/hello-go:1.0@sha256:cc33…
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations:
    vnd.docker.reference.type: attestation-manifest
```

- `unknown/unknown` with `attestation-manifest` = provenance info (how it was built) that BuildKit attaches by default (topic 43).

### Extract files instead of making an image

```bash
docker buildx build --target build --output type=local,dest=./dist .
ls dist/out
```

```
app
```

Handy for using Docker as a reproducible **build environment** that produces a binary for releases.

### `docker buildx bake`: many builds in one file

`docker-bake.hcl`:

```hcl
variable "TAG" { default = "dev" }
variable "REGISTRY" { default = "ghcr.io/acme" }

group "default" {
  targets = ["api", "worker"]
}

target "_common" {
  platforms = ["linux/amd64", "linux/arm64"]
  labels = { "org.opencontainers.image.source" = "https://github.com/acme/shop" }
}

target "api" {
  inherits   = ["_common"]
  context    = "./api"
  tags       = ["${REGISTRY}/api:${TAG}"]
}

target "worker" {
  inherits   = ["_common"]
  context    = "./worker"
  dockerfile = "Dockerfile.worker"
  tags       = ["${REGISTRY}/worker:${TAG}"]
  args       = { WORKER_MODE = "queue" }
}
```

```bash
docker buildx bake --print            # show the resolved config (JSON) without building
TAG=1.4.2 docker buildx bake --push   # build both targets, both platforms, in parallel, and push
docker buildx bake api                # build just one target
```

- **HCL** = HashiCorp Configuration Language (the same style as Terraform). Bake also reads `compose.yaml` and JSON.
- `group` = a set of targets built together. `inherits` = reuse common settings.
- Variables can be set from environment variables (`TAG=1.4.2`).
- This replaces long CI shell scripts with many `docker build` calls.

### Remove a builder

```bash
docker buildx use default
docker buildx rm multi
```

---

## Hands-on lab

1. `docker buildx ls`, then install binfmt and check again. Look in `/proc/sys/fs/binfmt_misc/`.
2. Run `docker run --rm --platform linux/arm64 alpine uname -m`. Then time an emulated CPU task:
   ```bash
   time docker run --rm alpine sh -c 'i=0; while [ $i -lt 2000000 ]; do i=$((i+1)); done'
   time docker run --rm --platform linux/arm64 alpine sh -c 'i=0; while [ $i -lt 2000000 ]; do i=$((i+1)); done'
   ```
   Note how much slower emulation is.
3. Build the Go example for both platforms into an OCI tarball (no registry needed):
   ```bash
   docker buildx create --name multi --driver docker-container --use --bootstrap
   docker buildx build --platform linux/amd64,linux/arm64 --output type=oci,dest=hello-multi.tar .
   tar -xOf hello-multi.tar index.json | python3 -m json.tool | head -30
   ```
   You'll see an index with manifests per platform.
4. Change the build stage to **not** use `--platform=$BUILDPLATFORM` and rebuild for arm64. Watch it get much slower (the Go compiler now runs under QEMU).
5. Write a `docker-bake.hcl` for two small images and run `docker buildx bake --print`.
6. Clean up: `docker buildx use default; docker buildx rm multi; rm hello-multi.tar`

---

## Where you'll use it

- **Saving cloud cost:** moving services to Graviton/ARM nodes requires arm64 (usually multi-platform) images.
- **Mixed Kubernetes clusters:** nodes of both architectures pull the right variant automatically.
- **Developer experience:** Apple Silicon developers get native-speed images.
- **CI:** `docker/setup-qemu-action` + `docker/setup-buildx-action` + `docker/build-push-action` in GitHub Actions, or bake files (topic 46).
- **Releases:** `--output type=local` to produce binaries reproducibly.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `exec /app: exec format error` | The binary is for another CPU architecture than the machine running it (e.g. an amd64 image on an arm64 node) | Build multi-platform, or the right `--platform`. Check with `docker image inspect -f '{{.Architecture}}'` |
| `no matching manifest for linux/arm64/v8 in the manifest list entries` | The image (or your base image) has no arm64 variant | Pick a base that supports arm64, or build it yourself |
| `ERROR: Multi-platform build is not supported for the docker driver. Switch to a different driver, or turn on the containerd image store` | Old overlay2 image store with the default driver | Create a `docker-container` builder, or enable the containerd image store (topic 49) |
| `docker exporter does not currently support exporting manifest lists` | `--load` of a multi-platform build without the containerd store | `--push`, `--output type=oci`, or the containerd store |
| Emulated build is extremely slow or crashes (`qemu: uncaught target signal 11`) | QEMU limits (some programs, JITs (just-in-time compilers that generate machine code while running, like the JVM or V8), or heavy compilation) | Cross-compile with `$BUILDPLATFORM`, or use native arm64 builders |
| `exec /bin/sh: exec format error` during an arm64 build | binfmt/QEMU isn't installed (or was lost on reboot) | `docker run --privileged --rm tonistiigi/binfmt --install arm64` |
| Image works on amd64 but arm64 behaves differently | Different native libraries/wheels per architecture | Test on real arm64 (CI runners exist for it) |

---

## Quick recap

- `docker buildx` = the BuildKit CLI. **Builders** run builds; **drivers** pick where (`docker`, `docker-container`, `kubernetes`, `remote`, cloud).
- Multi-platform image = one tag → an **index** → one manifest per platform. Machines pull their own.
- Other architectures can be built by **QEMU emulation** (registered via the kernel's `binfmt_misc`; easy but slow), **cross-compilation** (`FROM --platform=$BUILDPLATFORM` + `TARGETARCH`; fast), or **native builder nodes**.
- Outputs: local image, `--load`, `--push`, `--output type=local|tar|oci`.
- `docker buildx bake` defines many targets, platforms and tags in one HCL/Compose/JSON file.

**Interview questions**

1. **Q: How do you build a Docker image that runs on both x86 and ARM servers?**
   A: `docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:tag --push .`. BuildKit builds each platform (via QEMU emulation, cross-compilation or native nodes) and pushes an image index under one tag. Each host then pulls its own variant.

2. **Q: What does "exec format error" mean?**
   A: The kernel can't execute the binary because it was built for a different CPU architecture (or it's a script missing a valid shebang). Typically an amd64 image is running on an arm64 host.

3. **Q: Why use `FROM --platform=$BUILDPLATFORM` in a build stage?**
   A: It runs the compiler natively on the build machine instead of under emulation. With `TARGETOS`/`TARGETARCH`, the compiler cross-compiles for each target platform, which is much faster than QEMU.
