# 05 · Your First Container

> **Level:** ★ foundation
>
> **You will learn:** exactly what happens, step by step and layer by layer, when you type `docker run hello-world`. Then you'll run your first interactive container and see that the container stays behind after it exits.
>
> **You need:** topics 02–04 (namespaces/cgroups, the CLI → dockerd → containerd → shim → runc chain, a working install).

---

## What it is

`docker run` is the command that **creates and starts a container from an image**. It's really four steps in one:

1. **pull** the image if it isn't already on the machine,
2. **create** a container from it,
3. **start** that container,
4. **attach** your terminal to its output (unless you say `-d` for "detached").

`hello-world` is a tiny test image made by Docker. Its whole job is to print a message and exit, proving that every layer of your install works.

Quick definitions (each one gets a full topic later):
- **Image** = a read-only package: a filesystem (files and folders) plus some settings, like which program to run. Topic 06–07.
- **Container** = a running (or stopped) instance of an image: a process with namespaces, cgroups and a root filesystem made from the image, plus a small writable layer on top. Topic 08.
- **Registry** = a server that stores images. The default is **Docker Hub**. Topic 06.
- **Tag** = a version label on an image, like `latest` or `1.27`. Topic 06.

---

## Why it exists

You need one command that checks the whole chain at once: the CLI, the socket, dockerd, the network path to the registry, containerd, the shim, runc, the kernel features, and output streaming. If `hello-world` works, all of those work. If it fails, the error tells you which layer is broken.

---

## Analogy

Ordering food through a delivery app the first time with a **test order**. You order a bottle of water (the smallest item). If it arrives, you've proven the app, the payment, the restaurant, the driver and your address all work. Then you order real food.

---

## How it works inside

Run it:

```bash
docker run hello-world
```

First-time output:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
17eec7bbc9d7: Pull complete
Digest: sha256:56433a6be3fda188089fb548eae3d91df3ed0d6589f7c2656121b911198df065
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

(Your layer ID and digest will be different. Docker updates the image.)

Now step by step, with what each layer does.

### Step 1: The CLI expands the name

You typed `hello-world`. The CLI expands it to the full name:

```
docker.io/library/hello-world:latest
│         │       │           │
│         │       │           └─ tag: none given → "latest"
│         │       └─ repository name
│         └─ "library" = Docker's namespace for official images
└─ registry: none given → Docker Hub (docker.io)
```

Topic 06 explains these names fully.

### Step 2: The CLI asks dockerd to create a container

The CLI sends `POST /containers/create` with the image name over `/var/run/docker.sock`. dockerd answers "no such image". So the CLI prints:

```
Unable to find image 'hello-world:latest' locally
```

…and asks dockerd to pull it (`POST /images/create?fromImage=hello-world&tag=latest`).

### Step 3: dockerd pulls the image from Docker Hub

This is several HTTPS requests to the registry (all following the OCI Distribution spec from topic 03):

1. **Get a token:** Docker Hub requires a token even for public images. dockerd asks `auth.docker.io` for an anonymous pull token. (Anonymous pulls are rate-limited; topic 22.)
2. **Get the index:** `GET /v2/library/hello-world/manifests/latest`. The registry returns an **image index**, a list of variants of this image for different CPU types (amd64, arm64, arm/v7, …).
3. **Choose your platform:** your machine is `linux/amd64`, so dockerd picks that variant's **manifest** (a small JSON file listing the image's config and layers). That's why the output says `(amd64)`.
4. **Get the config:** a JSON file with settings like "run `/hello` when starting".
5. **Get the layers:** each layer is a compressed `.tar` file of files. `hello-world` has just one layer, `17eec7bbc9d7…`. Each one is downloaded, its **SHA-256 checksum** is verified (so you know nothing was changed in transit), and it's unpacked.
6. `Digest: sha256:56433a…` = the unique fingerprint of the index. The same digest always means exactly the same image (topic 06).

### Step 4: dockerd creates the container

- Picks a random **container ID** (64 hex characters) and a random **name** (like `elegant_hopper`), because you didn't give one.
- Prepares the root filesystem: the image's layer(s), read-only, plus a new empty **writable layer** on top, combined with the overlay filesystem (topic 07).
- Prepares networking: a new network namespace, a virtual cable to the `docker0` bridge, an IP address (Part 5).
- Builds an OCI runtime spec (`config.json`): which namespaces, which cgroup limits (none here), which mounts, which capabilities, the command `/hello`.

### Step 5: start → containerd → shim → runc

- The CLI sends `POST /containers/<id>/attach` (so it can receive output) and then `POST /containers/<id>/start`.
- dockerd asks containerd to start a task. containerd starts a `containerd-shim-runc-v2`. The shim runs `runc create` / `runc start`.
- runc creates the namespaces and the cgroup (topic 02), switches root with `pivot_root`, drops capabilities, applies seccomp, then `execve("/hello")`.
- runc exits. The shim stays as the parent of `/hello`.

### Step 6: output flows back to you

- `/hello` writes its message to **stdout** (standard output, the normal output stream of a process).
- The shim holds the other end of that stream and passes the bytes to dockerd.
- dockerd saves them to the container's log file (that's how `docker logs` can show them later; topic 33) **and** streams them to the attached CLI.
- The CLI prints them in your terminal.

### Step 7: the program exits, the container stays

- `/hello` finishes and exits with code `0` (success).
- PID 1 of the container is gone, so the PID namespace is torn down, the cgroup is emptied, and the shim reports the exit code to containerd and then exits.
- The container moves to the **Exited** state. **It isn't deleted.** Its writable layer, its logs and its settings stay on disk until you remove it.
- The CLI returns with the same exit code the container had.

### What's inside the hello-world image?

Almost nothing:

```bash
docker image inspect hello-world --format '{{.Config.Cmd}} {{.Size}}'
```

```
[/hello] 10072
```

- `[/hello]` = the command run at start.
- `10072` = about 10 KB.

It's built `FROM scratch` (an empty image, topic 19) with one **static binary** (a program that includes all its libraries inside itself, so it needs no other files). There's no shell, no `ls`, no `/bin`. The container's root filesystem has exactly one file. That's also why you can't `docker exec` into it with a shell.

---

## Commands / files

### See the stopped container

```bash
docker ps
docker ps -a
```

```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

CONTAINER ID   IMAGE         COMMAND    CREATED          STATUS                      PORTS     NAMES
a1b2c3d4e5f6   hello-world   "/hello"   2 minutes ago    Exited (0) 2 minutes ago              elegant_hopper
```

- `docker ps` = only **running** containers. Empty, because hello-world already exited.
- `docker ps -a` = **all** containers, including stopped ones.
- `CONTAINER ID` = the first 12 characters of the full 64-character ID. You can use any unique prefix in commands, even `a1b`.
- `COMMAND "/hello"` = the program it ran.
- `STATUS Exited (0)` = stopped, exit code 0 = success.
- `NAMES elegant_hopper` = a random adjective_scientist name, because you didn't use `--name`.

### Run it again

```bash
docker run hello-world
docker ps -a
```

No pull this time (the image is local). And `docker ps -a` now shows **two** containers. Every `docker run` creates a **new** container. This is how people end up with hundreds of dead containers.

### Auto-delete with `--rm`

```bash
docker run --rm hello-world
```

- `--rm` = delete the container automatically when it exits. Use it for one-off commands.

### See the logs of the stopped container

```bash
docker logs elegant_hopper | head -3
```

```

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

The output was saved to the log, even after the container stopped.

### Remove containers

```bash
docker rm elegant_hopper
docker container prune
```

```
elegant_hopper

WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Deleted Containers:
4f5e6d7c8b9a...

Total reclaimed space: 0B
```

- `docker rm <name or id>` = delete one stopped container.
- `docker container prune` = delete **all** stopped containers (topic 13).

### Your first interactive container

```bash
docker run -it --rm ubuntu bash
```

- `-i` = keep **stdin** (standard input, what you type) open.
- `-t` = give it a **TTY** (a terminal device), so you get a prompt and colors.
- `ubuntu` = the image. `bash` = the command to run instead of the image's default.

```
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
b08e2ff4391e: Pull complete
Digest: sha256:...
Status: Downloaded newer image for ubuntu:latest
root@7d3f2a1c9b8e:/#
```

- The prompt `root@7d3f2a1c9b8e` = you're root, and the hostname is the container's short ID (UTS namespace, topic 02).

Look around:

```bash
cat /etc/os-release | head -2
ps aux
hostname
exit
```

```
PRETTY_NAME="Ubuntu 24.04.3 LTS"
NAME="Ubuntu"
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4588  3840 pts/0    Ss   10:20   0:00 bash
root           9  0.0  0.0   7888  3968 pts/0    R+   10:20   0:00 ps aux
7d3f2a1c9b8e
```

- `bash` is PID 1. Only two processes. Exactly what you built by hand in topic 02.
- `exit` ends bash → PID 1 exits → the container stops → `--rm` deletes it.

---

## Hands-on lab

1. Watch the API calls happen. In terminal 1:
   ```bash
   docker events
   ```
   In terminal 2:
   ```bash
   docker run --rm hello-world > /dev/null
   ```
   Terminal 1 shows:
   ```
   2026-11-17T10:30:01.1+05:30 container create a1b2... (image=hello-world, name=busy_turing)
   2026-11-17T10:30:01.2+05:30 container attach a1b2... (image=hello-world, name=busy_turing)
   2026-11-17T10:30:01.3+05:30 network connect 9f8e... (container=a1b2..., name=bridge, type=bridge)
   2026-11-17T10:30:01.5+05:30 container start a1b2... (image=hello-world, name=busy_turing)
   2026-11-17T10:30:01.6+05:30 container die a1b2... (exitCode=0, image=hello-world, name=busy_turing)
   2026-11-17T10:30:01.7+05:30 network disconnect 9f8e... (container=a1b2..., name=bridge, type=bridge)
   2026-11-17T10:30:01.8+05:30 container destroy a1b2... (image=hello-world, name=busy_turing)
   ```
   Match each event to the steps above: create, attach, network connect, start, die (with exit code), disconnect, destroy (because of `--rm`). Stop `docker events` with Ctrl+C.

2. See the exit code pass through:
   ```bash
   docker run --rm ubuntu bash -c 'exit 42'; echo "exit code: $?"
   ```
   ```
   exit code: 42
   ```
   `$?` = the exit code of the last command. The container's exit code becomes `docker run`'s exit code. Scripts and CI pipelines depend on this.

3. Prove a new container is created each time:
   ```bash
   docker run ubuntu touch /i-was-here
   docker run ubuntu ls /i-was-here
   ```
   ```
   ls: cannot access '/i-was-here': No such file or directory
   ```
   The second `run` is a fresh container with a fresh writable layer. The file lives in the first (stopped) container's layer. (Topic 23 explains this fully.)

4. Clean up: `docker container prune -f`

---

## Where you'll use it

- **After every install or upgrade:** `docker run --rm hello-world` is the standard smoke test (a **smoke test** = a quick check that the basics work).
- **One-off tools:** `docker run --rm -it <image> <command>` runs a tool without installing it on the host (for example a specific version of `terraform` or `psql`).
- **CI pipelines:** running tests inside a container and using its exit code to pass or fail the build.
- **Debugging registries/network:** if `hello-world` can't pull, the problem is DNS, a proxy, a firewall or a rate limit, before any of your app's own issues.

---

## When it breaks

| Error | Which step failed | Meaning and fix |
|---|---|---|
| `Cannot connect to the Docker daemon...` | Step 2 | dockerd is down or wrong context. `systemctl status docker` (topic 04). |
| `Error response from daemon: Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io: no such host` | Step 3 (DNS) | The **daemon** can't resolve the registry name. Check DNS on the host (`resolvectl status`, `/etc/resolv.conf`). |
| `...: net/http: request canceled while waiting for connection` / `i/o timeout` | Step 3 (network) | Firewall or proxy. Behind a corporate proxy, **dockerd** needs proxy settings, not just your shell (topic 34). |
| `toomanyrequests: You have reached your unauthenticated pull rate limit` | Step 3 (rate limit) | Docker Hub limits anonymous pulls. `docker login`, use a mirror, or your own registry (topic 22). |
| `no matching manifest for linux/arm64/v8 in the manifest list entries` | Step 3 (platform) | The image has no variant for your CPU type (topic 21). |
| `docker: Error response from daemon: failed to create task ... exec: "bash": executable file not found in $PATH` | Step 5 (runc) | The command you gave doesn't exist **inside the image**. E.g. Alpine has `sh`, not `bash`. |
| `the input device is not a TTY` | Your terminal | You used `-t` where there's no terminal (e.g. a CI job or piped input). Drop `-t`, keep `-i` if you pipe input. |
| Container exits immediately with `Exited (0)` and you expected it to keep running | Step 7 | The main process finished. A container lives only as long as its PID 1 (topic 08). |

---

## Quick recap

- `docker run` = pull (if needed) + create + start + attach.
- Short names expand to `docker.io/library/<name>:latest`. The daemon, not the CLI, downloads the image, and it checks every piece by its SHA-256 hash.
- Output flows container stdout → shim → dockerd (saved to the log) → CLI → your terminal.
- When PID 1 exits, the container **stops but isn't deleted**. Use `docker ps -a` to see it and `--rm` to auto-delete.
- The container's exit code becomes `docker run`'s exit code.

**Interview questions**

1. **Q: Why does `docker ps` show nothing after you ran a container?**
   A: `docker ps` only shows running containers. The container's main process exited, so it's stopped. `docker ps -a` shows it.

2. **Q: What does `docker run` do if the image isn't present locally?**
   A: The daemon pulls it from the registry: it gets a token, the index, the manifest for its platform, the config and the layers, verifies their digests, and unpacks them. Then it creates and starts the container.

3. **Q: What's the difference between `docker run` and `docker start`?**
   A: `docker run` creates a **new** container from an image (and starts it). `docker start` starts an **existing** stopped container again.
