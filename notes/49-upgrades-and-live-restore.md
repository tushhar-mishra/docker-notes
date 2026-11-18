# 49 · Upgrading Docker and Live-Restore

> **Level:** ★★ intermediate
>
> **You will learn:** what each Docker package is and what restarting each one does to running containers, a safe upgrade procedure (release notes → staging → backup → pinned install → verify → rollback plan), exactly what `live-restore` protects and its limits, API version skew between clients/tools and the daemon (Engine 29 raised the minimum API version), and how to migrate an existing host to the containerd image store.
>
> **You need:** topics 03–04 (components, packages), topic 34 (`live-restore`, `daemon.json`), topic 48 (storage).

---

## What it is

A Docker host runs several separately-packaged components (topic 04):

| Package | Component | What restarting/upgrading it does to running containers |
|---|---|---|
| `docker-ce` | dockerd | Without `live-restore`: **all containers stop** (then restart if they have restart policies). With `live-restore`: containers keep running (within limits below) |
| `containerd.io` | containerd, shim, runc | containerd restarts; **running containers keep running** (the shims are independent). A new runc applies to **new** containers/execs only |
| `docker-ce-cli` | docker CLI | No effect on containers |
| `docker-buildx-plugin`, `docker-compose-plugin` | CLI plugins | No effect on containers |
| Kernel | The host kernel | **Requires a reboot**: every container restarts |

Docker Engine versions use `MAJOR.MINOR.PATCH` (e.g. 29.0.1 → 29.1.0 → 30.0.0). Patch releases bring fixes (often security fixes); minor/major releases can bring behaviour changes and remove deprecated features.

---

## Why it exists

You must upgrade: security fixes (runc and Engine CVEs; topic 40), bug fixes, new features, and support windows. But a careless upgrade on a production host can:
- stop every container (no `live-restore`),
- break tools that speak an older API version,
- change default behaviour (storage, networking, firewall rules),
- surprise you with "missing" images (image store change).

A routine, tested procedure turns upgrades into a non-event.

---

## Analogy

Replacing the **engine of a running bus**. Normally passengers must get off (containers stop). With `live-restore`, it's like swapping the **driver** at a stop while the passengers stay seated (dockerd restarts, containers keep running). It works for swapping drivers with the same training (patch upgrades). For a completely different bus model (major upgrade), passengers should get off and back on.

---

## How it works inside

### What `live-restore` really does

With `"live-restore": true` (topic 34):
1. On shutdown, dockerd **doesn't** stop containers. It just exits.
2. Containers keep running under their **shims**. containerd keeps track of them.
3. On startup, dockerd **reconnects**: it reads its saved container state, asks containerd for the running tasks, re-opens their stdio/log pipes, and reattaches networking state.

Limits you must know:
- **Upgrades across many versions:** Docker documents live-restore as supported for upgrades between **patch** releases (e.g. 29.0.1 → 29.0.2). Skipping minor/major versions may leave dockerd unable to reconnect, so containers keep running "orphaned", or need a restart. For minor/major upgrades, **plan a restart** of containers (a maintenance window, or move the traffic away first).
- **Not in Swarm mode.**
- **While dockerd is down:** no `docker` commands, no restarts of crashed containers, health checks pause, and container output is buffered in a FIFO. If dockerd stays down too long and the buffer fills (64 KB by default), **writes to stdout block** and apps can freeze. Keep the downtime short.
- **Host reboots** restart everything regardless.
- Published ports keep working (iptables rules stay), but `docker-proxy` processes are tied to dockerd (loopback-published traffic may break while it's down).

### Package upgrades and automatic restarts

On Debian/Ubuntu, upgrading `docker-ce` runs the package's post-install script, which **restarts docker.service**. So an unattended `apt upgrade` can restart dockerd at a random time. That's why you **pin** versions (`apt-mark hold`, topic 04) and upgrade on purpose.

### API version skew

The CLI and the daemon negotiate an API version (topic 03):
- **Newer client, older daemon:** the client automatically talks the daemon's (lower) version. It works, but new features are unavailable.
- **Older client, newer daemon:** works as long as the client's version is **at or above the daemon's minimum**. **Engine 29 raised the minimum API version to 1.44** (Engine 25-era). Older clients get:
  ```
  Error response from daemon: client version 1.41 is too old. Minimum supported API version is 1.44, please upgrade your client to a newer version
  ```
- Who's affected in practice: **tools with an embedded Docker SDK** pinned to an old API (old Testcontainers, old docker-py/docker SDK versions, older Portainer/Traefik/Watchtower/CI plugins, old `docker` CLI binaries inside CI images like `docker:20`).
- Check before upgrading: inventory tools that talk to the socket. `DOCKER_API_VERSION=1.44` forces a client to a specific version, but only helps if the tool supports it.
- The daemon can be told to accept older clients (`DOCKER_MIN_API_VERSION` environment variable for dockerd) as a **temporary** compatibility measure while you upgrade the tools.

### Migrating to the containerd image store

Fresh Engine 29 installs use the containerd image store. **Upgraded** hosts keep the classic overlay2 store until you switch:

```json
{ "features": { "containerd-snapshotter": true } }
```

What happens when you switch on an existing host:
- dockerd now uses containerd's image store. **Existing images and containers become invisible** (they're still on disk in the old store, not migrated).
- **Volumes are unaffected** (`/var/lib/docker/volumes`).
- You must **re-pull** (or `docker save` before / `docker load` after) images and **recreate** containers.
- Disk usage rises (compressed + unpacked; topic 48) and you temporarily hold both stores.
- Switching back (`false`) makes the old images and containers visible again.
- Not compatible with `userns-remap` (topic 42).

Benefits: multi-platform images and attestations locally, `--load` for multi-platform builds (topic 21), and the same storage model as Kubernetes nodes.

### Safe upgrade procedure (single host)

1. **Read the release notes** for every version between current and target: breaking changes, deprecations, minimum API version, changed defaults.
2. **Inventory clients**: tools that use the socket/API (and their SDK versions).
3. **Test in staging** (same OS, same `daemon.json`, same compose stacks). Run your smoke tests.
4. **Back up**: volumes (logical DB dumps; topic 24), `/etc/docker`, compose files. Note the current versions (`docker version`) for rollback.
5. **Choose the impact mode:**
   - patch upgrade + `live-restore` → containers stay up,
   - minor/major upgrade → maintenance window, or move traffic away from the host first (in a fleet: one host at a time behind the load balancer).
6. **Upgrade with pinned versions** (all of docker-ce, docker-ce-cli, containerd.io and plugins together, versions tested in staging).
7. **Verify**: `docker version`, `docker info` (warnings!), containers running and healthy, logs, published ports, a test deploy.
8. **Rollback plan**: install the previous pinned versions. (Downgrades across major versions aren't guaranteed to read newer state, which is why staging and backups matter.)

### Kernel and OS upgrades

- Kernel updates need a **reboot**: all containers restart (restart policies bring them back; topic 32). Plan these like the minor/major Docker upgrades: drain traffic, reboot, verify.
- Distribution major upgrades (Ubuntu 22.04 → 24.04) may change the iptables backend, the cgroup version or the AppArmor profiles. Treat them as a host rebuild and test carefully.

---

## Commands / files

### Before: record versions and state

```bash
docker version --format 'Client={{.Client.Version}} Server={{.Server.Version}} API={{.Server.APIVersion}} MinAPI={{.Server.MinAPIVersion}}'
dpkg -l | grep -E 'docker-ce|containerd.io|docker-buildx|docker-compose-plugin' | awk '{print $2, $3}'
docker info --format 'LiveRestore={{.LiveRestoreEnabled}} Driver={{.Driver}}'
docker ps --format '{{.Names}} {{.Status}}' > /root/pre-upgrade-containers.txt
```

```
Client=28.5.2 Server=28.5.2 API=1.51 MinAPI=1.24
containerd.io 1.7.28-1
docker-buildx-plugin 0.29.1-1~ubuntu.24.04~noble
docker-ce 5:28.5.2-1~ubuntu.24.04~noble
docker-ce-cli 5:28.5.2-1~ubuntu.24.04~noble
docker-compose-plugin 2.40.3-1~ubuntu.24.04~noble
LiveRestore=true Driver=overlay2
```

- Note `MinAPI=1.24` on Engine 28 vs `1.44` on Engine 29: older tools that work today may break after the upgrade.

### Find which clients talk to the daemon (and their API versions)

```bash
sudo journalctl -u docker --since "1 day ago" | grep -o 'API version [0-9.]*' | sort | uniq -c
sudo ss -xp | grep docker.sock | awk '{print $NF}' | sort -u | head
```

(With `"debug": true`, dockerd logs each request's path including `/v1.xx/`, which shows exactly which API versions clients use. Turn debug off afterwards.)

### Upgrade (pinned)

```bash
sudo apt-mark unhold docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
apt-cache madison docker-ce | head -3
V=5:29.0.1-1~ubuntu.24.04~noble
sudo apt-get install -y docker-ce=$V docker-ce-cli=$V containerd.io docker-buildx-plugin docker-compose-plugin
sudo apt-mark hold docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Watch containers survive a patch upgrade with live-restore

```bash
docker run -d --name survivor nginx:1.27-alpine
PID_BEFORE=$(docker inspect -f '{{.State.Pid}}' survivor)
sudo systemctl restart docker            # same effect as the package's restart
PID_AFTER=$(docker inspect -f '{{.State.Pid}}' survivor)
echo "$PID_BEFORE -> $PID_AFTER"
docker inspect -f '{{.State.StartedAt}}' survivor
```

```
48211 -> 48211
2026-11-17T12:00:01.123456789Z
```

- Same PID and the same start time: the container never stopped.

containerd restart (containers survive even **without** live-restore):

```bash
sudo systemctl restart containerd
docker inspect -f '{{.State.Pid}}' survivor
```

### Verify after

```bash
docker version --format 'Server={{.Server.Version}} API={{.Server.APIVersion}} MinAPI={{.Server.MinAPIVersion}}'
docker info 2>&1 | grep -i warning
diff <(cut -d' ' -f1 /root/pre-upgrade-containers.txt | sort) <(docker ps --format '{{.Names}}' | sort) && echo "same containers running"
docker ps --filter health=unhealthy
docker run --rm hello-world > /dev/null && echo "smoke test OK"
```

### An old client against a new daemon

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker:20.10-cli docker version
```

```
Client: Docker Engine - Community
 Version:           20.10.24
 API version:       1.41
 ...
Error response from daemon: client version 1.41 is too old. Minimum supported API version is 1.44, please upgrade your client to a newer version
```

### Switch an existing host to the containerd image store

```bash
docker images --format '{{.Repository}}:{{.Tag}}' | grep -v '<none>' > /root/images.txt
# edit /etc/docker/daemon.json → add:  "features": { "containerd-snapshotter": true }
sudo dockerd --validate --config-file /etc/docker/daemon.json && sudo systemctl restart docker
docker info | grep -A1 'Storage Driver'
docker images                                   # empty (old images are in the old store)
xargs -n1 docker pull < /root/images.txt        # re-pull
# recreate containers (e.g. docker compose up -d --force-recreate in each project)
```

```
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
```

---

## Hands-on lab

1. Record versions and state as in "Before". Turn `live-restore` on and test the `survivor` demo with `systemctl restart docker`, then with `systemctl restart containerd`.
2. Turn `live-restore` **off**, restart docker, and see `survivor` restarted (new PID) or stopped (depending on its restart policy). Turn it back on.
3. Run the old-client demo and read the error.
4. On a lab VM with an older Engine (e.g. install `5:28.5.x`), upgrade to the current pinned 29.x following the procedure. Check `docker info` warnings afterwards.
5. **Optional:** switch the lab VM to the containerd image store, see images "disappear", re-pull them, then switch back and see the old ones reappear.
6. Clean up: `docker rm -f survivor`

---

## Where you'll use it

- **Patch management:** monthly/urgent security upgrades of Engine, containerd and runc across fleets.
- **Change management:** written upgrade runbooks with staging tests and rollback steps.
- **Tool compatibility:** upgrading CI images, SDKs and dashboards before raising the Engine version.
- **Platform migrations:** moving to the containerd image store (and getting closer to how Kubernetes nodes work).
- **Kubernetes:** node upgrades follow the same idea: cordon/drain a node (move work away), upgrade containerd/kubelet/kernel, uncordon, one node at a time (topic 53).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| All containers stopped during an `apt upgrade` | dockerd restarted without `live-restore` | Enable live-restore; pin and hold packages; upgrade deliberately |
| After a major upgrade, `docker ps` shows containers as stopped, but processes still run | live-restore couldn't reattach across versions | Stop the orphans cleanly (`docker kill`, or kill the shim-held processes), then start containers; next time stop containers before major upgrades |
| `client version 1.xx is too old. Minimum supported API version is 1.44` | Old SDK/CLI after the Engine 29 upgrade | Upgrade the tool; temporarily set `DOCKER_MIN_API_VERSION` on dockerd |
| All images "gone" after the upgrade / config change | Image store switched (containerd snapshotter) | Expected; re-pull/load; or switch back |
| `docker info` shows new WARNING lines | Deprecated options in daemon.json, missing kernel features | Read and fix each one |
| New containers fail, old ones run fine | New runc/containerd behaviour (e.g. stricter mount or security handling) | Read release notes; test in staging; adjust options |
| Firewall/published ports behave differently after an OS upgrade | iptables backend change (legacy ↔ nft) | Check `iptables -V`; restart Docker to rebuild rules; align backends |
| Apps froze while dockerd was down (live-restore) | stdout pipe buffer filled | Keep dockerd downtime short; non-blocking log mode for noisy apps |

---

## Quick recap

- Restarting **dockerd** stops containers unless `live-restore` is on. Restarting **containerd** doesn't stop running containers (the shims keep them). The CLI and plugins don't matter. **Kernel** upgrades need a reboot.
- `live-restore` is reliable for **patch** upgrades. For minor/major ones, plan container restarts or move traffic away. It doesn't work in Swarm, and long daemon downtime can block app output.
- Pin and hold packages. Upgrade deliberately: release notes → staging → backup → pinned install → verify → rollback plan.
- **Engine 29's minimum API version is 1.44**: old SDKs, tools and CLI images break. Inventory them first.
- Switching an existing host to the **containerd image store** hides old images and containers (volumes are fine). Re-pull and recreate.

**Interview questions**

1. **Q: How do you upgrade Docker on a production host without downtime?**
   A: For patch releases, enable `live-restore` so containers keep running while dockerd restarts, and upgrade pinned packages after testing in staging. For bigger upgrades, or for guaranteed zero downtime, move traffic away from the host (fleet behind a load balancer), upgrade, verify, and move on to the next host.

2. **Q: What happens to running containers if containerd is restarted?**
   A: They keep running. Each container's process is parented by its containerd-shim, not containerd, and containerd reconnects to the shims when it starts again.

3. **Q: After upgrading Docker, a CI tool fails with "client version is too old". Why?**
   A: Newer daemons raise the minimum supported API version (1.44 in Engine 29), and the tool uses an old Docker SDK/CLI. Upgrade the tool or its Docker client library, and only temporarily lower the daemon's minimum API version if necessary.
