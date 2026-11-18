# 08 · The Container Lifecycle

> **Level:** ★ foundation
>
> **You will learn:** every state a container can be in (created, running, paused, restarting, exited, dead), the commands that move it between states, what each one does to the Linux process underneath, and how to read exit codes like 0, 1, 125, 126, 127, 137 and 143.
>
> **You need:** topics 02–07. From your Linux notes: processes, signals, `kill`.

---

## What it is

A container has a **life**: it's created, started, maybe paused, stopped, started again, and finally removed. Docker tracks this as a **state**.

```
                docker create
                     │
                     ▼
               ┌──────────┐   docker start   ┌──────────┐
               │ CREATED  │ ───────────────► │ RUNNING  │◄──────────┐
               └──────────┘                  └──────────┘           │
                                            │   │   ▲   │           │ restart policy
                          docker pause      │   │   │   │ main       │ (topic 32)
                         ┌──────────────────┘   │   │   │ process   ┌┴───────────┐
                         ▼                      │   │   │ exits /   │ RESTARTING │
                   ┌──────────┐ docker unpause  │   │   │ stop/kill └────────────┘
                   │  PAUSED  │ ────────────────┘   │   ▼                ▲
                   └──────────┘                     │ ┌──────────┐       │
                                       docker start │ │  EXITED  │───────┘
                                                    └─┤          │
                                                      └──────────┘
                                                          │ docker rm
                                                          ▼
                                                       (gone)
```

Plus two rare states: **removing** (being deleted right now) and **dead** (removal failed halfway, usually due to a busy mount).

**The golden rule:** a container is **running exactly as long as its main process (PID 1) is running**. When PID 1 exits, for any reason, the container is exited.

---

## Why it exists

Separating "create", "start", "stop" and "remove" lets you:
- prepare a container and start it later (`create` then `start`),
- stop it without losing its writable layer and logs (to debug why it died),
- restart it with the same settings,
- automatically restart crashed apps (restart policies, topic 32),
- freeze it temporarily (pause).

**Exit codes** let scripts, CI pipelines, Docker's restart policies and Kubernetes know whether the app finished successfully or crashed, and *how* it crashed.

---

## Analogy

A **rental car**:
- **created** = the car is assigned to you and parked, engine off.
- **running** = you're driving.
- **paused** = stopped at a red light, engine on, frozen in place.
- **stopped (exited)** = parked and engine off. Your stuff is still in the car.
- **removed** = returned to the rental company. Anything you left inside is gone.
- **exit code** = the report you file when you return it: "0 = all fine", "137 = the engine was forcibly shut off".

---

## How it works inside

What each command does to the real Linux process and resources:

| Command | State change | What happens underneath |
|---|---|---|
| `docker create` | → created | Writable layer prepared, config saved in `/var/lib/docker/containers/<id>/`. **No process yet.** No namespaces yet. |
| `docker start` | created/exited → running | containerd + shim + runc create the namespaces, cgroup, overlay mount, network, then exec the main process (topic 03). |
| `docker run` | → running | `create` + `start` (+ pull + attach). |
| `docker stop` | running → exited | Sends the **stop signal** (default **SIGTERM**) to PID 1. Waits up to **10 seconds**. If still alive, sends **SIGKILL**. |
| `docker kill` | running → exited | Sends **SIGKILL** immediately (or another signal with `-s`). |
| `docker pause` | running → paused | Uses the **cgroup freezer**: writes `1` to the container's `cgroup.freeze` file. Every process is frozen. The processes stay in memory but get no CPU time. |
| `docker unpause` | paused → running | Writes `0` to `cgroup.freeze`. |
| `docker restart` | running → running | `stop` + `start`. **Same container**: same ID, same writable layer, but a **new process** (new host PID) and possibly a new IP. |
| `docker rm` | exited/created → gone | Deletes the writable layer, logs, config, and the container's network endpoint. |
| `docker rm -f` | running → gone | `kill` (SIGKILL) + `rm`. |

> **Signal recap (Linux):** a signal is a small message the kernel delivers to a process. **SIGTERM** (number 15) means "please shut down cleanly"; a program can catch it, finish its work and exit. **SIGKILL** (9) can't be caught: the kernel just ends the process. **SIGINT** (2) is what Ctrl+C sends. Topic 12 goes deep on signals and why PID 1 handles them differently.

### Exit codes: how a container reports how it died

When PID 1 ends, Docker records its **exit code**:

- If the program called `exit(n)`, the exit code is `n` (0–255).
- If a **signal** killed it, the exit code is **128 + the signal number**. This is a shell convention that Docker follows.

| Code | Meaning | Typical cause |
|---|---|---|
| `0` | Success | The program finished its job normally |
| `1` | General error | The app crashed or reported an error (read its logs) |
| `2` | Misuse / bad arguments | Wrong command-line flags to the app, shell syntax error |
| `125` | **Docker itself failed** | Bad `docker run` flag, unknown option, daemon error. The container never ran. |
| `126` | **Command can't be executed** | The file exists but isn't executable (missing `chmod +x`), or it's a folder |
| `127` | **Command not found** | Typo, or the program isn't in this image (e.g. `bash` in Alpine) |
| `130` | 128 + 2 = **SIGINT** | Someone pressed Ctrl+C |
| `137` | 128 + 9 = **SIGKILL** | **OOM kill** (out of memory), `docker kill`, or `docker stop` timed out after 10 s |
| `139` | 128 + 11 = **SIGSEGV** | Segmentation fault: the program read invalid memory (a bug, or a library mismatch, e.g. glibc vs musl) |
| `143` | 128 + 15 = **SIGTERM** | Stopped with `docker stop` and exited because of the signal (normal for many apps) |

**137 is ambiguous.** It could be the OOM killer or a normal stop that took too long. Check the `OOMKilled` flag:

```bash
docker inspect --format '{{.State.OOMKilled}} {{.State.ExitCode}}' myapp
```

`true 137` = memory limit hit. `false 137` = killed by a person or by `docker stop`'s timeout. (Memory limits in depth: topic 31.)

### Where Docker keeps a container's state

Each container has a folder:

```
/var/lib/docker/containers/<full-container-id>/
├── config.v2.json        ← the container's full configuration and current state
├── hostconfig.json       ← runtime settings (ports, mounts, limits, restart policy)
├── <id>-json.log         ← its logs (with the default json-file log driver; topic 33)
├── hostname              ← mounted at /etc/hostname inside
├── hosts                 ← mounted at /etc/hosts inside
├── resolv.conf           ← mounted at /etc/resolv.conf inside
└── mounts/               ← e.g. /dev/shm
```

This is why a stopped container still takes disk space and still has its logs.

---

## Commands / files

### create, then start

```bash
docker create --name lc nginx:1.27-alpine
docker ps -a --filter name=lc --format 'table {{.Names}}\t{{.Status}}'
```

```
3e9a7c0f2b1d8e6f5a4c3b2a1908f7e6d5c4b3a2918f7e6d5c4b3a2918f7e6d5
NAMES     STATUS
lc        Created
```

- `docker create` prints the full container ID.
- `Created` = configured, no process.

```bash
docker start lc
docker ps --filter name=lc --format 'table {{.Names}}\t{{.Status}}'
```

```
lc
NAMES     STATUS
lc        Up 3 seconds
```

### pause and unpause (the cgroup freezer)

```bash
docker pause lc
docker ps --filter name=lc --format '{{.Status}}'
CID=$(docker inspect -f '{{.Id}}' lc)
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cgroup.freeze
ps -o pid,stat,cmd -p $(docker inspect -f '{{.State.Pid}}' lc)
```

```
Up 20 seconds (Paused)
1
    PID STAT CMD
  73112 S    nginx: master process nginx -g daemon off;
```

- `cgroup.freeze` = `1` → the whole cgroup is frozen.
- The process still exists (same PID) but won't run until unpaused. Network connections to it just hang.
- (If the cgroup path doesn't exist, your Docker uses the `cgroupfs` driver instead of `systemd`. Run `cat /proc/<pid>/cgroup` to find the real path. Topic 11 shows this.)

```bash
docker unpause lc
```

### stop: SIGTERM, wait, then SIGKILL

```bash
time docker stop lc
docker ps -a --filter name=lc --format '{{.Status}}'
```

```
lc

real    0m0.412s
Exited (0) 2 seconds ago
```

- nginx handled the signal (its image sets `STOPSIGNAL SIGQUIT`, a graceful-quit signal for nginx) and exited quickly with code 0.

A program that ignores SIGTERM:

```bash
docker run -d --name stubborn alpine sleep 1000
time docker stop stubborn
docker ps -a --filter name=stubborn --format '{{.Status}}'
```

```
stubborn

real    0m10.334s
Exited (137) 1 second ago
```

- **10 seconds**, then `137`. `sleep` running as PID 1 doesn't react to SIGTERM (topic 12 explains why PID 1 is special), so Docker waited 10 s and sent SIGKILL.

Change the timeout:

```bash
docker stop -t 30 myapp       # wait up to 30 s before SIGKILL (also: --timeout)
docker run --stop-timeout 30 ...   # set the default for this container
```

### kill

```bash
docker kill stubborn               # SIGKILL now
docker kill -s SIGHUP myproxy      # send any signal, e.g. SIGHUP to reload config
```

### restart: same container, new process

```bash
docker start stubborn
docker inspect -f '{{.State.Pid}} {{.NetworkSettings.IPAddress}}' stubborn
docker restart -t 1 stubborn
docker inspect -f '{{.State.Pid}} {{.NetworkSettings.IPAddress}}' stubborn
```

```
73501 172.17.0.3
73622 172.17.0.3
```

- New host PID: a brand new process. The IP *may* change on restart, especially on the default bridge, so **never hard-code container IPs** (use names, topic 29).

### wait: block until a container exits, print its exit code

```bash
docker run -d --name job alpine sh -c 'sleep 3; exit 7'
docker wait job
```

```
7
```

- Useful in scripts: start something with `-d`, do other work, then `docker wait` for the result.

### rename and update

```bash
docker rename job batch-job-2026-11-17
docker update --memory 256m --memory-swap 256m --cpus 0.5 lc
docker update --restart unless-stopped lc
```

- `rename` changes only the name; the ID stays.
- `update` changes **resource limits** and the **restart policy** of an existing (even running) container. Docker writes the new numbers into the container's cgroup files on the spot. Most other settings (ports, volumes, env vars) **can't** be changed. You must recreate the container.

### rm

```bash
docker rm lc
docker rm stubborn
docker rm -f $(docker ps -aq --filter name=batch-job)
```

```
Error response from daemon: cannot remove container "/lc": container is running: stop the container before removing or force remove
```

- If you see that error, the container is still running. Stop it, or use `-f`.
- `docker ps -aq` = IDs only (`-q`) of all containers (`-a`). Combined with `--filter`, it's a common scripting pattern.

### See the state as data

```bash
docker inspect -f '{{json .State}}' job | python3 -m json.tool
```

```json
{
    "Status": "exited",
    "Running": false,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 0,
    "ExitCode": 7,
    "Error": "",
    "StartedAt": "2026-11-17T06:10:01.123456789Z",
    "FinishedAt": "2026-11-17T06:10:04.130000000Z"
}
```

- `Pid: 0` = no process (it's not running).
- `ExitCode`, `OOMKilled`, `StartedAt`/`FinishedAt` = the first things to check when a container died.
- `Error` = filled if Docker couldn't even start it (e.g. "executable file not found").

---

## Hands-on lab

1. Make each famous exit code yourself:
   ```bash
   docker run --rm alpine sh -c 'exit 0';   echo $?
   docker run --rm alpine sh -c 'exit 1';   echo $?
   docker run --rm alpine notacommand;      echo $?
   docker run --rm alpine /etc/hosts;       echo $?
   docker run --rm --notaflag alpine;       echo $?
   ```
   ```
   0
   1
   docker: Error response from daemon: failed to create task for container: ... exec: "notacommand": executable file not found in $PATH
   127
   docker: Error response from daemon: failed to create task for container: ... exec: "/etc/hosts": permission denied
   126
   unknown flag: --notaflag
   125
   ```
   - 127: not found. 126: found but not executable. 125: Docker itself rejected the command.

2. Make a 137 by OOM and tell it apart from a stop. (`--memory 20m` caps the container's RAM at 20 MB using its cgroup's `memory.max` file from topic 02. Topic 31 covers limits fully.)
   ```bash
   docker run --name oomtest --memory 20m alpine sh -c 'tail /dev/zero'
   docker inspect -f 'OOMKilled={{.State.OOMKilled}} Exit={{.State.ExitCode}}' oomtest
   ```
   ```
   OOMKilled=true Exit=137
   ```
   (`tail /dev/zero` keeps reading an endless stream of zero bytes with no newline, so its memory grows until the 20 MB limit kills it.)

3. Make a 143:
   ```bash
   docker run -d --name t143 --init alpine sleep 1000
   docker stop t143
   docker inspect -f '{{.State.ExitCode}}' t143
   ```
   ```
   143
   ```
   With `--init`, a tiny init process runs as PID 1 and passes SIGTERM to `sleep`. `sleep` then dies from SIGTERM → 128+15. (Topic 12 explains `--init`.)

4. Watch the states change live: run `docker events --filter type=container` in a second terminal while you create, start, pause, unpause, stop, start and rm a container.

5. Clean up: `docker rm -f oomtest t143 2>/dev/null`

---

## Where you'll use it

- **Incident triage:** the first question for a dead container is "what was the exit code, and was it OOMKilled?"
- **Kubernetes:** the same exit codes appear in `kubectl describe pod` under "Last State: Terminated, Reason: OOMKilled, Exit Code: 137".
- **CI/CD:** test containers' exit codes pass or fail pipelines.
- **Deployments:** `docker stop` timeouts matter for graceful shutdown during deploys (topic 12).
- **Live tuning:** `docker update` to raise a memory limit on a running container during an incident, before a proper fix.

---

## When it breaks

| Symptom | Meaning | What to check |
|---|---|---|
| Container is `Exited (0)` right after start, and it should keep running | The main process finished: e.g. it ran in the background (a daemon that forks) or the command was a one-shot | Run the app in the **foreground** (e.g. `nginx -g 'daemon off;'`). A container needs a long-running PID 1. |
| `Exited (1)` / `Exited (2)` | The app failed | `docker logs <name>` |
| `Exited (137)` and `OOMKilled=true` | Hit its memory limit | Raise the limit or fix the leak (topic 31). Also check `dmesg \| grep -i oom` on the host. |
| `Exited (137)` and `OOMKilled=false` | Killed: `docker kill`, `docker stop` timeout, or host-level OOM killer in some setups | Did someone stop it? Did the app ignore SIGTERM for 10 s (topic 12)? |
| `Exited (139)` | Segfault | App bug, or a binary built for glibc running on Alpine's musl (topic 19) |
| `Exited (126)` | Not executable | `chmod +x` the script in the image; check the script's line endings (Windows `\r\n` breaks the `#!` line) |
| `Exited (127)` | Command not found | Wrong `CMD`/`ENTRYPOINT`, missing package, `bash` not in the image, wrong `PATH` |
| `Restarting (1) 5 seconds ago` forever | Restart policy + an app that keeps crashing (a "restart loop") | `docker logs`, exit code (topic 32, 50) |
| `docker stop` always takes exactly 10 s | PID 1 ignores SIGTERM | Topic 12 (exec form, `--init`, signal handling) |
| `Error response from daemon: cannot stop container: ... permission denied` | Often AppArmor/SELinux blocking signals (seen with snap-installed Docker or restrictive profiles) | `dmesg`, `journalctl -u docker`; check AppArmor (topic 41a) |
| State is `Dead` | Docker couldn't remove the container (usually a mount still busy) | `docker rm -f`; find what holds the mount (`grep <id> /proc/*/mountinfo`); restart docker as last resort |

---

## Quick recap

- States: **created → running ⇄ paused → exited → removed** (+ restarting, removing, dead).
- **A container runs exactly as long as its PID 1 runs.**
- `stop` = SIGTERM, wait 10 s, then SIGKILL. `kill` = SIGKILL now. `pause` = cgroup freezer.
- Exit code = the app's `exit(n)`, or **128 + signal number** if killed by a signal: 137 = SIGKILL (check `OOMKilled`), 143 = SIGTERM, 139 = segfault. 125/126/127 = Docker failure / not executable / not found.
- `restart` keeps the container (same writable layer) but starts a new process. `rm` deletes everything not on a volume.

**Interview questions**

1. **Q: A container exits with code 137. What does it mean, and how do you find the cause?**
   A: 137 = 128 + 9: the process got SIGKILL. Check `docker inspect` for `State.OOMKilled`. If it's true, the memory limit was hit. If not, someone ran `docker kill`, or `docker stop` timed out because the app ignored SIGTERM.

2. **Q: What's the difference between `docker stop` and `docker kill`?**
   A: `stop` sends the stop signal (SIGTERM by default), waits a grace period (10 s), then sends SIGKILL. `kill` sends SIGKILL (or a chosen signal) immediately, with no grace period.

3. **Q: Why does my container exit immediately after `docker run -d`?**
   A: Its main process finished. Either the command was short-lived, or the program sent itself to the background (daemonized), so PID 1 exited. Run the program in the foreground.
