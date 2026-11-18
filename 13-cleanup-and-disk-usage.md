# 13 · Cleaning Up and Disk Usage

> **Level:** ★ foundation
>
> **You will learn:** where Docker's disk space goes (images, containers, volumes, build cache, logs), how to measure it with `docker system df`, what "dangling" and "unused" really mean, every `prune` command and exactly what it deletes, and what is (and isn't) safe to delete on a production server.
>
> **You need:** topics 06–08 and 10.

---

## What it is

Docker never deletes anything on its own. Over time a host collects:

| Thing | Comes from | Where it lives |
|---|---|---|
| **Images** | every `pull` and `build` | `/var/lib/containerd/…` (containerd store) or `/var/lib/docker/overlay2` |
| **Stopped containers** | every `docker run` without `--rm` | their writable layers + `/var/lib/docker/containers/<id>/` |
| **Volumes** | databases and apps storing data (topic 24) | `/var/lib/docker/volumes/` |
| **Build cache** | every `docker build`: BuildKit keeps intermediate results to make the next build fast (topic 16) | `/var/lib/docker/buildkit/` (or the builder's own storage) |
| **Container logs** | everything apps print to stdout/stderr | `/var/lib/docker/containers/<id>/<id>-json.log` |
| **Networks** | `docker network create`, Compose | no real disk use, but they hold IP ranges (topic 27) |

`prune` commands delete the unused ones. Knowing **exactly what each one deletes** keeps you from deleting a production database.

---

## Why it exists

"Disk full on the Docker host" is one of the most common production incidents. When `/var/lib/docker` fills up:
- pulls fail, builds fail,
- containers can't write to their writable layer and crash,
- logs stop, and sometimes the daemon itself won't start.

CI runners and build servers are hit hardest: hundreds of builds a day, each leaving images and cache behind.

---

## Analogy

Your **kitchen** after weeks of cooking:
- **Dangling images** = empty jars with no label. Nobody will ever look for them by name.
- **Unused images** = labelled jars nobody is cooking with right now. You *might* need them again.
- **Stopped containers** = dirty pans left on the stove.
- **Volumes** = the **fridge**. Throwing out "unused" fridge contents can mean throwing away food someone planned to eat. Be careful.
- **Build cache** = prepped ingredients you saved to cook faster next time. Safe to throw away, but the next meal takes longer to make.
- **Logs** = receipts piling up in a drawer. Nobody cleans that drawer unless you set a rule (log rotation).

---

## How it works inside

### "Dangling" vs "unused"

- **Dangling image** = an image with **no tag** (shown as `<none>:<none>`). This happens when you build `myapp:latest` again: the new image takes the tag, and the old one is left nameless. Nothing refers to it by name. Usually safe to delete.
- **Unused image** = an image (tagged or not) that **no container uses**, running or stopped. Deleting it is "safe" in that nothing breaks right now, but the next `docker run` will need to pull or rebuild it (slow, and maybe impossible if the registry is down or the tag has moved).

### What the `prune` commands delete

| Command | Deletes | Keeps |
|---|---|---|
| `docker container prune` | All **stopped** containers (their writable layers and logs) | Running containers |
| `docker image prune` | **Dangling** images only | Every tagged image |
| `docker image prune -a` | **All images not used by any container** (running or stopped) | Images used by containers |
| `docker volume prune` | **Anonymous** volumes not used by any container | **Named** volumes (since Docker 23) |
| `docker volume prune -a` | **All** volumes not used by any container, **including named ones = possibly your data** | Volumes used by containers (even stopped ones) |
| `docker network prune` | Custom networks with no containers attached | Default networks (bridge, host, none) |
| `docker builder prune` | Build cache not used recently (dangling cache) | — |
| `docker builder prune -a` | **All** build cache | — |
| `docker system prune` | Stopped containers + unused networks + dangling images + dangling build cache | Volumes, tagged images |
| `docker system prune -a` | The above + **all unused images** + all build cache | Volumes |
| `docker system prune -a --volumes` | The above + **anonymous** unused volumes | Named volumes |

> **Anonymous volume** = a volume Docker created automatically without a name you chose (it gets a random 64-character name). **Named volume** = one you created with a name, like `pgdata`. Topic 24 explains both.

The safety rules hidden in that table:
1. **Nothing in use is deleted.** A volume or image used by **any** container (even a stopped one) stays.
2. **That means a stopped container protects its volume, and `container prune` removes that protection.** Running `container prune` and then `volume prune -a` can delete the data of a database whose container was just stopped for maintenance.
3. **Logs of running containers are never pruned.** Only log rotation limits them (topic 33).

### Filters

Most prune commands accept `--filter`:

```bash
docker image prune -a --filter "until=168h"          # unused images created more than 7 days ago
docker container prune --filter "until=24h"          # stopped containers older than 24 h
docker builder prune --filter "until=72h"            # cache not used in 3 days
docker image prune -a --filter "label!=keep=true"    # skip images labelled keep=true
docker builder prune --keep-storage 10GB             # trim the cache down to 10 GB
```

- `until` = only things **created** before that age (for build cache: last **used** before).
- `label` / `label!` = include / exclude by label.

### Space Docker doesn't show

`docker system df` doesn't include:
- container **log files** (check them with `du`),
- data in **bind mounts** (they're just host folders; topic 25),
- leftover files from crashed or old setups.

---

## Commands / files

### `docker system df`

```bash
docker system df
```

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          23        5         7.84GB    5.92GB (75%)
Containers      31        4         412MB     398MB (96%)
Local Volumes   9         3         2.1GB     640MB (30%)
Build Cache     187       0         4.33GB    4.33GB
```

- `TOTAL` = how many exist. `ACTIVE` = how many are in use by containers (for containers: running).
- `SIZE` = disk used. Shared image layers are counted once.
- `RECLAIMABLE` = what you'd get back by removing everything **not active**. For images that's "unused", which is more than "dangling".
- Here: 31 containers but only 4 running, so 27 stopped containers hold 398 MB. 4.33 GB of build cache, all reclaimable.

Detailed view:

```bash
docker system df -v
```

```
Images space usage:

REPOSITORY   TAG           IMAGE ID       CREATED        SIZE      SHARED SIZE   UNIQUE SIZE   CONTAINERS
myapp        1.4.2         a1b2c3d4e5f6   2 days ago     412MB     74.8MB        337MB         2
myapp        <none>        f6e5d4c3b2a1   3 days ago     410MB     74.8MB        335MB         0
...

Containers space usage:

CONTAINER ID   IMAGE         COMMAND      LOCAL VOLUMES   SIZE      CREATED       STATUS                   NAMES
...

Local Volumes space usage:

VOLUME NAME                                                        LINKS     SIZE
pgdata                                                             1         1.4GB
3f9a0c2e…(64 chars)…                                               0         640MB
...

Build cache usage: 4.33GB

CACHE ID       CACHE TYPE     SIZE      CREATED        LAST USED      USAGE     SHARED
k2j3h4g5f6d7   regular        312MB     2 days ago     2 days ago     1         false
...
```

- `SHARED SIZE` = bytes in layers other images also use. `UNIQUE SIZE` = what deleting **this** image would free.
- `myapp <none>` with `CONTAINERS 0` = a dangling image.
- Volumes: `LINKS 0` = no container uses it. A long random name = anonymous.

### Find dangling images

```bash
docker images --filter dangling=true
```

### Container logs (not in `system df`)

```bash
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail -5
```

```
12M     /var/lib/docker/containers/1c2d…/1c2d…-json.log
48M     /var/lib/docker/containers/9a8b…/9a8b…-json.log
6.2G    /var/lib/docker/containers/5e6f…/5e6f…-json.log
```

- One chatty container with no log rotation = 6.2 GB. Topic 33 fixes this permanently.
- **Emergency** fix without stopping the container: `sudo truncate -s 0 /var/lib/docker/containers/5e6f…/5e6f…-json.log`. Don't `rm` it: Docker keeps the file open, so the space isn't freed until the container restarts. `truncate` empties it in place.

### Prune examples with output

```bash
docker container prune -f
```

```
Deleted Containers:
4f5e6d7c8b9a…
3a2b1c0d9e8f…

Total reclaimed space: 398MB
```

- `-f` = don't ask for confirmation. Use it in scripts, carefully.

```bash
docker image prune -a --filter "until=168h"
```

```
WARNING! This will remove all images without at least one container associated to them.
Are you sure you want to continue? [y/N] y
Deleted Images:
untagged: myapp:1.3.9
untagged: myapp@sha256:…
deleted: sha256:…
...

Total reclaimed space: 3.1GB
```

```bash
docker builder prune --keep-storage 2GB -f
```

```
ID                                              RECLAIMABLE     SIZE            LAST ACCESSED
k2j3h4g5f6d7…                                   true            312MB           2 days ago
...
Total:  2.33GB
```

### Before deleting volumes, look

```bash
docker volume ls -f dangling=true
docker volume inspect 3f9a0c2e… --format '{{.CreatedAt}} {{.Labels}}'
sudo ls /var/lib/docker/volumes/3f9a0c2e…/_data | head
```

- `dangling=true` for volumes = not used by any container.
- Look at what's inside before you delete. A Postgres data folder (`PG_VERSION`, `base/`, `pg_wal/`) is not something to throw away.

---

## Hands-on lab

1. Make a mess on purpose:
   ```bash
   for i in 1 2 3 4 5; do docker run alpine echo $i; done           # 5 stopped containers
   mkdir -p ~/mess && cd ~/mess
   for v in 1 2 3; do printf 'FROM alpine:3.22\nRUN echo build %s > /v.txt\n' $v > Dockerfile; docker build -q -t mess:latest .; done
   docker volume create keepme
   docker run -v /data alpine true                                   # creates an anonymous volume
   docker system df
   docker images --filter dangling=true
   ```
   You'll see two dangling images (from rebuilding `mess:latest` twice), 6 stopped containers, and 2 volumes.

2. Prune step by step, checking `docker system df` after each:
   ```bash
   docker container prune -f
   docker image prune -f
   docker volume prune -f
   docker volume ls
   ```
   `keepme` (named) survives `volume prune`. The anonymous one is gone.

3. See the protection rule:
   ```bash
   docker run --name holder -v keepme:/data alpine true
   docker volume prune -a -f       # keepme is NOT deleted: a stopped container uses it
   docker volume ls
   docker rm holder
   docker volume prune -a -f       # now it is deleted
   docker volume ls
   ```

4. Clean up the rest: `docker rmi mess:latest; rm -rf ~/mess; docker builder prune -f`

---

## Where you'll use it

- **CI runners:** a scheduled job (cron or systemd timer) like:
  ```bash
  docker container prune -f --filter "until=24h"
  docker image prune -a -f --filter "until=72h"
  docker builder prune -f --keep-storage 20GB
  ```
- **Production servers:** careful, targeted cleanup. Never `--volumes` blindly. Usually `docker image prune -a --filter until=…` plus log rotation is enough.
- **Disk-full incidents:** `df -h` → `docker system df` → `du` on logs → prune or truncate (full method in topic 48).
- **Monitoring:** alert on `/var/lib/docker` (and `/var/lib/containerd`) usage above ~80%.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `no space left on device` on pull/build/run | The Docker data disk is full (or out of inodes: `df -i`) | `docker system df`, prune, check logs with `du`; see topic 48 |
| `docker system prune` freed little, disk still full | Space is in **logs**, **volumes**, **bind mounts**, or images still used by stopped containers | `du` on `/var/lib/docker/containers`, `docker system df -v`, `container prune` first |
| Deleted the log file with `rm`, but `df` still shows it used | The file is still open by dockerd; the space is freed only when it's closed | `truncate -s 0` instead; or restart the container |
| `Error response from daemon: a prune operation is already running` | Two prunes at once (e.g. two cron jobs) | Wait, and don't schedule overlapping prunes |
| A database lost its data after "cleanup" | A named volume was deleted with `volume prune -a` or `system prune --volumes`, after its container was removed | Restore from backup. Never prune volumes on production hosts without checking each one. Label important volumes and filter them out. |
| Next deploy is slow / fails pulling | `image prune -a` removed images you needed, and the registry is slow, rate-limited or down | Prune with an `until` filter; keep the current and previous release images |
| Build cache grows back to tens of GB fast | Normal on busy CI | `--keep-storage`, or configure BuildKit's garbage collection (topic 48) |

---

## Quick recap

- Disk space goes to: images, stopped containers, volumes, build cache, and **logs** (which `system df` doesn't show).
- **Dangling** = untagged images, almost always safe. **Unused** = not used by any container, safe for now but must be re-pulled later.
- `volume prune` deletes only **anonymous** unused volumes. `-a` includes **named** ones too, so it can delete real data.
- Anything used by a container (even a stopped one) is protected. `container prune` removes that protection.
- Truncate runaway log files, don't `rm` them. Set log rotation so it doesn't happen again (topic 33).

**Interview questions**

1. **Q: What's the difference between `docker image prune` and `docker image prune -a`?**
   A: Without `-a`, it removes only dangling (untagged) images. With `-a`, it removes every image that no container uses, tagged or not.

2. **Q: A Docker host's disk is full. How do you investigate?**
   A: `df -h` and `df -i` to confirm which filesystem and whether it's blocks or inodes. Then `docker system df -v` for images, containers, volumes and build cache, and `du` on `/var/lib/docker/containers/*/*-json.log` for logs. Then prune the safe categories, truncate huge logs, and set log rotation and scheduled cleanup.

3. **Q: Is `docker system prune -a --volumes` safe on a production server?**
   A: No. It removes every unused image and anonymous volume, and if containers were stopped or removed, data volumes may count as unused. Prefer targeted prunes with filters, and never delete volumes without checking their contents.
