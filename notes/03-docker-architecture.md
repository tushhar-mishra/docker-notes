# 03 · Docker's Architecture

> **Level:** ★★ intermediate
>
> **You will learn:** what really runs when you type a `docker` command. That's the chain CLI → `dockerd` → `containerd` → `containerd-shim` → `runc` → your process. You'll also learn what the Docker socket is, what OCI standards are, what a registry is, and how Docker Engine, Docker Desktop and Podman differ.
>
> **You need:** topic 02 (namespaces, cgroups, root filesystem).

---

## What it is

"Docker" is not one program. It's a **chain of programs**, each with one job:

```
 you
  │
  ▼
docker (CLI)                 ← the command you type. Just a client.
  │   REST API over a Unix socket: /var/run/docker.sock
  ▼
dockerd (Docker daemon)      ← the brain: images, networks, volumes, builds, API
  │   gRPC over /run/containerd/containerd.sock
  ▼
containerd                   ← manages container lifecycles and image content
  │   starts one shim per container
  ▼
containerd-shim-runc-v2      ← a small babysitter process, one per container
  │   runs runc once
  ▼
runc                         ← creates the namespaces + cgroups, then exits
  │   execve()
  ▼
your app (PID 1 in the container)
```

New terms:
- **Daemon** = a program that runs in the background all the time, waiting for requests (like `sshd`). `dockerd` is the Docker daemon.
- **CLI** = Command Line Interface, the `docker` program you type.
- **API** (Application Programming Interface) = a defined set of requests one program can send to another. Docker's API is a **REST API** (requests like `GET /containers/json` or `POST /containers/create`, sent over HTTP).
- **Unix socket** = a special file that two programs on the same machine use to talk, like a network connection that doesn't leave the machine. `/var/run/docker.sock` is one.
- **gRPC** = a fast way for programs to call each other's functions over a connection (made by Google).
- **Runtime** = the program that actually starts and runs containers.

---

## Why it exists

Early Docker (2013–2015) was one big program that did everything. That caused problems:
- Restarting or upgrading Docker killed every running container.
- Other tools (like Kubernetes) wanted to run containers without all of Docker's extras.
- Every company wanted a standard, not one vendor's private format.

So Docker was split into pieces, and the low-level pieces were donated to open foundations:
- **runc** went to the **OCI** (Open Container Initiative), 2015.
- **containerd** went to the **CNCF** (Cloud Native Computing Foundation, the foundation that also owns Kubernetes), 2017.

Now each layer can be used by other tools. Kubernetes uses containerd and runc **without dockerd at all** (topic 53).

---

## Analogy

A **restaurant**:
- **You (customer)** = the person typing commands.
- **The waiter** = the `docker` CLI. Takes your order, carries it to the kitchen, brings back the result. Doesn't cook.
- **The head chef** = `dockerd`. Knows all the recipes (images), plans the work, manages the pantry (volumes) and the dining room layout (networks).
- **The kitchen manager** = `containerd`. Makes sure each dish gets started, tracks which dishes are cooking, stores ingredients (image content).
- **A dedicated helper per dish** = the shim. Stays with that one dish the whole time, reports when it's done.
- **The line cook who lights the burner** = `runc`. Sets up the station (namespaces, cgroups), starts the dish, then walks away.

If the head chef takes a break (dockerd restarts), dishes already cooking can keep cooking, because their helpers are still there. That's the `live-restore` feature (topic 34).

---

## How it works inside

### 1. The `docker` CLI is only a client

The `docker` program doesn't create containers. It turns your command into an HTTP request and sends it to the daemon.

`docker ps` becomes:

```
GET /v1.51/containers/json HTTP/1.1
Host: docker
```

It's sent over the Unix socket `/var/run/docker.sock`. The `v1.51` part is the **API version** (different Docker versions speak different API versions; topic 49 covers version mismatch).

Because it's only a client, the CLI can also talk to a daemon **on another machine** (over SSH), set with `DOCKER_HOST` or `docker context` (topic 34).

### 2. The Docker socket is root access

`/var/run/docker.sock` is owned by `root` and the `docker` group:

```
srw-rw---- 1 root docker 0 Nov 17 09:00 /var/run/docker.sock
```

- `s` at the start = it's a socket.
- `rw-rw----` = root and the `docker` group can read and write. Others can't.

**Anyone who can write to this socket controls `dockerd`, and `dockerd` runs as root.** They can start a container that mounts the host's `/` and do anything. So:
- **Being in the `docker` group = being root on that machine.**
- **Mounting the socket into a container gives that container root on the host.**

Remember this. It's a top interview question and a top real-world mistake (topics 40 and 46).

### 3. `dockerd`: the Docker daemon

`dockerd` runs as a systemd service (`docker.service`) as root. It:
- serves the API on the socket,
- manages **images** (pull, build, tag, push),
- manages **networks** (bridges, iptables rules, DNS for containers: Part 5),
- manages **volumes** (Part 4),
- runs **builds** through **BuildKit**, its built-in build engine (topic 16),
- turns your `docker run` options into a **container spec** and asks containerd to run it.

`dockerd` does **not** create namespaces itself. That's lower down.

### 4. `containerd`: the container supervisor

`containerd` is a separate daemon (`containerd.service`). It:
- stores image content and **snapshots** (the prepared filesystem layers; see the note on Engine 29 below),
- creates and tracks containers and their running **tasks** (a task = the running process of a container),
- starts a **shim** for every container.

containerd uses **namespaces of its own** (a naming feature, *not* Linux namespaces) to keep clients apart. Docker's containers live in containerd's `moby` namespace. Kubernetes' live in `k8s.io`.

> **Moby** = the open-source project that Docker Engine is built from.

> **Engine 29 note:** since Docker Engine 29, fresh installs use the **containerd image store**. Images are stored *by containerd* (under `/var/lib/containerd`), not by dockerd's older "graph driver" storage (under `/var/lib/docker/overlay2`). Upgraded machines keep the old storage until you switch. Topics 07 and 48 show both.

### 5. `containerd-shim-runc-v2`: one babysitter per container

For every container, containerd starts a small **shim** process. The shim:
- calls `runc` to create the container,
- becomes the **parent** of your container's process after runc exits (so the container process isn't left without a parent),
- keeps the container's **stdin/stdout/stderr** open, which is how `docker logs` and `docker attach` work,
- collects the **exit code** when the container stops,
- lets containerd and dockerd **restart or upgrade without killing containers**.

That last point is the main reason the shim exists.

### 6. `runc`: the low-level runtime

`runc` is a small command-line tool that follows the **OCI Runtime Specification**. It receives:
- a folder with the root filesystem (the "bundle"),
- a `config.json` describing the process to run, the namespaces, cgroup limits, mounts, capabilities, seccomp profile and so on.

Then it does exactly what you did by hand in topic 02 (new namespaces, cgroup, `pivot_root`, drop capabilities, `execve` your program) and **exits**. runc is not running while your container runs. Only the shim and your process remain.

Other OCI runtimes can replace runc:
- `crun`: written in C, faster and lighter (default in Podman on Fedora/RHEL).
- `runsc` (**gVisor**): adds a user-space kernel for stronger isolation.
- `kata-runtime` (**Kata Containers**): runs each container in a tiny VM.

### 7. OCI: the standards that make it all portable

**OCI = Open Container Initiative**, a Linux Foundation project that keeps three specs:

| Spec | What it defines | Why it matters |
|---|---|---|
| **Image Spec** | How an image is packaged: layers, a config file, a manifest | An image built by Docker runs on Podman, containerd, Kubernetes |
| **Runtime Spec** | The `config.json` + root filesystem bundle a runtime receives | You can swap runc for crun, gVisor or Kata |
| **Distribution Spec** | The HTTP API for pushing and pulling images to and from registries | Any registry (Docker Hub, GHCR, ECR, Harbor) works with any client |

### 8. Registries

A **registry** is a server that stores images and serves them over the OCI Distribution API. Examples:
- **Docker Hub** (`docker.io`), the default registry,
- **GHCR** (GitHub Container Registry, `ghcr.io`),
- **Amazon ECR** (Elastic Container Registry), **Google Artifact Registry**, **Azure ACR** (Azure Container Registry),
- **Harbor**, a self-hosted open-source registry,
- `registry:2`, a simple registry you can run yourself (topic 22).

`docker pull` asks `dockerd` to download from a registry. `docker push` uploads. Topic 06 covers names and tags.

### 9. The full journey of `docker run -d nginx`

1. **CLI:** parses your command, sends `POST /containers/create` to `dockerd` over the socket.
2. **dockerd:** checks whether the `nginx` image is local. If not, pulls it from Docker Hub (layers are downloaded, checked against their SHA-256 hash, and unpacked).
3. **dockerd:** prepares the container: a writable layer on top of the image layers, a network namespace connected to the `docker0` bridge with an IP address, `/etc/hosts`, `/etc/resolv.conf`, mounts.
4. **CLI:** sends `POST /containers/<id>/start`.
5. **dockerd → containerd:** "create and start a task with this OCI spec."
6. **containerd:** starts `containerd-shim-runc-v2` for this container.
7. **shim → runc:** `runc create` then `runc start`. runc sets up namespaces, cgroup, mounts, capabilities, seccomp, then `execve`s `nginx`. runc exits.
8. **nginx** is now PID 1 in its namespaces. Its parent on the host is the shim.
9. **CLI** prints the container ID and returns.

If any step fails, you'll get an error from that layer. Knowing the layers helps you read the error (see "When it breaks").

### 10. Docker Engine vs Docker Desktop vs Podman

| | Docker Engine | Docker Desktop | Podman |
|---|---|---|---|
| What | The daemon + CLI on Linux | A desktop app for Windows/macOS/Linux | A daemonless, Docker-compatible tool from Red Hat |
| Where it runs containers | Directly on the Linux host | In a hidden Linux VM (WSL2 on Windows, a lightweight VM on macOS) | Directly on Linux (or a VM on Mac/Windows) |
| Daemon | Yes, `dockerd` as root | Yes, inside the VM | **No daemon**. Each `podman` command starts containers itself, via `conmon` (its version of the shim) |
| License | Open source (Apache 2.0) | Free for personal use, small businesses, education, open source. **Paid subscription for larger companies** | Open source |
| Used on servers | **Yes**, this is what production uses | No, it's for developer machines | Yes, common on RHEL |

Podman mostly accepts the same commands (`podman run`, `podman build`) and runs the same OCI images. It's **rootless** by default (topic 42). Many people `alias docker=podman`.

### 11. Where Docker stores things on disk

| Path | What's there |
|---|---|
| `/var/run/docker.sock` | The API socket |
| `/var/lib/docker/` | dockerd's data: containers, volumes, networks, (old) image layers in `overlay2/` |
| `/var/lib/containerd/` | containerd's data: image content and snapshots (the default image store since Engine 29) |
| `/etc/docker/daemon.json` | Daemon configuration (topic 34) |
| `/run/containerd/containerd.sock` | containerd's API socket |
| `~/.docker/config.json` | Your CLI config and registry login info |

---

## Commands / files

### See every piece running

```bash
ps -ef --forest | grep -E 'dockerd|containerd|shim|nginx' | grep -v grep
```

(after `docker run -d --name web nginx`)

```
root      1043     1  0 09:00 ?        00:00:05 /usr/bin/containerd
root      1290     1  0 09:00 ?        00:00:12 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root      6120     1  0 10:01 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 3f2a9c1b7e0d... -address /run/containerd/containerd.sock
root      6142  6120  0 10:01 ?        00:00:00  \_ nginx: master process nginx -g daemon off;
message+  6188  6142  0 10:01 ?        00:00:00      \_ nginx: worker process
```

What each line means:
- `containerd` (PID 1043) and `dockerd` (PID 1290): both started by systemd (parent PID 1).
- `dockerd -H fd://` = "listen on the socket systemd hands me" (socket activation; topic 34). `--containerd=...` = where to find containerd.
- `containerd-shim-runc-v2 -namespace moby -id 3f2a...`: the shim for our container, in containerd's `moby` namespace. **Its parent is PID 1, not dockerd.** That's why containers survive a dockerd restart (with `live-restore`).
- `nginx: master process` (PID 6142): **the container's PID 1**, seen with its host PID. Parent = the shim.
- `nginx: worker process`: a child of nginx, running as user `message+` (the host shows the user name for UID 101, which inside the container is `nginx`. User IDs are just numbers, and names come from each side's own `/etc/passwd`. More in topic 25).
- There's **no `runc`** in the list: it exited after starting the container.

### `docker version`: client and server are separate

```bash
docker version
```

```
Client: Docker Engine - Community
 Version:           29.0.1
 API version:       1.52
 Go version:        go1.25.4
 Git commit:        eedd969
 Built:             ...
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          29.0.1
  API version:      1.52 (minimum version 1.44)
  Go version:       go1.25.4
  Git commit:       198b5e3
  Built:            ...
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v2.1.5
  GitCommit:        ...
 runc:
  Version:          1.3.3
  GitCommit:        ...
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

(Your exact version numbers will be newer. What matters is the shape.)

- **Client** = the CLI on your machine. **Server** = the daemon it's connected to. They can be different versions, even on different machines.
- `API version: 1.52 (minimum version 1.44)`: the daemon speaks API 1.52 and refuses clients older than 1.44. Engine 29 raised this minimum, so very old CLIs or tools break (topic 49).
- `Context: default` = which daemon the CLI is talking to (topic 34).
- The **server** section lists every layer: Engine (dockerd), containerd, runc, and **docker-init** (`tini`, the tiny init used by `--init`; topic 12).
- If the Server section shows an error instead, the CLI can't reach the daemon (see "When it breaks").

### Talk to the API directly (proves the CLI is just a client)

```bash
sudo curl -s --unix-socket /var/run/docker.sock http://localhost/version | head -c 200
```

```
{"Platform":{"Name":"Docker Engine - Community"},"Components":[{"Name":"Engine","Version":"29.0.1",...
```

- `--unix-socket` tells curl to send the HTTP request through the socket file instead of over the network.
- The `localhost` in the URL is ignored (it's just required by HTTP syntax).
- This is exactly what the `docker` CLI does. Anything that can write to this socket can do anything Docker can.

```bash
sudo curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json | python3 -m json.tool | head -20
```

This returns the same data as `docker ps`, as JSON (JavaScript Object Notation: a text format of `"key": value` pairs and lists that programs use to exchange data).

### Look at containerd directly with `ctr`

`ctr` is containerd's own debugging CLI (installed with containerd).

```bash
sudo ctr namespaces list
sudo ctr -n moby containers list
sudo ctr -n moby tasks list
```

```
NAME LABELS
moby

CONTAINER                                                           IMAGE    RUNTIME
3f2a9c1b7e0d4a...                                                   -        io.containerd.runc.v2

TASK                                                                PID     STATUS
3f2a9c1b7e0d4a...                                                   6142    RUNNING
```

- `moby` = Docker's containerd namespace.
- The container ID is the same full ID Docker shows.
- `IMAGE -`: with the older graph-driver storage, containerd doesn't know the image name (dockerd handles images). With the containerd image store you'll see image names here too.
- `PID 6142` = the host PID of the container's main process, the same nginx we saw in `ps`.

Never use `ctr` to delete Docker's containers. dockerd won't know, and its state will get out of sync. Use it only to look.

---

## Hands-on lab

(Do this after installing Docker in topic 04. Come back here.)

1. Start a container and watch the process tree:
   ```bash
   docker run -d --name archlab nginx
   ps -ef --forest | grep -A2 containerd-shim | grep -v grep
   ```
   Find the shim, the nginx master and its workers. Note the shim's parent PID is 1.

2. Prove runc is gone:
   ```bash
   pgrep -a runc || echo "runc is not running"
   ```

3. Prove the CLI is just a client:
   ```bash
   sudo curl -s --unix-socket /var/run/docker.sock http://localhost/containers/archlab/json | python3 -m json.tool | grep -m1 '"Pid"'
   docker inspect --format '{{.State.Pid}}' archlab
   ```
   Both show the same PID.

4. Restart dockerd and see the container survive (only if `live-restore` is enabled; otherwise it'll be stopped and that's also a lesson):
   ```bash
   docker info --format '{{.LiveRestoreEnabled}}'
   sudo systemctl restart docker
   docker ps
   ```
   - `false` (the default): containers are stopped when dockerd stops, and restarted only if they have a restart policy (topic 32).
   - `true`: containers keep running through the daemon restart (topic 34).

5. Clean up: `docker rm -f archlab`

---

## Where you'll use it

- **Troubleshooting:** "docker commands hang" → is it the CLI, the daemon or containerd? You'll check `systemctl status docker containerd` and `journalctl -u docker`.
- **Security reviews:** finding containers or CI jobs with `/var/run/docker.sock` mounted, and users in the `docker` group.
- **Kubernetes:** kubelet talks to containerd over CRI (Container Runtime Interface). No dockerd. The same `ctr`/`crictl` knowledge applies (topic 53).
- **Upgrades:** planning Docker upgrades without downtime using `live-restore` (topic 49).
- **Choosing tools:** Docker Engine for servers, Docker Desktop or Docker Engine in WSL2 for laptops (and checking licensing), Podman on RHEL.

---

## When it breaks

| Error | Which layer | Meaning and fix |
|---|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` | CLI → dockerd | dockerd isn't running, or the CLI is pointing at the wrong place. Check `systemctl status docker`, `journalctl -u docker -n 50`, and `echo $DOCKER_HOST` / `docker context ls`. |
| `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` | CLI → socket | Your user can't write to the socket. Use `sudo`, or add the user to the `docker` group **knowing that means root** (topic 04), then log out and back in. |
| `Error response from daemon: client version 1.41 is too old. Minimum supported API version is 1.44` | API version | An old CLI or tool (an old Docker SDK in a CI image, for example) is talking to a new daemon. Upgrade the client. (Topic 49.) |
| `failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: ...` | containerd → runc | runc couldn't build the container. Read the rest of the message: often a missing executable (`exec: "xyz": executable file not found in $PATH`), a bad mount, or a permission problem. |
| `Error response from daemon: ... containerd: connection refused` | dockerd → containerd | containerd is down. `systemctl status containerd`, `journalctl -u containerd`. |
| `docker` commands hang forever | dockerd stuck | Check `journalctl -u docker -f`. Often a full disk, a stuck volume/network driver, or a hung NFS mount. `systemctl restart docker` is the last resort. |

---

## Quick recap

- The chain is: `docker` CLI → (REST API over `/var/run/docker.sock`) → `dockerd` → `containerd` → `containerd-shim-runc-v2` → `runc` → your process.
- runc sets up namespaces and cgroups and then **exits**. The **shim** stays as the parent of your process, which lets the daemons restart.
- **Access to the Docker socket = root on the host.**
- **OCI** specs (image, runtime, distribution) make images and runtimes portable across Docker, Podman, containerd and Kubernetes.
- Docker Engine runs natively on Linux servers. Docker Desktop runs Engine inside a hidden Linux VM. Podman is daemonless and rootless by default.

**Interview questions**

1. **Q: What happens when you run `docker run nginx`?**
   A: The CLI sends API requests to dockerd. dockerd pulls the image if needed, prepares the filesystem and network, and asks containerd to start a task. containerd starts a shim, which runs runc. runc creates namespaces and the cgroup, pivots to the image's root filesystem and execs nginx, then exits. The shim stays as nginx's parent.

2. **Q: Why is adding a user to the `docker` group a security concern?**
   A: It gives them write access to the Docker socket. dockerd runs as root, so they can start a container that mounts the host filesystem with full privileges. That's equal to root access.

3. **Q: What is the role of containerd-shim?**
   A: It's a small per-container parent process. It keeps the container's stdio and exit status, and it lets containerd and dockerd restart or upgrade without killing running containers.
