# 11 · A Container From the Inside

> **Level:** ★★★ advanced
>
> **You will learn:** how to look at a running Docker container **from the host**, with plain Linux tools: find its host PID, compare its namespaces with the host's, enter them with `nsenter`, read its cgroup limits and usage straight from `/sys/fs/cgroup`, see its root filesystem through `/proc/PID/root`, and read its security settings from `/proc/PID/status`. This is the skill that lets you debug containers that have no tools inside.
>
> **You need:** topic 02 (namespaces, cgroups), topic 03 (the shim), topic 07 (overlay), topic 10 (`inspect`).

---

## What it is

Topic 02 said: "a container is just a Linux process". This topic **proves it on a real Docker container**, and turns that fact into a debugging method.

Everything Docker sets up is visible from the host, through standard kernel interfaces:

| What | Where on the host |
|---|---|
| The process | `ps`, `/proc/<PID>/` |
| Its namespaces | `/proc/<PID>/ns/*` |
| Its cgroup and limits | `/proc/<PID>/cgroup` → `/sys/fs/cgroup/<path>/` |
| Its root filesystem | `/proc/<PID>/root/` and its overlay mount |
| Its mounts | `/proc/<PID>/mountinfo` |
| Its user, capabilities, seccomp | `/proc/<PID>/status` |
| Its environment and command | `/proc/<PID>/environ`, `/proc/<PID>/cmdline` |
| Its network | enter the namespace with `nsenter -n` |

---

## Why it exists

In production you'll often face:
- **Minimal images** with no shell, no `ps`, no `ip`, no `curl` (distroless, scratch). `docker exec` is useless.
- **A frozen or broken Docker daemon**: `docker` commands hang, but the containers are still running (thanks to the shim).
- **Kubernetes nodes** where there is no Docker CLI at all, only containerd.
- **Questions Docker doesn't answer directly**: "Is this process throttled?", "Which capabilities does it really have?", "Which folder on disk is its `/`?"

In all these cases, the host's own tools work, because the container is just a process.

---

## Analogy

A **one-way mirror** in an interview room. People inside the room (the container) see only the room. You stand outside the mirror (the host) and see everything in the room: who's there, what they have, how much they're eating. With `nsenter`, you can also **walk into** the room carrying your own tools.

---

## How it works inside

### `nsenter`: borrowing a container's namespaces

`nsenter` (from the `util-linux` package; present on almost all Linux hosts) runs a program **from the host** inside **some or all** of another process's namespaces, using the `setns()` system call (the same call `docker exec` uses).

The key trick: **you can choose which namespaces to enter.**

- `nsenter -t <PID> -n <cmd>` → **network namespace only**. Your command sees the container's network (its `eth0`, its IP, its open ports) but runs with the **host's filesystem**, so you can use the **host's** `ip`, `ss`, `curl`, `tcpdump`, even if the image has none of them. This is the most useful debugging trick in container networking (topic 30).
- `nsenter -t <PID> -m -u -i -n -p <cmd>` → all the main namespaces: almost like `docker exec`, but the command must then exist in the container's filesystem.

| `nsenter` flag | Namespace |
|---|---|
| `-m` | mount (the container's files) |
| `-u` | UTS (hostname) |
| `-i` | IPC |
| `-n` | network |
| `-p` | PID |
| `-U` | user |
| `-C` | cgroup |
| `-a` | all of them |

### `/proc/<PID>/root`: the container's `/` from the host

The kernel exposes each process's root directory as a magic link at `/proc/<PID>/root`. Reading through it means reading the container's files **as the container sees them** (the overlay merged view), from the host. `sudo cat /proc/<PID>/root/etc/os-release` shows the container's OS file.

### cgroup files: the source of truth for resources

`docker stats` is a friendly view of these files. When you need exact numbers ("how often was this throttled?", "how close is it to the limit?", "how many OOM kills?"), read the files directly.

With the `systemd` cgroup driver (the default on modern systems), each container's cgroup is:

```
/sys/fs/cgroup/system.slice/docker-<full-container-id>.scope/
```

With the `cgroupfs` driver (older setups): `/sys/fs/cgroup/docker/<full-container-id>/`. Don't guess. Read `/proc/<PID>/cgroup` and it will tell you.

---

## Commands / files

Start a test container with some limits so there's something to see:

```bash
docker run -d --name inside --memory 128m --cpus 0.5 --pids-limit 50 nginx:1.27-alpine
```

### Step 1: find the host PID

```bash
PID=$(docker inspect -f '{{.State.Pid}}' inside)
echo $PID
ps -o pid,ppid,user,cmd -p $PID
ps -o pid,cmd -p $(ps -o ppid= -p $PID)
```

```
91234
    PID    PPID USER     CMD
  91234   91210 root     nginx: master process nginx -g daemon off;
    PID CMD
  91210 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 5b7f… -address /run/containerd/containerd.sock
```

- The container's PID 1 is host PID `91234`.
- Its parent is the **shim** (topic 03).

Without Docker (daemon hung, or a Kubernetes node), find it through the cgroup or by name:

```bash
cat /sys/fs/cgroup/system.slice/docker-5b7f*.scope/cgroup.procs
pgrep -af 'nginx: master'
```

### Step 2: compare namespaces with the host

```bash
sudo ls -l /proc/1/ns /proc/$PID/ns | grep -E 'proc|net|pid |mnt|uts|ipc|cgroup|user'
```

Shortened, side by side:

| Namespace | Host (`/proc/1/ns`) | Container (`/proc/91234/ns`) | Same? |
|---|---|---|---|
| cgroup | `cgroup:[4026531835]` | `cgroup:[4026532398]` | no → own cgroup view |
| ipc | `ipc:[4026531839]` | `ipc:[4026532396]` | no |
| mnt | `mnt:[4026531841]` | `mnt:[4026532394]` | no → own files |
| net | `net:[4026531840]` | `net:[4026532399]` | no → own network |
| pid | `pid:[4026531836]` | `pid:[4026532397]` | no → own PID numbers |
| user | `user:[4026531837]` | `user:[4026531837]` | **YES** → same users as host (no user namespace) |
| uts | `uts:[4026531838]` | `uts:[4026532395]` | no → own hostname |

- The **user** namespace is shared. That's why root in the container is host UID 0 (topic 41–42).
- A container started with `--network host` would show the **same** `net` number as the host. A container using `--pid host` the same `pid` number. This is how you check such settings from the host.

Or list them with `lsns`:

```bash
sudo lsns -p $PID
```

```
        NS TYPE   NPROCS   PID USER COMMAND
4026531837 user      298     1 root /sbin/init
4026532394 mnt         5 91234 root nginx: master process nginx -g daemon off;
4026532395 uts         5 91234 root nginx: master process nginx -g daemon off;
4026532396 ipc         5 91234 root nginx: master process nginx -g daemon off;
4026532397 pid         5 91234 root nginx: master process nginx -g daemon off;
4026532398 cgroup      5 91234 root nginx: master process nginx -g daemon off;
4026532399 net         5 91234 root nginx: master process nginx -g daemon off;
```

- `NPROCS 5` = nginx master + 4 workers share these namespaces.
- The user namespace is the host's (PID 1 = `/sbin/init`, 298 processes).

### Step 3: one process, two PIDs

```bash
grep NSpid /proc/$PID/status
docker exec inside sh -c 'echo $$; cat /proc/1/cmdline | tr "\0" " "; echo'
```

```
NSpid:  91234   1
nginx: master process nginx -g daemon off;
```

- `NSpid: 91234 1` = this process is PID **91234** in the host's PID namespace and PID **1** in the container's. The kernel keeps both numbers.

### Step 4: enter only the network namespace and use host tools

```bash
sudo nsenter -t $PID -n ip -brief addr
sudo nsenter -t $PID -n ss -lntp
sudo nsenter -t $PID -n ip route
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0@if15        UP             172.17.0.2/16
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      511          0.0.0.0:80         0.0.0.0:*     users:(("nginx",pid=91234,fd=6))
LISTEN 0      511             [::]:80            [::]:*     users:(("nginx",pid=91234,fd=7))
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```

What you learn:
- The container's `eth0` has IP `172.17.0.2`. `@if15` = its other end is interface number 15 **on the host** (a "veth pair"; topic 26).
- nginx **listens on port 80 on all addresses** inside the container. If an app listens only on `127.0.0.1` inside a container, published ports can't reach it. You'd see that here (topic 30).
- The default route goes to `172.17.0.1` = the `docker0` bridge on the host.
- **None of `ip`, `ss` came from the image.** They're the host's tools, running in the container's network namespace.

Find the host side of that cable:

```bash
ip -brief link | grep '^veth' 
```

```
veth3a9c2f1@if2    UP             5e:12:9a:aa:01:7c
```

(Match the interface index: `ip -o link | grep '^15:'`.)

### Step 5: enter everything (like exec)

```bash
sudo nsenter -t $PID -a sh
```

```
/ # hostname
5b7f3c2a1e9d
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 nginx: master process nginx -g daemon off;
   30 nginx     0:00 nginx: worker process
   ...
   40 root      0:00 sh
/ # exit
```

- Now you're inside all namespaces, including mount, so `sh` and `ps` come from the image.
- Note: `nsenter` doesn't put you into the container's **cgroup** or apply its **seccomp/capabilities**. You keep the host root's full power. Powerful for debugging, but be careful.

### Step 6: read the cgroup: limits and usage

```bash
cat /proc/$PID/cgroup
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
echo $CG
```

```
0::/system.slice/docker-5b7f3c2a1e9d….scope
/sys/fs/cgroup/system.slice/docker-5b7f3c2a1e9d….scope
```

```bash
cat $CG/memory.max $CG/memory.current $CG/memory.peak
cat $CG/cpu.max
cat $CG/pids.max $CG/pids.current
```

```
134217728
6025216
6295552
50000 100000
50
5
```

- `memory.max` = `134217728` bytes = 128 MiB (from `--memory 128m`).
- `memory.current` = ~5.7 MiB used now. `memory.peak` = the highest it has been (newer kernels).
- `cpu.max` = `50000 100000` = **may use 50 000 µs of CPU time per 100 000 µs period** = 0.5 CPU (from `--cpus 0.5`). `max 100000` would mean no limit.
- `pids.max` = 50, `pids.current` = 5 processes now.

The two files you'll check most in incidents:

```bash
cat $CG/memory.events
cat $CG/cpu.stat
```

```
low 0
high 0
max 0
oom 0
oom_kill 0
oom_group_kill 0
usage_usec 81234
user_usec 40211
system_usec 41023
nr_periods 312
nr_throttled 0
throttled_usec 0
nr_bursts 0
burst_usec 0
```

- `memory.events` → `oom_kill` = **how many times a process in this container was OOM-killed**. Even if the container didn't die (a child process was killed instead of PID 1), this counter shows it.
- `cpu.stat` → `nr_throttled` / `throttled_usec` = **how often and how long the container was paused because it hit its CPU limit**. High numbers = latency spikes, even when "average CPU" looks fine (topic 31).

Inside the container, the cgroup namespace hides the path:

```bash
docker exec inside cat /proc/self/cgroup
```

```
0::/
```

- The container thinks it is at the root of the cgroup tree. That's the cgroup namespace at work.

### Step 7: the root filesystem from the host

```bash
sudo ls /proc/$PID/root/
sudo cat /proc/$PID/root/etc/os-release | head -2
sudo grep ' / ' /proc/$PID/mountinfo | tr ',' '\n' | grep -E 'lowerdir|upperdir' | cut -c1-120
```

```
bin  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
NAME="Alpine Linux"
ID=alpine
lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/58/fs:/var/lib/containerd/io.con...
upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/66/fs
```

- `/proc/$PID/root/` is a live view of the container's `/`. You can copy files out of a no-shell container this way: `sudo cp /proc/$PID/root/app/config.yaml /tmp/`.
- The mount line shows the overlay layers (topic 07). Your paths say `overlay2` if you use the older storage driver.

### Step 8: user, capabilities and seccomp

```bash
grep -E '^(Name|Uid|Gid|CapEff|CapBnd|NoNewPrivs|Seccomp)' /proc/$PID/status
```

```
Name:   nginx
Uid:    0       0       0       0
Gid:    0       0       0       0
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
NoNewPrivs:     0
Seccomp:        2
```

- `Uid: 0 0 0 0` = real, effective, saved and filesystem UID are all 0 (root).
- `CapEff` = the **effective capabilities**, as a bit mask. Decode it with `capsh` (package `libcap2-bin` on Ubuntu):

```bash
capsh --decode=00000000a80425fb
```

```
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

- This is Docker's **default set of 14 capabilities**. Host root has about 41. Missing ones include `cap_sys_admin`, `cap_net_admin`, `cap_sys_module`. This is part of why root in a container is weaker than root on the host (topic 41).
- `Seccomp: 2` = a seccomp **filter** is active (Docker's default profile). `0` would mean none (e.g. `--privileged` or `seccomp=unconfined`).
- `NoNewPrivs: 0` = `no-new-privileges` isn't set (topic 41 shows how to turn it on).

Compare with a worker process:

```bash
WPID=$(pgrep -P $PID | head -1); grep -E '^(Uid|CapEff)' /proc/$WPID/status
```

```
Uid:    101     101     101     101
CapEff: 0000000000000000
```

- Workers run as UID 101 with **no** effective capabilities. That's nginx dropping privileges itself.

### Step 9: environment and command line

```bash
sudo tr '\0' ' ' < /proc/$PID/cmdline; echo
sudo tr '\0' '\n' < /proc/$PID/environ
```

```
nginx: master process nginx -g daemon off;
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=5b7f3c2a1e9d
NGINX_VERSION=1.27.5
...
```

(Note: nginx rewrites its own command line to `nginx: master process ...`. Most programs don't.)

---

## Hands-on lab: debug a "no tools" container

1. Run a minimal image with **no shell**:
   ```bash
   docker run -d --name notools --memory 64m registry.k8s.io/pause:3.10
   docker exec -it notools sh
   ```
   ```
   OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH
   ```
   (The `pause` image is what Kubernetes uses to hold a Pod's namespaces. It contains one tiny static program that just sleeps forever, and nothing else: no shell, no tools.)

2. Investigate entirely from the host:
   ```bash
   PID=$(docker inspect -f '{{.State.Pid}}' notools)
   sudo ls -la /proc/$PID/root/                      # what files exist?
   sudo nsenter -t $PID -n ip -brief addr            # its IP, with host tools
   sudo nsenter -t $PID -n ss -lntp                  # is anything listening?
   CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
   cat $CG/memory.max $CG/memory.current             # limit and usage
   grep -E '^(Uid|CapEff|Seccomp)' /proc/$PID/status # user and security
   ```
3. Write down: its IP, its memory limit, its UID, and whether it has any capabilities.
4. **Bonus:** stop dockerd (`sudo systemctl stop docker docker.socket`). If `live-restore` is off, the containers stop too. Turn it on in topic 34 and repeat: the process, namespaces and cgroup stay readable with `docker` down. Start Docker again afterwards.
5. Clean up: `docker rm -f inside notools`

---

## Where you'll use it

- **Distroless/scratch images** in production: debug without changing the image.
- **Network problems:** `nsenter -n` + host `tcpdump`, `ss`, `curl`, `dig` (topic 30).
- **Performance:** `cpu.stat` throttling and `memory.events` OOM counts explain latency and silent restarts.
- **Kubernetes nodes:** the exact same method works for any Pod's containers (find the PID with `crictl inspect`; topic 53).
- **Security audits:** checking real capabilities, seccomp status and UIDs of running workloads.
- **When the daemon is broken:** containers keep running under their shims, and you can still inspect them.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `nsenter: cannot open /proc/91234/ns/net: Permission denied` | You need root to enter another process's namespaces | Use `sudo` |
| `nsenter: failed to execute ip: No such file or directory` | You entered the **mount** namespace too (`-m` or `-a`), so `ip` was looked up in the image | Enter only `-n` to use host tools |
| `docker inspect -f '{{.State.Pid}}'` prints `0` | The container isn't running | Start it; for a dead container use `docker logs`/`docker cp` |
| No `/sys/fs/cgroup/system.slice/docker-<id>.scope` | cgroupfs driver, rootless Docker, or cgroups v1 | Read `/proc/<PID>/cgroup` for the real path |
| `cat: memory.peak: No such file or directory` | Older kernel (memory.peak needs Linux 5.19+) | Use `memory.current`, `memory.stat` |
| `capsh: command not found` | Not installed | `sudo apt install libcap2-bin` |
| Files under `/proc/$PID/root` look different from the image | That's the merged view, including runtime changes and mounts (volumes) | Use `docker diff` and `/proc/$PID/mountinfo` to tell them apart |

---

## Quick recap

- A container's main process has a normal host PID: `docker inspect -f '{{.State.Pid}}'`. Its parent is the shim.
- `/proc/PID/ns/*` numbers show which namespaces it shares with the host. By default everything except the **user** namespace is separate.
- `nsenter -t PID -n <host tool>` runs host tools in the container's network: the best trick for no-tool images.
- `/proc/PID/cgroup` gives the cgroup path. `memory.max/current/events` and `cpu.max/cpu.stat` give exact limits, usage, OOM kills and throttling.
- `/proc/PID/root` is the container's `/`. `/proc/PID/status` shows UID, capabilities and seccomp.

**Interview questions**

1. **Q: How do you debug networking in a container that has no shell or network tools?**
   A: Get its host PID with `docker inspect`, then run host tools in only its network namespace: `nsenter -t <PID> -n ip addr` / `ss -lntp` / `tcpdump -i eth0`.

2. **Q: How can you tell if a container is being CPU-throttled?**
   A: Read `cpu.stat` in its cgroup folder. Rising `nr_throttled` and `throttled_usec` mean it keeps hitting its `cpu.max` quota.

3. **Q: Is root inside a container the same as root on the host?**
   A: By default it's the same UID 0, because Docker doesn't use a user namespace. But it has only 14 capabilities (instead of all of them), a seccomp filter, and limited namespaces. User namespaces or rootless mode can map it to an unprivileged host UID.
