# 32 · Restart Policies and Health Checks

> **Level:** ★★ intermediate
>
> **You will learn:** how Docker automatically restarts crashed containers (`no`, `on-failure[:N]`, `always`, `unless-stopped`), the restart back-off delay, what happens at boot and when dockerd restarts, how `HEALTHCHECK` works (states, timing options, where results are stored), and the important fact that **plain Docker does not restart unhealthy containers**.
>
> **You need:** topic 08 (lifecycle, exit codes), topic 12 (signals), topic 20 (HEALTHCHECK mention).

---

## What it is

- A **restart policy** tells dockerd what to do when a container's main process exits: leave it stopped, or start it again.
- A **health check** is a command Docker runs **inside** the container at an interval to decide whether the app actually **works** (not just whether its process exists). The result is a **health status**: `starting`, `healthy` or `unhealthy`.

---

## Why it exists

- Apps crash: a bug, a lost database connection, an OOM kill. Something should bring them back without a human, at 3 AM.
- Servers reboot. Containers should come back with them.
- A process can be **running but broken**: deadlocked, stuck in a loop, unable to reach its database, returning 500s. "Running" isn't "healthy". Health checks detect that, and let dependents (Compose `depends_on`, load balancers, Swarm) react.

---

## Analogy

- **Restart policy** = a **self-resetting circuit breaker**. If the machine trips, it switches back on automatically. But if it keeps tripping every second, it waits longer and longer between attempts.
- **Health check** = a **nurse checking a patient's pulse every 30 seconds**. A patient can be in the bed (process running) but not breathing properly (unhealthy). With plain Docker, the nurse only **writes "unhealthy" on the chart**. They don't call the doctor (restart). Orchestrators (Swarm, Kubernetes) are the ones who act on the chart.

---

## How it works inside

### Restart policies

| Policy | Restarts when the process exits… | After `docker stop`? | After host reboot / dockerd restart? |
|---|---|---|---|
| `no` (default) | Never | — | Not started |
| `on-failure[:N]` | Only with a **non-zero exit code**; at most N times (if N is given) | Stays stopped | **Not started** (the daemon restarting isn't a "failure") |
| `always` | Always, any exit code | Stays stopped **until dockerd restarts**, then **starts again**! | **Started** |
| `unless-stopped` | Always, any exit code | Stays stopped, **even after dockerd/host restarts** | Started, unless you stopped it manually |

Key details:
- **Manual stop wins.** `docker stop` marks the container as manually stopped, so the policy doesn't fight you. The difference between `always` and `unless-stopped` is only what happens at the **next daemon start**: `always` brings back containers you stopped, `unless-stopped` remembers that you stopped them.
- **Who restarts it:** dockerd. It watches containers (it gets the exit event from containerd) and starts them again. If dockerd is down, nothing restarts (unless `live-restore` kept the containers alive; topic 34).
- **Back-off:** to avoid a tight crash loop, dockerd waits **100 ms, then doubles the delay each time** (200 ms, 400 ms, …) up to a maximum of about **1 minute**. If the container then runs for at least **10 seconds**, the delay resets. While waiting, `docker ps` shows `Restarting (1) 3 seconds ago`.
- **`RestartCount`** in `docker inspect` counts restarts. It's one of the best crash-loop signals.
- **A restart is the same container**: same writable layer, same config, a new process (topic 08).
- `--rm` and restart policies can't be combined (except `no`).

### Health checks

Defined in the Dockerfile (`HEALTHCHECK`) or overridden at run time (`--health-cmd` etc.):

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --start-interval=5s --retries=3 \
  CMD ["curl", "-fsS", "http://127.0.0.1:8080/healthz"]
```

| Option | Default | Meaning |
|---|---|---|
| `--interval` | 30s | Time between checks |
| `--timeout` | 30s | A check taking longer counts as failed |
| `--retries` | 3 | Consecutive failures needed to become `unhealthy` |
| `--start-period` | 0s | Grace period after start: failures don't count (successes do mark it healthy) |
| `--start-interval` | 5s | How often to check **during** the start period (Engine 25+) |

The command's **exit code** decides:
- `0` = healthy,
- `1` = unhealthy,
- (`2` is reserved; don't use it).

How Docker runs it:
1. dockerd `exec`s the command **inside the container** (like `docker exec`), in the container's namespaces, with its user and filesystem. So the command (`curl`, `wget`, a binary) **must exist in the image**.
2. It stores the last **5 results** (exit code + first part of the output) in the container state.
3. The status goes `starting` → `healthy`, or → `unhealthy` after `retries` consecutive failures. One success → back to `healthy`.
4. Status changes produce events: `health_status: healthy` / `health_status: unhealthy`.

**What plain Docker does with `unhealthy`: nothing.** It doesn't restart, kill or remove the container. The status is information for:
- you and your monitoring (`docker ps` shows `(unhealthy)`),
- **Compose** `depends_on: condition: service_healthy` (topic 37) and `docker compose up --wait`,
- **Swarm**, which replaces unhealthy tasks,
- your own automation (some people run an "autoheal" container that restarts unhealthy containers; it needs the Docker socket, so mind the risk from topic 40).

**Kubernetes ignores `HEALTHCHECK`.** It uses its own liveness/readiness/startup probes (topic 53).

### What makes a good health check

- **Cheap and fast**: it runs every interval, forever.
- **Checks the app itself**, through its real interface (an HTTP `/healthz` endpoint, a DB `SELECT 1`, `redis-cli ping`).
- **Don't** make it depend on every downstream service. If the database is down and every app's health check fails, an orchestrator might restart all apps for nothing (a cascading restart). Keep "am I alive?" separate from "can I serve traffic?". Kubernetes makes this split explicit (liveness vs readiness).
- Works without a shell or curl in minimal images: use a tiny static health binary, or the app's own `--health` flag.

---

## Commands / files

### Restart policy: on-failure with a limit

```bash
docker run -d --name crashy --restart on-failure:3 alpine sh -c 'echo "start $(date +%T)"; sleep 1; exit 1'
sleep 20
docker ps -a --filter name=crashy --format '{{.Status}}'
docker inspect crashy -f 'RestartCount={{.RestartCount}} ExitCode={{.State.ExitCode}}'
docker logs crashy
```

```
Exited (1) 12 seconds ago
RestartCount=3 ExitCode=1
start 14:00:01
start 14:00:02
start 14:00:04
start 14:00:07
```

- 1 original run + 3 restarts, then it gave up.
- The gaps grow (1 s, 2 s, 3 s): the run time plus the growing back-off delay.

### always vs unless-stopped after a daemon restart

```bash
docker run -d --name a1 --restart always alpine sleep 3600
docker run -d --name u1 --restart unless-stopped alpine sleep 3600
docker stop a1 u1
sudo systemctl restart docker
docker ps --format '{{.Names}} {{.Status}}' | grep -E 'a1|u1'
```

```
a1 Up 2 seconds
```

- `a1` (always) came back even though you stopped it. `u1` stayed stopped.
- **Use `unless-stopped` for most long-running services** on single hosts. That way, stopping something for maintenance survives a reboot.

### Crash loop in action

```bash
docker run -d --name loop --restart always alpine sh -c 'exit 1'
for i in 1 2 3 4; do docker ps --filter name=loop --format '{{.Status}}'; sleep 3; done
docker inspect loop -f '{{.RestartCount}}'
```

```
Restarting (1) Less than a second ago
Restarting (1) 1 second ago
Up Less than a second
Restarting (1) 2 seconds ago
9
```

### Health check: define, watch, read

```bash
docker run -d --name hc -p 8085:80 \
  --health-cmd 'wget -q -O /dev/null http://127.0.0.1/ || exit 1' \
  --health-interval 5s --health-timeout 2s --health-retries 3 --health-start-period 5s \
  nginx:1.27-alpine
docker ps --filter name=hc --format '{{.Status}}'
sleep 8
docker ps --filter name=hc --format '{{.Status}}'
```

```
Up 1 second (health: starting)
Up 9 seconds (healthy)
```

Break it:

```bash
docker exec hc sh -c 'rm /usr/share/nginx/html/index.html'
sleep 20
docker ps --filter name=hc --format '{{.Status}}'
docker inspect hc -f '{{json .State.Health}}' | python3 -m json.tool | head -20
```

```
Up 40 seconds (unhealthy)
{
    "Status": "unhealthy",
    "FailingStreak": 4,
    "Log": [
        {
            "Start": "2026-11-17T14:10:21.1+05:30",
            "End": "2026-11-17T14:10:21.2+05:30",
            "ExitCode": 1,
            "Output": "wget: server returned error: HTTP/1.1 403 Forbidden\n"
        },
        ...
    ]
}
```

- `FailingStreak` = consecutive failures. `Log` = the last 5 results with **their output**: the first place to look when a container is unhealthy.
- (Without `index.html`, nginx returns 403 for `/`. That's a contrived failure, but it's exactly how a broken app shows up.)
- The container is **still running**. Docker didn't restart it.

### Find unhealthy containers and listen for events

```bash
docker ps --filter health=unhealthy
docker events --filter event=health_status --since 5m
```

```
CONTAINER ID   IMAGE               COMMAND                  CREATED         STATUS                     PORTS                  NAMES
7e8f9a0b1c2d   nginx:1.27-alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes (unhealthy)   0.0.0.0:8085->80/tcp   hc
2026-11-17T14:09:53 container health_status: healthy 7e8f… (image=nginx:1.27-alpine, name=hc)
2026-11-17T14:10:21 container health_status: unhealthy 7e8f… (image=nginx:1.27-alpine, name=hc)
```

### Disable an image's health check

```bash
docker run --no-healthcheck myimage
```

### Change the policy on an existing container

```bash
docker update --restart unless-stopped hc
docker inspect hc -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.RestartPolicy.MaximumRetryCount}}'
```

```
unless-stopped 0
```

---

## Hands-on lab

1. Run the `on-failure:3`, `always`/`unless-stopped` and crash-loop examples. Record `RestartCount` each time.
2. **Reboot test** (in WSL: `wsl --shutdown` from PowerShell, then reopen Ubuntu, which restarts dockerd): which containers come back?
3. Build an image with a Dockerfile `HEALTHCHECK` for `hello-flask` (topic 14): add a `/health` route and `HEALTHCHECK CMD ["python", "-c", "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2).status==200 else 1)"]`. (This avoids needing curl in the image.) Watch it go `starting → healthy`.
4. Make the app return 500 on `/health` (an environment-variable toggle, then recreate the container) and watch it turn unhealthy. Confirm it's **not** restarted.
5. **OOM + restart policy:** `docker run -d --name oomloop --restart on-failure --memory 50m python:3.13-slim python -c "x=' '*100_000_000"` and watch `RestartCount` grow and `OOMKilled` stay true.
6. Clean up: `docker rm -f crashy a1 u1 loop hc oomloop 2>/dev/null`

---

## Where you'll use it

- **Single-host production** (plain Docker or Compose): `--restart unless-stopped` on every long-running service, so they survive crashes and reboots.
- **Startup ordering in Compose:** health checks + `depends_on: condition: service_healthy` (topic 37).
- **Deploy pipelines:** `docker compose up --wait` waits until services are healthy before declaring success.
- **Monitoring:** alert on `health_status: unhealthy` events, `RestartCount` growth, and containers in `Restarting` state.
- **Kubernetes:** the same ideas as Pod `restartPolicy`, liveness and readiness probes, and `CrashLoopBackOff` (Kubernetes' version of the back-off: 10 s doubling up to 5 min).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Restarting (1) …` forever, `RestartCount` climbing | Crash loop: app exits right after start | `docker logs`, exit code, `OOMKilled` (topic 50) |
| Container didn't come back after reboot | Policy is `no`, or `unless-stopped` and it was stopped before the reboot, or dockerd isn't enabled at boot | Set a policy; `systemctl enable docker` |
| A container you stopped came back after reboot | Policy `always` | Use `unless-stopped` |
| `(unhealthy)` and nothing happens | Expected: plain Docker only reports | Fix the app, restart manually, or use Compose/Swarm/K8s logic |
| Health check fails with `exec: "curl": executable file not found` | Health command isn't in the image | Use a tool that exists (`wget` on Alpine), a small health binary, or the app itself |
| Healthy/unhealthy flapping | Timeout too short, or the check is expensive/slow under load | Raise `--timeout`; make the check cheaper |
| App marked unhealthy during slow startup | No start period | `--start-period` (and `--start-interval`) |
| All services unhealthy whenever the DB blips | Health checks depend on downstream services | Check only the app's own liveness |
| `Conflicting options: --restart and --rm` | Can't auto-remove a container that should restart | Choose one |

---

## Quick recap

- Policies: `no` (default), `on-failure[:N]` (non-zero exits only), `always`, `unless-stopped` (like always, but respects manual stops across daemon restarts). dockerd restarts containers with an exponential back-off (100 ms → ~1 min, reset after 10 s of uptime).
- `RestartCount` and `Restarting` status reveal crash loops.
- `HEALTHCHECK` runs a command **inside** the container at intervals. Exit 0 = healthy, 1 = unhealthy. Status: `starting`/`healthy`/`unhealthy`, with the last 5 results in `.State.Health.Log`.
- **Plain Docker never restarts unhealthy containers.** Compose (`service_healthy`, `--wait`), Swarm and your monitoring act on it. Kubernetes uses its own probes.
- Good checks are cheap, test the app itself, and use tools present in the image.

**Interview questions**

1. **Q: What's the difference between `always` and `unless-stopped`?**
   A: Both restart a container whenever it exits. With `always`, a container you stopped manually is started again when the Docker daemon restarts (e.g. after a reboot). `unless-stopped` remembers the manual stop and leaves it stopped.

2. **Q: Will Docker restart a container that becomes unhealthy?**
   A: No. Standalone Docker only marks it unhealthy and emits an event. Restart policies react only to the process exiting. Swarm, Compose dependency checks, external tooling or Kubernetes probes act on health.

3. **Q: How do you diagnose a container stuck in a restart loop?**
   A: Check `docker ps` (Restarting) and `docker inspect` for RestartCount, ExitCode and OOMKilled. Read `docker logs` (including previous runs: the log file persists across restarts) and fix the cause, such as a config error, a missing dependency, OOM, or a wrong command.
