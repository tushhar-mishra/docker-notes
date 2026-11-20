# 48a · Host Tuning for Container Hosts

> **Level:** ★★★ advanced
>
> **You will learn:** the kernel and OS settings that matter on a host running many containers, as **one checklist**: which `sysctl`s are **per container** (namespaced) and which are **host-wide**, limits that cause classic failures (conntrack table, ARP/neighbour table, inotify watches, open files, PIDs, memory maps), settings popular images complain about (Elasticsearch `vm.max_map_count`, Redis overcommit and transparent huge pages), journald and time sync, how to make all of it persistent and automated, and how to recognise and handle a host still on **cgroup v1**.
>
> **You need:** topic 26 (conntrack, bridges), topic 31 (cgroups, PIDs), topic 34 (daemon config), topic 41 (`--sysctl`), topic 48 (disks and inodes).

---

## What it is

Linux defaults are designed for a general-purpose machine running a few services. A container host may run **hundreds of processes in dozens of network namespaces**, with heavy NAT, many open connections, many file watchers and big data services. Several kernel limits that never matter on a laptop become **production incidents** on such hosts.

Two categories of setting:

| Kind | Where it lives | How you set it |
|---|---|---|
| **Namespaced** (per network or IPC namespace) | Each container has its own copy | `docker run --sysctl key=value`, Compose `sysctls:`, Kubernetes `securityContext.sysctls` |
| **Host-wide** (`vm.*`, `fs.*`, most `kernel.*`, conntrack limits) | One value for the whole kernel | On the host: `/etc/sysctl.d/*.conf`. **Containers can't change them** (topic 41) |

---

## Why it exists

The failures these limits cause are hard to diagnose because they look random:
- connections dropped "sometimes" (conntrack table full),
- containers can't reach their neighbours after scaling (ARP/neighbour table overflow),
- `docker compose watch`, log shippers or IDEs failing with "no space left on device" on a disk that's 20% full (inotify watches),
- "too many open files" under load (file descriptor limits),
- Elasticsearch refusing to start, Redis logging warnings and failing background saves.

Knowing the checklist turns a day of confusion into one `dmesg` line and one setting.

---

## Analogy

A **stadium** built for a small concert, now hosting a football final. The building is fine, but the number of turnstiles (conntrack entries), the size of the guest list at each gate (neighbour table), the number of radios for security staff (inotify watches) and the number of lockers (file descriptors) were all sized for a small event. Nothing is broken; the limits are just too small for the crowd.

---

## How it works inside: the checklist

### 1. Networking (host-wide)

| Setting | Default (typical) | Why it matters | Suggested start |
|---|---|---|---|
| `net.ipv4.ip_forward` | 0 | Required to route container traffic. **Docker sets it to 1** at startup (topic 26) | Leave to Docker; make sure no security tool resets it |
| `br_netfilter` module + `net.bridge.bridge-nf-call-iptables` / `-ip6tables` | Module may not be loaded | Lets iptables see bridged traffic. Docker loads/sets it; `docker info` warns if disabled | Load at boot: `/etc/modules-load.d/br_netfilter.conf` |
| `net.netfilter.nf_conntrack_max` | Scales with RAM (e.g. 65536–262144) | Every NATed/tracked connection uses an entry. Full table → **`nf_conntrack: table full, dropping packet`** and random connection failures (topic 30) | Size from peak connections × 2; e.g. 524288–1048576 on busy hosts |
| `nf_conntrack` hashsize (module parameter `/sys/module/nf_conntrack/parameters/hashsize`) | Derived from max | Lookup speed | About `nf_conntrack_max / 4` |
| `net.netfilter.nf_conntrack_tcp_timeout_established` | 432000 (5 days) | Dead connections occupy entries for days | Often lowered (e.g. 86400) on NAT-heavy hosts |
| `net.ipv4.neigh.default.gc_thresh1/2/3` | 128 / 512 / 1024 | The ARP/neighbour table holds one entry per IP the host talks to on directly-connected networks (every container on every bridge). Overflow → **`neighbour: arp_cache: neighbor table overflow!`** and unreachable containers | Raise to e.g. 4096 / 8192 / 16384 on dense hosts |

### 2. Networking (per container: set with `--sysctl`)

| Setting | Why |
|---|---|
| `net.core.somaxconn` | Length of the queue of accepted-but-not-yet-handled connections. Busy servers (nginx, Node) under bursts drop SYNs if it's too small (`ss -lnt` shows `Recv-Q` near `Send-Q`). Default is 4096 on newer kernels, 128 on old ones |
| `net.ipv4.ip_local_port_range` | Range of source ports for **outgoing** connections. Services making many outbound calls run out (`EADDRNOTAVAIL`, "Cannot assign requested address") |
| `net.ipv4.tcp_tw_reuse` | Reuse TIME_WAIT sockets for new outgoing connections (helps the same problem) |
| `net.ipv4.ip_unprivileged_port_start` | Docker sets 0 per container (topic 20) |
| `net.ipv4.ping_group_range` | Docker sets it so ping works without `NET_RAW` (topic 41) |

These live in each container's **network namespace**, so changing them on the host does **not** affect existing containers. Set them per service.

### 3. Files and watchers (host-wide)

| Setting | Why it matters | Suggested start |
|---|---|---|
| `fs.inotify.max_user_watches` | **inotify** (the kernel's file-change notification system, topic 38) has a per-user limit on watched files. File watchers in many containers (dev tools, `compose watch`, log shippers, config reloaders) share the limit **per UID across the whole host** (since kernel 5.11 the count is kept per user *per user namespace*, but without userns remapping, container root is host UID 0, so in practice it is host-wide). Exhausted → `inotify_add_watch` fails with **"no space left on device"** even though the disk is fine | 524288 |
| `fs.inotify.max_user_instances` | Number of inotify instances per user. Exhausted → **"too many open files"** from watcher setup | 1024–8192 |
| `fs.file-max` / `fs.nr_open` | System-wide / per-process maximum open files | Usually large already. Check with `cat /proc/sys/fs/file-nr` |
| Container `nofile` ulimit | Per-container open files limit: `--ulimit nofile=65536:65536` or daemon `default-ulimits` (topic 34) | 65536 for servers with many connections |
| dockerd's own limits | The systemd unit sets `LimitNOFILE=infinity` (topic 34) | Leave as is |

### 4. Memory and processes (host-wide)

| Setting | Why | Suggested |
|---|---|---|
| `vm.max_map_count` | Max memory-mapped areas per process. **Elasticsearch/OpenSearch refuse to start** below 262144: `max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]` | 262144 (or higher) on hosts running them |
| `vm.overcommit_memory` | Redis warns `WARNING Memory overcommit must be enabled!` (older versions say `overcommit_memory is set to 0`), because background saves (fork) can fail | `1` on Redis hosts (understand the trade-off: the kernel promises more memory than exists, and the OOM killer decides later) |
| Transparent Huge Pages (`/sys/kernel/mm/transparent_hugepage/enabled`) | Redis and MongoDB warn that THP causes latency and memory spikes | `madvise` or `never` for those hosts (set at boot via a systemd unit or kernel parameter) |
| `vm.swappiness` / swap | Swap hides memory problems as slowness. Containers' swap is controlled by `--memory-swap` (topic 31) | Many container hosts run with little or no swap; Kubernetes historically required swap off |
| `kernel.pid_max` / `kernel.threads-max` | Hundreds of containers × threads can hit the host-wide PID limit (default 32768 on some systems; 4194304 on newer 64-bit defaults) | `kernel.pid_max = 4194304`; per-container `--pids-limit` still required (topic 31) |
| `kernel.keys.maxkeys` / `maxbytes` | Each container's runtime may create a kernel keyring for the session; very dense hosts hit the per-user keyring quota (`Disk quota exceeded` errors at container start) | Raise only if you see that error |

### 5. Logging, time and disks

- **journald** limits: `SystemMaxUse=` (e.g. 2G) in `/etc/systemd/journald.conf`, so the journal can't fill `/`. Especially important with the `journald` log driver (topic 33).
- **Time sync**: containers use the host clock. Run **chrony**/`systemd-timesyncd` and alert on drift (TLS, logs, tokens and distributed systems all break with wrong time).
- **Disks** (topic 48): a separate filesystem for `/var/lib/docker` and `/var/lib/containerd`, ext4 or XFS `ftype=1`, enough inodes, monitoring.

### 6. cgroup v1 hosts (legacy)

Most of these notes assume **cgroup v2** (topic 02). You'll still meet **cgroup v1** on older systems: RHEL/CentOS 7 and 8, Ubuntu 20.04 and earlier, and older cloud images.

Check:

```bash
stat -fc %T /sys/fs/cgroup         # cgroup2fs = v2 ; tmpfs = v1 (or hybrid)
docker info --format '{{.CgroupVersion}} {{.CgroupDriver}}'
```

What changes on v1:

| Concept | cgroup v2 file (topic 31) | cgroup v1 equivalent |
|---|---|---|
| Container's cgroup | `/sys/fs/cgroup/system.slice/docker-<id>.scope/` | **One tree per controller**: `/sys/fs/cgroup/memory/docker/<id>/`, `/sys/fs/cgroup/cpu,cpuacct/docker/<id>/`, … (or `system.slice/docker-<id>.scope` under each controller with the systemd driver) |
| Memory limit / usage | `memory.max` / `memory.current` | `memory.limit_in_bytes` / `memory.usage_in_bytes` |
| OOM kills | `memory.events` → `oom_kill` | `memory.oom_control` → `oom_kill` (newer kernels), `memory.failcnt` |
| CPU limit | `cpu.max` (`quota period`) | `cpu.cfs_quota_us` + `cpu.cfs_period_us` |
| Throttling | `cpu.stat` → `nr_throttled`, `throttled_usec` | `cpu.stat` → `nr_throttled`, `throttled_time` (**nanoseconds**) |
| CPU weight | `cpu.weight` | `cpu.shares` (native) |
| PIDs | `pids.max` | `pids/…/pids.max` |
| Memory "working set" | `memory.current` − inactive file | `usage_in_bytes` − `total_inactive_file` |

Why move off v1: no unified accounting (memory and I/O writeback were tracked separately), weaker rootless support (topic 42), no PSI (pressure stall information). Most importantly, **support is ending**: recent Docker Engine releases have deprecated cgroup v1, and Kubernetes has moved it to maintenance mode, with newer kubelet versions refusing to run on v1 by default. Check the release notes for the versions you run.

Switching a systemd host to v2: kernel parameter `systemd.unified_cgroup_hierarchy=1` (via GRUB, then reboot). RHEL 8 supports it; RHEL 9+ and Ubuntu 22.04+ use v2 by default. Test monitoring agents and old JVMs (older Java 8 builds only understand v1 limits) before switching.

---

## Commands / files

### See current values and warnings

```bash
sysctl net.netfilter.nf_conntrack_max net.ipv4.neigh.default.gc_thresh3 fs.inotify.max_user_watches fs.inotify.max_user_instances vm.max_map_count vm.overcommit_memory kernel.pid_max
cat /sys/kernel/mm/transparent_hugepage/enabled
docker info 2>&1 | grep -i warning
```

```
net.netfilter.nf_conntrack_max = 262144
net.ipv4.neigh.default.gc_thresh3 = 1024
fs.inotify.max_user_watches = 65536
fs.inotify.max_user_instances = 128
vm.max_map_count = 65530
vm.overcommit_memory = 0
kernel.pid_max = 4194304
always [madvise] never
```

- `[madvise]` = the active THP mode (in brackets).
- No `docker info` warnings = good. Typical warnings: `bridge-nf-call-iptables is disabled`, `No swap limit support`.

### Usage against the limits

```bash
echo "conntrack: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
echo "neighbours: $(ip -4 neigh show | wc -l) (gc_thresh3=$(sysctl -n net.ipv4.neigh.default.gc_thresh3))"
echo "open files: $(cut -f1 /proc/sys/fs/file-nr) / $(cat /proc/sys/fs/file-max)"
echo "inotify instances per user:"; sudo find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | cut -d/ -f3 | xargs -r -n1 ps -o user= -p 2>/dev/null | sort | uniq -c | sort -rn | head -3
echo "PIDs: $(ls -d /proc/[0-9]* | wc -l) / $(cat /proc/sys/kernel/pid_max)"
sudo dmesg -T | grep -E 'conntrack: table full|neighbor table overflow|Out of memory' | tail -3
```

```
conntrack: 18234 / 262144
neighbours: 214 (gc_thresh3=1024)
open files: 9856 / 9223372036854775807
inotify instances per user:
     97 tushar
     12 root
PIDs: 1873 / 4194304
```

### Make the settings persistent

```bash
sudo tee /etc/sysctl.d/90-container-host.conf > /dev/null <<'EOF'
# --- container host tuning (review values for your workload) ---
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.ipv4.neigh.default.gc_thresh1 = 4096
net.ipv4.neigh.default.gc_thresh2 = 8192
net.ipv4.neigh.default.gc_thresh3 = 16384
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192
vm.max_map_count = 262144
kernel.pid_max = 4194304
EOF
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf
echo nf_conntrack | sudo tee /etc/modules-load.d/nf_conntrack.conf
sudo modprobe br_netfilter nf_conntrack
sudo sysctl --system | tail -5
```

- `/etc/sysctl.d/*.conf` files are applied at boot (and by `sysctl --system` now).
- `nf_conntrack_*` keys exist only once the `nf_conntrack` module is loaded (Docker loads it when it creates NAT rules), so loading it at boot via `modules-load.d` avoids "unknown key" errors early in boot.
- Put this file in **config management** (Ansible role, cloud-init) so every host matches.

THP at boot (a small systemd unit):

```ini
# /etc/systemd/system/thp-madvise.service
[Unit]
Description=Set Transparent Huge Pages to madvise
Before=docker.service
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo madvise > /sys/kernel/mm/transparent_hugepage/enabled'
[Install]
WantedBy=multi-user.target
```

### Per-container settings

```bash
docker run -d --name busyweb \
  --sysctl net.core.somaxconn=8192 \
  --sysctl net.ipv4.ip_local_port_range="10240 65535" \
  --ulimit nofile=65536:65536 \
  nginx:1.27-alpine
docker exec busyweb sh -c 'cat /proc/sys/net/core/somaxconn; ulimit -n'
```

```
8192
65536
```

Compose:

```yaml
services:
  web:
    sysctls:
      net.core.somaxconn: 8192
    ulimits:
      nofile: { soft: 65536, hard: 65536 }
```

### Elasticsearch: the classic

```bash
docker run --rm -e discovery.type=single-node docker.elastic.co/elasticsearch/elasticsearch:8.15.0 2>&1 | grep -m1 max_map_count
```

```
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

(Single-node dev mode may only warn instead of failing. Production mode fails the bootstrap check.) Fix on the **host**: `sudo sysctl -w vm.max_map_count=262144`, plus the persistent file above.

---

## Hands-on lab

1. Run the "see current values" and "usage against limits" commands on your lab host. Write down which values would be too small for a host running 200 containers.
2. **inotify exhaustion**: temporarily lower the limit and trigger the "no space left on device" error:
   ```bash
   OLD=$(sysctl -n fs.inotify.max_user_watches)
   sudo sysctl -w fs.inotify.max_user_watches=100
   docker run --rm -v /usr:/w:ro alpine sh -c 'apk add -q inotify-tools && inotifywait -r -m /w 2>&1 | head -3'
   sudo sysctl -w fs.inotify.max_user_watches=$OLD
   ```
   Expect `Failed to watch /w; upper limit on inotify watches reached!`. The underlying syscall error is "no space left on device". The limit is per UID **on the whole host**, so the container's root (UID 0) shares it with every root process on the host. Do this only on a lab machine, and restore the old value right away: other services that watch files may misbehave while it's at 100.
3. Run the Elasticsearch check, then fix `vm.max_map_count` on the host and run it again.
4. Start nginx with custom `--sysctl` and `--ulimit` values and verify them from inside.
5. Write your own `/etc/sysctl.d/90-container-host.conf` with a comment explaining each line. That file is a good interview talking point.
6. Clean up: `docker rm -f busyweb`

---

## Where you'll use it

- **Host provisioning** (Ansible/cloud-init/golden AMIs): the sysctl file, module loading, journald limits, time sync, THP setting.
- **Incident response**: `dmesg` lines like `table full`, `neighbor table overflow`, inotify errors, `Cannot assign requested address`.
- **Data services** in containers (Elasticsearch, Redis, MongoDB, Kafka) that check host settings.
- **Kubernetes nodes**: identical concerns. Node images and DaemonSets set them, and "unsafe" sysctls must be allowed per kubelet.
- **Legacy estates**: recognising cgroup v1 and planning the move to v2.

---

## When it breaks

| Symptom / log line | Cause | Fix |
|---|---|---|
| `nf_conntrack: table full, dropping packet` | Conntrack table exhausted | Raise `nf_conntrack_max` (+ hashsize); lower long timeouts; find the connection-heavy service |
| `neighbour: arp_cache: neighbor table overflow!`, containers intermittently unreachable | Neighbour table too small for the number of containers/IPs | Raise `gc_thresh1/2/3` |
| Watchers fail: "no space left on device" with free disk | inotify watches exhausted (per UID, host-wide) | Raise `max_user_watches` |
| "too many open files" when starting watchers | inotify instances exhausted | Raise `max_user_instances` |
| "too many open files" under load | Process `nofile` limit | `--ulimit nofile=…` / `default-ulimits` |
| `Cannot assign requested address` on outbound calls | Ephemeral ports exhausted | Wider `ip_local_port_range`, connection pooling/keep-alive, `tcp_tw_reuse` |
| Elasticsearch: `vm.max_map_count [65530] is too low` | Host-wide setting too low | Set on the host |
| `sysctl ... is not in a separate kernel namespace` | Tried to set a host-wide sysctl per container | Set it on the host instead |
| `sysctl: cannot stat /proc/sys/net/netfilter/nf_conntrack_max` at boot | Module not loaded yet | `modules-load.d/nf_conntrack.conf` |
| Monitoring shows no container memory/CPU data after an OS upgrade | cgroup version changed (v1 ↔ v2) and the agent reads old paths | Upgrade the agent; check `stat -fc %T /sys/fs/cgroup` |

---

## Quick recap

- Know which settings are **namespaced** (set per container: `somaxconn`, `ip_local_port_range`, other `net.*`) and which are **host-wide** (`vm.*`, `fs.*`, conntrack, neighbour table, `pid_max`). Containers can't change host-wide ones.
- Classic dense-host limits: **conntrack**, **neighbour table**, **inotify** (per UID, host-wide), **open files**, **ephemeral ports**, **PIDs**.
- Data-service prerequisites: `vm.max_map_count` (Elasticsearch), overcommit + THP (Redis/MongoDB).
- Persist with `/etc/sysctl.d/` + `/etc/modules-load.d/`, managed by config management. Cap journald, sync time, use separate Docker disks.
- **cgroup v1** hosts: different file paths and units (`cpu.cfs_*`, `throttled_time` in ns, `memory.limit_in_bytes`). Support is ending in Docker and Kubernetes, so plan the move to v2.

**Interview questions**

1. **Q: Containers on a busy host randomly fail to connect, and `dmesg` shows "nf_conntrack: table full". Explain and fix.**
   A: Every NATed or tracked connection uses a conntrack entry, and the table hit `nf_conntrack_max`, so new connections are dropped. Raise `nf_conntrack_max` (and hashsize) via `/etc/sysctl.d`, reduce long established timeouts if appropriate, and find which service opens so many connections (pooling, keep-alive). Then alert on count/max.

2. **Q: Why can't you set `vm.max_map_count` with `docker run --sysctl`?**
   A: It isn't namespaced: it's a host-wide kernel setting. Only namespaced sysctls (mostly `net.*` and some IPC `kernel.*`) can be set per container. Host-wide ones must be set on the host, usually in `/etc/sysctl.d`.

3. **Q: A file watcher fails with "no space left on device" but `df` shows plenty of space. Why?**
   A: That's inotify's watch limit (`fs.inotify.max_user_watches`), which is per UID and shared across all containers on the host. `inotify_add_watch` returns ENOSPC when it's exhausted. Raise the limit on the host.
