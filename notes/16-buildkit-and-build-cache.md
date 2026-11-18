# 16 · BuildKit and the Build Cache

> **Level:** ★★ intermediate
>
> **You will learn:** how BuildKit turns a Dockerfile into a graph of steps, how it decides whether a step can be reused from cache (the exact rules), why ordering your Dockerfile matters so much, how cache mounts make package installs fast, how to read `CACHED` in build output, and how to control and inspect the cache.
>
> **You need:** topic 07 (layers, content addressing), topic 14 (Dockerfile basics, build context).

---

## What it is

**BuildKit** is Docker's build engine. It's built into dockerd and has been the default since Docker 23. (The legacy builder is **deprecated**.)

The **build cache** is BuildKit's memory of previous build steps. If a step's inputs haven't changed, BuildKit **reuses the stored result** instead of running the step again. A rebuild that took 5 minutes can take 2 seconds.

---

## Why it exists

Developers and CI systems build images many times a day. Most builds change only a few lines of code. Re-downloading the base image, reinstalling every package and recompiling everything for a one-line change wastes minutes per build and hours per day.

---

## Analogy

**Meal prep with labelled containers.** You cook a recipe step by step, and after each step you put the result in a labelled box: "step 3: chopped onions, from 2 onions bought Monday". Next time, if the label matches (same ingredients, same step), you just grab the box. But if you change step 3 (3 onions now), then step 3 **and everything after it** must be redone, because every later step used those onions.

---

## How it works inside

### 1. From Dockerfile to a graph

```
Dockerfile ──► frontend (dockerfile:1) ──► LLB graph ──► solver ──► workers run steps ──► exporter (image)
```

- **Frontend**: the part that understands Dockerfile syntax (that's what `# syntax=docker/dockerfile:1` picks). It converts the Dockerfile into **LLB** (Low-Level Build definition): a graph of operations (pull this image, run this command, copy these files).
- **Solver**: walks the graph, works out which nodes are already cached, and runs only the rest. Independent branches (like separate stages in a multi-stage build, topic 17) run **in parallel**. Stages the final image doesn't need are **skipped**.
- **Exporter**: writes the result: a local image, a push to a registry, a tar file, or a local folder.

### 2. The cache key: when is a step reused?

Each step gets a **cache key** made from **its parent's key + its own inputs**:

| Instruction | What goes into its key |
|---|---|
| `FROM image:tag` | The image digest the tag resolves to |
| `RUN cmd` | The **exact command text** + the parent step + the environment/args that apply |
| `COPY` / `ADD` (local files) | A **checksum of the files' contents** (and their permissions and names), not the timestamps |
| `ENV`, `WORKDIR`, `USER`, `ARG` values used by the step | Their values |
| `ADD <url>` | The remote file's content (checked each build) |

The chain rule: **once a step misses the cache, every step after it misses too**, because each key includes its parent's key.

```
FROM python:3.13-slim          ✅ cached (same digest)
WORKDIR /app                   ✅ cached
COPY . .                       ❌ app.py changed → checksum differs
RUN pip install -r req.txt     ❌ re-runs (parent changed), even though req.txt didn't!
```

### 3. What does NOT invalidate the cache (gotchas)

- **The outside world.** `RUN apt-get update` has the same text every time, so it stays cached **forever**, even if the package lists are a month old. The same goes for `RUN curl https://…/latest.tar.gz` and `RUN git clone …`. BuildKit doesn't know the remote content changed.
- **The base image tag moving**, if you already have the old image locally. Use `docker build --pull` to check for a newer base.
- **File timestamps**: `touch app.py` doesn't break the cache. Only content changes do.

### 4. Order instructions from least to most frequently changed

**Bad** (every code change reinstalls all dependencies):

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "app.py"]
```

**Good** (dependencies cached until `requirements.txt` changes):

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

The same pattern for every language:

| Language | Copy first (rarely changes) | Install | Then copy code |
|---|---|---|---|
| Python | `requirements.txt` / `pyproject.toml` + lock file | `pip install` / `uv sync` | `COPY . .` |
| Node | `package.json`, `package-lock.json` | `npm ci` | `COPY . .` |
| Go | `go.mod`, `go.sum` | `go mod download` | `COPY . .` |
| Java (Maven) | `pom.xml` | `mvn dependency:go-offline` | `COPY src ./src` |

### 5. Cache mounts: keep package caches between builds

Even with good ordering, changing **one** dependency reinstalls all of them from the internet. A **cache mount** gives a `RUN` step a folder that **survives between builds** but **is not part of the image**:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

- `--mount=type=cache,target=/root/.cache/pip` = mount a persistent cache folder at pip's cache location during this step only.
- pip finds already-downloaded packages there and skips the downloads.
- The folder isn't in the image layer, so the image stays small (that's why `--no-cache-dir` isn't needed here).

Common targets:

| Tool | Target |
|---|---|
| pip | `/root/.cache/pip` |
| uv | `/root/.cache/uv` |
| npm | `/root/.npm` |
| Go | `/root/.cache/go-build` and `/go/pkg/mod` |
| Maven | `/root/.m2` |
| apt | `/var/cache/apt` and `/var/lib/apt` (with `sharing=locked`, and remove the image's `docker-clean` config) |

Cache mounts live on the **builder**. On a CI runner that is new every time, they start empty, unless you persist the builder (topic 46).

### 6. Bind mounts in RUN: use files without copying them in

```dockerfile
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

- The file is available during this step only, and doesn't become a layer by itself. The step's cache key still depends on the file's checksum.

(Secret and SSH mounts, `--mount=type=secret` and `type=ssh`, are in topic 18.)

### 7. Where the cache lives and how it's cleaned

- With the default builder (`docker` driver), the cache is stored by dockerd/BuildKit on the host (`/var/lib/docker/buildkit/` plus the image store).
- BuildKit has **garbage collection** (GC): when the cache passes a size limit, old entries are removed. Defaults are based on disk size, and you can configure them in `daemon.json` (topic 34) or clean manually with `docker builder prune` (topic 13).

---

## Commands / files

### See the cache working

Using the "good" Dockerfile from section 4 with the `hello-flask` app from topic 14:

```bash
docker build -t hf:dev .           # first build
echo "# comment" >> app.py
docker build -t hf:dev .           # code changed
```

Second build output:

```
[+] Building 1.2s (11/11) FINISHED                                         docker:default
 => [internal] load build definition from Dockerfile                                 0.0s
 => [internal] load metadata for docker.io/library/python:3.13-slim                  0.4s
 => [internal] load .dockerignore                                                    0.0s
 => [1/5] FROM docker.io/library/python:3.13-slim@sha256:…                           0.0s
 => [internal] load build context                                                    0.0s
 => => transferring context: 412B                                                    0.0s
 => CACHED [2/5] WORKDIR /app                                                        0.0s
 => CACHED [3/5] COPY requirements.txt .                                             0.0s
 => CACHED [4/5] RUN pip install --no-cache-dir -r requirements.txt                  0.0s
 => [5/5] COPY app.py .                                                              0.0s
 => exporting to image                                                               0.1s
```

- `CACHED` = this step's result was reused. The slow `pip install` was skipped.
- Only step 5 ran, because only `app.py` changed.

### See full output of each step

```bash
docker build --progress=plain -t hf:dev . 2>&1 | tail -20
```

- `--progress=plain` prints every line each `RUN` outputs. Use it to debug failing or slow steps.
- `#7 [4/5] RUN pip install …` then `#7 CACHED` or `#7 DONE 6.9s`.

### Force rebuilds

```bash
docker build --no-cache -t hf:dev .                      # ignore all cache
docker build --no-cache-filter final -t hf:dev .         # ignore cache only for the stage named "final" (topic 17)
docker build --pull -t hf:dev .                          # re-check the base image tag
```

A common trick to refresh `apt-get update` periodically: put a changing value in the command, for example with a build argument (topic 18):

```dockerfile
ARG CACHE_BUST=2026-11
RUN echo "$CACHE_BUST" && apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*
```

### Inspect and clean the cache

```bash
docker buildx du
```

```
ID                                              RECLAIMABLE     SIZE            LAST ACCESSED
r7h2v1x0q9w8e7t6y5u4i3o2p                       true            156.3MB         2 minutes ago
m1n2b3v4c5x6z7a8s9d0f1g2h*                      true            98.12MB         2 days ago
...
Shared:         120.4MB
Private:        1.92GB
Reclaimable:    2.04GB
Total:          2.04GB
```

- Each entry is a cached step result, a transferred build context, or a cache mount.
- `docker buildx du --verbose` shows each entry's **type** (`regular` = a step's result, `source.local` = a transferred build context, `exec.cachemount` = a `RUN --mount=type=cache` folder), its description (the step that made it), and when it was last used.

```bash
docker builder prune --filter until=72h -f
docker builder prune -a -f        # everything
```

### Compare build times with and without a cache mount

```bash
# Version A: no cache mount. Add one package to requirements.txt and rebuild → all packages re-download.
# Version B: with --mount=type=cache. Add one package and rebuild → only the new one downloads.
```

You'll see lines like `Using cached flask-3.1.2-py3-none-any.whl` in `--progress=plain` output for version B.

---

## Hands-on lab

1. **Bad vs good order.** Build the "bad" Dockerfile, change `app.py`, rebuild, and note that `pip install` runs again. Switch to the "good" one, build, change `app.py`, rebuild: `CACHED` on pip.
2. **Timestamps don't matter:** `touch requirements.txt` and rebuild. Still `CACHED`. Now add a blank line to `requirements.txt` and rebuild: the install step reruns (the content changed).
3. **The stale `apt-get update` trap:**
   ```dockerfile
   # syntax=docker/dockerfile:1
   FROM debian:bookworm-slim
   RUN apt-get update
   RUN apt-get install -y --no-install-recommends curl
   ```
   Build. Then change the second line to install `wget` too, and build again. The `apt-get update` step is `CACHED` from earlier (stale lists). This is why `update` and `install` must be in **one** `RUN`.
4. **Cache mount:** add `--mount=type=cache,target=/root/.cache/pip` to the pip step, add `requests` to `requirements.txt`, and build with `--progress=plain`. Look for `Using cached` lines on the second change.
5. `docker buildx du` before and after `docker builder prune -a -f`.

---

## Where you'll use it

- **Developer speed:** good ordering turns 3-minute rebuilds into seconds.
- **CI speed and cost:** CI runners usually start empty, so you **export** cache to a registry and **import** it on the next run (`--cache-to` / `--cache-from`, topic 46).
- **Security patching:** knowing when to bypass the cache (`--pull`, `--no-cache`) so base images and packages are really refreshed.
- **Debugging builds:** `--progress=plain` to see what a step did, and why it did or didn't use the cache.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Every build reinstalls all dependencies | Code is copied before the install step, so any code change invalidates it | Copy dependency manifests first, install, then copy code |
| Build "works" but installs old package versions | `apt-get update` (or a download) is cached from long ago | `update` + `install` in one `RUN`; `--no-cache` periodically; or a cache-bust ARG |
| Base image security fix not picked up | Local copy of the base tag is reused | `docker build --pull`; or pin by digest and update it deliberately (topic 20) |
| Cache never hits in CI | Each CI job starts with an empty builder | Use `--cache-from`/`--cache-to` with a registry or the CI cache (topic 46) |
| Cache misses after an unrelated change | A file in the context that the build copies changed (e.g. `COPY . .` copies `.git`, logs, local env files) | Better `.dockerignore`; copy only what's needed |
| `ERROR: failed to solve: ... no space left on device` | Build cache filled the disk | `docker builder prune`; configure GC limits (topic 34, 48) |
| `--mount` options rejected | Missing `# syntax=docker/dockerfile:1` on an older engine, or legacy builder in use | Add the syntax line; don't set `DOCKER_BUILDKIT=0` |

---

## Quick recap

- BuildKit turns the Dockerfile into a graph (LLB), runs independent parts in parallel, skips unused stages, and reuses cached step results.
- A step's cache key = parent's key + its own inputs (command text, file **contents** for COPY/ADD, relevant ARG/ENV values). **A miss invalidates every step after it.**
- Order from least to most frequently changed: base → system packages → dependency manifests → install → source code.
- `RUN --mount=type=cache` keeps package-manager caches between builds without bloating the image.
- The cache doesn't know about the outside world. `--pull` and `--no-cache` force refreshes.

**Interview questions**

1. **Q: How does Docker decide whether to use the cache for a COPY step?**
   A: It computes a checksum of the copied files' contents and metadata (not timestamps) and combines it with the parent step's cache key. Same key → reused. Any content change → this step and all following steps rebuild.

2. **Q: How would you speed up a slow Dockerfile build?**
   A: Order instructions so dependencies install before the source code is copied. Use a good `.dockerignore`. Use cache mounts for package managers. Use multi-stage builds so independent stages run in parallel. In CI, export and import the cache from a registry.

3. **Q: Why can `RUN apt-get update` on its own line be dangerous?**
   A: Its command text never changes, so it's cached indefinitely. A later `apt-get install` step then works from stale package lists, which can mean old versions or "package not found" errors. Combine update and install in one RUN.
