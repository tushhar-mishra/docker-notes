# 34 · Configuring the Docker Daemon

> **Level:** ★★ intermediate
>
> **You will learn:** how `dockerd` is started (the systemd unit and socket), how to configure it with `/etc/docker/daemon.json` (and why not with command-line flags), the production settings that matter (`data-root`, logging, `live-restore`, address pools, MTU, DNS, registry mirrors, proxies, image store, firewall backend, metrics, build cache GC), how to validate and reload safely, and how to manage **remote** Docker hosts securely over SSH with `docker context`, and why you never expose port 2375.
>
> **You need:** topics 03–04 (daemon architecture, systemd), plus the settings introduced in topics 27–33.

---

## What it is

`dockerd` reads its configuration from:
1. **`/etc/docker/daemon.json`** (the recommended place): a JSON file of settings.
2. **Command-line flags** in the systemd unit (`ExecStart=/usr/bin/dockerd -H fd:// --containerd=...`).

The same setting **can't be in both**. If it is, dockerd refuses to start.

A typical production `daemon.json`:

```json
{
  "data-root": "/var/lib/docker",
  "log-driver": "local",
  "log-opts": { "max-size": "20m", "max-file": "5" },
  "live-restore": true,
  "bip": "10.200.0.1/24",
  "default-address-pools": [{ "base": "10.201.0.0/16", "size": 24 }],
  "mtu": 1500,
  "dns": ["10.0.0.2", "10.0.0.3"],
  "registry-mirrors": ["https://mirror.internal.example.com"],
  "metrics-addr": "127.0.0.1:9323",
  "default-ulimits": { "nofile": { "Name": "nofile", "Soft": 65536, "Hard": 65536 } },
  "builder": { "gc": { "enabled": true, "defaultKeepStorage": "20GB" } },
  "features": { "containerd-snapshotter": true }
}
```

---

## Why it exists

Docker's defaults are made for **a laptop**: no log rotation, containers die when dockerd restarts, subnets that may clash with your network, public DNS fallback. Production hosts need these tuned **once, in one file**, ideally managed by config management (Ansible, Puppet, cloud-init) so every host is identical.

---

## Analogy

`daemon.json` is the **building's house rules**, posted at the entrance: how long to keep the mail (logs), which address block the apartments use (subnets), whether residents stay home during a power cut at reception (`live-restore`), and which post office to use (registry mirror). Rules on a separate scrap of paper (command-line flags) that contradict the posted rules = the building manager refuses to open (dockerd won't start).

---

## How it works inside

### The systemd unit and socket

```bash
systemctl cat docker.service
```

```
# /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
After=network-online.target nss-lookup.target docker.socket firewalld.service containerd.service time-set.target
Wants=network-online.target containerd.service
Requires=docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutStartSec=0
RestartSec=2
Restart=always
LimitNOFILE=infinity
LimitNPROC=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
OOMScoreAdjust=-500
...
```

What matters:
- `-H fd://` = listen on the socket that **systemd** created (`docker.socket` → `/run/docker.sock`; `/var/run` is a link to `/run`). That's **socket activation**: systemd owns the socket, so connecting to it can start dockerd.
- `ExecReload=kill -s HUP` = `systemctl reload docker` sends SIGHUP, which reloads **some** settings without a restart.
- `Restart=always` = systemd restarts dockerd if it crashes.
- `Delegate=yes` = dockerd is allowed to manage cgroups under its unit (for containers).
- `KillMode=process` = when stopping docker.service, systemd kills only dockerd, not every process in the unit's cgroup (containers live in their own scopes anyway).
- `OOMScoreAdjust=-500` = the kernel's OOM killer prefers other victims over dockerd.

**Changing the unit:** never edit the file in `/usr/lib/systemd/system/` (package upgrades overwrite it). Use a **drop-in**:

```bash
sudo systemctl edit docker
```

This opens `/etc/systemd/system/docker.service.d/override.conf`. Then `sudo systemctl daemon-reload && sudo systemctl restart docker`.

### Which settings reload without a restart?

`systemctl reload docker` (SIGHUP) applies changes to settings like: `debug`, `log-level`, `labels`, `live-restore`, `max-concurrent-downloads`/`uploads`, `registry-mirrors`, `insecure-registries`, `default-runtime`, `runtimes`, `shutdown-timeout`. Everything else needs a **restart** (which stops containers unless `live-restore` is on).

### The important settings explained

| Setting | What it does | Why you'd set it |
|---|---|---|
| `data-root` | Where dockerd keeps its data (default `/var/lib/docker`) | Put it on a big dedicated disk (topic 48) |
| `log-driver`, `log-opts` | Default logging for **new** containers | Log rotation (topic 33) |
| `live-restore` | **Containers keep running while dockerd is stopped/restarting** | Upgrade/restart dockerd without downtime (topic 49) |
| `bip` | `docker0`'s IP/subnet | Avoid clashing with your networks (topic 27) |
| `default-address-pools` | Subnets for new networks | Same |
| `mtu` | Default MTU for the default bridge | VPN/overlay paths (topic 30) |
| `dns`, `dns-search`, `dns-opts` | Default container DNS / upstream servers | Corporate DNS (topic 29) |
| `registry-mirrors` | Pull-through caches for **Docker Hub** | Rate limits, speed (topic 22) |
| `insecure-registries` | Registries allowed over plain HTTP / untrusted TLS | Labs only |
| `proxies` | `http-proxy`, `https-proxy`, `no-proxy` for **the daemon** (pulls, pushes) | Corporate proxies |
| `metrics-addr` | Expose dockerd's **Prometheus** metrics on this address | Monitoring (topic 47) |
| `default-ulimits` | Default resource limits like max open files (`nofile`) for containers | Apps needing many connections |
| `userland-proxy` | Use `docker-proxy` for published ports (default true) | Disable for performance (topic 28) |
| `iptables`, `ip6tables` | Let Docker manage firewall rules (default true) | **Don't disable** unless you write all rules yourself |
| `firewall-backend` | `iptables` (default) or `nftables` (experimental in 29) | Topic 28 |
| `features.containerd-snapshotter` | Use the containerd image store | Default on fresh Engine 29 installs; switching on an existing host hides old images (topic 49) |
| `builder.gc` | Build cache garbage collection limits | Keep the build cache bounded (topic 48) |
| `exec-opts: ["native.cgroupdriver=systemd"]` | cgroup driver | Already `systemd` on modern installs |
| `shutdown-timeout` | Seconds dockerd waits for containers to stop on daemon shutdown (default 15) | Graceful host shutdowns |
| `debug`, `log-level` | dockerd's **own** log verbosity | Troubleshooting dockerd |
| `default-runtime`, `runtimes` | Use another OCI runtime (crun, gVisor `runsc`, Kata) | Isolation/performance experiments |
| `userns-remap` | User namespace remapping | Topic 42 |
| `icc` | Container-to-container traffic on the default bridge | Hardening (topic 27, 40) |
| `no-new-privileges` | Default `no-new-privileges` for all containers | Hardening (topic 41) |
| `hosts` | Extra listen addresses (sockets, TCP) | **Conflicts with `-H` in the unit**. See below |

### Daemon proxy vs container proxy

Three separate places (topic 30):
- **Daemon** (`docker pull`): `"proxies"` in `daemon.json`, or `Environment="HTTPS_PROXY=..."` in a systemd drop-in.
- **Builds**: `--build-arg HTTPS_PROXY=...`.
- **Containers**: `-e` variables, or `"proxies"` in the **client's** `~/.docker/config.json` (the CLI then injects them into new containers and builds).

### `live-restore` in detail

Without it: stopping dockerd **stops every container**. Restarting it starts only those with restart policies.

With `"live-restore": true`:
- Stopping/restarting/upgrading dockerd leaves containers **running**. Their shims keep them alive (topic 03).
- When dockerd comes back, it **reconnects** to them.
- While dockerd is down: no `docker` commands, no new containers, no restarts of crashed containers, and logs go to a buffer (which can fill and **block** apps if dockerd stays down a long time).
- Doesn't help with containerd upgrades in all cases, and isn't compatible with **Swarm mode**.
- It also doesn't survive a host **reboot** (everything restarts then).

### Remote Docker: contexts and SSH

The CLI can control a daemon on another machine (topic 03). Options:

| Method | Security |
|---|---|
| `tcp://host:2375` (no TLS) | **Never.** Anyone who can reach the port is **root** on that host. Attackers scan the internet for open 2375 ports and use them for crypto-mining within minutes |
| `tcp://host:2376` with **TLS client certificates** (`tlsverify`) | Secure, but you have to manage a CA and certificates |
| **`ssh://user@host`** | **Recommended.** Uses your normal SSH keys and access control. The remote user must be allowed to use the socket (docker group = root there!) |

`docker context` saves named connection targets so you can switch between them:

```bash
docker context create prod-web-1 --docker "host=ssh://deploy@10.0.0.21"
docker context ls
docker context use prod-web-1
docker ps                                   # now runs against prod-web-1
docker --context default ps                 # one command against another context
DOCKER_CONTEXT=prod-web-1 docker ps         # via environment variable
DOCKER_HOST=ssh://deploy@10.0.0.21 docker ps  # ad hoc, without a context
```

- The active context is shown in `docker version` / `docker info` (`Context:`). It's a common surprise: "my containers disappeared!" when you're simply pointed at another host.

---

## Commands / files

### Edit, validate, restart

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
sudo dockerd --validate --config-file /etc/docker/daemon.json
```

```
configuration OK
```

- **Always validate before restarting.** A typo here means dockerd won't start, and on a production host that means every container is down (unless live-restore).

```bash
sudo systemctl restart docker
docker info --format 'LiveRestore={{.LiveRestoreEnabled}} LogDriver={{.LoggingDriver}} Root={{.DockerRootDir}}'
docker info | grep -A1 'Registry Mirrors'
```

```
LiveRestore=true LogDriver=local Root=/var/lib/docker
 Registry Mirrors:
  https://mirror.internal.example.com/
```

### Reload without restart

```bash
sudo sed -i 's/"live-restore": false/"live-restore": true/' /etc/docker/daemon.json
sudo systemctl reload docker
docker info --format '{{.LiveRestoreEnabled}}'
journalctl -u docker -n 5 --no-pager | grep -i reload
```

```
true
Nov 17 15:01:02 devbox dockerd[1290]: time="2026-11-17T15:01:02.1+05:30" level=info msg="Got signal to reload configuration, reloading from: /etc/docker/daemon.json"
```

### The `-H` vs `hosts` conflict

```json
{ "hosts": ["unix:///var/run/docker.sock", "tcp://127.0.0.1:2375"] }
```

```bash
sudo systemctl restart docker
journalctl -u docker -n 3 --no-pager
```

```
dockerd[2211]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: hosts: (from flag: [fd://], from file: [unix:///var/run/docker.sock tcp://127.0.0.1:2375])
systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
```

- Fix: remove `-H fd://` from `ExecStart` with a drop-in (an empty `ExecStart=` line resets it, then the new one):
  ```ini
  [Service]
  ExecStart=
  ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
  ```
  (And don't expose TCP without TLS. This example uses `127.0.0.1` only.)

### Daemon proxy via systemd drop-in

```bash
sudo systemctl edit docker
```

```ini
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.corp,10.0.0.0/8,registry.internal"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
docker info | grep -i proxy
```

```
 HTTP Proxy: http://proxy.corp:3128
 HTTPS Proxy: http://proxy.corp:3128
 No Proxy: localhost,127.0.0.1,.corp,10.0.0.0/8,registry.internal
```

### dockerd's own logs and debug mode

```bash
journalctl -u docker -f
journalctl -u docker --since "10 min ago" -p warning --no-pager
```

Turn on debug logging temporarily: `"debug": true` + `systemctl reload docker`. Remember to turn it off again (it's very verbose).

### Remote context over SSH

```bash
docker context create lab-vm --docker "host=ssh://tushar@192.168.1.60"
docker --context lab-vm info --format '{{.Name}} {{.ServerVersion}}'
```

```
lab-vm
lab-vm-hostname 29.0.1
```

---

## Hands-on lab

1. Write a `daemon.json` with: `local` log driver + rotation, `live-restore: true`, `default-address-pools` of `10.201.0.0/16` size 24, and `metrics-addr` `127.0.0.1:9323`. Validate, restart, and verify each with `docker info` and by creating a network (`docker network create t && docker network inspect t -f '{{json .IPAM.Config}}'`).
2. **live-restore test:**
   ```bash
   docker run -d --name lr nginx:1.27-alpine
   PID=$(docker inspect -f '{{.State.Pid}}' lr)
   sudo systemctl stop docker docker.socket
   ps -p $PID -o pid,cmd            # still running!
   sudo systemctl start docker
   docker ps --filter name=lr
   ```
3. **Metrics:** `curl -s 127.0.0.1:9323/metrics | grep -E '^engine_daemon_container_states_containers'`.
4. **Break and fix:** put a trailing comma in `daemon.json`, run `dockerd --validate` and read the error. Fix it.
5. If you have a second Linux VM: create an SSH context to it and run `docker ps` remotely.
6. Clean up: `docker rm -f lr; docker network rm t`

---

## Where you'll use it

- **Host provisioning:** a standard `daemon.json` deployed by Ansible/cloud-init to every Docker host.
- **Upgrades without downtime:** `live-restore` (topic 49).
- **Corporate networks:** proxies, DNS, address pools, MTU, registry mirrors, private CA certs.
- **Monitoring:** `metrics-addr` for Prometheus (topic 47).
- **Operations:** `docker context` to manage several hosts from one workstation.
- **Security audits:** checking for exposed TCP sockets (`ss -lntp | grep dockerd`), insecure registries and disabled iptables.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Job for docker.service failed` after editing `daemon.json` | Invalid JSON or an unknown key | `journalctl -u docker -n 20`; `dockerd --validate` |
| `the following directives are specified both as a flag and in the configuration file` | Same setting in the unit's flags and in `daemon.json` (often `hosts`) | Remove it from one place (drop-in with empty `ExecStart=`) |
| Restarting docker stopped all containers | `live-restore` off | Enable it (reloadable) |
| Changed `log-opts`/`dns`, no effect on running containers | These apply at container creation | Recreate the containers |
| `docker pull` fails behind a proxy, though `curl` works in your shell | The **daemon** has no proxy config | `proxies` in `daemon.json` or a systemd `Environment=` drop-in |
| "All my containers disappeared!" | CLI pointed at another context/host | `docker context ls`, `echo $DOCKER_HOST` |
| Host compromised, crypto-miners in containers | dockerd exposed on TCP 2375 without TLS | Remove the TCP listener; rotate everything; use SSH contexts |
| After enabling `containerd-snapshotter`, all images are "gone" | Different image store; the old images are still on disk in the old store | Expected. Re-pull, or switch back (topic 49) |
| `Error starting daemon: error initializing graphdriver: ... data-root` after moving it | Wrong permissions/filesystem at the new path, or files not copied | Topic 48 (moving data-root) |

---

**Later:** kernel settings for busy container hosts (conntrack, inotify, open files, `vm.max_map_count`, cgroup v1) are in topic 48a.

## Quick recap

- dockerd runs from `docker.service` (socket-activated via `docker.socket`, `-H fd://`). Change the unit only with **drop-ins**.
- Configure via **`/etc/docker/daemon.json`**. Never set the same option as a flag too. **Validate** with `dockerd --validate` before restarting.
- Key production settings: log rotation, `live-restore`, `bip`/`default-address-pools`, `mtu`, `dns`, `registry-mirrors`, `proxies`, `metrics-addr`, `builder.gc`, `data-root`.
- `systemctl reload docker` (SIGHUP) applies some settings without a restart. Container-level defaults only affect **new** containers.
- Manage remote hosts with **`docker context` over SSH**. Never expose TCP 2375 without TLS.

**Interview questions**

1. **Q: How do you restart or upgrade the Docker daemon without stopping containers?**
   A: Enable `"live-restore": true` in daemon.json (reloadable with SIGHUP). Containers keep running under their containerd shims while dockerd is down, and dockerd reattaches when it comes back. It doesn't work in Swarm mode or across host reboots.

2. **Q: Why is exposing the Docker API on tcp/2375 dangerous?**
   A: It's unauthenticated and unencrypted, and the API gives full control of dockerd, which runs as root, so anyone reaching the port is effectively root on the host. Use SSH contexts or TLS with client certificates instead.

3. **Q: Docker pulls fail behind a corporate proxy even though the shell has `HTTPS_PROXY` set. Why?**
   A: Pulls are performed by dockerd, a systemd service that doesn't inherit your shell's environment. Configure the proxy for the daemon in daemon.json (`proxies`) or with a systemd drop-in, then restart dockerd.
