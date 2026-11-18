# 46 · Building Images in CI Pipelines

> **Level:** ★★★ advanced
>
> **You will learn:** the standard CI pipeline for images (lint → build → test → scan → tag → push → sign → deploy by digest), how to keep CI builds fast with **registry/CI cache export**, how to authenticate without stored passwords (OIDC), a complete GitHub Actions example (plus a GitLab CI one), and the big security decision: **how** CI gets a builder: socket mount vs **Docker-in-Docker** vs **rootless BuildKit** vs remote builders, with the risks of each.
>
> **You need:** topics 16–22 (BuildKit, cache, multi-stage, buildx, registries), topics 40–44 (security, supply chain, secrets).

---

## What it is

**CI** (Continuous Integration) = an automated system (GitHub Actions, GitLab CI, Jenkins, CircleCI, Buildkite, Azure Pipelines…) that runs on every commit/PR. For containers, it turns source code into a **tested, scanned, signed, pushed image** with a known digest, ready to deploy.

```
commit ─► lint Dockerfile ─► build (with cache) ─► unit/integration tests ─► scan (gate)
       ─► push (SHA tag; semver on release) ─► SBOM + provenance ─► sign (keyless) ─► deploy by digest
```

---

## Why it exists

- **Build once, deploy many**: the exact bytes tested in CI are what runs in production (topic 22).
- **No "built on someone's laptop"** images in production: builds are reproducible and auditable.
- Security checks happen **before** anything reaches a registry users pull from.
- Developers get fast feedback on every change.

---

## Analogy

A **factory assembly line with quality control stations**. Raw materials (source) enter. Each station does one job: shape (build), test, inspect for defects (scan), stamp a serial number (tag/digest), seal (sign), then ship to the warehouse (registry). A defect at any station stops the line. Nothing reaches the shop without passing every station.

---

## How it works inside

### 1. Pipeline stages and what each produces

| Stage | Tooling | Fails the pipeline when |
|---|---|---|
| Lint | `docker build --check`, hadolint (topic 20) | Dockerfile anti-patterns |
| Build | `docker buildx build` (BuildKit) | Build errors |
| Test | `--target test` stage, or run tests inside the built image, or `docker compose up --wait` + integration tests | Tests fail |
| Scan | Docker Scout / Trivy with `--exit-code` (topic 43) | Critical/high fixable CVEs, secrets |
| Push | `--push` with tags: `sha-<git sha>`, plus semver on tags, maybe `main` | Auth/registry errors |
| Attest | `--sbom=true --provenance=mode=max` | — |
| Sign | `cosign sign` keyless (OIDC) | — |
| Deploy | Update manifests/compose with the **digest** (GitOps PR, `docker compose` over SSH, Helm…) | Health checks fail |

### 2. CI runners start empty, so caching is essential

A fresh CI runner has **no BuildKit cache**. Without help, every build starts from zero (topic 16). Options:

| Cache backend | Flag | Notes |
|---|---|---|
| **Registry** | `--cache-to type=registry,ref=REG/app:buildcache,mode=max` + `--cache-from type=registry,ref=REG/app:buildcache` | Works everywhere. `mode=max` caches **all** stages' layers (not just the final image) |
| **Inline** | `--cache-to type=inline` | Cache metadata inside the pushed image itself. Only `mode=min` (final stage layers) |
| **GitHub Actions cache** | `--cache-to type=gha,mode=max` + `--cache-from type=gha` | Uses the GitHub Actions cache service |
| **Local folder** | `--cache-to type=local,dest=/cache` | For persistent self-hosted runners or CI cache folders |
| **S3 / Azure blob** | `type=s3` / `type=azblob` | Shared cache for large teams |
| **Persistent builder** | A long-lived BuildKit (remote driver, Kubernetes driver, Docker Build Cloud) | Cache stays warm, including cache mounts |

Note: **`RUN --mount=type=cache` contents aren't exported** by these backends (only layers are). Persistent builders keep them.

### 3. Authentication without stored secrets: OIDC

Modern CI platforms can issue the job a short-lived **OIDC token** ("I am workflow X in repo Y on branch Z"). Cloud providers and registries trust that token in exchange for short-lived credentials:
- GitHub Actions → `GITHUB_TOKEN` for GHCR (built in), or OIDC → AWS IAM role (ECR), GCP Workload Identity Federation (Artifact Registry), Azure federated credentials (ACR).
- The same OIDC identity drives **keyless cosign signing** (topic 43).
- No long-lived registry passwords or cloud keys sitting in CI secrets to leak.

### 4. How the CI job gets a container builder (the security decision)

| Method | How | Risk | Use when |
|---|---|---|---|
| **Hosted runner's own Docker** (e.g. GitHub-hosted Ubuntu runners) | The runner is a throwaway VM with Docker installed | Low: the VM is destroyed after the job | Default choice on hosted CI |
| **Mount the host's socket** (`-v /var/run/docker.sock:...` into the job container) | Jobs control the host's dockerd | **Host root** for every job (topic 40). Jobs see and can kill each other's containers. Cache shared across jobs, which is also a poisoning risk | Only on dedicated, single-tenant runners you fully trust |
| **Docker-in-Docker (DinD)** (`docker:dind` service, **privileged**) | A separate dockerd runs inside a privileged container | `--privileged` = host root if the job is malicious. Nested overlay/storage quirks. Empty cache each time unless persisted | Legacy GitLab setups. Prefer alternatives |
| **Rootless BuildKit** (`moby/buildkit:rootless`, `buildctl-daemonless.sh`) | BuildKit runs as non-root in a user namespace, no Docker daemon | Much lower: no privileged mode needed (some setups need `seccomp=unconfined` / `apparmor=unconfined` or a user-namespace-friendly kernel) | Kubernetes-based runners; shared runners |
| **Remote builder** (buildx `remote`/`kubernetes` driver, Docker Build Cloud) | The job sends builds to a dedicated BuildKit service | Builder is isolated from job runners. Warm cache. The builder itself must be secured | Large teams, speed |
| Kaniko / Buildah | Daemonless image builders | Kaniko is no longer actively developed by Google (archived 2025; community forks exist). Buildah (Red Hat) is maintained and rootless-capable | Existing setups; Podman shops |

**Principles:** never run untrusted PRs (e.g. from forks) on runners with privileged Docker access or secrets. Separate build runners from deploy credentials. Prefer ephemeral runners.

### 5. Tagging in CI

From the git context (the `docker/metadata-action` in GitHub Actions automates this):

| Event | Tags |
|---|---|
| Every push | `sha-3f9a2c1` (immutable) |
| Push to `main` | `main` (moving, for dev environments) |
| Git tag `v1.4.2` | `1.4.2`, `1.4`, `1` (+ `latest` if you insist) |
| Pull request | `pr-123` (often not pushed at all) |

Plus labels: `org.opencontainers.image.revision`, `source`, `created`, `version`.

### 6. Deploy by digest

The build step outputs the **digest** (`docker buildx build --metadata-file meta.json` or the action's `digest` output). The deploy step uses `image@sha256:…`, never a moving tag.

---

## Commands / files

### GitHub Actions: a complete workflow

```yaml
# .github/workflows/image.yml
name: image

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:

permissions:
  contents: read
  packages: write        # push to GHCR with GITHUB_TOKEN
  id-token: write        # OIDC for keyless cosign
  attestations: write

env:
  IMAGE: ghcr.io/${{ github.repository }}
  TRIVY_IMAGE: aquasec/trivy:<version>@sha256:<digest>   # pinned scanner (topic 43)

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        run: docker build --check .

      - uses: docker/setup-qemu-action@v3          # emulation for arm64 (topic 21)
      - uses: docker/setup-buildx-action@v3        # docker-container builder

      - name: Test (build the test stage)
        uses: docker/build-push-action@v6
        with:
          target: test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Tags and labels
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=sha,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build for scanning (single platform, loaded locally)
        uses: docker/build-push-action@v6
        with:
          load: true
          tags: scan-me:${{ github.sha }}
          cache-from: type=gha

      - name: Scan (fail on fixable critical/high)
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock "$TRIVY_IMAGE" \
            image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed scan-me:${{ github.sha }}

      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push (multi-platform, SBOM, provenance)
        id: push
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          sbom: true
          provenance: mode=max
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - uses: sigstore/cosign-installer@v3
        if: github.event_name != 'pull_request'

      - name: Sign (keyless)
        if: github.event_name != 'pull_request'
        run: cosign sign --yes ${{ env.IMAGE }}@${{ steps.push.outputs.digest }}

      - name: Output digest
        if: github.event_name != 'pull_request'
        run: echo "Deploy ${{ env.IMAGE }}@${{ steps.push.outputs.digest }}" >> "$GITHUB_STEP_SUMMARY"
```

Key points:
- `permissions:` grants only what's needed. `id-token: write` enables OIDC.
- PRs build and test, but **don't push or sign** (and get no secrets). Safe for forks.
- Cache: `type=gha` with `mode=max`.
- The scan uses the runner's own Docker socket. That's fine here because the GitHub-hosted runner VM is ephemeral and single-job.
- The **digest** output is what deployments use.
- **The `@v4`-style tags above are for readability only.** In a real pipeline, pin every third-party action (and the scanner image, see `TRIVY_IMAGE`) to an exact **commit SHA**, because an action tag can be moved just like an image tag. The tj-actions compromise in March 2025 (topic 43) leaked secrets from thousands of pipelines that used moving tags. The format:
  ```yaml
  - uses: docker/build-push-action@<full-40-character-commit-sha>   # v6.x.y
  ```
  Get the SHA from the action's release page (or `git ls-remote https://github.com/docker/build-push-action refs/tags/v6.18.0`), keep the version in a comment, and let **Dependabot/Renovate** propose updates as reviewed pull requests. GitHub also has an org-level setting to *require* SHA-pinned actions.

### The same build with plain buildx (any CI)

```bash
docker buildx create --use --name ci --driver docker-container
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=registry.acme.com/api:buildcache \
  --cache-to   type=registry,ref=registry.acme.com/api:buildcache,mode=max \
  --label org.opencontainers.image.revision=$GIT_SHA \
  --sbom=true --provenance=mode=max \
  --metadata-file meta.json \
  -t registry.acme.com/api:sha-${GIT_SHA:0:7} \
  --push .
python3 -c 'import json;print(json.load(open("meta.json"))["containerimage.digest"])'
```

```
sha256:9f1e0c…
```

### GitLab CI with rootless BuildKit (no privileged DinD)

```yaml
# .gitlab-ci.yml
build:
  image:
    name: moby/buildkit:rootless
    entrypoint: [""]
  variables:
    BUILDKITD_FLAGS: --oci-worker-no-process-sandbox
  before_script:
    - mkdir -p ~/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf '%s:%s' "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" > ~/.docker/config.json
  script:
    - buildctl-daemonless.sh build
        --frontend dockerfile.v0
        --local context=. --local dockerfile=.
        --output type=image,name=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA,push=true
        --export-cache type=registry,ref=$CI_REGISTRY_IMAGE:buildcache,mode=max
        --import-cache type=registry,ref=$CI_REGISTRY_IMAGE:buildcache
```

- `buildctl-daemonless.sh` starts a temporary rootless BuildKit daemon for the job.
- `--oci-worker-no-process-sandbox` = don't create a PID namespace for build steps. It's needed when the runner's security settings block it, at the cost of weaker isolation between build steps.
- `CI_REGISTRY_*` are GitLab's built-in short-lived job credentials.
- Depending on the runner's kernel/security profile, the job container may need `seccomp=unconfined`/`apparmor=unconfined` (still far better than `--privileged`).

### Why DinD needs privileged (and what it looks like)

```yaml
# the classic (risky) way
build:
  image: docker:29
  services:
    - name: docker:29-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
```

- The `dind` service container must run **`--privileged`** (configured on the runner) so its inner dockerd can create namespaces, cgroups and overlay mounts. A malicious job can use that to become root on the runner host.

---

## Hands-on lab

1. Create a GitHub repository with `hello-flask` (or `hello-go`) and the workflow above (simplify it if you like: drop multi-platform first). Push and watch it run. Open the image's GHCR page and check the labels, provenance and signature.
2. Make a code-only change and compare build times **with** and **without** the `cache-from/cache-to` lines.
3. Introduce an old base image with known CVEs (e.g. `python:3.9-slim-bullseye`) and watch the scan step fail the pipeline.
4. Verify the signature locally with `cosign verify` using `--certificate-identity-regexp` for your workflow and `--certificate-oidc-issuer https://token.actions.githubusercontent.com`.
5. Locally, try rootless BuildKit:
   ```bash
   docker run --rm -it --security-opt seccomp=unconfined --security-opt apparmor=unconfined \
     -e BUILDKITD_FLAGS=--oci-worker-no-process-sandbox \
     -v "$PWD":/work -w /work --entrypoint buildctl-daemonless.sh moby/buildkit:rootless \
     build --frontend dockerfile.v0 --local context=. --local dockerfile=. --output type=tar,dest=/work/out.tar
   ls -lh out.tar
   ```
   A full build, inside an unprivileged container, without any Docker daemon.

---

## Where you'll use it

- **Every team that ships containers.** This is most of the day-to-day "Docker in DevOps" work.
- **Platform teams:** shared workflow templates, runner fleets, remote builders, cache strategy.
- **Security:** OIDC instead of stored secrets, removing privileged DinD and socket mounts, signing and provenance.
- **Interviews:** "Walk me through your container CI/CD pipeline" and "Why is Docker-in-Docker risky?"

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| Every CI build takes the full time | No cache import/export | `--cache-from/--cache-to` (gha/registry, `mode=max`) |
| `ERROR: failed to solve: ... cache export is not supported for the docker driver` | The default `docker` driver can't export some cache types | `docker/setup-buildx-action` / `docker buildx create --driver docker-container` |
| `denied: installation not allowed to Write organization package` / `403 Forbidden` on push | Token lacks package write permission | `permissions: packages: write`; check the org's package settings |
| `Cannot connect to the Docker daemon at tcp://docker:2375` (GitLab DinD) | DinD service not ready, TLS settings mismatched, or runner not privileged | Match `DOCKER_HOST`/`DOCKER_TLS_CERTDIR`; or migrate to rootless BuildKit |
| `toomanyrequests` pulling base images in CI | Shared runner IPs hit Docker Hub limits | Log in to Docker Hub in CI; use a mirror/pull-through cache |
| Scan step fails on unfixable CVEs | Gate too strict | `--ignore-unfixed`; documented ignore list with expiry |
| `cosign sign`: `getting signer: ... no OIDC token` | `id-token: write` permission missing | Add it to `permissions` |
| Build works locally, fails in CI on arm64 | QEMU not set up, or arm64-specific dependency issues | `setup-qemu-action`; cross-compile (topic 21) |
| Image in prod isn't what CI tested | Rebuilt per environment, or deployed by tag | Build once; promote/deploy by digest |

---

## Quick recap

- Pipeline: **lint → build → test → scan (gate) → push (SHA/semver tags) → SBOM/provenance → sign → deploy by digest**.
- CI runners are empty: export/import BuildKit cache (**registry** or **gha**, `mode=max`), or use persistent/remote builders.
- Authenticate with **OIDC** and built-in job tokens, not long-lived passwords. Don't give secrets or privileged runners to untrusted PRs.
- Builder options by risk: ephemeral hosted VM (fine) < remote builder / **rootless BuildKit** (good) < **DinD privileged** / **socket mount** (host root, avoid on shared runners).
- The **digest** from the build step is the deployment's source of truth.

**Interview questions**

1. **Q: Describe a secure container CI/CD pipeline.**
   A: Lint the Dockerfile, build with BuildKit using a registry or CI cache, run tests (a test stage or compose-based integration tests), scan with a gate on fixable critical issues, then push with immutable SHA and semver tags, attach SBOM and provenance, sign the digest keylessly via OIDC, and deploy by digest. PRs from forks build without secrets or push rights.

2. **Q: Why is Docker-in-Docker considered risky, and what are the alternatives?**
   A: DinD needs a privileged container, which gives a malicious job a path to host root. Mounting the host's Docker socket is equally dangerous. Alternatives are ephemeral VM runners, rootless BuildKit (daemonless), or a remote/isolated BuildKit builder.

3. **Q: How do you make Docker builds fast in CI when runners are ephemeral?**
   A: Export the BuildKit cache to a registry or the CI cache backend (`--cache-to type=registry|gha,mode=max`) and import it on the next run. Order Dockerfile steps for caching, keep the context small, or use a persistent remote builder whose local cache (including cache mounts) survives between jobs.
