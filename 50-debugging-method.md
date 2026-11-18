# 50 · A Debugging Method for Containers

> **Level:** ★★ intermediate
>
> **You will learn:** one repeatable method for any container problem, and specific decision paths for the six classic symptoms: **won't start**, **exits immediately**, **restart loop**, **can't connect**, **slow**, and **OOM-killed**. Each path names the exact commands to run and what each result means, linking back to the topic where the mechanism was explained.
>
> **You need:** everything so far. This topic is the index of your debugging knowledge.

---

## What it is

A **method** = a fixed order of questions that narrows the problem from "something is broken" to one layer: **image → container config → process → resources → storage → network → daemon → host**.

The first 60 seconds, for **any** container incident:

```bash
docker ps -a --filter name=<c>                                  # state, status, restarts, ports
docker inspect <c> -f 'Status={{.State.Status}} Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Restarts={{.RestartCount}} Error={{.State.Error}} Started={{.State.StartedAt}} Finished={{.State.FinishedAt}} Health={{if .State.Health}}{{.State.Health.Status}}{{end}}'
docker logs --tail 100 -t <c>                                   # what the app said (stdout/stderr)
docker events --since 30m --filter container=<c>                # what happened to it
```

Those four commands answer "what state, why did it end, what did it say, what happened" in most cases.

---

## Why it exists

Under pressure, people guess: restart things, rebuild images, change random settings. Each guess costs minutes and can destroy evidence (a removed container takes its logs and writable layer with it). A method is faster and doesn't destroy evidence.

**Golden rule: don't delete the broken container until you've collected evidence.** `docker logs`, `docker inspect` and `docker cp` work on stopped containers. If you need the service back immediately, **start a new container with a different name** and keep the broken one for investigation.

---

## Analogy

A **doctor's diagnosis**: first the vital signs (state, exit code, logs: 60 seconds), then the symptom-specific questions ("where does it hurt?"), then targeted tests. The doctor doesn't operate first and ask questions later.

---

## How it works inside: the decision paths

### Symptom 1: the container won't start (`docker run` / `start` errors immediately)

The error comes from a specific layer (topic 03). Read **which** one:

| Error contains | Layer | Typical cause | Topic |
|---|---|---|---|
| `Cannot connect to the Docker daemon` / `permission denied ... docker.sock` | CLI → daemon | Daemon down, wrong context, not in docker group | 03, 04, 34 |
| `pull access denied`, `manifest unknown`, `toomanyrequests`, `no matching manifest` | Registry | Name/tag/auth/rate limit/platform | 06, 22 |
| `port is already allocated` / `address already in use` | Networking | Host port taken | 28 |
| `invalid mount config`, `bind source path does not exist` | Mounts | Wrong host path | 25 |
| `Conflict. The container name "/x" is already in use` | Naming | Old container still exists | 09 |
| `OCI runtime create failed: ... exec: "x": executable file not found in $PATH` | runc | Wrong CMD/ENTRYPOINT; missing binary | 08, 15 |
| `... exec format error` | runc/kernel | Wrong CPU architecture; bad shebang | 21, 15 |
| `... permission denied` (exec) | runc | Not executable (exit 126) | 08, 15 |
| `no such file or directory` for an existing file | runc/loader | Dynamic binary missing its loader (glibc vs musl); CRLF shebang | 17, 19, 15 |
| `unable to find user x` | runc | `-u name` not in the image | 09 |
| `privileged mode is incompatible with user namespaces` | Daemon config | userns-remap | 42 |
| `no space left on device` | Storage | Disk/inodes full | 48 |

### Symptom 2: it starts, then exits immediately

```bash
docker inspect <c> -f 'Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Error={{.State.Error}}'
docker logs <c>
docker inspect <c> -f 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} User={{.Config.User}} WorkDir={{.Config.WorkingDir}}'
```

| Finding | Meaning | Topic |
|---|---|---|
| Exit 0, no error | Main process finished: it's a one-shot command, or the program daemonised (went to the background) | 08 |
| Exit 1/2 + error in logs | App error: config missing, can't reach DB, bad arguments | 45 |
| Exit 126 / 127 | Not executable / not found | 08 |
| Exit 137 + `OOM=true` | Memory limit | 31 |
| Exit 139 | Segfault: libc mismatch or app bug | 19 |
| Logs empty | App crashed before logging, logs to a file, or output is buffered | 33 |

Reproduce interactively with the **same image and config**, but a shell instead of the app:

```bash
docker run --rm -it --entrypoint sh <same image> 
# then run the original entrypoint/cmd by hand and watch it fail
docker run --rm -it --entrypoint sh -e ...same env... -v ...same mounts... <image>
```

### Symptom 3: restart loop (`Restarting (1) …`, RestartCount climbing)

```bash
docker inspect <c> -f 'Restarts={{.RestartCount}} Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Policy={{.HostConfig.RestartPolicy.Name}}'
docker logs --tail 200 <c>                 # includes output from previous runs (same container, same log file)
docker events --since 15m --filter container=<c> --filter event=die --format '{{.Time}} exit={{.Actor.Attributes.exitCode}}'
```

- Same exit code every time → same cause each run (config, dependency, OOM).
- To stop the loop while investigating without losing the container: `docker update --restart no <c>`. Then `docker start -a <c>` to watch one attempt live.
- Common causes: a dependency not ready (topic 37), a migration failing, a bad config/secret, OOM at startup (topic 31), a health-check-driven external restarter, port/permission errors on volumes (topic 25).

### Symptom 4: can't connect (to it, or from it)

Follow topic 30's path exactly:
1. Classify: **refused** / **timeout** / **no route** / **reset** / **DNS failure**.
2. Is the app listening on **`0.0.0.0`** and the right port? → `nsenter -t PID -n ss -lntp`.
3. Works inside its netns? → `nsenter -t PID -n curl 127.0.0.1:PORT`.
4. Works from a peer on the same network (by name)? → netshoot.
5. Published correctly? → `docker port`, `ss -lntp` on the host, DNAT counters.
6. From outside: security groups / `DOCKER-USER` / bound to 127.0.0.1?
7. Outbound: route → gateway → NAT → DNS (`ExtServers`) → proxy → **MTU**.

### Symptom 5: slow

| Check | Command | Meaning if bad | Topic |
|---|---|---|---|
| CPU throttling | `cat $CG/cpu.stat` (nr_throttled) | Latency spikes from `--cpus` | 31 |
| Memory pressure | `cat $CG/memory.stat`, `memory.events` (`high`, `max`), swap use | Reclaim/swapping slows everything | 31 |
| Host saturation | `top`, `vmstat 1`, `iostat -x 1` on the host | Noisy neighbours, disk I/O wait | 31, 47 |
| Writable-layer I/O | `docker ps -s`, `docker diff` | Heavy writes through overlay/copy-up | 07, 23 |
| Log driver blocking | Remote log driver in blocking mode, `dockerd` CPU high | App writes block | 33 |
| DNS latency | `dig` timing inside; `ndots`/search domains | Every outgoing call slow | 29 |
| Network path | MTU black holes, conntrack full (`dmesg`) | Hangs/retries | 30 |
| Runtime sizing | Threads vs `cpu.max`; JVM heap/GC | Thread contention, GC storms | 31 |
| The app itself | Profilers, `docker exec` / `nsenter` + `strace -c`, `perf top -p PID` | App code | 51 |

### Symptom 6: OOM-killed

```bash
docker inspect <c> -f 'OOM={{.State.OOMKilled}} Exit={{.State.ExitCode}} Limit={{.HostConfig.Memory}}'
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$(docker inspect -f '{{.State.Pid}}' <c>)/cgroup)   # if running
cat $CG/memory.events; grep -E '^(anon|file|shmem|kernel) ' $CG/memory.stat
sudo dmesg -T | grep -iE 'oom|killed process' | tail -5
```

Decide:
- **anon** (heap) growing steadily → leak, or limit too low for the real working set. Profile, or raise the limit.
- **shmem** large → tmpfs or `/dev/shm` usage (topic 25).
- **file** large → page cache; usually reclaimable, rarely the OOM cause unless it's dirty pages under heavy writes.
- Runtime heap sized above the limit → JVM/Node/Go settings (topic 31).
- Container still "Up" but `oom_kill > 0` → child processes being killed (topic 31).
- `dmesg` shows a **global** (not cgroup) OOM → the **host** ran out of memory (containers without limits).

### When the daemon itself is the problem

```bash
systemctl status docker containerd --no-pager
journalctl -u docker -n 100 --no-pager
journalctl -u containerd -n 50 --no-pager
docker info 2>&1 | tail -20
df -h /var/lib/docker; df -i /var/lib/docker
```

- `docker` commands hang → dockerd is stuck: full disk, a stuck volume driver/NFS mount, a huge number of containers, or a deadlock. Get a goroutine dump before restarting: `sudo kill -SIGUSR1 $(pidof dockerd)` writes a stack dump file (its path is printed in the dockerd log).
- Containers keep running (shims) even if dockerd is broken. Inspect them with host tools (topic 11).

---

## Commands / files

### An evidence-collection script (run before you "fix" anything)

```bash
#!/usr/bin/env bash
# collect.sh <container> — saves evidence to ./evidence-<container>-<time>/
set -u
C="$1"; D="evidence-$C-$(date +%Y%m%d-%H%M%S)"; mkdir -p "$D"
docker inspect "$C"                      > "$D/inspect.json"   2>&1
docker logs -t "$C"                      > "$D/logs.txt"       2>&1
docker events --since 2h --until 0s --filter container="$C" > "$D/events.txt" 2>&1
docker diff "$C"                         > "$D/diff.txt"       2>&1
docker ps -a -s --filter name="$C"       > "$D/ps.txt"         2>&1
docker info                              > "$D/docker-info.txt" 2>&1
PID=$(docker inspect -f '{{.State.Pid}}' "$C" 2>/dev/null)
if [ -n "$PID" ] && [ "$PID" != "0" ]; then
  CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
  for f in memory.current memory.max memory.events memory.stat cpu.max cpu.stat pids.current pids.max; do
    echo "== $f"; cat "$CG/$f"; done > "$D/cgroup.txt" 2>&1
  sudo nsenter -t "$PID" -n ss -tanp > "$D/sockets.txt" 2>&1
  sudo nsenter -t "$PID" -n ip addr   > "$D/ipaddr.txt"  2>&1
fi
sudo dmesg -T | tail -200 > "$D/dmesg.txt" 2>&1
journalctl -u docker --since "2 hours ago" --no-pager > "$D/dockerd.log" 2>&1
echo "Evidence saved in $D"
```

- `--until 0s` makes `docker events` print past events and exit instead of streaming forever.

### Reproduce a container's exact config with a shell

```bash
docker inspect <c> -f '{{.Config.Image}}'
docker inspect <c> -f '{{range .Config.Env}}-e {{printf "%q" .}} {{end}}'
docker inspect <c> -f '{{range .Mounts}}-v {{.Source}}:{{.Destination}} {{end}}'
# build a docker run --rm -it --entrypoint sh ... command from these
```

---

## Hands-on lab: six broken containers

Run each, then diagnose it **with the method only** (no peeking at the command):

```bash
# 1. won't start
docker run -d --name b1 -p 8080:80 nginx:1.27-alpine; docker run -d --name b1b -p 8080:80 nginx:1.27-alpine
# 2. exits immediately
docker run -d --name b2 python:3.13-slim python -c "import os; os.environ['DATABASE_URL']"
# 3. restart loop
docker run -d --name b3 --restart always alpine sh -c 'echo "config missing"; exit 3'
# 4. can't connect
docker run -d --name b4 -p 8094:8000 python:3.13-slim python -m http.server 8000 --bind 127.0.0.1
# 5. slow
docker run -d --name b5 --cpus 0.1 python:3.13-slim python -c "
import time
while True:
    t=time.time(); sum(i*i for i in range(2_000_000)); print('took', round(time.time()-t,2), flush=True)"
# 6. OOM
docker run -d --name b6 --restart on-failure:2 --memory 64m python:3.13-slim python -c "x=bytearray(200*1024*1024)"
```

Expected diagnoses:
1. `port is already allocated` → port conflict.
2. Exit 1, `KeyError: 'DATABASE_URL'` in logs → missing config.
3. `Restarting`, RestartCount rising, exit 3, "config missing" → app config error in a loop.
4. `curl localhost:8094` fails, `nsenter ... ss -lntp` shows `127.0.0.1:8000` → wrong bind address.
5. Logs show slow iterations; `cpu.stat` shows heavy throttling → CPU limit.
6. `OOMKilled=true`, exit 137, `dmesg` OOM line, RestartCount 2 → memory limit.

Clean up: `docker rm -f b1 b1b b2 b3 b4 b5 b6`

---

## Where you'll use it

- **On-call** for any Docker/Compose environment.
- **Kubernetes:** the same paths. `kubectl describe pod` (events, last state, exit code, OOMKilled), `kubectl logs --previous`, `kubectl debug`, and node-level `crictl`/`nsenter` (topic 53).
- **Interviews:** "A container keeps restarting, how do you debug it?" is one of the most common practical questions.

---

## When it breaks (debugging the debugging)

| Problem | Fix |
|---|---|
| Container removed before you could look (e.g. `--rm`, a CI cleanup) | Ship logs centrally; drop `--rm` while debugging; `docker events` history from monitoring |
| `docker logs` empty | App logs to files: `docker cp` them out, or check `/proc/PID/root/...` |
| `exec` impossible (no shell / stopped) | `docker cp`, `nsenter`, `docker debug`, a debug sidecar (topic 51) |
| Too many moving parts in Compose | `docker compose ps -a`, `docker compose logs --since 10m`, check `depends_on` and health (topic 37) |
| Can't reproduce locally | Same image digest, same env/mounts (from `inspect`), same limits, same platform |

---

## Quick recap

- First 60 seconds: `docker ps -a`, `docker inspect` (state, exit code, OOMKilled, restarts, error, health), `docker logs --tail`, `docker events`.
- **Keep the evidence**: don't remove broken containers. Collect inspect/logs/events/diff/cgroup/sockets/dmesg/journal first.
- Map the symptom to a path: won't start → read which layer errored; exits → exit code + logs + reproduce with `--entrypoint sh`; loop → same exit each time, stop the loop with `docker update --restart no`; can't connect → topic 30's hop-by-hop path; slow → throttling/memory/I/O/DNS/MTU; OOM → `memory.stat` breakdown + `dmesg`.
- Daemon problems → `journalctl -u docker/containerd`, disk, SIGUSR1 stack dump. Containers survive via shims.

**Interview questions**

1. **Q: A container is in a restart loop. Walk me through your debugging.**
   A: `docker ps` and `docker inspect` for RestartCount, ExitCode and OOMKilled, then `docker logs` (which includes previous attempts) and `docker events` for the die events. If the exit code is constant, it's a deterministic cause: config, dependency, permission, OOM. Pause the loop with `docker update --restart no`, reproduce with the same image and config and `--entrypoint sh`, fix the cause, and restore the policy.

2. **Q: How do you debug a container that exits immediately with code 0?**
   A: Its main process finished. Check the effective entrypoint/cmd in `docker inspect`. Usually it's a one-shot command or a program that daemonises into the background. Run the program in the foreground as PID 1.

3. **Q: What evidence do you collect before restarting a broken container?**
   A: `docker inspect` (state and config), logs with timestamps, recent events, `docker diff`, the cgroup stats (memory.events, cpu.stat), open sockets from its network namespace, `dmesg`, and dockerd/containerd journal logs, so the root cause can still be found after service is restored.
