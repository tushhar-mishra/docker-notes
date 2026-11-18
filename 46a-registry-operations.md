# 46a · Registry Operations

> **Level:** ★★★ advanced
>
> **You will learn:** how to **run** a container registry in production, not just push to one: how a registry stores data (blobs, manifests, tags), why deleting an image doesn't free disk space until **garbage collection** runs, how to run GC safely, **retention policies** (Harbor tag retention, ECR lifecycle policies, GHCR cleanup), **Harbor** (projects, RBAC, robot accounts, scanning, proxy cache, replication, immutable tags, quotas), pull-through caches for Docker Hub, high availability and backup, and what to monitor.
>
> **You need:** topic 07 (blobs, manifests, digests), topic 22 (push/pull, `registry:3`, credentials), topic 39a (TLS), topics 43 and 46 (signing, CI).

---

## What it is

A **registry** is a storage server with an HTTP API (the OCI Distribution spec, topic 03). Running one in production means owning:
- its **storage** (it grows with every CI build),
- its **availability** (if it's down, nobody can deploy, scale or recover from a node failure),
- its **security** (who can push and pull, what may be pulled),
- its **hygiene** (retention, garbage collection, scanning).

| Option | Who runs it | Typical use |
|---|---|---|
| `registry:3` (CNCF Distribution) | You | Simple private registry, pull-through cache, labs |
| **Harbor** (CNCF) | You | Full enterprise registry: UI, RBAC, scanning, replication, policies |
| ECR / Artifact Registry / ACR / GHCR | Cloud provider | Managed. You still own retention, access and policies |
| Artifactory / Nexus | You or SaaS | Multi-format artifact managers (Maven, npm, Docker…) |

---

## Why it exists

Typical problems teams hit after a year or two:
- **Storage explosion**: every commit pushes a new image (hundreds of MB). A busy CI pipeline creates terabytes, and the cloud bill grows quietly.
- **"We deleted images but the disk is still full"**: deletes don't free space until GC.
- **Outages caused by the registry**: Docker Hub rate limits, or your own registry down during an incident, so you can't roll back.
- **Security**: anyone can push to `prod/`, old vulnerable images stay pullable forever, and credentials are shared by everyone.

---

## Analogy

A **library with a card catalogue**:
- **Blobs** = the books on the shelves (layers, configs).
- **Manifests** = catalogue cards listing which books make up a "set" (an image).
- **Tags** = friendly labels on the cards ("latest edition").
- Removing a card (deleting a manifest or tag) doesn't remove the books. Someone has to walk the shelves later and remove every book that no card refers to anymore. That walk is **garbage collection**, and the library must stop accepting new books while it happens, or a book still being catalogued could be thrown away.

---

## How it works inside

### 1. Storage layout

A Distribution registry stores everything by **digest** (content addressing, topic 07):

```
/var/lib/registry/docker/registry/v2/
├── blobs/sha256/<first 2 hex>/<digest>/data          ← layers, configs, manifests (the actual bytes)
└── repositories/<name>/
    ├── _layers/sha256/<digest>/link                   ← "this repository uses this blob"
    ├── _manifests/revisions/sha256/<digest>/link      ← manifests that exist in this repository
    └── _manifests/tags/<tag>/current/link             ← tag → manifest digest
```

- A blob shared by 10 repositories is stored **once**. Each repository only has a small `link` file to it.
- A **tag** is just a pointer file to a manifest digest.

### 2. What "delete" really does

| Action | Effect | Disk freed? |
|---|---|---|
| Delete a **tag** | The tag pointer is removed; the manifest still exists (reachable by digest) | No |
| Delete a **manifest** (by digest) | The manifest is unlinked from the repository; the blobs remain | No |
| **Garbage collection** | Walks every repository, marks every blob referenced by any remaining manifest, then **sweeps** (deletes) unmarked blobs | **Yes** |

- Deletes are **disabled** by default in `registry:3`. Enable them with `storage.delete.enabled: true` (or the environment variable `REGISTRY_STORAGE_DELETE_ENABLED=true`).
- GC is a **mark-and-sweep** process run with the `registry garbage-collect` command.
- **Safety:** if a push is uploading blobs while GC runs, GC may delete those blobs (they're not referenced by a manifest yet). Run GC with the registry **stopped** or in **read-only mode** (`storage.maintenance.readonly.enabled: true`). Harbor schedules GC for you and handles this.
- `--delete-untagged` also removes manifests that have no tags. Careful: images deployed **by digest** (topic 22) may be untagged but still in use. Check what's running before you enable it.
- **Multi-platform indexes and attestations** (topics 21, 43): an index references platform manifests and attestation manifests. Deleting only the index can leave them as orphans. Use the tool's index-aware delete or retention features.

### 3. Retention policies

Decide **what to keep**, write it down as a policy, and let a tool enforce it on a schedule. A common policy:

| Keep | Why |
|---|---|
| All **release** tags (`v*`, semver) for N months or forever | Rollback, audits |
| The last N `sha-*` builds per repository (e.g. 50) | Recent rollback/debugging |
| Anything currently **deployed** (all environments) | Never break running workloads |
| Untagged manifests older than 7 days | Usually safe to expire |
| Branch/PR builds for 14 days | Short-lived |

Where it's enforced:
- **Harbor:** tag retention rules per project, plus scheduled GC.
- **ECR:** **lifecycle policies** (JSON rules that expire images by age or count; ECR frees the storage itself, so no GC to run).
- **GHCR:** no server-side policy; cleanup with the API or an action in a scheduled workflow.
- **Artifact Registry / ACR:** cleanup policies / purge tasks.

### 4. Harbor in one picture

```
           clients (docker, CI, Kubernetes)
                     │ HTTPS (topic 39a)
             ┌───────▼────────┐
             │ nginx (proxy)  │
             ├────────────────┤
             │ core (API, UI, │──► PostgreSQL (metadata, users, policies)
             │ auth, RBAC)    │──► Redis (cache, job queue)
             ├────────────────┤
             │ registry       │──► storage: filesystem / S3 / Azure / GCS
             │ (Distribution) │
             ├────────────────┤
             │ jobservice     │  replication, GC, retention, scans
             │ trivy adapter  │  vulnerability scanning
             └────────────────┘
```

Key features:
- **Projects** = folders of repositories with their own members, quotas and policies (`harbor.example.com/payments/api`).
- **RBAC** (role-based access control): project roles (admin, maintainer, developer, guest), with OIDC/LDAP login for humans.
- **Robot accounts** = non-human credentials with limited scope (e.g. "pull only from project X", with an expiry). Use these for CI and Kubernetes nodes, never personal accounts.
- **Vulnerability scanning** (Trivy) on push, and a policy to **prevent pulling** images with critical CVEs.
- **Immutable tags** (topic 22) and **content trust / cosign signature verification** before pulls.
- **Proxy cache projects**: a pull-through cache for Docker Hub, GHCR and others (`harbor.example.com/dockerhub-proxy/library/nginx:1.27`).
- **Replication**: push or pull images between Harbors or clouds (DR, multi-region, air-gapped sites).
- **Quotas** per project, so one team can't fill the storage.
- It's installed with an installer that generates a **Compose** stack (topic 35), or with a Helm chart on Kubernetes.

### 5. Pull-through caches

- `registry:3` can act as a **read-only mirror** of one upstream: `proxy.remoteurl: https://registry-1.docker.io`. Point dockerd at it with `registry-mirrors` (topic 34). Note that `registry-mirrors` only applies to **Docker Hub** images.
- For several upstreams (GHCR, Quay…), use Harbor proxy-cache projects, or cloud pull-through cache rules (ECR supports Docker Hub, GHCR, Quay and others), and reference images through the cache's name.
- Benefits: no rate limits, faster pulls, and images survive upstream deletions and outages.

### 6. Availability and backup

- **Stateless registry processes** behind a load balancer, with storage on **object storage** (S3/GCS/Azure Blob) and Harbor's Postgres/Redis made highly available (or managed).
- **Backup**: object storage versioning/replication, plus database backups (Harbor's metadata). Test restores.
- **Disaster recovery**: replication to a second region/registry. Keep the images your platform needs to boot (base images, ingress/CNI images, the registry itself!) available locally, so an outage of the registry doesn't stop you recovering the cluster that runs it.

### 7. What to monitor

- Storage used and growth rate (alert on forecast, not just a threshold).
- Request rate, latency and **5xx errors**; **401/403** spikes (expired robot credentials).
- Pull failures from nodes (`ImagePullBackOff` in Kubernetes, topic 53).
- GC and retention job results (failed jobs = storage keeps growing).
- Certificate expiry (topic 39a).
- Registry exposes Prometheus metrics (Distribution: `http.debug.prometheus` in the config; Harbor: built-in exporter).

---

## Commands / files

### Run `registry:3` with deletes enabled

```bash
docker run -d --name reg -p 5000:5000 \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -v regdata:/var/lib/registry registry:3
```

### Push two versions to the same tag

```bash
for v in 1 2; do
  printf 'FROM alpine:3.22\nRUN head -c 20000000 /dev/urandom > /blob-%s\n' $v | docker build -q -t localhost:5000/app:latest -
  docker push -q localhost:5000/app:latest
done
docker exec reg du -sh /var/lib/registry
```

```
46.1M   /var/lib/registry
```

- Two 20 MB random layers plus the shared Alpine base. The first version is now **untagged** but its blobs are still stored.

### Find and delete a manifest by digest

```bash
ACCEPT='application/vnd.oci.image.index.v1+json, application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json'
curl -sI -H "Accept: $ACCEPT" localhost:5000/v2/app/manifests/latest | grep -i docker-content-digest
curl -s -o /dev/null -w '%{http_code}\n' -X DELETE localhost:5000/v2/app/manifests/sha256:<digest-to-delete>
```

```
docker-content-digest: sha256:4b1f…
202
```

- The `Accept` header matters: without it, the registry may return a different (converted) manifest type with a different digest.
- `202 Accepted` = the manifest is unlinked. Disk space isn't freed yet.
- `405 Method Not Allowed` = deletes aren't enabled.

### Garbage collection (registry stopped or read-only)

```bash
docker stop reg
docker run --rm -v regdata:/var/lib/registry registry:3 \
  garbage-collect --dry-run /etc/distribution/config.yml | tail -3
docker run --rm -v regdata:/var/lib/registry registry:3 \
  garbage-collect --delete-untagged /etc/distribution/config.yml | tail -3
docker start reg
docker exec reg du -sh /var/lib/registry
```

```
blob eligible for deletion: sha256:9c2e…
blob eligible for deletion: sha256:a71b…
blob eligible for deletion: sha256:9c2e…
26.4M   /var/lib/registry
```

- `--dry-run` = only list what would be deleted. Always run it first.
- Config path: `registry:3` uses `/etc/distribution/config.yml`. The older `registry:2` image used `/etc/docker/registry/config.yml`.

### ECR lifecycle policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images after 7 days",
      "selection": { "tagStatus": "untagged", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 7 },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep only the last 50 sha- builds",
      "selection": { "tagStatus": "tagged", "tagPrefixList": ["sha-"], "countType": "imageCountMoreThan", "countNumber": 50 },
      "action": { "type": "expire" }
    }
  ]
}
```

```bash
aws ecr put-lifecycle-policy --repository-name api --lifecycle-policy-text file://policy.json
aws ecr get-lifecycle-policy-preview --repository-name api     # see what would expire, before it happens
```

- Release tags (`v*`) aren't matched by any rule, so they're kept.
- **Make sure deployed images are protected** (e.g. by a dedicated tag prefix that no rule expires).

### Pull-through cache for Docker Hub with `registry:3`

```yaml
# mirror-config.yml
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
proxy:
  remoteurl: https://registry-1.docker.io
  username: <dockerhub-user>          # optional: authenticated pulls = higher limits
  password: <dockerhub-access-token>
```

```bash
docker run -d --name mirror -p 5001:5000 -v "$PWD/mirror-config.yml:/etc/distribution/config.yml:ro" -v mirrordata:/var/lib/registry registry:3
# daemon.json on every Docker host:  "registry-mirrors": ["https://mirror.internal.example.com"]   (TLS in production, topic 39a)
```

### Harbor robot account for CI (via the API)

```bash
curl -s -u admin:'<password>' -H 'Content-Type: application/json' -X POST https://harbor.example.com/api/v2.0/robots -d '{
  "name": "ci-payments", "duration": 90, "level": "project",
  "permissions": [{ "kind": "project", "namespace": "payments",
    "access": [{ "resource": "repository", "action": "push" }, { "resource": "repository", "action": "pull" }] }]
}'
```

- Returns a robot name (like `robot$payments+ci-payments`) and a secret. Store the secret in your CI's secret store. It expires in 90 days, so automate rotation.

---

## Hands-on lab

1. Run the `registry:3` GC lab above. Record the storage size before deletion, after deletion (unchanged!) and after GC.
2. Try to delete with deletes disabled (start a second registry without the env var) and see `405`.
3. Set up the Docker Hub pull-through mirror on port 5001 (without TLS, add it to `insecure-registries` for this lab only), point `registry-mirrors` at `http://localhost:5001`, pull `nginx:1.27-alpine`, then check `docker exec mirror ls /var/lib/registry/docker/registry/v2/repositories/library`.
4. Write a retention policy for your team in plain language (what to keep, for how long, how deployed images are protected), then translate it into an ECR lifecycle policy.
5. **Optional:** install Harbor in a VM with its online installer, create a project, a robot account and a proxy-cache project for Docker Hub, and enable scan-on-push.
6. Clean up: `docker rm -f reg mirror; docker volume rm regdata mirrordata`

---

## Where you'll use it

- **Platform teams:** owning the company registry (Harbor or cloud), its policies, cost and availability.
- **Cost incidents:** "registry storage costs doubled" → retention + GC.
- **Reliability:** pull-through caches against rate limits and upstream outages, and DR replication.
- **Security:** robot accounts, scan-and-block policies, immutable tags, signature verification.
- **Interviews:** "Why didn't deleting images free space?" and "How do you manage registry growth?"

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `405 Method Not Allowed` / `UNSUPPORTED` on DELETE | Deletes disabled | `storage.delete.enabled: true` |
| Storage didn't shrink after deleting images | GC hasn't run | Run GC (stopped/read-only registry) or schedule it (Harbor) |
| `blob unknown` / pulls fail right after GC | GC deleted blobs of an in-progress upload, or of a manifest referenced only by digest | Never GC a writable registry; check digest-deployed images before `--delete-untagged` |
| Deployed image disappeared from the registry | Retention rule too aggressive | Protect deployed tags; preview policies before applying |
| `unauthorized` from CI suddenly | Robot account/token expired | Rotation automation; alert on 401 spikes |
| Mirror not used for GHCR/Quay images | `registry-mirrors` only applies to Docker Hub | Use Harbor proxy projects or reference the cache's hostname |
| Registry full, pushes fail with `500` / `no space left on device` | Storage exhausted | Emergency GC/retention; move to object storage; quotas |
| Can't roll back during an incident: registry down | Single point of failure | HA, replication, local copies of critical images |

---

## Quick recap

- A registry stores **blobs by digest**. Manifests list blobs, and tags point to manifests.
- **Deleting a tag or manifest doesn't free disk space.** Garbage collection (mark and sweep) does. Run it on a stopped or read-only registry, dry-run first, and beware `--delete-untagged` with digest deployments and indexes.
- Write a **retention policy** (keep releases, last N builds, everything deployed) and enforce it: Harbor rules, ECR lifecycle policies, scheduled cleanup for GHCR.
- **Harbor** adds projects, RBAC, robot accounts, scanning with pull blocking, proxy caches, replication, immutable tags and quotas.
- Treat the registry as **critical infrastructure**: HA storage, backups, DR replication, monitoring of storage, errors, auth failures and job results.

**Interview questions**

1. **Q: We deleted thousands of old images from our self-hosted registry, but disk usage didn't change. Why?**
   A: Deleting tags or manifests only removes references. The layer blobs stay until garbage collection marks the blobs still referenced and sweeps the rest. Run GC with the registry stopped or read-only, or use a registry like Harbor that schedules it.

2. **Q: How would you control registry growth in a busy CI environment?**
   A: A retention policy (keep releases and anything deployed, the last N commit builds, expire branch builds and untagged images after a few days), enforced automatically (Harbor retention, ECR lifecycle), plus scheduled GC, per-project quotas and storage-growth alerts.

3. **Q: How do you protect deployments from Docker Hub rate limits and outages?**
   A: Pull through a cache or mirror (registry:3 proxy, a Harbor proxy-cache project, or an ECR pull-through rule) with authenticated upstream credentials, or copy the images you depend on into your own registry and deploy from there by digest.
