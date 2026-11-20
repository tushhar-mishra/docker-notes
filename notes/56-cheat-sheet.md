# 56 · Command Cheat Sheet

> **Level:** ★ foundation
>
> **You will learn:** nothing new. This is every important command from topics 01–55 on one page, grouped by task, with the topic number to go back to when you need the "why".
>
> **You need:** the topics it points to.

---

## What it is

A quick-reference card. Each line: the command, what it's for, and **→ topic** for the full explanation.

## Why it exists

During incidents and interviews you need the exact syntax fast. The understanding is in the topics. This page is the index to your fingers.

## Analogy

The **laminated checklist** pilots keep in the cockpit: not a replacement for training, but it makes sure nothing is forgotten under pressure.

## How it works inside: how to use it

Find the task heading → copy the command → if anything surprises you, open the linked topic. Replace `<c>` = container, `<img>` = image, `<v>` = volume, `<n>` = network.

---

## Commands / files

### Install and daemon (→ 04, 34, 49)

```bash
docker version                                     # client + server versions, API versions
docker info                                        # storage driver, cgroups, root dir, warnings
docker context ls / use <name>                     # which daemon am I talking to?
systemctl status docker containerd                 # service state
journalctl -u docker -n 100 --no-pager             # daemon logs
sudo dockerd --validate --config-file /etc/docker/daemon.json
sudo systemctl reload docker                       # SIGHUP: reload reloadable settings
docker context create prod --docker "host=ssh://user@host"
```

### Images (→ 06, 07, 13, 22)

```bash
docker pull <img>[:tag|@sha256:…]  [--platform linux/arm64]
docker images  /  docker images --digests  /  docker images -f dangling=true
docker image inspect <img> -f '{{json .Config}}'
docker history [--no-trunc] <img>                  # how it was built, layer sizes
docker buildx imagetools inspect <img>             # remote index/platforms without pulling
docker tag <img> registry/ns/repo:tag
docker push registry/ns/repo:tag
docker rmi <img>
docker save -o img.tar <img>  /  docker load -i img.tar
echo "$TOKEN" | docker login <registry> -u <user> --password-stdin
```

### Run and lifecycle (→ 05, 08, 09, 12, 32)

```bash
docker run -d --name <c> -p 127.0.0.1:8080:80 <img>
docker run -it --rm <img> sh                       # interactive, auto-remove
docker run --rm --entrypoint sh -it <img>          # bypass entrypoint
docker run -e K=V --env-file f.env -w /app -u 10001:10001 --hostname h <img>
docker run --init --stop-timeout 30 --restart unless-stopped <img>
docker create / start / stop [-t 30] / kill [-s SIG] / restart / pause / unpause / rm [-f] <c>
docker wait <c>                                    # block, print exit code
docker update --memory 512m --memory-swap 512m --cpus 1 --restart unless-stopped <c>
docker rename <old> <new>
docker attach <c>                                  # detach: Ctrl+P Ctrl+Q
```

### Inspect and debug (→ 10, 11, 50, 51)

```bash
docker ps [-a] [-s] [--filter status=exited|health=unhealthy|label=k=v] [--format '{{.Names}} {{.Status}}']
docker logs [-f] [--tail 100] [-t] [--since 10m] <c>
docker inspect <c> -f 'Status={{.State.Status}} Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Restarts={{.RestartCount}}'
docker inspect <c> -f '{{.State.Pid}}'                                     # host PID
docker inspect <c> -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker inspect <c> -f '{{json .Mounts}}'
docker exec -it [-u root] <c> sh
docker top <c> [-o pid,ppid,stat,args]
docker stats [--no-stream]
docker diff <c>                                    # A/C/D in the writable layer
docker cp <c>:/path ./  /  docker cp ./f <c>:/path
docker port <c>
docker events [--since 30m] [--filter container=<c>] [--filter event=die|oom|health_status]
docker run --rm -it --pid container:<c> --network container:<c> --cap-add SYS_PTRACE nicolaka/netshoot
```

### From the host (→ 02, 11, 31, 51)

```bash
PID=$(docker inspect -f '{{.State.Pid}}' <c>)
sudo ls -l /proc/$PID/ns                           # namespace IDs
grep NSpid /proc/$PID/status                       # host PID vs container PID
sudo nsenter -t $PID -n ss -lntp                   # host tools in its network namespace
sudo nsenter -t $PID -n tcpdump -ni eth0
sudo nsenter -t $PID -a sh                         # all namespaces (needs sh in the image)
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
cat $CG/memory.max $CG/memory.current $CG/memory.events
cat $CG/cpu.max $CG/cpu.stat                       # nr_throttled / throttled_usec
cat $CG/pids.max $CG/pids.current
grep -E 'Uid|CapEff|Seccomp|NoNewPrivs' /proc/$PID/status; capsh --decode=<CapEff>
sudo ls /proc/$PID/root/                           # its filesystem
sudo grep ' / ' /proc/$PID/mountinfo               # its overlay mount (lower/upper)
sudo strace -f -p $PID [-c | -e trace=network]
sudo dmesg -T | grep -i 'out of memory'
sudo ctr -n moby tasks ls  /  sudo crictl ps       # containerd / Kubernetes nodes
```

### Build (→ 14–21)

```bash
docker build -t app:1.0 [-f Dockerfile.prod] [--target final] [--pull] [--no-cache] [--progress=plain] .
docker build --check .                             # lint rules only
docker build --build-arg V=1.2 --label org.opencontainers.image.revision=$SHA .
docker build --secret id=npmrc,src=$HOME/.npmrc --ssh default .
docker buildx ls / create --name b --driver docker-container --use --bootstrap / rm b
docker buildx build --platform linux/amd64,linux/arm64 -t reg/app:1.0 --push .
docker buildx build --cache-from type=registry,ref=reg/app:cache --cache-to type=registry,ref=reg/app:cache,mode=max .
docker buildx build --sbom=true --provenance=mode=max --push -t reg/app:1.0 .
docker buildx build --output type=local,dest=./out .
docker buildx bake [--print] [--push] [target]
docker buildx du  /  docker builder prune [-a] [--filter until=72h] [--keep-storage 20GB]
docker run --privileged --rm tonistiigi/binfmt --install all
docker init
```

Dockerfile essentials:

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true
FROM python:3.13-slim AS build
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=secret,id=token,env=TOKEN ./fetch-private.sh
COPY --chown=10001:10001 --chmod=0755 --link src/ /app/
USER 10001:10001
HEALTHCHECK --interval=30s --timeout=3s CMD ["/app/healthcheck"]
STOPSIGNAL SIGTERM
ENTRYPOINT ["/app/entrypoint.sh"]       # ends with: exec "$@"
CMD ["python", "app.py"]
```

### Data (→ 23–25)

```bash
docker volume create <v> / ls [-f dangling=true] / inspect <v> / rm <v> / prune [-a]
docker run --mount type=volume,src=<v>,dst=/data <img>
docker run --mount type=bind,src=/host/path,dst=/path,readonly <img>
docker run -v /host/dir:/path:Z <img>              # SELinux relabel (private)
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m <img>
docker run --rm -v <v>:/data:ro -v "$PWD":/backup alpine tar -czf /backup/v.tgz -C /data .   # backup
docker run --rm -v <v>:/data -v "$PWD":/backup alpine tar -xzf /backup/v.tgz -C /data          # restore
docker ps -a --filter volume=<v>
```

### Networking (→ 26–30)

```bash
docker network ls / create [--subnet 10.201.5.0/24] [--internal] <n> / inspect <n> / rm <n> / prune
docker network connect / disconnect <n> <c>
docker run --network host|none|container:<c>|<n> [--network-alias a] <img>
docker run --dns 10.0.0.2 --dns-search corp --add-host host.docker.internal:host-gateway <img>
ip -br addr; ip route; bridge link; sysctl net.ipv4.ip_forward
sudo iptables -t nat -S DOCKER; sudo iptables -t nat -L DOCKER -n -v; sudo iptables -S DOCKER-USER
sudo iptables -I DOCKER-USER -i eth0 -p tcp -m conntrack --ctorigdstport 8080 -j DROP
sudo ss -lntp | grep docker-proxy
sudo conntrack -L | head; sudo conntrack -S
docker run --rm --network <n> nicolaka/netshoot dig +short <service>
docker run --rm nicolaka/netshoot ping -c2 -M do -s 1472 1.1.1.1    # MTU test
```

### Resources (→ 31)

```bash
docker run --memory 512m --memory-swap 512m --memory-reservation 256m \
           --cpus 1.5 --cpu-shares 512 --cpuset-cpus 0,1 --pids-limit 256 \
           --device-read-bps /dev/sda:10mb --shm-size 256m --ulimit nofile=65536:65536 <img>
docker run --gpus all <img>                         # NVIDIA Container Toolkit
```

### Logging (→ 33)

```bash
docker run --log-driver local --log-opt max-size=20m --log-opt max-file=5 <img>
docker run --log-driver journald <img>; journalctl CONTAINER_NAME=<c>
docker run --log-opt mode=non-blocking --log-opt max-buffer-size=4m ...
sudo truncate -s 0 $(docker inspect -f '{{.LogPath}}' <c>)          # emergency
```

### Cleanup and disk (→ 13, 48)

```bash
docker system df [-v]
docker container prune -f [--filter until=24h]
docker image prune [-a] -f [--filter until=168h]
docker builder prune -f [--filter until=72h]
docker network prune -f
df -h /var/lib/docker; df -i /var/lib/docker
sudo du -xh --max-depth=1 /var/lib/docker /var/lib/containerd | sort -h
sudo lsof +L1 | grep -E 'dockerd|containerd'
```

### Compose (→ 35–39)

```bash
docker compose up -d [--build] [--wait] [--wait-timeout 120] [--remove-orphans] [--force-recreate] [-V]
docker compose up --watch     /  docker compose watch
docker compose down [-v] [--rmi local]
docker compose ps [-a] / logs [-f] [--tail 50] [svc] / top / images / ls
docker compose exec <svc> sh  /  docker compose run --rm [--service-ports] <svc> <cmd>
docker compose config [--quiet]                    # merged, interpolated result
docker compose pull / build [--pull] / restart <svc> / stop / start
docker compose -f compose.yaml -f compose.prod.yaml --env-file prod.env up -d
docker compose --profile debug up -d   /   COMPOSE_PROFILES=debug docker compose up -d
docker compose up -d --no-deps --scale web=4 --no-recreate web
```

Compose essentials:

```yaml
services:
  api:
    image: reg/api:${TAG:?set TAG}
    build: { context: ., target: dev }
    env_file: [api.env]
    environment: { DB_HOST: db }
    ports: ["127.0.0.1:8080:8080"]
    depends_on: { db: { condition: service_healthy } }
    healthcheck: { test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:8080/healthz"], interval: 10s, retries: 3, start_period: 20s }
    restart: unless-stopped
    init: true
    stop_grace_period: 30s
    user: "10001:10001"
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    deploy: { resources: { limits: { cpus: "1", memory: 512m, pids: 256 } } }
    logging: { driver: local, options: { max-size: "20m", max-file: "5" } }
    secrets: [db_password]
    networks: [frontend, backend]
    develop: { watch: [{ action: sync, path: ./src, target: /app/src }, { action: rebuild, path: requirements.txt }] }
networks: { frontend: {}, backend: { internal: true } }
volumes: { pgdata: {} }
secrets: { db_password: { file: ./secrets/db_password } }
```

### Security (→ 40–44a)

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE --security-opt no-new-privileges <img>
docker run --security-opt seccomp=./profile.json --security-opt apparmor=docker-default <img>
docker run --userns=host ...                       # opt out of userns-remap (careful)
docker run --runtime runsc <img>                   # gVisor
docker scout quickview <img>; docker scout cves --only-severity critical,high --only-fixed <img>
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 <img>
trivy image --scanners secret <img>
cosign sign --yes reg/app@sha256:…; cosign verify reg/app@sha256:… --certificate-identity-regexp … --certificate-oidc-issuer …
sudo sh docker-bench-security.sh
getent group docker; sudo ss -lntp | grep dockerd
dockerd-rootless-setuptool.sh install; docker context use rootless
```

### `daemon.json` baseline (→ 34)

```json
{
  "log-driver": "local",
  "log-opts": { "max-size": "20m", "max-file": "5" },
  "live-restore": true,
  "bip": "10.200.0.1/24",
  "default-address-pools": [{ "base": "10.201.0.0/16", "size": 24 }],
  "dns": ["10.0.0.2"],
  "registry-mirrors": ["https://mirror.internal.example.com"],
  "metrics-addr": "10.200.0.1:9323",
  "no-new-privileges": true,
  "builder": { "gc": { "enabled": true, "defaultKeepStorage": "20GB" } }
}
```

### TLS and reverse proxies (→ 39a)

```bash
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
openssl s_client -connect host:443 -servername host -showcerts </dev/null 2>/dev/null | grep -E '^ *[0-9] s:|i:'   # chain sent?
curl -v --resolve host:443:10.0.0.21 https://host/        # test one backend server, bypassing DNS
docker run --rm curlimages/curl -sS https://internal.example/   # does the IMAGE trust the CA?
# Dockerfile: COPY corp-ca.crt /usr/local/share/ca-certificates/ && RUN update-ca-certificates
```

### Runtime security and incident response (→ 44a)

```bash
docker diff <c>                                              # drift: new/changed files
docker events --filter event=exec_start                      # who is exec-ing into containers
sudo ausearch -k docker -i | tail                            # auditd: who touched the socket/config
# contain → preserve → pause
for n in $(docker inspect <c> -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'); do docker network disconnect $n <c>; done
docker update --restart no <c>
sudo cp /proc/<pid>/exe ./evidence-exe                       # recover even a deleted binary
sudo nsenter -t <pid> -n ss -tanp                            # its live connections
docker pause <c>; docker export <c> | gzip > fs.tar.gz; docker commit <c> forensics/<c>:<ts>
```

### Registry operations (→ 46a)

```bash
docker run -d --name reg -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true -v regdata:/var/lib/registry registry:3
curl -sI -H 'Accept: application/vnd.oci.image.index.v1+json, application/vnd.oci.image.manifest.v1+json' localhost:5000/v2/<repo>/manifests/<tag> | grep -i docker-content-digest
curl -X DELETE localhost:5000/v2/<repo>/manifests/sha256:<digest>          # 202 = unlinked, NOT freed
docker stop reg; docker run --rm -v regdata:/var/lib/registry registry:3 garbage-collect --dry-run /etc/distribution/config.yml
aws ecr put-lifecycle-policy --repository-name <repo> --lifecycle-policy-text file://policy.json
```

### Host tuning (→ 48a)

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
ip -4 neigh show | wc -l; sysctl net.ipv4.neigh.default.gc_thresh3
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances vm.max_map_count
sudo dmesg -T | grep -E 'conntrack: table full|neighbor table overflow'
sudo sysctl --system                                         # apply /etc/sysctl.d/*.conf
docker run --sysctl net.core.somaxconn=8192 --ulimit nofile=65536:65536 <img>   # namespaced only
stat -fc %T /sys/fs/cgroup                                   # cgroup2fs = v2, tmpfs = v1
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:<version> <img>   # wasted space (→ 20)
```

### Kubernetes equivalents (→ 53)

```bash
kubectl get pods,svc,deploy -o wide
kubectl describe pod <p>                           # Last State, Reason (OOMKilled), Exit Code, events
kubectl logs <p> [--previous] [-c container]
kubectl exec -it <p> -- sh
kubectl debug -it <p> --image=nicolaka/netshoot --target=<container>
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status|undo deploy/<d>
kubectl port-forward svc/<s> 8080:80
sudo crictl ps / pods / inspect <id> / logs <id> / images      # on the node
```

### Exit codes (→ 08)

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 / 2 | App error / misuse |
| 125 | Docker failed to run the container |
| 126 | Command not executable |
| 127 | Command not found |
| 130 | SIGINT (Ctrl+C) |
| 137 | SIGKILL: OOM (check OOMKilled) or stop timeout / kill |
| 139 | SIGSEGV (segfault, e.g. glibc vs musl) |
| 143 | SIGTERM |

---

## Hands-on lab

1. Without looking, write the commands for: host PID of a container; its memory limit from the cgroup; its listening ports with host tools; its last 50 log lines with timestamps; the effective entrypoint and command. Check against this sheet.
2. Print this page (or keep it open) during your first week of real on-call.

## Where you'll use it

Incidents, interviews (whiteboard/live terminal), and daily work.

## When it breaks

If a command here fails on your version, check `docker <command> --help` and the release notes. Flags occasionally change (e.g. `docker images` output, the image store, firewall backends), and topic 49 explains how to keep up.

## Quick recap

- Every command is grouped by task, with the topic to revisit.
- First 60 seconds of any incident: `docker ps -a`, `docker inspect` (state), `docker logs --tail`, `docker events`.
- From the host: PID → `/proc/PID/ns`, `nsenter -n`, cgroup files, `/proc/PID/root`.

**Interview questions**

1. **Q: What are the first four commands you run when a container misbehaves?**
   A: `docker ps -a`, `docker inspect` for state, exit code, OOMKilled and restarts, `docker logs --tail 100 -t`, and `docker events --since 30m --filter container=<c>`.

2. **Q: How do you get a container's host PID and read its memory limit?**
   A: `docker inspect -f '{{.State.Pid}}'`, then read `/sys/fs/cgroup/<path from /proc/PID/cgroup>/memory.max`.

3. **Q: One command to see everything Compose will actually apply?**
   A: `docker compose config`, which shows the merged and interpolated configuration.
