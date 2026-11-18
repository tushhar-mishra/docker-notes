# 53 · From Docker to Kubernetes

> **Level:** ★★ intermediate
>
> **You will learn:** why one Docker host eventually isn't enough, what an orchestrator does, Docker Swarm in one paragraph, how Kubernetes runs containers (kubelet → **CRI** → containerd → runc, the same chain as Docker minus dockerd), what the "**dockershim removal**" really meant, how to inspect containers on a Kubernetes node with **`crictl`**, and a complete map from every Docker/Compose concept you've learned to its Kubernetes equivalent (Pods, Deployments, Services, probes, PersistentVolumes, ConfigMaps/Secrets, securityContext, `CrashLoopBackOff`…).
>
> **You need:** all of Parts 1–10. This topic is the bridge.

---

## What it is

**Kubernetes** (K8s) is an **orchestrator**: a system that runs containers across **many machines** (nodes) and keeps them in the **desired state** you declare.

You don't say "run this container on server 3". You say "I want 3 replicas of this image, each with 512 MB of memory, reachable at the name `api`, restarted if unhealthy", and Kubernetes:
- **schedules** them onto nodes with enough free resources,
- **restarts or replaces** them when they crash, become unhealthy, or their node dies,
- **load-balances** traffic across healthy replicas,
- **rolls out** new versions gradually and rolls back on failure,
- **scales** them (manually or automatically),
- attaches **storage**, **configuration** and **secrets**.

The good news: **every container-level mechanism is the same** as in this course. Kubernetes nodes run containerd and runc, with namespaces, cgroups, overlayfs, veth pairs, iptables/eBPF, OCI images. Kubernetes adds the **cluster-level** layer on top.

---

## Why it exists

The limits of one Docker host (topic 39):
- **Single point of failure**: the host dies → everything dies.
- **No self-healing** of unhealthy containers, **no rolling updates**, **no autoscaling**.
- **Capacity**: one machine's CPU and RAM.
- **Manual placement** across hosts doesn't scale to hundreds of services and teams.

---

## Analogy

Docker on one host = **you running one restaurant kitchen** yourself. Kubernetes = **a restaurant chain's head office**: you send it the menu and the target ("300 burgers an hour, always at least 3 kitchens open, never more than 2 closed for renovation at once"), and it assigns kitchens, replaces a kitchen that burns down, shifts staff when it gets busy, and phases in new recipes gradually. Each kitchen still cooks exactly the way you learned.

---

## How it works inside

### Docker Swarm in one paragraph

**Swarm mode** is Docker's built-in orchestrator (`docker swarm init`, `docker service create`, `docker stack deploy -c compose.yaml`). It reuses the Compose file format, has services with replicas, rolling updates, overlay networks (topic 27), encrypted secrets (topic 44) and a routing mesh for published ports. It's simple and still maintained (Mirantis), but the industry has largely standardised on Kubernetes, so learn Swarm only if your company uses it.

### Kubernetes architecture (just enough)

```
                     CONTROL PLANE
  ┌──────────────────────────────────────────────────────┐
  │ kube-apiserver  ← everything talks to this REST API   │
  │ etcd            ← key-value store of desired/actual state │
  │ kube-scheduler  ← picks a node for each new Pod       │
  │ controller-manager ← loops that make actual = desired │
  └──────────────────────────────────────────────────────┘
                 ▲ watch / report
  ┌──────────────┴─────────────────────── each NODE ──────┐
  │ kubelet  ── CRI (gRPC) ──► containerd ──► shim ──► runc ──► containers
  │ kube-proxy (or an eBPF CNI) ← Service load balancing (iptables/IPVS/eBPF)
  │ CNI plugin (Calico, Cilium, Flannel…) ← Pod networking (veth, routes, NAT)
  └───────────────────────────────────────────────────────┘
```

- **kubelet** = the node agent. It watches the API server for Pods assigned to its node and asks the container runtime to run them.
- **CRI** (Container Runtime Interface) = the gRPC API kubelet uses to talk to a runtime. **containerd** implements CRI with a built-in plugin, and so does **CRI-O** (Red Hat's runtime).
- **CNI** (Container Network Interface) = the standard for plugins that set up Pod networking (topic 26 by another name).
- **kube-proxy** = programs Service load balancing on each node with iptables, or **IPVS** (IP Virtual Server, a kernel load balancer that scales better than long iptables chains). Some CNIs replace it with eBPF (topic 26).
- **CSI** (Container Storage Interface) = the standard for storage plugins (like Docker volume drivers, topic 24).

### The dockershim removal

- Early Kubernetes talked to **Docker** (dockerd) through an adapter inside kubelet called **dockershim**. Docker itself didn't speak CRI.
- Kubernetes **removed dockershim in version 1.24 (2022)**. Nodes now use containerd or CRI-O **directly**.
- What it did **not** mean: "Kubernetes can't run Docker images". Images built with Docker are **OCI images** (topic 03) and run unchanged. Only the node's runtime changed: kubelet → containerd, instead of kubelet → dockershim → dockerd → containerd. One fewer layer.
- (A separately maintained adapter, `cri-dockerd`, exists for those who still need Docker Engine on nodes.)
- Consequence for you: on a modern node, **`docker ps` shows nothing** (or Docker isn't even installed). Use **`crictl`**.

### The Pod: Kubernetes' unit

A **Pod** = one or more containers that **share a network namespace** (same IP, `localhost` between them) and optionally volumes, scheduled together on one node. Underneath, it's the `container:` network mode from topic 27: a tiny **pause** container holds the namespaces (topic 11), and the app containers join them.

### Concept map: Docker / Compose → Kubernetes

| Docker / Compose | Kubernetes | Notes |
|---|---|---|
| Image, tag, digest, registry | **Same** (`image:` field) | Deploy by digest; `imagePullPolicy` (`IfNotPresent`, `Always`) ≈ `--pull` |
| Registry login | `imagePullSecrets` (a Secret of type docker-registry) | Or node-level credentials / cloud IAM |
| Container | Container **inside a Pod** | |
| `container:` network mode | Containers in the same **Pod** | Sidecars |
| `docker run` | A **Pod** spec (but you rarely create Pods directly) | |
| Compose **service** with replicas | **Deployment** (→ ReplicaSet → Pods) | Rolling updates, rollback (`kubectl rollout undo`) |
| Stateful service with its own volume | **StatefulSet** | Stable names and volumes per replica |
| One-shot container (migration) | **Job**; scheduled → **CronJob** | ≈ `service_completed_successfully` |
| Init/migration before the app | **initContainers** | Run to completion before app containers start |
| `-p host:container` | **Service** (`ClusterIP` internal, `NodePort`, `LoadBalancer`), **Ingress** / **Gateway API** for HTTP | kube-proxy uses DNAT like topic 28, or eBPF |
| Service name DNS (127.0.0.11) | **CoreDNS**: `api.namespace.svc.cluster.local` | Pods get `ndots:5` → watch DNS latency (topic 29) |
| User-defined networks for isolation | One flat Pod network + **NetworkPolicy** (allow/deny rules) | Needs a CNI that enforces policies |
| Project (Compose) | **Namespace** | Also for RBAC, quotas |
| `environment`, `env_file` | `env`, `envFrom` with a **ConfigMap** | |
| Config file mount | **ConfigMap** mounted as a volume | Directory mounts update live (unlike single-file `subPath` mounts, the same inode trap as topic 25) |
| Secrets (`/run/secrets`) | **Secret** (as env or volume) | Base64, **not encrypted** by default. Enable encryption at rest; use External Secrets / CSI with a vault (topic 44) |
| Named volume | **PersistentVolumeClaim** → PersistentVolume, via a **StorageClass** + CSI driver | |
| Bind mount | `hostPath` | Avoid: ties Pods to nodes, security risk |
| tmpfs | `emptyDir` with `medium: Memory` | Counts toward memory, like topic 25 |
| Writable layer | Same, plus `emptyDir` (Pod-lifetime scratch space) | Ephemeral storage limits exist |
| `--memory`, `--cpus` | `resources.limits` | Same cgroup files (topic 31) |
| `--cpu-shares` / reservations | `resources.requests` | Scheduler uses requests to place Pods. Sets `cpu.weight` |
| `--restart` | `restartPolicy` (Always/OnFailure/Never) + controllers recreating Pods | |
| `HEALTHCHECK` | **Probes**: `livenessProbe` (restart if failing), `readinessProbe` (remove from Service endpoints), `startupProbe` (slow starts) | `HEALTHCHECK` in the image is **ignored** |
| `depends_on` + healthy | initContainers, readiness probes, **app retries** | No `depends_on` in Kubernetes |
| `docker stop` timeout | `terminationGracePeriodSeconds` (default **30 s**), `preStop` hook | Same SIGTERM → SIGKILL (topic 12) |
| `--init` | tini in the image, or `shareProcessNamespace: true` (pause becomes PID 1) | |
| `-u`, `--cap-drop`, `no-new-privileges`, `--read-only`, seccomp | **securityContext**: `runAsUser`, `runAsNonRoot`, `capabilities.drop: [ALL]`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, `seccompProfile: RuntimeDefault` | Pod Security Standards enforce them (topic 41) |
| `--privileged` | `privileged: true` | Same danger (topic 40) |
| userns-remap | `hostUsers: false` | User namespaces for Pods |
| Logs (stdout) | `kubectl logs` (`--previous` = the crashed previous container) | Files under `/var/log/pods/` on the node, rotated by kubelet |
| `docker exec` | `kubectl exec` | |
| `docker debug` / sidecar | `kubectl debug` (ephemeral containers) | Topic 51 |
| `docker events` | `kubectl get events` / `kubectl describe` | |
| `docker stats` | `kubectl top` (metrics-server); cAdvisor in kubelet | Topic 47 |
| `docker compose up` | `kubectl apply -f` (or Helm/Kustomize, GitOps with Argo CD/Flux) | |
| Exit code / OOMKilled in `inspect` | `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137` | Same codes (topic 08) |
| Restart loop (`Restarting`) | **`CrashLoopBackOff`** (back-off 10 s, doubling to 5 min) | Same debugging method (topic 50) |
| Pull errors | **`ErrImagePull` / `ImagePullBackOff`** | Name/tag/auth/rate limit/platform (topics 06, 21, 22) |
| Can't schedule (no capacity) | **`Pending`** with an event like `0/3 nodes are available: insufficient memory` | Requests too high or nodes full |

### Status words you'll see, and the Docker topic that explains them

| Kubernetes says | It means | Go back to |
|---|---|---|
| `CrashLoopBackOff` | Container keeps exiting; kubelet waits longer between restarts | 08, 12, 32, 50 |
| `OOMKilled` (Last State) | Memory limit hit | 31 |
| `ImagePullBackOff` | Can't pull the image | 06, 21, 22 |
| `CreateContainerConfigError` | Missing ConfigMap/Secret key referenced by the Pod | 36, 44 |
| `RunContainerError` / `exec format error` | Wrong entrypoint or architecture | 15, 21 |
| `Pending` | Not scheduled (resources, node selectors, PVC not bound) | 31, 24 |
| `Terminating` stuck | Grace period, finalizers, or a process ignoring SIGTERM | 12 |
| Readiness probe failing | Removed from Service endpoints, no traffic | 32, 45 |

---

## Commands / files

### The `counter` app (topic 35) as Kubernetes objects

```yaml
# counter.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: web
          image: ghcr.io/acme/counter-web:1.0@sha256:<digest>
          env:
            - name: REDIS_HOST
              value: cache
          ports:
            - containerPort: 8000
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 256Mi }
          readinessProbe:
            httpGet: { path: /, port: 8000 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /, port: 8000 }
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: [ALL] }
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 8000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cache
spec:
  serviceName: cache
  replicas: 1
  selector:
    matchLabels: { app: cache }
  template:
    metadata:
      labels: { app: cache }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports: [{ containerPort: 6379 }]
          volumeMounts:
            - { name: data, mountPath: /data }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: Service
metadata:
  name: cache
spec:
  selector: { app: cache }
  ports: [{ port: 6379 }]
```

Read it with your Docker knowledge:
- `Deployment` = Compose service `web` with 2 replicas. `image@digest` = topic 22.
- `resources` = cgroup limits (topic 31). `requests.cpu: 100m` = 0.1 CPU reserved for scheduling.
- Probes = health checks (topic 32), split into liveness/readiness (topic 45).
- `securityContext` = topics 41 and 41a in YAML.
- `Service web` = a stable name + virtual IP load-balancing to ready Pods (≈ published port + DNS).
- `StatefulSet` + `volumeClaimTemplates` = Redis with a named volume (topic 24).

```bash
kubectl apply -f counter.yaml
kubectl get pods,svc
kubectl port-forward svc/web 8000:80          # like -p 127.0.0.1:8000:80, through the API server
curl -s localhost:8000
```

### Debug with the same method (topic 50)

```bash
kubectl get pods
kubectl describe pod web-6d4f9c7b8d-x2k9p
kubectl logs web-6d4f9c7b8d-x2k9p --previous
kubectl get events --sort-by=.lastTimestamp | tail
kubectl debug -it web-6d4f9c7b8d-x2k9p --image=nicolaka/netshoot --target=web
```

```
NAME                   READY   STATUS             RESTARTS      AGE
web-6d4f9c7b8d-x2k9p   0/1     CrashLoopBackOff   5 (40s ago)   4m
...
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Mon, 17 Nov 2026 18:01:02 +0530
      Finished:     Mon, 17 Nov 2026 18:01:31 +0530
    Ready:          False
    Restart Count:  5
```

- `CrashLoopBackOff` + `OOMKilled` + `137` = exactly topic 31's story.

### On the node: crictl (the Kubernetes "docker ps")

```bash
sudo crictl ps
sudo crictl pods
sudo crictl inspect <container-id> | grep -m1 '"pid"'
sudo crictl logs <container-id>
sudo crictl images
```

```
CONTAINER       IMAGE           CREATED         STATE    NAME   ATTEMPT  POD ID          POD
4f1e2d3c4b5a6   9a8b7c6d5e4f3   2 minutes ago   Running  web    0        7e6d5c4b3a29    web-6d4f9c7b8d-q7w2e
"pid": 51234,
```

- With the PID, **every technique from topic 11 works unchanged**: `nsenter -t 51234 -n ss -lntp`, cgroup files under `/sys/fs/cgroup/kubepods.slice/...`, `/proc/51234/root`.
- `crictl` talks CRI to containerd (namespace `k8s.io`). `ctr -n k8s.io` and `nerdctl --namespace k8s.io` also work (topic 51).

### Converting Compose files

`kompose convert -f compose.yaml` generates Kubernetes YAML from a compose file. Useful as a **starting point**, not a finished result: you still need probes, resources, securityContext, proper Secrets and storage classes.

---

## Hands-on lab

1. Install a local cluster: **kind** (Kubernetes in Docker: each "node" is a Docker container) or **k3d**/**minikube**:
   ```bash
   # install the kind and kubectl binaries first (each project's install page gives a one-line download for linux-amd64)
   kind create cluster --name lab
   kubectl get nodes
   docker ps --filter name=lab        # the "node" is a container running containerd inside!
   ```
2. Load your image into the cluster without a registry: `kind load docker-image counter-web:dev --name lab`, change the YAML to `image: counter-web:dev` with `imagePullPolicy: IfNotPresent`, and `kubectl apply`.
3. Get inside the "node" and use crictl: `docker exec -it lab-control-plane crictl ps`. Find a Pod's PID and read its cgroup memory limit from the node.
4. Break things and diagnose: set `limits.memory: 20Mi` (→ OOMKilled/CrashLoopBackOff), use a wrong image tag (→ ImagePullBackOff), make readiness fail (→ Pod Running but not Ready, Service has no endpoints: `kubectl get endpoints web`).
5. Rolling update: change the image tag, `kubectl rollout status deploy/web`, then `kubectl rollout undo deploy/web`.
6. Clean up: `kind delete cluster --name lab`

---

## Where you'll use it

- **Your next learning step**: everything here (images, Dockerfiles, limits, probes, signals, security contexts, debugging) transfers directly to Kubernetes.
- **Interviews:** "Docker vs Kubernetes", "what changed with dockershim", "how do you debug CrashLoopBackOff", "requests vs limits".
- **Migrations:** Compose → Kubernetes, with the concept map as a checklist.
- **Node troubleshooting:** crictl + nsenter + cgroups when kubectl isn't enough.

---

## When it breaks

| Kubernetes symptom | Docker-level cause to check |
|---|---|
| `CrashLoopBackOff` | Exit code and logs (`--previous`), OOM, bad config, missing dependency (topics 08, 31, 50) |
| `ImagePullBackOff` | Name/tag/digest, `imagePullSecrets`, rate limits, platform (06, 21, 22) |
| Pod `Running` but not `Ready` | Readiness probe failing; app binding `127.0.0.1` (30, 45) |
| Service has no endpoints | Labels/selector mismatch, or no Ready Pods |
| Slow DNS from Pods | `ndots:5` + search domains (29) |
| Pods OOMKilled but "memory looks fine" | Non-heap memory, tmpfs/emptyDir in memory, runtime sizing (31) |
| Latency spikes | CPU limits → CFS throttling (31) |
| Graceful shutdown errors on deploys | SIGTERM handling, preStop delay, grace period (12, 45) |
| `docker ps` empty on a node | Expected: no Docker. Use `crictl` |

---

## Quick recap

- Kubernetes orchestrates containers across nodes from a **declared desired state**: scheduling, self-healing, rolling updates, scaling, service discovery.
- Nodes run **kubelet → CRI → containerd → shim → runc**: the same container mechanics as Docker, without dockerd. The **dockershim removal** (1.24) changed the node runtime, not image compatibility.
- **Pod** = containers sharing a network namespace (pause container). Deployments/StatefulSets/Jobs manage Pods. Services/Ingress expose them. CoreDNS gives names. PVC/CSI give storage. ConfigMaps/Secrets give config.
- Probes replace `HEALTHCHECK`. `resources` set the same cgroups. `securityContext` sets the same capabilities/seccomp/user settings.
- `CrashLoopBackOff`, `OOMKilled 137`, `ImagePullBackOff`: the same failures with new names. Debug with `kubectl describe/logs --previous/debug` and, on nodes, `crictl` + `nsenter`.

**Interview questions**

1. **Q: What's the difference between Docker and Kubernetes?**
   A: Docker builds images and runs containers on one host. Kubernetes orchestrates containers across a cluster: scheduling, self-healing, rolling updates, service discovery, scaling, config and storage, using a CRI runtime (usually containerd) on each node. They complement each other: images built with Docker run on Kubernetes.

2. **Q: Kubernetes removed dockershim. Does that mean Docker images no longer work?**
   A: No. Images are OCI-standard and run on containerd or CRI-O. Only the node-level integration with Docker Engine was removed, and kubelet now talks CRI directly to containerd/CRI-O. Use `crictl` instead of `docker` on nodes.

3. **Q: How do Docker concepts map to Kubernetes resource limits and health checks?**
   A: `--memory`/`--cpus` become `resources.limits` (the same cgroup `memory.max`/`cpu.max`), and CPU shares/reservations become `resources.requests` (used for scheduling and `cpu.weight`). Docker's single HEALTHCHECK becomes three probes: liveness (restart), readiness (traffic routing) and startup (slow boot).
