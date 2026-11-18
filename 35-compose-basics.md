# 35 · Compose Basics

> **Level:** ★ foundation
>
> **You will learn:** what Docker Compose is (and the deprecated v1 you should ignore), how to describe a multi-container app in `compose.yaml`, the everyday commands (`up`, `down`, `ps`, `logs`, `exec`, `run`, `config`, `pull`, `build`), how **project names** work, and exactly what Compose creates underneath (networks, containers, volumes and the **labels** it uses to track them), including why a container is sometimes recreated and sometimes not.
>
> **You need:** Parts 2–6 (containers, images, builds, volumes, networks, limits, restart, logging).

---

## What it is

**Docker Compose** lets you define a whole application (several containers, their networks and volumes) in **one YAML file** (YAML = "YAML Ain't Markup Language", a human-readable data format where **indentation with spaces** shows structure, `key: value` makes a map and `- item` makes a list), and manage it with one command.

```yaml
# compose.yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      REDIS_HOST: cache
    depends_on:
      - cache
  cache:
    image: redis:7-alpine
```

```bash
docker compose up -d
```

That one command builds `web`, pulls `redis`, creates a network, and starts both containers with the right settings.

- **Compose v2** = `docker compose` (with a space): a Go **CLI plugin** (topic 04). It's the current one.
- **Compose v1** = `docker-compose` (with a hyphen): the old Python tool. **Deprecated and end-of-life since 2023.** Don't install or teach it. If a script uses `docker-compose`, switch it to `docker compose`.
- The file format follows the open **Compose Specification**. The top-level `version:` key is **obsolete**: Compose ignores it and prints a warning. Leave it out.
- The preferred file name is **`compose.yaml`** (also accepted: `compose.yml`, `docker-compose.yaml`, `docker-compose.yml`).

---

## Why it exists

Running a real app with plain `docker run` means long, error-prone command lines for every container, in the right order, with the right network, volumes and environment, and remembering them all. Compose:
- puts the whole setup **in a file** (version-controlled and reviewable),
- creates everything with `up` and removes it with `down`,
- **only changes what changed** when you edit the file and run `up` again,
- gives each app its own isolated network with DNS by **service name**.

---

## Analogy

A **film script** vs directing actors by shouting. With `docker run`, you shout each instruction to each actor every time. With Compose, you hand everyone the **script** (`compose.yaml`), say "action" (`up`), and to reshoot one scene you change only that page. The director (Compose) works out which actors need to redo their part.

---

## How it works inside

### Compose is just an API client

There's **no Compose daemon**. `docker compose up` reads the YAML and makes the same Docker API calls you'd make by hand (create network, create volume, pull/build image, create container, start container) against the current Docker context (topic 34).

### The project

Everything Compose creates belongs to a **project**. The project name:
1. `-p <name>` on the command line, or
2. `COMPOSE_PROJECT_NAME` environment variable, or
3. the top-level `name:` in the file, or
4. **the folder name** of the compose file (lowercased, cleaned up).

Two copies of the same app in different folders = two projects = no clash. The same folder name in two places = **same project** = they fight over the same containers.

### What `up` creates, and the names

For project `shop`:

| Object | Name |
|---|---|
| Default network | `shop_default` (a user-defined bridge: DNS by service name!) |
| Containers | `shop-web-1`, `shop-cache-1` (`<project>-<service>-<index>`) |
| Named volumes | `shop_<volume>` |
| Built images | `shop-web` (if you don't set `image:`) |

### Labels: how Compose remembers what it owns

Compose doesn't keep a state file. It **labels** every object and later **asks Docker** for objects with its labels:

```
com.docker.compose.project=shop
com.docker.compose.service=web
com.docker.compose.container-number=1
com.docker.compose.config-hash=7b1f3c…      ← hash of this service's config
com.docker.compose.project.config_files=/home/tushar/shop/compose.yaml
com.docker.compose.project.working_dir=/home/tushar/shop
com.docker.compose.oneoff=False
com.docker.compose.version=2.40.3
```

### Why some containers are recreated and others aren't

On each `up`, for each service Compose computes the **config hash** of the desired configuration (image, env, ports, volumes, command…) and compares it with the label on the running container:
- **Same hash, same image** → leave it running (`Running`).
- **Different** (you changed the YAML, or the image ID changed after a pull/build) → **recreate** it: stop, remove, create, start (`Recreated`).
- `--force-recreate` → recreate anyway. `--no-recreate` → never.

Recreated means a **new container**: its writable layer is gone (topic 23). Keep state in volumes.

### Service name = DNS name

On the project network, each service is resolvable by its **service name** (`cache`), via the embedded DNS (topic 29). With several replicas, the name returns all their IPs (round-robin).

### `depends_on` (first look)

`depends_on: [cache]` = start `cache` **before** `web`, and stop it after. It does **not** wait for cache to be **ready**. Topic 37 fixes that with health checks.

---

## Commands / files

### A complete small project

```
counter/
├── app.py
├── requirements.txt
├── Dockerfile
└── compose.yaml
```

`app.py`:

```python
import os, redis
from flask import Flask

app = Flask(__name__)
r = redis.Redis(host=os.getenv("REDIS_HOST", "localhost"), port=6379)

@app.get("/")
def hit():
    n = r.incr("hits")
    return f"Hello! This page has been seen {n} times.\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

`requirements.txt`:

```
flask==3.1.2
redis==6.4.0
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
ENV PYTHONUNBUFFERED=1
USER 10001
CMD ["python", "app.py"]
```

`compose.yaml`:

```yaml
name: counter

services:
  web:
    build: .
    image: counter-web:dev
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      REDIS_HOST: cache
    depends_on:
      - cache
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    restart: unless-stopped

volumes:
  redisdata:
```

Key parts:
- `services:` = one entry per container type. The key (`web`, `cache`) is the **service name** = DNS name.
- `build: .` = build from the Dockerfile in this folder. `image:` = the tag to give it (or, without `build`, the image to pull).
- `ports:` = like `-p`. **Quote them**: YAML reads some `xx:yy` values as base-60 numbers.
- `environment:` = like `-e`.
- `volumes:` under a service = mounts. The top-level `volumes:` declares named volumes (Compose creates `counter_redisdata`).
- `restart:` = restart policy (topic 32).

### up

```bash
cd counter
docker compose up -d
```

```
[+] Building 9.8s (10/10) FINISHED
 ...
[+] Running 4/4
 ✔ Network counter_default      Created     0.1s
 ✔ Volume "counter_redisdata"   Created     0.0s
 ✔ Container counter-cache-1    Started     0.4s
 ✔ Container counter-web-1      Started     0.6s
```

- `-d` = detached (like `docker run -d`). Without it, logs of all services stream in your terminal, and Ctrl+C stops them.
- Order: network → volume → `cache` (dependency) → `web`.

```bash
curl -s localhost:8000; curl -s localhost:8000
```

```
Hello! This page has been seen 1 times.
Hello! This page has been seen 2 times.
```

### ps, logs, top, images

```bash
docker compose ps
```

```
NAME              IMAGE             COMMAND                  SERVICE   CREATED          STATUS          PORTS
counter-cache-1   redis:7-alpine    "docker-entrypoint.s…"   cache     30 seconds ago   Up 29 seconds   6379/tcp
counter-web-1     counter-web:dev   "python app.py"          web       30 seconds ago   Up 29 seconds   127.0.0.1:8000->8000/tcp
```

```bash
docker compose logs -f --tail 20 web     # one service, follow
docker compose logs --since 5m           # all services, prefixed by service name
docker compose top
docker compose images
```

### exec and run

```bash
docker compose exec cache redis-cli GET hits
docker compose run --rm web python -c "import flask; print(flask.__version__)"
```

```
"2"
3.1.2
```

- `exec` = run a command in the **running** container of that service (like `docker exec`).
- `run` = start a **new one-off** container for that service (with its config, network and volumes), e.g. for migrations or admin scripts. `--rm` removes it afterwards. One-off containers are labelled `oneoff=True` and named like `counter-web-run-3f2a…`. By default `run` **doesn't** publish the service's ports (add `--service-ports` if you need them).

### config: see the final, resolved file

```bash
docker compose config
```

```yaml
name: counter
services:
  cache:
    image: redis:7-alpine
    networks:
      default: null
    restart: unless-stopped
    volumes:
      - type: volume
        source: redisdata
        target: /data
        volume: {}
  web:
    build:
      context: /home/tushar/counter
      dockerfile: Dockerfile
    depends_on:
      cache:
        condition: service_started
        required: true
    environment:
      REDIS_HOST: cache
    ...
networks:
  default:
    name: counter_default
volumes:
  redisdata:
    name: counter_redisdata
```

- Shows the file after **merging overrides, filling defaults and replacing variables** (topic 36), in the long syntax. Your first debugging step for "why isn't my setting applied?". `docker compose config --quiet` just validates.

### Edit and re-up: only what changed is recreated

Change `REDIS_HOST: cache` → add `LOG_LEVEL: debug` to `web`, then:

```bash
docker compose up -d
```

```
[+] Running 2/2
 ✔ Container counter-cache-1  Running     0.0s
 ✔ Container counter-web-1    Recreated   0.3s  (then Started)
```

- `cache` untouched. `web` recreated because its config hash changed.

```bash
docker inspect counter-web-1 --format '{{index .Config.Labels "com.docker.compose.config-hash"}}'
```

### pull, build, restart, stop, start

```bash
docker compose pull              # pull newer images for services with image: (not build-only ones)
docker compose build             # (re)build services with build:
docker compose up -d --build     # build, then up
docker compose restart web       # restart (doesn't apply YAML changes! use up -d for that)
docker compose stop / start      # stop/start without removing
```

### down

```bash
docker compose down
```

```
[+] Running 3/3
 ✔ Container counter-web-1      Removed
 ✔ Container counter-cache-1    Removed
 ✔ Network counter_default      Removed
```

- Removes containers and networks. **Keeps named volumes** (your Redis data survives).
- `docker compose down -v` also removes **named volumes declared in the file** → data loss. Be careful.
- `docker compose down --rmi local` also removes images built by Compose.

### See all projects on the host

```bash
docker compose ls
```

```
NAME      STATUS       CONFIG FILES
counter   running(2)   /home/tushar/counter/compose.yaml
```

### Compose's objects through plain Docker

```bash
docker ps --filter label=com.docker.compose.project=counter --format '{{.Names}}'
docker network ls --filter label=com.docker.compose.project=counter
```

---

## Hands-on lab

1. Build and run the `counter` project. Hit it a few times.
2. `docker compose down`, then `up -d` again. The counter continues (data in the named volume).
3. `docker compose down -v`, then `up -d`. The counter restarts at 1.
4. Edit only the `cache` service (add `command: ["redis-server", "--appendonly", "yes"]`) and `up -d`. Confirm only `cache` is recreated.
5. Copy the folder to `counter2` and remove the `name:` line from both copies. `up -d` in each: two independent projects (`counter`/`counter2`) with separate networks and volumes. Port 8000 will clash: change it in the copy.
6. `docker compose run --rm web env | grep REDIS` to see a one-off container's environment.
7. Run `docker compose config` and compare it with your file.
8. Clean up: `docker compose down -v` in both folders.

---

## Where you'll use it

- **Local development:** the whole stack (app, DB, cache, queue) with one command (topic 38).
- **CI:** spinning up dependencies for integration tests, then `down -v`.
- **Single-server production:** many small companies and internal tools run on Compose (topic 39).
- **Demos and reproductions:** a `compose.yaml` in a bug report reproduces the environment exactly.
- **Bridge to Kubernetes:** services ≈ Deployments, the project network ≈ a namespace, service names ≈ Services (topic 53).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `docker: 'compose' is not a docker command` | Compose plugin not installed | `sudo apt install docker-compose-plugin` |
| `docker-compose: command not found` (old scripts) | v1 isn't installed (and shouldn't be) | Use `docker compose` |
| `WARN[0000] /…/compose.yaml: the attribute 'version' is obsolete` | Old-style file with `version: "3.8"` | Delete the `version:` line |
| `no configuration file provided: not found` | No compose file in the current folder | `cd` there, or `-f path/compose.yaml` |
| `yaml: line 12: did not find expected key` | YAML indentation/syntax error | Spaces only (no tabs), consistent indentation; `docker compose config` |
| A setting seems ignored | Overridden by another file or variable (topic 36), or the container wasn't recreated | `docker compose config`; `up -d` (not `restart`) |
| Two projects interfere | Same project name (same folder name or `name:`) | Use `-p` or `name:` |
| Data lost after `down` | You used `down -v`, or the data wasn't on a named volume | Named volumes; avoid `-v` in production |
| `Bind for 0.0.0.0:8000 failed: port is already allocated` | Another project/container uses that host port | Change the published port |
| `web` crashes on start: can't connect to `cache` | `depends_on` doesn't wait for readiness | Health checks + `condition: service_healthy` (topic 37), and retry logic in the app |

---

## Quick recap

- `docker compose` (v2, a Go plugin) replaces the deprecated `docker-compose` (v1). No `version:` key. Use the file name `compose.yaml`.
- A **project** (named by `-p`, `COMPOSE_PROJECT_NAME`, `name:`, or the folder) groups networks, volumes and containers named `<project>-<service>-<n>`.
- Compose has no daemon or state file: it tracks objects with **labels** and recreates a container only when its **config hash** or image changes.
- Every project gets its own network where **service names are DNS names**.
- Daily commands: `up -d`, `ps`, `logs -f`, `exec`, `run --rm`, `config`, `pull`, `build`, `down` (keeps volumes; `-v` deletes them).

**Interview questions**

1. **Q: What is Docker Compose, and how does `docker compose up` work under the hood?**
   A: A CLI plugin that reads a declarative YAML description of services, networks and volumes, and calls the Docker API to create them. Everything is labelled with the project and service name, and on each `up` it compares a config hash to recreate only changed services.

2. **Q: What's the difference between `docker compose run` and `docker compose exec`?**
   A: `exec` runs a command inside an already-running service container. `run` creates a new one-off container from the service definition (same image, env, network and volumes), usually for tasks like migrations.

3. **Q: Does `docker compose down` delete data?**
   A: It removes containers and networks, but keeps named volumes. `down -v` also removes the volumes, deleting their data.
