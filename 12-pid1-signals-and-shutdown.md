# 12 · PID 1, Signals and Graceful Shutdown

> **Level:** ★★ intermediate
>
> **You will learn:** why `docker stop` sometimes takes exactly 10 seconds, why PID 1 in a container ignores signals other processes obey, the difference between "shell form" and "exec form" commands, what zombie processes are and why they pile up in containers, how `--init` (tini) fixes both problems, and how to design a clean shutdown.
>
> **You need:** topic 08 (stop/kill, exit codes 137/143), topic 11 (`NSpid`, host view). From your Linux notes: signals, `kill`, parent/child processes, `systemd` as PID 1.

---

## What it is

Three connected ideas:

1. **Signals**: small messages the kernel delivers to a process ("please stop", "stop now", "reload"). `docker stop` works by sending signals.
2. **PID 1**: the first process in a PID namespace. Linux gives PID 1 **special treatment**: it is protected from signals it hasn't asked to handle, and it adopts orphaned processes. In a container, **your app is PID 1**, and most apps weren't written to be PID 1.
3. **Graceful shutdown**: on a stop request, the app finishes in-flight work, closes connections and exits by itself, **before** Docker loses patience and kills it.

---

## Why it exists

Every deploy, scale-down, node drain and restart **stops containers**. If the app doesn't shut down gracefully:
- users get errors (requests cut in half, connections reset),
- jobs are interrupted halfway (half-written files, half-processed messages),
- every stop takes the full timeout (10 s in Docker, 30 s in Kubernetes), so deploys are slow,
- the app gets SIGKILL and exits with 137, which looks like a crash in your monitoring.

And if PID 1 doesn't reap zombie processes, they pile up until the container hits its process limit and nothing new can start.

---

## Analogy

**Closing time at a restaurant.**
- **SIGTERM** = the manager says "we're closing in 10 minutes". Good staff finish serving the current tables, stop seating new guests, clean up and leave.
- **SIGKILL** = the building's power is cut. Whatever was happening just stops, mid-meal.
- **PID 1** = the head chef who doesn't wear the earpiece. The manager's announcement never reaches them, so they keep cooking until the power is cut.
- **Zombies** = finished staff who never handed in their timesheets. They can't leave the system until someone (the head chef = PID 1) collects the sheets. If nobody does, the staff list fills up with ghosts, and no new staff can be hired.
- **tini (`--init`)** = a proper floor manager who listens to announcements, passes them to the chef, and collects every timesheet.

---

## How it works inside

### Signal recap

| Signal | Number | Default action | Can the app catch/ignore it? | Typical use |
|---|---|---|---|---|
| SIGTERM | 15 | terminate | Yes | "Please shut down" (`docker stop`, `kill`, Kubernetes) |
| SIGINT | 2 | terminate | Yes | Ctrl+C |
| SIGQUIT | 3 | terminate + core dump | Yes | Graceful quit for nginx; thread dump for Java |
| SIGHUP | 1 | terminate | Yes | Often "reload config" for daemons |
| SIGKILL | 9 | terminate | **No** | Forced kill. The kernel removes the process. |
| SIGCHLD | 17 | ignore | Yes | Sent to a parent when a child exits |

Programs can install a **signal handler** (code that runs when a signal arrives) for any signal except SIGKILL and SIGSTOP. With no handler, the **default action** applies, which for SIGTERM is "die".

### Why PID 1 is different

The kernel protects the init process of a namespace. For PID 1:

> **A signal is ignored if PID 1 has no handler for it** (when the signal comes from inside its namespace or from its parent namespace via normal `kill`). The default action ("die") is **not** applied.

The exception is SIGKILL (and SIGSTOP) sent from an **ancestor** namespace (the host), which always works. That's why `docker kill` always works.

So:
- A normal process with no SIGTERM handler → dies on SIGTERM (default action).
- **The same program as PID 1 in a container → SIGTERM is silently ignored.** `docker stop` waits 10 s, then SIGKILL → exit **137**.

Programs that install a SIGTERM handler (nginx, most web servers, Go/Java/Node apps that are written for it) are fine as PID 1. Simple scripts, `sleep`, and many naive apps are not.

### Shell form vs exec form

How you give the command decides **which program becomes PID 1**:

**Exec form**: the program is started directly:

```bash
docker run myimage python app.py
```

```
PID 1: python app.py        ← receives SIGTERM directly
```

**Shell form**: the command is wrapped in a shell:

```bash
docker run myimage sh -c "python app.py"
```

```
PID 1: /bin/sh -c "python app.py"
  └─ PID 7: python app.py   ← never receives SIGTERM
```

- `sh` is PID 1. It has no SIGTERM handler, so the signal is ignored (PID 1 rule).
- Even if it did die, `sh` **doesn't forward signals to its children**.
- Result: `docker stop` takes 10 s, and `python` is SIGKILLed with no chance to clean up.

> Some shells (bash, newer dash) quietly replace themselves with the command when it's a **single simple command**, so `sh -c "python app.py"` *might* end up with python as PID 1. Don't rely on this. With two commands (`cd /app && python app.py`) or a script, the shell stays PID 1.

In a Dockerfile (topic 14–15), the same two forms exist:

```dockerfile
CMD ["python", "app.py"]     # exec form  (JSON array) → python is PID 1
CMD python app.py            # shell form (plain text)  → /bin/sh -c "python app.py" is PID 1
```

**Always use exec form for the main process.**

### The entrypoint-script pattern: `exec "$@"`

Many images use a startup script to prepare things (wait for a database, render a config file) and then start the app. The script must **replace itself** with the app using `exec`:

```sh
#!/bin/sh
set -e
echo "preparing config..."
envsubst < /app/config.tmpl > /app/config.yaml
exec "$@"          # replace this shell with the real command; the app becomes PID 1
```

- `exec` = a shell built-in that runs the `execve()` system call: the shell process **becomes** the new program (same PID). No shell is left.
- `"$@"` = all the arguments passed to the script (the command), each kept as a separate word.
- Without `exec`, the shell stays as PID 1 and swallows signals.

### Zombies and reaping

**Linux recap:**
- When a process exits, it doesn't vanish immediately. It becomes a **zombie** (state `Z`, shown as `<defunct>`): just an entry in the process table holding its exit code, until its **parent** reads that code with the `wait()` system call. Reading it is called **reaping**.
- If a parent dies first, its children become **orphans** and are **re-parented to PID 1** of their PID namespace. On a normal host, `systemd` reaps them.
- **In a container, PID 1 is your app.** If your app spawns processes (shell scripts, `subprocess` calls, health check commands, browsers like headless Chrome), and some get orphaned, they're adopted by your app, which probably never calls `wait()` for children it didn't create.

Result: zombies pile up. They use no CPU or memory, but each takes a **PID**. Once `pids.max` (or the kernel's global limit) is hit: `fork: Resource temporarily unavailable`, and nothing new can start.

### The fix: a tiny init as PID 1 (`--init` / tini)

`docker run --init` puts **`docker-init`** (which is **tini**, a minimal init program) in as PID 1 and runs your command as its child:

```
PID 1: /sbin/docker-init -- python app.py
  └─ PID 7: python app.py
```

tini does exactly two jobs:
1. **Forwards signals** it receives (SIGTERM, SIGINT, …) to its child.
2. **Reaps zombies**: it `wait()`s for every orphan it adopts.

When the child exits, tini exits with the same code. Because the app is no longer PID 1, the default action (die on SIGTERM) works normally too, even if the app has no handler.

Other options: install `tini` or `dumb-init` in the image and use it as the entrypoint (common when you can't control `docker run` flags), or use `init: true` in Compose (topic 37). In Kubernetes there's no `--init` flag, so you bake tini into the image, or rely on the app handling signals properly.

### The full `docker stop` timeline

```
t=0s    docker stop web
        → dockerd sends the STOP SIGNAL to PID 1 of the container
          (default SIGTERM; images can change it with STOPSIGNAL; you can with --stop-signal)
t=0..T  app should: stop accepting new work → finish in-flight requests → close DB connections
        → flush buffers → exit(0)
t=T     if PID 1 is still alive after the timeout T (default 10 s; --stop-timeout, docker stop -t)
        → dockerd sends SIGKILL → exit code 137
After   PID 1 is gone → the kernel kills every other process in the PID namespace with SIGKILL
        → container state = exited
```

Important detail: **only PID 1 gets the stop signal.** Other processes in the container get SIGKILL when PID 1 exits. If you run several processes, PID 1 must pass the signal on (that's another reason one-process-per-container is the norm).

### What a well-behaved app does on SIGTERM

1. Stop accepting **new** connections or jobs (and fail its readiness check, so load balancers stop sending traffic; topics 32, 53).
2. Finish in-flight requests, up to a deadline shorter than the stop timeout.
3. Close database connections, flush logs and buffers, commit or release queue messages.
4. Exit with code 0.

---

## Commands / files

### See who is PID 1

```bash
docker run -d --name shellform alpine sh -c 'sleep 1000 & wait'
docker run -d --name execform alpine sleep 1000
docker top shellform -o pid,ppid,args
docker exec shellform ps
```

```
PID     PPID    COMMAND
95100   95080   sh -c sleep 1000 & wait
95130   95100   sleep 1000
PID   USER     TIME  COMMAND
    1 root      0:00 sh -c sleep 1000 & wait
    7 root      0:00 sleep 1000
   8 root      0:00 ps
```

- `docker top -o` accepts `ps` column options.
- In `shellform`, `sh` is PID 1 and `sleep` is its child.

### Time the stop

```bash
time docker stop shellform
time docker stop execform
docker ps -a --filter name=form --format '{{.Names}} {{.Status}}'
```

```
real    0m10.4s
real    0m10.3s
shellform Exited (137) 5 seconds ago
execform Exited (137) 15 seconds ago
```

- Both took 10 s and were SIGKILLed: `sh` as PID 1 ignores SIGTERM, and `sleep` as PID 1 has no handler, so it ignores SIGTERM too.

### Fix 1: the app handles SIGTERM

```bash
docker run -d --name trapper alpine sh -c 'trap "echo got SIGTERM, cleaning up; exit 0" TERM; while true; do sleep 1; done'
time docker stop trapper
docker logs trapper
docker inspect -f '{{.State.ExitCode}}' trapper
```

```
real    0m0.9s
got SIGTERM, cleaning up
0
```

- `trap "..." TERM` = install a SIGTERM handler in the shell. Now PID 1 has a handler, so the signal is delivered. Fast stop, exit 0.
- (The shell runs the trap after the current `sleep 1` finishes, hence ~1 s.)

### Fix 2: `--init`

```bash
docker run -d --name inited --init alpine sleep 1000
docker top inited -o pid,ppid,args
time docker stop inited
docker inspect -f '{{.State.ExitCode}}' inited
```

```
PID     PPID    COMMAND
95400   95380   /sbin/docker-init -- sleep 1000
95420   95400   sleep 1000
real    0m0.4s
143
```

- `docker-init` is PID 1, `sleep` is its child (not PID 1 anymore, so the default "die on SIGTERM" applies).
- Exit `143` = 128 + 15: `sleep` died from SIGTERM, and tini passed that status on. Fast and clean.

### Change the stop signal and timeout

```bash
docker run -d --name w1 --stop-signal SIGQUIT --stop-timeout 30 nginx:1.27-alpine
docker inspect -f '{{.Config.StopSignal}} {{.Config.StopTimeout}}' w1
docker stop -t 5 w1           # override the timeout for this one stop
```

```
SIGQUIT 30
```

- The nginx image already sets `STOPSIGNAL SIGQUIT` (graceful quit for nginx).

### See zombies

Save this as `zombie.py`:

```python
import os, time
# Make 5 orphans: each child forks a grandchild and exits at once.
# The grandchild is re-parented to PID 1 and exits 1 s later.
for i in range(5):
    pid = os.fork()
    if pid == 0:                 # child
        if os.fork() == 0:       # grandchild
            time.sleep(1)
            os._exit(0)
        os._exit(0)              # child exits → grandchild is orphaned
    os.waitpid(pid, 0)           # we reap our own child only
print("spawned; now sleeping", flush=True)
time.sleep(3600)
```

```bash
docker run -d --name zomb -v "$PWD/zombie.py:/z.py" python:3.13-alpine python /z.py
sleep 3
docker top zomb -o pid,ppid,stat,args
```

```
PID     PPID    STAT   COMMAND
96001   95980   Ss     python /z.py
96020   96001   Z      [python] <defunct>
96021   96001   Z      [python] <defunct>
96022   96001   Z      [python] <defunct>
96023   96001   Z      [python] <defunct>
96024   96001   Z      [python] <defunct>
```

- `-v "$PWD/zombie.py:/z.py"` makes your file visible inside the container (a bind mount; topic 25).
- `STAT Z` and `<defunct>` = zombies. Their parent is python (PID 1 inside), which never `wait()`s for them.

With `--init`:

```bash
docker run -d --name zomb2 --init -v "$PWD/zombie.py:/z.py" python:3.13-alpine python /z.py
sleep 3
docker top zomb2 -o pid,ppid,stat,args
```

```
PID     PPID    STAT   COMMAND
96101   96080   Ss     /sbin/docker-init -- python /z.py
96120   96101   S      python /z.py
```

- No zombies. The orphans were adopted by tini, which reaped them.

---

## Hands-on lab

1. Run all the commands above and record stop time + exit code for: `shellform`, `execform`, `trapper`, `inited`.
2. **Entrypoint script with and without `exec`.** Create two scripts:
   ```bash
   cat > noexec.sh <<'EOF'
   #!/bin/sh
   echo "setup done"
   sleep 1000
   EOF
   cat > withexec.sh <<'EOF'
   #!/bin/sh
   echo "setup done"
   exec sleep 1000
   EOF
   chmod +x noexec.sh withexec.sh
   docker run -d --name s1 -v "$PWD/noexec.sh:/start.sh" alpine /start.sh
   docker run -d --name s2 --init -v "$PWD/withexec.sh:/start.sh" alpine /start.sh
   docker top s1 -o pid,args; docker top s2 -o pid,args
   time docker stop s1; time docker stop s2
   ```
   `s1`: the script's shell stays as PID 1 and holds `sleep` as a child (10 s stop). `s2`: the shell replaced itself with `sleep` (`exec`), and `--init` makes sure it receives SIGTERM (fast stop).
3. **Read the default stop signal of popular images:**
   ```bash
   for i in nginx:1.27-alpine postgres:17-alpine redis:7-alpine httpd:2.4-alpine; do
     docker pull -q $i >/dev/null; echo "$i → $(docker image inspect -f '{{.Config.StopSignal}}' $i)"; done
   ```
   (Empty = SIGTERM. You'll see `SIGQUIT` for nginx, `SIGINT` for postgres, `SIGWINCH` for httpd: each picked the signal that means "graceful shutdown" for that program.)
4. Clean up: `docker rm -f shellform execform trapper inited w1 zomb zomb2 s1 s2; rm -f zombie.py noexec.sh withexec.sh`

---

## Where you'll use it

- **Zero-downtime deploys:** graceful SIGTERM handling is required for rolling updates in Docker Compose, Swarm and Kubernetes.
- **Kubernetes:** exactly the same rules. Kubernetes sends SIGTERM, waits `terminationGracePeriodSeconds` (default **30 s**), then SIGKILL. `preStop` hooks run before SIGTERM.
- **Batch/queue workers:** finish or release the current message on SIGTERM, so it isn't lost or processed twice.
- **Images that spawn processes:** CI runners, headless browsers, shell-heavy images: use tini to avoid zombie buildup.
- **Code reviews of Dockerfiles:** "is the CMD in exec form?" and "does the entrypoint script `exec "$@"`?" are standard checks.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `docker stop` always takes exactly 10 s, exit code 137 | PID 1 ignores SIGTERM: shell form, a script without `exec`, or an app with no handler | Exec form; `exec "$@"`; add a SIGTERM handler; or `--init` |
| Exit code 143 on every stop shows up as "errors" in monitoring | The app died from SIGTERM instead of exiting cleanly | Handle SIGTERM and `exit(0)`; or treat 143 as a normal stop |
| Requests fail during every deploy | App exits immediately on SIGTERM without draining, or traffic is still routed to it | Drain in-flight requests; fail readiness first; add a short delay before shutdown |
| `ps` shows many `<defunct>` processes; later `fork: Resource temporarily unavailable` | Zombies not reaped by PID 1; PID limit reached | `--init` / tini / dumb-init as PID 1 |
| Ctrl+C doesn't stop a foreground `docker run` | No `-t`, and PID 1 has no SIGINT handler | Use `-it`, `--init`, or handle SIGINT |
| Child processes keep running after the app exits | Not possible in a container: when PID 1 exits, the kernel SIGKILLs everything else | If you saw this, the "child" is outside the container (e.g. started with `--pid host`) |
| App gets SIGKILL even with a handler | Cleanup takes longer than the stop timeout | Make shutdown faster, or raise `--stop-timeout` (and in Kubernetes, `terminationGracePeriodSeconds`) |

---

## Quick recap

- `docker stop` sends the stop signal (default SIGTERM) to **PID 1 only**, waits 10 s, then SIGKILL (exit 137).
- **PID 1 ignores signals it has no handler for.** Apps not built for it, and shells, don't stop on SIGTERM.
- Use **exec form** (`["python","app.py"]`), and end entrypoint scripts with **`exec "$@"`**.
- Orphans are re-parented to PID 1. If PID 1 doesn't `wait()`, **zombies** fill the PID table.
- `--init` (tini) as PID 1 forwards signals and reaps zombies. A clean shutdown = stop taking work, finish in-flight work, close connections, exit 0.

**Interview questions**

1. **Q: Why does my container take 10 seconds to stop?**
   A: PID 1 isn't handling SIGTERM. Often it's a shell (shell-form CMD or an entrypoint script without `exec`), or an app with no handler. PID 1 ignores unhandled signals, so Docker waits for the timeout and sends SIGKILL. Fix with exec form, `exec "$@"`, a signal handler, or `--init`.

2. **Q: What is a zombie process, and why are they a container problem?**
   A: A zombie is a process that has exited but whose parent hasn't read its exit status with `wait()`. Orphans are adopted by PID 1. In a container that's usually the app, which doesn't reap them, so zombies accumulate until the PID limit stops new processes.

3. **Q: What does `docker run --init` do?**
   A: It runs tini (docker-init) as PID 1, with your command as its child. tini forwards signals to the child and reaps zombies, then exits with the child's exit code.
