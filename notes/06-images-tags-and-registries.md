# 06 · Images, Tags and Registries

> **Level:** ★ foundation
>
> **You will learn:** how image names work (`registry/namespace/repository:tag@digest`), the difference between a **tag** (a movable label) and a **digest** (a permanent fingerprint), how Docker Hub and other registries are organized, and the everyday commands `pull`, `images`, `tag`, `rmi`.
>
> **You need:** topic 05 (what `docker run` does when pulling).

---

## What it is

An **image** is a read-only package that contains:
1. **a filesystem**: all the files your app needs (stored as layers; topic 07),
2. **a config**: settings such as the default command, environment variables, working directory, the user to run as, ports it listens on.

An image is identified by a **name**, which tells Docker *where* to get it and *which version*:

```
ghcr.io/acme/payments-api:2.4.1
└──┬──┘ └┬─┘ └────┬─────┘ └─┬─┘
registry │   repository    tag
      namespace
```

A **registry** is the server that stores images. A **repository** is one image's collection of versions (all the versions of `payments-api`). A **tag** is a human-friendly version label.

---

## Why it exists

You need a reliable way to say "give me **this** app, **this** version, from **this** place", on any machine, without copying files around by hand. Names and tags make images easy to talk about. **Digests** make them exact and tamper-proof.

---

## Analogy

A **library**:
- The **registry** is the library building (Docker Hub, GHCR).
- The **namespace** is the section (`library` for official books, `bitnami` for one publisher's books).
- The **repository** is one book title (`nginx`).
- A **tag** is a **sticky note** on a copy saying "latest edition" or "edition 1.27". Anyone can move the sticky note to another copy.
- The **digest** is the book's exact **ISBN plus a fingerprint of every page**. It can never point to a different book. If even one letter changed, the fingerprint changes.

---

## How it works inside

### 1. The full name, part by part

```
[registry/][namespace/]repository[:tag][@digest]
```

| Part | Example | If you leave it out |
|---|---|---|
| registry | `docker.io`, `ghcr.io`, `123456789012.dkr.ecr.ap-south-1.amazonaws.com`, `localhost:5000` | `docker.io` (Docker Hub) |
| namespace | `library`, `bitnami`, `acme` | `library` (only on Docker Hub; that's where official images live) |
| repository | `nginx`, `payments-api` | Required |
| tag | `1.27.3`, `1.27-alpine`, `latest` | `latest` |
| digest | `sha256:5d16…` | Not used |

How Docker decides if the first part is a registry: if it contains a `.` or a `:` (a port), or is exactly `localhost`, it's treated as a registry host. So `myteam/app` = Docker Hub user `myteam`, but `myteam.io/app` = a registry at `myteam.io`.

Examples of expansion:

| You type | Docker uses |
|---|---|
| `nginx` | `docker.io/library/nginx:latest` |
| `nginx:1.27-alpine` | `docker.io/library/nginx:1.27-alpine` |
| `bitnami/redis` | `docker.io/bitnami/redis:latest` |
| `ghcr.io/acme/api:v2` | exactly that |
| `localhost:5000/api` | `localhost:5000/api:latest` |

### 2. Tags are movable labels

A tag is just a pointer from a name to a manifest. The registry stores "`nginx:1.27` → manifest `sha256:abc…`". When nginx 1.27.4 comes out, the maintainers **move** the `1.27` tag to the new manifest.

So `nginx:1.27` today and `nginx:1.27` next month can be **different images**. This is useful (you get patches) and dangerous (things change without you noticing).

### 3. `latest` is not magic

`latest` is **just the default tag name** used when you don't give one. It does **not** mean "newest":
- If a maintainer pushes `app:2.0` but never moves `latest`, `latest` still points to whatever it pointed to before.
- `docker run nginx` uses your **local** copy of `nginx:latest` if you already have one. It doesn't check for a newer one. Only `docker pull` (or `--pull always`) checks.

**In production, never deploy `latest`.** You can't tell which version is running, and rollbacks are guesswork. Use specific tags, and digests for full certainty (topic 22).

### 4. Digests are permanent fingerprints

A **digest** is the SHA-256 hash of the image's manifest. SHA-256 is a function that turns any data into a 64-hex-character fingerprint. Change one byte of input, and the fingerprint changes completely.

The manifest lists the SHA-256 of the config and of every layer. So **one digest locks the whole image**: every file, every setting.

```
nginx@sha256:5d16c8d2ff5d…   ← always exactly the same bytes, forever
```

- If the registry (or an attacker) tried to serve different content for that digest, the hash check would fail and Docker would refuse it.
- You can combine both forms: `nginx:1.27@sha256:5d16…`. The tag is for humans to read. When a digest is present, **Docker uses the digest** and ignores the tag.

### 5. Multi-platform images: one name, several variants

Many images are built for several CPU types. The tag points to an **image index** (also called a **manifest list**), which lists one manifest per **platform**:

```
nginx:1.27  →  index sha256:5d16…
                 ├─ linux/amd64   → manifest sha256:aa11…  (Intel/AMD servers)
                 ├─ linux/arm64   → manifest sha256:bb22…  (AWS Graviton, Apple M-series)
                 ├─ linux/arm/v7  → manifest sha256:cc33…  (Raspberry Pi)
                 └─ ...
```

When you pull, Docker picks the variant matching your machine. That's why the same command works on an Intel server and an ARM laptop. Topic 21 covers building these.

### 6. Image ID vs digest

Two different numbers, often confused:

| | Image ID | Digest (RepoDigest) |
|---|---|---|
| What it's a hash of | Your **local** image (the config, or with the containerd store, the index/manifest) | The manifest/index as stored **in the registry** |
| Where it's meaningful | Only on this machine | Everywhere: in any registry, on any machine |
| Used for | Local commands (`docker rmi 1a2b3c`) | Pinning, deployments, security |

### 7. Registries and repositories

| Registry | Address | Notes |
|---|---|---|
| Docker Hub | `docker.io` (API at `registry-1.docker.io`) | The default. Official images under `library/`. Rate limits for anonymous and free users. |
| GitHub Container Registry | `ghcr.io` | Tied to GitHub accounts and Actions |
| Amazon ECR | `<account>.dkr.ecr.<region>.amazonaws.com` | AWS's private registry |
| Google Artifact Registry | `<region>-docker.pkg.dev` | Google Cloud |
| Azure Container Registry | `<name>.azurecr.io` | Azure |
| Quay | `quay.io` | Red Hat's registry |
| Harbor | your own hostname | Self-hosted, with scanning and replication |

Kinds of images on Docker Hub:
- **Docker Official Images** (`library/…`, e.g. `nginx`, `postgres`, `python`): maintained with Docker, reviewed, regularly rebuilt with security fixes. The best default.
- **Verified Publisher** images: from vendors who have been verified (e.g. `bitnami/…`, `grafana/…`).
- **Community** images: anyone can push. Treat them like random code from the internet. Read the Dockerfile and check how they're maintained.

### 8. What "pull" actually stores locally

When you pull, the daemon downloads the index, the manifest for your platform, the config and the layers (topic 05). It stores:
- the content (compressed and/or unpacked layers),
- a **name → image** record, so `nginx:1.27` locally points to what was downloaded.

Pulling a second image that shares layers with one you have **skips those layers** (`Already exists`). That's a big reason images share common base images.

---

## Commands / files

### `docker pull`

```bash
docker pull nginx:1.27-alpine
```

```
1.27-alpine: Pulling from library/nginx
f18232174bc9: Pull complete
61ca4f733c80: Pull complete
b464cfdf2a63: Pull complete
d7e507024086: Pull complete
81bd8ed7ec67: Pull complete
197eb75867ef: Pull complete
34a64644b756: Pull complete
39c2ddfd6010: Pull complete
Digest: sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
Status: Downloaded newer image for nginx:1.27-alpine
docker.io/library/nginx:1.27-alpine
```

- `Pulling from library/nginx` = Docker Hub, official namespace.
- Each `f18232174bc9: Pull complete` = one **layer** (the short ID of its digest) downloaded, verified and unpacked.
- `Digest:` = the digest of what the tag points to **right now**. Save this if you want this exact image later.
- The last line = the fully expanded name.

Pull again:

```
1.27-alpine: Pulling from library/nginx
Digest: sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
Status: Image is up to date for nginx:1.27-alpine
```

- Docker asked the registry "what does this tag point to now?" Same digest → nothing to download.

Pull by digest:

```bash
docker pull nginx@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
```

Pull a specific platform:

```bash
docker pull --platform linux/arm64 alpine:3.22
```

### `docker images` (same as `docker image ls`)

Newer CLIs (around Docker 29) show a grouped "tree-style" view by default. The exact column names have changed between CLI releases, so treat this as an example and confirm with `docker images --help` on your version:

```bash
docker images
```

```
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   56433a6be3fd       20.3kB         3.96kB    U
nginx:1.27-alpine    65645c7bb6a0       77.8MB         21.4MB
ubuntu:latest        66460d557b25        117MB         29.7MB    U
```

- `IMAGE` = name:tag.
- `ID` = the first 12 characters of the local image ID.
- `DISK USAGE` = space used on disk (including unpacked layers).
- `CONTENT SIZE` = the compressed size (roughly what was downloaded).
- `EXTRA` → `U` = **in use** by at least one container (running or stopped). You can't delete it without removing those containers.

Older CLIs show the classic view. On any version, you get it with an explicit format, which is also what you should use in scripts (a format string never changes when the default view does):

```bash
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.CreatedSince}}\t{{.Size}}'
```

```
REPOSITORY    TAG           IMAGE ID       CREATED        SIZE
nginx         1.27-alpine   65645c7bb6a0   3 weeks ago    77.8MB
ubuntu        latest        66460d557b25   4 weeks ago    117MB
hello-world   latest        56433a6be3fd   6 months ago   20.3kB
```

- `CREATED` = when the image was **built**, not when you pulled it.

Show digests too:

```bash
docker images --digests --format 'table {{.Repository}}\t{{.Tag}}\t{{.Digest}}'
```

```
REPOSITORY    TAG           DIGEST
nginx         1.27-alpine   sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
ubuntu        latest        sha256:66460d557b25769b102175144d538d88219c077c678a49af4afca6fbfc1b5252
```

Filter:

```bash
docker images nginx                          # only the nginx repository
docker images --filter dangling=true         # untagged leftovers (topic 13)
docker images -q                             # only IDs, handy in scripts
```

### `docker tag`: add another name

```bash
docker tag nginx:1.27-alpine myregistry.example.com/web/nginx:1.27-alpine
docker images --format 'table {{.Repository}}:{{.Tag}}\t{{.ID}}'
```

```
REPOSITORY:TAG                                     IMAGE ID
myregistry.example.com/web/nginx:1.27-alpine        65645c7bb6a0
nginx:1.27-alpine                                  65645c7bb6a0
```

- Same **ID** = the same image. `docker tag` copies nothing; it just adds a name. You need a name with a registry prefix before you can push there (topic 22).

### `docker image inspect`: see the config and digests

```bash
docker image inspect nginx:1.27-alpine --format '{{json .RepoDigests}}'
docker image inspect nginx:1.27-alpine --format '{{.Config.Cmd}} {{.Config.ExposedPorts}} {{.Architecture}}/{{.Os}}'
```

```
["nginx@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10"]
[nginx -g daemon off;] map[80/tcp:{}] amd64/linux
```

- `RepoDigests` = the digest(s) this image has in a registry.
- `Config.Cmd` = the default command.
- `ExposedPorts` = ports the image says it uses (documentation only; topic 28).
- `amd64/linux` = the platform you got.

### Look at a remote image without pulling it

```bash
docker buildx imagetools inspect nginx:1.27-alpine
```

```
Name:      docker.io/library/nginx:1.27-alpine
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10

Manifests:
  Name:        docker.io/library/nginx:1.27-alpine@sha256:4a2c…
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64

  Name:        docker.io/library/nginx:1.27-alpine@sha256:9b7e…
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64/v8
  ...
```

- `MediaType: ...image.index...` = it's a multi-platform index.
- Each entry under `Manifests` = one platform variant with its own digest. Some entries show `Platform: unknown/unknown`: those are **attestations** (signed build info; topic 43), not runnable images.

### `docker rmi` (same as `docker image rm`)

```bash
docker rmi myregistry.example.com/web/nginx:1.27-alpine
```

```
Untagged: myregistry.example.com/web/nginx:1.27-alpine
```

- Only the **name** was removed, because another name (`nginx:1.27-alpine`) still points to the image.

```bash
docker rmi nginx:1.27-alpine
```

```
Untagged: nginx:1.27-alpine
Untagged: nginx@sha256:65645c7bb6a0…
Deleted: sha256:65645c7bb6a0…
```

- Last name gone → the image (and any layers no other image uses) is deleted.

### Search Docker Hub from the CLI

```bash
docker search --filter is-official=true redis
```

```
NAME      DESCRIPTION                                     STARS     OFFICIAL
redis     Redis is the world's fastest data platform f…   13000     [OK]
```

(For real choices, use the Docker Hub website: it shows tags, sizes, vulnerabilities and the Dockerfile link.)

---

## Hands-on lab

1. **See that a tag can move but a digest can't.** Pull an old and a new tag and compare:
   ```bash
   docker pull alpine:3.21
   docker pull alpine:3.22
   docker images alpine --digests --format '{{.Tag}} {{.Digest}}'
   ```
   Different digests. Now "move a tag" yourself:
   ```bash
   docker tag alpine:3.21 alpine:mytag
   docker images alpine --format '{{.Tag}} {{.ID}}'
   docker tag alpine:3.22 alpine:mytag
   docker images alpine --format '{{.Tag}} {{.ID}}'
   ```
   `mytag` first had 3.21's ID, then 3.22's. That's exactly what registry maintainers do with `latest`.

2. **Prove `docker run` doesn't check for updates:**
   ```bash
   docker run --rm alpine:mytag cat /etc/alpine-release
   docker run --rm --pull always alpine:3.22 cat /etc/alpine-release
   ```
   `--pull always` asks the registry first. The default (`missing`) only pulls if the image isn't local.

3. **See the platforms of an image:**
   ```bash
   docker buildx imagetools inspect python:3.13-slim | grep Platform
   ```

4. **See shared layers:**
   ```bash
   docker pull python:3.13-slim
   docker pull python:3.12-slim
   ```
   Watch for `Already exists` on layers the second image shares with the first (both are built on the same Debian base).

5. Clean up: `docker rmi alpine:mytag alpine:3.21 alpine:3.22 python:3.13-slim python:3.12-slim`

---

## Where you'll use it

- **Deployments:** Kubernetes manifests and Compose files reference images by tag or digest. Production best practice is a version tag plus a digest.
- **Incident response:** "which exact image is running?" → you check the digest, not the tag.
- **Security scanning:** scanners report vulnerabilities per image digest.
- **Registries per environment:** images flow from a dev registry to a production registry (topic 22).
- **Cost/speed:** choosing images that share base layers saves disk and pull time.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Error response from daemon: manifest for nginx:1.99 not found: manifest unknown` | That tag doesn't exist in the repository | Check the tag on the registry website, or `docker buildx imagetools inspect` |
| `pull access denied for myapp, repository does not exist or may require 'docker login'` | Wrong name, a private repository, or you're not logged in. Note: `myapp` alone means `docker.io/library/myapp`, which doesn't exist | Fix the full name (registry and namespace), then `docker login <registry>` |
| `no matching manifest for linux/arm64/v8 in the manifest list entries` | No variant for your CPU | Use another tag/image, build multi-platform (topic 21), or `--platform linux/amd64` with emulation |
| `toomanyrequests: You have reached your unauthenticated pull rate limit` | Docker Hub rate limit | Log in; use a pull-through cache/mirror (topic 34) |
| `Error response from daemon: conflict: unable to remove repository reference "nginx" (must force) - container 3f2a… is using its referenced image` | A container (even a stopped one) uses this image | Remove the container first, or `docker rmi -f` (it untags; the data stays while used) |
| `image with reference X was found but does not match the specified platform` | You asked for a different platform than the one you have locally | Pull with `--platform` |
| The app "suddenly changed behaviour" after a redeploy, with the same tag | The tag was moved upstream | Pin by digest; read the release notes |

---

## Quick recap

- Full name: `registry/namespace/repository:tag@digest`. Defaults: `docker.io`, `library`, `latest`.
- A **tag** is a movable label. A **digest** is a permanent SHA-256 fingerprint of the manifest that locks every layer and setting.
- `latest` just means "no tag given". It's not "newest", and `docker run` doesn't check for updates.
- A tag can point to an **index** with one manifest per platform (amd64, arm64…). Docker picks yours.
- `docker tag` only adds a name. `docker rmi` removes names, and deletes data only when the last name is gone and no container uses it.

**Interview questions**

1. **Q: What is the difference between a tag and a digest?**
   A: A tag is a human-readable, mutable pointer that can be moved to a new image. A digest is the SHA-256 hash of the image manifest: immutable, and it uniquely identifies exact content.

2. **Q: Why shouldn't you use `latest` in production?**
   A: It's ambiguous and mutable. You can't tell which version is running, different nodes might pull different images, and rollbacks are unreliable. Use version tags, ideally pinned by digest.

3. **Q: How can the same image name work on both an Intel server and an ARM laptop?**
   A: The tag points to an image index (manifest list) that has a manifest per platform. The client picks the one matching its OS/architecture.
