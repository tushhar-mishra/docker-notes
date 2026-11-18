# 42 · User Namespaces and Rootless Docker

> **Level:** ★★★ advanced
>
> **You will learn:** how the Linux **user namespace** makes "root inside the container" a **normal, unprivileged user on the host**, two ways Docker uses it: **`userns-remap`** (the daemon is still root, containers are remapped) and **rootless mode** (the daemon itself runs as a normal user), how `/etc/subuid` and `/etc/subgid` work, what changes for file ownership, ports, networking, storage and cgroup limits, and how this compares with Podman.
>
> **You need:** topic 02 (user namespace), topics 11, 25, 40, 41.

---

## What it is

A **user namespace** gives a process its own **mapping of UIDs/GIDs**:

```
inside the container        on the host
UID 0     (root)     ──►    UID 100000   (an unprivileged nobody)
UID 1     (daemon)   ──►    UID 100001
...
UID 65535            ──►    UID 165535
```

The process is "root" for everything **inside its namespaces** (it can install packages, chown files in its container, bind ports in its own network namespace), but to the host kernel, outside those namespaces, it's UID 100000, **which owns nothing and can do nothing** on the host.

Docker offers two ways to use this:

| | **userns-remap** | **Rootless mode** |
|---|---|---|
| dockerd runs as | **root** | **a normal user** |
| Containers' root maps to | A subordinate UID range of a helper user (`dockremap`) | A subordinate UID range of **your** user |
| Protects against | Container escape → host root | Container escape **and** dockerd/runtime bugs → host root |
| Setup | One `daemon.json` line | A per-user install |
| Compatibility | Most features; some per-container opt-outs | More limitations (networking, ports, some storage/cgroup features) |

---

## Why it exists

Topic 40: by default, container root **is** host root (UID 0), only fenced by capabilities/seccomp/LSM. If anything goes wrong (a host mount, a kernel bug, a runtime bug), the attacker is root on the host.

With a user namespace, an escape lands the attacker as **UID 100000**: an account with no files, no sudo, nothing. It's one of the strongest single hardening steps available. Rootless mode goes further: even the **Docker daemon** isn't root, so the "Docker socket = root" problem (topic 40) shrinks to "Docker socket = that user".

---

## Analogy

A **film set**. The actor playing the king has a crown and gives orders **on set** (root inside the container): the extras obey, the props move. But once they walk off the set into the real city (the host), they're an ordinary person with no authority. Without user namespaces, the "actor" is the real king: walking off set, they still rule the city.

---

## How it works inside

### `/etc/subuid` and `/etc/subgid`

These files give users **ranges of extra UIDs/GIDs** they're allowed to map in user namespaces:

```
# /etc/subuid      name:start:count
dockremap:100000:65536
tushar:165536:65536
```

- `dockremap` may use host UIDs 100000–165535. `tushar` may use 165536–231071.
- The kernel only allows mapping ranges listed here (enforced by the setuid helpers `newuidmap`/`newgidmap` from the `uidmap` package).
- Ranges must not overlap between users.

### The mapping a process sees

```bash
cat /proc/<pid>/uid_map
```

```
         0     100000      65536
```

- Read as: "inside UID **0** = outside UID **100000**, for **65536** IDs". So inside UID 33 (www-data) = host UID 100033.

### userns-remap

```json
{ "userns-remap": "default" }
```

- `default` = Docker creates the user `dockremap` and adds subuid/subgid ranges for it (or use `"userns-remap": "tushar"` for an existing user with ranges).
- dockerd still runs as root, but starts every container in a **new user namespace** with that mapping.
- **Storage moves**: dockerd uses a subfolder `/var/lib/docker/100000.100000/`, with all images unpacked again with shifted ownership. Your existing images and containers "disappear" (they're still in the old location) until you turn it off.
- **Not compatible with the containerd image store** (topic 07). That store isn't available when userns-remap is enabled.
- Per-container opt-out: `docker run --userns=host` (needed for `--privileged`, `--pid host`, `--network host` in some cases, and some tools).

### Rootless mode

dockerd, containerd and runc all run **as your user**, inside a user namespace created by **RootlessKit** (a helper that sets up the namespace and networking for the daemon).

```
your user (UID 1000) ──► rootlesskit ──► user namespace (you = "root" inside, mapped via /etc/subuid)
                                         └── dockerd ── containerd ── runc ── containers
```

What changes:

| Area | Rootless behaviour |
|---|---|
| **Socket** | `$XDG_RUNTIME_DIR/docker.sock` (e.g. `/run/user/1000/docker.sock`), owned by you. Clients set `DOCKER_HOST=unix:///run/user/1000/docker.sock` or use the `rootless` context |
| **Data** | `~/.local/share/docker` |
| **Service** | A **systemd user service**: `systemctl --user start docker`. For it to run without you logged in: `sudo loginctl enable-linger <user>` |
| **Networking** | The daemon can't create real veth/bridges on the host or iptables rules in the host namespace. Outbound traffic goes through a **user-space network stack** (historically slirp4netns; also pasta; newer Engine releases default to gvisor-tap-vsock). It's slower than kernel bridging |
| **Published ports** | Forwarded by RootlessKit's port driver. Ports **below 1024** need the host sysctl `net.ipv4.ip_unprivileged_port_start` lowered (or a capability on rootlesskit). Some port drivers don't preserve the client's source IP |
| **Storage** | overlayfs inside a user namespace (kernel 5.11+), else `fuse-overlayfs` |
| **Resource limits** | Need **cgroup v2** with systemd **delegation** of controllers to user sessions (cpu, memory, pids, io). Otherwise `--memory`/`--cpus` are ignored or fail |
| **Not supported / limited** | `--privileged` (only within your user's powers), AppArmor in some setups, host networking semantics, some network drivers (macvlan, overlay) |
| **Security** | Container escape → your user (not root). dockerd exploit → your user. The Docker socket gives control of your user's containers only |

### Podman comparison

**Podman** (topic 03) is **rootless by default** and **daemonless**: each `podman run` sets up the user namespace itself (using `/etc/subuid` the same way), with `conmon` as the per-container parent and `pasta`/`slirp4netns` for networking. It also supports **Quadlet** (systemd unit files generated from container definitions) for running containers as services. Same kernel features, different packaging. Many RHEL shops use rootless Podman for exactly the reasons in this topic.

### Kubernetes note

Kubernetes added user-namespace support for Pods (`hostUsers: false` in the Pod spec), so the same protection is available there.

---

## Commands / files

### Turn on userns-remap

```bash
sudo apt-get install -y uidmap
echo '{ "userns-remap": "default" }' | sudo tee /etc/docker/daemon.json   # merge with your existing settings instead of overwriting!
sudo systemctl restart docker
grep dockremap /etc/subuid /etc/subgid
docker info --format '{{.DockerRootDir}} {{.SecurityOptions}}'
```

```
/etc/subuid:dockremap:100000:65536
/etc/subgid:dockremap:100000:65536
/var/lib/docker/100000.100000 [name=apparmor name=seccomp,profile=builtin name=userns name=cgroupns]
```

- `name=userns` in security options. Root dir moved to `100000.100000`.

### See the mapping from both sides

```bash
docker run -d --name un alpine sleep 3600
docker exec un id
PID=$(docker inspect -f '{{.State.Pid}}' un)
ps -o pid,uid,user,cmd -p $PID
cat /proc/$PID/uid_map
sudo ls -l /proc/1/ns/user /proc/$PID/ns/user
```

```
uid=0(root) gid=0(root) groups=0(root),...
    PID   UID USER     CMD
  95432 100000 100000  sleep 3600
         0     100000      65536
lrwxrwxrwx 1 root root 0 ... /proc/1/ns/user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 ... /proc/95432/ns/user -> user:[4026532611]
```

- Inside: root. On the host: UID **100000**.
- The **user** namespace is now different from the host's (compare topic 11, where it was shared).

### File ownership with bind mounts

```bash
mkdir -p ~/unmount && docker run --rm -v ~/unmount:/data alpine sh -c 'touch /data/x' ; echo "exit=$?"
sudo chown 100000:100000 ~/unmount && docker run --rm -v ~/unmount:/data alpine sh -c 'touch /data/x && echo ok'
ls -ln ~/unmount
```

```
touch: /data/x: Permission denied
exit=1
ok
-rw-r--r-- 1 100000 100000 0 Nov 17 16:00 x
```

- Container root (= host 100000) couldn't write to a folder owned by your UID 1000. After chowning it to 100000, it can. Files appear on the host as owned by 100000.

### Opt a container out

```bash
docker run --rm --userns=host alpine id
docker run --rm --privileged alpine true
```

```
uid=0(root) gid=0(root) ...
docker: Error response from daemon: privileged mode is incompatible with user namespaces.  You must run the container in the host namespace when running privileged mode.
```

(Turn userns-remap off again for the rest of the notes, unless you want to keep it: remove the line, restart Docker.)

### Install rootless Docker (Ubuntu)

```bash
sudo apt-get install -y uidmap docker-ce-rootless-extras
grep "^$(whoami):" /etc/subuid /etc/subgid
dockerd-rootless-setuptool.sh install
```

```
/etc/subuid:tushar:165536:65536
/etc/subgid:tushar:165536:65536
[INFO] Creating /home/tushar/.config/systemd/user/docker.service
...
[INFO] Installed docker.service successfully.
[INFO] To control docker.service, run: `systemctl --user (start|stop|restart) docker.service`
[INFO] To run docker.service on system startup, run: `sudo loginctl enable-linger tushar`

[INFO] Creating CLI context "rootless"
Successfully created context "rootless"
[INFO] Using CLI context "rootless"
Current context is now "rootless"

[INFO] Make sure the following environment variable(s) are set (or add them to ~/.bashrc):
export PATH=/usr/bin:$PATH
```

- If the system-wide (rootful) Docker is also running, both can coexist: switch with `docker context use default` / `docker context use rootless`.

```bash
docker context use rootless
docker info --format '{{.SecurityOptions}} {{.DockerRootDir}}'
docker run -d --name rl -p 8088:80 nginx:1.27-alpine
ps -o pid,user,cmd -C dockerd
PID=$(docker inspect -f '{{.State.Pid}}' rl); ps -o pid,user,cmd -p $PID
```

```
[name=seccomp,profile=builtin name=rootless name=cgroupns] /home/tushar/.local/share/docker
    PID USER     CMD
  11223 tushar   dockerd
    PID USER     CMD
  11400 165536   nginx: master process nginx -g daemon off;
```

- `name=rootless`. **dockerd runs as `tushar`**, and the container's "root" nginx is host UID **165536**.

Privileged ports in rootless mode:

```bash
docker run --rm -p 80:80 nginx:1.27-alpine
```

```
docker: Error response from daemon: driver failed programming external connectivity on endpoint ...: error while calling RootlessKit PortManager.AddPort(): cannot expose privileged port 80, you can add 'net.ipv4.ip_unprivileged_port_start=80' to /etc/sysctl.conf (currently 1024), or set CAP_NET_BIND_SERVICE on rootlesskit binary, or choose a larger port number (>= 1024)
```

- The error tells you the three fixes.

Resource limits need cgroup delegation:

```bash
cat /sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers
```

```
cpu memory pids
```

- If `cpu`/`memory`/`io` are missing, add a systemd drop-in for `user@.service` with `Delegate=cpu cpuset io memory pids`, then reboot.

---

## Hands-on lab

1. Enable `userns-remap`, run the mapping demos, and prove an "escape" lands nowhere: `docker run --rm -v /etc:/hostetc alpine sh -c 'echo hacked >> /hostetc/hostname'` → `Permission denied` (container root = 100000, which can't write host `/etc`). Compare with the same command without userns-remap on a **throwaway VM**.
2. Turn userns-remap off again. Note how images "come back" (the original data-root).
3. Install rootless Docker. Run nginx on port 8088, check UIDs with `ps`, and try port 80 to see the error.
4. Check cgroup delegation and test `docker run --rm --memory 64m alpine cat /sys/fs/cgroup/memory.max`.
5. Switch contexts back: `docker context use default`.
6. Clean up: `docker rm -f un rl`

---

## Where you'll use it

- **Shared hosts and CI runners:** rootless Docker (or rootless Podman/BuildKit) so a job can't become host root (topic 46).
- **Hardening sensitive servers:** userns-remap as a defence-in-depth layer when compatible with your workloads.
- **Developer machines** where people shouldn't have root.
- **Kubernetes:** user namespaces for Pods (`hostUsers: false`), and rootless BuildKit for in-cluster builds.
- **Interviews:** "How do you make root in a container not be root on the host?"

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| After enabling userns-remap, all images/containers are gone | New data-root subfolder | Expected. Re-pull, or disable to get them back |
| `Permission denied` on bind mounts with userns-remap/rootless | Host files owned by your UID, container root maps to 100000+ | `chown` to the mapped UID, or use volumes |
| `privileged mode is incompatible with user namespaces` | userns-remap is on | `--userns=host` for that container (understand the risk) |
| `newuidmap: write to uid_map failed: Operation not permitted` | No `/etc/subuid` entry, or `uidmap` package missing | Add ranges; install `uidmap` |
| Rootless: `cannot expose privileged port 80` | Ports < 1024 need extra config | Lower `ip_unprivileged_port_start`, setcap rootlesskit, or use ≥1024 |
| Rootless: `--memory` ignored / `cgroup ... not supported` | Controllers not delegated to the user | systemd `Delegate=` drop-in, cgroup v2 |
| Rootless containers stop when you log out | User systemd instance stops | `sudo loginctl enable-linger <user>` |
| Rootless network slower, client IPs wrong | User-space network/port drivers | Accept the trade-off, or pick a port driver that keeps source IPs; rootful for network-heavy workloads |
| Wrong daemon answering | Two daemons (rootful and rootless) | `docker context ls`; `echo $DOCKER_HOST` |

---

## Quick recap

- A user namespace maps container UIDs to an **unprivileged host range** from `/etc/subuid`/`/etc/subgid`. Root inside = e.g. UID 100000 outside.
- **userns-remap**: the daemon is root, containers are remapped. One setting. Separate data-root. Incompatible with `--privileged` (opt out with `--userns=host`) and the containerd image store.
- **Rootless mode**: the whole engine runs as a normal user (RootlessKit, user-space networking, `~/.local/share/docker`, systemd user service). The strongest protection, with limits on ports < 1024, networking and cgroup delegation.
- Bind-mount ownership must match the **mapped** UIDs.
- Podman is rootless and daemonless by default, with the same kernel mechanism. Kubernetes supports Pod user namespaces.

**Interview questions**

1. **Q: How can you make root inside a container not be root on the host?**
   A: Use user namespaces: either Docker's `userns-remap`, which maps container UIDs to a subordinate range like 100000–165535, or rootless Docker/Podman, where the whole engine runs as an unprivileged user. An escape then lands as an unprivileged host UID.

2. **Q: What's the difference between userns-remap and rootless Docker?**
   A: With userns-remap the daemon still runs as root and only containers are remapped. With rootless mode the daemon, containerd and runc themselves run as a normal user, so a daemon or runtime compromise also doesn't give host root. Rootless has more functional limits (privileged ports, networking performance, cgroup delegation).

3. **Q: After enabling userns-remap, a container can't write to a bind-mounted folder. Why?**
   A: Container root maps to an unprivileged host UID (e.g. 100000), which doesn't own the host folder. Change ownership to the mapped UID, or use a Docker volume.
