# 10 · Inspecting Containers

> **Level:** ★ foundation
>
> **You will learn:** the commands that tell you what a container is doing and how it's configured: `ps`, `logs`, `inspect` (with `--format`), `exec`, `top`, `stats`, `diff`, `cp`, `port` and `events`. You'll also learn what each one does underneath, so you know what it can and can't show you.
>
> **You need:** topics 08–09.

---

## What it is

A toolbox for answering the questions every incident starts with:

| Question | Command |
|---|---|
| What's running? Since when? Is it healthy? | `docker ps` |
| What did the app print? | `docker logs` |
| How exactly is it configured? IP? Mounts? Exit code? | `docker inspect` |
| Let me look around inside | `docker exec` |
| Which processes are in it? | `docker top` |
| How much CPU/RAM/network/disk I/O is it using? | `docker stats` |
| What files did it change? | `docker diff` |
| Copy a file in or out | `docker cp` |
| Which host ports map to it? | `docker port` |
| What happened to it, and when? | `docker events` |

---

## Why it exists

Containers hide things on purpose (namespaces). That's good for isolation and bad for debugging. These commands give you a **controlled window** into the container without installing tools inside it or logging into the host as root.

---

## Analogy

A **hospital patient**:
- `ps` = the ward board: who's in which bed, how long they've been there.
- `logs` = the patient's diary.
- `inspect` = the full medical file.
- `stats` = the live heart-rate monitor.
- `top` = a list of what's going on inside right now.
- `exec` = the doctor walking in to examine them.
- `diff` = "what's changed since admission".
- `events` = the ward's activity log.

---

## How it works inside

| Command | Where the data comes from |
|---|---|
| `ps` | dockerd's in-memory list of containers (state from containerd) |
| `logs` | The **log file** written by the logging driver (default `json-file`: `/var/lib/docker/containers/<id>/<id>-json.log`). Only output written to the container's **stdout/stderr** is captured. Files the app writes inside the container are **not** logs to Docker. |
| `inspect` | dockerd's stored config + state (`config.v2.json`, `hostconfig.json`) + live data |
| `exec` | dockerd → containerd → shim → `runc exec`: a **new process** is created that **joins** the container's existing namespaces (with the `setns()` system call) and its cgroup. Its parent is the container's shim. |
| `top` | Runs `ps` **on the host** for the PIDs in the container's cgroup, so it works even if the image has no `ps` |
| `stats` | Reads the container's **cgroup files** (`memory.current`, `cpu.stat`, `io.stat`, …) and the network counters of its network namespace |
| `diff` | Compares the container's **writable layer (upperdir)** against the image (topic 07) |
| `cp` | Reads/writes through the container's root filesystem from the daemon side (works even if the container is stopped, and even without `tar` in the image) |
| `port` | The container's port mapping config (topic 28) |
| `events` | dockerd's live event stream (create, start, die, oom, kill, health status, network connect, …) |

**Important limitation of `exec`:** the new process sees the **container's** filesystem, so it can only run programs that **exist in the image**. A minimal image (distroless, scratch) has no shell, so `docker exec -it x sh` fails. Topic 51 shows tools for that case (`docker debug`, `nsenter`, debug sidecar containers).

---

## Commands / files

Start some containers for the examples:

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
docker run -d --name worker alpine sh -c 'while true; do echo "tick $(date +%T)"; sleep 2; done'
docker run --name failed alpine sh -c 'echo "about to fail" >&2; exit 3'
```

### `docker ps`

```bash
docker ps
```

```
CONTAINER ID   IMAGE               COMMAND                  CREATED          STATUS          PORTS                                     NAMES
1c2d3e4f5a6b   alpine              "sh -c 'while true; …"   30 seconds ago   Up 29 seconds                                             worker
9a8b7c6d5e4f   nginx:1.27-alpine   "/docker-entrypoint.…"   35 seconds ago   Up 34 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
```

- `COMMAND` = entrypoint + command (truncated). `--no-trunc` shows it all.
- `STATUS` = `Up 29 seconds`, or `Exited (3) ...`, `Up 2 minutes (healthy)` (health checks: topic 32), `Restarting (1)`.
- `PORTS` = `0.0.0.0:8080->80/tcp`: host port 8080 on all IPv4 addresses forwards to container port 80 (topic 28).

Useful variations:

```bash
docker ps -a                                   # include stopped
docker ps -q                                   # IDs only
docker ps -s                                   # add SIZE (writable layer size)
docker ps --filter status=exited               # filter: status, name, label, ancestor (image), health
docker ps --filter ancestor=nginx:1.27-alpine
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
```

```
NAMES     STATUS                      IMAGE
worker    Up 2 minutes                alpine
web       Up 2 minutes                nginx:1.27-alpine
```

`docker ps -s`:

```
... SIZE
... 1.09kB (virtual 47.9MB)
```

- `1.09kB` = the container's **writable layer** size. If this grows to GBs, the app is writing data into the container filesystem (a problem; topic 07, 23).
- `virtual 47.9MB` = writable layer + image.

### `docker logs`

```bash
docker logs worker
docker logs --tail 3 worker
docker logs -f --tail 0 worker        # follow live, like tail -f (Ctrl+C to stop watching)
docker logs --since 1m -t worker      # last 1 minute, with timestamps
docker logs failed
```

```
tick 10:31:02
tick 10:31:04
tick 10:31:06
2026-11-17T05:01:10.123456789Z tick 10:31:10
about to fail
```

- `-t` adds Docker's timestamp (in UTC) of when the line was received.
- `--since` accepts `10m`, `2h`, or a date like `2026-11-17T10:00:00`.
- `failed`'s output to **stderr** (`>&2`) is in the logs too. Docker captures both streams. To see only one: `docker logs failed 2>/dev/null` shows stdout only.

### `docker inspect` and `--format`

`docker inspect web` prints a large JSON document with **everything**. Learn to pull out just what you need with `--format` (or `-f`), which uses **Go templates** (Docker is written in Go; `{{ }}` marks a field to print).

```bash
docker inspect -f '{{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' failed
docker inspect -f '{{.State.Pid}}' web
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
docker inspect -f '{{.Config.Image}} {{.Image}}' web
docker inspect -f '{{json .Mounts}}' web
docker inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' web
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' web
docker inspect -f '{{.LogPath}}' web
```

```
exited exit=3 oom=false
84213
172.17.0.2
nginx:1.27-alpine sha256:65645c7bb6a0…
[]
no 0
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_VERSION=1.27.5
...
/var/lib/docker/containers/9a8b…/9a8b…-json.log
```

How to read the templates:
- `.State.Status` = walk into the JSON: the `State` object, then its `Status` field.
- `{{range .X}}...{{end}}` = loop over a list or map. Networks is a map (network name → settings), so we loop.
- `{{json .Mounts}}` = print that part as JSON.
- `{{println .}}` = print each item on its own line.
- `.HostConfig.Memory` = `0` means "no limit".
- `.Config.Image` = what you typed. `.Image` = the exact image ID it uses.

Find the field you need with `jq` (a JSON command-line tool; `sudo apt install jq`):

```bash
docker inspect web | jq '.[0] | keys'
docker inspect web | jq '.[0].NetworkSettings.Ports'
```

```
[ "AppArmorProfile", "Args", "Config", "Created", "Driver", "ExecIDs", "GraphDriver", "HostConfig", "HostnamePath", "HostsPath", "Id", "Image", "LogPath", "Mounts", "Name", "NetworkSettings", "Path", "Platform", "ProcessLabel", "ResolvConfPath", "RestartCount", "State" ]
{
  "80/tcp": [
    { "HostIp": "0.0.0.0", "HostPort": "8080" },
    { "HostIp": "::", "HostPort": "8080" }
  ]
}
```

`docker inspect` works on images, volumes and networks too (`docker image inspect`, `docker volume inspect`, `docker network inspect`).

### `docker exec`

```bash
docker exec web nginx -t
docker exec -it web sh
docker exec -u root -w /etc/nginx -e DEBUG=1 web ls
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
/ #
conf.d  fastcgi.conf  fastcgi_params  mime.types  modules  nginx.conf  scgi_params  uwsgi_params
```

- `docker exec web nginx -t` = run a command inside and print its output. Here: test the nginx config.
- `-it ... sh` = an interactive shell inside. `exit` here ends only **this** shell, not the container.
- `-u root` = run as root even if the container's main user is not root. Very useful for debugging. `-w` and `-e` work like in `run`.
- `exec` only works on a **running** container.

### `docker top`

```bash
docker top web
```

```
UID      PID     PPID    C   STIME   TTY   TIME       CMD
root     84213   84190   0   10:30   ?     00:00:00   nginx: master process nginx -g daemon off;
101      84260   84213   0   10:30   ?     00:00:00   nginx: worker process
101      84261   84213   0   10:30   ?     00:00:00   nginx: worker process
```

- **Host** PIDs (84213…), not container PIDs. `PPID 84190` = the shim.
- `UID 101` = the nginx user inside (no matching name on the host, so the number is shown).
- Works even though the image has no `ps` installed: it runs on the host.

### `docker stats`

```bash
docker stats --no-stream
```

```
CONTAINER ID   NAME     CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O         PIDS
1c2d3e4f5a6b   worker   0.05%     1.02MiB / 7.66GiB     0.01%     1.2kB / 0B        0B / 0B           2
9a8b7c6d5e4f   web      0.00%     9.88MiB / 7.66GiB     0.13%     2.1kB / 1.3kB     0B / 12.3kB       5
```

- `CPU %` = share of **one** CPU core. `250%` = 2.5 cores busy.
- `MEM USAGE / LIMIT` = from the cgroup's `memory.current` (minus some file cache) / `memory.max`. If there's no limit, the limit shown is the host's total RAM.
- `NET I/O` = bytes received / sent by the container's network interface(s).
- `BLOCK I/O` = bytes read / written to disk.
- `PIDS` = number of processes and threads (from `pids.current`).
- Without `--no-stream`, it refreshes live like `top`.

```bash
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
```

### `docker diff`

```bash
docker exec web sh -c 'echo hi > /tmp/x'
docker diff web
```

```
C /tmp
A /tmp/x
C /run
A /run/nginx.pid
C /etc
C /etc/nginx
C /etc/nginx/conf.d
C /etc/nginx/conf.d/default.conf
C /var/cache/nginx
A /var/cache/nginx/client_temp
...
```

- `A` added, `C` changed, `D` deleted (topic 07). Here we see nginx's own startup changes plus our file.

### `docker cp`

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf     # container → host
docker cp ./index.html web:/usr/share/nginx/html/    # host → container
docker cp failed:/etc/os-release - | tar -xO         # from a stopped container, to stdout
```

```
Successfully copied 2.56kB to /home/tushar/nginx.conf
Successfully copied 2.05kB to web:/usr/share/nginx/html/
NAME="Alpine Linux"
...
```

- `-` as the destination = write a tar stream to stdout.
- Works on **stopped** containers, which is great for grabbing logs or core dumps from a crashed container before you remove it.
- Changes made by `cp` go to the writable layer: lost when the container is removed. Never use `cp` to "deploy" anything to production.

### `docker port`

```bash
docker port web
```

```
80/tcp -> 0.0.0.0:8080
80/tcp -> [::]:8080
```

### `docker events`

```bash
docker events --since 10m --filter container=failed
docker events --filter event=die --filter event=oom --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}} exit={{.Actor.Attributes.exitCode}}'
```

```
2026-11-17T10:30:41.000000000+05:30 container create 5e6f… (image=alpine, name=failed)
2026-11-17T10:30:41.000000000+05:30 container attach 5e6f… (image=alpine, name=failed)
2026-11-17T10:30:41.000000000+05:30 network connect 3b2a… (container=5e6f…, name=bridge, type=bridge)
2026-11-17T10:30:41.000000000+05:30 container start 5e6f… (image=alpine, name=failed)
2026-11-17T10:30:41.000000000+05:30 container die 5e6f… (exitCode=3, image=alpine, name=failed)
```

- Docker keeps only a limited number of past events in memory (and loses them on daemon restart). For history, ship them to your monitoring system (topic 47).

---

## Hands-on lab

**The "mystery container" exercise.** (`--memory 64m` caps the container's RAM at 64 MB through its cgroup; topic 31 explains limits fully.) Run this, then answer the questions using only the commands from this topic:

```bash
docker run -d --name mystery -e SECRET_MODE=on -u 1000 -w /tmp \
  --memory 64m alpine sh -c 'echo started; dd if=/dev/zero of=/tmp/fill bs=1M count=30 2>/dev/null; echo filled; sleep 3600'
```

1. Which user and working dir does it run as? → `docker inspect -f '{{.Config.User}} {{.Config.WorkingDir}}' mystery`
2. What environment variables does it have? → `docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' mystery`
3. What did it print? → `docker logs mystery`
4. What's its memory limit in bytes? → `docker inspect -f '{{.HostConfig.Memory}}' mystery` (64 × 1024 × 1024 = 67108864)
5. What file did it create, and how big is its writable layer? → `docker diff mystery`, `docker ps -s --filter name=mystery`
6. Which host PID is it, and which processes run inside? → `docker top mystery`
7. How much memory is it using? → `docker stats --no-stream mystery`. (The 30 MB file under `/tmp` lives in the writable layer on disk, but the kernel also counts the file's cached pages against the cgroup. You'll see this "page cache counts toward memory" effect again in topic 31.)
8. Copy the file out and check its size → `docker cp mystery:/tmp/fill . && ls -lh fill`
9. Clean up → `docker rm -f mystery web worker failed; rm -f fill nginx.conf`

---

## Where you'll use it

- **Every incident:** `docker ps -a` → `docker logs --tail 100` → `docker inspect -f '{{.State}}'`. That's the first 60 seconds of almost every container investigation.
- **Config audits:** checking a running container's actual env, mounts, limits, user and restart policy against what was intended.
- **Capacity:** `docker stats` for quick resource checks. Proper monitoring: topic 47.
- **Forensics:** `docker diff` and `docker cp` from a stopped container.
- **Scripting:** `--format` and `-q` make output easy to use in shell scripts.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `docker logs` is empty, but the app "logs a lot" | The app writes to **files** inside the container, not stdout/stderr | Configure the app to log to stdout, or symlink its log file to `/dev/stdout` (the official nginx image does this) |
| `Error response from daemon: configured logging driver does not support reading` | The container uses a log driver like `syslog` or `fluentd` without the local cache | Read logs in that system, or enable dual logging (topic 33) |
| `Error response from daemon: container ... is not running` (on exec) | `exec` needs a running container | `docker logs` / `docker cp` work on stopped containers. Or `docker start` it. |
| `OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH` | No shell in the image | Use `docker debug` or a sidecar debug container (topic 51) |
| `docker stats` memory looks lower than you expect | Docker subtracts some page cache from the number | Read the raw cgroup files (topic 11, 31) |
| `template: :1:8: executing "" at <.State.Foo>: map has no entry for key "Foo"` | Wrong field name in `--format` | Check field names with `docker inspect x \| jq '.[0] \| keys'` |
| `docker logs` very slow or huge | The log file is GBs because there's no rotation | `--tail`, and set log rotation (topic 33) |

---

## Quick recap

- `ps` lists containers. `logs` shows **stdout/stderr only**. `inspect` shows full config and state; use `-f` with Go templates.
- `exec` starts a **new process** that joins the container's namespaces (setns). It needs the program to exist in the image and the container to be running.
- `top` and `stats` read from the host side (`ps` on the cgroup's PIDs; cgroup files), so they work without tools in the image.
- `diff` = the writable layer. `cp` works even on stopped containers.
- `events` streams lifecycle events (die, oom, kill, health) as they happen.

**Interview questions**

1. **Q: How do you get a shell in a running container, and what happens underneath?**
   A: `docker exec -it <name> sh`. Docker asks containerd/runc to start a new process that joins the container's existing namespaces with `setns()` and is placed in its cgroup. It's a sibling of PID 1, not a child.

2. **Q: `docker logs` shows nothing, but the app is running. Why?**
   A: Docker only captures stdout/stderr of the container's main process. The app is probably writing to log files inside the container, or using a log driver that `docker logs` can't read.

3. **Q: How do you find a container's IP address and its host PID?**
   A: `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <name>` and `docker inspect -f '{{.State.Pid}}' <name>`.
