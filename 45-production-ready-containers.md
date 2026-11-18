# 45 · Production-Ready Containers

> **Level:** ★★ intermediate
>
> **You will learn:** what makes a containerised app ready for production, as one practical checklist: configuration, state, logs, shutdown, health endpoints, startup behaviour, dependencies and retries, security settings, resources and runtime sizing, time zones and locale, observability, and image hygiene. Each item has the reason it matters and how to check it. This topic pulls everything from the earlier parts together.
>
> **You need:** Parts 1–8. This is a synthesis topic.

---

## What it is

A **production-ready container** behaves well when the platform (Docker, Compose, Kubernetes) does its normal things: **starts it, stops it, restarts it, scales it, moves it, limits it, and watches it**, often with no human involved. Most production container incidents come from apps that weren't built for these normal operations, not from Docker itself.

The usual reference is the **Twelve-Factor App** methodology (12 rules for cloud apps: config in the environment, stateless processes, logs as streams, disposability, dev/prod parity…). The checklist below is its practical, container-specific version.

---

## Why it exists

In production, containers are **cattle, not pets**: any instance can be killed and replaced at any time (deploys, node failures, autoscaling, OOM). An app that stores state locally, ignores SIGTERM, takes 3 minutes to start, logs to files and has no health endpoint will cause outages during **routine** operations.

---

## Analogy

A **professional delivery driver** vs a friend helping you move. The professional:
- follows the dispatcher's instructions (config from outside),
- doesn't keep packages at home overnight (stateless),
- reports progress on the radio (logs to stdout),
- finishes the current delivery when told to stop, then parks (graceful shutdown),
- answers "are you OK?" on the radio (health checks),
- can be replaced by another driver mid-route without losing packages.

---

## How it works inside: the checklist

### 1. Configuration

| Rule | Why | How to check |
|---|---|---|
| Config comes from **environment variables or mounted files**, never baked into the image | The same image runs in dev/staging/prod (build once, deploy many; topic 22) | `docker run` the same digest with different env in two environments |
| **Secrets** come from files/secret managers, not image layers or plain env where avoidable | Topic 44 | `docker history --no-trunc`, `docker inspect` env |
| Required config is **validated at startup** and the app fails fast with a clear message | A typo should fail the deploy, not cause errors hours later | Start it with a missing variable; expect an immediate, readable error and a non-zero exit |
| Sensible defaults for non-critical settings | Fewer env vars to manage | — |

### 2. State

| Rule | Why | How to check |
|---|---|---|
| **No important data in the writable layer** | Gone on recreate (topic 23) | `docker ps -s` stays small; `docker diff` |
| Persistent data on **volumes** or **external services** (managed DB, object storage like S3) | Replaceable containers | Recreate the container; nothing is lost |
| Sessions/caches in Redis or similar, not in process memory, if you run several replicas | Requests may hit any replica | Scale to 2; log in; kill one; still logged in |
| Writes only to **known paths** (volume, `/tmp` tmpfs) | Allows `--read-only` | Run with `--read-only` + tmpfs (topic 25) |

### 3. Logs

| Rule | Why | How to check |
|---|---|---|
| Log to **stdout/stderr** | The platform collects them (topic 33) | `docker logs` shows everything |
| **Unbuffered** output | Real-time logs, nothing lost on crash | `PYTHONUNBUFFERED=1` etc. |
| **Structured** (JSON) with level, timestamp, request ID | Searchable, multi-line safe | Logs parse as JSON |
| No secrets or personal data in logs | Compliance | Review/redaction tests |
| Log rotation on the host | Disk safety | `daemon.json` (topics 33–34) |

### 4. Startup

| Rule | Why | How to check |
|---|---|---|
| **Fast startup** (seconds, not minutes) | Fast deploys, rollbacks and scaling | Time from `docker run` to healthy |
| **Retries with back-off** for dependencies (DB, queue, other APIs) | Start order ≠ readiness; dependencies restart (topic 37) | Start the app before the DB: it waits and connects, it doesn't crash-loop forever |
| **Migrations run separately** (a one-shot job/init container), not by every replica at startup | Avoid races and long startups | Separate `migrate` command/service |
| **Idempotent startup** (safe to run many times) | Restarts happen | Restart repeatedly: no duplicate side effects |

### 5. Shutdown

| Rule | Why | How to check |
|---|---|---|
| Main process is **PID 1 in exec form** (or tini via `--init`) | Receives SIGTERM (topic 12) | `docker top` shows the app, not `sh -c` |
| On SIGTERM: **stop accepting new work → finish in-flight → close connections → exit 0** within the grace period | No dropped requests or half-done jobs during deploys | `time docker stop` is fast, exit code 0, no client errors under load |
| Grace period configured to match the longest normal request/job | Avoid SIGKILL mid-work | `--stop-timeout` / `stop_grace_period` / Kubernetes `terminationGracePeriodSeconds` |
| Queue workers release or finish messages | No lost/duplicate processing | Kill a worker mid-job; check the message is redelivered once |

### 6. Health

| Rule | Why | How to check |
|---|---|---|
| A **liveness** endpoint (`/livez` or `/healthz`): "the process works", cheap, **no** dependency checks | Restart only when restarting helps | Returns 200 while the DB is down |
| A **readiness** endpoint (`/readyz`): "I can serve traffic now" (dependencies reachable, warm-up done) | Load balancers/Kubernetes route traffic only to ready instances | Returns 503 during startup and shutdown |
| Fail readiness **as the first step of shutdown** | Load balancers stop sending traffic before the app stops | Watch readiness flip on SIGTERM |
| Docker `HEALTHCHECK`/Compose `healthcheck` uses the liveness endpoint | Topic 32 | `docker ps` shows `(healthy)` |

### 7. Security (from Part 8)

| Rule | How to check |
|---|---|
| Non-root numeric `USER` | `docker image inspect -f '{{.Config.User}}'` |
| `--cap-drop ALL` (+ only needed), `no-new-privileges` | `grep Cap /proc/<pid>/status` |
| Read-only root filesystem + tmpfs | `--read-only` works |
| Minimal, pinned, scanned, signed image | Scout/Trivy gate; cosign verify (topic 43) |
| No Docker socket, no `--privileged`, minimal host mounts | Risky-container one-liner (topic 40) |
| Only necessary ports published, bound to the right interface | `docker ps`, `ss -lntp` |

### 8. Resources

| Rule | Why | How to check |
|---|---|---|
| **Memory limit** set; runtime heap sized from it (JVM `MaxRAMPercentage`, `GOMEMLIMIT`, Node `--max-old-space-size`) | No OOM loops, no noisy neighbours (topic 31) | Load test: no `OOMKilled` |
| CPU: shares/requests set; hard limits only with care (throttling) | Predictable latency | `cpu.stat` `nr_throttled` under load |
| **Thread/worker counts** match the CPU limit (not host cores) | `nproc` lies in containers | Check config vs `cpu.max` |
| PID limit set | Fork bombs, zombie buildup | `pids.max` |
| Connection pools sized for `replicas × pool size` < DB max connections | Scaling out shouldn't exhaust the DB | Scale to N, check DB connections |
| **Timeouts** on every outgoing call | One slow dependency mustn't hang all workers | Inject latency; the app degrades instead of hanging |

### 9. Time, locale and DNS

| Rule | Why |
|---|---|
| Containers run in **UTC**. Convert only for display | Consistent logs and timestamps across hosts/regions. Containers have no time-zone data unless installed |
| If a local zone is truly needed: install `tzdata` and set `TZ=Asia/Kolkata` (don't bind-mount `/etc/localtime` unless the image expects it) | Minimal images lack zoneinfo (topic 19) |
| Set a **UTF-8 locale** where the runtime needs it (`LANG=C.UTF-8`) | Avoid encoding errors with non-ASCII data |
| The **clock** comes from the host kernel: keep NTP/chrony on the **host** | Containers can't (and shouldn't) set the time |
| Respect **DNS TTLs**; don't cache resolved IPs forever | Service IPs change (topic 29) |

### 10. Observability

| Rule | Why |
|---|---|
| **Metrics** endpoint (`/metrics` in Prometheus format): request rate, errors, latency, saturation | Alerting and capacity (topic 47) |
| **Version info**: `/version` endpoint and/or a log line at startup with version + git SHA (from build args/labels) | "What's running?" during incidents |
| **Tracing** (OpenTelemetry) with request IDs propagated | Debugging across services |
| Errors reported with context (Sentry-like tools, or structured error logs) | Faster diagnosis |

### 11. Image hygiene

| Rule | Reference |
|---|---|
| Multi-stage build; minimal base; `.dockerignore` | Topics 14–19 |
| OCI labels: source, revision, version, created | Topic 18 |
| One process per container; no SSH, no cron inside (use a separate scheduled container/job) | Topic 20 |
| Deployed by **digest**, built once in CI | Topics 22, 46 |
| Base image updates automated; regular rebuilds | Topics 20, 43 |

---

## Commands / files

### A production-style `docker run` (everything in one place)

```bash
docker run -d --name api \
  --init \
  --user 10001:10001 \
  --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop ALL --security-opt no-new-privileges \
  --memory 512m --memory-swap 512m --cpus 1 --pids-limit 256 \
  --restart unless-stopped --stop-timeout 30 \
  --health-cmd '/app/healthcheck' --health-interval 10s --health-retries 3 --health-start-period 20s \
  --log-driver local --log-opt max-size=20m --log-opt max-file=5 \
  -e TZ=UTC -e LOG_FORMAT=json \
  --mount type=bind,src=/etc/api/secrets/db_password,dst=/run/secrets/db_password,readonly \
  -e DB_PASSWORD_FILE=/run/secrets/db_password \
  --network backend -p 127.0.0.1:8080:8080 \
  ghcr.io/acme/api:1.4.2@sha256:<digest>
```

(Careful: `docker run --health-cmd` **always** runs its string through `/bin/sh -c` (CMD-SHELL), so the image needs a shell. For distroless images, define the check in the Dockerfile in exec form instead: `HEALTHCHECK CMD ["/app/healthcheck"]`. That form runs without a shell.)

### A graceful-shutdown skeleton (Python, for reference)

```python
import signal, sys, threading, time
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

ready = threading.Event()

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/livez":
            self.send_response(200); self.end_headers(); self.wfile.write(b"ok\n")
        elif self.path == "/readyz":
            self.send_response(200 if ready.is_set() else 503); self.end_headers()
        else:
            time.sleep(2)                                  # simulate work
            self.send_response(200); self.end_headers(); self.wfile.write(b"done\n")
    def log_message(self, fmt, *args):                    # structured-ish log to stdout
        print('{"msg": "%s"}' % (fmt % args), flush=True)

srv = ThreadingHTTPServer(("0.0.0.0", 8080), H)
srv.daemon_threads = False                                 # let in-flight requests finish

def on_term(signum, frame):
    print('{"msg": "SIGTERM received, draining"}', flush=True)
    ready.clear()                                          # 1. fail readiness
    time.sleep(5)                                          # 2. give load balancers time to notice
    threading.Thread(target=srv.shutdown).start()          # 3. stop accepting; finish in-flight

signal.signal(signal.SIGTERM, on_term)
ready.set()
print('{"msg": "listening on 8080", "version": "1.4.2"}', flush=True)
srv.serve_forever()
srv.server_close()
print('{"msg": "bye"}', flush=True)
sys.exit(0)
```

### Quick production-readiness probe of a running container

```bash
C=api
docker inspect $C -f 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} Mem={{.HostConfig.Memory}} Pids={{.HostConfig.PidsLimit}} Restart={{.HostConfig.RestartPolicy.Name}} Init={{.HostConfig.Init}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}} Log={{.HostConfig.LogConfig.Type}}{{.HostConfig.LogConfig.Config}} CapDrop={{.HostConfig.CapDrop}} SecOpt={{.HostConfig.SecurityOpt}}'
docker ps -s --filter name=$C --format '{{.Size}}'
time docker stop $C && docker inspect $C -f 'exit={{.State.ExitCode}}'
```

```
User=10001:10001 ReadOnly=true Mem=536870912 Pids=256 Restart=unless-stopped Init=true Health=healthy Log=localmap[max-file:5 max-size:20m] CapDrop=[ALL] SecOpt=[no-new-privileges]
0B (virtual 41.2MB)
real    0m5.6s
exit=0
```

---

## Hands-on lab

1. Take your `hello-flask` (or the Python skeleton above) and grade it against every table. Write down the gaps.
2. Fix them one by one: non-root user, `/livez` + `/readyz`, SIGTERM drain, JSON logs, config validation, read-only root.
3. **Shutdown test under load:**
   ```bash
   (for i in $(seq 1 50); do curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/work & done; wait) > codes.txt &
   sleep 0.5; docker stop api; wait
   sort codes.txt | uniq -c
   ```
   All 200s = graceful shutdown works. Some `000` (connection failed) = requests were cut off.
4. **Dependency test:** start the app before its database. It should retry and become ready once the DB appears, not crash-loop.
5. **Resource test:** with `--memory 256m`, load test and confirm no OOM kills (`docker inspect`, `memory.events`).

---

## Where you'll use it

- **Readiness reviews** before a service goes live (many companies have a formal checklist like this one).
- **Platform engineering:** templates and base images that make the right thing the default.
- **Kubernetes:** the same checklist maps directly to probes, `securityContext`, resources, `terminationGracePeriodSeconds`, ConfigMaps/Secrets (topic 53).
- **Interviews:** "What makes an application container-ready?"

---

## When it breaks (symptom → missing checklist item)

| Symptom in production | Missing item |
|---|---|
| 502/503 errors during every deploy | Graceful shutdown / readiness / replicas |
| Data loss after redeploy | State in writable layer |
| Crash loop when the DB restarts | Retries/back-off; liveness depending on the DB |
| OOMKilled after traffic grows | Memory limit and heap sizing |
| Latency spikes at moderate CPU | CPU throttling; thread counts vs limits |
| DB "too many connections" after scaling out | Pool sizing × replicas |
| Logs missing or delayed | stdout + unbuffered |
| Wrong timestamps across services | UTC everywhere; host NTP |
| "Which version is running?" takes an hour | Version endpoint/labels, deploy by digest |
| Security audit findings | Non-root, caps, read-only, scanning |

---

## Quick recap

- Production containers are **disposable**: config and secrets from outside, state elsewhere, logs to stdout.
- **Start fast** with retries for dependencies. **Stop gracefully** on SIGTERM (fail readiness → drain → exit 0) as PID 1 / with tini.
- Separate **liveness** (cheap, no dependencies) from **readiness** (can serve traffic).
- Secure by default (non-root, caps dropped, no-new-privileges, read-only), with **limits** and runtimes sized from the cgroup.
- UTC, UTF-8, metrics/version/tracing, minimal pinned scanned images deployed by digest.

**Interview questions**

1. **Q: What makes an application "container-ready"?**
   A: External config and secrets, no local state (volumes or external services), logs to stdout, fast startup with dependency retries, graceful SIGTERM handling as PID 1, liveness and readiness endpoints, non-root with a read-only filesystem, resource limits with runtime sizing, and observability (metrics, version, traces).

2. **Q: What's the difference between liveness and readiness checks?**
   A: Liveness answers "is the process working, or should it be restarted?" It should be cheap and not depend on other services. Readiness answers "should traffic be sent here right now?" It can fail during startup, overload, dependency loss and shutdown without triggering a restart.

3. **Q: Why should containers use UTC?**
   A: Consistent timestamps across hosts, regions and services make logs and data easy to correlate. Minimal images often lack time-zone data anyway. Convert to local time only for display.
