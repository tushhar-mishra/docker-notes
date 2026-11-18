# 36 · Compose: Networks, Volumes, Environment, Secrets and Profiles

> **Level:** ★★ intermediate
>
> **You will learn:** the two different jobs of `.env` vs `env_file` (the most confusing part of Compose), variable interpolation (`${VAR:-default}`, `${VAR:?error}`, `$$`) and its precedence rules, multiple and internal networks, aliases, external networks and volumes, the long volume syntax and NFS volumes, `secrets` and `configs` (and what they really are underneath), `profiles` for optional services, and `x-` extension fields with YAML anchors to avoid repetition.
>
> **You need:** topic 35 (Compose basics), topics 24–29 (volumes, networks, DNS).

---

## What it is

The parts of a `compose.yaml` that wire services to **data**, to **each other**, and to **configuration**:

```yaml
services:
  api:
    image: ghcr.io/acme/api:${API_TAG:-latest}      # interpolation from .env / shell
    env_file: [api.env]                               # variables INTO the container
    environment:
      DB_HOST: db
    networks: [frontend, backend]
    secrets: [db_password]
    configs:
      - source: api_config
        target: /etc/api/config.yaml
  db:
    image: postgres:17-alpine
    networks: [backend]
    volumes: [pgdata:/var/lib/postgresql/data]
    secrets: [db_password]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
  adminer:
    image: adminer
    profiles: [debug]                                 # only started when asked
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true                                    # no internet access

volumes:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt

configs:
  api_config:
    file: ./config/api.yaml
```

---

## Why it exists

- One compose file should work in **many environments** (laptop, CI, staging), with differences coming from variables, not copies of the file.
- Services need **separate networks** for security (the DB shouldn't be reachable from the edge).
- Passwords shouldn't sit in the YAML or in environment variables visible to `docker inspect`.
- Optional tools (DB admin UIs, debuggers) shouldn't start by default.

---

## Analogy

A **wedding seating plan**:
- **Interpolation** (`${VAR}`) = the plan template with blanks: "Table for `${FAMILY}`", filled in per wedding.
- **env_file / environment** = the note **placed on each table** for the guests (variables the container itself sees).
- **Networks** = which rooms each table is in. Guests can only talk to tables in their room. `internal` = a room with no door to the street.
- **Secrets** = sealed envelopes handed only to the tables that need them, never written on the plan.
- **Profiles** = "the photo booth is only set up if someone asks".

---

## How it works inside

### `.env` vs `env_file`: two different things

| | `.env` (in the project folder) | `env_file:` (per service) |
|---|---|---|
| Who reads it | **Compose itself**, when parsing the YAML | Compose, to pass into **the container** |
| Used for | **Interpolation**: filling `${VAR}` in compose.yaml | Environment variables **inside** the container |
| Reaches the container? | **No**, unless the YAML references it (`environment: X: ${X}`) | **Yes** |
| Name | Exactly `.env` by default (or `--env-file other.env` on the CLI) | Any file(s) you list |

This is the #1 Compose confusion: putting `DB_PASSWORD=...` in `.env` and expecting the container to see it. It won't, unless you map it.

### Interpolation rules

| Syntax | Result |
|---|---|
| `${VAR}` / `$VAR` | Value, or **empty string + warning** if unset |
| `${VAR:-default}` | Value, or `default` if unset **or empty** |
| `${VAR-default}` | Value, or `default` only if **unset** |
| `${VAR:?message}` | **Error** with `message` if unset or empty (good for required settings) |
| `${VAR:+other}` | `other` if VAR is set and non-empty, else empty |
| `$$` | A literal `$` (so the **container's** shell can expand it later) |

**Precedence for interpolation values** (highest first):
1. Variables in **your shell** environment,
2. `--env-file` file(s) given on the CLI (or `.env` in the project directory by default).

So `API_TAG=1.5.0 docker compose up -d` overrides `.env`. (CI pipelines rely on this.)

**Precedence for a container's environment** (highest first):
1. `docker compose run -e`,
2. `environment:` in the service,
3. `env_file:` files (later files override earlier ones),
4. `ENV` in the image's Dockerfile.

### Networks

- Without `networks:`, every service joins `<project>_default`.
- Declare several networks to segment (topic 27): a service on `[frontend, backend]` gets one interface per network.
- `internal: true` = Docker creates the bridge **without NAT/external route**: containers on it can talk to each other but **can't reach the internet** (good for databases).
- `aliases` = extra DNS names on a network:
  ```yaml
  networks:
    backend:
      aliases: [database, pg]
  ```
- `external: true` = use a network created **outside** this project (e.g. a shared `proxy` network used by a reverse proxy and many projects). Compose won't create or delete it:
  ```yaml
  networks:
    proxy:
      external: true
      name: shared-proxy
  ```
- Driver options: `driver_opts: { com.docker.network.driver.mtu: 1400 }`, and fixed subnets with `ipam:`.

### Volumes

Short and long syntax:

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data              # named volume
      - ./init:/docker-entrypoint-initdb.d:ro        # bind mount (relative to the compose file!)
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 67108864
      - type: volume
        source: pgdata
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
```

- Relative host paths (`./init`) are resolved **relative to the compose file's folder**, not your current directory.
- Top-level `volumes:` declares named volumes. Compose prefixes them: `<project>_pgdata`. To use an exact name, add `name: pgdata`. To use one created outside: `external: true`.
- NFS through the local driver:
  ```yaml
  volumes:
    shared:
      driver: local
      driver_opts:
        type: nfs
        o: addr=10.0.0.20,rw,nfsvers=4.1
        device: ":/exports/shared"
  ```

### Secrets and configs: what they really are

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt      # from a file on the host
  api_key:
    environment: API_KEY                 # from a variable in Compose's environment
configs:
  nginx_conf:
    file: ./nginx.conf
```

In standalone Compose (not Swarm):
- **A secret is a read-only bind mount** of the source file into the container at **`/run/secrets/<name>`** (or a custom `target`).
- **A config is a read-only bind mount** at `/<name>` (or a custom `target`).
- The difference from putting the value in `environment:`: the value **isn't** in `docker inspect`'s `Config.Env`, isn't inherited by every child process, and isn't printed by accident in debug dumps. The app reads a file.
- Many official images support `*_FILE` variables for this (`POSTGRES_PASSWORD_FILE`, `MYSQL_ROOT_PASSWORD_FILE`).
- Limitation: the file is still a plain file on the host, protected only by filesystem permissions. It's a big step up from env vars, but not a vault (topic 44). In **Swarm**, secrets are encrypted in the cluster store and kept on an in-memory filesystem in containers.

### Profiles

```yaml
services:
  adminer:
    image: adminer
    profiles: [debug]
  loadtest:
    image: grafana/k6
    profiles: [test]
```

- Services **without** `profiles` always start. Services **with** profiles start only when a profile is enabled:
  ```bash
  docker compose --profile debug up -d
  COMPOSE_PROFILES=debug,test docker compose up -d
  ```
- Naming a profiled service directly also starts it: `docker compose run --rm loadtest`.

### Avoid repetition: `x-` fields and YAML anchors

```yaml
x-common: &common
  restart: unless-stopped
  logging:
    driver: local
    options:
      max-size: "20m"
  deploy:
    resources:
      limits:
        memory: 512m

services:
  api:
    <<: *common
    image: ghcr.io/acme/api:1.4.2
  worker:
    <<: *common
    image: ghcr.io/acme/worker:1.4.2
```

- Top-level keys starting with `x-` are **ignored** by Compose: a place to define reusable blocks.
- `&common` = a YAML **anchor** (a name for this block). `*common` = a reference to it. `<<:` = merge its keys into this map.

---

## Commands / files

### See interpolation results

`.env`:

```
API_TAG=1.4.2
PUBLIC_PORT=8080
```

`compose.yaml` fragment:

```yaml
services:
  api:
    image: ghcr.io/acme/api:${API_TAG:?API_TAG must be set}
    ports:
      - "${PUBLIC_PORT:-8000}:8000"
    environment:
      APP_VERSION: ${API_TAG}
      GREETING: "price is $$5"
```

```bash
docker compose config --format yaml | grep -E 'image:|published|APP_VERSION|GREETING'
API_TAG=2.0.0 docker compose config | grep image:
```

```
    image: ghcr.io/acme/api:1.4.2
        published: "8080"
      APP_VERSION: 1.4.2
      GREETING: price is $5
    image: ghcr.io/acme/api:2.0.0
```

- The shell variable `API_TAG=2.0.0` beat `.env`.
- `$$5` became a literal `$5`.

Unset a required variable:

```bash
mv .env .env.bak; docker compose config; mv .env.bak .env
```

```
error while interpolating services.api.image: required variable API_TAG is missing a value: API_TAG must be set
```

### See what the container actually gets

```bash
docker compose run --rm api env | sort | grep -E 'APP_|DB_|PUBLIC'
```

- `PUBLIC_PORT` is **not** inside: it only lived in `.env` for interpolation.

### Secrets, from inside

```bash
mkdir -p secrets && printf 's3cret-pw' > secrets/db_password.txt && chmod 600 secrets/db_password.txt
docker compose up -d db
docker compose exec db sh -c 'ls -l /run/secrets; cat /run/secrets/db_password; echo'
docker inspect $(docker compose ps -q db) -f '{{json .Config.Env}}' | tr ',' '\n' | grep -i pass
docker inspect $(docker compose ps -q db) -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

```
total 4
-rw------- 1 1000 1000 9 Nov 17 15:30 db_password
s3cret-pw
"POSTGRES_PASSWORD_FILE=/run/secrets/db_password"
/home/tushar/proj/secrets/db_password.txt -> /run/secrets/db_password
...
```

- The env only has the **path**, not the password.
- `Mounts` shows it's a bind mount of your file.
- Owner `1000` = your host user. If the container's user can't read a `600` file owned by someone else, you'll get permission denied (topic 25). Compose's long syntax supports `uid`, `gid` and `mode` for secrets, but in standalone (non-Swarm) mode those may be ignored, so set host file permissions so the container user can read it.

### Internal network check

```bash
docker compose exec db sh -c 'wget -q -T 3 -O /dev/null https://example.com && echo online || echo "no internet"'
```

```
no internet
```

### Profiles

```bash
docker compose up -d
docker compose ps --services
docker compose --profile debug up -d
docker compose ps --services
```

```
api
db
adminer
api
db
```

---

## Hands-on lab

1. Build a project with `api` (any small image, e.g. `nginx:1.27-alpine` as a stand-in), `db` (postgres) and `adminer` (profile `debug`), using the full example from "What it is".
2. Prove `.env` doesn't reach containers: put `SECRET_THING=x` in `.env`, run `docker compose run --rm api env | grep SECRET_THING` (nothing). Then add `environment: SECRET_THING: ${SECRET_THING}` and repeat.
3. Prove precedence: set `API_TAG` in `.env`, then override it from the shell, and check with `docker compose config`.
4. Check the `backend` network has no internet, and that `api` (on frontend + backend) does.
5. Use secrets for the Postgres password with `POSTGRES_PASSWORD_FILE`. Show that `docker inspect` doesn't reveal the password.
6. Refactor repeated `restart`/`logging` blocks into an `x-common` anchor. Confirm with `docker compose config` that each service got them.
7. Clean up: `docker compose --profile debug down -v`

---

## Where you'll use it

- **Multi-environment deployments:** one compose file + per-environment `.env` (image tags, ports, hostnames).
- **Security:** internal networks for databases, secrets as files, `*_FILE` variables.
- **Shared infrastructure:** an external `proxy` network joined by many Compose projects behind Traefik/nginx.
- **Developer tooling:** profiles for admin UIs, mocks and load generators.
- **Clean, reviewable files:** anchors and `x-` fields for common logging/limits/restart settings.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `WARN[0000] The "DB_HOST" variable is not set. Defaulting to a blank string.` | `${DB_HOST}` used but not defined in shell or `.env` | Define it, or use `${DB_HOST:-default}` / `:?` |
| Container doesn't see a variable that's in `.env` | `.env` is only for interpolation | Map it in `environment:` or use `env_file:` |
| A literal `$` in a value disappears | Compose interpolated it | Use `$$` |
| Value has unexpected quotes | `env_file` quoting differs from shell rules | Check `docker compose config`, or remove the quotes |
| `network proxy declared as external, but could not be found` | External network doesn't exist yet | `docker network create shared-proxy` first |
| `service "api" refers to undefined volume pgdata` | Named volume not declared at top level | Add it under top-level `volumes:` |
| Secret file reads as permission denied inside | Host file mode/owner doesn't allow the container's UID | Adjust host file permissions |
| `adminer` never starts | It has a profile | `--profile debug` or `COMPOSE_PROFILES` |
| Bind mount points to the wrong folder | Relative paths resolve from the compose file's folder | Use paths relative to the compose file, or `${PWD}` |
| Service on an `internal` network can't call an external API | That's what internal means | Put that service on a non-internal network too |

---

## Quick recap

- **`.env` = values for `${VAR}` in the YAML** (shell variables win). **`env_file`/`environment` = variables inside the container** (`environment` wins over `env_file`).
- Use `${VAR:-default}` and `${VAR:?error}`. `$$` = a literal `$`. Check the result with `docker compose config`.
- Networks: several per service, `internal: true` for no internet, `aliases`, `external` for shared networks.
- Volumes: named (prefixed with the project), bind (relative to the compose file), tmpfs, NFS via `driver_opts`, `external`.
- `secrets` (→ `/run/secrets/<name>`) and `configs` are read-only bind mounts in standalone Compose: better than env vars, not a vault. `profiles` gate optional services. `x-` + anchors remove repetition.

**Interview questions**

1. **Q: What's the difference between the `.env` file and `env_file` in Compose?**
   A: `.env` supplies values for variable substitution inside compose.yaml and doesn't automatically reach containers. `env_file` lists files whose variables are injected into the container's environment.

2. **Q: How do you keep a database password out of `docker inspect` in Compose?**
   A: Define it as a Compose secret (from a file), which is mounted read-only at `/run/secrets/<name>`, and point the app at it (e.g. `POSTGRES_PASSWORD_FILE`). Only the file path appears in the environment.

3. **Q: How do you prevent a database container from reaching the internet while other services can still reach it?**
   A: Put it only on a network declared with `internal: true`, and attach the services that need it to that network too (plus a normal network for their own outbound traffic).
