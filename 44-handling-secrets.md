# 44 · Handling Secrets

> **Level:** ★★ intermediate
>
> **You will learn:** every place a secret (password, API key, token, private key) can leak in a container setup, why environment variables are convenient but risky, the options from weakest to strongest (env vars → mounted files → Compose/Swarm secrets → external secret managers with short-lived credentials), how to give secrets to builds vs running containers, how to detect leaked secrets, and what to do when one leaks.
>
> **You need:** topic 18 (build secrets), topic 36 (Compose secrets), topic 07 (layers are permanent), topic 25 (tmpfs).

---

## What it is

A **secret** is any value that grants access: database passwords, API keys, cloud credentials, TLS private keys, signing keys, OAuth client secrets, tokens.

Two separate problems:
1. **Build-time secrets**: credentials a build step needs (private package registries, private git repos). They must **never** end up in the image. Solved in topic 18 with `RUN --mount=type=secret` / `--ssh`.
2. **Runtime secrets**: credentials the running app needs. They must reach the process **without** being stored in the image, the compose file in git, logs, or anywhere readable by people and tools who shouldn't have them.

---

## Why it exists

Leaked secrets are among the most common causes of breaches. Attackers automatically scan public images on Docker Hub and public git repositories for keys, and use found cloud keys within **minutes**. Inside a company, a secret visible to everyone with `docker inspect` access, or printed in CI logs, is a secret shared with far more people than intended.

---

## Analogy

House keys:
- **Baking the key into the image** = engraving the key's shape on the front door. Everyone who ever sees the door can copy it.
- **Environment variable** = leaving the key **under the doormat**. Convenient. Anyone who looks there (inspect, `/proc`, crash dumps, child processes) finds it.
- **Mounted secret file** = a key in a **locked drawer** inside the house: only people inside, with the drawer key (file permissions), can get it.
- **Secret manager with short-lived credentials** = a **hotel key card** issued at check-in that **expires tomorrow** and is logged every time it's used. If it's stolen, it's soon useless, and you know who issued it.

---

## How it works inside: where secrets leak

| Place | How it leaks | Who can see it |
|---|---|---|
| **Image layers** (`COPY .env`, `RUN echo $TOKEN > file`, even if deleted later) | Permanent in the layer tar (topic 07) | Anyone who can pull the image, forever |
| **`ARG` values used in `RUN`** | Recorded in image history (topic 18) | Anyone with the image |
| **`ENV` in Dockerfile** | Stored in image config | Anyone with the image |
| **`docker run -e PASSWORD=...`** | In the container config (`docker inspect`), in **shell history**, in `ps` output of the docker command | Anyone with Docker access (= root anyway) and anyone who reads history |
| **`/proc/<pid>/environ`** | Environment of the process, readable by the same user and root | Other processes running as the same UID, host root |
| **Child processes** | Inherit the environment automatically | Every subprocess, health check, shell |
| **Logs and errors** | Apps printing their config, stack traces dumping env, debug mode | Everyone with log access, the log vendor |
| **Crash/core dumps, `/debug` endpoints** | Process memory includes secrets | Whoever gets the dump |
| **compose.yaml / .env in git** | Committed by accident | Everyone with repo access, forever in git history |
| **CI logs** | `echo $TOKEN`, `set -x`, verbose tools | Everyone with CI log access |
| **Build cache exported to a registry** | If a secret got into a layer of a build stage | Anyone with cache access |

### Options, from weakest to strongest

**1. Environment variables**
- How: `-e`, `--env-file`, Compose `environment`/`env_file`.
- Pros: universally supported, the "twelve-factor" convention, easy.
- Cons: visible in `docker inspect`, `/proc/PID/environ` and every child process; easily logged by accident; no rotation.
- Acceptable for **low-sensitivity** values, or as a last step when the platform injects them securely (e.g. Kubernetes from a secret store). Avoid for high-value credentials where you have a choice.

**2. Secret files mounted into the container**
- How: a read-only bind mount or Compose `secrets:` → `/run/secrets/<name>`, or a file written into a **tmpfs** by an init step. The app reads the file (`*_FILE` convention: `POSTGRES_PASSWORD_FILE=/run/secrets/db_password`).
- Pros: not in `docker inspect` env, not inherited by children, file permissions apply, easy to rotate (replace the file, then reload/restart).
- Cons: on standalone Docker/Compose the source file sits **unencrypted on the host disk**. Protect it (permissions `0400`/`0600`, root-owned dirs, disk encryption).

**3. Swarm secrets**
- How: `docker secret create`, `docker service create --secret`.
- Pros: **encrypted at rest** in the Swarm's Raft store (Raft = the consensus algorithm Swarm managers use to agree on and replicate cluster state), sent only to nodes running tasks that need them, mounted on an **in-memory filesystem** in the container (never written to the node's disk).
- Cons: only in Swarm mode.

**4. External secret managers (the production standard)**
- Examples: **HashiCorp Vault / OpenBao**, **AWS Secrets Manager** / **SSM Parameter Store**, **Azure Key Vault**, **Google Secret Manager**, **1Password/Doppler/Infisical**.
- How the container gets the secret:
  - **The app fetches it** at startup using an SDK and its **workload identity** (e.g. the EC2 instance's IAM role (IAM = Identity and Access Management, the cloud's system of identities and permissions), Kubernetes service account via IRSA/Workload Identity), so no secret is needed to fetch secrets,
  - **an entrypoint or init step** fetches it and writes it to a **tmpfs** file,
  - **an agent/sidecar** (Vault Agent) keeps it fresh in a shared tmpfs,
  - in Kubernetes: **External Secrets Operator** (syncs into K8s Secrets) or the **Secrets Store CSI driver** (mounts as files).
- Pros: central access control, **audit logs** (who read what, when), **rotation**, and **dynamic, short-lived credentials** (Vault can create a database user that expires after 1 hour; a leaked password is useless soon after).
- Cons: another critical system to run (or pay for). Apps must handle reload.

**5. No secret at all: identity-based access**
- The best secret is one that doesn't exist. Cloud IAM roles for workloads, **OIDC federation** from CI to cloud (GitHub Actions → AWS/GCP/Azure without stored keys), mTLS between services (mutual TLS: both sides present certificates; topic 39a), with certificates issued and rotated automatically, e.g. by a service mesh or SPIFFE/SPIRE (SPIFFE = Secure Production Identity Framework For Everyone, a standard for workload identities; SPIRE is its reference implementation). Credentials are short-lived and obtained automatically.

### Secrets in git: encrypt them, or don't commit them

- `.env`, `secrets/`, keys → `.gitignore` **and** `.dockerignore`.
- If config with secrets must live in git (GitOps), **encrypt** the values: **SOPS** (Secrets OPerationS, with age/PGP/cloud KMS keys; KMS = Key Management Service, a cloud service that holds encryption keys) or **Sealed Secrets** in Kubernetes. Decrypt at deploy time.

### Detection

- **Git:** `gitleaks`, `trufflehog`, GitHub secret scanning / push protection. Run them in pre-commit hooks and CI.
- **Images:** `trivy image --scanners secret`, Docker Scout. Also check `docker history --no-trunc` and image config.
- **Runtime:** review `docker inspect` env of running containers for anything sensitive.

### When a secret leaks: response

1. **Rotate/revoke it immediately.** Assume it's compromised. Deleting the commit or the image is **not** enough: copies and caches exist.
2. Check access logs for misuse (cloud audit logs, database logs).
3. Remove it from images (rebuild without it, delete old tags/digests from registries), from git history if needed (`git filter-repo`), and from CI logs.
4. Fix the cause (secret mounts, scanners in CI, pre-commit hooks).

---

## Commands / files

### See how env-var secrets leak

```bash
docker run -d --name leaky -e DB_PASSWORD=SuperSecret123 alpine sleep 3600
docker inspect leaky -f '{{json .Config.Env}}'
PID=$(docker inspect -f '{{.State.Pid}}' leaky)
sudo tr '\0' '\n' < /proc/$PID/environ | grep DB_
docker exec leaky sh -c 'sh -c "env | grep DB_"'
history | grep SuperSecret | tail -1
```

```
["DB_PASSWORD=SuperSecret123","PATH=/usr/local/sbin:..."]
DB_PASSWORD=SuperSecret123
DB_PASSWORD=SuperSecret123
  512  docker run -d --name leaky -e DB_PASSWORD=SuperSecret123 alpine sleep 3600
```

- Four leaks: inspect, `/proc`, child processes, shell history.

### Better: a secret file on tmpfs, read by the app

```bash
sudo install -d -m 700 /etc/myapp/secrets
printf 'SuperSecret123' | sudo tee /etc/myapp/secrets/db_password > /dev/null
sudo chmod 400 /etc/myapp/secrets/db_password
sudo chown 10001:10001 /etc/myapp/secrets/db_password

docker run -d --name better -u 10001:10001 \
  --mount type=bind,src=/etc/myapp/secrets/db_password,dst=/run/secrets/db_password,readonly \
  -e DB_PASSWORD_FILE=/run/secrets/db_password \
  alpine sleep 3600
docker inspect better -f '{{json .Config.Env}}'
docker exec better cat /run/secrets/db_password; echo
```

```
["DB_PASSWORD_FILE=/run/secrets/db_password","PATH=..."]
SuperSecret123
```

- Only the **path** is in the environment.
- File readable only by UID 10001 (`chmod 400`), in a root-only directory on the host.
- `printf ... | sudo tee` keeps the secret out of a command-line argument visible in `ps`. (It's still in your shell history as typed. For real secrets, read them from a password manager or prompt: `read -rs PW`.)

### The `*_FILE` pattern in an entrypoint (for apps that only read env vars)

```sh
#!/bin/sh
set -e
# For every VAR_FILE variable, read the file into VAR (only for this process tree).
for v in $(env | sed -n 's/^\([A-Z_]*\)_FILE=.*/\1/p'); do
  f=$(printenv "${v}_FILE")
  export "$v"="$(cat "$f")"
  unset "${v}_FILE"
done
exec "$@"
```

- This still ends up as an environment variable **inside** the app process (visible in `/proc/PID/environ`), but it's no longer in `docker inspect`, the compose file or the image. It's a common compromise for apps that can't read files.

### Swarm secrets (encrypted at rest)

```bash
docker swarm init
printf 'SuperSecret123' | docker secret create db_password -
docker service create --name app --secret db_password alpine sh -c 'cat /run/secrets/db_password; sleep 3600'
docker secret inspect db_password --format '{{.Spec.Name}} {{.CreatedAt}}'
docker exec $(docker ps -q -f name=app) sh -c 'mount | grep secrets'
docker service rm app; docker secret rm db_password; docker swarm leave --force
```

```
db_password 2026-11-17 11:00:01.123 +0000 UTC
tmpfs on /run/secrets/db_password type tmpfs (ro,relatime)
```

- The secret value can't be read back with `docker secret inspect`. In the container it's a **tmpfs** file.

### Fetch from AWS Secrets Manager at startup (pattern)

```sh
#!/bin/sh
# entrypoint: uses the instance/task IAM role, no stored keys
set -e
DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id prod/shop/db --query SecretString --output text)
umask 077
printf '%s' "$DB_PASSWORD" > /run/secrets/db_password      # /run/secrets is a tmpfs mount
unset DB_PASSWORD
exec "$@"
```

```bash
docker run --tmpfs /run/secrets:size=1m,mode=0700,uid=10001 -u 10001 myapp:1.0
```

(In practice the app often uses the cloud SDK directly instead of the CLI.)

### Scan for secrets

```bash
docker run --rm -v "$PWD":/repo zricethezav/gitleaks:latest detect --source /repo --no-banner
trivy image --scanners secret myapp:1.0
```

```
Finding:     aws_secret_access_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
Secret:      wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
RuleID:      aws-access-token
File:        config/dev.env
Line:        3
Commit:      7a8b9c…
...
leaks found: 1
```

---

## Hands-on lab

1. Run the `leaky` demo and find the secret in four places.
2. Run the `better` demo. Confirm `docker inspect` shows only the path, and that a second container running as a different UID (`-u 2000`) can't read the file.
3. Write a Compose stack where Postgres gets its password from `secrets:` (`POSTGRES_PASSWORD_FILE`), and the app reads `/run/secrets/db_password` (topic 36). Add `secrets/` to `.gitignore` and `.dockerignore`.
4. Put a fake AWS key in a file in a git repo and run gitleaks. Then add a pre-commit hook that runs it.
5. Build an image that (wrongly) `COPY`s a `.env`, then run `trivy image --scanners secret` on it.
6. Clean up: `docker rm -f leaky better; sudo rm -rf /etc/myapp`

---

## Where you'll use it

- **Every production deployment** needs a decided secret strategy: who stores them, how they reach containers, how they rotate.
- **Compose on one host:** Compose secrets from root-only files, `*_FILE` variables.
- **Cloud:** IAM roles + Secrets Manager/Key Vault/Secret Manager. OIDC from CI instead of stored cloud keys (topic 46).
- **Kubernetes:** External Secrets Operator or Secrets Store CSI driver with a vault, encrypted etcd, RBAC on Secrets (topic 53).
- **Incident response:** fast rotation playbooks.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Secret found in an image layer or history | `COPY`/`ARG`/`ENV` misuse | Rotate; rebuild with secret mounts; purge old images |
| Secret visible in `docker inspect` | Passed as an env var | Mounted files / secret manager |
| Secret in CI logs | Printed by a script or `set -x` | Mask secrets in CI, remove echo/debug, rotate |
| App can't read `/run/secrets/...` (permission denied) | File owner/mode doesn't match the container user | Set host file owner to the container UID, mode 0400 |
| App still uses the old password after rotation | Secret read only at startup | Restart/reload; or make the app re-read files; dynamic credentials with renewal |
| Secret manager outage stops app startups | Hard dependency at boot | Caching agent, retries, sensible timeouts; HA for the secret store |
| `.env` with production secrets committed | Not in `.gitignore` | Rotate; remove from history; add pre-commit secret scanning |

---

**Next:** topic 44a covers detecting attacks at runtime and the full incident-response procedure, including which secrets to rotate after a compromise.

## Quick recap

- Secrets leak through image layers, ARG/ENV, `docker inspect`, `/proc/PID/environ`, child processes, logs, shell history, git and CI logs.
- Build-time: `RUN --mount=type=secret` / `--ssh` only. Runtime, from weakest to strongest: env vars → mounted files (`/run/secrets`, `*_FILE`) → Swarm secrets (encrypted, tmpfs) → external secret managers with short-lived credentials → identity-based access with no secret at all.
- Keep secrets out of git (`.gitignore`, `.dockerignore`), or encrypt them (SOPS/Sealed Secrets).
- Scan repos and images (gitleaks, trufflehog, Trivy).
- **If a secret leaks: rotate first**, then clean up and fix the cause.

**Interview questions**

1. **Q: Why are environment variables a weak way to pass secrets to containers?**
   A: They're visible in `docker inspect` and `/proc/<pid>/environ`, inherited by all child processes, often logged accidentally, and can end up in shell history or compose files. Mounted secret files or a secret manager limit exposure and support rotation.

2. **Q: How would you manage secrets for containers in production?**
   A: Store them in a central secret manager (Vault, AWS Secrets Manager…) with access tied to workload identity. Deliver them as files on tmpfs or fetch them at startup, prefer short-lived dynamic credentials, rotate regularly, audit access, and never bake them into images or git.

3. **Q: A developer pushed an image containing an AWS key. What do you do?**
   A: Immediately deactivate and rotate the key, and check CloudTrail for misuse. Then delete the image tags and digests from the registry, rebuild without the key (using build secrets), and add secret scanning to CI and pre-commit hooks.
