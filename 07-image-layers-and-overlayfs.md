# 07 · Image Layers, Manifests and the Overlay Filesystem

> **Level:** ★★ intermediate
>
> **You will learn:** what a layer really is (a tar file of changes), how an image is described by an **index → manifest → config + layers**, how the Linux **overlay filesystem** stacks layers into one folder, how **copy-on-write** and **whiteouts** work, how to see all of this on disk, and the two storage backends you'll meet in 2026 (the classic `overlay2` driver and the containerd snapshotter).
>
> **You need:** topic 02 (the container's `/` is a folder), topic 06 (manifests, digests).

---

## What it is

An image is not one big file. It's a **stack of layers**:

- Each **layer** is a set of filesystem **changes**: files added, changed or deleted compared to the layer below. It's stored as a `.tar` archive (usually gzip- or zstd-compressed).
- Layers are **read-only** and **shared**: ten images built on `debian:bookworm-slim` store that base layer once.
- When you run a container, Docker stacks the image's read-only layers and puts one thin **writable layer** on top. The Linux **overlay filesystem** makes the whole stack look like one normal folder, and that folder becomes the container's `/`.

```
      container sees ONE folder:  /  (bin, etc, usr, app, ...)
                    ▲
           overlay "merged" view
                    │
   ┌────────────────────────────────────┐
   │ writable layer (container)   RW    │  ← "upper": your changes at runtime
   ├────────────────────────────────────┤
   │ layer 3: COPY app.py /app    RO    │  ┐
   │ layer 2: RUN pip install ... RO    │  ├ "lower": the image, shared, never modified
   │ layer 1: debian base files   RO    │  ┘
   └────────────────────────────────────┘
```

---

## Why it exists

Without layers:
- Every image would carry a full copy of the OS. 50 images × 100 MB base = 5 GB of identical files.
- Every pull would download everything again.
- Every container would need its own full copy of the image's files before it could start (slow, wasteful).

With layers:
- **Shared storage**: a layer used by many images is stored once.
- **Faster pulls and pushes**: only missing layers move over the network.
- **Faster builds**: unchanged layers are reused from cache (topic 16).
- **Instant container start**: no copying. The container just gets an empty writable layer on top of the shared read-only ones.

---

## Analogy

**Transparent sheets on an overhead projector.**
- The bottom sheet has the base drawing (the OS files).
- The next sheet adds some details (installed packages).
- The top sheet adds your app.
- Looking down through all of them, you see **one** picture.
- The container gets one extra **blank** sheet on top where it can draw. To "change" something on a lower sheet, it draws over it on its own sheet (copy-on-write). To "erase" something, it paints a white patch on its own sheet (a whiteout). The lower sheets are never touched, so other containers sharing them see the original.

---

## How it works inside

### Part 1: The image's structure on the registry (OCI Image Spec)

Three kinds of JSON documents plus the layer blobs. Everything is named by its SHA-256 digest (**content addressing** = the name of a thing *is* the hash of its content, so the name proves the content).

```
image index (optional, for multi-platform)       sha256:6564…  ← the tag points here
   └─ manifest for linux/amd64                    sha256:4a2c…
        ├─ config                                 sha256:1f9e…
        └─ layers (in order, bottom first)
             ├─ sha256:f182…  (tar.gz, 3.6 MB)
             ├─ sha256:61ca…  (tar.gz, 1.8 MB)
             └─ ...
```

**Manifest** (simplified):

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:1f9e…",
    "size": 8952
  },
  "layers": [
    { "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip", "digest": "sha256:f182…", "size": 3642247 },
    { "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip", "digest": "sha256:61ca…", "size": 1806172 }
  ]
}
```

- `config.digest` = which config file belongs to this image.
- `layers` = the ordered list of compressed layer tarballs, with their digests and sizes.

**Config** (simplified):

```json
{
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "NGINX_VERSION=1.27.5"],
    "Cmd": ["nginx", "-g", "daemon off;"],
    "ExposedPorts": { "80/tcp": {} },
    "StopSignal": "SIGQUIT"
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": ["sha256:08000c…", "sha256:a2d6…"]
  },
  "history": [
    { "created_by": "ADD alpine-minirootfs-3.21.3-x86_64.tar.gz / # buildkit" },
    { "created_by": "CMD [\"nginx\" \"-g\" \"daemon off;\"]", "empty_layer": true }
  ]
}
```

- `config` = the runtime defaults: environment variables, command, ports, stop signal.
- `rootfs.diff_ids` = the SHA-256 of each layer **uncompressed**. (The manifest has the hashes of the **compressed** blobs. Two different numbers for the same layer. That's why you sometimes see different hashes for "the same" layer in different tools.)
- `history` = how each layer was made. `empty_layer: true` = a step that only changed config (like `CMD`), and so made no filesystem layer.

**Image index** = a list of manifests, one per platform (topic 06).

### Part 2: The overlay filesystem (Linux recap)

**OverlayFS** is a Linux kernel filesystem that **merges folders**. You give it:

| Part | What it is |
|---|---|
| `lowerdir` | One or more **read-only** folders (stacked; the leftmost is on top) |
| `upperdir` | One **writable** folder where all changes go |
| `workdir` | An empty helper folder on the same filesystem as upperdir (overlay uses it internally for safe, atomic changes) |
| mount point (`merged`) | Where the combined view appears |

Rules when you look at the merged folder:
- **Reading a file:** overlay looks in upper first, then each lower from top to bottom. The first one found wins.
- **Creating a file:** it's created in upper.
- **Changing a file that exists only in a lower layer:** overlay first **copies the whole file up** into upper, then changes the copy. This is **copy-on-write (CoW)**, called "copy-up". The lower file is untouched.
- **Deleting a file that exists in a lower layer:** overlay can't delete from a read-only folder. It creates a **whiteout** in upper: a special **character device file with device number 0/0** and the same name. The merged view then hides the file.
- **Deleting a whole folder and recreating it:** overlay marks the new folder in upper as **opaque** (an extended attribute `trusted.overlay.opaque=y`), meaning "ignore everything below with this name".

In image layers (tar files), whiteouts are stored as files named `.wh.<name>`, and opaque folders as `.wh..wh..opq`. When a layer is unpacked onto overlay, these become real whiteouts.

### Part 3: How Docker uses overlay

For a running container:
- `lowerdir` = all the image's layers (unpacked), read-only.
- `upperdir` = the container's private **writable layer**.
- `merged` = the container's root filesystem. runc `pivot_root`s into it (topic 02).

Consequences you must know:

1. **Starting a container is instant.** No files are copied. Just one overlay mount.
2. **Deleting a container deletes its upperdir.** Every change it made is gone (topic 23).
3. **Deleting a file in a later layer doesn't make the image smaller.** The file is still in the lower layer's tar. The upper layer just hides it. `RUN rm` in a separate step is useless for size (topic 20).
4. **Copy-up costs time and space.** Changing one byte of a 2 GB file in a lower layer copies the whole 2 GB into upper first. That's why databases must keep their data in **volumes**, not the writable layer (topic 24).
5. **Many layers slow lookups a bit.** Each file lookup can check many folders. Keep layer counts reasonable.
6. **Images can't change.** Containers sharing an image all see the same lower layers. One container's changes go to its own upper only.

### Part 4: The two storage backends in 2026

Docker has two ways to store and mount layers. You'll meet both:

| | `overlay2` (graph driver) | `overlayfs` (containerd snapshotter) |
|---|---|---|
| Who manages layers | dockerd | containerd |
| Default for | Installs from before Engine 29 (and upgraded hosts) | **Fresh installs of Engine 29+**, and Docker Desktop |
| Where layers live | `/var/lib/docker/overlay2/<id>/diff` | `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<n>/fs` |
| `docker info` shows | `Storage Driver: overlay2` | `Storage Driver: overlayfs`, `driver-type: io.containerd.snapshotter.v1` |
| Extra abilities | none | stores multi-platform images and attestations locally, same image store as Kubernetes uses |

Both use **the same kernel OverlayFS** underneath. Only the bookkeeping and folder paths differ. The method that always works for both: **look at the actual overlay mount** with `findmnt` or `/proc/<pid>/mountinfo`.

> A **snapshotter** is containerd's name for the component that prepares layer folders ("snapshots") and mounts them. `overlayfs` is the default snapshotter.

> Other storage drivers you may see in old docs: `aufs` and `devicemapper` (**removed**), `btrfs`, `zfs`, `fuse-overlayfs` (for rootless setups on old kernels), `vfs` (no sharing, very slow; only for testing). Use overlay.

---

## Commands / files

### `docker history`: how the image was built, layer by layer

```bash
docker history nginx:1.27-alpine
```

```
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
65645c7bb6a0   3 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   COPY 30-tune-worker-processes.sh /docker-ent…   16.4kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -x     && addgroup -g 101…   5.6MB     buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV NGINX_VERSION=1.27.5                        0B        buildkit.dockerfile.v0
<missing>      4 weeks ago   CMD ["/bin/sh"]                                 0B        buildkit.dockerfile.v0
<missing>      4 weeks ago   ADD alpine-minirootfs-3.21.3-x86_64.tar.gz /…   8.5MB     buildkit.dockerfile.v0
```

- Read it **bottom to top**: the bottom row is the first step (the Alpine base files).
- `SIZE 0B` rows = config-only steps (`CMD`, `ENV`, `EXPOSE`). They don't create a filesystem layer.
- Rows with a size = real layers.
- `<missing>` = no local image ID exists for that intermediate step (normal for pulled images). It's not an error.
- `--no-trunc` shows full commands. This is a quick way to see "what's in this image" when you don't have its Dockerfile.

### See the layer digests

```bash
docker image inspect nginx:1.27-alpine --format '{{json .RootFS.Layers}}' | tr ',' '\n'
```

```
["sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350"
"sha256:a2d62fee2a7c3cda9fa7b5d6e1ab9d6c41da76f3f8b0ae8e9e1fa3c1b1a5f1d0"
...]
```

- These are the **diff_ids** (uncompressed layer hashes) from the config.

### The raw manifest from the registry

```bash
docker buildx imagetools inspect nginx:1.27-alpine --raw | head -20
```

Prints the index JSON. To get one platform's manifest:

```bash
docker buildx imagetools inspect --raw nginx:1.27-alpine@sha256:<amd64-manifest-digest>
```

### Find a container's overlay mount (works for both backends)

```bash
docker run -d --name ovl nginx:1.27-alpine
PID=$(docker inspect -f '{{.State.Pid}}' ovl)
sudo grep ' / ' /proc/$PID/mountinfo
```

- `/proc/<PID>/mountinfo` = the mounts as seen **by that process**, i.e. inside its mount namespace. The line with mount point ` / ` is its root filesystem.

Example output with **overlay2** (wrapped for reading):

```
1203 1050 0:62 / / rw,relatime master:443 - overlay overlay rw,
  lowerdir=/var/lib/docker/overlay2/l/QWCB5YVK7SDJ3:/var/lib/docker/overlay2/l/7Z2PQHGM4K5LT:/var/lib/docker/overlay2/l/DJV6VK2DXZ4WE,
  upperdir=/var/lib/docker/overlay2/9c3e1f…/diff,
  workdir=/var/lib/docker/overlay2/9c3e1f…/work
```

Example output with the **containerd snapshotter**:

```
1203 1050 0:62 / / rw,relatime - overlay overlay rw,
  lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/31/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/30/fs:...,
  upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/40/fs,
  workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/40/work
```

How to read it:
- `- overlay overlay` = the filesystem type is overlay.
- `lowerdir=A:B:C` = the image layers, **top first**. In overlay2, the `l/XXXX` entries are short symlinks to layer folders (they keep the mount options under the kernel's page-size limit).
- `upperdir=…` = this container's writable layer. **Anything the container writes ends up here.**
- `workdir=…` = overlay's helper folder.

With **overlay2**, `docker inspect` also shows the folders directly:

```bash
docker inspect ovl --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
```

```json
{
    "LowerDir": "/var/lib/docker/overlay2/9c3e1f…-init/diff:/var/lib/docker/overlay2/5ad2…/diff:...",
    "MergedDir": "/var/lib/docker/overlay2/9c3e1f…/merged",
    "UpperDir": "/var/lib/docker/overlay2/9c3e1f…/diff",
    "WorkDir": "/var/lib/docker/overlay2/9c3e1f…/work"
}
```

- `…-init` = a small layer Docker adds under the container's writable layer, holding files Docker manages for every container (`/etc/hosts`, `/etc/resolv.conf`, `/etc/hostname` placeholders).

With the containerd snapshotter, `GraphDriver.Data` may be empty. Use the `mountinfo` method above.

### Watch copy-on-write happen

```bash
docker exec ovl sh -c 'echo "# changed" >> /etc/nginx/nginx.conf; touch /newfile; rm /etc/motd'
docker diff ovl
```

```
C /etc
C /etc/nginx
C /etc/nginx/nginx.conf
D /etc/motd
A /newfile
C /run
A /run/nginx.pid
C /var/cache/nginx
A /var/cache/nginx/client_temp
...
```

- `docker diff` lists the writable layer's contents compared to the image.
- `A` = added, `C` = changed, `D` = deleted (a whiteout exists for it).
- `nginx.conf` was **copied up** and changed. `/etc/motd` was **whited out**.

Look at the upper folder directly from the host (replace with your `upperdir` path):

```bash
UPPER=$(sudo grep ' / ' /proc/$PID/mountinfo | sed 's/.*upperdir=\([^,]*\).*/\1/')
sudo ls -la $UPPER/etc
```

```
total 16
drwxr-xr-x 3 root root 4096 Nov 17 11:02 .
drwxr-xr-x 7 root root 4096 Nov 17 11:02 ..
c--------- 2 root root 0, 0 Nov 17 11:02 motd
drwxr-xr-x 2 root root 4096 Nov 17 11:02 nginx
```

- `c--------- ... 0, 0 motd` = **the whiteout**: a character device (`c`) with device number `0, 0`. That's how "deleted" is recorded.
- `nginx/` here contains only the copied-up `nginx.conf`, not the rest of `/etc/nginx`.

---

## Hands-on lab

### Lab A: overlay by hand (no Docker)

```bash
mkdir -p ~/ovl/{lower1,lower2,upper,work,merged}
echo "from lower1" > ~/ovl/lower1/a.txt
echo "base config" > ~/ovl/lower1/config.txt
echo "from lower2 (wins over lower1)" > ~/ovl/lower2/a.txt
echo "only in lower2" > ~/ovl/lower2/b.txt

cd ~/ovl
sudo mount -t overlay overlay \
  -o lowerdir=lower2:lower1,upperdir=upper,workdir=work merged
```

- `lowerdir=lower2:lower1` = `lower2` is stacked **above** `lower1`.

```bash
ls merged; cat merged/a.txt
```

```
a.txt  b.txt  config.txt
from lower2 (wins over lower1)
```

Now write, change and delete:

```bash
echo "new" | sudo tee merged/c.txt > /dev/null
echo "edited" | sudo tee -a merged/config.txt > /dev/null
sudo rm merged/b.txt
ls -la upper
cat lower1/config.txt
ls lower2
```

```
total 16
drwxr-xr-x 2 root root 4096 . 
drwxr-xr-x 7 ...       4096 ..
c--------- 2 root root 0, 0 b.txt
-rw-r--r-- 1 root root    4 c.txt
-rw-r--r-- 1 root root   19 config.txt
base config
a.txt  b.txt
```

What you just proved:
- `c.txt` (new) → created in **upper**.
- `config.txt` (changed) → **copied up** to upper and edited. `lower1/config.txt` is still `base config`.
- `b.txt` (deleted) → a **whiteout** `c 0,0` in upper. `lower2/b.txt` still exists.

This is exactly a container's writable layer. Clean up:

```bash
sudo umount ~/ovl/merged && rm -rf ~/ovl
```

### Lab B: see layer sharing

```bash
docker pull python:3.13-slim
docker pull python:3.13-slim-bookworm 2>/dev/null || true
docker system df -v | sed -n '1,12p'
```

Look at the `SHARED SIZE` column: the size of layers shared with other images.

### Lab C: prove `rm` in a later layer doesn't shrink an image (preview of topic 14)

```bash
mkdir -p ~/layerlab && cd ~/layerlab
cat > Dockerfile <<'EOF'
FROM alpine:3.22
RUN head -c 50000000 /dev/urandom > /big.bin
RUN rm /big.bin
EOF
docker build -t layerlab .
docker history layerlab
```

```
IMAGE          CREATED          CREATED BY                                      SIZE
a7c1…          5 seconds ago    RUN /bin/sh -c rm /big.bin # buildkit            0B
<missing>      6 seconds ago    RUN /bin/sh -c head -c 50000000 /dev/urandom…   50MB
<missing>      4 weeks ago      CMD ["/bin/sh"]                                 0B
<missing>      4 weeks ago      ADD alpine-minirootfs-…                         8.5MB
```

The 50 MB layer is still in the image. The `rm` layer only added a whiteout. (A `Dockerfile` is the recipe file for building an image, and `docker build` runs it. Topic 14 teaches both properly.)

Clean up: `docker rm -f ovl; docker rmi layerlab; rm -rf ~/layerlab`

---

## Where you'll use it

- **Image size reviews:** `docker history` shows which step added the bulk.
- **Security:** a secret copied in one layer and deleted in the next **is still in the image**. Anyone who pulls it can extract the earlier layer (topics 18, 44).
- **Performance problems:** apps writing heavily to the container filesystem (logs, caches, database files) cause copy-up overhead and fill the disk. Use volumes.
- **Disk-full incidents:** knowing where layers and writable layers live (`/var/lib/docker`, `/var/lib/containerd`) tells you where the space went (topic 48).
- **Forensics:** `docker diff` and the upperdir show exactly what an attacker or a buggy process changed inside a container.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `failed to register layer: ... no space left on device` while pulling | The disk holding layers is full (or out of inodes) | `df -h /var/lib/docker /var/lib/containerd`, `df -i`; prune (topic 13, 48) |
| Image is huge even though you `rm` files | Files deleted in a later layer still exist in an earlier one | Delete in the same `RUN` step that created them, or use multi-stage builds (topic 17) |
| Writes inside the container are slow, the disk fills, `docker diff` shows huge files | App writes big or frequently-changing files to the writable layer (copy-up) | Put data on a volume or tmpfs (topics 24–25) |
| `overlay: filesystem on '/var/lib/docker/...' not supported as upperdir` / `driver not supported` | The backing filesystem doesn't support overlay as an upper layer (e.g. some NFS mounts, or an XFS formatted with `ftype=0`) | Put Docker's data on ext4 or XFS with `ftype=1` (`xfs_info <mount> \| grep ftype`) |
| `rename ... : invalid cross-device link` inside a container | Some programs `rename()` folders from lower layers; overlay can't always rename lower folders | Usually an app/kernel combo issue. Copy instead of rename, or keep that data on a volume |
| Different hashes for "the same" layer in different tools | One shows the compressed digest (manifest), the other the uncompressed `diff_id` (config) | Both are right. They hash different bytes |

---

## Quick recap

- An image = **index → manifest → config + ordered layers**. Everything is addressed by its SHA-256 digest.
- A layer is a tar of changes. Deletions are stored as **whiteouts**.
- OverlayFS merges read-only **lower** layers with a writable **upper** layer into one **merged** view. That view is the container's `/`.
- Changing a lower file triggers **copy-up** of the whole file. Deleting creates a whiteout. Lower layers never change.
- Engine 29 fresh installs use the **containerd overlayfs snapshotter** (`/var/lib/containerd/...`); older ones use **overlay2** (`/var/lib/docker/overlay2`). `/proc/<pid>/mountinfo` shows the real mount either way.

**Interview questions**

1. **Q: How does Docker start a container so fast if the image is hundreds of MB?**
   A: It copies nothing. It mounts an overlay filesystem with the image's read-only layers as lowerdirs and a new empty writable upperdir. The merged view becomes the container's root.

2. **Q: You delete a 500 MB file in a Dockerfile step after the step that created it. Why is the image still big?**
   A: Layers are immutable. The file stays in the earlier layer, and the later layer only adds a whiteout that hides it. Delete it in the same `RUN` step, or use a multi-stage build.

3. **Q: Why shouldn't a database write to the container's filesystem?**
   A: Writes go to the writable layer: copy-up overhead, overlay performance limits, and the data is lost when the container is removed. Use a volume, which bypasses overlay and outlives the container.
