# 22 · Tagging, Pushing and Private Registries

> **Level:** ★★ intermediate
>
> **You will learn:** how to tag images for real releases (semver, git SHA, why never only `latest`), how `docker login` and push/pull work underneath, where credentials are stored and how credential helpers keep them safe, Docker Hub pull rate limits, logging in to GHCR and ECR, running your own registry, promoting images between environments without rebuilding, deploying by digest, and moving images without a registry (`save`/`load`).
>
> **You need:** topics 06–07 (names, digests, layers), topic 21 (multi-platform index).

---

## What it is

- **Tagging** = giving an image a name in a registry: `docker tag local-image registry/namespace/repo:tag`.
- **Pushing** = uploading the image's layers, config and manifest(s) to that registry: `docker push`.
- **A private registry** = a registry where only authorised users can pull and/or push: your company's images live here.
- **Logging in** = getting and storing credentials so the Docker daemon can authenticate to a registry.

---

## Why it exists

Images must get from the build machine (CI) to every server that runs them. A registry is the shared, versioned, access-controlled store in the middle. **Tags** let humans and deploy tools say *which* version. **Digests** guarantee *exactly* which bytes.

---

## Analogy

A **warehouse with labelled shelves**:
- The **registry** is the warehouse, with locked doors (auth).
- **Pushing** = delivering boxes (layers) to the warehouse. If an identical box is already on a shelf, the delivery driver skips it.
- **Tags** = shelf labels. You can move a label to another box (mutable). Some warehouses forbid moving labels (**immutable tags**).
- **Digest** = the box's tamper-proof serial number.
- **`docker save`** = putting the boxes in your car and driving them over yourself.

---

## How it works inside

### Push, step by step

`docker push ghcr.io/acme/api:1.4.2`:

1. dockerd reads the credentials for `ghcr.io` (from your CLI config or a credential helper).
2. **Auth:** asks the registry's token service for a token with `push` and `pull` permission on `acme/api`.
3. **For each layer:** `HEAD /v2/acme/api/blobs/<digest>`. If the registry already has it → `Layer already exists`, skip. If another repository on the same registry has it → **cross-repo mount** (`Mounted from acme/base`), a server-side copy with no upload. Otherwise upload it (`POST` then `PUT` of the blob).
4. Upload the **config** blob.
5. `PUT /v2/acme/api/manifests/1.4.2` with the manifest (or index). **This is the moment the tag exists/moves.**
6. The registry replies with the manifest's **digest**, printed as `1.4.2: digest: sha256:… size: …`.

Layers are uploaded before the manifest, so a half-finished push never leaves a tag pointing at missing data.

### Where credentials are stored

`docker login` saves credentials in **`~/.docker/config.json`** of the user running the CLI (the CLI passes them to dockerd with each request):

```json
{
  "auths": {
    "ghcr.io": { "auth": "dHVzaGFyOmdocF9hYmMxMjM0NTY3ODk=" }
  }
}
```

- `auth` is just **base64** of `username:password`. Base64 is an encoding, **not encryption**: `echo dHVz… | base64 -d` shows the password. On a shared server, that's a leak waiting to happen. Docker warns: `WARNING! Your credentials are stored unencrypted in '/home/…/.docker/config.json'`.

**Credential helpers** fix this. They're small programs that store secrets in a proper store, and Docker calls them instead:

| Helper | Stores in |
|---|---|
| `docker-credential-desktop` | Docker Desktop (Windows Credential Manager / macOS Keychain) |
| `docker-credential-secretservice` | Linux desktop keyring (GNOME Keyring) |
| `docker-credential-pass` | `pass` (GPG-encrypted files), good for headless Linux |
| `docker-credential-ecr-login` | Gets fresh AWS ECR tokens from your AWS credentials automatically |
| `docker-credential-gcr`, `acr-env` | Google and Azure equivalents |

Config with helpers:

```json
{
  "credsStore": "pass",
  "credHelpers": {
    "123456789012.dkr.ecr.ap-south-1.amazonaws.com": "ecr-login",
    "europe-docker.pkg.dev": "gcloud"
  }
}
```

- `credsStore` = the default helper for all registries. `credHelpers` = a specific helper per registry.
- Docker runs the program `docker-credential-<name>` (it must be on the `PATH`).

**On servers and in CI**, prefer short-lived tokens (OIDC in CI, ECR tokens, robot accounts with pull-only rights) over personal passwords (topics 44, 46).

### Docker Hub rate limits

Docker Hub limits how many image **pulls** (manifest requests) you can make:
- **Anonymous** pulls are limited **per IP address**. Behind a company NAT (where many machines share one IP) or on shared CI runners, everyone shares one small allowance.
- **Logged-in** free users get more, and **paid** accounts get much more or no limit.
- The exact numbers change. Check Docker's current pricing/usage page, and measure your remaining allowance with the headers shown below.

When you hit it: `toomanyrequests: You have reached your pull rate limit`. Deploys and autoscaling fail. Ways to avoid it:
1. Log in (even a free account helps; paid org accounts are better).
2. A **pull-through cache / mirror**: a registry in your network that fetches from Docker Hub once and serves everyone (`registry-mirrors` in `daemon.json`, topic 34; Harbor proxy cache; ECR/Artifact Registry pull-through cache rules).
3. Copy the images you depend on into your own registry and pull them from there.

### Tag strategy for real releases

| Tag | Example | Mutable? | Use |
|---|---|---|---|
| **Git commit SHA** | `api:3f9a2c1` | Never moved | Exact traceability. CI pushes one per build |
| **Semver release** | `api:1.4.2` | Should never move | Releases |
| Floating minor/major | `api:1.4`, `api:1` | Moves on each patch/minor | For consumers who want automatic patches |
| `latest` | `api:latest` | Moves | Convenience only. **Never deploy it** |
| Environment tags | `api:prod`, `api:staging` | Moves | Risky: "what's in prod?" changes silently. Prefer deploying exact versions/digests, and keep environment state in git (GitOps: the desired state of every environment lives in a git repository, and a tool applies whatever is merged) |

Good practice:
- CI tags each build with the **git SHA** (and the semver on release).
- Deployment manifests reference **`repo:1.4.2@sha256:…`**, readable and exact.
- Turn on **immutable tags** in the registry (ECR, Harbor and Artifact Registry support it; GHCR has no equivalent immutable-tag setting, so there you protect release tags by process and deploy by digest) so `1.4.2` can't be overwritten.

### Promoting without rebuilding

**Build once, deploy many**: the image tested in staging must be the **same bytes** in production. Promote by **copying or re-tagging**, never rebuild:

```bash
docker buildx imagetools create -t registry.prod.acme.com/api:1.4.2 registry.dev.acme.com/api:1.4.2
```

- Copies the whole index (all platforms) registry-to-registry. The **digest stays the same**.
- Other tools that do this: `crane copy`, `skopeo copy`, `oras cp`.

### Moving images without a registry: `save` / `load`

For air-gapped servers (no internet), or quick transfers:

```bash
docker save -o api-1.4.2.tar api:1.4.2           # image(s) → tar (layers, config, manifest, tags)
scp api-1.4.2.tar server:/tmp/
ssh server docker load -i /tmp/api-1.4.2.tar
```

- `save`/`load` keep **everything**: layers, history, tags, config.
- Don't confuse with `docker export` / `docker import`: those work on a **container's flattened filesystem** (one layer, **no** CMD/ENV/history). Use them only to make a base image from a filesystem.

---

## Commands / files

### Tag and push to Docker Hub

```bash
docker login
docker tag hello-flask:1.0 <dockerhub-user>/hello-flask:1.0
docker push <dockerhub-user>/hello-flask:1.0
```

```
Login Succeeded
The push refers to repository [docker.io/<dockerhub-user>/hello-flask]
5f4e3d2c1b0a: Pushed
9a8b7c6d5e4f: Pushed
2c3d4e5f6a7b: Mounted from library/python
e1f2a3b4c5d6: Mounted from library/python
1.0: digest: sha256:4c1a9e…f8 size: 1786
```

- `Mounted from library/python` = Docker Hub already had these base layers. Server-side copy, no upload.
- `digest:` = the manifest digest of what you pushed. **Record this for deployments.**

`docker login` for Docker Hub uses a web-based device flow on new CLIs, or asks for a username and a **personal access token** (recommended over your password).

### Log in without exposing the password

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u <github-user> --password-stdin
```

- `--password-stdin` = read the password from standard input, so it doesn't appear in shell history or in `ps` output (which `-p password` would).

### Amazon ECR

```bash
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

- ECR tokens expire after **12 hours**. The `ecr-login` credential helper refreshes them automatically.

### Check your Docker Hub pull limit

```bash
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
curl -s --head -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest | grep -i ratelimit
```

Example of the **format** (the numbers are illustrative only):

```
ratelimit-limit: 100;w=21600
ratelimit-remaining: 96;w=21600
```

- `limit: N;w=S` = N pulls per window of S seconds, for this IP (anonymous) or account (logged in). Docker Hub **changed its limits in 2025** and may change them again, so never hard-code numbers from blog posts (or from these notes). Read the live headers, and check Docker's current pricing/usage page.
- `remaining` = what's left. A HEAD request like this doesn't use up your allowance.

### Run your own registry

```bash
docker run -d --name registry -p 5000:5000 -v registry-data:/var/lib/registry registry:3
docker tag hello-flask:1.0 localhost:5000/hello-flask:1.0
docker push localhost:5000/hello-flask:1.0
curl -s localhost:5000/v2/_catalog
curl -s localhost:5000/v2/hello-flask/tags/list
```

```
{"repositories":["hello-flask"]}
{"name":"hello-flask","tags":["1.0"]}
```

- `registry:3` = the official **Distribution** registry (the CNCF project behind Docker Hub's registry). `registry:2` is the older line you'll still see in many guides.
- `-v registry-data:/var/lib/registry` = keep the stored images in a named volume (topic 24).
- `localhost:5000` is allowed **without TLS** by default. Any other address needs HTTPS, or an `insecure-registries` entry in `daemon.json` (for labs only; topic 34).
- The `/v2/...` URLs are the **OCI Distribution API** (topic 03).

For production, add TLS, authentication, and storage on S3 or similar, or use **Harbor** (adds a UI, RBAC (Role-Based Access Control), scanning, replication, proxy cache, immutable tags).

### Pull and run by digest

```bash
DIGEST=$(docker buildx imagetools inspect localhost:5000/hello-flask:1.0 --format '{{json .Manifest.Digest}}' | tr -d '"')
docker run --rm -d --name byd -p 8001:8000 localhost:5000/hello-flask@$DIGEST
docker inspect -f '{{.Config.Image}}' byd
```

```
localhost:5000/hello-flask@sha256:4c1a9e…f8
```

### save and load

```bash
docker save -o hf.tar hello-flask:1.0
ls -lh hf.tar
tar -tf hf.tar | head
docker rmi hello-flask:1.0
docker load -i hf.tar
```

```
-rw------- 1 tushar tushar 52M Nov 17 12:01 hf.tar
blobs/
blobs/sha256/
blobs/sha256/1f9e…
index.json
manifest.json
oci-layout
repositories
Loaded image: hello-flask:1.0
```

- The tar is in **OCI image layout** (`index.json`, `oci-layout`, `blobs/sha256/…`) plus Docker's legacy `manifest.json`, so other tools can read it too.
- Compress for transfer: `docker save hello-flask:1.0 | gzip > hf.tar.gz` and `docker load < hf.tar.gz` (load detects gzip).

---

## Hands-on lab

1. Run the local registry above. Tag and push `hello-flask:1.0`. Push it again and see every layer show `Layer already exists`.
2. **Mutable tags in action:** change `app.py`, rebuild as `hello-flask:1.0`, push again. The tag now points to a **new digest**. Pull by the **old** digest: still works. That's why deployments use digests.
3. **Promotion:** run a second registry on port 5001 (`-p 5001:5000 --name registry2`). Copy with `docker buildx imagetools create -t localhost:5001/hello-flask:1.0 localhost:5000/hello-flask:1.0`. (Run `docker buildx use default` first if you created other builders.) Compare the digests in both registries: identical.
4. **Credentials file:** `docker login localhost:5000` doesn't need a login (no auth configured), so instead look at `cat ~/.docker/config.json` after logging in to Docker Hub, and decode the `auth` field with `base64 -d` if there is one. (Then think about a credential helper.)
5. `save`/`load` the image, and compare with `docker export` of a container from it (`docker export <ctr> | tar -t | head`): no manifest, no config, just files.
6. Clean up: `docker rm -f registry registry2 byd; docker volume rm registry-data; rm hf.tar`

---

## Where you'll use it

- **CI/CD:** build → tag (SHA + semver) → push → deploy by digest (topic 46).
- **Environment promotion:** dev → staging → prod registries with `imagetools create`/`crane`/`skopeo`.
- **Rate-limit incidents:** autoscaling failures traced to Docker Hub limits; fixed with mirrors and auth.
- **Air-gapped deployments:** `save` → transfer → `load`, or a local registry in the isolated network.
- **Security:** credential helpers, robot accounts, immutable tags, removing leaked images.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `denied: requested access to the resource is denied` | Not logged in, wrong namespace, or no push rights | `docker login <registry>`; check the name's namespace matches your account/org |
| `unauthorized: authentication required` | Missing or expired credentials (ECR tokens expire after 12 h) | Log in again, or use a credential helper |
| `name unknown: repository name not known to registry` | Repository doesn't exist (ECR requires creating it first) | Create the repository |
| `http: server gave HTTP response to HTTPS client` | Registry is plain HTTP but isn't `localhost` | Enable TLS (proper fix) or add it to `insecure-registries` (lab only) |
| `x509: certificate signed by unknown authority` | Registry uses a private CA | Put the CA cert in `/etc/docker/certs.d/<registry:port>/ca.crt` |
| `toomanyrequests: You have reached your pull rate limit` | Docker Hub limit | Authenticate; mirror/pull-through cache; copy images to your registry |
| `manifest invalid` / `tag invalid: The image tag '1.4.2' already exists ... and cannot be overwritten because the repository is immutable` | Immutable-tag policy | Good: bump the version instead |
| `error storing credentials - err: exit status 1, out: 'pass not initialized'` | Credential helper misconfigured | Initialise `pass` (`pass init <gpg-id>`) or remove `credsStore` |
| Push very slow every time | Base layers not shared (different base per build), or no cross-repo mount | Common bases; push to the same registry as the base |

---

**Later:** running a registry in production (garbage collection, retention, Harbor, pull-through caches) is topic 46a.

## Quick recap

- `docker tag` names an image for a registry. `docker push` uploads missing layers, then the config, then the manifest (which creates or moves the tag) and prints the **digest**.
- Credentials live in `~/.docker/config.json` as base64 (not encrypted). Use **credential helpers** and `--password-stdin`.
- Tag with git SHA + semver, deploy by **digest**, never only `latest`, and make release tags immutable.
- Docker Hub rate-limits pulls. Authenticate and use mirrors/pull-through caches.
- Promote images registry-to-registry without rebuilding (same digest). `save`/`load` for air-gapped transfer (`export`/`import` lose metadata).

**Interview questions**

1. **Q: How do you guarantee production runs exactly the image that passed testing?**
   A: Build once, push, and record the digest. Promote the same image (copy/retag) between environments, and deploy by digest, ideally with immutable tags.

2. **Q: Where does Docker store registry credentials, and is it secure?**
   A: In `~/.docker/config.json`, by default as base64 (just encoding, not encryption). Use a credential helper (OS keychain, `pass`, cloud helpers) and short-lived tokens or robot accounts.

3. **Q: What's the difference between `docker save` and `docker export`?**
   A: `save` writes a whole image (all layers, config, history, tags) to a tar, and `load` restores it. `export` writes a container's flattened filesystem only, with no image config or history. `import` makes a single-layer image from it.
