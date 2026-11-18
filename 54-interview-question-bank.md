# 54 · Interview Question Bank

> **Level:** ★★ intermediate
>
> **You will learn:** how Docker is tested in DevOps/SRE interviews, with **questions across every part**, model answers you can say in under a minute, the **follow-up** an interviewer is likely to ask next, and the topic to re-read if you're unsure.
>
> **You need:** topics 01–53.

---

## What it is

A self-test, arranged in the same order as the notes. Each entry has:
- **Q**: the question as it's usually asked,
- **A**: a short, correct answer (what to say),
- **Follow-up**: where the interviewer goes next (be ready),
- **→ topic**: where to revise.

## Why it exists

Interviewers rarely ask for definitions alone. They ask **why**, **how it works underneath**, and **what you'd do when it breaks**. These notes were built for exactly that. This bank lets you check you can say it out loud, briefly.

## Analogy

A **flight simulator** before the real flight: the same controls and situations, with no consequences for mistakes.

## How it works inside: how to answer well

1. **Start with one sentence** that directly answers the question.
2. **Add the mechanism** ("underneath, Docker uses…").
3. **Add a production angle** ("in production, this matters because…" or "I'd check…").
4. Stop. Let them ask the follow-up.

Practical questions ("this container keeps restarting, what do you do?") → describe your **method** (topic 50), naming the commands.

---

## Commands / files: the questions

### Part 1: Foundations

1. **Q: What problem do containers solve?**
   **A:** They package an app with its user-space dependencies so it runs the same everywhere, isolate apps from each other on one host, start in seconds, and enable immutable, replace-don't-patch deployments.
   **Follow-up:** "What don't they solve?" → kernel differences, CPU architecture, external dependencies. → 01

2. **Q: Container vs VM?**
   **A:** A VM virtualises hardware and runs its own kernel. A container is an isolated process sharing the host kernel. Containers are lighter and faster, VMs isolate more strongly.
   **Follow-up:** "Can you run a Windows container on Linux?" → no, the kernel is shared. → 01

3. **Q: What is a container, technically?**
   **A:** A normal Linux process with namespaces (what it sees), cgroups (what it can use), and its own root filesystem from the image via pivot_root, plus capabilities, seccomp and LSM restrictions.
   **Follow-up:** "Name the namespaces." → mnt, pid, net, uts, ipc, user, cgroup, time. → 02

4. **Q: Walk me through `docker run nginx`.**
   **A:** CLI → REST over docker.sock → dockerd pulls the image if needed, prepares the overlay rootfs and network → containerd → shim → runc creates namespaces/cgroup, pivots root, drops capabilities, execs nginx and exits. The shim stays as parent.
   **Follow-up:** "Why does the shim exist?" → so the daemons can restart without killing containers; it holds stdio and the exit code. → 03, 05

5. **Q: What is OCI?**
   **A:** The Open Container Initiative: standards for image format, runtime config and distribution API, so images and runtimes are interchangeable (Docker, Podman, containerd, Kubernetes).
   **Follow-up:** "Name an alternative runtime to runc." → crun, gVisor (runsc), Kata. → 03

6. **Q: Why is the `docker` group dangerous?**
   **A:** Access to the Docker socket controls a root daemon, so a member can start a privileged container that mounts `/`. That's root.
   **Follow-up:** "Alternatives?" → rootless Docker, sudo rules, CI systems. → 04, 40

### Part 2: Images and containers

7. **Q: Tag vs digest?**
   **A:** A tag is a mutable label. A digest is the SHA-256 of the manifest, which is immutable and identifies exact content.
   **Follow-up:** "How do you deploy safely?" → by digest, build once, immutable tags. → 06, 22

8. **Q: How do image layers work?**
   **A:** Each layer is a tar of filesystem changes, content-addressed. OverlayFS stacks read-only layers with a writable upper layer. Writes copy-up whole files; deletes create whiteouts.
   **Follow-up:** "Why doesn't `RUN rm` shrink the image?" → the earlier layer still contains the file. → 07

9. **Q: What does exit code 137 mean? 143? 127?**
   **A:** 137 = SIGKILL (check OOMKilled, or a stop timeout). 143 = SIGTERM. 127 = command not found. 126 = not executable. 125 = Docker itself failed.
   **Follow-up:** "How do you tell OOM from a manual kill?" → `docker inspect .State.OOMKilled`, `memory.events`, `dmesg`. → 08, 31

10. **Q: `docker stop` vs `docker kill`?**
    **A:** Stop sends the stop signal (SIGTERM), waits 10 s, then SIGKILL. Kill sends SIGKILL (or a chosen signal) immediately.
    **Follow-up:** "Why does my stop take exactly 10 s?" → PID 1 ignores SIGTERM. → 08, 12

11. **Q: `docker exec` vs `docker attach`?**
    **A:** Exec starts a new process that joins the container's namespaces (setns). Attach connects to PID 1's stdio, and exiting it can stop the container.
    **Follow-up:** "How does exec work without a shell in the image?" → it doesn't; use sidecars, nsenter, docker debug. → 09, 10, 51

12. **Q: How do you see a container from the host?**
    **A:** Host PID via inspect, `/proc/PID/ns` to compare namespaces, `nsenter -t PID -n` with host tools, the cgroup path from `/proc/PID/cgroup`, files via `/proc/PID/root`, capabilities via `/proc/PID/status`.
    **Follow-up:** "Is container root the same as host root?" → same UID 0 by default, with fewer capabilities, seccomp and LSM. User namespaces change that. → 11

13. **Q: Why is PID 1 special in containers?**
    **A:** The kernel doesn't apply default signal actions to PID 1 if it has no handler, so SIGTERM is ignored. PID 1 also adopts orphans and must reap zombies.
    **Follow-up:** "How do you fix it?" → exec form, `exec "$@"`, signal handlers, `--init` (tini). → 12

14. **Q: How do you clean up disk safely on a Docker host?**
    **A:** `docker system df -v` and `du` (logs), prune stopped containers, dangling/old images and build cache with filters, truncate huge logs, never prune volumes blindly, and set log rotation.
    **Follow-up:** "Is `system prune -a --volumes` safe?" → no. → 13, 48

### Part 3: Building images

15. **Q: What is the build context?**
    **A:** The folder sent to the builder. COPY/ADD can only read from it. `.dockerignore` keeps it small and keeps secrets out.
    **Follow-up:** "COPY vs ADD?" → ADD extracts tars and fetches URLs/git; prefer COPY. → 14

16. **Q: CMD vs ENTRYPOINT?**
    **A:** ENTRYPOINT is the fixed executable, CMD the default arguments (or the default command). Run arguments replace CMD; `--entrypoint` replaces ENTRYPOINT. Use exec form.
    **Follow-up:** "What do they map to in Kubernetes?" → command = ENTRYPOINT, args = CMD. → 15

17. **Q: How does the build cache decide what to rebuild?**
    **A:** Each step's key is the parent key plus its inputs: command text, file content checksums for COPY, ARG/ENV values. A miss invalidates all later steps.
    **Follow-up:** "How do you speed up builds?" → order steps, cache mounts, .dockerignore, multi-stage, registry cache in CI. → 16

18. **Q: What are multi-stage builds for?**
    **A:** Build with full toolchains in early stages, then copy only the artifacts into a minimal final stage. Smaller, safer images, and unused stages are skipped by BuildKit.
    **Follow-up:** "Why does a binary fail with 'no such file or directory' on Alpine?" → glibc vs musl loader. → 17, 19

19. **Q: ARG vs ENV, and how do you pass secrets to a build?**
    **A:** ARG is build-time only (but visible in history when used by RUN). ENV is baked into the image. Secrets go through `RUN --mount=type=secret` / `--ssh`, never ARG/ENV/COPY.
    **Follow-up:** "Why doesn't deleting a copied secret help?" → layers are immutable. → 18, 44

20. **Q: How do you choose a base image?**
    **A:** Slim (glibc) as a safe default, distroless for runtime, scratch for static binaries, Alpine with care (musl), UBI on Red Hat, hardened images for minimal CVEs. Pin and update automatically.
    **Follow-up:** "How do you debug a distroless image?" → sidecars, nsenter, docker debug. → 19, 51

21. **Q: Name five Dockerfile best practices.**
    **A:** Non-root numeric USER; pin versions/digests with automated updates; clean up in the same RUN with `--no-install-recommends`; `COPY --chown` instead of `chown -R`; exec-form CMD; multi-stage; `.dockerignore`; lint with `docker build --check`/hadolint.
    **Follow-up:** "What does `COPY --link` do?" → independent layers that can be rebased. → 20

22. **Q: How do you build for amd64 and arm64?**
    **A:** `docker buildx build --platform linux/amd64,linux/arm64 --push`, via QEMU emulation, cross-compilation (`FROM --platform=$BUILDPLATFORM` + `TARGETARCH`) or native nodes. The result is an image index.
    **Follow-up:** "What's 'exec format error'?" → architecture mismatch. → 21

### Part 4: Data

23. **Q: Why does container data disappear?**
    **A:** Writes go to the container's writable layer, which is deleted with the container. Upgrades replace containers. Use volumes, bind mounts or external services.
    **Follow-up:** "Does data survive `docker restart`?" → yes. → 23

24. **Q: Volume vs bind mount vs tmpfs?**
    **A:** A volume is Docker-managed (pre-filled from the image when empty). A bind mount is a host path as-is. tmpfs is RAM, temporary, and counts toward memory.
    **Follow-up:** "What's the VOLUME instruction trap?" → anonymous volumes on recreate. → 24, 25

25. **Q: Container gets "permission denied" on a mounted folder. Why?**
    **A:** Numeric UID/GID mismatch between the container user and the host files, or SELinux labels. Fix ownership, match the UID, use `:Z`.
    **Follow-up:** "Why don't config edits appear in the container?" → a single-file bind mount pinned to the old inode. → 25

### Part 5: Networking

26. **Q: How does a container reach the internet?**
    **A:** veth pair → docker0 bridge (gateway) → host routing with ip_forward → iptables MASQUERADE to the host IP. conntrack handles replies.
    **Follow-up:** "How do two containers both use port 80?" → separate network namespaces. → 26

27. **Q: Default bridge vs user-defined bridge?**
    **A:** User-defined gives DNS by name, isolation and its own subnet. The default bridge has no name resolution (`--link` is deprecated).
    **Follow-up:** "host vs bridge networking?" → no netns and no NAT vs isolation + NAT. → 27

28. **Q: How does `-p 8080:80` work, and why doesn't ufw block it?**
    **A:** A DNAT rule rewrites host:8080 → container:80, plus docker-proxy for loopback. The traffic goes through FORWARD, not INPUT, so ufw's rules are bypassed.
    **Follow-up:** "How do you restrict it?" → bind 127.0.0.1, DOCKER-USER rules, security groups. → 28

29. **Q: How does container DNS work?**
    **A:** On user-defined networks, resolv.conf points to 127.0.0.11. dockerd answers names, aliases and services, and forwards the rest upstream (from daemon.json or the host's resolvers).
    **Follow-up:** "Containers can't resolve external names but the host can?" → upstream DNS reachability, systemd-resolved, corporate DNS. → 29

30. **Q: How do you troubleshoot "connection refused" vs "timeout"?**
    **A:** Refused = nothing listening (check the bind address/port inside the netns). Timeout = dropped (firewall, security group, route, MTU). Then walk the path hop by hop with nsenter, netshoot, tcpdump.
    **Follow-up:** "Small requests work, big ones hang?" → MTU. → 30

### Part 6: Runtime behaviour

31. **Q: What happens when a container exceeds its memory limit? Its CPU limit?**
    **A:** Memory → OOM kill inside the cgroup (137, OOMKilled). CPU → CFS throttling, not death.
    **Follow-up:** "Why do JVMs get OOMKilled?" → heap/non-heap sizing vs the limit; `MaxRAMPercentage`. → 31

32. **Q: Restart policies, and does Docker restart unhealthy containers?**
    **A:** no / on-failure / always / unless-stopped, with exponential back-off. Unhealthy containers are **not** restarted by plain Docker. Compose/Swarm/K8s act on health.
    **Follow-up:** "always vs unless-stopped?" → manual stops are respected across daemon restarts. → 32

33. **Q: How do container logs work, and why do disks fill?**
    **A:** stdout/stderr → shim → dockerd → logging driver. json-file has no rotation by default. Set `max-size`/`max-file` or use `local`, and recreate containers.
    **Follow-up:** "Risk of blocking mode with remote drivers?" → app stalls. → 33

34. **Q: Which daemon settings do you configure on every production host?**
    **A:** Log rotation, live-restore, address pools/bip, DNS, registry mirrors, proxies, metrics-addr, build cache GC, data-root on a dedicated disk. Validate with `dockerd --validate`.
    **Follow-up:** "Why never expose 2375?" → unauthenticated root API. → 34

### Part 7: Compose

35. **Q: How does `docker compose up` decide what to recreate?**
    **A:** It labels everything with project/service and a config hash, and recreates only services whose hash or image changed.
    **Follow-up:** "What does `down -v` do?" → removes named volumes too. → 35

36. **Q: `.env` vs `env_file`?**
    **A:** `.env` is for interpolating the compose file. `env_file` injects variables into containers.
    **Follow-up:** "How do you keep a DB password out of inspect?" → Compose secrets + `*_FILE`. → 36

37. **Q: Does `depends_on` wait for readiness?**
    **A:** Not by default. Use healthchecks + `condition: service_healthy`, `service_completed_successfully` for migrations, and app retries.
    **Follow-up:** "CI gate?" → `up --wait`. → 37

38. **Q: Fast dev loops with Compose?**
    **A:** `docker compose watch` (sync/rebuild/sync+restart) or bind mounts with hot reload. Override files for dev.
    **Follow-up:** "node_modules disappeared?" → the bind mount covers it. → 38

39. **Q: How do you run Compose in production on one server?**
    **A:** Pull pinned images built in CI, limits, rotation, restart policies, health checks, deploy with `pull` + `up -d --wait` + rollback, replicas behind a proxy, backups, systemd at boot.
    **Follow-up:** "When do you move to Kubernetes?" → HA, rolling updates, self-healing, autoscaling. → 39

### Part 8: Security

40. **Q: Are containers secure?**
    **A:** Reasonably, with layered controls, but they share the kernel. Most incidents are misconfigurations: socket mounts, privileged mode, host mounts, exposed API, root users.
    **Follow-up:** "Stronger isolation?" → gVisor, Kata, microVMs. → 40

41. **Q: What does `--privileged` do?**
    **A:** All capabilities, no seccomp/LSM, all devices, writable /sys. Effectively host root.
    **Follow-up:** "What would you use instead?" → specific `--cap-add` and `--device`. → 40, 41, 41a

42. **Q: Explain capabilities, seccomp and AppArmor/SELinux.**
    **A:** Capabilities split root powers (Docker keeps 14). Seccomp filters syscalls (the default profile blocks mount, modules, keyctl…). LSMs enforce mandatory access policies (docker-default, container_t + MCS).
    **Follow-up:** "What is no-new-privileges?" → blocks setuid escalation. → 41, 41a

43. **Q: How do you make root in a container not root on the host?**
    **A:** User namespaces: userns-remap (the daemon stays root) or rootless mode (the whole engine is unprivileged). Maps container UID 0 to a subuid range.
    **Follow-up:** "Rootless limitations?" → privileged ports, networking, cgroup delegation. → 42

44. **Q: How do you secure the image supply chain?**
    **A:** Minimal pinned bases with automated updates, scan with a gate (Scout/Trivy), SBOM + provenance attestations, keyless cosign signatures, verified at deploy (admission policies), deploy by digest.
    **Follow-up:** "What's an SBOM?" → a component inventory (SPDX/CycloneDX). → 43

45. **Q: How should containers get secrets?**
    **A:** Never in images. Prefer files (`/run/secrets`, tmpfs) or a secret manager with workload identity and short-lived credentials. Env vars are visible in inspect/proc. Rotate on leaks.
    **Follow-up:** "Leaked AWS key in an image?" → revoke and rotate first, then purge and fix. → 44

### Part 9: Production operations

46. **Q: What makes an app container-ready?**
    **A:** External config, stateless, stdout logs, fast start with retries, graceful SIGTERM as PID 1, liveness vs readiness, non-root/read-only, limits with runtime sizing, UTC, metrics and version info.
    **Follow-up:** "Liveness vs readiness?" → restart vs route traffic. → 45

47. **Q: Describe your container CI pipeline.**
    **A:** Lint → build with cache → test → scan gate → push SHA/semver tags → SBOM/provenance → keyless sign → deploy by digest. OIDC, no stored passwords. No privileged runners for untrusted PRs.
    **Follow-up:** "Why is DinD risky?" → privileged mode; use rootless BuildKit/remote builders. → 46

48. **Q: How do you monitor containers?**
    **A:** cAdvisor (cgroup metrics), node_exporter (host), dockerd metrics, Prometheus/Grafana/Alertmanager, central logs. Alerts on OOM events, memory vs limit, throttling, restarts, disk/inodes.
    **Follow-up:** "Which memory metric?" → working set vs limit. → 47

49. **Q: "No space left on device" but `df -h` shows free space?**
    **A:** Inodes (`df -i`), or a different filesystem, or deleted-but-open files (`lsof +L1`).
    **Follow-up:** "How do you move data-root?" → stop services, rsync -aHAXS --numeric-ids, update config. → 48

50. **Q: How do you upgrade Docker without downtime?**
    **A:** live-restore for patch upgrades; for bigger ones, drain traffic and upgrade host by host; pinned packages; staging first; check the API minimum (1.44 in Engine 29) for old clients.
    **Follow-up:** "What happens when containerd restarts?" → containers keep running via the shims. → 49

### Part 10: Debugging

51. **Q: A container keeps restarting. Debug it.**
    **A:** `ps`/`inspect` (RestartCount, ExitCode, OOMKilled), logs including previous runs, events. Pause the loop with `update --restart no`, reproduce with `--entrypoint sh` and the same config, fix the cause.
    **Follow-up:** "Exit 0 immediately?" → the process finished or daemonised. → 50

52. **Q: How do you debug a distroless container?**
    **A:** A sidecar with `--pid/--network container:X` plus netshoot and `/proc/1/root`; `nsenter` from the host; `docker debug`; `kubectl debug` in K8s.
    **Follow-up:** "Find what a hung process is doing?" → strace -p, py-spy/thread dumps, /proc/PID/fd. → 51

53. **Q: Give an example of a production Docker incident you'd expect, and the fix.**
    **A:** Pick one from topic 52 and tell it as symptom → investigation → root cause → mitigation → prevention (e.g. logs filling the disk → truncate, rotation in daemon.json, disk alerts).
    **Follow-up:** "How would you prevent it fleet-wide?" → config management + monitoring. → 52

### Part 11: Kubernetes bridge

54. **Q: Docker vs Kubernetes?**
    **A:** Docker builds and runs containers on a host. Kubernetes orchestrates them across nodes (scheduling, healing, rollout, services, scaling) via CRI runtimes like containerd.
    **Follow-up:** "What was the dockershim removal?" → kubelet stopped talking to dockerd; OCI images still work. → 53

55. **Q: What's CrashLoopBackOff, and how does it relate to Docker?**
    **A:** A container repeatedly exiting, with increasing restart delays: the same as a Docker restart loop. Check `describe` (Last State, exit code, OOMKilled) and `logs --previous`.
    **Follow-up:** "requests vs limits?" → scheduling/weight vs cgroup caps. → 53

---

## Hands-on lab

1. **Say it out loud.** Pick 10 random questions (e.g. `shuf -i 1-55 -n 10`). Answer each in under 60 seconds without notes, then compare with the model answer.
2. **Follow-up drill:** for each, answer the follow-up too, then invent your own deeper follow-up and answer it.
3. **Whiteboard:** draw from memory (a) the CLI → runc chain, (b) a packet's path for `-p 8080:80`, (c) overlay lower/upper/merged, (d) the SIGTERM → SIGKILL timeline.
4. **Practical:** have a friend run one of topic 52's setups. Narrate your debugging as in an interview.

---

## Where you'll use it

- Before **DevOps/SRE/Platform interviews**: one or two passes the week before.
- **Teaching others:** these questions make a good onboarding quiz for a team.
- **Self-assessment:** any question you can't answer points to the topic to re-read.

---

## When it breaks (common interview mistakes)

| Mistake | Better |
|---|---|
| Reciting definitions only | Add "underneath…" and "in production…" |
| Saying "containers are lightweight VMs" | "Isolated processes sharing the host kernel" |
| "EXPOSE opens the port" | "EXPOSE documents. `-p` publishes" |
| "latest is the newest" | "latest is just the default tag" |
| "Docker restarts unhealthy containers" | "Plain Docker only reports health" |
| "Kubernetes doesn't support Docker images anymore" | "dockershim was removed. OCI images work everywhere" |
| Jumping to "I'd restart it" | Evidence first, then method, then fix and prevention |

---

**Next:** these questions test knowledge. Senior interviews test **reasoning under uncertainty**: incident scenarios, system design and judgement calls. Continue with topic 54a.

## Quick recap

- Answer in layers: one sentence → mechanism → production angle → stop.
- Expect "why" and "what breaks" follow-ups. Each question links to the topic to revise.
- Practise aloud, draw the key diagrams from memory, and narrate real debugging.

**Interview questions (meta)**

1. **Q: What's the single most important Docker concept to explain well?**
   A: That a container is a process with namespaces, cgroups and its own root filesystem on a shared kernel. Most other answers (security, limits, networking, debugging) follow from it.

2. **Q: How do you show depth beyond "I've used Docker"?**
   A: Explain what happens underneath (shim/runc, overlay, DNAT, cgroups), and describe a real failure with how you diagnosed it from the host.

3. **Q: What should you do when you don't know an answer?**
   A: Say what you do know and how you'd find out (e.g. "I'd check `docker inspect` and the cgroup files"). Reasoning from mechanisms is valued more than memorisation.
