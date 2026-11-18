# 23 · Why Container Data Disappears

> **Level:** ★ foundation
>
> **You will learn:** exactly where a container's files are written, when they survive (stop, restart) and when they're destroyed (`rm`, upgrades), why "upgrading a container" always means **replacing** it, and the three ways to keep data outside the container's lifecycle: **volumes**, **bind mounts** and **tmpfs**.
>
> **You need:** topic 07 (writable layer, overlay), topic 08 (lifecycle).

---

## What it is

Every container has **one private writable layer** (the overlay `upperdir`, topic 07). Every file the container creates or changes goes there, **unless** that path is a mount.

That writable layer is **tied to the container**:

| Action | Writable layer |
|---|---|
| `docker stop` / `docker start` | **Kept** |
| `docker restart` | **Kept** |
| `docker pause` | Kept |
| Host reboot (container restarted by a restart policy) | **Kept** |
| `docker rm` | **Deleted** |
| `docker run --rm` (on exit) | **Deleted** |
| "Upgrade" to a new image version | **Gone**: a new container gets a new, empty writable layer |
| `docker compose down` / `up` recreating the container | **Gone** |

So data in the writable layer lives exactly as long as **that one container**. And in the container world, containers are replaced all the time.

---

## Why it exists

This is on purpose. Containers are meant to be **disposable** (immutable infrastructure, topic 01): you can delete and recreate them at any moment, and get exactly the image again. That only works if **state is kept somewhere else**.

The problem it creates: databases, uploaded files, caches you want to keep, certificates you generated. These must **outlive** any single container. Docker gives you mounts for that.

---

## Analogy

A **hotel room**. You can unpack, rearrange the furniture and leave notes on the desk (the writable layer). If you go out and come back (stop/start), it's all still there. When you **check out** (`rm`), housekeeping resets the room to its standard layout (the image). If you want to keep things across stays, put them in a **storage locker** that belongs to you, not the room (a volume).

---

## How it works inside

### Where the writes go

```
container process writes /var/lib/app/data.db
        │
        ▼
  is /var/lib/app (or a parent) a mount?
        │
   ┌────┴─────────────────────────┐
   │ no                           │ yes
   ▼                              ▼
 overlay upperdir            the mount's backing storage:
 (container's writable       • volume  → /var/lib/docker/volumes/<name>/_data
  layer, deleted on rm)      • bind    → a host folder you chose
                             • tmpfs   → RAM (gone on stop)
```

A mount **covers** that folder inside the container's mount namespace (topic 02). The overlay never sees the writes. The data lives in the mount's backing storage, whose lifetime is independent of the container (except tmpfs).

### Why "upgrade" means "replace"

You can't change a container's image. To run `myapp:1.5` instead of `1.4`:

```
docker stop app && docker rm app
docker run -d --name app ... myapp:1.5
```

The new container starts with **an empty writable layer on top of the new image**. Anything the old one wrote in its writable layer is gone. Compose and Kubernetes do exactly the same thing. That's why "it lost all its data after we updated the image" is such a common story.

### Problems with storing data in the writable layer (even short-term)

1. **Lost on replace** (above).
2. **Performance**: every write goes through overlay; modifying a file from the image triggers a **copy-up** of the whole file (topic 07). Databases doing many small writes suffer.
3. **Not shareable**: another container can't easily read it.
4. **Hard to back up**: it's buried in `/var/lib/docker/...` or `/var/lib/containerd/...` under a random ID.
5. **Disk-full risk**: it grows silently on the Docker data disk.

### The three ways to keep data

| | **Volume** | **Bind mount** | **tmpfs mount** |
|---|---|---|---|
| What | Storage **managed by Docker** | **Any host path** mounted into the container | **RAM**-backed filesystem |
| Where on the host | `/var/lib/docker/volumes/<name>/_data` (local driver) | Wherever you say, e.g. `/srv/app/config` | Memory only |
| Survives `docker rm`? | **Yes** | **Yes** (it's just a host folder) | **No**: gone when the container stops |
| Created by | `docker volume create`, or automatically on `run` | You (the path must exist for `--mount`) | Docker, at start |
| Best for | **Databases, app state**: the default choice for persistent data | Config files, source code in development, host files like certificates | Secrets, scratch/temp data, fast caches you don't need to keep |
| Topic | 24 | 25 | 25 |

---

## Commands / files

### Data survives stop/start, dies with rm

```bash
docker run -d --name notes alpine sleep 3600
docker exec notes sh -c 'echo "important" > /notes.txt'
docker stop notes && docker start notes
docker exec notes cat /notes.txt
docker rm -f notes
docker run -d --name notes alpine sleep 3600
docker exec notes cat /notes.txt
```

```
important
cat: can't open '/notes.txt': No such file or directory
```

### See the writable layer get deleted

```bash
docker run -d --name wl alpine sh -c 'echo data > /keep.txt; sleep 3600'
PID=$(docker inspect -f '{{.State.Pid}}' wl)
UPPER=$(sudo grep ' / ' /proc/$PID/mountinfo | sed 's/.*upperdir=\([^,]*\).*/\1/')
echo $UPPER
sudo ls $UPPER
docker rm -f wl
sudo ls $UPPER
```

```
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/88/fs
keep.txt
ls: cannot access '/var/lib/containerd/.../snapshots/88/fs': No such file or directory
```

- The folder holding `keep.txt` was removed together with the container.

### Same thing with a volume: data survives

```bash
docker run --rm -v mydata:/data alpine sh -c 'echo "important" > /data/notes.txt'
docker run --rm -v mydata:/data alpine cat /data/notes.txt
docker volume ls
```

```
important
DRIVER    VOLUME NAME
local     mydata
```

- `-v mydata:/data` = mount the volume named `mydata` at `/data` (created automatically the first time). Topic 24 explains everything here.
- Two **different** containers (both deleted with `--rm`), and the data survived in the volume.

### Check what's mounted in a container

```bash
docker run -d --name m -v mydata:/data -v "$PWD":/src:ro --tmpfs /scratch alpine sleep 600
docker inspect -f '{{range .Mounts}}{{.Type}}  {{.Source}} -> {{.Destination}}  rw={{.RW}}{{println}}{{end}}' m
docker exec m df -h /data /src /scratch
```

```
volume  /var/lib/docker/volumes/mydata/_data -> /data  rw=true
bind  /home/tushar/lab -> /src  rw=false
tmpfs   -> /scratch  rw=true
Filesystem                Size      Used Available Use% Mounted on
/dev/sdb                 250.9G     12.3G    225.8G   5% /data
/dev/sdb                 250.9G     12.3G    225.8G   5% /src
tmpfs                      3.8G         0      3.8G   0% /scratch
```

- `/data` and `/src` are on the host's real disk (outside overlay). `/scratch` is RAM.
- `:ro` = read-only (topic 25).
- (`docker inspect` doesn't list tmpfs mounts in `.Mounts`, which is why its Source is empty. `docker inspect -f '{{json .HostConfig.Tmpfs}}' m` shows them.)

### Find containers storing data in their writable layer

```bash
docker ps -s --format 'table {{.Names}}\t{{.Size}}'
```

```
NAMES     SIZE
m         0B (virtual 8.31MB)
legacydb  4.12GB (virtual 4.55GB)
```

- `legacydb` has 4 GB in its writable layer. That's a database without a volume: one `docker rm` from data loss. (Fix in the lab below.)

---

## Hands-on lab

1. **Lose data on purpose:**
   ```bash
   docker run -d --name pg1 -e POSTGRES_PASSWORD=pw postgres:17-alpine
   sleep 5
   docker exec pg1 psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (42);"
   docker exec pg1 psql -U postgres -c "SELECT * FROM t;"
   docker rm -f pg1
   docker run -d --name pg1 -e POSTGRES_PASSWORD=pw postgres:17-alpine
   sleep 5
   docker exec pg1 psql -U postgres -c "SELECT * FROM t;"
   ```
   The last command fails: `relation "t" does not exist`. Wait: was the data really in the writable layer? Run `docker inspect -f '{{json .Mounts}}' pg1`. The postgres image declares a **`VOLUME`**, so Docker created an **anonymous volume** for each container. The data is in the *old* anonymous volume, which still exists but isn't attached (`docker volume ls`). Topic 24 explains this trap.
2. **Keep data:** repeat with `-v pgdata:/var/lib/postgresql/data` on both `docker run` commands. The table survives the replacement.
3. **Simulate an upgrade:** replace `postgres:17-alpine` with `postgres:17.6-alpine` (a patch version) on the second run, still with `-v pgdata:...`. The data survives. (A **major** upgrade, like 16 → 17, needs a dump/restore or `pg_upgrade`. A new image doesn't change the data format for you.)
4. Clean up: `docker rm -f pg1 m; docker volume rm pgdata mydata; docker volume prune -f`

---

## Where you'll use it

- **Every stateful service:** databases, message queues, search indexes, file uploads. Data on volumes (or external managed services).
- **Upgrades and deploys:** "can I recreate this container safely?" = "is all its state on mounts?"
- **Audits:** `docker ps -s` to find containers secretly storing data in their writable layers.
- **Kubernetes:** the same idea as `emptyDir` (temporary), `hostPath` (bind) and PersistentVolumes (volumes) (topic 53).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| "All data gone after redeploy / `compose up`" | Data was in the writable layer or an anonymous volume, and the container was recreated | Use a **named** volume or a bind mount for the data path |
| Data appears in a **new** anonymous volume after each recreate | The image has a `VOLUME` instruction and you didn't mount anything there | Mount a named volume at that path (topic 24) |
| `docker ps -s` shows GBs in a container | App writes data/logs/cache into the writable layer | Move those paths to volumes/tmpfs; log to stdout |
| Docker data disk fills up | Writable layers growing | Same as above; find them with `docker ps -s` |
| Data "lost" after a host reboot with a tmpfs mount | tmpfs is RAM | Use a volume for anything you need to keep |
| Wrote files to a path, but they're not in the volume | Wrote to a **different** path than the mount point (e.g. `/var/lib/postgres` vs `/var/lib/postgresql/data`) | Check `docker inspect .Mounts` and the app's real data folder |

---

## Quick recap

- Each container's writable layer **survives stop/restart** but is **deleted with the container**. Upgrades replace the container, so the layer is gone.
- Writable-layer storage is also slow (copy-up), hard to share and hard to back up.
- Keep state on mounts: **volumes** (Docker-managed, the default for data), **bind mounts** (host paths), **tmpfs** (RAM, temporary).
- A mount covers the folder inside the container, so writes there bypass overlay entirely.
- Check with `docker inspect .Mounts` and `docker ps -s`.

**Interview questions**

1. **Q: Does container data survive a `docker restart`? A `docker rm`?**
   A: Restart keeps the writable layer, so data survives. `rm` deletes the writable layer, so data not on a volume or bind mount is lost.

2. **Q: How do you persist a database's data in Docker?**
   A: Mount a named volume (or a bind mount) at the database's data directory, e.g. `-v pgdata:/var/lib/postgresql/data`. The data then lives outside the container and survives recreation and upgrades.

3. **Q: Name the three mount types and when to use each.**
   A: Volumes for persistent app data, managed by Docker. Bind mounts for host files like configs, certificates or source code in development. tmpfs for temporary or sensitive data that should live only in memory.
