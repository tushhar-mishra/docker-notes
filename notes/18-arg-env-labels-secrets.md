# 18 · ARG, ENV, Labels and Build Secrets

> **Level:** ★★ intermediate
>
> **You will learn:** the difference between `ARG` (build-time variables) and `ENV` (variables baked into the image for runtime), their scope rules (where they're visible), Dockerfile variable syntax, BuildKit's automatic platform ARGs, `LABEL` and the standard OCI label keys, and how to use secrets and SSH keys **during a build** without them ever entering a layer.
>
> **You need:** topics 14–17 (Dockerfile, cache, stages). Topic 07 (layers are permanent).

---

## What it is

| | `ARG` | `ENV` |
|---|---|---|
| Set with | `ARG NAME=default` + `docker build --build-arg NAME=value` | `ENV NAME=value` (+ `docker run -e` to override) |
| Visible during build | Yes (after it's declared, in its stage) | Yes (after it's set) |
| Visible in the running container | **No** | **Yes** |
| Stored in the image | Not as a variable. **But values used by `RUN` appear in the image history.** | Yes, in the image config |
| Use for | Versions, build options, target platform | App defaults (`PORT`, `PATH`, `NODE_ENV`) |

- **`LABEL`** = metadata attached to the image (who made it, git commit, source URL, version). Doesn't affect how it runs.
- **Build secrets** = passwords, tokens or keys a build step needs (e.g. to download private packages), given to that one step as a **temporary file or variable** that's never saved in the image.

---

## Why it exists

- One Dockerfile should build many variants: different versions, base images, platforms. That's `ARG`.
- Apps need sensible defaults at runtime that can be overridden per environment. That's `ENV`.
- Teams and tools need to know where an image came from (which repo, which commit) for audits, scanning and incident response. That's `LABEL`.
- Builds often need credentials (private npm/PyPI registries, private git repos). Putting them in `ARG`, `ENV` or `COPY` **leaks them permanently** into the image (topic 07: layers are immutable). Build secrets solve that.

---

## Analogy

Ordering a custom cake:
- **ARG** = the instructions you give the baker ("chocolate, size large"). They shape the cake, but they're not part of it. Still, someone might **write them on the order slip that comes with the box** (the image history).
- **ENV** = the label stuck **on** the box ("keep refrigerated"). It goes with the cake everywhere.
- **LABEL** = the bakery's sticker: who made it, when, batch number.
- **Build secret** = the key to the special-ingredients storeroom. The baker uses it during baking and hands it back. It's never inside the cake, never on the box.

---

## How it works inside

### ARG scope rules

```dockerfile
# syntax=docker/dockerfile:1
ARG PY_VERSION=3.13                 # (1) global ARG, before any FROM

FROM python:${PY_VERSION}-slim AS build   # (2) usable in FROM lines
ARG PY_VERSION                      # (3) must be re-declared to use it inside the stage
RUN echo "building with $PY_VERSION"

ARG APP_VERSION=dev                 # (4) stage-local ARG
RUN echo "app $APP_VERSION" > /version.txt

FROM python:${PY_VERSION}-slim AS final
# APP_VERSION is NOT visible here unless declared again
COPY --from=build /version.txt /
```

1. An `ARG` **before the first `FROM`** is global, but is only usable **in `FROM` lines** by default.
2. That's how you choose a base image version at build time.
3. To use it **inside** a stage, write `ARG NAME` (no value) again in that stage. It then inherits the global value.
4. An `ARG` inside a stage exists only in that stage, from that line on.

`--build-arg` sets values:

```bash
docker build --build-arg PY_VERSION=3.12 --build-arg APP_VERSION=1.4.2 -t myapp .
```

### ARG and the cache

An ARG affects the cache key of the `RUN` steps that **use it** (BuildKit is precise about this). Changing `APP_VERSION` reruns only the steps after its declaration that depend on it. Tip: declare frequently-changing ARGs (like a git SHA) **as late as possible**.

### ARG values are not secret

With BuildKit, a `RUN` step records the build args it used in the image history:

```
RUN |1 APP_VERSION=1.4.2 /bin/sh -c echo "app $APP_VERSION" > /version.txt
```

- `|1 APP_VERSION=1.4.2` = 1 build arg, with its value, **visible to anyone who pulls the image** (`docker history --no-trunc`).
- So `--build-arg NPM_TOKEN=...` leaks the token. `docker build` warns about this (rule `SecretsUsedInArgOrEnv`; topic 20).

### Automatic platform ARGs

BuildKit sets these for you (declare them with `ARG NAME` to use them):

| ARG | Example value | Meaning |
|---|---|---|
| `TARGETPLATFORM` | `linux/arm64` | The platform being built **for** |
| `TARGETOS` | `linux` | |
| `TARGETARCH` | `arm64` / `amd64` | |
| `TARGETVARIANT` | `v7` (for arm/v7) | |
| `BUILDPLATFORM` | `linux/amd64` | The platform the build **runs on** |
| `BUILDARCH` etc. | | |

Used heavily for cross-compiling multi-platform images (topic 21):

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25 AS build
ARG TARGETOS TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app .
```

There are also predefined proxy ARGs (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` and lowercase versions) that you can pass with `--build-arg` without declaring them. They're **excluded from history** on purpose.

### ENV details

```dockerfile
ENV APP_HOME=/app \
    PORT=8000 \
    PATH="/opt/venv/bin:${PATH}"
```

- Stored in the image config. Every container gets them, and `docker run -e PORT=9000` overrides.
- `${PATH}` refers to the existing value, so this **prepends** to PATH.
- Anyone with the image can read every ENV with `docker image inspect`. **Never put secrets in ENV.**
- `ENV DEBIAN_FRONTEND=noninteractive` is a common mistake: it then stays set for the app at runtime. For build-only settings, prefer `ARG DEBIAN_FRONTEND=noninteractive` or set it inline in the `RUN` command.

### Variable substitution syntax

Works in `FROM`, `RUN` (expanded by the shell), `COPY`, `ADD`, `ENV`, `WORKDIR`, `USER`, `LABEL`, `EXPOSE` and others:

| Syntax | Meaning |
|---|---|
| `$VAR` or `${VAR}` | The value |
| `${VAR:-default}` | The value, or `default` if VAR is unset or empty |
| `${VAR:+alt}` | `alt` if VAR is set, otherwise empty |
| `${VAR:?message}` | Fail the build with `message` if VAR is unset (newer Dockerfile syntax) |

Exec-form `CMD`/`ENTRYPOINT` (JSON arrays) are **not** expanded at runtime (topic 15).

### Labels and the OCI standard keys

```dockerfile
LABEL org.opencontainers.image.title="payments-api" \
      org.opencontainers.image.description="Handles card payments" \
      org.opencontainers.image.source="https://github.com/acme/payments-api" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.vendor="Acme Corp"
```

- These `org.opencontainers.image.*` keys are defined by the **OCI image spec**. Registries (GHCR uses `source` to link the image to its repository), scanners and dashboards understand them.
- Labels from the base image are **inherited**. Yours override keys with the same name.
- Add labels without editing the Dockerfile: `docker build --label org.opencontainers.image.revision=$(git rev-parse HEAD) .`
- **Annotations** are a related OCI feature: metadata on the **manifest/index** rather than in the config (`docker buildx build --annotation "index:org.opencontainers.image.description=..."`). Some registries display those instead. Use labels by default.

### Build secrets: how they stay out of the image

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

What happens:
1. The CLI reads `~/.npmrc` and sends it to BuildKit **over the build session**, not as part of the context.
2. For this one `RUN` step, BuildKit mounts it as a file at `/root/.npmrc` on a **tmpfs** (a filesystem in RAM).
3. When the step ends, the mount is removed **before** the layer is saved. The secret's content isn't in any layer, history or cache key.

Other forms:

```dockerfile
# Default target is /run/secrets/<id>
RUN --mount=type=secret,id=api_token \
    TOKEN=$(cat /run/secrets/api_token) ./download-private-thing.sh

# Expose the secret as an environment variable for this step only
RUN --mount=type=secret,id=api_token,env=API_TOKEN \
    ./download-private-thing.sh

# Fail if the secret wasn't provided
RUN --mount=type=secret,id=api_token,required=true ...
```

```bash
docker build --secret id=api_token,env=API_TOKEN .        # take it from your shell's $API_TOKEN
docker build --secret id=api_token,src=./token.txt .      # take it from a file
```

### SSH forwarding: clone private repos without copying keys

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.22
RUN apk add --no-cache git openssh-client
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
RUN --mount=type=ssh git clone git@github.com:acme/private-lib.git /src/private-lib
```

```bash
eval $(ssh-agent) && ssh-add ~/.ssh/id_ed25519
docker build --ssh default .
```

- `--ssh default` = forward your running **ssh-agent** (a program that holds your unlocked keys) into the build.
- `RUN --mount=type=ssh` = that step can use the agent through a socket. **The private key file itself never enters the build.**
- `ssh-keyscan` adds GitHub's host key so `git` doesn't ask "are you sure you want to connect?".

### The wrong ways (and why)

| Wrong | Why it leaks |
|---|---|
| `ARG NPM_TOKEN` + `--build-arg NPM_TOKEN=…` | Value recorded in image history for any `RUN` using it |
| `ENV NPM_TOKEN=…` | Stored in the image config: `docker inspect` shows it |
| `COPY .npmrc /root/` then `RUN rm /root/.npmrc` | The file stays in the `COPY` layer (topic 07) |
| `COPY id_rsa` | Same, and it's your private key |
| Secret in a multi-stage **build** stage only | Safer (not in the final image), but it's in the build cache and in any pushed cache or `--target`ed image. Use secret mounts anyway |

---

## Commands / files

### ARG and ENV in action

```dockerfile
# syntax=docker/dockerfile:1
ARG BASE=alpine:3.22
FROM ${BASE}
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION} \
    PORT=8080
LABEL org.opencontainers.image.version=${APP_VERSION}
RUN echo "built $APP_VERSION" > /build.txt
CMD ["sh", "-c", "echo running $APP_VERSION on port $PORT; cat /build.txt"]
```

- `ENV APP_VERSION=${APP_VERSION}` = the common pattern to **carry a build arg into runtime** on purpose (fine for non-secret values).

```bash
docker build --build-arg APP_VERSION=1.4.2 -t argenv .
docker run --rm argenv
docker run --rm -e PORT=9000 argenv
docker image inspect argenv -f '{{json .Config.Env}}'
docker image inspect argenv -f '{{json .Config.Labels}}'
docker history --no-trunc argenv --format '{{.CreatedBy}}' | grep build.txt
```

```
running 1.4.2 on port 8080
built 1.4.2
running 1.4.2 on port 9000
built 1.4.2
["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","APP_VERSION=1.4.2","PORT=8080"]
{"org.opencontainers.image.version":"1.4.2"}
RUN |1 APP_VERSION=1.4.2 /bin/sh -c echo "built $APP_VERSION" > /build.txt # buildkit
```

- The last line shows how build-arg values end up in history.

### A secret mount, proven safe

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.22
RUN --mount=type=secret,id=token,env=TOKEN \
    echo "token length is ${#TOKEN}" && echo "used the token" > /result.txt
```

```bash
export TOKEN='s3cr3t-value'
docker build --secret id=token,env=TOKEN --progress=plain -t sectest . 2>&1 | grep 'token length'
docker history --no-trunc sectest | grep -c s3cr3t
docker run --rm sectest sh -c 'env | grep TOKEN; ls /run/secrets 2>&1'
```

```
#6 0.215 token length is 12
0
ls: /run/secrets: No such file or directory
```

- The step could use the secret (length 12).
- `grep -c s3cr3t` found **0** matches in the history. The runtime container has no `TOKEN` and no secrets folder.

### The build check warns you

```bash
cat > Dockerfile.bad <<'EOF'
FROM alpine:3.22
ARG DB_PASSWORD
ENV API_KEY=abc
EOF
docker build --check -f Dockerfile.bad .
```

```
WARNING: SecretsUsedInArgOrEnv - https://docs.docker.com/go/dockerfile/rule/secrets-used-in-arg-or-env/
Do not use ARG or ENV instructions for sensitive data (ARG "DB_PASSWORD")
 Dockerfile.bad:2
...
WARNING: SecretsUsedInArgOrEnv - ...
Do not use ARG or ENV instructions for sensitive data (ENV "API_KEY")
```

- `--check` runs the Dockerfile **linter rules** without building (topic 20).

---

## Hands-on lab

1. Build the `argenv` example with two different `APP_VERSION` values and compare `docker history --no-trunc`.
2. **Leak demo (to remember forever):**
   ```dockerfile
   # syntax=docker/dockerfile:1
   FROM alpine:3.22
   ARG TOKEN
   RUN echo "pretend we use $TOKEN"
   COPY secret.txt /tmp/secret.txt
   RUN rm /tmp/secret.txt
   ```
   ```bash
   echo "TOP-SECRET" > secret.txt
   docker build --build-arg TOKEN=abc123 -t leaky .
   docker history --no-trunc leaky | grep abc123
   docker save leaky -o leaky.tar && mkdir leaky && tar -xf leaky.tar -C leaky
   grep -r "TOP-SECRET" leaky/ | head
   ```
   `docker save` writes the image (all layers) to a tar (topic 22). If grep finds nothing, the layers are gzip-compressed: unpack them with `for f in leaky/blobs/sha256/*; do tar -xzOf "$f" 2>/dev/null; done | grep -a TOP-SECRET`. You'll find the token in history and the file in the `COPY` layer, despite the `rm`.
3. Rewrite it with `--mount=type=secret` and repeat the checks. Nothing found.
4. Add OCI labels with `--label org.opencontainers.image.revision=$(git rev-parse HEAD 2>/dev/null || echo test)` and read them back with `docker image inspect`.
5. Clean up: `docker rmi argenv sectest leaky; rm -rf leaky leaky.tar secret.txt Dockerfile.bad`

---

## Where you'll use it

- **CI pipelines:** `--build-arg APP_VERSION=$TAG`, `--label org.opencontainers.image.revision=$GIT_SHA` on every build (topic 46).
- **Private dependencies:** `--secret` for npm/pip/Maven credentials, `--ssh` for private git repos.
- **One Dockerfile, many variants:** base image version, debug vs release builds, multi-platform builds.
- **Audits and incident response:** "which commit is this image from?" → read the `revision` label.
- **Security reviews:** checking image history and config for leaked tokens.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| ARG is empty inside a stage | Declared before `FROM` (global) but not re-declared in the stage | Add `ARG NAME` inside the stage |
| `[Warning] One or more build-args [FOO] were not consumed` | You passed `--build-arg FOO` but the Dockerfile never declares `ARG FOO` | Declare it, or remove the flag |
| Changing an ARG rebuilds everything | The ARG is declared early and used by early steps | Declare it as late as possible |
| Secret file is empty / missing in the RUN step | Wrong `id`, or `--secret` not passed | Match the `id` exactly. Use `required=true` to fail loudly |
| `failed to solve: unsupported secret mount` / unknown flag `--mount` | Legacy builder or old frontend | Use BuildKit and `# syntax=docker/dockerfile:1` |
| `Permission denied (publickey)` with `--mount=type=ssh` | No agent running, the key isn't added, or `--ssh` wasn't passed | `ssh-add -l` to check; `docker build --ssh default` |
| `Host key verification failed` | known_hosts is missing in the build | `ssh-keyscan github.com >> ~/.ssh/known_hosts` in a `RUN` before |
| Token found in an image during a security scan | ARG/ENV/COPY used for secrets | **Rotate the token right away** (assume it's compromised), rebuild with secret mounts, and delete the old images from registries |

---

## Quick recap

- `ARG` = build-time only, and scoped (global ARGs work in `FROM`; re-declare inside a stage). `ENV` = baked into the image for runtime and overridable with `-e`.
- **Neither is for secrets.** ARG values appear in history, ENV in the config.
- BuildKit gives you `TARGETPLATFORM`/`TARGETARCH`/`BUILDPLATFORM` for cross-builds.
- Use `org.opencontainers.image.*` labels (source, revision, version, created) for traceability.
- `RUN --mount=type=secret` and `RUN --mount=type=ssh` give a step credentials without writing them to any layer.

**Interview questions**

1. **Q: What's the difference between ARG and ENV?**
   A: ARG exists only during the build and can be set with `--build-arg`. ENV is stored in the image and is present in every container at runtime (overridable with `-e`). ARG values used in RUN still show up in image history.

2. **Q: How do you use a private package-registry token during a build without leaking it?**
   A: Pass it with `docker build --secret id=…` and consume it in a `RUN --mount=type=secret,id=…` step. BuildKit mounts it in memory for that step only, so it never enters a layer, the history or the cache key.

3. **Q: Why doesn't `RUN rm` of a copied secret file help?**
   A: Layers are immutable. The file still exists in the COPY layer, and the later layer only adds a whiteout. Anyone can extract the earlier layer.
