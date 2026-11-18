# 51 · Debugging Tools

> **Level:** ★★★ advanced
>
> **You will learn:** how to debug containers that have **no tools inside**: debug **sidecar** containers that join the target's PID and network namespaces (`--pid container:`, `--network container:`), `docker debug`, host-side `nsenter` with `strace`, `tcpdump`, `perf` and `gdb`, reaching the container's files via `/proc/PID/root`, language-specific profilers (py-spy, jcmd, Node inspector), core dumps, and containerd's own CLIs (`ctr`, `nerdctl`) for when dockerd is broken or absent.
>
> **You need:** topic 11 (namespaces from the host, `nsenter`), topic 27 (`container:` network mode), topic 41 (capabilities: `SYS_PTRACE`), topic 50 (the method).

---

## What it is

Production images are (and should be) minimal: no shell, no `ps`, no `curl`, no `strace` (topic 19). So debugging means **bringing tools to the container** instead of putting them in it. Three ways:

| Approach | Your tools run… | Sees the target's… | Works when |
|---|---|---|---|
| **Debug sidecar container** (`--pid container:X --network container:X`) | In a separate toolbox container (netshoot, busybox, ubuntu) | Processes, network, and files (via `/proc/<pid>/root`) | Docker access, no host shell needed |
| **`docker debug`** | In a toolbox attached by Docker Desktop | Filesystem, processes, network | Docker Desktop (developer machines) |
| **Host tools + `nsenter`** | On the host, entering only the namespaces you choose | Whatever namespaces you enter | You have root on the host. Works even if dockerd is down |

---

## Why it exists

Adding debug tools to production images increases size and attack surface, and you can't rebuild and redeploy in the middle of an incident anyway. Linux namespaces make it possible to **share** a container's view temporarily, with a different filesystem full of tools.

---

## Analogy

A **mechanic and a sealed car**. You don't weld a toolbox into every car (debug tools in images). The mechanic brings their own toolbox and plugs into the car's **diagnostic port** (namespaces), seeing the engine's live data without opening it up.

---

## How it works inside

### Sidecar: joining namespaces

```bash
docker run --rm -it \
  --pid container:app \            # share app's PID namespace: see and trace its processes
  --network container:app \        # share app's network namespace: same IP, ports, localhost
  --cap-add SYS_PTRACE \           # allow strace/gdb/py-spy to attach
  nicolaka/netshoot
```

Inside the sidecar:
- `ps aux` shows **the app's processes** (PID 1 = the app).
- `ss -lntp`, `curl localhost:8080`, `tcpdump -i eth0` work on **the app's** network.
- `/proc/1/root/` = **the app's filesystem**, so you can read its config files and logs.
- Your own `/` is netshoot's filesystem with all its tools.

Limits:
- Mount namespace is **not** shared (Docker can't share it), so use `/proc/<pid>/root`.
- Attaching with `strace`/`gdb` needs `SYS_PTRACE`, and the default seccomp/AppArmor may block `ptrace` on some older kernels/profiles (add `--security-opt seccomp=unconfined` only for the debug sidecar if needed).
- To trace a process running as UID 10001, run the sidecar as root (default) with `SYS_PTRACE`.

Kubernetes has the same thing built in: `kubectl debug -it pod/app --image=nicolaka/netshoot --target=app` (an **ephemeral container** sharing the target container's process namespace).

### `docker debug`

A Docker Desktop CLI feature that attaches a toolbox shell to **any** container, even with no shell or a stopped container, with the container's filesystem mounted, plus a package manager for extra tools (`install tcpdump`). It's convenient on developer machines. On servers with plain Docker Engine, use sidecars and `nsenter`.

### Host tools via `nsenter`

From topic 11: choose exactly which namespaces to enter, and keep the **host's filesystem** (and so its tools) by **not** entering the mount namespace:

| Need | Command |
|---|---|
| Network view with host tools | `nsenter -t PID -n ss -tanp` / `tcpdump -i eth0` / `curl` / `dig` |
| Trace syscalls (no namespace needed: ptrace works by host PID) | `strace -f -p PID` |
| CPU profile | `perf top -p PID`, `perf record -g -p PID` |
| Debugger | `gdb -p PID` (symbols may need the container's binary: `/proc/PID/root/usr/bin/app`) |
| Open files/sockets | `ls -l /proc/PID/fd`, `lsof -p PID` |
| Container files | `ls /proc/PID/root/etc/app/` |
| Run a command "like exec" (all namespaces) | `nsenter -t PID -a <cmd from the image>` |

### What strace tells you (very useful in production)

`strace` shows every system call a process makes: files opened, network connects, reads/writes, sleeps, errors.
- `strace -f -p PID -e trace=network` = only network-related calls (connect, sendto, recvfrom…).
- `strace -c -f -p PID` (Ctrl+C after a while) = a **summary table** of which syscalls take time. Great for "it's slow, but where?"
- Seeing `connect(… 10.0.0.5:5432 …) = -1 ETIMEDOUT` or `openat("/etc/app/config.yaml") = -1 ENOENT` often ends an investigation in seconds.
- Cost: it slows the traced process noticeably. Use it briefly in production.

### Language-specific tools

| Runtime | Tool | Notes |
|---|---|---|
| Python | **py-spy** (`py-spy dump --pid PID`, `py-spy top --pid PID`) | Sampling profiler; no restart; needs `SYS_PTRACE`. Run from the host or a sidecar |
| Java | `jcmd`, `jstack`, `jmap` (JDK tools) | The JVM attach mechanism uses files in `/tmp` and expects the same UID: easiest via `docker exec` if the image has a JDK, or a sidecar with `--pid container:` running as the same UID and sharing `/tmp`. Or `kill -3 PID` (SIGQUIT) prints a thread dump to the app's stdout = `docker logs` |
| Go | `pprof` endpoint (`net/http/pprof`), or `kill -QUIT PID` → goroutine dump to stderr | Plan it in the app |
| Node.js | `--inspect` (debug port), `kill -USR1 PID` to enable the inspector at runtime | Publish/forward port 9229 carefully (`127.0.0.1` only) |
| Any native | `perf`, `gdb`, `eu-stack` | Need symbols |

### Core dumps

When a process crashes with SIGSEGV (exit 139), the kernel can write a **core dump** (a memory snapshot) for analysis with `gdb`. Where it goes is set by the **host** setting `/proc/sys/kernel/core_pattern` (often piped to `systemd-coredump` on the host: `coredumpctl list`). Inside containers, also allow it: `--ulimit core=-1`. Core dumps contain memory, so they can contain **secrets**: handle them carefully.

### When dockerd is broken or absent: containerd tools

- **`ctr`** (ships with containerd): low-level, for inspection. `ctr -n moby containers ls`, `ctr -n moby tasks ls`, `ctr -n moby tasks exec`. **Don't modify Docker's containers with it** (topic 03).
- **`nerdctl`**: a Docker-compatible CLI for containerd (`nerdctl ps`, `nerdctl exec`, `nerdctl logs`, even `nerdctl compose`). Use `--namespace moby` to look at Docker's containers, or `k8s.io` for Kubernetes'.
- **`crictl`**: the Kubernetes CRI CLI on Kubernetes nodes (topic 53).

### Freezing evidence

- `docker commit <c> forensics:<date>` = save the writable layer as an image (topic 14).
- `docker export <c> > fs.tar` = the whole merged filesystem.
- `docker pause <c>` = freeze processes (cgroup freezer) while you collect memory/state, e.g. with `gcore PID` from the host for a memory dump.

---

## Commands / files

### Sidecar on a no-shell app

```bash
docker run -d --name app -p 8090:8080 gcr.io/distroless/python3-debian12 \
  -c "import http.server,socketserver; socketserver.TCPServer(('0.0.0.0',8080), http.server.SimpleHTTPRequestHandler).serve_forever()"
docker exec -it app sh
```

```
OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH
```

```bash
docker run --rm -it --pid container:app --network container:app --cap-add SYS_PTRACE nicolaka/netshoot
```

Inside the sidecar:

```bash
ps aux
ss -lntp
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080
ls /proc/1/root/
cat /proc/1/root/etc/os-release | head -2
strace -f -p 1 -e trace=network -o /tmp/s.txt &  sleep 3; curl -s localhost:8080 >/dev/null; sleep 1; kill %1; head -5 /tmp/s.txt
```

```
PID   USER     TIME  COMMAND
    1 root      0:00 /usr/bin/python3.11 -c import http.server,socketserver; ...
   13 root      0:00 zsh
   ...
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*     users:(("python3.11",pid=1,fd=3))
200
bin  boot  dev  etc  home  lib  lib64  proc  root  run  sbin  sys  tmp  usr  var
PRETTY_NAME="Distroless"
NAME="Debian GNU/Linux"
1     accept4(3, {sa_family=AF_INET, sin_port=htons(51234), sin_addr=inet_addr("127.0.0.1")}, [16], SOCK_CLOEXEC) = 4
1     getsockname(4, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
1     recvfrom(4, "GET / HTTP/1.1\r\nHost: localhost:"..., 8192, 0, NULL, NULL) = 78
1     sendto(4, "HTTP/1.0 200 OK\r\nServer: SimpleH"..., 143, 0, NULL, 0) = 143
```

- You saw the app's processes, sockets and files, and traced a request live, all without changing the app's image.

### Host-side equivalents

```bash
PID=$(docker inspect -f '{{.State.Pid}}' app)
sudo strace -c -f -p $PID & SP=$!; for i in $(seq 1 50); do curl -s localhost:8090 >/dev/null; done; sudo kill -INT $SP; wait $SP 2>/dev/null
```

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 38.12    0.004210          21       200           sendto
 22.40    0.002474          12       200           recvfrom
 14.03    0.001549           7       200           newfstatat
 ...
100.00    0.011044                  1450        13 total
```

```bash
sudo perf top -p $PID              # live CPU hotspots (Ctrl+C to quit)
sudo ls -l /proc/$PID/fd | head
```

### Python profiling without restarting (py-spy)

```bash
docker run -d --name busy python:3.13-slim python -c "
def work():
    return sum(i*i for i in range(10**6))
while True: work()"
docker run --rm --pid container:busy --cap-add SYS_PTRACE python:3.13-slim \
  sh -c 'pip install -q py-spy && py-spy dump --pid 1'
```

```
Process 1: python -c ...
Python v3.13.9 (/usr/local/bin/python3.13)

Thread 0x7F2C1A3B6740 (active): "MainThread"
    <genexpr> (<string>:3)
    work (<string>:3)
    <module> (<string>:4)
```

- The exact line where the program is spending its time.

### Java thread dump through `docker logs`

```bash
docker kill -s QUIT myjavaapp
docker logs --tail 50 myjavaapp
```

```
Full thread dump OpenJDK 64-Bit Server VM (21.0.5+11-LTS mixed mode, sharing):
"main" #1 [1] prio=5 os_prio=0 cpu=812.55ms elapsed=3600.12s tid=0x00007f... nid=1 waiting on condition
   java.lang.Thread.State: TIMED_WAITING (sleeping)
...
```

- SIGQUIT makes the JVM print all thread stacks to stdout and **keep running**.

### containerd's view when dockerd is down

```bash
sudo systemctl stop docker docker.socket          # with live-restore, containers keep running
sudo ctr -n moby tasks ls
sudo nerdctl --namespace moby ps                  # if nerdctl is installed
sudo systemctl start docker
```

```
TASK                                                                PID      STATUS
5b7f3c2a1e9d…                                                       91234    RUNNING
```

### Forensic snapshot

```bash
docker pause app
docker commit -m "incident 2026-11-17" app forensics/app:2026-11-17
docker export app | gzip > app-fs-2026-11-17.tar.gz
docker unpause app
```

---

## Hands-on lab

1. Run the distroless `app` and debug it with a sidecar: processes, sockets, files, and a live `strace` of one request.
2. From the **host**, do the same with `nsenter -n` (network) and `strace -p` (syscalls), without any sidecar.
3. Profile the `busy` Python container with py-spy (`dump` and `top`).
4. Simulate "slow because of DNS": run a container with `--dns 10.255.255.1` that calls `curl https://example.com` in a loop, and use `strace -f -e trace=network -p PID` to watch it waiting on UDP port 53.
5. Stop dockerd (with live-restore on) and inspect containers with `ctr` and `/proc`. Start dockerd again.
6. Clean up: `docker rm -f app busy`

---

## Where you'll use it

- **Production incidents** on minimal images, which are standard now.
- **Performance work:** `perf`, `strace -c`, py-spy/async-profiler without redeploying.
- **Network issues:** netshoot sidecars, `nsenter -n tcpdump`.
- **Kubernetes:** `kubectl debug` (ephemeral containers, the same idea as the sidecar), `crictl` + `nsenter` on nodes.
- **Security forensics:** commit/export/pause to preserve evidence.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `strace: attach: ptrace(PTRACE_SEIZE, 1): Operation not permitted` | Missing `SYS_PTRACE`, or seccomp/AppArmor/Yama blocking ptrace | `--cap-add SYS_PTRACE`; for a debug sidecar only, `--security-opt seccomp=unconfined`; host: `kernel.yama.ptrace_scope` |
| `docker: Error response from daemon: cannot join PID namespace of a non running container` | Target stopped | Use `docker cp`/`docker export`/`docker logs` on stopped containers |
| `gdb`/`perf` show no function names | Stripped binaries/no symbols, or symbols live inside the container | Point tools at `/proc/PID/root/...` binaries; build debug-symbol packages |
| JVM tools: `Unable to open socket file` | JVM attach needs the same UID and a shared `/tmp` | Use `docker exec` as the app user, SIGQUIT thread dumps, or JFR configured in advance |
| `docker debug`: unknown command | Not available with plain Docker Engine | Use sidecars or nsenter |
| Traced app becomes very slow | strace overhead | Trace briefly; use `-c` summaries; prefer perf/eBPF tools for long runs |
| `ctr` shows the containers but `docker` doesn't | dockerd down or out of sync | Restart dockerd; never change Docker's containers with `ctr` |

---

## Quick recap

- Bring tools **to** the container: a **sidecar** with `--pid container:X --network container:X --cap-add SYS_PTRACE` sees its processes, network and files (`/proc/1/root`).
- `docker debug` is a convenience on Docker Desktop. On servers, use **`nsenter`** with host tools (enter only `-n` to keep host binaries), `strace -f -p`, `perf`, `gdb`, `/proc/PID/fd` and `/proc/PID/root`.
- Runtime tools: py-spy (Python), SIGQUIT thread dumps / jcmd (Java), pprof or SIGQUIT (Go), the Node inspector.
- When dockerd is down: `ctr`/`nerdctl --namespace moby` (look only) and `/proc`. In Kubernetes, `crictl`.
- Preserve evidence with `pause`, `commit`, `export` and core dumps (handle them as sensitive).

**Interview questions**

1. **Q: How do you debug a running container whose image has no shell or tools?**
   A: Start a debug container that shares its PID and network namespaces (`--pid container:X --network container:X`, plus `SYS_PTRACE` if tracing) with a toolbox image like netshoot, and read its files via `/proc/1/root`. Or use `nsenter` from the host with host tools. In Kubernetes, use `kubectl debug` with an ephemeral container.

2. **Q: How would you find out what a hung process in a container is doing?**
   A: Get its host PID, then use `strace -f -p PID` to see which syscall it's blocked on (e.g. a connect or read to a specific address), a language-specific dump (py-spy, SIGQUIT thread dump), and `/proc/PID/fd` plus `nsenter -n ss -tanp` to see its open connections.

3. **Q: What can you do if the Docker daemon is unresponsive but containers are still running?**
   A: Inspect containers from the host: `ps` for the shims and processes, `/proc/PID/*`, cgroup files, and `nsenter`. Use `ctr`/`nerdctl` in the `moby` namespace to list tasks, read-only. Collect dockerd logs and a SIGUSR1 stack dump before restarting it, ideally with live-restore enabled.
