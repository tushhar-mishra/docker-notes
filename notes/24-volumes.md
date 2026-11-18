# 24 · Volumes

> **Level:** ★★ intermediate
>
> **You will learn:** what a Docker volume really is on disk, named vs anonymous volumes, the `-v` and `--mount` syntaxes, the `VOLUME` instruction and the "orphan anonymous volume" trap, how Docker fills a new volume from the image, sharing volumes between containers, backing up and restoring volumes, and volume drivers (NFS, CIFS and plugins).
>
> **You need:** topic 23 (why data disappears; the three mount types).

---

## What it is

A **volume** is a folder **managed by Docker** that lives outside any container and is mounted into containers at a path you choose.

With the default `local` driver, a volume is literally a directory on the host:

```
/var/lib/docker/volumes/
└── pgdata/
    └── _data/          ← this folder appears inside the container at the mount point
        ├── PG_VERSION
        ├── base/
        └── ...
```

Two kinds:
- **Named volume**: you pick the name (`pgdata`). Easy to find, reuse and back up.
- **Anonymous volume**: Docker picks a random 64-character name. Created when a path needs a volume but you didn't name one.

---

## Why it exists

Compared to bind mounts (topic 25), volumes:
- don't depend on a host's folder layout (`/home/tushar/...` exists on your laptop, not on the server),
- are created and permission-initialised by Docker,
- are **pre-filled** from the image on first use (very handy),
- can live on remote storage through **drivers** (NFS, cloud disks) with the same `-v name:/path` syntax,
- are listed, inspected, backed up and pruned with `docker volume` commands,
- work the same on Docker Desktop (where host paths are awkward because Docker runs in a VM).

That's why volumes are **the recommended default for persistent data**.

---

## Analogy

A **storage locker** in a building (Docker manages the building). You rent locker `pgdata`. Any tenant (container) you allow can open it. When a tenant moves out (`docker rm`), the locker and its contents stay. A **named** locker has your name on it. An **anonymous** one has just a serial number: easy to forget, and building management may clear it out (`prune`).

---

## How it works inside

### Mounting = a bind mount of the volume's folder

When a container starts with `-v pgdata:/var/lib/postgresql/data`:
1. dockerd makes sure the volume exists (creating `/var/lib/docker/volumes/pgdata/_data` if not).
2. In the container's **mount namespace**, runc **bind-mounts** `.../pgdata/_data` onto `/var/lib/postgresql/data` (a bind mount makes one folder appear at another path, topic 25).
3. That path is now covered by the volume, so reads and writes there bypass overlay entirely and go straight to the host filesystem at full speed.

So a "local volume" is just a **bind mount of a folder Docker owns**.

### Pre-filling a new volume from the image ("copy-up")

If you mount an **empty** volume at a path that **already has files in the image**, Docker **copies the image's files into the volume** first (once, when the volume is first used). This includes their ownership and permissions.

Example: the nginx image has files in `/usr/share/nginx/html`. `docker run -v html:/usr/share/nginx/html nginx` → the new volume `html` gets `index.html` and `50x.html` copied in.

- It happens **only if the volume is empty**. Next time, the volume's content wins, even if the image changed. (So a new image version **won't** update files in an existing volume.)
- **Bind mounts never do this**: they just cover the image's folder with whatever is in the host folder (even if it's empty).
- Turn it off with `--mount ...,volume-nocopy` (or `-v name:/path:nocopy`).

### The `VOLUME` instruction and anonymous volumes

```dockerfile
VOLUME /var/lib/postgresql/data
```

This tells Docker: "this path holds data; **always** mount a volume here". If you run the container **without** mounting anything there, Docker **creates an anonymous volume** for it automatically.

The trap (you saw it in topic 23's lab):
1. `docker run postgres` → anonymous volume `a1b2…` holds your data.
2. You recreate the container (upgrade) **without** `-v` → **another new** anonymous volume `c3d4…`. The data is still in `a1b2…`, but nothing uses it.
3. It looks like data loss. Orphan volumes pile up. `docker volume prune` deletes them for real.

Other side effects of `VOLUME`:
- Changes to that path in later Dockerfile `RUN` steps are **discarded** (with older builders; BuildKit handles it better, but don't rely on it).
- You can't "un-declare" a VOLUME from a base image in your own Dockerfile.

**Rule:** for any image that declares a `VOLUME` (check with `docker image inspect -f '{{json .Config.Volumes}}'`), always mount a **named** volume there.

### `-v` vs `--mount`

Both do the same job. `--mount` is more explicit and errors out instead of guessing.

```bash
# -v syntax: source:target[:options]
docker run -v pgdata:/var/lib/postgresql/data postgres:17-alpine
docker run -v pgdata:/var/lib/postgresql/data:ro ...         # read-only

# --mount syntax: key=value pairs
docker run --mount type=volume,source=pgdata,target=/var/lib/postgresql/data postgres:17-alpine
docker run --mount type=volume,src=pgdata,dst=/data,readonly ...
```

| | `-v` | `--mount` |
|---|---|---|
| Source starts with `/` or `.` | Treated as a **bind mount** | You must say `type=bind` |
| Source is a name | Volume | `type=volume` |
| Source doesn't exist | Volume: created. Bind path: **created as an empty root-owned folder** (often a surprise) | Volume: created. Bind path: **error** |
| Volume driver options | Not supported | `volume-driver=…,volume-opt=…` |
| Readability | Short | Clear, self-documenting |

Use `--mount` in scripts and production. `-v` is fine for quick commands. (Compose has its own clear syntax; topic 36.)

### Mounting only part of a volume: `volume-subpath`

```bash
docker run --mount type=volume,src=shared,dst=/etc/app,volume-subpath=app1 ...
```

- Mounts only the `app1` subfolder of the volume (Docker 26+). Useful when one volume holds data for several services.

### Sharing a volume between containers

Several containers can mount the same volume at the same time. Docker doesn't coordinate their writes: **the application must handle concurrent access** (most databases **must not** share their data folder with another running instance).

Good uses: one container writes files, another serves them (e.g. a build step writes static files, nginx serves them read-only); a sidecar reads logs.

`--volumes-from other` mounts **all** the volumes of another container at the same paths. It's old-style but still handy for backups.

### Volume drivers

The `local` driver can also mount **remote** storage, using the same options as the Linux `mount` command:

```bash
# NFS (Network File System) share
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.20,rw,nfsvers=4.1 \
  --opt device=:/exports/app-data \
  nfsdata

# CIFS/SMB (Windows file share)
docker volume create --driver local \
  --opt type=cifs \
  --opt device=//fileserver/share \
  --opt o=username=svc,password=…,uid=10001,gid=10001 \
  smbdata
```

- The remote mount happens **when a container first uses the volume**, not at create time. A wrong address shows up as an error on `docker run`.
- Plugins add other backends (cloud block storage, distributed storage), installed with `docker plugin install`. In Kubernetes, the **CSI** (Container Storage Interface) plays this role (topic 53).

### Where volumes live on Docker Desktop

Inside the hidden Linux VM, not on your Windows/Mac disk directly. Use `docker volume` commands or a helper container to access them.

---

## Commands / files

### Create, list, inspect

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
```

```
pgdata
DRIVER    VOLUME NAME
local     pgdata
[
    {
        "CreatedAt": "2026-11-17T12:30:01+05:30",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
        "Name": "pgdata",
        "Options": null,
        "Scope": "local"
    }
]
```

- `Mountpoint` = where the data really is on the host.
- `Scope: local` = this host only (not shared across a cluster).

Add labels to protect or group volumes:

```bash
docker volume create --label app=billing --label backup=daily billing-db
docker volume ls --filter label=backup=daily
```

### Use it

```bash
docker run -d --name pg \
  --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=pw postgres:17-alpine
sudo ls /var/lib/docker/volumes/pgdata/_data | head -5
sudo stat -c '%U:%G %u:%g %n' /var/lib/docker/volumes/pgdata/_data
```

```
PG_VERSION
base
global
pg_commit_ts
pg_dynshmem
70:70 70:70 /var/lib/docker/volumes/pgdata/_data
```

- The postgres image set the folder's owner to UID 70 (`postgres` in the Alpine image). On the host there's no user 70, so only numbers are shown. UIDs are just numbers (topic 25).

### See the image's declared volumes and the pre-fill behaviour

```bash
docker image inspect postgres:17-alpine -f '{{json .Config.Volumes}}'
docker run --rm -v html:/usr/share/nginx/html nginx:1.27-alpine true
docker run --rm -v html:/x alpine ls /x
```

```
{"/var/lib/postgresql/data":{}}
50x.html
index.html
```

- The empty `html` volume was filled from nginx's image on first use.

### Find anonymous and orphan volumes

```bash
docker volume ls -f dangling=true
docker ps -a --filter volume=pgdata --format '{{.Names}}'
```

```
DRIVER    VOLUME NAME
local     7c1e9b0f3d2a8e4b6c5f1a0d9e8b7c6a5f4e3d2c1b0a9f8e7d6c5b4a3f2e1d0c
pg
```

- `dangling=true` = no container (running or stopped) uses it.
- `--filter volume=` = which containers use a given volume.

### Backup and restore a volume

**Logical backups** (tools like `pg_dump`) are best for databases, because a file copy of a running database can be inconsistent. For general volumes, or a database that's **stopped**, a file-level backup:

```bash
# Backup: run a throwaway container with the volume (read-only) and a host folder for the archive
docker stop pg
docker run --rm \
  -v pgdata:/data:ro \
  -v "$PWD":/backup \
  alpine tar -czf /backup/pgdata-$(date +%F).tar.gz -C /data .
docker start pg
ls -lh pgdata-*.tar.gz
```

```
-rw-r--r-- 1 root root 6.1M Nov 17 12:40 pgdata-2026-11-17.tar.gz
```

- `-C /data .` = archive the volume's **contents** (not the folder path), keeping ownership and permissions.

```bash
# Restore into a new (empty) volume
docker volume create pgdata-restored
docker run --rm \
  -v pgdata-restored:/data \
  -v "$PWD":/backup:ro \
  alpine sh -c 'tar -xzf /backup/pgdata-2026-11-17.tar.gz -C /data'
docker run -d --name pg2 -v pgdata-restored:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw postgres:17-alpine
```

And the logical way for Postgres:

```bash
docker exec pg pg_dump -U postgres -Fc mydb > mydb.dump
docker exec -i pg2 pg_restore -U postgres -d mydb --clean < mydb.dump
```

### Remove volumes

```bash
docker volume rm pgdata
```

```
Error response from daemon: remove pgdata: volume is in use - [5e6f7a…]
```

- A volume used by any container (even a stopped one) can't be removed. Remove the container first. `docker rm -v <ctr>` removes a container **and its anonymous volumes** (named ones are kept).

---

## Hands-on lab

1. **The VOLUME trap, fixed:**
   ```bash
   docker run -d --name r1 redis:7-alpine
   docker exec r1 redis-cli SET k hello
   docker exec r1 redis-cli SAVE
   docker inspect -f '{{range .Mounts}}{{.Name}} -> {{.Destination}}{{end}}' r1
   docker rm -f r1
   docker run -d --name r1 redis:7-alpine
   docker exec r1 redis-cli GET k            # (nil): new anonymous volume
   docker volume ls -f dangling=true         # the old data is in here
   ```
   Now redo it with `-v redisdata:/data` on both runs. `GET k` returns `hello`.
2. **Pre-fill vs bind mount:**
   ```bash
   docker run --rm -v filled:/usr/share/nginx/html nginx:1.27-alpine ls /usr/share/nginx/html
   mkdir -p emptydir && docker run --rm -v "$PWD/emptydir":/usr/share/nginx/html nginx:1.27-alpine ls /usr/share/nginx/html
   ```
   The volume shows nginx's files. The bind mount shows nothing: the empty host folder covers the image's files.
3. **Backup and restore** the `redisdata` volume with the tar method, restore it into `redisdata2`, and start a redis with it to check the key.
4. **Two containers, one volume:** a writer `docker run -d --name w -v shared:/s alpine sh -c 'while true; do date >> /s/log; sleep 1; done'` and a reader `docker run --rm -v shared:/s:ro alpine tail -n 3 /s/log`. Try writing from the reader: `Read-only file system`.
5. Clean up: `docker rm -f r1 w pg pg2 2>/dev/null; docker volume rm filled redisdata redisdata2 shared pgdata-restored 2>/dev/null; docker volume prune -f; rm -rf emptydir`

---

## Where you'll use it

- **Databases and stateful services** on single hosts and in Compose.
- **Backups:** scheduled tar or logical dumps of volumes, shipped off-host.
- **Upgrades:** recreate containers freely, because the data stays in the named volume.
- **Shared storage:** NFS-backed volumes for apps that need shared files across hosts (with care).
- **Kubernetes mental model:** named volume ≈ PersistentVolumeClaim; volume driver ≈ CSI driver (topic 53).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| Data "disappeared" after recreating the container | Anonymous volume from the image's `VOLUME`; the new container got a new one | Mount a named volume; recover from `docker volume ls -f dangling=true` |
| `volume is in use` on `docker volume rm` | A container (maybe stopped) uses it | `docker ps -a --filter volume=<name>`, remove it first |
| `permission denied` writing to the volume | Volume owned by a different UID than the container's user | Fix ownership (one-off `chown` container), or create the folder in the image with the right owner so pre-fill copies it (topic 25) |
| New image version's files don't show up in the volume | Pre-fill happens only once, into an empty volume | Update the files yourself (migration step), or don't keep image-provided files in a volume |
| `error while mounting volume ... : failed to mount local volume: mount :/exports/app: ... connection timed out` | NFS server unreachable or export wrong | Test with `mount -t nfs` on the host; check firewall/exports |
| Database corrupted after a "hot" tar backup | Files copied while the DB was writing | Stop the DB, or use the database's own dump/backup tool |
| Two database containers on one volume → corruption | Concurrent writers | One writer per data folder |

---

## Quick recap

- A local volume is a Docker-managed folder (`/var/lib/docker/volumes/<name>/_data`) **bind-mounted** into the container. It survives container removal.
- Use **named** volumes for data. Images with a `VOLUME` instruction create **anonymous** volumes if you don't mount one, and recreating the container then "loses" data into orphans.
- An empty volume is **pre-filled** from the image's files at that path (bind mounts are not).
- Prefer `--mount` in scripts (explicit, errors on typos). Use labels to organise and protect volumes.
- Back up with logical dumps (databases) or a `tar` helper container. Drivers let volumes live on NFS/CIFS and other storage.

**Interview questions**

1. **Q: What's the difference between a volume and a bind mount?**
   A: A volume is managed by Docker (created, named, stored under Docker's data dir or a driver backend) and pre-filled from the image when empty. A bind mount maps an existing host path into the container as-is, and depends on the host's folder layout.

2. **Q: What does the `VOLUME` instruction do, and what's the risk?**
   A: It marks a path as needing a volume. If none is mounted, Docker creates an anonymous volume there. Recreating the container without a named volume creates a new empty anonymous volume, which looks like data loss and leaves orphans.

3. **Q: How do you back up a Docker volume?**
   A: For databases, use their own tool (`pg_dump`, `mysqldump`). For files, run a temporary container that mounts the volume read-only plus a host folder, and tar the volume's contents into it. Stop the writer first if consistency matters.
