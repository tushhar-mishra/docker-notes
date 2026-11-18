# 15 · CMD vs ENTRYPOINT

> **Level:** ★★ intermediate
>
> **You will learn:** the difference between `CMD` and `ENTRYPOINT`, exactly how they combine (including the confusing shell-form cases), how to override each one at runtime, the entrypoint-script pattern with `exec "$@"`, and two smaller instructions: `STOPSIGNAL` and `SHELL`.
>
> **You need:** topic 12 (PID 1, exec vs shell form), topic 14 (Dockerfile basics).

---

## What it is

Both tell Docker **what to run when the container starts**. Together they form the container's command line:

```
final command = ENTRYPOINT + CMD
```

- **ENTRYPOINT** = the **fixed part**: the program that always runs.
- **CMD** = the **default arguments** (or the default command if there's no ENTRYPOINT). Easy to override.

| You want | Use |
|---|---|
| "Run this app by default, but let people run something else easily" | `CMD ["python", "app.py"]` only |
| "This image **is** a tool; users pass arguments to it" | `ENTRYPOINT ["curl"]` + `CMD ["--help"]` |
| "Do some setup first, then run whatever command was given" | `ENTRYPOINT ["/entrypoint.sh"]` + `CMD ["python", "app.py"]` |

---

## Why it exists

Images are used in two styles:
1. **Service images** (a web app): usually run with no arguments, but sometimes you want a shell or a one-off command instead. CMD fits: easy to replace.
2. **Tool images** (`curl`, `terraform`, `aws-cli`): you always run the same program with different arguments. ENTRYPOINT fits: `docker run mytool --version` passes `--version` to the tool.

And many service images need a **startup script** (create config from env vars, wait for the DB, fix permissions) that must run **before** whatever command the user chose. ENTRYPOINT + CMD handles that.

---

## Analogy

A **coffee machine**:
- **ENTRYPOINT** = the machine itself. It always makes coffee. You can't make it toast bread (without opening it up: `--entrypoint`).
- **CMD** = the default button: "medium latte". If you press another button ("espresso, double"), that replaces the default, but it's still the coffee machine doing the work.

---

## How it works inside

### The two forms (again, because it matters here)

- **Exec form** (JSON array): `["program", "arg1", "arg2"]` → run directly with `execve()`. No shell. No variable expansion (`$HOME` stays literal).
- **Shell form** (plain string): `program arg1 arg2` → becomes `["/bin/sh", "-c", "program arg1 arg2"]`. Shell features work (`$VAR`, `&&`, `|`), but the **shell is PID 1** (signal problems; topic 12).

### The combination table

How Docker builds the final command from what's in the image:

| | **No ENTRYPOINT** | **ENTRYPOINT shell form** `ENTRYPOINT e1 e2` | **ENTRYPOINT exec form** `ENTRYPOINT ["e1","e2"]` |
|---|---|---|---|
| **No CMD** | error: no command | `/bin/sh -c "e1 e2"` | `e1 e2` |
| **CMD exec form** `CMD ["c1","c2"]` | `c1 c2` | `/bin/sh -c "e1 e2"` (**CMD ignored!**) | `e1 e2 c1 c2` ✅ |
| **CMD shell form** `CMD c1 c2` | `/bin/sh -c "c1 c2"` | `/bin/sh -c "e1 e2"` (**CMD ignored!**) | `e1 e2 /bin/sh -c "c1 c2"` (odd) |

Rules to remember:
1. **Shell-form ENTRYPOINT ignores CMD and any arguments from `docker run`.** Almost never what you want.
2. **ENTRYPOINT exec form + CMD exec form** is the useful combination: CMD is appended as arguments.
3. **Setting ENTRYPOINT in your Dockerfile resets the CMD inherited from the base image** to empty. Set CMD again if you need one.
4. If there are several CMD (or ENTRYPOINT) lines, **only the last one counts**.

### Overriding at runtime

| You run | Effect |
|---|---|
| `docker run img` | ENTRYPOINT + CMD as in the image |
| `docker run img a b` | ENTRYPOINT + **`a b`** (replaces CMD) |
| `docker run --entrypoint x img` | **`x`** + (nothing: CMD is reset when you override the entrypoint) |
| `docker run --entrypoint x img a b` | `x a b` |
| `docker run --entrypoint "" img a b` | `a b` (no entrypoint at all) |

`--entrypoint` takes **only the program**, not arguments. Arguments go after the image name.

### How it's stored

Both end up in the image config (topic 07):

```bash
docker image inspect redis:7-alpine --format 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}}'
```

```
Entrypoint=["docker-entrypoint.sh"] Cmd=["redis-server"]
```

At start, dockerd joins them into the OCI spec's `process.args` = `["docker-entrypoint.sh", "redis-server"]`, and runc `execve`s that (topic 03).

### The entrypoint-script pattern

Official images like `postgres`, `redis`, `mysql`, `nginx` all use it. A typical script:

```sh
#!/bin/sh
set -e

# 1. If the first argument looks like a flag (e.g. "--port 7000"), assume the user
#    wants the default program with extra flags.
if [ "${1#-}" != "$1" ]; then
    set -- redis-server "$@"
fi

# 2. Do setup only when running the real server, not for "sh" or "redis-cli".
if [ "$1" = "redis-server" ]; then
    echo "preparing data directory..."
    chown -R redis:redis /data 2>/dev/null || true
fi

# 3. Replace the shell with the final command → it becomes PID 1 (topic 12).
exec "$@"
```

- `set -e` = stop the script if any command fails.
- `"${1#-}" != "$1"` = "does the first argument start with `-`?"
- `set -- redis-server "$@"` = rewrite the argument list to put `redis-server` in front.
- `exec "$@"` = run the final command **as PID 1**.

This gives the best of both worlds:
- `docker run redis` → script → `exec redis-server`
- `docker run redis --port 7000` → script → `exec redis-server --port 7000`
- `docker run -it redis sh` → script skips setup → `exec sh`

### `STOPSIGNAL`

```dockerfile
STOPSIGNAL SIGQUIT
```

- Sets the signal `docker stop` sends first (default SIGTERM; topic 12).
- Use it when your program shuts down gracefully on a different signal (nginx: SIGQUIT; Apache httpd: SIGWINCH; PostgreSQL's image uses SIGINT for "fast shutdown").
- Override at runtime with `docker run --stop-signal`.

### `SHELL`

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

- Changes the shell used for **shell-form** `RUN`, `CMD` and `ENTRYPOINT` (default on Linux: `["/bin/sh", "-c"]`).
- Common reason: `-o pipefail`. Without it, `RUN curl -fsSL url | tar xz` **succeeds even if `curl` fails**, because only the last command's exit code (`tar`) counts. With `pipefail`, any failure in the pipe fails the step.
- The shell must exist in the image (`bash` isn't in Alpine by default).

---

## Commands / files

### Example 1: CMD only (service image)

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.22
CMD ["echo", "default command"]
```

```bash
docker build -t cmdonly .
docker run --rm cmdonly
docker run --rm cmdonly echo "replaced"
docker run --rm cmdonly ls /etc/alpine-release
```

```
default command
replaced
/etc/alpine-release
```

- Anything after the image name **replaces** CMD completely.

### Example 2: ENTRYPOINT + CMD (tool image)

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.22
RUN apk add --no-cache curl
ENTRYPOINT ["curl", "-sS"]
CMD ["--help"]
```

```bash
docker build -t mycurl .
docker run --rm mycurl | head -2
docker run --rm mycurl -I https://example.com
docker run --rm --entrypoint sh mycurl -c 'which curl'
```

```
Usage: curl [options...] <url>
 -d, --data <data>           HTTP POST data
HTTP/2 200
content-type: text/html
...
/usr/bin/curl
```

- No arguments → `curl -sS --help`.
- `-I https://example.com` → replaces CMD → `curl -sS -I https://example.com`.
- `--entrypoint sh` + `-c 'which curl'` → `sh -c 'which curl'`.

### Example 3: the shell-form ENTRYPOINT trap

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.22
ENTRYPOINT echo "entry says:"
CMD ["hello"]
```

```bash
docker build -t badentry .
docker run --rm badentry
docker run --rm badentry world
```

```
entry says:
entry says:
```

- CMD (`hello`) and your argument (`world`) are both **silently ignored**. That's rule 1 from the table.

### Example 4: entrypoint script

```
entrydemo/
├── Dockerfile
└── entrypoint.sh
```

`entrypoint.sh`:

```sh
#!/bin/sh
set -e
echo "[entrypoint] APP_ENV=${APP_ENV:-dev}, preparing..."
mkdir -p /tmp/app-cache
exec "$@"
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
COPY --chmod=755 entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["python", "-m", "http.server", "8000"]
```

- `COPY --chmod=755` = copy and make it executable in the same step (otherwise you get exit **126**, topic 08).

```bash
docker build -t entrydemo .
docker run -d --name ed -e APP_ENV=staging entrydemo
docker logs ed
docker top ed -o pid,args
docker run --rm entrydemo python --version
```

```
[entrypoint] APP_ENV=staging, preparing...
PID     COMMAND
97101   python -m http.server 8000
[entrypoint] APP_ENV=dev, preparing...
Python 3.13.9
```

- After `exec`, `python` is the only process: the script's shell is gone.
- The setup ran for the one-off command too, and then your command replaced CMD.

### Inspect any image's start command

```bash
for i in nginx:1.27-alpine postgres:17-alpine python:3.13-slim; do
  echo "$i: $(docker image inspect -f 'EP={{json .Config.Entrypoint}} CMD={{json .Config.Cmd}}' $i)"
done
```

```
nginx:1.27-alpine: EP=["/docker-entrypoint.sh"] CMD=["nginx","-g","daemon off;"]
postgres:17-alpine: EP=["docker-entrypoint.sh"] CMD=["postgres"]
python:3.13-slim: EP=null CMD=["python3"]
```

---

## Hands-on lab

1. Build all four examples above. For each, predict the output **before** running it, then check.
2. **Reset trap:** make an image `FROM nginx:1.27-alpine` with only `ENTRYPOINT ["/docker-entrypoint.sh"]`. Run it. It exits right away. Inspect: `Cmd` is now `null`, because setting ENTRYPOINT reset the inherited CMD. Fix it by adding `CMD ["nginx", "-g", "daemon off;"]`.
3. **pipefail:**
   ```dockerfile
   # syntax=docker/dockerfile:1
   FROM debian:bookworm-slim
   RUN false | true && echo "pipe succeeded?!"
   SHELL ["/bin/bash", "-o", "pipefail", "-c"]
   RUN false | true && echo "never printed"
   ```
   Build with `--progress=plain`. The first `RUN` passes. The second fails with `exit code: 1`.
4. **Missing `exec`:** remove `exec` from `entrypoint.sh`, rebuild, run, and `time docker stop ed`. It takes 10 s now (topic 12). Put `exec` back.
5. Clean up: `docker rm -f ed; docker rmi cmdonly mycurl badentry entrydemo`

---

## Where you'll use it

- **Writing app images:** `CMD` in exec form for simple apps; an entrypoint script + `exec "$@"` when setup is needed.
- **Writing tool images** for CI (linters, scanners, deploy tools): `ENTRYPOINT ["tool"]`.
- **Debugging:** `docker run --rm -it --entrypoint sh image` to get inside an image whose entrypoint is the app.
- **Kubernetes:** a Pod's `command:` overrides **ENTRYPOINT** and `args:` overrides **CMD**. People mix this up all the time.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| Your `docker run img arg` arguments are ignored | Shell-form ENTRYPOINT | Use exec form `ENTRYPOINT ["..."]` |
| Container exits at once after you added an ENTRYPOINT | Inherited CMD was reset to empty | Add your own `CMD` |
| `exec /entrypoint.sh: permission denied` / exit 126 | Script isn't executable | `COPY --chmod=755` or `RUN chmod +x` |
| `exec /entrypoint.sh: no such file or directory`, but the file exists | The `#!` interpreter doesn't exist (`#!/bin/bash` on Alpine), or the file has **Windows line endings** (`\r\n`), so the kernel looks for `/bin/sh\r` | Use `#!/bin/sh`; convert with `dos2unix` or set `git config core.autocrlf input`; add `*.sh text eol=lf` to `.gitattributes` |
| `docker stop` takes 10 s with an entrypoint script | Script doesn't `exec` the final command | `exec "$@"` |
| `$HOME` or `$PORT` appears literally in the command | Exec form doesn't expand variables | Expand in the entrypoint script, or use `sh -c` deliberately (and then `exec` inside it) |
| Kubernetes `command:` bypasses your setup script | `command` replaces ENTRYPOINT | Use `args:` to change only CMD |

---

## Quick recap

- Final command = **ENTRYPOINT + CMD**. ENTRYPOINT is the fixed program. CMD is the default arguments (or the default command if there's no ENTRYPOINT).
- `docker run img args` replaces CMD. `--entrypoint` replaces ENTRYPOINT (and resets CMD).
- Use **exec form** for both. Shell-form ENTRYPOINT ignores CMD and runtime arguments.
- Setting ENTRYPOINT resets the base image's CMD.
- Entrypoint scripts do setup and then `exec "$@"`. `STOPSIGNAL` sets the graceful stop signal. `SHELL ["/bin/bash","-o","pipefail","-c"]` makes pipes fail properly.

**Interview questions**

1. **Q: What's the difference between CMD and ENTRYPOINT?**
   A: ENTRYPOINT defines the executable that always runs. CMD provides default arguments to it, or the default command if no ENTRYPOINT exists. Arguments to `docker run` replace CMD. Replacing ENTRYPOINT needs `--entrypoint`.

2. **Q: Why do entrypoint scripts end with `exec "$@"`?**
   A: So the shell replaces itself with the final command. The app becomes PID 1, receives signals like SIGTERM directly, and no leftover shell sits in between.

3. **Q: In Kubernetes, which fields map to ENTRYPOINT and CMD?**
   A: `command` overrides ENTRYPOINT and `args` overrides CMD.
