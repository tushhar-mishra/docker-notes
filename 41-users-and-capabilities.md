# 41 · Users, Capabilities and no-new-privileges

> **Level:** ★★★ advanced. Read slowly. Do the lab.
>
> **You will learn:** the first half of how Docker limits what a container's process may do: the **user** it runs as, **Linux capabilities** (root's power split into pieces; Docker's default 14; `--cap-drop`/`--cap-add`; the five capability sets), **`no-new-privileges`**, and safer alternatives to `--privileged` for devices and kernel settings (`--device`, `--sysctl`). Topic 41a covers the second half: seccomp, AppArmor/SELinux and masked paths.
>
> **You need:** topic 11 (`/proc/PID/status`, `capsh`), topic 40 (security model), topic 25 (UIDs on mounts).

---

## What it is

Even when a container runs as root, the kernel restricts it through several independent mechanisms. This topic covers the first three:

| Mechanism | Question it answers | Docker's default |
|---|---|---|
| **User (UID/GID)** | Who is this process? | Root, unless the image/`-u` says otherwise |
| **Capabilities** | Which **root powers** does it have? | 14 of ~41 |
| **no-new-privileges** | Can it **gain** more privileges later (setuid programs)? | Off |

Topic 41a covers the other two: **seccomp** (which system calls are allowed) and **LSMs** (AppArmor/SELinux policies).

---

## Why it exists

Classic Unix has two levels: **root** (can do everything) and **everyone else**. That's too coarse for containers. A web server may need to bind port 80 (a root power), but it should never be able to load kernel modules, change the clock or reconfigure the host's network. These mechanisms give **least privilege**: each process gets only the powers it needs. If the app is compromised, the attacker gets only those powers.

---

## Analogy

A **hotel staff badge**:
- **User** = the name printed on the badge.
- **Capabilities** = which doors the badge opens. The general manager's badge (root) normally opens every door. Docker gives container root a badge that opens only 14 of them.
- **no-new-privileges** = "during your shift, you can't swap your badge for a better one", even if you find the manager's badge lying in a drawer (a setuid program).

---

## How it works inside

### 1. Users (recap)

- The kernel checks permissions by **UID/GID numbers** (topic 25).
- Without user namespaces, container UID 0 = host UID 0 (topics 11, 42).
- A non-root UID gets **no effective capabilities** by default. So **running as non-root removes most of the risk at once**, and most of what follows matters mainly for root-in-container.

### 2. Capabilities

Since Linux 2.2, root's power has been split into about 41 **capabilities**. For each privileged action, the kernel checks for one specific capability instead of "is this root?". Examples: `CAP_NET_BIND_SERVICE` to bind ports below 1024, `CAP_SYS_ADMIN` for mount and many admin operations, `CAP_NET_ADMIN` to change network config, `CAP_SYS_PTRACE` to trace other processes.

Each process has five **capability sets**:

| Set | Meaning |
|---|---|
| **Effective** | What the kernel checks **right now** |
| **Permitted** | What the process **may** make effective |
| **Bounding** | The hard ceiling. Can never be exceeded, even by running a setuid binary or a file with capabilities |
| **Inheritable** | What may pass across `execve()` (only together with file capabilities) |
| **Ambient** | What passes across `execve()` to normal (non-setuid) programs |

`--cap-drop` removes a capability from the bounding set (and the others). After that, nothing inside the container can ever get it back.

**Docker's default 14** (the effective/bounding set of container root):

| Capability | Allows | Why Docker keeps it |
|---|---|---|
| `CHOWN` | Change file owners | Package installs, entrypoints fixing permissions |
| `DAC_OVERRIDE` | Bypass file read/write/execute permission checks | Root expects to read any file |
| `FOWNER` | Bypass "must be owner" checks (chmod on others' files) | Same |
| `FSETID` | Keep setuid/setgid bits when changing files | Package installs |
| `KILL` | Send signals to any process (in its namespace) | Process management |
| `SETGID`, `SETUID` | Change group/user ID | Servers dropping privileges (nginx master → workers) |
| `SETPCAP` | Change capability sets | Dropping capabilities |
| `NET_BIND_SERVICE` | Bind ports < 1024 | Web servers on 80/443 |
| `NET_RAW` | Raw and packet sockets: packet capture (tcpdump), crafting packets | Historically needed for `ping`. **Often dropped for security**, because it enables spoofing tricks (ARP, DNS) inside the network. Note: modern `ping` no longer needs it in Docker (see below) |
| `SYS_CHROOT` | `chroot()` | Some daemons |
| `MKNOD` | Create device files | Package installs. The device **cgroup** still blocks *using* them |
| `AUDIT_WRITE` | Write to the kernel audit log | Login tools |
| `SETFCAP` | Set file capabilities | Package installs |

**Not** included (examples): `SYS_ADMIN`, `NET_ADMIN`, `SYS_MODULE`, `SYS_PTRACE`, `SYS_TIME`, `SYS_BOOT`, `SYS_RAWIO`, `DAC_READ_SEARCH`, `BPF`, `PERFMON`, `SYS_RESOURCE`, `IPC_LOCK`, `LINUX_IMMUTABLE`.

**Why `ping` works without `NET_RAW` today:** since Docker 20.10, every container gets the sysctl `net.ipv4.ping_group_range = 0 2147483647`. That allows **unprivileged ICMP "datagram" sockets**, and modern `ping` (BusyBox and iputils) uses them when raw sockets aren't allowed. So dropping `NET_RAW` breaks packet capture and raw packet crafting, but usually **not** ping. That's a good thing: you can drop `NET_RAW` and keep basic network checks working.

**Best practice:** `--cap-drop ALL`, then `--cap-add` only what's needed. Many apps need **none**.

| App needs to… | Add |
|---|---|
| Bind port 80 as root (when the unprivileged-port sysctl isn't available, topic 20) | `NET_BIND_SERVICE` |
| Change network settings (VPN, routing, iptables inside) | `NET_ADMIN` |
| Capture packets (tcpdump) | `NET_RAW` (and often `NET_ADMIN` for promiscuous mode) |
| Debug with strace/gdb | `SYS_PTRACE` |
| Change the system clock (NTP daemon) | `SYS_TIME` |
| Lock memory (some DBs, Elasticsearch `mlockall`) | `IPC_LOCK` |
| Entrypoint that `chown`s and then drops to a user (`gosu`/`su-exec`) | `CHOWN`, `SETUID`, `SETGID` (maybe `DAC_OVERRIDE`, `FOWNER`) |

**Non-root + capabilities:** if the container runs as UID 10001, added capabilities sit in the bounding set but **aren't effective** for a normal binary. To give a non-root program a capability, set a **file capability** on the binary in the image (`setcap cap_net_bind_service=+ep /usr/bin/myapp`) **and** keep that capability in the bounding set (don't drop it).

### 3. `no-new-privileges`

```bash
docker run --security-opt no-new-privileges ...
```

- Sets the kernel's `PR_SET_NO_NEW_PRIVS` flag on the process. From then on, **`execve()` can never grant more privileges**: setuid-root binaries (`sudo`, `su`, `passwd`) run **without** gaining root, and file capabilities are ignored.
- It blocks a whole class of privilege escalation inside the container, and the process can't undo it.
- Set it for all containers with `"no-new-privileges": true` in `daemon.json` (topic 34). It conflicts with the file-capability trick above, so you choose one or the other.
- Kubernetes equivalent: `allowPrivilegeEscalation: false`.

### 4. Devices and sysctls: precise alternatives to `--privileged`

- `--device /dev/ttyUSB0` = give access to **one** host device (the device cgroup allows exactly that one). Much safer than `--privileged`, which exposes **all** host devices (topic 40, 41a).
- `--sysctl net.core.somaxconn=1024` = set a **namespaced** kernel parameter for this container only (mostly `net.*` and some `kernel.*` keys). Host-wide settings (like `vm.max_map_count`) can't be set from a container. Set them on the host.

---

## Commands / files

### See the capabilities

```bash
docker run --rm alpine sh -c 'grep Cap /proc/self/status'
docker run --rm --cap-drop ALL alpine sh -c 'grep Cap /proc/self/status'
docker run --rm --cap-drop ALL --cap-add NET_BIND_SERVICE alpine sh -c 'grep CapEff /proc/self/status'
capsh --decode=0000000000000400
```

```
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000000000000000
CapAmb: 0000000000000000
CapEff: 0000000000000400
0x0000000000000400=cap_net_bind_service
```

- `capsh` comes from the `libcap2-bin` package (topic 11).

### Watch capabilities in action

```bash
docker run --rm alpine sh -c 'chown nobody /tmp && echo "chown OK"'
docker run --rm --cap-drop CHOWN alpine sh -c 'chown nobody /tmp && echo "chown OK"'
docker run --rm --cap-drop NET_RAW nicolaka/netshoot tcpdump -c1 -ni eth0
docker run --rm --cap-drop NET_RAW alpine sh -c 'cat /proc/sys/net/ipv4/ping_group_range; ping -c1 -W1 127.0.0.1 >/dev/null && echo "ping still OK"'
```

```
chown OK
chown: /tmp: Operation not permitted
tcpdump: eth0: You don't have permission to perform this capture on that device
(socket: Operation not permitted)
0	2147483647
ping still OK
```

- Dropping `CHOWN` blocks `chown`.
- Dropping `NET_RAW` blocks packet capture (it needs a raw/packet socket)…
- …but `ping` keeps working through unprivileged ICMP sockets, thanks to `ping_group_range`.
- (Without `--cap-drop NET_RAW`, the same tcpdump command starts capturing instead of erroring.)

`NET_ADMIN` example:

```bash
docker run --rm alpine ip link add dummy0 type dummy
docker run --rm --cap-add NET_ADMIN alpine sh -c 'ip link add dummy0 type dummy && echo "NET_ADMIN OK"'
```

```
ip: RTNETLINK answers: Operation not permitted
NET_ADMIN OK
```

- If the second command says `Unknown device type` instead, the host doesn't have the `dummy` kernel module loaded. A container **can't load kernel modules** (no `SYS_MODULE`, and seccomp blocks it too, topic 41a). Run `sudo modprobe dummy` on the host, or use another test such as `ip link set lo mtu 1400`.

### Minimal capabilities for a real app

```bash
docker run -d --name hardened \
  --cap-drop ALL --cap-add NET_BIND_SERVICE --cap-add CHOWN --cap-add SETUID --cap-add SETGID \
  --security-opt no-new-privileges \
  --read-only --tmpfs /var/cache/nginx --tmpfs /var/run --tmpfs /tmp \
  -p 8082:80 nginx:1.27-alpine
curl -sI localhost:8082 | head -1
```

```
HTTP/1.1 200 OK
```

- nginx's master process (root) needs to bind port 80, `chown` its cache folders and switch its workers to the `nginx` user. Nothing else.

### no-new-privileges blocks setuid escalation

We make a **setuid-root copy of python** and run it as a normal user. This is exactly what an attacker's privilege-escalation trick looks like:

```bash
SETUP='cp $(readlink -f $(which python3)) /tmp/py && chmod u+s /tmp/py && useradd -m u'
TEST='su u -c "/tmp/py -c \"import os; print(\\\"effective UID:\\\", os.geteuid())\""'
docker run --rm python:3.13-slim sh -c "$SETUP && $TEST"
docker run --rm --security-opt no-new-privileges python:3.13-slim sh -c "$SETUP && $TEST"
```

```
effective UID: 0
effective UID: 1000
```

- Without the flag, user `u` running the setuid binary **became root** (effective UID 0).
- With `no-new-privileges`, the kernel ignored the setuid bit, and it stayed UID 1000.

### Devices and sysctls

```bash
docker run --rm --device /dev/null:/dev/mynull alpine sh -c 'echo test > /dev/mynull && echo "device OK"'
docker run --rm --sysctl net.core.somaxconn=1024 alpine cat /proc/sys/net/core/somaxconn
docker run --rm --sysctl vm.max_map_count=262144 alpine true
```

```
device OK
1024
docker: Error response from daemon: ... sysctl "vm.max_map_count" is not in a separate kernel namespace
```

- `vm.*` settings are host-wide. Set them on the host (`sysctl -w vm.max_map_count=262144`, made permanent in `/etc/sysctl.d/`).

### Compose equivalents

```yaml
services:
  web:
    user: "10001:10001"
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    security_opt:
      - no-new-privileges:true
    devices: ["/dev/ttyUSB0:/dev/ttyUSB0"]
    sysctls:
      net.core.somaxconn: 1024
```

---

## Hands-on lab

1. Decode the default capability mask (`capsh --decode=00000000a80425fb`) and match each capability to the table.
2. Run the capability demos (`chown`, tcpdump, ping, `ip link add`) with and without drops/adds. Explain why ping survives `--cap-drop NET_RAW`.
3. Harden nginx with `--cap-drop ALL` and find the **minimum** set it needs: start with none, read the errors in `docker logs`, and add one capability at a time.
4. Run the setuid demo with and without `no-new-privileges`.
5. Clean up: `docker rm -f hardened`

---

## Where you'll use it

- **Hardening standards:** `cap_drop: [ALL]` and `no-new-privileges` are standard production settings.
- **Kubernetes:** `securityContext.capabilities.drop: ["ALL"]`, `allowPrivilegeEscalation: false`, `runAsNonRoot`. The "restricted" Pod Security Standard requires them.
- **Troubleshooting "Operation not permitted":** the first question is "which capability does this action need?"
- **Security reviews:** `cap_add: SYS_ADMIN` or `NET_ADMIN` on a normal web app is a red flag.

---

## When it breaks

| Error | Usual cause | How to confirm / fix |
|---|---|---|
| `Operation not permitted` on `chown`/`chmod`/`kill`/`setuid` | A dropped capability | Check `CapEff`; add the specific capability |
| `tcpdump: ... You don't have permission to perform this capture` | `NET_RAW` dropped | Add it only for debug containers (topic 51) |
| `su`/`sudo` fails in the container | `no-new-privileges` | Expected. Don't rely on setuid in containers |
| App can't bind port 80 as non-root, even with `--cap-add NET_BIND_SERVICE` | The capability isn't effective for non-root without a file capability | `setcap` on the binary, the unprivileged-port sysctl (topic 20), or port 8080 |
| `ip link add ... Unknown device type` | Host kernel module not loaded; containers can't load modules | `modprobe` on the host |
| `sysctl ... is not in a separate kernel namespace` | Host-wide sysctl | Set it on the host |
| Something "only works with `--privileged`" | Usually one capability or one device | Find exactly which (errors, `strace`, topic 51) and grant only that |

---

## Quick recap

- Run **non-root**: non-root processes have no effective capabilities, which removes most of the risk.
- **Capabilities** split root's power. Docker keeps 14. Use `--cap-drop ALL` plus only what's needed. `NET_RAW` is no longer needed for ping in Docker (`ping_group_range`).
- **`no-new-privileges`**: setuid binaries and file capabilities can't raise privileges. Turn it on everywhere you can.
- Use `--device` and namespaced `--sysctl` instead of `--privileged`.
- Next: topic 41a (seccomp, AppArmor/SELinux, masked paths, what `--privileged` disables).

**Interview questions**

1. **Q: What are Linux capabilities, and how does Docker use them?**
   A: Root's privileges split into ~41 units (e.g. `NET_BIND_SERVICE`, `SYS_ADMIN`). Docker grants containers 14 by default. You can drop all and add back only what's required with `--cap-drop ALL --cap-add ...`. A dropped capability is removed from the bounding set, so nothing in the container can regain it.

2. **Q: What does `no-new-privileges` do?**
   A: It sets the kernel's no_new_privs flag, so neither the process nor its children can gain privileges through `execve`. setuid binaries and file capabilities no longer elevate. In Kubernetes it's `allowPrivilegeEscalation: false`.

3. **Q: Why does `ping` still work after `--cap-drop NET_RAW`?**
   A: Docker sets `net.ipv4.ping_group_range` so any group may use unprivileged ICMP datagram sockets, and modern ping uses those. `NET_RAW` is still needed for raw and packet sockets, such as tcpdump or packet crafting.
