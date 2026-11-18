# 02 · What a Container Really Is

> **Level:** ★★ intermediate
>
> **You will learn:** that a container is an ordinary Linux process with three things added: **namespaces** (what it can see), **cgroups** (what it can use), and its **own root filesystem** (which files it gets). You'll build a mini container by hand with plain Linux commands, before you even install Docker.
>
> **You need:** topic 01. Your Linux notes on processes, `/proc` and the filesystem.

---

## What it is

A container is **a normal Linux process** (or a group of processes) that the kernel treats specially:

1. **Namespaces** change **what the process can see**. It sees its own process list, its own network, its own hostname, its own mounts.
2. **cgroups** (control groups) limit **what the process can use**: how much memory, how much CPU, how many processes.
3. **A separate root filesystem** gives it **its own files**: its own `/bin`, `/etc`, `/usr/lib`, taken from the image.

That's it. There is no "container object" inside the Linux kernel. The kernel has no idea what a "container" is. It only knows processes, namespaces and cgroups. "Container" is a name we give to this combination.

> Two more kernel features add security on top: **capabilities** (splitting root's power into small pieces) and **seccomp** (blocking dangerous system calls). We'll mention them here and cover them properly in topics 41 and 41a.

---

## Why it exists

Remember the goals from topic 01: isolate apps from each other on one machine, without the cost of a full VM.

The kernel already runs every process. So the cheapest possible isolation is: **ask the kernel to lie to the process** about what exists (namespaces), **ask the kernel to ration resources** (cgroups), and **give it a different `/`** (root filesystem). No hardware emulation, no second kernel.

---

## Analogy

Think of an **open-plan office building** (the host machine, with one building manager: the kernel).

- **Namespaces = the walls and one-way glass.** Each team gets a room. From inside, they can only see their own people (PID namespace), their own phone line (network namespace), their own name on the door (UTS namespace). The rooms are in the same building, and the manager sees everyone.
- **cgroups = the building's meters and fuses.** Each room gets a power limit and a water limit. If one room tries to use too much, the fuse trips (the OOM killer) for that room only. The building stays up.
- **Root filesystem = the furniture in the room.** One team has Ubuntu furniture, another has Alpine furniture. It's still the same building structure (the kernel).

---

## How it works inside

### Part A: Namespaces (what the process sees)

**Recap from Linux:** every process on Linux has a PID (process ID), sees a hostname, sees a set of network interfaces, and sees a tree of mounted filesystems. Normally all processes share the same view.

A **namespace** wraps one of these global resources so that processes inside the namespace get **their own private copy**. Processes in different namespaces can't see each other's copy.

Linux has 8 namespace types:

| Namespace | Short name | What gets its own copy | What the container sees |
|---|---|---|---|
| Mount | `mnt` | The list of mounted filesystems | Its own `/`, its own mounts |
| Process ID | `pid` | The process number list | Its main process is PID 1; it can't see host processes |
| Network | `net` | Network interfaces, IPs, routes, firewall rules, ports | Its own `eth0`, own IP, own `localhost` |
| UTS | `uts` | Hostname and domain name (UTS = UNIX Time-Sharing, a historical name) | Its own hostname (the container ID by default) |
| IPC | `ipc` | Shared memory and message queues (IPC = Inter-Process Communication) | Can't touch other containers' shared memory |
| User | `user` | User and group ID numbers | Root inside can map to a normal user outside (off by default in Docker, topic 42) |
| Cgroup | `cgroup` | Its view of the cgroup tree | Sees its own cgroup as the root |
| Time | `time` | Some system clocks | Not used by Docker |

**How the kernel does it:** each process has a pointer to one namespace of each type. You can see these pointers as files under `/proc/<PID>/ns/`:

```bash
ls -l /proc/self/ns
```

```
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 net -> 'net:[4026531840]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 time -> 'time:[4026531834]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 tushar tushar 0 Nov 17 10:00 uts -> 'uts:[4026531838]'
```

- `/proc/self` = the process running this command (`ls` itself).
- Each line is one namespace this process belongs to.
- The number in brackets (e.g. `4026531840`) is the **namespace ID** (technically an inode number). **Two processes with the same number share that namespace.** Different number = different namespace = isolated.
- `pid_for_children` = which PID namespace this process's children will be born into.

This is the single most useful trick for "seeing a container from the inside": compare these numbers between a host process and a container process (topic 11).

**The system calls behind it** (a system call = a request a program makes to the kernel):
- `clone()` = create a new process, optionally in new namespaces.
- `unshare()` = move the current process into new namespaces.
- `setns()` = join an existing namespace. This is how `docker exec` and `nsenter` get "inside" a running container.

**PID namespace detail:** the first process in a new PID namespace becomes **PID 1** inside it. PID 1 is special in Linux (it's normally `init`/`systemd`). It must reap zombie processes, and signals are treated differently for it. This causes real production bugs, covered in topic 12. From the host, the same process has a normal, large PID like `48213`. **One process, two PIDs**: one inside the namespace, one on the host.

### Part B: cgroups (what the process can use)

**Recap from Linux:** without limits, any process can use all the RAM and all the CPU. One runaway app can starve everything else, including SSH, so you can't even log in to fix it.

A **cgroup** (control group) is a group of processes that the kernel **measures and limits together**. cgroups are controlled through a special filesystem, mounted at `/sys/fs/cgroup`. You manage them by creating folders and writing numbers into files. No special API is needed.

Today's Linux uses **cgroups v2** (one unified tree). Older systems used v1 (a separate tree per resource). Docker supports both, but every modern distribution (Ubuntu 22.04+, RHEL 9+, Debian 11+) uses v2. These notes use v2.

Main **controllers** (a controller = the part of cgroups that manages one resource):

| Controller | Limits | Example file |
|---|---|---|
| `memory` | RAM use | `memory.max` |
| `cpu` | CPU time | `cpu.max` |
| `cpuset` | Which CPU cores and memory nodes | `cpuset.cpus` |
| `io` | Disk read/write speed | `io.max` |
| `pids` | Number of processes | `pids.max` |

Check your cgroup version:

```bash
stat -fc %T /sys/fs/cgroup
```

```
cgroup2fs
```

- `cgroup2fs` = cgroups v2. If you see `tmpfs`, you're on v1.

See which cgroup your shell is in:

```bash
cat /proc/self/cgroup
```

```
0::/user.slice/user-1000.slice/session-3.scope
```

- `0::` = the v2 unified tree (always `0` and an empty controller list in v2).
- `/user.slice/...` = the path under `/sys/fs/cgroup`. systemd put your login session here.

When Docker starts a container, it creates a cgroup for it, typically `/sys/fs/cgroup/system.slice/docker-<container-id>.scope`, and writes your limits into files there. If the processes in that cgroup use more memory than `memory.max`, the kernel's **OOM killer** (OOM = Out Of Memory) kills one of them. That's the famous **exit code 137** (topic 08 and topic 31).

### Part C: The root filesystem (which files it gets)

**Recap from Linux:** every process has a **root directory**, the folder it sees as `/`. Normally that's the real `/` of the machine.

Two ways to change it:
- `chroot` (change root): an old, simple call. It changes where `/` points for a process. Easy to escape if you're root, so it's not enough for containers.
- `pivot_root`: swaps the whole root mount inside a **mount namespace**, then the old root is unmounted. The old host filesystem is no longer reachable at all. **This is what container runtimes (runc) use.**

Where does the new root come from? From the **image**: a set of folders with a full Linux user space (`/bin/sh`, `/etc/os-release`, `/usr/lib/...`). Docker stacks the image's layers with the **overlay filesystem** to make one folder, and that folder becomes the container's `/`. Topic 07 explains overlay in detail. For now: **the container's `/` is just a folder on the host**.

This is why an "Ubuntu container" on a RHEL host works: "Ubuntu" here is just Ubuntu's *files*. The kernel underneath is still RHEL's.

### Putting it together: what a runtime does to start a container

When Docker (through its low-level runtime called **runc**, explained in topic 03) starts a container, it roughly does this:

1. Create a cgroup and write the limits (`memory.max`, `cpu.max`, `pids.max`).
2. Call `clone()` with flags for new mnt, pid, net, uts, ipc and cgroup namespaces.
3. Inside the new process: set the hostname (UTS), set up mounts (`/proc`, `/dev`, `/sys`), `pivot_root` into the image's folder.
4. Drop dangerous **capabilities** and apply a **seccomp** filter (topics 41, 41a).
5. Put the process into the cgroup.
6. `execve()` your program (for example `nginx`). Now your program runs as PID 1 in its own little world.

> `execve()` = the system call that replaces the current program with a new one. Your app "becomes" the container's process.

### Container vs VM: one more time, technically

- A VM's process runs on a **virtual CPU** under a **guest kernel**.
- A container's process runs on the **real CPU** under the **host kernel**, like any other process. `top` on the host shows it. `kill` on the host can kill it.

---

## Commands / files

Docker isn't installed yet. These are the Linux tools that do the same things Docker does. All need a Linux machine (a VM, a cloud server, or WSL2 from topic 04). If you only have Windows right now, read along and come back to the lab after topic 04.

### `unshare`: run a program in new namespaces

```bash
sudo unshare --uts --pid --fork --mount-proc bash
```

- `unshare` = run a program in new namespaces.
- `--uts` = new hostname namespace.
- `--pid` = new PID namespace.
- `--fork` = fork first, so the child (not `unshare` itself) becomes PID 1 in the new namespace. (Without it, you get errors like `fork: Cannot allocate memory`, because the namespace's PID 1 would be gone.)
- `--mount-proc` = mount a fresh `/proc` for this PID namespace, so `ps` shows only processes inside. (It also creates a new mount namespace, so this mount doesn't affect the host.)
- `bash` = the program to run.

Inside it:

```bash
ps aux
```

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   8932  5376 pts/1    S    10:05   0:00 bash
root           8  0.0  0.0  10856  4352 pts/1    R+   10:05   0:00 ps aux
```

- Only two processes. `bash` is **PID 1**. The hundreds of host processes are invisible.

```bash
hostname container-demo
hostname
```

```
container-demo
```

Open another terminal on the host and run `hostname`: it is unchanged. The UTS namespace isolated it.

### `lsns`: list namespaces

From the host (second terminal):

```bash
sudo lsns -t pid
```

```
        NS TYPE NPROCS   PID USER COMMAND
4026531836 pid     212     1 root /sbin/init
4026532301 pid       1 51234 root bash
```

- `NS` = namespace ID (the same number as in `/proc/PID/ns/pid`).
- `NPROCS` = how many processes are in it.
- `PID` = the lowest **host** PID in that namespace. Our `bash` is PID 1 inside but **51234** on the host.
- The first line is the host's normal PID namespace with 212 processes.

### Making a cgroup by hand

```bash
sudo mkdir /sys/fs/cgroup/demo
ls /sys/fs/cgroup/demo | head
```

```
cgroup.controllers
cgroup.events
cgroup.freeze
cgroup.kill
cgroup.max.depth
cgroup.max.descendants
cgroup.procs
cgroup.stat
cgroup.subtree_control
cgroup.threads
```

- Making a folder **is** creating a cgroup. The kernel fills it with control files instantly.
- `cgroup.procs` = list of PIDs in this group. Write a PID here to move a process in.
- If you don't see `memory.max`, the memory controller isn't enabled for children yet. Check `cat /sys/fs/cgroup/cgroup.subtree_control`. It usually already contains `memory pids cpu` on systemd machines.

---

## Hands-on lab: build a tiny container with no Docker

Goal: combine all three pieces (namespaces, cgroup, own root filesystem) with plain Linux tools.

**Step 1: get a root filesystem.** We'll use Alpine's "minirootfs", a tiny Linux user space as a tar file.

```bash
mkdir -p ~/minicontainer/rootfs
cd ~/minicontainer
curl -fsSLo alpine.tar.gz https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.2-x86_64.tar.gz
sudo tar -xzf alpine.tar.gz -C rootfs
ls rootfs
```

```
bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

This folder is a complete Alpine Linux user space. No kernel inside it. (If the URL changes, browse `https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/` for the current `alpine-minirootfs-*.tar.gz`. If your CPU is ARM, use the `aarch64` folder instead.)

**Step 2: create a cgroup with a memory limit.**

```bash
sudo mkdir -p /sys/fs/cgroup/minic
echo 50M | sudo tee /sys/fs/cgroup/minic/memory.max
echo 20  | sudo tee /sys/fs/cgroup/minic/pids.max
```

- Processes in this group may use 50 MB of RAM and at most 20 processes.

**Step 3: start a shell in new namespaces, put it in the cgroup, and change root.**

```bash
sudo unshare --uts --pid --mount --net --ipc --fork \
  bash -c 'echo $$ > /sys/fs/cgroup/minic/cgroup.procs;
           hostname mini;
           mount -t proc proc rootfs/proc;
           exec chroot rootfs /bin/sh'
```

- `--net` gives it an empty network namespace (only a `lo` loopback that is down).
- `echo $$ > .../cgroup.procs` moves this shell into our cgroup **before** it turns into the container shell. (`$$` = this shell's own PID.)
- `mount -t proc` gives it a `/proc` that matches its PID namespace.
- `exec chroot rootfs /bin/sh` replaces bash with Alpine's `sh`, with `rootfs` as `/`. (Real runtimes use `pivot_root`, but `chroot` is fine for a demo.)

**Step 4: look around inside.**

```sh
cat /etc/os-release | head -2
ps
hostname
ip addr
uname -r
```

```
NAME="Alpine Linux"
ID=alpine
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    5 root      0:00 ps
mini
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6.8.0-45-generic
```

What each line proves:
- `Alpine Linux`: own root filesystem, even if your host is Ubuntu.
- `PID 1 /bin/sh`: own PID namespace.
- `mini`: own hostname (UTS namespace).
- Only `lo`, and it's DOWN: own, empty network namespace.
- `6.8.0-45-generic`: **the host's kernel**. Shared kernel, exactly as topic 01 said.

**Step 5: watch it from the host** (second terminal).

```bash
cat /sys/fs/cgroup/minic/cgroup.procs
cat /sys/fs/cgroup/minic/memory.current
ps -ef | grep '/bin/sh' | grep -v grep
```

```
51790
1236992
root       51790   51789  0 10:12 pts/1    00:00:00 /bin/sh
```

- `51790` = the container's PID 1, seen from the host with its real PID.
- `memory.current` = bytes used right now (about 1.2 MB).
- From the host it's just a normal process. You've built a container.

**Step 6: test the cgroup limit.** Inside the mini container, try to create too many processes:

```sh
for i in $(seq 1 30); do sleep 100 & done
```

```
sh: can't fork: Resource temporarily unavailable
```

The `pids.max` of 20 stopped it. Exactly what `docker run --pids-limit 20` does.

**Step 7: clean up.** Type `exit` in the mini container, then on the host:

```bash
sudo umount ~/minicontainer/rootfs/proc 2>/dev/null
sudo rmdir /sys/fs/cgroup/minic
```

You can only `rmdir` a cgroup when no processes are left in it.

---

## Where you'll use it

- **Debugging production:** "Is this process inside the container or on the host?" You compare `/proc/PID/ns` numbers. "Why was it killed?" You read the cgroup's memory files. Topic 11 does this with real Docker containers.
- **Kubernetes:** Pods are the same thing: a group of processes sharing some namespaces (network, IPC) with cgroups for limits. Requests and limits in Kubernetes become cgroup files.
- **Security reviews:** knowing that containers share the kernel tells you why `--privileged` or a mounted Docker socket is dangerous (topic 40).
- **Interviews:** "Explain namespaces and cgroups" is a classic. Saying "namespaces = what you can see, cgroups = what you can use" and naming the namespace types is a strong answer.

---

## When it breaks

| Symptom | Meaning | Fix / check |
|---|---|---|
| `unshare: unshare failed: Operation not permitted` | Creating most namespaces needs root (or a user namespace). | Use `sudo`, or check if your environment (some CI runners, some containers) blocks it. |
| `fork: Cannot allocate memory` right after `unshare --pid` | You forgot `--fork`. The PID namespace's first process exited, so no new process can be created in it. | Add `--fork`. |
| `ps` inside shows all host processes | `/proc` is still the host's `/proc`. | Use `--mount-proc`, or mount a fresh `proc` inside. |
| `echo 50M > memory.max: Permission denied` | You used `sudo echo ... > file`. The redirect runs as your user, not root. | Use `echo 50M \| sudo tee file`. |
| `No such file or directory` for `memory.max` | The memory controller isn't enabled in the parent's `cgroup.subtree_control`, or you're on cgroups v1. | `cat /sys/fs/cgroup/cgroup.subtree_control`; check `stat -fc %T /sys/fs/cgroup`. |
| `rmdir: failed to remove ... Device or resource busy` | Processes are still in the cgroup. | Kill them or move them out first. |

---

## Quick recap

- A container is a normal Linux process with **namespaces** (its own view), **cgroups** (its own limits) and its **own root filesystem** (files from the image).
- The kernel has no "container" object. `/proc/PID/ns/*` shows which namespaces a process is in. Same number = same namespace.
- cgroups v2 lives at `/sys/fs/cgroup`. You make groups by making folders, and set limits by writing files like `memory.max`.
- The container's `/` is a folder on the host. Real runtimes switch to it with `pivot_root`.
- All containers use the host's kernel. `uname -r` inside = `uname -r` outside.

**Interview questions**

1. **Q: What are namespaces and cgroups?**
   A: Namespaces isolate what a process can see: PIDs, network, mounts, hostname, IPC, users. cgroups limit and measure what it can use: memory, CPU, I/O, number of processes. A container is a process using both, plus its own root filesystem.

2. **Q: Which namespaces does Docker use?**
   A: mnt, pid, net, uts, ipc and cgroup by default. The user namespace is optional (userns-remap or rootless mode). The time namespace isn't used.

3. **Q: A process is PID 1 in a container. What is it on the host?**
   A: It's a normal process with a normal host PID like 48213. A PID namespace gives one process two PIDs: one inside the namespace and one on the host.
