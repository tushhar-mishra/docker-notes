# 09 · Running Containers: The Essential Options

> **Level:** ★ foundation
>
> **You will learn:** the `docker run` options you'll type every day, and what each one changes in the container underneath: `-d`, `-i`, `-t`, `--name`, `-e`, `--env-file`, `-w`, `-u`, `--hostname`, `--entrypoint`, overriding the command, `--pull`, `--platform`, `-l`, and how to attach to and detach from a running container.
>
> **You need:** topic 08 (lifecycle, PID 1). Your Linux notes on users/UIDs, environment variables and terminals.

---

## What it is

The general shape of `docker run`:

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
           └───┬───┘ └─┬─┘ └───────┬───────┘
     how to run it   what   what to run inside (optional; overrides the image default)
```

**Options must come before the image name.** Everything **after** the image name is passed to the container as its command and arguments, not to Docker.

```bash
docker run --rm alpine ls -l /       # -l goes to ls (after the image)
docker run --rm -l x=y alpine ls /   # -l goes to docker (a label; before the image)
```

Options you'll learn later (and where):
- `-p` publish ports → topic 28
- `-v` / `--mount` volumes and bind mounts → topics 24–25
- `--network` → topic 27
- `--memory`, `--cpus` → topic 31
- `--restart` → topic 32
- `--cap-add`, `--security-opt`, `--read-only` → topics 25, 41

---

## Why it exists

The image gives **defaults** (command, user, working directory, environment). Real deployments need to change some of them **without rebuilding** the image: a different database password per environment, a different command for a one-off migration, a different user for security. `docker run` options are how you set these per container.

---

## Analogy

The image is a **ready-made meal kit** with default instructions. The `docker run` options are **your notes to the cook**: "make it spicy" (`-e SPICE=high`), "cook it in kitchen 2" (`-w /kitchen2`), "the cook should be Sam, not the head chef" (`-u sam`), "make soup instead of the default dish" (command override).

---

## How it works inside

Each option becomes part of the container's configuration, which turns into the OCI runtime spec that runc reads (topic 03):

| Option | Ends up as | What runc/kernel does with it |
|---|---|---|
| `-e KEY=VAL` | `process.env` in the spec | Passed to `execve()` as the process's environment |
| `-w /path` | `process.cwd` | runc `chdir()`s there before starting your program |
| `-u 1000:1000` | `process.user.uid/gid` | runc calls `setgid()`/`setuid()` to switch to that user before exec |
| `--hostname x` | `hostname` | Set inside the UTS namespace |
| `-t` | `process.terminal: true` | A **pseudo-terminal (PTY)** is created; the container's stdin/stdout/stderr are connected to it |
| `-i` | stdin kept open | The shim keeps a pipe from your terminal to the process's stdin |
| command after image | `process.args` | The program and arguments passed to `execve()` |
| `--entrypoint` | first part of `process.args` | Replaces the image's entrypoint (topic 15) |

> **PTY (pseudo-terminal)** = a fake terminal device made by the kernel (`/dev/pts/N`). Programs behave differently when connected to a terminal: shells show a prompt, `ls` uses colors and columns, Ctrl+C becomes SIGINT. Without `-t`, the program sees plain pipes, like when you redirect output to a file.

### Foreground vs detached

- **Foreground (default):** your terminal is attached to the container's output. When the container exits, the command returns. Pressing Ctrl+C sends SIGINT to the container **only if `-t` is used**. Without `-t`, Docker forwards the signal anyway, but the process may ignore it if it's PID 1 (topic 12).
- **Detached (`-d`):** Docker starts the container, prints its ID and returns immediately. The container keeps running in the background. Output still goes to the log (`docker logs`).

### The `-i` / `-t` combinations

| Flags | Use for | Example |
|---|---|---|
| none | Run a command, see output | `docker run --rm alpine date` |
| `-it` | An interactive shell or program | `docker run -it --rm ubuntu bash` |
| `-i` only | Piping data in (no terminal) | `cat dump.sql \| docker run -i --rm postgres:17 psql ...` |
| `-d` | Long-running services | `docker run -d nginx` |
| `-dit` | A background container with a shell you can attach to later | `docker run -dit --name box ubuntu` |

### Users inside the container

By default, a container runs as **whatever user the image says**, and if the image doesn't say, as **root (UID 0)**. That root is the **same UID 0 as on the host** (unless user namespaces are on; topic 42), only fenced in by namespaces, capabilities and seccomp. That's why running as root in containers is a security risk (topic 41).

`-u` takes a name or a number:
- `-u nginx` → the name is looked up in the **container's** `/etc/passwd`. If it's not there: error.
- `-u 1000:1000` → numbers work even if no such user exists in the image. The process just runs with that UID/GID. Files it creates are owned by UID 1000 **on the host too** (topic 25).

### Environment variables

- Set by the image (`ENV` in the Dockerfile; topic 18), overridden by `-e` or `--env-file`.
- `-e KEY` (without `=value`) copies the value from **your shell** into the container.
- Environment variables are **visible to anyone who can run `docker inspect`** and to every process in the container. **Don't put real secrets in them** if you can avoid it (topic 44).

---

## Commands / files

### `-d` and `--name`

```bash
docker run -d --name web nginx:1.27-alpine
```

```
8b4f2c1d9e7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c
```

- `-d` printed the container ID and returned.
- `--name web` = you can use `web` instead of the ID in every command. Names must be **unique** on the host.

### `-it`: interactive shell

```bash
docker run -it --rm --name shell ubuntu:24.04 bash
```

```
root@4c2a9e1f3b7d:/#
```

### Overriding the command

```bash
docker run --rm nginx:1.27-alpine nginx -v
docker run --rm nginx:1.27-alpine cat /etc/nginx/conf.d/default.conf | head -5
```

```
nginx version: nginx/1.27.5
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
```

- Normally this image runs `nginx -g 'daemon off;'`. Your command replaced it. This is a great way to **look inside an image** without starting the real app.

### `-e` and `--env-file`

```bash
docker run --rm -e APP_ENV=staging -e LOG_LEVEL=debug alpine env
```

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=3a1b2c3d4e5f
APP_ENV=staging
LOG_LEVEL=debug
HOME=/root
```

- `PATH` came from the image. `HOSTNAME` is set by Docker. `HOME` is set based on the user.

File `app.env`:

```
APP_ENV=production
LOG_LEVEL=info
# comments are allowed
DB_HOST=db.internal
```

```bash
docker run --rm --env-file app.env alpine env | grep -E 'APP|LOG|DB'
```

```
APP_ENV=production
LOG_LEVEL=info
DB_HOST=db.internal
```

- Format: one `KEY=VALUE` per line. **No quotes are removed**: `KEY="x"` gives the value `"x"` including the quotes. No `export`, no variable expansion.

### `-w`: working directory

```bash
docker run --rm -w /tmp alpine pwd
```

```
/tmp
```

- If the folder doesn't exist, Docker creates it.

### `-u`: run as another user

```bash
docker run --rm alpine id
docker run --rm -u 1000:1000 alpine id
docker run --rm -u nobody alpine id
docker run --rm -u nosuchuser alpine id
```

```
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
uid=1000 gid=1000 groups=1000
uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
docker: Error response from daemon: unable to find user nosuchuser: no matching entries in passwd file
```

- `uid=1000` with no name in brackets = the UID works, but no `/etc/passwd` entry has it inside this image.
- `nosuchuser` → error at start, because a **name** must exist in the image.

### `--hostname`

```bash
docker run --rm --hostname api-1 alpine hostname
```

```
api-1
```

### `--entrypoint`

Some images have an **entrypoint**: a fixed program that always runs, with your command passed to it as arguments (topic 15 explains this fully). To bypass it, for example to get a shell in an image whose entrypoint is the app:

```bash
docker run --rm -it --entrypoint sh nginx:1.27-alpine
```

### `--pull` and `--platform`

```bash
docker run --pull always --rm alpine:3.22 cat /etc/alpine-release   # always check the registry first
docker run --pull never  --rm alpine:3.22 true                      # fail if not local
docker run --platform linux/arm64 --rm alpine uname -m              # needs emulation on amd64 (topic 21)
```

- `--pull missing` is the default: pull only if not local.

### `-l`: labels

```bash
docker run -d --name api -l team=payments -l env=staging nginx:1.27-alpine
docker ps --filter label=team=payments --format '{{.Names}}'
```

```
api
```

- **Labels** are key=value notes attached to containers (and images, volumes, networks). Docker ignores them. You, scripts, Compose (topic 35) and monitoring tools use them to find and group things.

### attach and detach

Start a container with a shell in the background:

```bash
docker run -dit --name box ubuntu:24.04 bash
docker attach box
```

```
root@9d8c7b6a5f4e:/#
```

- `docker attach` connects your terminal to the container's **main process** (PID 1) stdin/stdout. Not a new shell: the same `bash` that is PID 1.
- **Danger:** if you type `exit` or press Ctrl+D here, you end **PID 1**, so the **container stops**.
- To leave without stopping: press **Ctrl+P then Ctrl+Q** (the default **detach keys**). Works only if the container was started with `-it`.

```
root@9d8c7b6a5f4e:/# read escape sequence
```

- That message means you detached; the container keeps running.

Change the detach keys (useful when Ctrl+P is used by your shell for history):

```bash
docker attach --detach-keys="ctrl-x,x" box
```

For **getting a shell** inside a running container, you almost always want `docker exec` (topic 10), which starts a **new** process and doesn't touch PID 1.

---

## Hands-on lab

1. **Command override vs image default:**
   ```bash
   docker image inspect redis:7-alpine --format 'Entrypoint={{.Config.Entrypoint}} Cmd={{.Config.Cmd}}'
   docker run --rm redis:7-alpine redis-server --version
   ```
   See what the image runs by default, then run something else.

2. **See the environment and user of a running container from the host:**
   ```bash
   docker run -d --name envlab -u 1000 -e COLOR=blue alpine sleep 600
   PID=$(docker inspect -f '{{.State.Pid}}' envlab)
   sudo cat /proc/$PID/environ | tr '\0' '\n'
   ps -o pid,uid,user,cmd -p $PID
   ```
   ```
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   HOSTNAME=1f2e3d4c5b6a
   COLOR=blue
   HOME=/
       PID   UID USER     CMD
     81234  1000 tushar   sleep 600
   ```
   - `/proc/PID/environ` = the environment variables the process was started with (NUL-separated, hence `tr`).
   - On the host, UID 1000 shows as **your** username, because the host's `/etc/passwd` maps 1000 → you. The kernel only sees the number. This is why UIDs matter for file ownership (topic 25).

3. **Feel the difference between `-t` and no `-t`:**
   ```bash
   docker run --rm alpine ls /
   docker run --rm -t alpine ls /
   ```
   Without `-t`: one name per line (output is a pipe). With `-t`: columns (output is a terminal).

4. **Attach and detach safely:**
   ```bash
   docker run -dit --name box2 alpine sh
   docker attach box2
   # press Ctrl+P, then Ctrl+Q
   docker ps --filter name=box2 --format '{{.Status}}'
   ```
   Still `Up`. Now attach again and type `exit`. Check again: `Exited (0)`.

5. Clean up: `docker rm -f web api box box2 envlab 2>/dev/null`

---

## Where you'll use it

- **One-off jobs:** database migrations, backups, admin scripts: `docker run --rm -e ... myapp:1.4 ./migrate.sh`.
- **Debugging images:** override the command with `sh` or `cat` to see what's inside.
- **Per-environment config:** `--env-file prod.env` vs `--env-file staging.env`, with the same image.
- **Security:** `-u` to run as non-root when the image doesn't do it itself.
- **Inventory:** labels for team, environment, cost centre; filters for cleanup scripts.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `docker: Error response from daemon: Conflict. The container name "/web" is already in use by container "8b4f…"` | Names are unique, and a stopped container still holds the name | `docker rm web` (or `-f`), or pick another name |
| `the input device is not a TTY` | `-t` used in a script or CI where there's no terminal | Remove `-t` (keep `-i` if piping input) |
| Flags seem ignored, e.g. `docker run alpine --rm ls` | Everything after the image name goes to the container | Put Docker options **before** the image |
| `unable to find user xyz: no matching entries in passwd file` | `-u` name doesn't exist in the image | Use a numeric UID, or create the user in the Dockerfile |
| `exec: "bash": executable file not found in $PATH` | The image has no bash (Alpine, distroless) | Use `sh`, or the right image |
| Typed `exit` in `docker attach`, and the container stopped | You ended PID 1 | Use Ctrl+P Ctrl+Q to detach, or use `docker exec` for shells |
| Env var has literal quotes, e.g. `"secret"` | `--env-file` doesn't strip quotes | Remove the quotes in the file |
| `permission denied` writing files with `-u 1000` | That UID can't write to folders owned by root in the image | Fix ownership in the Dockerfile (`chown`) or use a writable path like `/tmp` (topic 25) |

---

## Quick recap

- `docker run [OPTIONS] IMAGE [COMMAND] [ARGS]`: options before the image, the command after it.
- `-d` = background. `-it` = interactive terminal. `--name` = a friendly unique name. `--rm` = delete on exit.
- `-e`/`--env-file`, `-w`, `-u`, `--hostname`, `--entrypoint` and the command override the image's defaults **without rebuilding**. Each one ends up in the OCI spec that runc applies.
- Default user is root (UID 0, the same number as host root) unless the image or `-u` says otherwise.
- `docker attach` connects to PID 1: detach with Ctrl+P Ctrl+Q, never `exit`. Use `docker exec` for shells.

**Interview questions**

1. **Q: What does `-it` do?**
   A: `-i` keeps stdin open so you can type into the process. `-t` gives it a pseudo-terminal, so it behaves like an interactive terminal (prompt, line editing, Ctrl+C as SIGINT). Together they give you an interactive shell.

2. **Q: How do you run a container as a non-root user without changing the image?**
   A: `docker run -u <uid>:<gid> ...`. The numeric UID works even if no such user exists in the image. Files the process creates are owned by that UID on the host too.

3. **Q: What's the difference between `docker attach` and `docker exec`?**
   A: `attach` connects your terminal to the main process (PID 1). Exiting it can stop the container. `exec` starts a new, extra process in the container's namespaces, and exiting it doesn't affect the main process.
