# 37 · Compose: Dependencies, Health and Restarts

> **Level:** ★★ intermediate
>
> **You will learn:** the difference between **start order** and **readiness**, how `depends_on` works with `service_started`, `service_healthy` and `service_completed_successfully`, how to write `healthcheck:` blocks for common services, one-time init/migration services, `restart:` vs `deploy.restart_policy`, `docker compose up --wait`, `init: true`, stop behaviour (`stop_grace_period`, `stop_signal`), and why your app still needs retry logic.
>
> **You need:** topic 32 (restart policies, HEALTHCHECK), topic 35–36 (Compose).

---

## What it is

- **`depends_on`** = "start that service before this one" (and stop it after). With a **condition**, Compose waits until the dependency is **healthy** or has **finished successfully**, not just started.
- **`healthcheck:`** = the Compose version of `HEALTHCHECK` (topic 32).
- **`restart:`** = the restart policy for the service's containers.

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_PASSWORD: pw
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  migrate:
    image: ghcr.io/acme/api:1.4.2
    command: ["./manage", "migrate"]
    depends_on:
      db:
        condition: service_healthy

  api:
    image: ghcr.io/acme/api:1.4.2
    depends_on:
      db:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    restart: unless-stopped
```

---

## Why it exists

"**Started**" means the process exists. "**Ready**" means it accepts connections and works. A Postgres container starts in milliseconds, but takes seconds to initialise its data folder before accepting connections. If the app starts at the same moment and tries to connect once, it crashes.

Compose's plain `depends_on: [db]` only guarantees **order**, so this race happens all the time. Health-based conditions fix the **startup** race. Retry logic in the app fixes the **runtime** case (the database restarts later, the network blips).

---

## Analogy

Opening a **restaurant**:
- **Start order** = the chef arrives before the waiters.
- **Readiness** = the ovens are **hot** and the kitchen is **prepped**. A waiter who takes orders the second the chef walks in gets "not ready yet" for every dish.
- **`service_healthy`** = the waiters wait until the head chef says "kitchen ready".
- **`service_completed_successfully`** = the morning delivery (migration) must be **unpacked and checked** before anyone cooks.
- **Retry logic** = a waiter who, if the kitchen is briefly busy, waits a moment and asks again, instead of quitting.

---

## How it works inside

### `depends_on` conditions

| Condition | Compose waits until… | Use for |
|---|---|---|
| `service_started` (the default for the short list form) | The dependency's container has **started** | Order only |
| `service_healthy` | The dependency's health status is **`healthy`** | Databases, queues, anything with a health check |
| `service_completed_successfully` | The dependency **exited with code 0** | Migrations, init jobs, seeding, permission fixes |

Other options:
- `restart: true`: when Compose **restarts** the dependency (e.g. `docker compose restart db`, or a recreate), it also restarts this service.
- `required: false`: if the dependency isn't running or isn't defined (e.g. it's in a disabled profile), carry on without it.

What happens during `docker compose up`:
1. Compose builds a **dependency graph** of services.
2. It starts services with no dependencies first.
3. For each dependent, it **polls the container state** (`State.Health.Status` or exit code) until the condition is met. If a dependency becomes **unhealthy** or exits with an error, `up` fails: `dependency failed to start: container shop-db-1 is unhealthy`.
4. Stopping (`down`/`stop`) goes in **reverse order**.

**Important:** conditions only apply **while Compose is starting things** (`up`). They don't stop a running service if its dependency later fails, and a container restarted by the **Docker daemon** (restart policy) doesn't wait for anything.

### `healthcheck:` in Compose

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]        # exec form: run directly
  # test: ["CMD-SHELL", "curl -fsS http://localhost:8080/health || exit 1"]   # via /bin/sh -c
  interval: 10s
  timeout: 3s
  retries: 5
  start_period: 30s
  start_interval: 2s
  # disable: true                           # turn off the image's HEALTHCHECK
```

- `CMD` = exec form (no shell). `CMD-SHELL` = run with `/bin/sh -c` (needed for `||`, pipes, variables). The image must contain `sh` and the tools used.
- Overrides any `HEALTHCHECK` in the image.

Health checks for common services:

| Service | `test` |
|---|---|
| PostgreSQL | `["CMD-SHELL", "pg_isready -U $${POSTGRES_USER:-postgres}"]` |
| MySQL/MariaDB | `["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-p$${MYSQL_ROOT_PASSWORD}"]` (MariaDB images ship `healthcheck.sh --connect --innodb_initialized`) |
| Redis | `["CMD", "redis-cli", "ping"]` |
| RabbitMQ | `["CMD", "rabbitmq-diagnostics", "-q", "ping"]` |
| Elasticsearch | `["CMD-SHELL", "curl -fsS http://localhost:9200/_cluster/health?wait_for_status=yellow&timeout=1s"]` |
| HTTP app (with curl) | `["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]` |
| HTTP app (Alpine, wget) | `["CMD", "wget", "-q", "--spider", "http://localhost:8080/healthz"]` |

(`$$` = a literal `$` so the variable is expanded **inside the container**, not by Compose; topic 36.)

### Init and migration services

A **one-shot service**: a container that runs a task and exits with 0. Other services depend on it with `service_completed_successfully`.
- Typical jobs: DB migrations, creating buckets/topics, fixing volume permissions, generating config.
- Give it `restart: "no"` (the default). A `restart: always` init job would run forever.
- `docker compose ps -a` shows it as `Exited (0)`.

### `restart:` vs `deploy.restart_policy`

- `restart: no | always | on-failure[:N] | unless-stopped` = the Docker restart policy (topic 32). **This is what plain `docker compose` uses.**
- `deploy.restart_policy` (condition, delay, max_attempts, window) comes from Swarm. Docker Compose supports parts of `deploy` (notably `resources` limits and `replicas`), but for restarts, use `restart:`.

### `docker compose up --wait`

```bash
docker compose up -d --wait --wait-timeout 120
```

- Returns only when every service is **running and healthy** (or completed, for one-shot services). Exits non-zero otherwise.
- Ideal for CI and deploy scripts: "deploy, and fail the pipeline if it doesn't become healthy".

### Stop behaviour

```yaml
services:
  api:
    stop_grace_period: 30s      # like docker stop -t 30 (default 10s)
    stop_signal: SIGINT         # like --stop-signal
    init: true                  # like docker run --init (tini as PID 1)
```

These are the topic 12 ideas, expressed in Compose.

### Why apps still need retries

Even with `service_healthy`:
- the dependency can restart later (crash, upgrade, OOM),
- a health check passing doesn't guarantee **every** later request works,
- in Kubernetes there's **no** `depends_on` at all.

So apps should **retry connections with back-off** at startup and **reconnect** when a connection drops. Treat `depends_on` conditions as a nice startup optimisation, not your resilience strategy.

---

## Commands / files

### Without readiness: the race

```yaml
# race/compose.yaml
services:
  db:
    image: postgres:17-alpine
    environment: { POSTGRES_PASSWORD: pw }
  app:
    image: postgres:17-alpine
    depends_on: [db]
    command: ["psql", "postgresql://postgres:pw@db:5432/postgres", "-c", "SELECT 'connected!'"]
```

```bash
docker compose up app
```

```
[+] Running 3/3
 ✔ Network race_default  Created
 ✔ Container race-db-1   Created
 ✔ Container race-app-1  Created
Attaching to app-1
app-1  | psql: error: connection to server at "db" (172.21.0.2), port 5432 failed: Connection refused
app-1  | 	Is the server running on that host and accepting TCP/IP connections?
app-1 exited with code 2
```

- `db` had **started**, but Postgres wasn't accepting connections yet.

### With `service_healthy`

```yaml
services:
  db:
    image: postgres:17-alpine
    environment: { POSTGRES_PASSWORD: pw }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 3s
      retries: 15
  app:
    image: postgres:17-alpine
    depends_on:
      db:
        condition: service_healthy
    command: ["psql", "postgresql://postgres:pw@db:5432/postgres", "-c", "SELECT 'connected!'"]
```

```bash
docker compose down; docker compose up app
```

```
[+] Running 2/2
 ✔ Container race-db-1   Healthy     4.2s
 ✔ Container race-app-1  Created     0.0s
Attaching to app-1
app-1  |   ?column?
app-1  | ------------
app-1  |  connected!
app-1  | (1 row)
app-1 exited with code 0
```

- `Healthy 4.2s` = Compose waited 4.2 s for `db` to become healthy.

### Migration job pattern

```yaml
services:
  db:
    image: postgres:17-alpine
    environment: { POSTGRES_PASSWORD: pw }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      retries: 15
  migrate:
    image: postgres:17-alpine
    depends_on:
      db: { condition: service_healthy }
    command: ["psql", "postgresql://postgres:pw@db/postgres", "-c",
              "CREATE TABLE IF NOT EXISTS items(id serial primary key, name text);"]
  api:
    image: postgres:17-alpine
    depends_on:
      migrate: { condition: service_completed_successfully }
    command: ["sh", "-c", "psql postgresql://postgres:pw@db/postgres -c '\\dt'; sleep 3600"]
```

```bash
docker compose up -d --wait
docker compose ps -a --format 'table {{.Service}}\t{{.Status}}'
docker compose logs api | head
```

```
SERVICE   STATUS
api       Up 3 seconds
db        Up 10 seconds (healthy)
migrate   Exited (0) 4 seconds ago
api-1  |          List of relations
api-1  |  Schema | Name  | Type  |  Owner
api-1  | --------+-------+-------+----------
api-1  |  public | items | table | postgres
```

### A failing dependency stops `up`

Break the migration (`CREATE TABLE` → `CREAT TABLE`) and run `docker compose up -d`:

```
 ✘ Container proj-migrate-1  service "migrate" didn't complete successfully: exit 3
service "migrate" didn't complete successfully: exit 3
```

- `api` is never started, which protects it from running against an un-migrated schema.

### Health status from Compose

```bash
docker compose ps --format 'table {{.Service}}\t{{.Health}}\t{{.Status}}'
docker inspect $(docker compose ps -q db) -f '{{json .State.Health.Log}}' | python3 -m json.tool | tail -8
```

---

## Hands-on lab

1. Reproduce the race, then fix it with `service_healthy`.
2. Add the `migrate` one-shot service and confirm the order: db healthy → migrate exits 0 → api starts.
3. Break the migration and observe `up` failing. Fix it.
4. Add `restart: true` under `api`'s `depends_on.db`, run `docker compose restart db`, and see `api` restarted too.
5. Use `docker compose up -d --wait --wait-timeout 30` and check `echo $?` both when healthy and when you break `db`'s health check (e.g. `pg_isready -U nobody -h 10.255.255.1`).
6. Write a tiny app (any language) that retries the DB connection 10 times with a 1-second delay. Run it **without** a health condition, and see that it still succeeds.
7. Clean up: `docker compose down -v`

---

## Where you'll use it

- **Every multi-service Compose stack** with databases or queues.
- **CI integration tests:** `up -d --wait` before running tests, `down -v` after.
- **Deployments on single hosts:** migrations as one-shot services before the app starts.
- **Interviews:** "start order vs readiness" and "how do you handle dependencies between containers?"
- **Kubernetes:** the same needs are met by init containers (like one-shot services), readiness probes (like `service_healthy`) and app-level retries.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| App crashes with "connection refused" on first `up`, works on second | No readiness wait (plain `depends_on`) | `condition: service_healthy` + health check; retries in the app |
| `dependency failed to start: container x is unhealthy` | The dependency's health check failed `retries` times | `docker inspect … .State.Health.Log`; fix the check or the service; raise `start_period` |
| `up` hangs for a long time then fails | Health check never passes (wrong command, tool missing, wrong credentials) | Run the check manually with `docker compose exec db <test cmd>` |
| `service "migrate" didn't complete successfully: exit 1` | The one-shot job failed | `docker compose logs migrate` |
| Init job runs again and again | It has `restart: always`/`unless-stopped` | Use `restart: "no"` for one-shot services |
| Health check uses an env var that comes out empty | Compose interpolated `$VAR` itself | Use `$$VAR` |
| App broke when the DB restarted later, even though startup was fine | `depends_on` only affects startup | Reconnect/retry logic in the app; `restart: unless-stopped` |
| `depends_on` ignored with `docker compose run` | `run` starts dependencies, but waits only according to conditions | Use conditions, or `--no-deps` deliberately |

---

## Quick recap

- **Started ≠ ready.** Plain `depends_on` gives order only.
- Conditions: `service_started`, **`service_healthy`** (needs a `healthcheck:`), **`service_completed_successfully`** (one-shot jobs like migrations). Plus `restart: true` and `required: false`.
- Write cheap, accurate health checks with tools that exist in the image (`pg_isready`, `redis-cli ping`, curl/wget). Use `$$` for container-side variables.
- `docker compose up -d --wait` makes deploys and CI fail fast if services don't become healthy.
- Conditions apply at startup only. Apps still need retries and reconnection.

**Interview questions**

1. **Q: Does `depends_on` make Compose wait until a database is ready?**
   A: Not by default. It only orders container start. To wait for readiness, give the database a healthcheck and use `depends_on` with `condition: service_healthy`. The app should still retry connections.

2. **Q: How do you run database migrations before starting the app in Compose?**
   A: Add a one-shot `migrate` service that depends on the DB being healthy and exits 0 when done. The app depends on it with `condition: service_completed_successfully`.

3. **Q: How do you make a CI pipeline fail if the Compose stack doesn't come up healthy?**
   A: `docker compose up -d --wait --wait-timeout <seconds>`. It returns non-zero if any service fails to become running/healthy in time.
