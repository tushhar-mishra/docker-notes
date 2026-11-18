# 39 · Compose on a Single Production Host

> **Level:** ★★ intermediate
>
> **You will learn:** how to run Compose properly in production on one server: deploy by **pulling** pinned images (never building on the server), set resource limits, logging and restart policies in the file, run a safe deploy and rollback script, scale services, update with little or no downtime behind a reverse proxy, start the stack with systemd, back up data, and recognise when you've outgrown Compose.
>
> **You need:** topics 31–38 (limits, restart/health, logging, daemon config, Compose).

---

## What it is

Many real systems (internal tools, small SaaS products, side projects, edge sites) run on **one Docker host with Compose**. That's a legitimate production setup **if** it's done carefully:

```yaml
# /opt/shop/compose.yaml
name: shop

x-defaults: &defaults
  restart: unless-stopped
  logging:
    driver: local
    options: { max-size: "20m", max-file: "5" }
  init: true

services:
  proxy:
    <<: *defaults
    image: traefik:v3.5@sha256:<digest>
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports: ["80:80"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro      # see the security note below
    deploy:
      resources:
        limits: { cpus: "0.5", memory: 256m }

  web:
    <<: *defaults
    image: ghcr.io/acme/shop-web:${WEB_TAG:?set WEB_TAG}
    env_file: [web.env]
    labels:
      - traefik.enable=true
      - traefik.http.routers.web.rule=Host(`shop.example.com`)
      - traefik.http.services.web.loadbalancer.server.port=8000
      - traefik.http.services.web.loadbalancer.healthcheck.path=/healthz
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:8000/healthz"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 20s
    stop_grace_period: 30s
    depends_on:
      db: { condition: service_healthy }
    deploy:
      replicas: 2
      resources:
        limits: { cpus: "1.0", memory: 512m, pids: 256 }
        reservations: { memory: 256m }

  db:
    <<: *defaults
    image: postgres:17.6-alpine@sha256:<digest>
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [default]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
    deploy:
      resources:
        limits: { memory: 1g }
    shm_size: 256m

volumes:
  pgdata:
    name: shop_pgdata

secrets:
  db_password:
    file: ./secrets/db_password
```

---

## Why it exists

Compose is simple, cheap and easy to understand. One VM with Compose can serve a lot of traffic. The risks are about **operations**: data safety, updates, monitoring, and the host itself being a single point of failure. This topic is about reducing those risks.

---

## Analogy

Running a **small shop by yourself**. You don't need a franchise headquarters (Kubernetes), but you do need: stock delivered from a trusted warehouse (pull, don't build), a lock on the till (secrets), daily takings in the safe (backups), a sign saying "back in 5 minutes" during restocking (low-downtime updates), and the honesty to know when the shop has become too busy for one person.

---

## How it works inside

### Rule 1: build in CI, pull on the server

- The server should **never** run `docker compose build`. Build in CI, test, scan, push to a registry with an immutable tag (topic 22, 46).
- The server's compose file references **pinned tags** (ideally with digests). The version to deploy comes from a variable (`WEB_TAG`) in `.env`, set by the deploy script.
- Benefits: what you tested is exactly what runs, builds don't eat server CPU, and rolling back = changing a tag.

### Rule 2: limits, logs, restart and health in the file

- `deploy.resources.limits` (`cpus`, `memory`, `pids`) and `reservations` (memory → `--memory-reservation`) become the cgroup settings from topic 31. (The older service-level keys `mem_limit`, `cpus`, `pids_limit` also work.)
- `logging:` with rotation for every service (or set the daemon default; topic 33/34).
- `restart: unless-stopped` for long-running services.
- `healthcheck:` for everything that has a meaningful check.
- `init: true` and `stop_grace_period` for clean shutdowns (topic 12).

### Rule 3: how Compose updates a service (and why it causes downtime)

When `WEB_TAG` changes and you run `docker compose up -d`, for each container of `web` Compose:
1. **stops** the old container (SIGTERM → grace period),
2. **removes** it,
3. **creates and starts** a new one.

Between steps 1 and 3, **nothing serves traffic** for that container. For one replica, that's a short outage (seconds to a minute, depending on start time). Compose has no built-in rolling update. To get close to zero downtime on one host:

**Option A: two replicas behind a reverse proxy, updated one at a time.**
- A reverse proxy (Traefik, nginx, Caddy, HAProxy) load-balances across the `web` replicas and **health-checks** them. (It also terminates HTTPS. Certificates, ACME, forwarded headers and proxy timeouts are covered in topic 39a.)
- Compose recreates replicas one after another. While `web-1` restarts, `web-2` serves (the proxy drops the stopped one).
- Still a small risk window, and both replicas end up on the new version within seconds. Fine for many apps.

**Option B: scale up, then scale down.**
```bash
docker compose up -d --no-deps --scale web=4 --no-recreate web     # start 2 extra containers on the new image
# wait until the new ones are healthy (proxy sends them traffic)
docker stop shop-web-1 shop-web-2 && docker rm shop-web-1 shop-web-2   # remove the old ones
docker compose up -d --no-deps --scale web=2 --no-recreate web     # back to the desired count
```
- Works because `--no-recreate` keeps old containers while new ones are added from the **updated** definition. Community plugins (e.g. `docker rollout`) automate this pattern.

**Option C: blue/green with two projects.**
- `shop-blue` and `shop-green` stacks behind the proxy. Deploy the new version to the idle colour, check it, switch the proxy, keep the old colour for instant rollback.

**Databases are different:** you don't run two Postgres containers on the same volume. DB upgrades are planned maintenance (or use a managed database).

### Rule 4: the Docker socket and the proxy

Traefik reads container labels through the Docker API, so it needs the Docker socket. **Socket access = root on the host** (topic 40), even mounted `:ro` (read-only only affects the socket *file*, not what API calls can do). Mitigations: a **socket proxy** container (e.g. `tecnativa/docker-socket-proxy`) that only allows read-only `GET` endpoints, or Traefik's file provider (static config, no socket). The same applies to Watchtower-style auto-updaters: convenient, but they hold root and deploy whatever appears in the registry. Many teams avoid them in production.

### Rule 5: start at boot

Two options:
1. **Restart policies** (`unless-stopped`) + `systemctl enable docker`. dockerd restarts the containers at boot. Simple and usually enough.
2. **A systemd unit** for the stack, if you want ordering with other host services, or `up --wait` at boot:

```ini
# /etc/systemd/system/shop.service
[Unit]
Description=Shop (Docker Compose)
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/shop
ExecStart=/usr/bin/docker compose up -d --wait --remove-orphans
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

- `Type=oneshot` + `RemainAfterExit=yes` = the unit counts as "active" after `up` finishes (the containers run under dockerd, not under this unit).
- Don't mix both approaches carelessly: with this unit, `systemctl stop shop` runs `down` (removes containers).

### Rule 6: back up what matters

- **Volumes with data**: logical DB dumps on a schedule (cron/systemd timer), copied **off the host** (S3, another server). Test restores regularly (topic 24).
- **The deployment folder**: compose files, `.env`, configs: keep them in git. Secrets: in a secret manager or encrypted (topic 44).
- The host itself is replaceable if everything above is backed up and the setup is scripted.

### When Compose stops being enough

| Need | Compose on one host | What to look at |
|---|---|---|
| Survive a **host failure** | ❌ the host is a single point of failure | Multiple hosts + orchestrator, or managed services |
| **Rolling updates** with health gates, automatic rollback | Manual scripting | Kubernetes, Swarm, Nomad, ECS |
| **Restart unhealthy** containers automatically | ❌ | Same |
| **Autoscaling** on load | ❌ | Kubernetes HPA (Horizontal Pod Autoscaler), ECS, Cloud Run |
| Scheduling across many machines | ❌ | Kubernetes, Nomad |
| Fine-grained RBAC, multi-team isolation | Limited | Kubernetes |

Signs you've outgrown it: you need high availability, you have more than a handful of hosts, deploys need to be hands-off and gated, or many teams deploy independently. **Swarm** is the smallest step (it reads compose files, `docker stack deploy`). **Kubernetes** is the industry standard (topic 53).

---

## Commands / files

### A deploy script with rollback

```bash
#!/usr/bin/env bash
# /opt/shop/deploy.sh  — usage: ./deploy.sh 1.4.3
set -euo pipefail
cd /opt/shop

NEW_TAG="$1"
OLD_TAG=$(grep '^WEB_TAG=' .env | cut -d= -f2)
echo "Deploying web ${OLD_TAG} -> ${NEW_TAG}"

sed -i "s/^WEB_TAG=.*/WEB_TAG=${NEW_TAG}/" .env

docker compose pull web
if docker compose up -d --wait --wait-timeout 120 --remove-orphans web; then
  echo "Deploy OK: web ${NEW_TAG}"
  docker image prune -af --filter "until=168h" >/dev/null
else
  echo "Deploy FAILED, rolling back to ${OLD_TAG}" >&2
  sed -i "s/^WEB_TAG=.*/WEB_TAG=${OLD_TAG}/" .env
  docker compose up -d --wait --wait-timeout 120 web
  exit 1
fi
```

- `set -euo pipefail` = stop on errors, unset variables and pipe failures.
- `pull` first, so the outage window isn't spent downloading.
- `up --wait` fails if the new containers don't become healthy → automatic rollback to the previous tag.
- `--remove-orphans` = remove containers of services that were deleted from the file.
- The image prune keeps a week of old images, so rollbacks don't need a pull.

### Run it remotely from your machine or CI

```bash
docker context create shop-prod --docker "host=ssh://deploy@shop-prod-1"
DOCKER_CONTEXT=shop-prod docker compose -f compose.yaml --env-file prod.env up -d --wait
```

- Compose runs **locally** but talks to the remote daemon over SSH (topic 34). Relative bind-mount paths and `secrets: file:` refer to the **remote** host's filesystem when the daemon creates the mounts, which is a common surprise. Keep the deployment folder on the server, or avoid bind mounts.

### Check limits and replicas were applied

```bash
docker compose ps
docker inspect shop-web-1 -f 'mem={{.HostConfig.Memory}} cpus={{.HostConfig.NanoCpus}} pids={{.HostConfig.PidsLimit}} restart={{.HostConfig.RestartPolicy.Name}} log={{.HostConfig.LogConfig.Type}}'
```

```
NAME          IMAGE                          SERVICE   STATUS                    PORTS
shop-db-1     postgres:17.6-alpine           db        Up 2 hours (healthy)      5432/tcp
shop-proxy-1  traefik:v3.5                   proxy     Up 2 hours                0.0.0.0:80->80/tcp
shop-web-1    ghcr.io/acme/shop-web:1.4.3    web       Up 5 minutes (healthy)
shop-web-2    ghcr.io/acme/shop-web:1.4.3    web       Up 5 minutes (healthy)
mem=536870912 cpus=1000000000 pids=256 restart=unless-stopped log=local
```

### Nightly DB backup (systemd timer)

```ini
# /etc/systemd/system/shop-backup.service
[Service]
Type=oneshot
WorkingDirectory=/opt/shop
ExecStart=/bin/sh -c 'docker compose exec -T db pg_dump -U postgres -Fc postgres > /backup/shop-$(date +%%F).dump && find /backup -name "shop-*.dump" -mtime +14 -delete'
```

```ini
# /etc/systemd/system/shop-backup.timer
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
[Install]
WantedBy=timers.target
```

- `exec -T` = no TTY (needed when output is redirected). `%%` = a literal `%` in systemd units.
- Ship `/backup` off the host (rclone/aws s3 sync) in another step.

---

## Hands-on lab

1. Turn the `counter` app into a "production" setup: an image pushed to your local registry (topic 22) with tags `1.0` and `1.1`, a compose file using `${WEB_TAG}`, limits, log rotation, health check, 2 replicas, and Traefik in front (or nginx with a static upstream to both replicas).
2. Write `deploy.sh` and deploy `1.0`, then `1.1`. While deploying, run `while true; do curl -s -o /dev/null -w '%{http_code}\n' localhost/; sleep 0.2; done` and count non-200 responses.
3. Break `1.2` on purpose (a health endpoint that returns 500) and deploy it. Watch the automatic rollback.
4. Install the systemd unit, `sudo systemctl enable --now shop`, reboot (or `wsl --shutdown`), and check the stack comes back.
5. Set up the backup timer and run it once manually: `sudo systemctl start shop-backup.service`, then restore the dump into a scratch Postgres.
6. Clean up: `docker compose down` (keep or remove volumes as you like), disable the units.

---

## Where you'll use it

- **Small and medium production systems**, internal platforms, staging environments, on-prem appliances, edge servers.
- **Interviews:** "How would you deploy this app on a single VM?" and "How do you do zero-downtime deploys with Compose?"
- **Recognising the limit:** the move from Compose to Kubernetes/ECS/Nomad is a common real-world project.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Short outage on every deploy | Single replica recreated (stop → start) | Two replicas + health-checking proxy, scale-up/scale-down, or blue/green |
| Deploy "succeeded" but the app is broken | No health checks / no `--wait` | Health checks + `up --wait` + rollback |
| Server CPU maxed during deploys | Building on the server | Build in CI, pull on the server |
| `pull` fails during deploy (rate limit/registry down) | External dependency at deploy time | Private registry/mirror; keep previous images locally |
| Containers not back after reboot | No restart policy / docker not enabled / unit not enabled | `restart: unless-stopped`; `systemctl enable docker` (and your unit) |
| Old service containers still running after removing them from the file | Orphans | `up -d --remove-orphans` |
| Disk fills over weeks | Old images, logs, build cache | Log rotation, scheduled image prune with `until`, no builds on the server |
| Traefik/updater container compromised = host compromised | Docker socket mounted | Socket proxy with read-only API, or a file-based config |
| `--scale` fails: `port is already allocated` | Replicas publishing the same fixed host port | Don't publish replica ports. Publish only the proxy |

---

**Next:** topic 39a puts HTTPS in front of this stack (TLS, certificates, ACME, proxy headers and timeouts).

## Quick recap

- **Build in CI, pull on the server.** Pin images (tag + digest), choose the version with a variable, deploy with `pull` + `up -d --wait`, and roll back by restoring the previous tag.
- Put limits (`deploy.resources`), log rotation, `restart: unless-stopped`, health checks, `init` and grace periods in the file.
- Compose updates = stop/remove/create, so use replicas behind a health-checking proxy, scale-up/scale-down, or blue/green for low downtime.
- Start at boot with restart policies (or a systemd oneshot unit). Back up volumes off-host and test restores.
- Outgrown when you need HA across hosts, health-based self-healing, rolling updates, autoscaling or multi-team isolation → Swarm/Kubernetes/ECS/Nomad.

**Interview questions**

1. **Q: How would you deploy and update a Compose application on a single production server?**
   A: Build and push images in CI with immutable tags. On the server, keep a compose file that references a tag variable, set the new tag, `docker compose pull`, then `docker compose up -d --wait`. On failure, set the previous tag and `up` again. Add limits, log rotation, restart policies and health checks, plus scheduled off-host backups of volumes.

2. **Q: Why does a Compose update cause downtime, and how do you reduce it?**
   A: Compose recreates containers by stopping the old one before starting the new one, with no rolling update. Run several replicas behind a reverse proxy that health-checks them, scale up new containers before removing old ones, or use blue/green stacks.

3. **Q: When would you move from Compose to an orchestrator?**
   A: When you need high availability across hosts, automated rolling updates with health gates and rollback, self-healing of unhealthy containers, autoscaling, scheduling across many machines, or stronger multi-team isolation.
