# 14 · Dockerfile Basics

> **Level:** ★ foundation
>
> **You will learn:** how to write a Dockerfile and build an image from it: the build context, `.dockerignore`, the instructions `FROM`, `RUN`, `COPY`, `ADD`, `WORKDIR`, `CMD`, `EXPOSE` (plus a first look at `ENV` and `USER`), the `# syntax=` line, heredocs, how to read build output, `docker init`, and why `docker commit` is not how images should be made.
>
> **You need:** topic 07 (layers), topic 12 (exec form vs shell form, PID 1).

---

## What it is

A **Dockerfile** is a plain text file with step-by-step instructions to build an image. `docker build` reads it and produces an image.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

Each line is an **instruction** (written in CAPITALS by convention) followed by arguments. Instructions run top to bottom. Instructions that change files (`RUN`, `COPY`, `ADD`) create a **layer**. The others (`CMD`, `EXPOSE`, `ENV`, `WORKDIR`…) only change the image **config** (topic 07).

---

## Why it exists

You could make an image by starting a container, installing things by hand, and saving it (`docker commit`). But then:
- nobody knows exactly what was done,
- you can't repeat it for the next security patch,
- you can't review it in a pull request.

A Dockerfile is **the image as code**: versioned in git, reviewed, repeatable by anyone, rebuilt automatically in CI. It replaces the "server setup checklist" that drifted over time (topic 01).

---

## Analogy

A **recipe card** vs **a photo of a finished cake**. `docker commit` gives you the photo: you have the cake, but no idea how to make another one, or how to make it with less sugar next time. The Dockerfile is the recipe: anyone can bake the exact same cake again, and you can change one step and bake a new version.

---

## How it works inside

### The build context

When you run `docker build -t myapp .`, the final `.` is the **build context**: the folder whose files the build is allowed to use.

1. The CLI sends the Dockerfile and (as needed) files from the context to the builder, **BuildKit**, which runs inside dockerd.
2. `COPY` and `ADD` can only copy files **from the context**. `COPY ../secret.txt .` fails: you can't reach outside it.
3. BuildKit only transfers the files the build actually uses, and only the ones that changed since last time. (The old builder sent the **whole folder** every time. That's one reason it's deprecated.)

> **BuildKit** = Docker's build engine (the default since Docker 23). It runs build steps in parallel when possible, has a smart cache, and supports secrets and cache mounts. Topic 16 covers it in depth. The **legacy builder** (used when `DOCKER_BUILDKIT=0`) is **deprecated**.

### `.dockerignore`

A file named `.dockerignore` in the context root lists files to **exclude** from the context, using patterns like `.gitignore`:

```
.git
node_modules
__pycache__/
*.pyc
.env
*.log
Dockerfile*
docker-compose*.yml
compose*.yaml
```

Why it matters:
- **Speed**: no sending huge folders (`.git`, `node_modules`) to the builder.
- **Correctness**: a `COPY . .` won't overwrite the clean `node_modules` you installed in the image with the one from your laptop (built for a different OS).
- **Security**: `.env` files and keys don't end up inside the image.

### What each instruction does

| Instruction | What it does | Creates a layer? |
|---|---|---|
| `FROM image` | Start from this base image. Must be the first instruction (after `ARG`/comments). `FROM scratch` = start empty. | Uses the base's layers |
| `RUN command` | Runs a command **during the build** in a temporary container, and saves the filesystem changes as a new layer | Yes |
| `COPY src dest` | Copies files from the build context into the image | Yes |
| `ADD src dest` | Like `COPY`, plus: **auto-extracts** local `.tar` files, and can download **URLs** and **git repos** | Yes |
| `WORKDIR /path` | Sets the current folder for later `RUN`, `COPY`, `CMD`, and for the container at runtime. Creates it if missing | Config (+ an empty folder) |
| `CMD [...]` | The **default command** when a container starts (can be overridden by `docker run image <cmd>`) | Config only |
| `ENTRYPOINT [...]` | A fixed program that always runs. CMD becomes its arguments (topic 15) | Config only |
| `EXPOSE 8000` | **Documents** that the app listens on port 8000. Does **not** open or publish anything (topic 28) | Config only |
| `ENV KEY=value` | Sets an environment variable for later build steps **and** at runtime (topic 18) | Config only |
| `ARG NAME` | A build-time variable (topic 18) | — |
| `USER name` | Run later steps and the container as this user (topic 20) | Config only |
| `LABEL k=v` | Adds metadata (topic 18) | Config only |
| `VOLUME /data` | Marks a folder as a volume (topic 24) | Config only |
| `HEALTHCHECK` | How to test if the app is healthy (topic 32) | Config only |
| `STOPSIGNAL`, `SHELL` | Topic 15 | Config only |

### What `RUN` really does

For every `RUN`, BuildKit:
1. Takes the filesystem so far (all layers up to now).
2. Starts a temporary container on it.
3. Runs the command: shell form `RUN apt-get update` runs as `/bin/sh -c "apt-get update"`; exec form `RUN ["apt-get","update"]` runs directly.
4. Saves **only the differences** (the container's upper layer, topic 07) as a new layer.
5. Throws the temporary container away.

Consequences:
- **Processes don't survive between `RUN` steps.** `RUN service postgres start` does nothing useful for the next step or the final container.
- **`cd` doesn't carry over** between `RUN` lines. Use `WORKDIR`.
- **Environment variables you `export` inside a `RUN` are gone after it.** Use `ENV`.
- **Files deleted in a later `RUN` still take space** in the earlier layer (topic 07). Clean up in the **same** `RUN`:

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

### `COPY` vs `ADD`

**Use `COPY`** unless you specifically need `ADD`'s extra features. `ADD`'s magic (auto-extracting tar files) surprises people. Valid uses of `ADD`:

```dockerfile
ADD rootfs.tar.gz /                                            # auto-extracts into /
ADD --checksum=sha256:24454f8…  https://example.com/tool.tar.gz /tmp/   # download, verified by hash
ADD https://github.com/acme/repo.git#v1.2.0 /src               # clone a git ref (BuildKit)
```

### The `# syntax=` line

```dockerfile
# syntax=docker/dockerfile:1
```

- Must be the **very first line**.
- Tells BuildKit to use the latest stable **Dockerfile frontend** version 1.x (the part of BuildKit that understands Dockerfile syntax). It's pulled as an image (`docker/dockerfile:1`).
- Gives you the newest Dockerfile features (heredocs, `--mount`, `--chmod`, `--link`) even on an older Docker Engine. Recommended for every Dockerfile.

### Heredocs: multi-line scripts inline

With the syntax line above, you can write multi-line `RUN` and `COPY` content:

```dockerfile
RUN <<EOF
set -e
apt-get update
apt-get install -y --no-install-recommends curl
rm -rf /var/lib/apt/lists/*
EOF

COPY <<EOF /app/config.yaml
port: 8000
log_level: info
EOF
```

- `<<EOF ... EOF` = everything between the markers is one script (or one file).
- Cleaner than long chains of `&& \`. Add `set -e` so the step fails if any line fails.

### Why not `docker commit`

`docker commit <container> myimage` saves a container's writable layer as a new image layer. It exists, it works, and you should **not** use it to make images:
- Not reproducible: there's no record of what was done.
- It captures junk: caches, temp files, shell history, maybe secrets you typed.
- It can't be rebuilt automatically when the base image gets a security fix.

Acceptable use: **forensics**, freezing a broken container's state to investigate later.

---

## Commands / files

### A complete small project

```
hello-flask/
├── .dockerignore
├── Dockerfile
├── app.py
└── requirements.txt
```

`app.py`:

```python
from flask import Flask
import os, socket

app = Flask(__name__)

@app.get("/")
def hello():
    return f"Hello from {socket.gethostname()} (env={os.getenv('APP_ENV', 'dev')})\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

- `host="0.0.0.0"` = listen on **all** network interfaces in the container. If an app listens on `127.0.0.1` only, it's unreachable from outside the container (topic 28/30).

`requirements.txt`:

```
flask==3.1.2
```

`.dockerignore`:

```
.git
__pycache__/
*.pyc
.env
.venv
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim

# All later paths are relative to /app
WORKDIR /app

# Dependencies first (changes rarely → cached; see topic 16)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then the code (changes often)
COPY app.py .

ENV APP_ENV=production
EXPOSE 8000

# Exec form → python is PID 1 and receives SIGTERM (topic 12)
CMD ["python", "app.py"]
```

### Build it

```bash
cd hello-flask
docker build -t hello-flask:1.0 .
```

```
[+] Building 14.2s (11/11) FINISHED                                         docker:default
 => [internal] load build definition from Dockerfile                                  0.0s
 => => transferring dockerfile: 421B                                                  0.0s
 => resolve image config for docker-image://docker.io/docker/dockerfile:1             1.0s
 => docker-image://docker.io/docker/dockerfile:1@sha256:…                             0.8s
 => [internal] load metadata for docker.io/library/python:3.13-slim                   1.1s
 => [internal] load .dockerignore                                                     0.0s
 => => transferring context: 72B                                                      0.0s
 => [1/5] FROM docker.io/library/python:3.13-slim@sha256:…                            3.4s
 => [internal] load build context                                                     0.0s
 => => transferring context: 512B                                                     0.0s
 => [2/5] WORKDIR /app                                                                0.1s
 => [3/5] COPY requirements.txt .                                                     0.0s
 => [4/5] RUN pip install --no-cache-dir -r requirements.txt                          6.9s
 => [5/5] COPY app.py .                                                               0.0s
 => exporting to image                                                                0.8s
 => => exporting layers                                                               0.6s
 => => exporting manifest sha256:…                                                    0.0s
 => => exporting config sha256:…                                                      0.0s
 => => naming to docker.io/library/hello-flask:1.0                                    0.0s
```

How to read it:
- `docker:default` (top right) = which **builder** is used (topic 21).
- `load build definition` = reading the Dockerfile.
- `resolve image config for docker/dockerfile:1` = the `# syntax=` line: fetching the Dockerfile frontend.
- `load metadata for python:3.13-slim` = asking the registry what the tag points to.
- `load .dockerignore` / `load build context` → `transferring context: 512B` = only 512 bytes sent, thanks to `.dockerignore`.
- `[1/5] … [5/5]` = the steps that touch the filesystem, with their time. Slow steps stand out (here `pip install`, 6.9 s).
- `exporting to image` = writing layers, manifest and config, then tagging.

Common `docker build` options:

```bash
docker build -t myapp:1.0 -t myapp:latest .     # several tags at once
docker build -f docker/Dockerfile.prod .        # Dockerfile in another place
docker build --progress=plain .                 # full output of every step (for debugging)
docker build --no-cache .                       # ignore the cache, rebuild every step
docker build --pull .                           # always check for a newer base image
```

### Run it

```bash
docker run -d --name hf -p 8000:8000 hello-flask:1.0
curl -s localhost:8000
docker history hello-flask:1.0 | head -8
```

```
Hello from 3c1a2b9e8d7f (env=production)
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
5f2e…          30 seconds ago   CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
<missing>      30 seconds ago   EXPOSE map[8000/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      30 seconds ago   ENV APP_ENV=production                          0B        buildkit.dockerfile.v0
<missing>      30 seconds ago   COPY app.py . # buildkit                        8.19kB    buildkit.dockerfile.v0
<missing>      31 seconds ago   RUN /bin/sh -c pip install --no-cache-dir -r…   14.2MB    buildkit.dockerfile.v0
<missing>      38 seconds ago   COPY requirements.txt . # buildkit              4.1kB     buildkit.dockerfile.v0
<missing>      38 seconds ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
```

- `-p 8000:8000` = forward host port 8000 to container port 8000 (fully explained in topic 28).
- Your Dockerfile's steps are on top of the base image's history.

### `docker init`: generate a starting Dockerfile

```bash
cd my-node-project
docker init
```

```
Welcome to the Docker Init CLI!

This utility will walk you through creating the following files with sensible defaults for your project:
  - .dockerignore
  - Dockerfile
  - compose.yaml
  - README.Docker.md

? What application platform does your project use? Node
? What version of Node do you want to use? 22
? Which package manager do you want to use? npm
? What command do you want to use to start the app? node server.js
? What port does your server listen on? 3000

✔ Created → .dockerignore
✔ Created → Dockerfile
✔ Created → compose.yaml
✔ Created → README.Docker.md
```

- Detects Go, Python, Node, Rust, ASP.NET, PHP, Java and creates a decent starting point (multi-stage, non-root user, cache mounts). **Read and understand what it generated** before you use it. After topic 20 you'll be able to judge it.

### `docker commit` (for forensics only)

```bash
docker commit -m "snapshot of broken state" hf hf-forensics:2026-11-17
```

---

## Hands-on lab

1. Build and run the `hello-flask` project above. Visit `http://localhost:8000` from your Windows browser.
2. **Context and `.dockerignore`:** create a big junk file and see the context size:
   ```bash
   head -c 200000000 /dev/urandom > junk.bin
   docker build --progress=plain -t hello-flask:1.0 . 2>&1 | grep 'transferring context'
   echo 'junk.bin' >> .dockerignore
   docker build --progress=plain -t hello-flask:1.0 . 2>&1 | grep 'transferring context'
   rm junk.bin
   ```
   (BuildKit only sends files the build needs. A `COPY . .` would need **everything**, so add `COPY . .` to the Dockerfile and try again to see the 200 MB transfer.)
3. **Processes don't survive `RUN`:** add `RUN cd /tmp && export FOO=bar` and then `RUN pwd && echo "FOO=$FOO"`. Build with `--progress=plain` and read the output: `/app` and `FOO=`.
4. **Heredoc:** add a `COPY <<EOF /app/build-info.txt` with some text, rebuild, and `docker run --rm hello-flask:1.0 cat /app/build-info.txt`.
5. **See why not `commit`:**
   ```bash
   docker run -it --name manual python:3.13-slim bash
   # inside: pip install flask; echo "mysecretpassword" > /root/.pgpass; exit
   docker commit manual manual-image
   docker run --rm manual-image cat /root/.pgpass
   ```
   The "secret" is baked into the image forever, and nothing records how the image was made.
6. Clean up: `docker rm -f hf manual; docker rmi manual-image`

---

## Where you'll use it

- **Every app you deploy** needs a Dockerfile, usually kept in the app's git repository.
- **Code reviews:** you'll review Dockerfiles for size, security and cache efficiency (topics 16–20).
- **CI:** pipelines run `docker build` on every commit (topic 46).
- **Base images for teams:** a company "golden" base image built from its own Dockerfile.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory` | No `Dockerfile` in the context folder | Use `-f path/to/Dockerfile`, or run from the right folder |
| `failed to compute cache key: failed to calculate checksum of ref …: "/requirements.txt": not found` | `COPY` source isn't in the context, or is excluded by `.dockerignore` | Check the path relative to the context; check `.dockerignore` |
| `COPY failed: forbidden path outside the build context: ../x` | You tried to copy from outside the context | Move the file in, or make the context the parent folder (`docker build -f app/Dockerfile .`) |
| `process "/bin/sh -c pip install ..." did not complete successfully: exit code: 1` | A `RUN` command failed | Scroll up for the real error. Use `--progress=plain` to see full output |
| `E: Unable to locate package curl` | `apt-get install` without `apt-get update` in the **same** `RUN`, or a stale cached `update` layer | `RUN apt-get update && apt-get install -y ...` in one step |
| `/bin/sh: 1: source: not found` | `RUN` uses `/bin/sh` (dash on Debian), which lacks bash features | Use `.` instead of `source`, or `SHELL ["/bin/bash","-c"]` (topic 15) |
| Container can't be reached on its port even with `-p` | The app listens on `127.0.0.1` inside the container | Listen on `0.0.0.0` |
| Build context is huge and slow | No `.dockerignore`, and `COPY . .` | Add `.dockerignore` |
| `the --chmod option requires BuildKit` / unknown flag in `RUN --mount` | Old builder, or no `# syntax` line on an older engine | Use BuildKit (the default) and add `# syntax=docker/dockerfile:1` |

---

## Quick recap

- A Dockerfile is the image as code. `docker build -t name:tag <context>` builds it with **BuildKit**.
- The **build context** is the only place `COPY`/`ADD` can read from. `.dockerignore` keeps junk and secrets out.
- `RUN`, `COPY`, `ADD` create layers. `CMD`, `EXPOSE`, `ENV`, `WORKDIR`, `USER` only change config. Each `RUN` is a fresh temporary container: no processes, `cd` or `export` carry over.
- Prefer `COPY` over `ADD`. Use exec-form `CMD`. Start with `# syntax=docker/dockerfile:1`.
- `docker commit` isn't reproducible. Keep it for forensics.

**Interview questions**

1. **Q: What is the build context, and why does `.dockerignore` matter?**
   A: The build context is the folder given to `docker build`. It's the only source for `COPY`/`ADD`. `.dockerignore` excludes files from it, which speeds up builds, avoids overwriting things like `node_modules`, and keeps secrets such as `.env` out of the image.

2. **Q: What's the difference between COPY and ADD?**
   A: Both copy files into the image. `ADD` also auto-extracts local tar archives and can fetch URLs and git repositories. Use `COPY` unless you need those features, because `ADD`'s implicit behaviour can surprise you.

3. **Q: Why is `EXPOSE 8080` not enough to reach the app?**
   A: `EXPOSE` only documents the port in the image metadata. To reach the app from outside, you must publish the port with `-p host:container` (or connect from another container on the same network).
