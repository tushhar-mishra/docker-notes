# 04 · Installing Docker and Setting Up Your Lab

> **Level:** ★ foundation
>
> **You will learn:** how to install Docker Engine the production way (from Docker's own package repository), how to build a proper Linux lab on your Windows laptop with WSL2, what Docker Desktop really is (a hidden VM), how to check an installation with `docker version` and `docker info`, and why the `docker` group equals root.
>
> **You need:** topic 03 (the CLI/daemon split, the Docker socket).

---

## What it is

Installing Docker on Linux means installing the pieces from topic 03 as packages:

| Package | What it installs |
|---|---|
| `docker-ce` | `dockerd`, the daemon (CE = Community Edition, the free open-source edition) |
| `docker-ce-cli` | The `docker` CLI |
| `containerd.io` | `containerd`, `containerd-shim-runc-v2` and `runc` |
| `docker-buildx-plugin` | `docker buildx`, the BuildKit client for advanced builds (topic 21) |
| `docker-compose-plugin` | `docker compose` (Compose v2, topic 35) |

> **Plugin** = a separate program the `docker` CLI calls for a subcommand. `docker compose ...` really runs `/usr/libexec/docker/cli-plugins/docker-compose`.

---

## Why it exists (why this way)

There are several ways to "get Docker". They're not equal:

| Way | Use it? |
|---|---|
| **Docker's official apt/dnf repository** | **Yes, for servers.** Latest stable versions, security updates through your normal package updates. |
| Your distribution's package (`docker.io` on Ubuntu, `moby-engine` on Fedora) | Works, but often older, and package names differ. Not what most production docs assume. |
| The convenience script (`curl https://get.docker.com \| sh`) | Only for quick tests. It runs a script as root from the internet, and you can't easily control what version you get. |
| **Docker Desktop** | Developer laptops (Windows/macOS). Check licensing for companies. Not for servers. |
| **Docker Engine inside WSL2 on Windows** | **Yes, for your learning lab.** A real Linux Docker host, where all the "look from the host" labs work. |

**Why not just Docker Desktop for learning?** Docker Desktop runs the Docker Engine inside a hidden VM you don't normally get a shell on. Labs like "find the container's process on the host" or "read its cgroup files" happen *inside that VM*, which is awkward. With Docker Engine installed directly in a WSL2 Ubuntu, **WSL2 Ubuntu is the host**. Everything in these notes works as it would on a production server.

---

## Analogy

Installing Docker Engine in WSL2 is like **renting a small workshop inside your apartment building**. Windows is the building. WSL2 is a real room with real Linux tools, where you can open up machines and look inside. Docker Desktop is more like **ordering from a sealed vending machine**: it works fine, but you can't see the mechanism.

---

## How it works inside

### WSL2: a real Linux kernel on Windows

**WSL2** (Windows Subsystem for Linux, version 2) runs a **real Linux kernel** (built by Microsoft) inside a lightweight VM, managed by Windows' Hyper-V technology. Each Linux distribution you install (Ubuntu, Debian…) runs on that kernel, in its own container-like environment.

```
Windows 11
 └─ WSL2 utility VM (real Linux kernel, e.g. 6.6.x-microsoft-standard-WSL2)
     └─ Ubuntu 24.04 distribution
         └─ systemd → containerd + dockerd → your containers
```

- Because WSL2 has a real Linux kernel with namespaces and cgroups, Docker runs natively there.
- WSL2 supports **systemd** (needed so `docker.service` starts like on a server). New Ubuntu installs have it on by default. You can check in `/etc/wsl.conf`.
- Ports you publish in WSL2 (e.g. `-p 8080:80`) are reachable from your Windows browser at `http://localhost:8080`, because WSL2 forwards localhost.

### Docker Desktop: a hidden VM

Docker Desktop installs:
- a VM (on Windows, a special WSL2 distribution called `docker-desktop`; on macOS, a lightweight VM using Apple's virtualization framework),
- Docker Engine inside that VM,
- a `docker` CLI on your Windows/macOS side, pointed at the Engine in the VM,
- port forwarding, file sharing and a GUI.

So **Docker is always Linux underneath** for Linux containers. On a Mac or Windows laptop, `uname -r` inside a container shows the VM's Linux kernel, not your laptop's OS.

> If you already have Docker Desktop with "WSL integration" turned on for your Ubuntu distro, it puts its own `docker` CLI into Ubuntu. That conflicts with a native Engine. **Turn off WSL integration for that distro** in Docker Desktop → Settings → Resources → WSL Integration, or quit Docker Desktop, before following the lab below.

### What the package install does

1. Adds Docker's **GPG key** (GPG = GNU Privacy Guard; a key used to check that packages really come from Docker and weren't changed).
2. Adds Docker's **repository** (the download location) to apt/dnf.
3. Installs the packages.
4. Creates the `docker` group and the systemd units `docker.service`, `docker.socket` and `containerd.service`.
5. Starts the services (on Ubuntu automatically; on RHEL you enable them).
6. `dockerd` creates the `docker0` bridge network interface and iptables rules (Part 5), and `/var/lib/docker`.

---

## Commands / files

### A. Your lab: Docker Engine in WSL2 Ubuntu (Windows)

**Step 1 (in Windows PowerShell as Administrator):** install WSL2 with Ubuntu.

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot if asked. Open "Ubuntu 24.04" from the Start menu and create your Linux username and password.

Check it's WSL **2** (not 1) and see the kernel:

```powershell
wsl -l -v
```

```
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2
```

- `VERSION 2` = WSL2 with a real kernel. WSL1 can't run Docker Engine.

**Step 2 (inside Ubuntu):** check systemd is running.

```bash
ps -p 1 -o comm=
cat /etc/wsl.conf
```

```
systemd
[boot]
systemd=true
```

- If PID 1 isn't `systemd`, add those two lines to `/etc/wsl.conf` (with `sudo nano /etc/wsl.conf`), then in PowerShell run `wsl --shutdown` and reopen Ubuntu.

**Step 3 (inside Ubuntu):** continue with section B. It's identical to a real Ubuntu server.

### B. Ubuntu / Debian server (and your WSL2 Ubuntu)

Remove anything that conflicts (old unofficial packages):

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

Add Docker's repository:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

What each part does:
- `install -m 0755 -d /etc/apt/keyrings` = create the folder for trusted keys.
- `curl ... -o docker.asc` = download Docker's public signing key.
- The `.sources` file tells apt: "packages from this URL, for my Ubuntu release (e.g. `noble`), component `stable`, signed with that key".
- `$(. /etc/os-release && echo ...)` reads your Ubuntu codename automatically. On Debian, replace `ubuntu` with `debian` in both URLs.

Install:

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Install a specific version** (common in production, so all servers match):

```bash
apt-cache madison docker-ce | head -3
```

```
 docker-ce | 5:29.0.1-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
 docker-ce | 5:29.0.0-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
 docker-ce | 5:28.5.2-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
```

```bash
VERSION=5:29.0.1-1~ubuntu.24.04~noble
sudo apt-get install -y docker-ce=$VERSION docker-ce-cli=$VERSION containerd.io docker-buildx-plugin docker-compose-plugin
sudo apt-mark hold docker-ce docker-ce-cli
```

- `apt-mark hold` stops automatic upgrades, so Docker only upgrades when you plan it (topic 49).

### C. RHEL / Rocky / AlmaLinux / CentOS Stream server

```bash
sudo dnf remove -y docker docker-client docker-common docker-latest docker-engine podman runc
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

- RHEL ships Podman by default. The Docker packages conflict with Podman's `runc`, so the first line removes them.
- On RHEL, the service is **not** started automatically. `enable --now` = start now and at every boot.
- On Fedora, use `.../linux/fedora/docker-ce.repo`. Newer Fedora uses dnf5, where the command is `sudo dnf config-manager addrepo --from-repofile=<url>`.
- On CentOS Stream / Rocky / Alma, use the `centos` repo URL.

### D. Check the services

```bash
systemctl status docker --no-pager
```

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-11-17 09:00:12 IST; 2min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1290 (dockerd)
      Tasks: 9
     Memory: 32.4M
        CPU: 410ms
     CGroup: /system.slice/docker.service
             └─1290 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

- `Loaded: ... enabled` = starts at boot.
- `Active: active (running)` = running now.
- `TriggeredBy: docker.socket` = systemd owns the socket and hands it to dockerd (that's the `-H fd://`). If something connects to the socket while dockerd is stopped, systemd starts dockerd.
- `CGroup: /system.slice/docker.service` = dockerd's own cgroup (containers get separate ones).

```bash
systemctl status containerd --no-pager | head -3
```

```
● containerd.service - containerd container runtime
     Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-11-17 09:00:11 IST; 2min ago
```

### E. `docker info`: the health report of a Docker host

```bash
sudo docker info
```

Trimmed sample:

```
Client: Docker Engine - Community
 Version:    29.0.1
 Context:    default
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.30.0
  compose: Docker Compose (Docker Inc.)
    Version:  v2.40.3

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 29.0.1
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 Security Options:
  apparmor
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.8.0-45-generic
 Operating System: Ubuntu 24.04.3 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 7.7GiB
 Name: devbox
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  ::1/128
  127.0.0.0/8
 Live Restore Enabled: false
```

What to read and why it matters:

| Line | Meaning | Why you care |
|---|---|---|
| `Plugins: buildx, compose` | The build and Compose plugins are installed | If missing, `docker compose` fails with "unknown command" |
| `Containers / Running / Stopped` | Counts | Quick health check |
| `Storage Driver: overlayfs` + `driver-type: io.containerd.snapshotter.v1` | Images are stored by **containerd's overlay snapshotter** (the Engine 29+ default for new installs) | On upgraded hosts you'll see `Storage Driver: overlay2` (the older graph driver). Topic 07 shows both. |
| `Logging Driver: json-file` | Container logs are written as JSON files | Without rotation they can fill the disk (topic 33) |
| `Cgroup Driver: systemd`, `Cgroup Version: 2` | systemd manages container cgroups; cgroups v2 | Must match what Kubernetes/kubelet expects; v2 is the modern default |
| `Security Options: apparmor, seccomp, cgroupns` | Default security features are on | `cgroupns` = each container has a private cgroup namespace (topic 41) |
| `Kernel Version` | The host kernel all containers share | Kernel bugs and features apply to every container |
| `Docker Root Dir: /var/lib/docker` | Where Docker keeps its data | If this disk fills up, Docker breaks (topic 48) |
| `Insecure Registries` | Registries Docker allows without TLS (Transport Layer Security: the encryption and certificate checking behind HTTPS; topic 39a) | Only localhost by default. Should stay that way |
| `Live Restore Enabled: false` | Containers stop when dockerd stops | Turn on for less downtime during upgrades (topic 34) |
| `WARNING:` lines at the end | Things Docker thinks are wrong | e.g. "No swap limit support", "bridge-nf-call-iptables is disabled". Always read them. |

### F. Run without `sudo`: the `docker` group (= root!)

```bash
sudo usermod -aG docker $USER
```

- `usermod -aG docker $USER` = **a**ppend your user to the supplementary **G**roup `docker`.
- Group membership is read when you log in, so **log out and back in** (in WSL: close all Ubuntu windows, run `wsl --shutdown` in PowerShell, reopen). Or run `newgrp docker` for the current shell only.

```bash
id
docker run --rm hello-world
```

```
uid=1000(tushar) gid=1000(tushar) groups=1000(tushar),27(sudo),988(docker)
```

**Security truth:** as topic 03 showed, write access to the Docker socket = root on the host. Proof (don't do this on a shared server):

```bash
docker run --rm -v /:/host alpine cat /host/etc/shadow | head -2
```

```
root:*:20000:0:99999:7:::
daemon:*:20000:0:99999:7:::
```

A normal user, without `sudo`, just read the root-only password file, because the container runs as root and `-v /:/host` mounts the host's whole filesystem inside it. (Volumes and `-v` are explained in topic 24. For now: it makes a host folder visible inside the container.)

On a personal lab machine, the `docker` group is fine. On shared servers, **treat the `docker` group like `sudo`**. Give it only to admins, or use rootless Docker (topic 42).

### G. Docker Desktop (if you choose it for daily work)

- Download from Docker's website, install, enable "Use the WSL 2 based engine".
- Check: `docker context ls` shows `desktop-linux` as the active context.
- Licensing: free for personal use, education, open source and small businesses (Docker's terms set the employee/revenue limits); larger companies need a paid subscription.
- For **these notes**, use the WSL2 Docker Engine from section A, so you can inspect the host side directly.

---

## Hands-on lab

1. Install Docker Engine in WSL2 Ubuntu (sections A and B).
2. Run:
   ```bash
   docker version
   docker info | grep -E 'Server Version|Storage Driver|driver-type|Cgroup|Kernel|Root Dir|Live Restore'
   ```
   Write down your answers. You'll use them in later topics (storage driver especially).
3. Prove the shared kernel (from topic 01):
   ```bash
   uname -r
   docker run --rm alpine uname -r
   ```
   Both lines are identical (a `...-microsoft-standard-WSL2` kernel in WSL).
4. Look at the pieces from topic 03:
   ```bash
   systemctl list-units --type=service | grep -E 'docker|containerd'
   ls -l /var/run/docker.sock
   ls /usr/libexec/docker/cli-plugins/
   ```
   ```
   containerd.service  loaded active running containerd container runtime
   docker.service      loaded active running Docker Application Container Engine
   srw-rw---- 1 root docker 0 Nov 17 09:00 /var/run/docker.sock
   docker-buildx  docker-compose
   ```
5. Prove the daemon is needed: `sudo systemctl stop docker docker.socket`, then `docker ps` (see the error), then `sudo systemctl start docker`.
6. From Windows, open a browser after running `docker run -d -p 8080:80 --name webtest nginx` in WSL. Visit `http://localhost:8080`. You'll see the nginx welcome page. Clean up with `docker rm -f webtest`.

---

## Where you'll use it

- **Provisioning servers:** the repository install (often automated with Ansible or cloud-init) with pinned versions across a fleet.
- **CI runners:** installing Docker on build agents (topic 46).
- **Laptops:** WSL2 Engine or Docker Desktop for developers.
- **Audits:** checking who's in the `docker` group, and reading `docker info` warnings.
- **Kubernetes nodes:** you usually install **containerd only**, not Docker. Same `containerd.io` package, different setup (topic 53).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `E: Unable to locate package docker-ce` | The Docker repository isn't set up, or the codename is wrong | `cat /etc/apt/sources.list.d/docker.sources`, then `sudo apt-get update` and read errors |
| `NO_PUBKEY 7EA0A9C3F273FCD8` / `The following signatures couldn't be verified` | The GPG key wasn't downloaded or isn't readable | Re-run the `curl` and `chmod a+r` steps |
| `Problem: problem with installed package podman-docker` (RHEL) | Podman's Docker compatibility package conflicts | `sudo dnf remove podman-docker podman runc` |
| `System has not been booted with systemd as init system` (WSL) | systemd isn't enabled in WSL | Add `[boot] systemd=true` to `/etc/wsl.conf`, `wsl --shutdown` |
| `permission denied while trying to connect to the Docker daemon socket` | You're not in the `docker` group yet, or didn't log out and in | `id` to check; re-login or `newgrp docker` |
| `docker: 'compose' is not a docker command` | The Compose plugin isn't installed | `sudo apt-get install docker-compose-plugin` |
| `Job for docker.service failed` | dockerd couldn't start | `journalctl -u docker -n 50 --no-pager`. Common causes: a bad `/etc/docker/daemon.json` (JSON typo), an iptables problem, or a full disk |
| `docker` in WSL points at Docker Desktop (`Context: desktop-linux`) | Docker Desktop's WSL integration is on | Turn it off for this distro, or switch with `docker context use default` |
| In WSL, containers can't reach the internet | WSL's DNS or VPN setup | Check `cat /etc/resolv.conf` in WSL; try `wsl --shutdown`; corporate VPNs often need WSL's "mirrored" networking mode |

---

## Quick recap

- On Linux servers, install from **Docker's official repository**: `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`. Pin versions in production.
- On Windows, the best learning lab is **Docker Engine inside WSL2 Ubuntu**. WSL2 has a real Linux kernel, so you can inspect everything from the host side.
- Docker Desktop runs Docker Engine in a hidden Linux VM. It's for laptops, with licensing limits for larger companies.
- `docker version` shows client and server. `docker info` shows storage driver, cgroup version, logging driver, root dir and warnings.
- **`docker` group = root.** Treat it like `sudo`.

**Interview questions**

1. **Q: How do you run Linux containers on a Mac or Windows machine?**
   A: Through a Linux VM. Docker Desktop runs Docker Engine in a hidden Linux VM (WSL2 on Windows). Containers share that VM's kernel, not the host OS's.

2. **Q: How would you install Docker on a fleet of production servers?**
   A: From Docker's official package repository, with a pinned version held against automatic upgrades, usually automated with a tool like Ansible. Enable the service at boot and add daemon settings (log rotation, live-restore) in `daemon.json`.

3. **Q: A developer asks to be added to the `docker` group on a shared server. What do you say?**
   A: That it gives them root on the server, because the Docker socket can start privileged containers that mount the host filesystem. Use sudo rules, rootless Docker, or a CI system instead, depending on the need.
