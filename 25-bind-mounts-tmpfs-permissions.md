# 25 · Bind Mounts, tmpfs and Permissions

> **Level:** ★★ intermediate
>
> **You will learn:** how bind mounts work (the Linux `mount --bind` feature), the traps (auto-created folders, hidden image files, single-file mounts that go stale), how UIDs/GIDs decide who can read and write mounted files and how to fix "permission denied", SELinux labels `:z`/`:Z`, tmpfs mounts in RAM, read-only root filesystems, mount propagation, and WSL2 performance tips.
>
> **You need:** topics 23–24. From your Linux notes: users, UIDs/GIDs, permissions (`rwx`), `mount`.

---

## What it is

- A **bind mount** makes an **existing host file or folder** appear at a path inside the container. Both sides see the **same** files: a change on one side is instantly visible on the other.
- A **tmpfs mount** is a filesystem that lives **only in RAM**. It's fast, never written to disk, and gone when the container stops.
- A **read-only root filesystem** (`--read-only`) makes the container's whole `/` unwritable, except for the mounts you explicitly allow.

---

## Why it exists

- **Bind mounts**: give containers host files that aren't part of the image: configs, TLS certificates, the Docker socket (careful, topic 40), host logs for a log shipper, and your **source code during development** (edit on the host, the container sees it instantly).
- **tmpfs**: scratch space that must be fast, or must **never touch the disk** (secrets, session data), or must be writable even when the root filesystem is read-only.
- **Read-only root**: if an attacker gets in, they can't change binaries or drop tools into the filesystem. It's also a good way to catch apps that write to unexpected places.

---

## Analogy

- **Bind mount** = a **window** cut between two rooms. Both sides look at the same table. Move a cup on one side, and the other side sees it moved.
- **tmpfs** = a **whiteboard**. Fast to write on, wiped clean at the end of the day.
- **Read-only root** = the room's walls and furniture are **glued down**. You can only use the whiteboard and the boxes you were given.

---

## How it works inside

### Bind mount = `mount --bind`

**Linux recap:** `mount --bind /src /dst` makes the folder `/src` also reachable at `/dst`. It's the same directory (same inodes, the same data on disk), with two paths to reach it.

For `-v /srv/app/config:/etc/app`:
1. runc, inside the container's new mount namespace, runs a bind mount of host `/srv/app/config` onto `<container-root>/etc/app`.
2. The mount **covers** whatever the image had at `/etc/app`. Those image files are hidden (not deleted) while the mount is there.
3. Reads and writes go straight to the host folder. No overlay, no copy-up.

### Trap 1: `-v` creates missing host paths (as root)

```bash
docker run -v /srv/typo-path:/data alpine ls /data
```

- If `/srv/typo-path` doesn't exist, `-v` **silently creates an empty folder owned by root** and mounts it. Your app then starts with no config and no error.
- `--mount type=bind,...` **refuses** with `bind source path does not exist`. That's safer.

### Trap 2: mounting over a folder hides the image's files

Mount an empty host folder at `/usr/share/nginx/html`, and nginx's `index.html` is hidden. Unlike volumes, **bind mounts are never pre-filled** (topic 24).

### Trap 3: single-file bind mounts go stale

```bash
docker run -v /srv/app/config.yaml:/app/config.yaml myapp
```

A bind mount points to the file's **inode** (the file's identity on disk), not its name. Many editors (vim, sed `-i`, most config-management tools) save by **writing a new file and renaming it over the old one**. That's a **new inode**. The container still sees the **old** inode, so **your change never appears inside**, even after you "reload" the app.

Fixes: mount the **folder** instead of the single file, or restart the container after changes.

### UIDs, GIDs and "permission denied"

**Linux recap:** the kernel decides access using **numbers**: the process's UID/GID versus the file's owner UID/GID and mode bits (`rwxr-x---`). Names (`tushar`, `postgres`) are just labels from `/etc/passwd`, which can differ between host and container.

Because containers share the host kernel (and, by default, the host's **user namespace**), a container process with UID 10001 is UID 10001 **for the kernel**, when it touches host files through a bind mount.

Common situations:

| Situation | What happens |
|---|---|
| Container runs as **root (UID 0)**, writes to a bind mount | Files on the host are **owned by root**. Your normal user can't edit or delete them without sudo |
| Container runs as UID 10001, host folder owned by UID 1000 with mode `755` | `Permission denied` when writing: UID 10001 is "others", which only has `r-x` |
| Container runs as UID 1000, and your host user is UID 1000 | Works perfectly: same number, same owner |
| Volume pre-filled from the image, where the folder is owned by the app user | Works: ownership is copied from the image (topic 24) |

How to fix it, in order of preference:
1. **Make the IDs match**: run the container with your UID in development: `docker run -u "$(id -u):$(id -g)" ...`.
2. **Set ownership on the host** for the UID the container uses: `sudo chown -R 10001:10001 /srv/app/data`.
3. **Use a shared group**: give the folder group-write permission for a GID, and add that group to the container with `--group-add`.
4. **Use volumes** and create the folder in the image with the right owner, so the pre-fill sets ownership.
5. **User namespaces/rootless Docker** remap the IDs automatically (topic 42), but they change how ownership looks on the host.

Avoid `chmod 777`: it "works" by giving every user on the host write access.

### SELinux labels: `:z` and `:Z`

On RHEL/Fedora/CentOS with **SELinux** enforcing (SELinux = Security-Enhanced Linux, a kernel security module that labels every file and process and only allows label combinations the policy permits; topic 41a), containers run with the label `container_t` and can only access files labelled `container_file_t`. A bind-mounted host folder usually has some other label, so you get **permission denied even as root**.

| Option | Effect |
|---|---|
| `-v /srv/data:/data:z` | Relabel the host folder so **all containers** can share it |
| `-v /srv/data:/data:Z` | Relabel it **private to this one container** (others can't use it) |

- Relabeling changes the **host** files' labels. **Never** use `:z`/`:Z` on system folders like `/`, `/home`, `/usr` or `/etc`: you can break the host.
- On Ubuntu/Debian (AppArmor instead of SELinux), these options are accepted and ignored.
- Check labels with `ls -Z /srv/data`.

### tmpfs mounts

```bash
docker run --tmpfs /tmp:rw,noexec,nosuid,size=64m,mode=1777 myapp
docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=67108864,tmpfs-mode=1777 myapp
```

- Lives in RAM (and can be swapped out if the host has swap).
- **Counts toward the container's memory limit** (it's page cache in its cgroup). A 1 GB tmpfs with a 512 MB memory limit = OOM kill when it fills (topic 31).
- `size=` caps it. Without a cap, it can grow up to half the host's RAM by default.
- `noexec` = files here can't be executed (stops attackers running downloaded tools from `/tmp`).
- `mode=1777` = writable by everyone with the "sticky bit" (people can only delete their own files), the normal `/tmp` permissions.
- **Gone when the container stops.** Not even `docker restart` keeps it.
- Docker gives every container a small tmpfs at **`/dev/shm`** (shared memory, 64 MB by default). Some apps (Chrome, PostgreSQL, ML frameworks) need more: `--shm-size=1g`.

### Read-only root filesystem

```bash
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /run:rw,size=8m \
  -v appdata:/var/lib/app \
  myapp
```

- The overlay root is mounted **read-only**. Any write outside the listed mounts fails with `Read-only file system`.
- You add writable spots only where the app needs them: temp files, PID files, the data folder.
- First run: watch for errors to find where the app writes. `docker diff` on a normal run also shows you (topic 10).
- Kubernetes: `securityContext.readOnlyRootFilesystem: true`. The same idea.

### Mount propagation (advanced)

By default, bind mounts are **`rprivate`**: if something new is mounted on the host **under** a bind-mounted folder **after** the container started, the container doesn't see it (and the reverse).

Options: `rshared`, `shared`, `rslave`, `slave`, `rprivate`, `private` (e.g. `--mount type=bind,src=/mnt,dst=/mnt,bind-propagation=rslave`). Storage agents and CSI drivers need these. Normal apps don't. Know they exist.

### WSL2 and Docker Desktop note

- In WSL2, files under `/home/<you>/...` live on WSL's Linux filesystem. They're fast to bind-mount.
- Files under `/mnt/c/...` are your **Windows** drive, accessed over a translation layer. Bind-mounting them is **much slower** and has permission quirks. **Keep your projects inside the WSL filesystem.**

---

## Commands / files

### Bind mount a config folder, read-only

```bash
mkdir -p ~/site && echo '<h1>Hello from the host</h1>' > ~/site/index.html
docker run -d --name site -p 8081:80 \
  --mount type=bind,src=$HOME/site,dst=/usr/share/nginx/html,readonly \
  nginx:1.27-alpine
curl -s localhost:8081
echo '<h1>Changed live</h1>' > ~/site/index.html
curl -s localhost:8081
docker exec site sh -c 'echo hack > /usr/share/nginx/html/x'
```

```
<h1>Hello from the host</h1>
<h1>Changed live</h1>
sh: can't create /usr/share/nginx/html/x: Read-only file system
```

- Host edits appear immediately. The container can't write.

### `--mount` refuses missing paths, `-v` doesn't

```bash
docker run --rm --mount type=bind,src=/nope,dst=/data alpine true
docker run --rm -v /nope:/data alpine true; ls -ld /nope
```

```
docker: Error response from daemon: invalid mount config for type "bind": bind source path does not exist: /nope
drwxr-xr-x 2 root root 4096 Nov 17 13:01 /nope
```

### The ownership problem and fixes

```bash
mkdir -p ~/out
docker run --rm -v ~/out:/out alpine sh -c 'echo hi > /out/root-file'
docker run --rm -u "$(id -u):$(id -g)" -v ~/out:/out alpine sh -c 'echo hi > /out/my-file'
docker run --rm -u 10001 -v ~/out:/out alpine sh -c 'echo hi > /out/x'
ls -ln ~/out
```

```
sh: can't create /out/x: Permission denied
-rw-r--r-- 1 1000 1000 3 Nov 17 13:05 my-file
-rw-r--r-- 1    0    0 3 Nov 17 13:05 root-file
```

- `ls -ln` shows numeric owners. `root-file` (UID 0) needs sudo to delete on the host.
- `-u $(id -u):$(id -g)` → files owned by you.
- UID 10001 → denied, because the folder is `drwxr-xr-x 1000 1000`.

Fix for a fixed container UID:

```bash
sudo chown 10001:10001 ~/out && docker run --rm -u 10001 -v ~/out:/out alpine sh -c 'echo ok > /out/x' && ls -ln ~/out/x
```

### The stale single-file mount

```bash
echo "version=1" > ~/app.conf
docker run -d --name sf -v ~/app.conf:/app.conf alpine sleep 600
sed -i 's/1/2/' ~/app.conf             # sed -i writes a new file and renames it (new inode)
cat ~/app.conf
docker exec sf cat /app.conf
ls -i ~/app.conf; docker exec sf ls -i /app.conf
```

```
version=2
version=1
1835102 /home/tushar/app.conf
1835099 /app.conf
```

- Different inode numbers: the container still has the old file. Restart the container (`docker restart sf`) or mount the folder instead.

### tmpfs and read-only root

```bash
docker run --rm --read-only --tmpfs /tmp:size=16m alpine sh -c '
  echo ok > /tmp/a && echo "tmp write ok";
  echo no > /etc/b || echo "root fs is read-only";
  df -h /tmp'
```

```
tmp write ok
sh: can't create /etc/b: Read-only file system
root fs is read-only
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    16.0M      4.0K     16.0M   0% /tmp
```

### SELinux labels (on RHEL/Fedora)

```bash
ls -Zd /srv/data
docker run --rm -v /srv/data:/data alpine ls /data        # Permission denied
docker run --rm -v /srv/data:/data:Z alpine ls /data      # works
ls -Zd /srv/data
```

```
unconfined_u:object_r:var_t:s0 /srv/data
ls: can't open '/data': Permission denied
system_u:object_r:container_file_t:s0:c123,c456 /srv/data
```

- After `:Z`, the folder has the `container_file_t` type with this container's unique **MCS** categories (`c123,c456`; MCS = Multi-Category Security, the part of the label that keeps containers apart).

---

## Hands-on lab

1. **Live-reload development:** bind-mount a folder with a small Python script into `python:3.13-slim`, run it, edit it on the host, run it again without rebuilding.
2. **Fix permissions three ways:** with `-u $(id -u):$(id -g)`, with `chown` to the container's UID, and with a shared group (`chgrp 2000 dir; chmod g+w dir; docker run --group-add 2000 -u 10001 ...`).
3. **Find where an app writes:** run `nginx:1.27-alpine` with `--read-only`. It fails. Read the errors (`/var/cache/nginx/...`, `/var/run/nginx.pid`), then add tmpfs mounts until it starts:
   ```bash
   docker run -d --name ro --read-only \
     --tmpfs /var/cache/nginx --tmpfs /var/run --tmpfs /tmp nginx:1.27-alpine
   docker logs ro
   ```
4. **tmpfs counts as memory:**
   ```bash
   docker run --rm --memory 64m --tmpfs /t:size=200m alpine sh -c 'dd if=/dev/zero of=/t/fill bs=1M count=100; echo done'
   echo "exit: $?"
   ```
   The OOM killer steps in before 100 MB are written, because tmpfs pages count toward the 64 MB limit. Depending on which process it picks, you'll see `Killed` from `dd` (and maybe `done` after it), or the whole container exits with 137. Either way, `docker run` with `--memory` plus a big tmpfs is a memory trap. (Topic 31 shows how to read the `oom_kill` counter.)
5. Reproduce the stale single-file mount, then fix it by mounting the folder.
6. Clean up: `docker rm -f site sf ro; sudo rm -rf ~/out /nope; rm -rf ~/site ~/app.conf`

---

## Where you'll use it

- **Development:** bind-mount source code (topic 38's `compose watch` is the modern alternative).
- **Config and certificates:** read-only bind mounts of `/etc/myapp` or `/etc/ssl/...` from config management.
- **Hardening:** `--read-only` + tmpfs for `/tmp`/`/run` + `noexec`, matching Kubernetes `readOnlyRootFilesystem`.
- **Agents:** log shippers and monitoring agents bind-mount host paths (`/var/log`, `/proc`, `/sys`) read-only.
- **RHEL hosts:** `:Z`/`:z` for SELinux.
- **Incidents:** "permission denied on a volume", one of the most common container tickets (topic 52).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Permission denied` writing to a bind mount | Container UID doesn't have write permission on the host folder | Match UIDs (`-u`), `chown`, or a shared group |
| Files on host owned by root after running a container | Container ran as root | Run with `-u $(id -u):$(id -g)` in dev; non-root `USER` in images |
| `Permission denied` even as root, on RHEL/Fedora | SELinux label mismatch | `:Z` or `:z` (never on system dirs); check `ausearch -m avc` / `dmesg` |
| App starts with empty config, no error | `-v` created a missing host path as an empty folder | Use `--mount type=bind` (fails fast); fix the path |
| Config edits on the host aren't seen in the container | Single-file bind mount + editor replaced the inode | Mount the folder; or restart the container |
| `Read-only file system` | `--read-only` (or `:ro`) and the app writes there | Add a tmpfs or volume for that path |
| OOM kill while writing to `/tmp` | tmpfs uses memory counted against the limit | Set tmpfs `size`, raise memory, or use a volume |
| `Bus error` / Chrome or Postgres crashes about shared memory | `/dev/shm` (64 MB) too small | `--shm-size=1g` |
| Very slow file access in WSL/Docker Desktop | Bind mount from the Windows filesystem (`/mnt/c`) | Keep files inside the WSL Linux filesystem |

---

## Quick recap

- Bind mount = Linux `mount --bind` of a host path into the container. Same files both ways, never pre-filled, and it hides the image's files at that path.
- Use `--mount type=bind` (it errors on missing paths). Beware single-file mounts going stale when editors replace the file.
- Access is decided by **numeric UID/GID**: match them (`-u`), `chown`, or use groups. Avoid 777.
- SELinux hosts: `:z` (shared) / `:Z` (private) relabel host files, never on system folders.
- tmpfs = RAM, counts toward the memory limit, gone on stop. `--read-only` + tmpfs for `/tmp` and `/run` hardens containers.

**Interview questions**

1. **Q: A container gets "permission denied" writing to a mounted folder. How do you troubleshoot?**
   A: Compare the container process's UID/GID (`docker exec id`, or `/proc/PID/status`) with the host folder's numeric owner and mode (`ls -ln`). Fix by matching IDs, changing ownership or group permissions. On SELinux hosts, also check labels and use `:Z`/`:z`.

2. **Q: Why might an updated config file not be seen by a container that bind-mounts it?**
   A: A single-file bind mount is tied to the file's inode. Editors that save by writing a new file and renaming it create a new inode, so the container keeps seeing the old one. Mount the directory instead, or restart.

3. **Q: What is `--read-only` and how do you make apps work with it?**
   A: It mounts the container's root filesystem read-only. You add writable tmpfs mounts (for `/tmp` or `/run`) and volumes only for paths the app must write. That limits what an attacker can change.
