# Docker: From Zero to Production

One learning path, in reading order. Each topic only uses ideas from topics before it. When a later idea has to appear early, it is explained right there in a line or two.

**Every topic has the same shape:** What it is · Why it exists · Analogy · How it works inside (including what Linux is doing) · Commands / files (real output, explained) · Hands-on lab · Where you'll use it · When it breaks · Quick recap + interview questions.

**Current Docker only (checked November 2026):** Docker Engine 29.x, Compose v2 (`docker compose`), BuildKit, `buildx`, multi-stage builds, the containerd image store (the default for new installs since Engine 29). Old ways (`docker-compose` v1, the legacy builder, `--link`, `version:` in compose files, Docker Content Trust) are marked **deprecated** where they come up and are never taught as the main way.

**Your machine:** you're on Windows. Topic 04 sets up a real Linux Docker Engine inside WSL2 (Windows Subsystem for Linux), so every "look from the host" lab works exactly as it would on a production Linux server.

**Sample outputs:** they show the *shape* of real output. Version numbers, IDs, hashes and sizes are illustrative. Your own machine is the source of truth, so run every lab.

**Levels:** ★ foundation (read in one sitting) · ★★ intermediate (read + do the lab) · ★★★ advanced (read slowly, do the lab twice, come back later). Topics with a letter (e.g. `39a`) were inserted later in reading order; read them right after the topic with the same number.

---

## Part 1 · Why containers exist

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 01 | ★ | [Why containers](notes/01-why-containers.md) | "Works on my machine", dependency conflicts, slow servers; physical → VM → container; what Docker added |
| 02 | ★★ | [What a container really is](notes/02-what-a-container-really-is.md) | A container is just a Linux process: namespaces (what it sees), cgroups (what it can use), its own root filesystem; why it isn't a VM |
| 03 | ★★ | [Docker's architecture](notes/03-docker-architecture.md) | CLI → `dockerd` → containerd → shim → runc; the Docker socket; OCI (Open Container Initiative) specs; registries; Docker Engine vs Docker Desktop; Podman |
| 04 | ★ | [Installing Docker and setting up your lab](notes/04-installing-docker.md) | Docker Engine in WSL2 Ubuntu (your lab), on Ubuntu/RHEL servers, Docker Desktop and its hidden VM, `docker version` / `docker info`, the `docker` group (= root) |
| 05 | ★ | [Your first container](notes/05-your-first-container.md) | What happens, step by step, when you type `docker run hello-world` |

## Part 2 · Images and containers

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 06 | ★ | [Images, tags and registries](notes/06-images-tags-and-registries.md) | Image names (registry/repository:tag), tags vs digests, Docker Hub, `pull`, `images`, `rmi`, official vs community images |
| 07 | ★★ | [Image layers, manifests and the overlay filesystem](notes/07-image-layers-and-overlayfs.md) | Layers, manifest + config + index, SHA-256 content addressing, `docker history`, overlay (lower/upper/merged), copy-on-write, whiteouts, overlay2 driver vs containerd snapshotter |
| 08 | ★ | [The container lifecycle](notes/08-container-lifecycle.md) | create → start → pause → stop/kill → rm; states; exit codes (0, 1, 125–127, 137, 143); `--rm`, `wait`, `rename`, `update` |
| 09 | ★ | [Running containers: the essential options](notes/09-running-containers.md) | `-d`, `-it`, `--name`, `-e`, `-w`, `-u`, `--hostname`, `--entrypoint`, overriding the command, `attach` and detach keys |
| 10 | ★ | [Inspecting containers](notes/10-inspecting-containers.md) | `ps`, `logs`, `inspect` (with `--format`), `exec`, `top`, `stats`, `diff`, `cp`, `port`, `events` |
| 11 | ★★★ | [A container from the inside](notes/11-container-from-the-inside.md) | Find the container's PID on the host, compare `/proc/PID/ns`, enter with `nsenter`, read its cgroup files and overlay mount |
| 12 | ★★ | [PID 1, signals and graceful shutdown](notes/12-pid1-signals-and-shutdown.md) | Why `docker stop` takes 10 s, SIGTERM → SIGKILL, shell form vs exec form, zombies, `--init` (tini) |
| 13 | ★ | [Cleaning up and disk usage](notes/13-cleanup-and-disk-usage.md) | `docker system df`, dangling images, `prune` commands, what is safe to delete |

## Part 3 · Building images

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 14 | ★ | [Dockerfile basics](notes/14-dockerfile-basics.md) | `FROM`, `RUN`, `COPY`, `ADD`, `WORKDIR`, `CMD`, `EXPOSE`; the build context; `.dockerignore`; `# syntax=`; heredocs; `docker init`; why not `docker commit` |
| 15 | ★★ | [CMD vs ENTRYPOINT](notes/15-cmd-vs-entrypoint.md) | Exec vs shell form, how they combine, overriding them, entrypoint scripts with `exec "$@"`, `STOPSIGNAL`, `SHELL` |
| 16 | ★★ | [BuildKit and the build cache](notes/16-buildkit-and-build-cache.md) | How BuildKit runs a build (a graph, not a list), cache invalidation rules, ordering for fast builds, cache mounts, reading build output |
| 17 | ★★ | [Multi-stage builds](notes/17-multi-stage-builds.md) | Build in one stage, ship only the result; `--target`; Go, Java, Node and Python examples |
| 18 | ★★ | [ARG, ENV, labels and build secrets](notes/18-arg-env-labels-secrets.md) | ARG vs ENV, scope rules, `LABEL` and OCI annotations, `--secret` and `--ssh` mounts (never secrets in layers) |
| 19 | ★★ | [Choosing a base image](notes/19-choosing-base-images.md) | Full vs slim vs Alpine (musl) vs distroless vs scratch vs UBI vs hardened images; size, security and debuggability trade-offs |
| 20 | ★★ | [Dockerfile best practices](notes/20-dockerfile-best-practices.md) | Non-root `USER`, pinning versions and digests, `COPY --chown/--chmod/--link`, small layers, reproducible builds, `docker build --check`, hadolint, `dive` |
| 21 | ★★★ | [buildx and multi-platform images](notes/21-buildx-and-multi-platform.md) | Builders and drivers, `--platform`, amd64 vs arm64, QEMU emulation, manifest lists, `docker buildx bake` |
| 22 | ★★ | [Tagging, pushing and private registries](notes/22-tagging-pushing-registries.md) | Tag strategy (semver, git SHA, never only `latest`), `login`, credential helpers, Docker Hub rate limits, GHCR, your own registry, pulling by digest, `save`/`load` |

## Part 4 · Data

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 23 | ★ | [Why container data disappears](notes/23-container-data.md) | The writable layer, what `docker rm` really deletes, the three ways to keep data |
| 24 | ★★ | [Volumes](notes/24-volumes.md) | Named and anonymous volumes, the `VOLUME` instruction, where volumes live, initialising from the image, backup and restore, volume drivers |
| 25 | ★★ | [Bind mounts, tmpfs and permissions](notes/25-bind-mounts-tmpfs-permissions.md) | Bind mounts, UID/GID mismatches, SELinux `:z`/`:Z`, tmpfs, `--mount` syntax, read-only root filesystems |

## Part 5 · Networking

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 26 | ★★★ | [Linux networking recap for containers](notes/26-linux-networking-recap.md) | Network namespaces, veth pairs, Linux bridges, routing, NAT with iptables/nftables, built by hand |
| 27 | ★★ | [Docker network drivers](notes/27-docker-network-drivers.md) | Default bridge (`docker0`) vs user-defined bridges, `host`, `none`, macvlan/ipvlan, `container:` mode, IPv6, subnet clashes with your company network |
| 28 | ★★★ | [Publishing ports](notes/28-publishing-ports.md) | `-p`, DNAT rules and `docker-proxy`, binding to 127.0.0.1, why Docker bypasses ufw/firewalld, the `DOCKER-USER` chain, the nftables backend, `host.docker.internal` |
| 29 | ★★ | [Container DNS and service discovery](notes/29-container-dns.md) | The embedded DNS server at 127.0.0.11, names and aliases, round-robin, `/etc/resolv.conf` and `/etc/hosts` inside containers, `--dns`, `--add-host` |
| 30 | ★★★ | [Network troubleshooting](notes/30-network-troubleshooting.md) | Can't connect / can't resolve / connection refused / hangs: a step-by-step method with `nsenter`, netshoot, `tcpdump` on veth, MTU problems |

## Part 6 · Runtime behaviour

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 31 | ★★★ | [Resource limits](notes/31-resource-limits.md) | cgroups v2 in depth; `--memory`, `--cpus`, `--cpuset-cpus`, `--pids-limit`; OOM kills and exit 137; CPU throttling; reading the cgroup files; `--gpus` in one paragraph |
| 32 | ★★ | [Restart policies and health checks](notes/32-restart-and-healthchecks.md) | `no`/`on-failure`/`always`/`unless-stopped`, restart back-off, `HEALTHCHECK` states, why plain Docker doesn't restart unhealthy containers |
| 33 | ★★ | [Logging](notes/33-logging.md) | stdout/stderr → logging driver; `json-file`, `local`, `journald`, `fluentd`; log rotation; why disks fill with logs |
| 34 | ★★ | [Configuring the Docker daemon](notes/34-daemon-configuration.md) | `daemon.json`, the systemd unit and socket, `data-root`, default log settings, `live-restore`, `default-address-pools`, registry mirrors, HTTP proxies, remote Docker over SSH and `docker context` |

## Part 7 · Docker Compose

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 35 | ★ | [Compose basics](notes/35-compose-basics.md) | `compose.yaml`, services, `docker compose up/down/ps/logs/exec/run/config`, project names, what Compose actually creates (labels) |
| 36 | ★★ | [Compose: networks, volumes, environment and secrets](notes/36-compose-networks-volumes-env.md) | `.env` and interpolation, `env_file`, named volumes, multiple networks, `secrets` and `configs`, profiles |
| 37 | ★★ | [Compose: dependencies, health and restarts](notes/37-compose-dependencies-health.md) | `depends_on` with `condition: service_healthy`, healthchecks, restart policies, startup order vs readiness, `up --wait` |
| 38 | ★★ | [Compose for development](notes/38-compose-for-development.md) | Bind-mounted source, `docker compose watch`, override files, `include`, `extends`, building with Compose |
| 39 | ★★ | [Compose on a single production host](notes/39-compose-on-a-server.md) | Pulling instead of building, limits, logging, scaling, updates with little downtime, running under systemd, when Compose stops being enough |
| 39a | ★★ | [TLS and reverse proxies in production](notes/39a-tls-and-reverse-proxies.md) | Certificates and chains, the TLS handshake and SNI, Traefik/Caddy/nginx, ACME/Let's Encrypt (HTTP-01, DNS-01), `X-Forwarded-*`, 502 vs 504, private CAs in containers, mTLS, `openssl s_client` |

## Part 8 · Security

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 40 | ★★ | [Docker's security model](notes/40-security-model.md) | What isolates containers and what doesn't (one shared kernel), the Docker socket = root, attack paths, CIS Benchmark and docker-bench-security |
| 41 | ★★★ | [Users, capabilities and no-new-privileges](notes/41-users-and-capabilities.md) | Root in a container, `USER`, Linux capabilities and their five sets, `--cap-drop/--cap-add`, why ping no longer needs `NET_RAW`, `no-new-privileges`, `--device`, `--sysctl` |
| 41a | ★★★ | [Seccomp, AppArmor/SELinux and `--privileged`](notes/41a-seccomp-lsms-and-privileged.md) | Syscall filtering and custom profiles, LSMs (`docker-default`, `container_t` + MCS), masked paths, finding which layer said no, what `--privileged` turns off |
| 42 | ★★★ | [User namespaces and rootless Docker](notes/42-userns-and-rootless.md) | `userns-remap`, rootless mode, `/etc/subuid`, what changes for ports, networking and storage, comparison with Podman |
| 43 | ★★ | [Image supply-chain security](notes/43-image-supply-chain.md) | Vulnerability scanning (Docker Scout, Trivy), SBOM (Software Bill of Materials), provenance attestations, signing with cosign, digest pinning, pinning your tools |
| 44 | ★★ | [Handling secrets](notes/44-handling-secrets.md) | Why not `ENV` or image layers, build secrets, Compose secrets, runtime files, external secret stores |
| 44a | ★★★ | [Runtime security and incident response](notes/44a-runtime-security-and-incident-response.md) | Falco (eBPF syscall rules), drift detection, auditing the Docker API with auditd, indicators of compromise, the full response procedure: contain → preserve → investigate → eradicate → recover |

## Part 9 · Production operations

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 45 | ★★ | [Production-ready containers](notes/45-production-ready-containers.md) | Config from the environment, logs to stdout, graceful shutdown, health endpoints, read-only root, limits, time zones: a full checklist |
| 46 | ★★★ | [Building images in CI pipelines](notes/46-building-in-ci.md) | Build → test → scan → tag → push, registry cache export, OIDC, SHA-pinned actions, Docker-in-Docker vs socket mount vs rootless BuildKit, and their risks |
| 46a | ★★★ | [Registry operations](notes/46a-registry-operations.md) | How registries store blobs/manifests/tags, why deletes don't free space until garbage collection, retention policies (Harbor, ECR lifecycle), Harbor (RBAC, robot accounts, scanning, proxy cache, replication), pull-through caches, HA and monitoring |
| 47 | ★★ | [Monitoring containers](notes/47-monitoring-containers.md) | `docker stats` and events, cAdvisor, the daemon's Prometheus metrics, shipping logs, what to alert on |
| 48 | ★★★ | [Storage internals and disk problems](notes/48-storage-and-disk-problems.md) | `/var/lib/docker` and `/var/lib/containerd` layout, build cache growth, "no space left on device" (blocks and inodes), moving `data-root` |
| 48a | ★★★ | [Host tuning for container hosts](notes/48a-host-tuning-for-container-hosts.md) | Namespaced vs host-wide sysctls, conntrack and neighbour tables, inotify, open files, ephemeral ports, `vm.max_map_count`, Redis overcommit/THP, persistence with sysctl.d, recognising and leaving cgroup v1 |
| 49 | ★★ | [Upgrading Docker and live-restore](notes/49-upgrades-and-live-restore.md) | Engine/containerd upgrades, what happens to running containers, `live-restore`, API version skew, migrating to the containerd image store |

## Part 10 · Debugging

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 50 | ★★ | [A debugging method for containers](notes/50-debugging-method.md) | Won't start / exits at once / restarts in a loop / can't connect / slow / OOM-killed: where to look for each |
| 51 | ★★★ | [Debugging tools](notes/51-debugging-tools.md) | Sidecar debug containers (`--pid`/`--network container:`), `docker debug`, `nsenter` + `strace` from the host, `ctr` and `nerdctl` |
| 52 | ★★★ | [Production scenarios](notes/52-production-scenarios.md) | Break-fix labs: full disk, OOM loop, wrong port binding, DNS failure, permission denied on a volume, zombie PID 1, subnet clash, and more |

## Part 11 · Bridge to Kubernetes, interviews and next steps

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 53 | ★★ | [From Docker to Kubernetes](notes/53-docker-to-kubernetes.md) | Why one host isn't enough, Swarm in one paragraph, CRI (Container Runtime Interface), the dockershim removal, `crictl`, how Docker ideas map to Pods, Deployments, Services, probes, PersistentVolumes and `CrashLoopBackOff` |
| 54 | ★★ | [Interview question bank](notes/54-interview-question-bank.md) | 55 questions across every part, with model answers and follow-ups |
| 54a | ★★★ | [Senior-level scenario interviews](notes/54a-senior-scenario-interviews.md) | 8 production incidents (keepalive 502s, CPU throttling, 5-second DNS, native-memory OOM…), 5 system designs, 5 judgement calls, 2 hands-on reviews; the S.C.O.P.E. and R.A.D.I.O.S. answer frameworks |
| 55 | ★ | [What to learn next](notes/55-what-to-learn-next.md) | Where Docker skills lead, with capstone projects |
| 56 | ★ | [Command cheat sheet](notes/56-cheat-sheet.md) | Every command from these notes on one page, grouped by task |
