# 40 · Docker's Security Model

> **Level:** ★★ intermediate
>
> **You will learn:** what actually isolates a container (and what doesn't), why "one shared kernel" is the core of every container security discussion, the realistic attack paths (Docker socket, `--privileged`, dangerous mounts, the `docker` group, exposed API, vulnerable or malicious images, leaked secrets, kernel/runtime bugs), a defence-in-depth checklist, stronger-isolation runtimes (gVisor, Kata), and how to audit a host with the **CIS Docker Benchmark** and **docker-bench-security**.
>
> **You need:** topics 02, 03, 11 (namespaces, cgroups, socket, capabilities/seccomp seen from the host), topic 25, 28, 34.

---

## What it is

Container security is **layers**, each covering a different risk:

```
┌───────────────────────────────────────────────────────────────┐
│ Image: minimal base, no secrets, scanned, signed (43, 44)      │
├───────────────────────────────────────────────────────────────┤
│ Process: non-root USER, no-new-privileges (20, 41)             │
├───────────────────────────────────────────────────────────────┤
│ Capabilities: drop ALL, add only what's needed (41)            │
├───────────────────────────────────────────────────────────────┤
│ Seccomp: block dangerous system calls (41a)                    │
├───────────────────────────────────────────────────────────────┤
│ LSM: AppArmor / SELinux mandatory access control (41a)         │
├───────────────────────────────────────────────────────────────┤
│ Namespaces: separate view of PIDs, net, mounts, users… (02)    │
│ User namespace (optional): root inside ≠ root outside (42)     │
├───────────────────────────────────────────────────────────────┤
│ cgroups: limits (memory, CPU, PIDs) against DoS (31)           │
├───────────────────────────────────────────────────────────────┤
│ Read-only root, minimal mounts, no socket (25)                 │
╞═══════════════════════════════════════════════════════════════╡
│        ONE SHARED HOST KERNEL (the real security boundary)     │
├───────────────────────────────────────────────────────────────┤
│ Host: patched kernel, hardened dockerd, restricted access (34) │
└───────────────────────────────────────────────────────────────┘
```

---

## Why it exists

A VM has a hypervisor between guests and hardware: a guest must break the hypervisor to escape. A container's process **talks to the host kernel directly**, through hundreds of system calls. Every kernel bug reachable from a container is a potential escape route. And the default settings leave several **configuration** escape routes open, which are far more common in real incidents than kernel exploits.

Real-world container compromises usually come from:
1. **Misconfiguration**: exposed Docker API, mounted Docker socket, `--privileged`, host path mounts.
2. **Vulnerable app code or dependencies** inside the container (the attacker gets in through the app).
3. **Malicious or outdated images** (crypto-miners in community images, typosquatting).
4. **Leaked secrets** in images, env vars or logs.
5. **Kernel or runtime vulnerabilities** (rarer, but high-impact).

---

## Analogy

Flats in **one apartment building**. Each flat has its own door and walls (namespaces), its own utility limits (cgroups), and residents carry limited keys (capabilities). But they all share **one foundation and one set of pipes** (the kernel). A crack in the foundation affects everyone. And some keys are master keys: the **Docker socket** is the building manager's master key; `--privileged` is "this resident has no walls at all". A VM is a **separate house**: much harder to get from one to another.

---

## How it works inside: the attack paths

### 1. The Docker socket = root on the host

Anyone (or any container) that can talk to `/var/run/docker.sock` can ask dockerd (root) to:

```bash
docker run --rm -it --privileged --pid host -v /:/host alpine chroot /host
```

…and get a **root shell on the host**. So:
- **Mounting the socket into a container gives that container root on the host.** CI runners, Traefik, Portainer and auto-updaters often do this. `:ro` on the socket mount **doesn't help** (it only makes the socket file read-only; API calls still work).
- **The `docker` group = root** (topic 04).
- **TCP 2375 without TLS = root for anyone on the network** (topic 34). Attackers scan the internet for it constantly.
- Mitigations: don't mount it; if you must, use a **socket proxy** that allows only safe read-only endpoints; rootless Docker (topic 42) limits the blast radius.

### 2. `--privileged`

`--privileged` turns off almost every protection:
- **all capabilities** (including `CAP_SYS_ADMIN`, the "new root"),
- **no seccomp** filter, **no AppArmor/SELinux** confinement,
- **all host devices** visible in `/dev` (disks, etc.),
- `/sys` and `/proc` writable in ways normally blocked.

From a privileged container you can mount the host's disk (`mount /dev/sda1 /mnt`), load kernel modules, and more. **Privileged container = root on the host.** Valid uses are rare (Docker-in-Docker, some system tools). Prefer specific `--cap-add` and `--device` flags (topics 41, 41a).

### 3. Dangerous host mounts

| Mount | Why it's dangerous |
|---|---|
| `-v /:/host` | Full read/write of the host's files: edit `/etc/shadow`, cron, SSH keys |
| `-v /var/run/docker.sock:...` | Root via the API (above) |
| `-v /etc:/etc` | Change users, sudoers, systemd units |
| `-v /proc:/host/proc` (writable) or `--pid host` | See and signal all host processes; read their environment and memory maps |
| `-v /dev:/dev` | Raw device access |
| `-v /root/.ssh` / `~/.aws` | Credential theft |

Monitoring agents legitimately mount some of these **read-only**. Review every host mount.

### 4. Root in the container

Without user namespaces, container root **is UID 0** on the kernel (topic 11). It's weakened (14 capabilities, seccomp, LSM), but combined with any of the above (a host mount, a socket, a kernel bug), it becomes host root. Running as non-root (topic 20) is one of the highest-value, lowest-cost defences.

### 5. Kernel and runtime vulnerabilities

Examples (know the shape, not the details):
- **runc CVE-2019-5736**: a malicious container could overwrite the host's `runc` binary when someone `exec`ed into it.
- **"Leaky Vessels" CVE-2024-21626** (runc): a leaked file descriptor let a container access the host filesystem through a crafted `WORKDIR`.
- **Dirty Pipe CVE-2022-0847** (kernel): overwrite read-only files, usable for container escapes.
- New runc and Engine advisories keep appearing (for example the 2025 runc fixes around `/proc` mount handling, and the 2026 Engine 29.x security releases).

Defences: **patch the kernel, runc, containerd and Docker promptly**; run as non-root; keep seccomp/AppArmor/SELinux on; consider user namespaces and sandboxed runtimes for untrusted code.

### 6. Image and supply-chain risks

Malicious images (miners, backdoors), typosquatted names (`ngnix`), outdated bases full of CVEs, secrets left in layers. Covered in topics 43 and 44.

### 7. Resource abuse (DoS)

Without limits, one container can starve the host (memory, CPU, PIDs, disk via logs). Limits (topic 31) and log rotation (topic 33) are security controls too.

### 8. Network exposure

Published ports bypassing ufw/firewalld (topic 28). Flat default bridge networks letting any container reach any other (topic 27).

### Stronger isolation: sandboxed runtimes

When you must run **untrusted** code (customer workloads, CI for external contributors), plain containers aren't enough:

| Runtime | How | Trade-off |
|---|---|---|
| **gVisor** (`runsc`) | A user-space "kernel" (written in Go) intercepts the container's system calls. The host kernel sees far fewer calls | Some syscalls/features unsupported; slower I/O |
| **Kata Containers** | Each container runs inside a lightweight VM with its own kernel | VM-level isolation; more memory and start-up overhead |
| **Firecracker microVMs** | Minimal VMs (AWS Lambda/Fargate use this) | Used through platforms rather than plain Docker |

With Docker, register them as runtimes in `daemon.json` and use `docker run --runtime runsc ...`.

### Defence-in-depth checklist

**Host**
- Patched kernel, Docker, containerd, runc. Minimal host OS.
- Nobody in the `docker` group who shouldn't be root. No TCP API (or TLS + certificates only).
- `daemon.json`: log rotation, `live-restore`, `no-new-privileges: true`, `icc: false` (if you use the default bridge), `userns-remap` where possible.
- Firewall published ports (bind `127.0.0.1`, `DOCKER-USER`, security groups).
- Audit with docker-bench-security (below).

**Images**
- Minimal/distroless bases, multi-stage, pinned + automatically updated, scanned, signed. No secrets.

**Containers**
- Non-root `USER`; `--cap-drop ALL` + minimal `--cap-add`; `--security-opt no-new-privileges`; default (or stricter) seccomp and AppArmor/SELinux.
- `--read-only` + tmpfs; no host mounts except what's necessary (read-only where possible).
- Memory/CPU/PID limits.
- **Never** `--privileged`, the Docker socket, `--pid host`, `--network host` without a strong reason.

### The CIS Docker Benchmark

The **CIS** (Center for Internet Security) **Docker Benchmark** is a free, widely-used checklist of ~100 recommendations grouped as: host configuration, daemon configuration, daemon files permissions, images/build files, container runtime, security operations. Auditors and compliance teams (PCI DSS, SOC 2…) often ask for it. **docker-bench-security** is a script from Docker that checks a host against it.

---

## Commands / files

### Prove the socket = root (on a lab VM only)

Topic 04 showed a `docker` group member reading `/etc/shadow`. Here's the same attack from **inside a container** that has the socket mounted, which is the CI-runner/Traefik situation:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker:cli \
  docker run --rm -v /:/host alpine head -1 /host/etc/shadow
```

- The first container has no special flags, but through the socket it started a **second** container with the host's `/` mounted. Same result as topic 04: host root.

### Compare normal vs privileged

```bash
docker run --rm alpine sh -c 'grep CapEff /proc/self/status; ls /dev | wc -l; grep Seccomp: /proc/self/status'
docker run --rm --privileged alpine sh -c 'grep CapEff /proc/self/status; ls /dev | wc -l; grep Seccomp: /proc/self/status'
```

```
CapEff: 00000000a80425fb
15
Seccomp:        2
CapEff: 000001ffffffffff
212
Seccomp:        0
```

- Privileged: **all** capability bits set, the host's **devices** visible (212 entries), and **no seccomp** (`0`).

```bash
docker run --rm --privileged alpine sh -c 'mkdir /h && mount /dev/sda1 /h 2>/dev/null || mount $(ls /dev/sd?1 /dev/vd?1 /dev/nvme?n1p1 2>/dev/null | head -1) /h; ls /h | head'
```

- On many hosts this mounts the **host's root disk** inside the container. (In WSL2 the disk is `/dev/sdc` or similar; the device names vary.)

### Find risky containers on a host

```bash
docker ps -q | xargs docker inspect --format \
'{{.Name}} privileged={{.HostConfig.Privileged}} user={{.Config.User}} pid={{.HostConfig.PidMode}} net={{.HostConfig.NetworkMode}} caps={{.HostConfig.CapAdd}} mounts={{range .Mounts}}{{.Source}}:{{.Destination}} {{end}}'
```

```
/traefik privileged=false user= pid= net=shop_default caps=[] mounts=/var/run/docker.sock:/var/run/docker.sock
/agent privileged=true user= pid=host net=host caps=[] mounts=/:/host
/web privileged=false user=10001 pid= net=shop_default caps=[] mounts=
```

- Read it as a reviewer: `traefik` has the socket (host root via API), `agent` is privileged with host PID, host network and `/` mounted (host root), `web` looks good.

### Check the daemon side

```bash
sudo ss -lntp | grep dockerd          # any TCP listener? (should be empty)
getent group docker                   # who is effectively root?
docker info --format '{{.SecurityOptions}}'
```

```
docker:x:988:tushar,ci-runner
[name=apparmor name=seccomp,profile=builtin name=cgroupns]
```

### Run docker-bench-security

```bash
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sudo sh docker-bench-security.sh -c container_runtime
```

Trimmed output:

```
# --------------------------------------------------------------------------------------------
# Docker Bench for Security
# Checks for dozens of common best-practices around deploying Docker containers in production.
# Based on the CIS Docker Benchmark.
# --------------------------------------------------------------------------------------------

[INFO] 5 - Container Runtime
[PASS] 5.1  - Ensure that, if applicable, an AppArmor Profile is enabled
[WARN] 5.4  - Ensure that privileged containers are not used
[WARN]      * Container running in Privileged mode: agent
[WARN] 5.12 - Ensure that the container's root filesystem is mounted as read only
[WARN]      * Container running with root FS mounted R/W: web
[WARN] 5.26 - Ensure that container health is checked at runtime
[PASS] 5.32 - Ensure that the Docker socket is not mounted inside any containers
...
[INFO] Checks: 31
[INFO] Score: 4
```

- Each `[WARN]` line is a CIS recommendation with the offending containers listed. `-c` limits it to a section. Run without `-c` for the full host audit.
- Some warnings are accepted risks (document why). The point is to make decisions explicitly.

---

## Hands-on lab (on a disposable VM, not a shared machine)

1. Run the socket and privileged demos above, and understand exactly why each works.
2. Run the "risky containers" one-liner on your lab host after starting a few containers with different flags.
3. Run docker-bench-security fully. Pick three `[WARN]` items and fix them (e.g. enable `no-new-privileges` and `live-restore` in `daemon.json`, run a container read-only, add a health check). Re-run and see the score change.
4. **Optional, gVisor:** install `runsc` (from gvisor.dev), add it to `daemon.json` (`"runtimes": {"runsc": {"path": "/usr/local/bin/runsc"}}`), restart Docker, and compare:
   ```bash
   docker run --rm alpine dmesg | head -3
   docker run --rm --runtime runsc alpine dmesg | head -3
   ```
   With gVisor, `dmesg` shows gVisor's own fake kernel messages: the container is talking to gVisor, not the host kernel.

---

## Where you'll use it

- **Security reviews and audits:** CIS benchmark evidence, container configuration reviews, socket and privileged-mode hunts.
- **Platform design:** deciding which workloads can share hosts, when to use sandboxed runtimes, CI runner isolation (topic 46).
- **Incident response:** "a container was compromised; could the attacker reach the host?" Check user, capabilities, mounts, socket, privileged mode, kernel version.
- **Interviews:** "Are containers secure?", "Why is mounting the Docker socket dangerous?", "What does --privileged do?"
- **Kubernetes:** Pod Security Standards (`privileged`, `baseline`, `restricted`) encode the same ideas (topic 53).

---

## When it breaks

| Finding | Risk | Fix |
|---|---|---|
| Container mounts `/var/run/docker.sock` | Host root via API | Remove; socket proxy (read-only endpoints); rootless; redesign |
| `privileged=true` | Host root | Specific `--cap-add`/`--device`; isolate on a dedicated host if unavoidable |
| Host paths like `/`, `/etc`, `/root` mounted RW | Host takeover | Mount only what's needed, read-only |
| dockerd listening on `0.0.0.0:2375` | Remote root, actively scanned | Disable TCP; SSH contexts or mTLS (mutual TLS: the client must also present a certificate; topic 39a) |
| Many users in `docker` group | All of them are root | Remove; sudo rules; rootless Docker |
| Containers run as root | Escalation if any other weakness exists | Non-root USER; user namespaces |
| Old kernel/runc with known escape CVEs | Container escape | Patch; subscribe to Docker/runc security advisories |
| No limits | DoS of host and neighbours | Memory/CPU/PID limits |
| Unknown community images | Malware/miners | Official/verified images, your own registry, scanning, signing |

---

This topic is about **prevention**. Detection and response (Falco, auditing, the incident procedure) are in topic 44a.

## Quick recap

- Containers are isolated by namespaces, cgroups, capabilities, seccomp and LSMs, but **all share one kernel**: weaker than VMs.
- The biggest real risks are configuration: **Docker socket = root**, **`--privileged` = root**, **`docker` group = root**, **exposed 2375 = remote root**, dangerous host mounts.
- Defence in depth: patched host, non-root, drop capabilities, `no-new-privileges`, seccomp/LSM on, read-only root, limits, minimal scanned images, no secrets in images.
- Untrusted workloads need stronger isolation: gVisor, Kata, microVMs.
- Audit against the **CIS Docker Benchmark** with **docker-bench-security**.

**Interview questions**

1. **Q: Are containers as secure as virtual machines?**
   A: No. Containers share the host kernel, so a kernel vulnerability or a misconfiguration can lead to a host compromise. VMs isolate at the hypervisor level. Containers can be hardened a lot, and sandboxed runtimes (gVisor, Kata) narrow the gap.

2. **Q: Why is mounting `/var/run/docker.sock` into a container dangerous, even read-only?**
   A: The socket gives full access to the Docker API, and dockerd runs as root. The container can start a privileged container that mounts the host filesystem. The `:ro` flag only affects the socket file, not what the API allows.

3. **Q: What does `--privileged` do?**
   A: It gives the container all capabilities, disables seccomp and AppArmor/SELinux confinement, and exposes all host devices. That's effectively root on the host, so avoid it and grant only specific capabilities or devices.
