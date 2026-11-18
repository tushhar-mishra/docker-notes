# 48 · Storage Internals and Disk Problems

> **Level:** ★★★ advanced
>
> **You will learn:** the layout of `/var/lib/docker` and `/var/lib/containerd` (what every folder holds), why the containerd image store uses more disk, a step-by-step method for "**no space left on device**" (including inode exhaustion and deleted-but-open files), BuildKit garbage collection settings, why you must never delete files inside `overlay2` by hand, and how to **move Docker's data** to a bigger disk safely.
>
> **You need:** topic 07 (layers, snapshotters), topic 13 (prune), topic 33 (logs), topic 34 (`daemon.json`).

---

## What it is

Everything Docker stores lives in two trees:

```
/var/lib/docker/                      ← dockerd's data-root
├── containers/<id>/                  ← per-container config, hostname/hosts/resolv.conf, LOGS (json-file/local)
├── volumes/<name>/_data              ← named & anonymous volumes (local driver)
├── overlay2/<id>/{diff,work,merged,link,lower}   ← image layers + writable layers (overlay2 graph driver only)
│   └── l/                            ← short symlinks used in mount options
├── image/overlay2/                   ← image & layer metadata for overlay2 (imagedb, layerdb)
├── buildkit/                         ← BuildKit cache metadata and content
├── network/files/local-kv.db         ← network configuration database
├── rootfs/overlayfs/<id>/            ← container root mount points (containerd snapshotter)
├── tmp/  plugins/  runtimes/  swarm/  trust/
└── engine-id

/var/lib/containerd/                  ← containerd's root
├── io.containerd.content.v1.content/blobs/sha256/<digest>   ← COMPRESSED blobs (layers, manifests, configs)
├── io.containerd.snapshotter.v1.overlayfs/snapshots/<n>/fs  ← UNPACKED layers + writable layers
├── io.containerd.metadata.v1.bolt/meta.db                   ← containerd's database (namespaces, images, leases)
└── io.containerd.runtime.v2.task/...                        ← running task state
```

---

## Why it exists

"Disk full on the Docker host" is the most common Docker production incident (topic 13). When it happens, you need to know **in minutes** which of these folders grew and **what's safe** to delete. Deleting the wrong thing (a volume, a layer that's still in use) turns a disk problem into data loss or a corrupted Docker installation.

---

## Analogy

A **warehouse** with labelled zones: incoming shipments still in boxes (compressed blobs), unpacked stock on shelves (snapshots/overlay2), customers' private lockers (volumes), daily receipts (logs), and the prep kitchen (build cache). When the warehouse is full, you don't start throwing random boxes off random shelves. You check each zone's size, ask the inventory system what's unused, and let **it** do the clearing (`prune`). Throwing things away by hand corrupts the inventory records.

---

## How it works inside

### Classic `overlay2` storage

- Each image layer is a folder `overlay2/<id>/diff` holding that layer's unpacked files.
- Each container gets `overlay2/<id>/diff` (its writable layer), plus `<id>-init` (topic 07).
- Metadata (which layers make which image, layer chain IDs, sizes) lives in `image/overlay2/`.
- Pulled layers are stored **once, unpacked**. The compressed download is discarded after unpacking.

### containerd image store (Engine 29 default)

- containerd keeps the **compressed blobs** in its content store (needed to push images again without recompressing, and for multi-platform/attestation support) **and** the **unpacked snapshots**.
- So the same image can take roughly **compressed + unpacked** size on disk: noticeably more than with overlay2. Plan disk sizes accordingly, and use `docker system df` to see real numbers.
- Garbage collection: when you remove images, containerd's GC deletes unreferenced blobs and snapshots (protected by "leases" while operations are in progress).

### The usual space hogs (in order of frequency)

1. **Container logs** (`containers/*/*-json.log`) without rotation.
2. **Build cache** on CI/build hosts (`buildkit/` and cache content).
3. **Old images** (every deploy pulls a new tag; old ones stay).
4. **Writable layers** of containers writing data/logs/caches into their own filesystem (`docker ps -s`).
5. **Volumes** (real data, or orphaned anonymous volumes).
6. **Stopped containers** (their writable layers and logs).

### "No space left on device" can mean three different things

| Cause | Check | Example |
|---|---|---|
| **Blocks** full (bytes) | `df -h` | Logs, images |
| **Inodes** full (number of files) | `df -i` | Millions of tiny files: `node_modules` layers, cache folders, mail queues. `df -h` still shows free space! |
| **Space held by deleted-but-open files** | `lsof +L1` / `df` vs `du` mismatch | A log file `rm`ed while dockerd still has it open (topic 13) |

(XFS inodes are allocated dynamically, so inode exhaustion is mostly an **ext4** problem. ext4 fixes the inode count when the filesystem is created.)

### BuildKit garbage collection

BuildKit cleans its cache automatically based on policies. Configure it in `daemon.json`:

```json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB",
      "policy": [
        { "keepStorage": "10GB", "filter": ["unused-for=168h"] },
        { "keepStorage": "50GB", "all": true }
      ]
    }
  }
}
```

- `defaultKeepStorage` = target maximum size for the build cache.
- `policy` rules are applied in order: e.g. "anything unused for 7 days may be removed until ≤ 10 GB", then "everything, until ≤ 50 GB".
- Newer BuildKit versions also accept `reservedSpace`, `maxUsedSpace` and `minFreeSpace` (keep at least N GB of the disk free). Check your Engine's docs for the exact keys.
- `docker-container` builders have their own config (`buildkitd.toml`).

### Never delete inside `overlay2` (or containerd's folders) by hand

`rm -rf /var/lib/docker/overlay2/<something>` "to free space" leaves Docker's metadata pointing at missing folders. The results are broken images (`failed to register layer`), containers that won't start, and `docker rmi` errors. Always delete through Docker (`rm`, `rmi`, `prune`). The only files safe to touch directly are **container log files** (`truncate`, not `rm`).

If Docker's storage is truly corrupted beyond repair (rare), the clean recovery is: stop Docker, move the whole data-root aside, start Docker fresh, re-pull images and restore volumes from backup.

### Putting Docker's data on its own disk

Best practice for servers: a **dedicated filesystem** for `/var/lib/docker` (and `/var/lib/containerd`):
- a full Docker disk then can't fill the root filesystem (which would break SSH logins, journald, package managers),
- it can be sized and monitored separately,
- use **ext4** or **XFS with `ftype=1`** (required for overlay; topic 07).

Two ways:
1. Mount the new disk **at** `/var/lib/docker` (and/or `/var/lib/containerd`). No Docker config changes needed.
2. Mount it elsewhere (`/data`) and set `"data-root": "/data/docker"` in `daemon.json`, and containerd's `root = "/data/containerd"` in `/etc/containerd/config.toml`.

---

## Commands / files

### The disk-full method

**Step 1: which filesystem, blocks or inodes?**

```bash
df -h /var/lib/docker /var/lib/containerd /
df -i /var/lib/docker /
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       200G  196G  4.0G  99% /
Filesystem       Inodes   IUsed  IFree IUse% Mounted on
/dev/sdb1      13107200 2104112 11003088   17% /
```

- Blocks are the problem here (99%), not inodes.

**Step 2: which top-level folder?**

```bash
sudo du -xh --max-depth=1 /var/lib/docker /var/lib/containerd 2>/dev/null | sort -h | tail -12
```

```
1.2G    /var/lib/docker/volumes
3.4G    /var/lib/docker/buildkit
71G     /var/lib/docker/containers
88G     /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs
21G     /var/lib/containerd/io.containerd.content.v1.content
```

- `-x` = stay on this filesystem (don't count mounts inside it).
- `containers` = 71 GB → almost certainly **logs**.

**Step 3: drill into the big one.**

```bash
sudo find /var/lib/docker/containers -name '*-json.log' -size +1G -exec ls -lh {} \; | sort -k5 -h | tail -3
docker ps -a --no-trunc --format '{{.ID}} {{.Names}}' | grep 5e6f7a
```

```
-rw-r----- 1 root root 64G Nov 17 17:40 /var/lib/docker/containers/5e6f7a…/5e6f7a…-json.log
5e6f7a… chatty-worker
```

**Step 4: Docker's own view.**

```bash
docker system df
docker system df -v | sed -n '/Containers space usage/,/Local Volumes/p' | sort -k5 -h | tail -5
```

**Step 5: deleted-but-open files.**

```bash
sudo lsof +L1 2>/dev/null | grep -E 'dockerd|containerd' | head
```

```
COMMAND   PID USER   FD   TYPE DEVICE   SIZE/OFF NLINK  NODE NAME
dockerd  1290 root   45w   REG   8,17 21474836480     0 39321 /var/lib/docker/containers/9a8b…/9a8b…-json.log (deleted)
```

- 20 GB held by a log file someone deleted with `rm`. The space returns when dockerd closes it (restart the container), or truncate it now through `/proc`: `sudo truncate -s 0 /proc/1290/fd/45`.

**Step 6: fix (in order of safety).**

```bash
sudo truncate -s 0 /var/lib/docker/containers/5e6f7a…/5e6f7a…-json.log   # immediate relief
docker builder prune -f --filter until=72h                                # build cache
docker container prune -f --filter until=24h                              # dead containers
docker image prune -af --filter until=168h                                # old unused images
docker volume ls -f dangling=true                                         # LOOK before deleting volumes
```

Then the **permanent** fixes: log rotation defaults + recreate containers (topic 33), BuildKit GC limits, scheduled pruning, a separate Docker disk, monitoring and alerts (topic 47).

### Inode exhaustion example

```bash
df -i /
```

```
Filesystem      Inodes   IUsed IFree IUse% Mounted on
/dev/sda1      6553600 6553600     0  100% /
```

```bash
sudo du --inodes -x --max-depth=2 /var/lib/docker 2>/dev/null | sort -n | tail -5
```

```
412000   /var/lib/docker/volumes
3910000  /var/lib/docker/overlay2
```

- Millions of small files in image layers (many images with big `node_modules`). Fix: prune images, and use slimmer images and multi-stage builds. Longer term: XFS or ext4 created with more inodes (`mkfs.ext4 -i 8192`).

### Move data-root to a new disk

```bash
# 0. Prepare: new disk formatted (ext4 or XFS ftype=1) and mounted at /data
xfs_info /data 2>/dev/null | grep ftype          # if XFS: must show ftype=1

# 1. Stop everything that uses the data
sudo systemctl stop docker docker.socket containerd

# 2. Copy, preserving everything (permissions, owners, hard links, ACLs, xattrs, sparse files)
sudo rsync -aHAXS --numeric-ids /var/lib/docker/ /data/docker/
sudo rsync -aHAXS --numeric-ids /var/lib/containerd/ /data/containerd/

# 3. Point Docker and containerd to the new places
#    /etc/docker/daemon.json:          "data-root": "/data/docker"
#    /etc/containerd/config.toml:      root = "/data/containerd"
#    (if config.toml doesn't exist:    containerd config default | sudo tee /etc/containerd/config.toml, then edit root)
#    Careful: Docker's containerd.io package may ship a config.toml with some plugins disabled.
#    Edit only the root line rather than replacing the file.

# 4. Start and verify
sudo systemctl start containerd docker
docker info --format '{{.DockerRootDir}}'
docker images; docker ps -a; docker volume ls

# 5. After a few days of normal operation, delete the old folders
sudo mv /var/lib/docker /var/lib/docker.old    # then later: sudo rm -rf /var/lib/docker.old
```

- `-H` = keep hard links. `-A`/`-X` = ACLs and extended attributes (overlay uses xattrs, e.g. opaque directories, topic 07). `-S` = sparse files. `--numeric-ids` = keep UIDs as numbers (important for volumes and userns setups).
- The simpler alternative: stop services, copy the same way to the new disk, then **mount the new disk at `/var/lib/docker`** (and `/var/lib/containerd`) via `/etc/fstab`. No config changes.

### Storage-related daemon checks

```bash
docker info --format 'Driver={{.Driver}} Root={{.DockerRootDir}}'
docker info | sed -n '/Storage Driver/,/Logging Driver/p'
findmnt -T /var/lib/docker
```

```
Driver=overlayfs Root=/var/lib/docker
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: local
TARGET          SOURCE    FSTYPE OPTIONS
/var/lib/docker /dev/sdc1 xfs    rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

---

## Hands-on lab

1. Run the disk-full method on your lab machine (even if it isn't full) and write down the size of each folder.
2. Create a log hog: `docker run -d --name hog alpine sh -c 'while :; do echo "spam spam spam spam spam spam"; done'`. After 30 seconds, find it with Steps 2–3, then truncate the log. Recreate it with `--log-opt max-size=10m --log-opt max-file=2` and confirm it stays bounded.
3. **Deleted-but-open:** with `hog` running (no rotation), `sudo rm` its log file, check `df` (space not freed) and `lsof +L1`, then truncate through `/proc/<pid>/fd/<n>`, or restart the container.
4. Set BuildKit GC (`defaultKeepStorage: "2GB"`), build a few images, and check `docker buildx du` stays near the limit.
5. **Optional:** add a second virtual disk to a Linux VM, format it, and move data-root with the procedure above.
6. Clean up: `docker rm -f hog`

---

## Where you'll use it

- **Incident response:** disk-full on Docker hosts and CI runners. This method finds the cause in minutes.
- **Host design:** a separate filesystem for Docker data, filesystem choice (XFS ftype=1/ext4), sizing for the containerd store.
- **CI fleets:** BuildKit GC policies and scheduled pruning.
- **Migrations:** moving data-root, changing disks, switching image stores (topic 49).
- **Kubernetes nodes:** the same folders (`/var/lib/containerd`, `/var/log/pods`), with kubelet's image garbage collection and eviction on disk pressure (topic 53).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `no space left on device` but `df -h` shows free space | Inodes exhausted | `df -i`; prune; slimmer images; filesystem with more inodes |
| `df` says full, `du` finds much less | Deleted-but-open files | `lsof +L1`; truncate via `/proc/PID/fd`; restart the holder |
| `failed to register layer: ... no such file or directory` | Layer store corrupted (often after manual deletion in overlay2) | Remove the affected image(s) and re-pull; worst case a fresh data-root |
| `error creating overlay mount to ...: invalid argument` after moving data-root | Target filesystem unsupported (XFS `ftype=0`, NFS, or a missing xattr copy) | Use ext4/XFS ftype=1; copy with `rsync -aHAXS` |
| Docker starts but shows no images/containers after moving | data-root/containerd root not updated, or a copy path mistake (`/data/docker/docker`) | Check `docker info` Root Dir and the folder contents |
| Disk usage much higher after upgrading to Engine 29 / enabling containerd store | Compressed blobs + unpacked snapshots are both stored | Expected. Size disks for it; prune; set GC |
| `/` filling breaks SSH logins and journald | Docker data on the root filesystem | Separate filesystem for Docker data |
| Build cache keeps returning to tens of GB | No GC limit | `builder.gc` in daemon.json; scheduled `builder prune` |

---

**Next:** topic 48a tunes the rest of the host: kernel limits that cause "random" failures on busy container hosts.

## Quick recap

- `/var/lib/docker`: containers (incl. **logs**), volumes, overlay2 layers (classic), buildkit, network. `/var/lib/containerd`: **compressed blobs** + **unpacked snapshots** + metadata (the Engine 29 image store uses more disk).
- Disk-full method: `df -h` / `df -i` → `du -x` by folder → `docker system df -v` → logs → `lsof +L1` → fix safely → fix permanently.
- Three meanings of "no space": blocks, **inodes**, **deleted-but-open** files.
- **Never delete inside overlay2/containerd folders by hand.** Use prune/rm/rmi. Truncate logs, don't `rm` them.
- Use a dedicated disk (ext4 / XFS `ftype=1`). Move with services stopped and `rsync -aHAXS --numeric-ids`, then update `data-root` and containerd's `root`.

**Interview questions**

1. **Q: A Docker host reports "no space left on device", but `df -h` shows 30% free. What's going on?**
   A: Probably inode exhaustion: `df -i` shows 100% inode use from many small files in image layers or volumes. It could also be a different filesystem than you checked. Prune images/containers and consider filesystem/inode sizing.

2. **Q: How do you safely move Docker's data to a bigger disk?**
   A: Stop docker, docker.socket and containerd. Copy `/var/lib/docker` and `/var/lib/containerd` with `rsync -aHAXS --numeric-ids` to a filesystem that supports overlay (ext4 or XFS ftype=1). Then mount it at the original path, or set `data-root` and containerd's `root`, start the services, verify, and only later delete the old copy.

3. **Q: Why shouldn't you delete folders under `/var/lib/docker/overlay2` to free space?**
   A: Docker's metadata references those layer folders. Removing them by hand corrupts images and containers (missing layers, failing mounts). Free space with `docker image/container/builder/system prune`, `docker rm`/`rmi`, and log truncation.
