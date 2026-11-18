# 38 · Compose for Development

> **Level:** ★★ intermediate
>
> **You will learn:** how to get a fast edit-and-see loop with containers: bind-mounted source code with hot reload (and the `node_modules` trap), the modern **`docker compose watch`** (sync / rebuild / sync+restart), the `build:` section in depth, **override files** and how Compose merges several files, and splitting big setups with **`include`** and **`extends`**.
>
> **You need:** topics 35–37, topic 25 (bind mounts, UIDs), topics 14–17 (builds, stages).

---

## What it is

In development you want:
1. **Edit code on your machine → see the change in the running container within seconds**, without a full rebuild.
2. **The same compose file** (mostly) for dev and production, with dev-only additions kept separate.
3. **Split configuration** for big projects (many services, several teams).

Compose gives you three tools:
- **bind mounts** of your source folder (the classic way),
- **`develop.watch`** + `docker compose watch` (the modern way),
- **multiple files** (`compose.override.yaml`, `-f`, `include`, `extends`).

---

## Why it exists

Rebuilding an image for every code change (even with a good cache) takes seconds to minutes and breaks your flow. Developers then skip containers and run things directly on their laptop, which brings back "works on my machine" (topic 01). Fast container dev loops keep everyone on the same environment.

---

## Analogy

Testing a recipe:
- **Rebuild every time** = baking a whole new cake to taste a pinch more salt.
- **Bind mount** = cooking at a stove that's **directly wired to your recipe book**. Change the book, and the cook sees it immediately. But the cook can also scribble back into your book (writes go both ways).
- **Watch (sync)** = an assistant who **copies** each changed page of your book to the cook's station the moment you save. If you change the shopping list (dependencies), the assistant says "this needs a full re-bake" (rebuild).

---

## How it works inside

### Approach 1: bind mount + the framework's hot reload

```yaml
services:
  web:
    build: .
    command: ["flask", "--app", "app", "run", "--host", "0.0.0.0", "--port", "8000", "--debug"]
    volumes:
      - ./:/app              # your source folder → /app in the container
    ports:
      - "8000:8000"
```

- The container sees your files live (topic 25). Frameworks with **auto-reload** (Flask `--debug`, `uvicorn --reload`, `nodemon`, `air` for Go, Spring DevTools) notice the change and restart themselves.
- File change notifications (**inotify**, the Linux kernel's file-watching feature) work across bind mounts on native Linux and inside WSL's own filesystem. They're unreliable from `/mnt/c` (topic 25). If reload doesn't trigger, the framework may need **polling** mode.

**The `node_modules` trap** (and the same thing for Python `.venv`, `vendor/` …):
- The image ran `npm ci`, so `/app/node_modules` in the image has Linux-built packages.
- The bind mount `./:/app` **covers all of `/app`**, including `node_modules`. Now the container sees **your host's** `node_modules` (maybe empty, maybe built for Windows/macOS) → crashes like `Cannot find module` or `invalid ELF header`.
- Classic fix: mount an **anonymous volume** at that sub-path, so the image's folder shows through:
  ```yaml
  volumes:
    - ./:/app
    - /app/node_modules        # anonymous volume: keeps the image's node_modules
  ```
  A more specific mount (`/app/node_modules`) wins over a broader one (`/app`). The empty anonymous volume is pre-filled from the image (topic 24). After changing dependencies, you must recreate it (`docker compose up -d --renew-anon-volumes` or `-V`).

**The ownership trap:** files the container creates in your source folder (caches, generated files) are owned by the container's UID (often root). Run dev containers with your UID: `user: "${UID}:${GID}"` (export UID/GID in your shell or `.env`), topic 25.

### Approach 2: `docker compose watch` (recommended)

Instead of mounting, Compose **watches your files** and acts on changes:

```yaml
services:
  web:
    build: .
    ports: ["8000:8000"]
    command: ["flask", "--app", "app", "run", "--host", "0.0.0.0", "--port", "8000", "--debug"]
    develop:
      watch:
        - action: sync               # copy changed files into the running container
          path: ./src
          target: /app/src
          ignore:
            - "**/__pycache__/"
        - action: rebuild            # rebuild the image and recreate the container
          path: requirements.txt
        - action: sync+restart       # copy, then restart the container (for config files the app reads only at startup)
          path: ./config
          target: /app/config
```

| Action | What happens when a matching file changes | Use for |
|---|---|---|
| `sync` | Compose copies the file into the container (like `docker cp`), with no restart. The framework's hot reload picks it up | Source code |
| `rebuild` | `docker compose build` + recreate the service | Dependency manifests (`package.json`, `requirements.txt`, `go.mod`), the Dockerfile |
| `sync+restart` | Copy, then restart the container's main process | Config files read only at start |
| `sync+exec` | Copy, then run a command in the container | e.g. clear a cache |

Advantages over bind mounts:
- No `node_modules` problem: nothing is mounted over the image's folders.
- No permission/ownership mess: files are copied in, with the container's ownership.
- Works the same on Linux, WSL, macOS and Windows (no slow cross-OS mounts).
- Dependency changes trigger a proper rebuild automatically.

Run with:

```bash
docker compose watch            # starts the services (if needed) and watches
docker compose up --watch       # up + watch in one, with logs in the same terminal
```

Requirement: the image needs the files at `target` already (they're copied in at build), and for `sync` the container must have a way to write them (Compose copies through the API; the target folder must be writable by the container's user).

### The `build:` section in depth

```yaml
services:
  api:
    image: ghcr.io/acme/api:${TAG:-dev}       # the tag given to the built image
    build:
      context: ./api                          # build context folder
      dockerfile: Dockerfile.dev              # relative to the context
      target: dev                             # multi-stage target (topic 17)
      args:
        PY_VERSION: "3.13"
      secrets: [npmrc]                        # build secrets (topic 18)
      ssh: [default]
      cache_from: [type=registry,ref=ghcr.io/acme/api:buildcache]
      platforms: [linux/amd64, linux/arm64]
      labels:
        org.opencontainers.image.source: https://github.com/acme/api
secrets:
  npmrc:
    file: ~/.npmrc
```

- One Dockerfile with stages `dev` (with debug tools) and `final` (production) → `target: dev` in development.
- `docker compose build --pull --no-cache api` / `docker compose up -d --build`.
- Compose can also hand builds to **Bake** (topic 21) for parallel builds. Turn it on with `COMPOSE_BAKE=true`. Newer Compose releases have been moving toward Bake as the default, so check your version's release notes (`docker compose version`) rather than assuming.

### Multiple files and how they merge

By default, `docker compose` reads **`compose.yaml` + `compose.override.yaml`** (if present) and merges them. Typical use:

```
compose.yaml            ← shared base (production-like)
compose.override.yaml   ← dev-only additions; loaded automatically on dev machines
compose.prod.yaml       ← production additions; used explicitly
```

```bash
docker compose up -d                                          # base + override (dev)
docker compose -f compose.yaml -f compose.prod.yaml up -d     # base + prod (no override!)
```

Once you pass `-f`, only the files you list are used.

**Merge rules** (later files win):
- **Single values** (`image`, `command`, `restart`, `user`…): **replaced**.
- **Maps** (`environment`, `labels`, `deploy.resources`…): **merged key by key**.
- **`ports`, `expose`, `dns`, …**: **combined** (added together).
- **`volumes`, `devices`**: merged **by container path** (a later file can replace the mount at `/app`).
- To **remove** something set in a base file: `!reset` (e.g. `ports: !reset []`). To replace a whole list or map instead of merging: `!override`.

Always check the result: `docker compose -f a.yaml -f b.yaml config`.

### `include`: compose files made of other compose files

```yaml
# compose.yaml (top level)
include:
  - path: ./infra/compose.yaml        # postgres, redis, … owned by the platform team
  - path: ./observability/compose.yaml
    env_file: ./observability/.env
services:
  api:
    build: ./api
    depends_on: [db]                  # 'db' is defined in the included infra file
```

- Each included file keeps its own **relative paths** and `.env` (resolved from its own folder). That makes it ideal for separate teams/repos.
- Everything ends up in **one project** (one network by default, so services see each other by name).
- Conflicting service names are an error (no silent overriding).

### `extends`: reuse one service's definition

```yaml
# common.yaml
services:
  base-python:
    image: python:3.13-slim
    environment: { PYTHONUNBUFFERED: "1" }
    restart: unless-stopped
```

```yaml
# compose.yaml
services:
  worker:
    extends:
      file: common.yaml
      service: base-python
    command: ["python", "worker.py"]
```

- `worker` starts as a copy of `base-python`, then applies its own settings on top. (`depends_on` and some links aren't inherited.)

---

## Commands / files

### Dev setup with override

`compose.yaml` (base):

```yaml
services:
  web:
    image: ghcr.io/acme/web:${TAG:-latest}
    build:
      context: .
      target: final
    environment:
      LOG_LEVEL: info
    restart: unless-stopped
```

`compose.override.yaml` (dev):

```yaml
services:
  web:
    build:
      target: dev
    environment:
      LOG_LEVEL: debug
      FLASK_DEBUG: "1"
    ports:
      - "127.0.0.1:8000:8000"
      - "127.0.0.1:5678:5678"          # debugger port (e.g. debugpy)
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: requirements.txt
```

```bash
docker compose config | grep -E 'target|LOG_LEVEL|published'
```

```
      target: dev
      LOG_LEVEL: debug
        published: "8000"
        published: "5678"
```

```bash
docker compose -f compose.yaml config | grep -E 'target|LOG_LEVEL|published'
```

```
      target: final
      LOG_LEVEL: info
```

### Watch in action

```bash
docker compose up --watch
```

```
[+] Running 2/2
 ✔ Network proj_default  Created
 ✔ Container proj-web-1  Started
        ⦿ Watch enabled
web-1  |  * Running on http://0.0.0.0:8000
...  (you edit src/app.py) ...
        ⦿ Syncing service "web" after 1 changes were detected
web-1  |  * Detected change in '/app/src/app.py', reloading
web-1  |  * Restarting with stat
...  (you edit requirements.txt) ...
        ⦿ Rebuilding service(s) ["web"] after changes were detected...
[+] Building 6.3s (12/12) FINISHED
        ⦿ service(s) ["web"] successfully built
[+] Running 1/1
 ✔ Container proj-web-1  Recreated
```

### Remove a base setting in an override

```yaml
# compose.ci.yaml
services:
  web:
    ports: !reset []          # CI doesn't need published ports
    restart: "no"
```

### `--renew-anon-volumes` after dependency changes (bind-mount approach)

```bash
docker compose up -d --build -V
```

---

## Hands-on lab

1. Take the `counter` project from topic 35. Add a `develop.watch` section that syncs `app.py` (with `flask --debug` as the command) and rebuilds on `requirements.txt`. Run `docker compose up --watch`, edit the greeting text, refresh the browser.
2. Switch to the bind-mount approach for comparison (`volumes: - ./:/app`). Create a file from inside the container (`docker compose exec web touch /app/created-in-container`) and check its owner on the host (`ls -ln`). Fix with `user: "${UID}:${GID}"`.
3. **node_modules trap** (if you know Node): make a tiny Express app, bind-mount `./:/app` without the anonymous volume, delete your host `node_modules`, and see `Cannot find module 'express'`. Add `- /app/node_modules` and recreate.
4. Split into `compose.yaml` (production-like) + `compose.override.yaml` (dev). Use `docker compose config` with and without `-f` to see both results. Use `!reset` to drop ports in a CI file.
5. Move redis into `infra/compose.yaml` and `include:` it. Confirm `web` still reaches `cache` by name.
6. Clean up: `docker compose down -v`

---

## Where you'll use it

- **Team onboarding:** `git clone && docker compose up --watch` = a working dev environment in minutes.
- **Monorepos and platform teams:** `include` lets each team own its own compose file.
- **One file, many environments:** base + override + prod/CI files, checked with `docker compose config` in CI.
- **Dev/prod parity:** the same Dockerfile with `dev` and `final` stages.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Cannot find module …` / `invalid ELF header` / `ModuleNotFoundError` after adding a bind mount | The mount covers the image's installed dependencies | Anonymous volume for `node_modules`/`.venv`, or switch to `watch` |
| Code changes not picked up (bind mount) | No hot reload, or inotify events don't cross the mount (`/mnt/c`, some VMs) | Enable the framework's reload; keep files in the WSL filesystem; polling mode |
| `watch` prints nothing on changes | The `path` doesn't match where you edit, or the files are excluded by `.dockerignore` (watch respects it) | Check paths; check `.dockerignore` |
| `watch` sync: `permission denied` in the container | The target folder isn't writable by the container's user | `COPY --chown` the folder to the app user in the dev stage |
| Root-owned files appear in your project | Container wrote to the bind mount as root | `user: "${UID}:${GID}"` |
| Prod run picked up dev settings | `compose.override.yaml` was loaded automatically | In prod, always pass explicit `-f` files |
| A port or volume from the base file can't be removed in the override | Lists are merged, not replaced | `!reset` or `!override` |
| `include` path's relative bind mounts point to wrong folders | Expected: they resolve relative to the included file | Keep them relative to that file's folder |
| Changes in dependencies don't apply (bind-mount approach) | Old anonymous volume still has old `node_modules` | `docker compose up -d --build -V` |

---

## Quick recap

- **Bind mount + hot reload** is simple, but watch out for `node_modules`/`.venv` being covered, root-owned files, and slow cross-OS mounts.
- **`docker compose watch`** (`develop.watch`): `sync` code, `rebuild` on dependency changes, `sync+restart` for configs. No mount problems. The recommended dev loop.
- `build:` supports context, dockerfile, **target**, args, secrets, ssh, cache, platforms. One Dockerfile with `dev` and `final` stages.
- `compose.override.yaml` is merged automatically. `-f` picks files explicitly. Lists merge, scalars replace, `!reset`/`!override` change that. Verify with `docker compose config`.
- `include` composes whole files (each with its own paths and `.env`). `extends` reuses one service definition.

**Interview questions**

1. **Q: How do you get fast code reloads with Docker Compose in development?**
   A: Use `docker compose watch` with `develop.watch` rules: `sync` source files into the running container (where the framework's hot reload restarts the app) and `rebuild` when dependency manifests change. Or bind-mount the source and use the framework's reloader.

2. **Q: Why might bind-mounting your project folder break a Node or Python app in a container?**
   A: The mount hides the dependencies installed in the image under the same path (e.g. `/app/node_modules`), replacing them with the host's (missing or built for a different OS). Fix it with an anonymous volume over that sub-path, or use watch/sync instead.

3. **Q: How do Compose override files work?**
   A: Compose merges several files in order: `compose.yaml` plus `compose.override.yaml` by default, or the files you pass with `-f`. Later files replace single values, merge maps and append to most lists (`!reset`/`!override` change this). `docker compose config` shows the merged result.
