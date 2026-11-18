# 31 · Resource Limits

> **Level:** ★★★ advanced
>
> **You will learn:** how to limit a container's memory, CPU, processes and disk I/O, exactly which **cgroup v2 files** each `docker run` flag writes, what happens when a limit is hit (**OOM kill** for memory, **throttling** for CPU), why apps inside containers often misjudge their limits (Java, Node, Go, `nproc`, `/proc/meminfo`), how to read the real numbers, and a short note on GPUs.
>
> **You need:** topic 02 (cgroups intro), topic 08 (exit 137), topic 11 (reading cgroup files from the host).

---

## What it is

By default, **a container has no limits**. It can use all the host's RAM and all its CPUs, like any other process. Resource limits put a container's processes in a cgroup with caps:

| Flag | cgroup v2 file | Effect when exceeded |
|---|---|---|
| `--memory 512m` (`-m`) | `memory.max` | **OOM kill** (a process is killed) |
| `--memory-reservation 256m` | `memory.low` | Soft: protected from reclaim up to this amount |
| `--memory-swap 1g` | `memory.swap.max` (= swap value − memory value) | Total memory + swap cap |
| `--cpus 1.5` | `cpu.max` (`150000 100000`) | **Throttled** (paused until the next period) |
| `--cpu-shares 512` | `cpu.weight` | Relative share, only matters when CPUs are busy |
| `--cpuset-cpus 0,1` | `cpuset.cpus` | Can only run on those cores |
| `--pids-limit 200` | `pids.max` | `fork` fails ("Resource temporarily unavailable") |
| `--device-read-bps /dev/sda:10mb` | `io.max` | Disk reads slowed down |
| `--blkio-weight 300` | `io.weight` | Relative disk share |

---

## Why it exists

- **Noisy neighbours:** one container leaking memory or spinning CPU must not take down the whole host (and every other container, plus SSH, plus dockerd).
- **Predictability:** a service with known limits behaves the same in test and production.
- **Capacity planning:** limits let you pack hosts safely.
- **Safety:** a fork bomb or runaway thread pool is stopped by `pids.max`.
- **Kubernetes:** Pod `resources.limits` become exactly these cgroup files. If you understand this topic, you understand Kubernetes OOMKilled and CPU throttling.

---

## Analogy

A **shared house with utility meters per room**:
- **Memory limit** = a fuse per room. Plug in too much and the fuse trips: the appliance (process) is switched off **instantly**.
- **CPU limit** = a hot-water allowance that refills every **100 milliseconds** (the CFS period). Use it all up in the first 20 ms, and everyone in the room waits 80 ms for the refill. That repeated short wait is exactly what shows up as latency spikes. You aren't thrown out, just **made to wait** (throttled).
- **CPU shares** = when everyone wants hot water at the same time, rooms with more shares get more of it. When nobody else wants it, anyone can use all of it.
- **PIDs limit** = a maximum number of people allowed in a room.

---

## How it works inside

### Where the cgroup is

With the systemd cgroup driver (default): `/sys/fs/cgroup/system.slice/docker-<full-id>.scope/` (topic 11). `docker run` → runc creates the cgroup and writes the values **before** your process starts. `docker update` rewrites them **live**.

### Memory in detail

- **`memory.max`** (the hard limit): when the cgroup's memory reaches it, the kernel first tries to **reclaim** memory (drop clean page cache, swap if allowed). If it can't free enough, the **OOM killer** runs **inside that cgroup** and kills the process with the highest "badness" score (usually the biggest one). It's often PID 1, which stops the container (exit **137**, `OOMKilled: true`). If a **child** process is killed instead, the container keeps running, but something inside it died. Only `memory.events` (`oom_kill` counter) shows that.
- **What counts as memory:** anonymous memory (heap, stacks), **page cache** for files the container read or wrote, **tmpfs** content (topic 25), kernel memory for its sockets and so on. Page cache can usually be reclaimed. tmpfs and anonymous memory can't (without swap).
- **`memory.high`**: a throttle point below `max`. Over it, the kernel aggressively reclaims and slows the cgroup down instead of killing. (Docker doesn't set it by default. Kubernetes can.)
- **`memory.low`** (`--memory-reservation`): memory protected from reclaim when the **host** is under pressure.
- **Swap:** `--memory-swap` is the **total** of memory + swap. `-m 512m --memory-swap 512m` = **no swap**. `--memory-swap 1g` = up to 512 MB swap. If you don't set it, Docker allows swap equal to the memory limit (when the host has swap). Swap usage hides OOM problems behind slowness, so production containers usually disable it.
- **Host-level OOM:** if the **whole host** runs out of memory (containers without limits), the kernel's global OOM killer picks a victim from **anywhere**, possibly dockerd or sshd. Docker lowers dockerd's chance of being picked (`oom_score_adj -500`), but setting container limits is the real fix.

### CPU in detail

- **`cpu.max` = "quota period"** (the CFS bandwidth controller; CFS = Completely Fair Scheduler, Linux's CPU scheduler). `--cpus 1.5` → `150000 100000`: in every **100 ms period**, the cgroup's threads together may use **150 ms of CPU time** (across any cores).
  - A multi-threaded app with 8 busy threads burns 150 ms of quota in about **19 ms of wall time**, then **all its threads are paused for the remaining ~81 ms**. That's **throttling**. Average CPU looks low, but requests hit 80 ms pauses: **latency spikes**.
  - Visible in `cpu.stat`: `nr_throttled`, `throttled_usec`.
- **`cpu.weight`** (`--cpu-shares` is a cgroup v1 idea; on v2 the runtime converts it to `cpu.weight` (range 1–10000). If you don't set it, the weight stays at the default 100. **The conversion formula depends on the runtime version**: older runc/crun used a linear formula (1024 → 39), and newer releases changed it so the default 1024 maps to 100. Don't memorise numbers. Check `cat $CG/cpu.weight` on your system): only used when CPUs are **contended**. A container with twice the weight gets twice the CPU time **when there's competition**. With no competition, anyone can use idle CPU. It never throttles.
- **`cpuset.cpus`** (`--cpuset-cpus`): pins the container to specific cores. Good for latency-sensitive workloads and NUMA (Non-Uniform Memory Access) placement, and it's the one CPU limit that `nproc` sees.

### PIDs

- `pids.max` caps processes **and threads** in the cgroup. It protects against fork bombs and zombie pile-ups (topic 12). Docker's default is unlimited. Set something like `--pids-limit 512` for most services.

### Disk I/O

- `io.max` (`--device-read-bps`, `--device-write-bps`, `--device-read-iops`, `--device-write-iops`) and `io.weight` (`--blkio-weight`). They only control **direct** I/O well. Buffered writes are attributed through the page cache (works with cgroup v2 writeback, with caveats). Disk **space** isn't limited by cgroups (overlay storage quotas need XFS project quotas and the `--storage-opt size=` option, rarely used).

### Why apps misjudge their limits

Inside a container:
- `/proc/meminfo` and `free` show the **host's** memory, not the limit.
- `nproc` / `os.cpu_count()` show **all host CPUs** (unless `--cpuset-cpus`), not `--cpus`.

So apps that size themselves from these numbers get it wrong:

| Runtime | Container-aware? | What to do |
|---|---|---|
| **Java** (JDK 11+; 8u191+) | Yes: reads cgroup memory and CPU limits | Set heap as a percentage: `-XX:MaxRAMPercentage=75` (the default is only 25%) |
| **Go** | Go 1.25+: `GOMAXPROCS` follows the CPU limit automatically. Memory isn't automatic | Set `GOMEMLIMIT` to ~90% of the limit |
| **Node.js** | Heap size is partly based on the limit in recent versions, but don't rely on it | `--max-old-space-size=<MB>` at ~75% of the limit |
| **Python** | No | Size worker counts (gunicorn `-w`) yourself from the limit |
| **.NET** | Yes | Defaults are usually fine |
| Thread-pool libraries, databases | Often no | Configure explicitly (e.g. Postgres `shared_buffers`) |

A common incident: a JVM on an old JDK thinks it has 64 GB (host RAM), grows its heap past the 2 GB limit, and gets OOM-killed in a loop.

### The `docker stats` memory number

`docker stats` shows `memory.current` **minus inactive file cache**. Kubernetes uses a similar "working set" number. Raw `memory.current` includes all page cache, which is why it looks higher. When diagnosing OOM, read `memory.stat` (anon vs file vs shmem).

---

## Commands / files

### Set limits and read them back

```bash
docker run -d --name lim --memory 256m --memory-swap 256m --cpus 0.5 --pids-limit 100 nginx:1.27-alpine
docker inspect lim -f 'mem={{.HostConfig.Memory}} swap={{.HostConfig.MemorySwap}} nanocpus={{.HostConfig.NanoCpus}} pids={{.HostConfig.PidsLimit}}'
CG=/sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' lim).scope
cat $CG/memory.max $CG/memory.swap.max $CG/cpu.max $CG/pids.max
```

```
mem=268435456 swap=268435456 nanocpus=500000000 pids=100
268435456
0
50000 100000
100
```

- `NanoCpus 500000000` = 0.5 CPU (in billionths of a CPU).
- `memory.swap.max 0` = no swap (swap total equals memory).
- `cpu.max 50000 100000` = 50 ms per 100 ms.

### Watch an OOM kill

```bash
docker run --name oom --memory 100m --memory-swap 100m python:3.13-slim \
  python -c "a=[]; [a.append(' '*10_000_000) for _ in range(100)]"
echo "exit: $?"
docker inspect oom -f 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
sudo dmesg | grep -i 'out of memory' | tail -1
```

```
exit: 137
OOMKilled=true ExitCode=137
[12345.678901] Memory cgroup out of memory: Killed process 61234 (python) total-vm:131072kB, anon-rss:101420kB, file-rss:4096kB, shmem-rss:0kB, UID:0 pgtables:280kB oom_score_adj:0
```

- The kernel log names the cgroup OOM, the process, and its memory (`anon-rss` ≈ 100 MB: it hit the limit).
- `docker events --filter event=oom` also reports it live.

### A child OOM that doesn't stop the container

```bash
docker run -d --name childoom --memory 100m --memory-swap 100m python:3.13-slim \
  sh -c 'python -c "a=\" \"*200_000_000" ; echo "child died, shell lives on"; sleep 3600'
sleep 3
docker ps --filter name=childoom --format '{{.Status}}'
docker logs childoom
CG=/sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' childoom).scope
grep oom_kill $CG/memory.events
```

```
Up 5 seconds
child died, shell lives on
oom_kill 1
```

- The container is "Up", but a process inside was OOM-killed. **Monitor `oom_kill` in `memory.events`** (or the `oom` events), not just container exit codes.

### Watch CPU throttling

```bash
docker run -d --name burn --cpus 0.5 alpine sh -c 'while :; do :; done'
sleep 10
docker stats --no-stream burn
CG=/sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' burn).scope
cat $CG/cpu.stat
```

```
NAME   CPU %     MEM USAGE / LIMIT   ...
burn   50.12%    396KiB / 7.66GiB    ...
usage_usec 5021334
user_usec 5019876
system_usec 1458
nr_periods 100
nr_throttled 100
throttled_usec 4973210
```

- Capped at exactly 50%.
- `nr_throttled 100` of `nr_periods 100`: throttled in **every** period, about 50 ms each time.

### Change limits live

```bash
docker update --cpus 1 --memory 512m --memory-swap 512m burn
cat $CG/cpu.max $CG/memory.max
```

```
100000 100000
536870912
```

### What the container thinks it has

```bash
docker run --rm --memory 256m --cpus 1 alpine sh -c 'free -m | head -2; nproc'
docker run --rm --cpuset-cpus 0 alpine nproc
```

```
              total        used        free      shared  buff/cache   available
Mem:           7843        2102        3140          12        2600        5490
8
1
```

- `free` shows 7.8 GB (host), not 256 MB. `nproc` shows 8 (host CPUs), not 1. Only `--cpuset-cpus` changes `nproc`.

Read the real limit from inside:

```bash
docker run --rm --memory 256m --cpus 1.5 alpine sh -c 'cat /sys/fs/cgroup/memory.max /sys/fs/cgroup/cpu.max'
```

```
268435456
150000 100000
```

### GPUs (one paragraph)

To give a container NVIDIA GPUs, install the **NVIDIA Container Toolkit** on the host. It's a runtime hook that mounts the GPU device files and driver libraries into the container. Then `docker run --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi`, or `--gpus '"device=0,1"'` for specific GPUs. Newer setups use **CDI** (Container Device Interface, a standard way to describe devices for runtimes): `docker run --device nvidia.com/gpu=all ...`. GPU memory isn't controlled by cgroups. The GPU driver manages it.

---

## Hands-on lab

1. Run the OOM example, then rerun it with `--memory 300m` and see it succeed.
2. Run the child-OOM example and find the evidence three ways: `memory.events`, `dmesg`, and `docker events --since 5m --filter event=oom`.
3. **Throttling vs shares:** start two CPU burners **without** `--cpus`, but with `--cpu-shares 1024` and `--cpu-shares 512`, both pinned to the same core (`--cpuset-cpus 0`). `docker stats` shows the higher-weight one getting the bigger share. First read both containers' `cpu.weight` files: the split follows the ratio of **those** numbers, which depends on your runtime's conversion formula (often close to 2:1, but not guaranteed). Stop one, and the other goes to 100%. Weights only matter under contention.
4. **Fork bomb protection:**
   ```bash
   docker run --rm --pids-limit 50 alpine sh -c 'for i in $(seq 1 100); do sleep 30 & done; wait'
   ```
   You'll see `can't fork: Resource temporarily unavailable` after ~50.
5. **JVM sizing:**
   ```bash
   docker run --rm --memory 1g eclipse-temurin:21-jre java -XX:+PrintFlagsFinal -version 2>/dev/null | grep -E ' MaxHeapSize '
   docker run --rm --memory 1g eclipse-temurin:21-jre java -XX:MaxRAMPercentage=75 -XX:+PrintFlagsFinal -version 2>/dev/null | grep -E ' MaxHeapSize '
   ```
   About 256 MB (25% default) vs about 768 MB.
6. Clean up: `docker rm -f lim oom childoom burn 2>/dev/null`

---

## Where you'll use it

- **Every production container** should have memory limits (and usually a PID limit). CPU limits are a deliberate choice (many teams set only CPU **requests/shares** to avoid throttling).
- **Incident response:** exit 137 → `OOMKilled` → `memory.stat`/`dmesg` → leak or undersized limit. Latency spikes → `cpu.stat` throttling.
- **Runtime tuning:** JVM `MaxRAMPercentage`, `GOMEMLIMIT`, Node `--max-old-space-size`, worker counts.
- **Kubernetes:** `requests.cpu` → `cpu.weight`, `limits.cpu` → `cpu.max`, `limits.memory` → `memory.max`. Same OOMKilled and throttling behaviour (topic 53).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Exit 137, `OOMKilled=true` | Hit `memory.max` | Raise the limit, fix the leak, size the runtime's heap below the limit |
| Container "Up", but workers keep dying / requests fail | A child process was OOM-killed | `memory.events` `oom_kill`; same fixes |
| Host freezes, random processes (even sshd) killed | **Host-level** OOM: containers without limits | Set memory limits on all containers; monitor host memory |
| p99 latency spikes, CPU average looks fine | CPU throttling from `cpu.max` | `cpu.stat`; raise `--cpus`, reduce thread counts, or use shares instead of hard limits |
| Java/Node/Go app OOM-killed while "heap usage is fine" | Heap sized from host memory, or non-heap memory (threads, buffers, metaspace) not counted | Set heap % below the limit; leave 20–25% headroom |
| `fork: Resource temporarily unavailable` / `pthread_create failed` | `pids.max` reached (threads count too) | Raise `--pids-limit`; check for zombies or thread leaks |
| `docker stats` shows lower memory than the cgroup file | stats subtracts inactive file cache | Read `memory.stat` for anon vs file |
| `Your kernel does not support swap limit capabilities` warning | Swap accounting is off (older kernels / cgroup v1) | Enable `swapaccount=1` in kernel cmdline, or use cgroup v2 |
| OOM while writing to `/tmp` or `/dev/shm` | tmpfs/shm pages count as memory | Size tmpfs, raise limits (topic 25) |

---

## Quick recap

- No limits by default. Limits = cgroup v2 files: `memory.max`, `cpu.max`, `cpu.weight`, `cpuset.cpus`, `pids.max`, `io.max`.
- **Memory over limit → OOM kill** (exit 137, `OOMKilled`). Child-process kills show only in `memory.events`. Page cache and tmpfs count.
- **CPU over quota → throttled** in each 100 ms period: latency spikes, visible in `cpu.stat`. Shares/weight only matter under contention.
- Inside containers, `free`/`nproc` show host values. Configure runtimes (JVM %, `GOMEMLIMIT`, Node heap) from the cgroup limit.
- `docker update` changes limits live. Always set memory (and PID) limits in production.

**Interview questions**

1. **Q: What happens when a container exceeds its memory limit vs its CPU limit?**
   A: Memory: the kernel tries reclaim, then the OOM killer kills a process in the container's cgroup (often PID 1, giving exit 137 and OOMKilled=true). CPU: the cgroup is throttled. Its threads are paused until the next CFS period, so it runs slower but isn't killed.

2. **Q: A Java service keeps getting OOMKilled even though its heap metrics look fine. Why?**
   A: The JVM uses memory beyond the heap (metaspace, thread stacks, direct buffers, code cache), and the heap may be sized close to or above the container limit. Set `-XX:MaxRAMPercentage` around 70–75 and leave headroom, and check non-heap usage.

3. **Q: How do you tell if a container is CPU-throttled?**
   A: Read `cpu.stat` in its cgroup: rising `nr_throttled` and `throttled_usec` relative to `nr_periods`. In monitoring, use the cAdvisor/Kubernetes metric `container_cpu_cfs_throttled_periods_total`.
