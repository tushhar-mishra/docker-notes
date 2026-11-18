# 43 · Image Supply-Chain Security

> **Level:** ★★ intermediate
>
> **You will learn:** how to answer three questions about any image: **What's inside it?** (SBOM), **Does it have known vulnerabilities?** (scanning with Docker Scout and Trivy, how scanners actually work, and how to read and prioritise results), and **Where did it come from, and was it changed?** (provenance attestations and signatures with cosign). You'll also see how to gate CI on these checks and enforce them at deploy time.
>
> **You need:** topics 06–07 (digests, manifests, attestation manifests), topics 17–22 (builds, multi-stage, buildx, registries).

---

## What it is

The **software supply chain** of an image = everything that went into it and everything it passed through: base image, OS packages, language libraries, your code, the build system, the registry. An attacker or a bug at any point ends up in production.

| Tool / artifact | Answers |
|---|---|
| **SBOM** (Software Bill of Materials) | "Exactly which packages and versions are in this image?" |
| **Vulnerability scanning** | "Which of those have known CVEs, and are fixes available?" |
| **Provenance attestation** | "Which source, commit, builder and build steps produced this image?" |
| **Signature** | "Was this image produced/approved by us, and is it unchanged since?" |
| **Digest pinning** | "Is the image we deploy exactly the one we checked?" (topic 06) |

---

## Why it exists

- New CVEs are published **every day** for packages already in your images. An image that was clean last month may have critical issues today.
- Attacks target the chain: malicious packages in public registries (typosquatting npm/PyPI packages), compromised base images, hijacked CI pipelines, and swapped tags in registries.
- Regulations and customers increasingly require SBOMs and signed artifacts (for example the US Executive Order 14028 and the EU Cyber Resilience Act). **SLSA** (Supply-chain Levels for Software Artifacts, pronounced "salsa") is a widely used framework for build integrity.

---

## Analogy

**Food safety:**
- **SBOM** = the **ingredients list** on the package.
- **Vulnerability scan** = checking that list against the **public recall list** ("batch 42 of this flour is contaminated").
- **Provenance** = the **farm-to-shelf record**: which factory, which batch, which machine, which date.
- **Signature** = the **tamper-evident seal** with the manufacturer's stamp.
- **Digest pinning** = buying by **exact batch number**, not just brand name.

---

## How it works inside

### How a scanner works

1. **Inventory (SBOM):** the scanner opens the image's layers and reads package databases:
   - OS packages: `/var/lib/dpkg/status` (Debian/Ubuntu), `/lib/apk/db/installed` (Alpine), the RPM database (RHEL),
   - language packages: `package-lock.json`, `requirements.txt` / installed `site-packages` metadata, `go.sum` / Go binary build info, JARs' `pom.properties`, `Gemfile.lock`, `Cargo.lock`…
2. **Matching:** each package + version is checked against **vulnerability databases**: distro security trackers (Debian, Ubuntu, Alpine, Red Hat), the **NVD** (National Vulnerability Database), **GitHub Security Advisories (GHSA)**, OSV.
3. **Report:** CVE ID, severity (Critical/High/Medium/Low, often from a **CVSS** score = Common Vulnerability Scoring System, 0–10), installed version, **fixed version** (if any).

Consequences:
- Scanners only find what's **in package databases**. A binary you `curl`ed into `/usr/local/bin` without metadata may be invisible. (Distroless and Chainguard images keep metadata for scanners.)
- **Distro severity can differ from NVD**: Debian may rate a CVE "low" because their build isn't affected. Trust the distro's rating for OS packages.
- "Vulnerable package present" ≠ "exploitable". Your app may never call the vulnerable function. **VEX** (Vulnerability Exploitability eXchange) documents let vendors state "not affected", and modern scanners can read them.

### Prioritising findings (real-world triage)

1. **Critical/High with a fix available** in packages you actually use → fix now (update base image/dependency, rebuild).
2. **Known exploited** vulnerabilities (CISA's KEV catalogue = Known Exploited Vulnerabilities) and high **EPSS** scores (Exploit Prediction Scoring System, the probability of exploitation) → top priority even at lower severity.
3. **No fix available** → reduce exposure (minimal base image, drop the package, mitigations), and document/accept the risk.
4. Low/Medium in unused OS packages → fix by moving to slim/distroless bases (topic 19) rather than one by one.

Most findings are fixed by **rebuilding on an updated base image**. That's why automated base updates (topic 20) and regular rebuilds matter more than any single scan.

### SBOM formats

- **SPDX** (Software Package Data Exchange, a Linux Foundation standard),
- **CycloneDX** (a standard from OWASP, the Open Worldwide Application Security Project).

Both are JSON/XML lists of components with names, versions, licences and identifiers (**PURL** = Package URL, like `pkg:deb/debian/openssl@3.0.15-1`).

### Attestations: SBOM and provenance attached to the image

BuildKit can generate and **attach** them to the image in the registry, as extra manifests in the image index (the `unknown/unknown` entries you saw in topics 06 and 21):

- `--sbom=true` → an SPDX SBOM made at build time (more accurate than scanning later, because it sees the build context).
- `--provenance=mode=max` → **SLSA provenance**: source repo, commit, Dockerfile, build arguments, builder, timestamps, and the base images' digests. BuildKit adds a minimal provenance attestation by default when pushing (`mode=min`).

These are **in-toto attestations** (a standard format: a signed "statement" about an artifact identified by its digest).

### Signing with cosign (Sigstore)

**Sigstore** is a free, open-source signing system (Linux Foundation). Its CLI **cosign** signs **image digests** and stores the signature in the registry next to the image (as another OCI artifact).

Two modes:
- **Key-based:** `cosign generate-key-pair`, sign with the private key, verify with the public key.
- **Keyless:** cosign gets a short-lived certificate from **Fulcio** (Sigstore's certificate authority) tied to an **OIDC identity** (OpenID Connect: e.g. "the GitHub Actions workflow `acme/api/.github/workflows/release.yml` on `refs/heads/main`"). The signature is recorded in **Rekor**, a public transparency log. Verification checks **who** signed (identity + issuer), with no keys to manage. This is the common CI approach.

You verify by **identity**, e.g. "only accept images signed by our release workflow".

> **Docker Content Trust** (`DOCKER_CONTENT_TRUST=1`, based on Notary v1) is the old Docker signing system. It's **deprecated/legacy**. Use cosign (or Notation from the Notary Project v2, common with Azure).

### Enforcement: verify before running

Signing is only useful if something **checks** signatures:
- in CI/CD deploy steps (`cosign verify` before deploying),
- in Kubernetes with **admission controllers**: Kyverno, Sigstore policy-controller, Gatekeeper/Ratify (they reject Pods whose images aren't signed by an allowed identity, or have critical CVEs).

---

## Commands / files

### Docker Scout

```bash
docker scout quickview python:3.13-slim
```

```
    i New version 1.x available ...
  Target             │  python:3.13-slim  │    0C     0H     2M    23L
    digest           │  4ec4a6c8c6a2      │
  Base image         │  debian:trixie-slim │    0C     0H     2M    23L
  Refreshed base image │  debian:trixie-slim │    0C     0H     1M    23L
                     │                    │                  -1
```

- Shows the image's findings and its **base image**, and whether a **refreshed base** (same tag, rebuilt) would reduce them.

```bash
docker scout cves --only-severity critical,high --only-fixed myapp:1.0
```

```
    ✗ HIGH CVE-2025-XXXXX
      https://scout.docker.com/v/CVE-2025-XXXXX
      Affected range : <3.0.16-1~deb12u1
      Fixed version  : 3.0.16-1~deb12u1
      CVSS Score     : 7.5
      ...
  pkg:deb/debian/openssl@3.0.15-1~deb12u1?os_distro=bookworm&os_name=debian&os_version=12
```

- `--only-fixed` = only show CVEs that have a fix (the actionable ones).
- The `pkg:` line is the **PURL** of the vulnerable package.

```bash
docker scout recommendations myapp:1.0
```

- Suggests base image updates/alternatives that reduce vulnerabilities.

### Trivy

```bash
# Pin the scanner itself: look up the current release on github.com/aquasecurity/trivy/releases
TRIVY_IMAGE=aquasec/trivy:<version>          # e.g. a specific 0.x.y tag, ideally also @sha256:<digest>
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock "$TRIVY_IMAGE" image --severity HIGH,CRITICAL --ignore-unfixed python:3.13
```

(Or install the `trivy` binary from Aqua's package repository. Mounting the socket here is OK on your lab machine. In shared CI, scan a pushed image or a tarball instead.)

> **Why pin the scanner?** Security tools are attractive supply-chain targets: they run in CI with access to images, tokens and sometimes the Docker socket. In March 2025 the popular GitHub Action `tj-actions/changed-files` was compromised (CVE-2025-30066): attackers moved its version **tags** to malicious code that dumped CI secrets into logs. Pipelines that had pinned the action to a commit SHA were unaffected. Apply the same rule as for your own images (topics 20, 22): **pin by version and digest/SHA, and update through reviewed pull requests**.

```
python:3.13 (debian 13.1)
==========================
Total: 5 (HIGH: 4, CRITICAL: 1)

┌──────────────┬────────────────┬──────────┬────────┬──────────────────┬──────────────────┬───────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │ Installed Version│  Fixed Version   │               Title               │
├──────────────┼────────────────┼──────────┼────────┼──────────────────┼──────────────────┼───────────────────────────────────┤
│ libexpat1    │ CVE-2025-XXXXX │ CRITICAL │ fixed  │ 2.7.1-1          │ 2.7.1-2          │ expat: ...                        │
...

Python (python-pkg)
===================
Total: 1 (HIGH: 1, CRITICAL: 0)
...
```

- Two sections: OS packages and language packages.
- `--ignore-unfixed` = hide CVEs with no fix. `--exit-code 1` = fail CI if anything matching is found.
- Trivy also scans filesystems (`trivy fs .`), git repos, IaC files and Kubernetes clusters.

Generate an SBOM with Trivy or Syft:

```bash
trivy image --format cyclonedx --output sbom.cdx.json myapp:1.0
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock anchore/syft myapp:1.0 -o spdx-json > sbom.spdx.json
```

### Build with SBOM and provenance attached

```bash
docker buildx build --sbom=true --provenance=mode=max \
  -t ghcr.io/<you>/hello-go:1.1 --push .
docker buildx imagetools inspect ghcr.io/<you>/hello-go:1.1 --format '{{json .SBOM}}' | head -c 400; echo
docker buildx imagetools inspect ghcr.io/<you>/hello-go:1.1 --format '{{json .Provenance}}' | python3 -m json.tool | grep -E '"(configSource|entryPoint|uri|digest)"' | head
```

```
{"SPDX":{"SPDXID":"SPDXRef-DOCUMENT","creationInfo":{"created":"2026-11-17T10:30:00Z","creators":["Organization: Anchore, Inc","Tool: syft-..."] ...
            "entryPoint": "Dockerfile",
            "uri": "https://github.com/acme/hello-go.git",
            "digest": { "sha1": "3f9a2c1e..." }
```

- The SBOM was generated by BuildKit's scanner (based on Syft) during the build.
- Provenance shows the source repo, the commit and the Dockerfile used.

### Sign and verify with cosign

**Key-based (simple to try locally):**

```bash
cosign generate-key-pair                                   # creates cosign.key (private) and cosign.pub
DIGEST=$(docker buildx imagetools inspect ghcr.io/<you>/hello-go:1.1 --format '{{json .Manifest.Digest}}' | tr -d '"')
cosign sign --key cosign.key ghcr.io/<you>/hello-go@$DIGEST
cosign verify --key cosign.pub ghcr.io/<you>/hello-go@$DIGEST | python3 -m json.tool | head -12
```

```
Verification for ghcr.io/<you>/hello-go@sha256:9f1e… --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
[
    {
        "critical": {
            "identity": { "docker-reference": "ghcr.io/<you>/hello-go" },
            "image": { "docker-manifest-digest": "sha256:9f1e…" },
            "type": "cosign container image signature"
        },
```

- **Always sign the digest**, not a tag (a tag can move).

**Keyless in CI (GitHub Actions):** with `permissions: id-token: write`, simply run `cosign sign --yes ghcr.io/acme/api@$DIGEST`. Verify by identity:

```bash
cosign verify ghcr.io/acme/api@sha256:… \
  --certificate-identity-regexp '^https://github.com/acme/api/.github/workflows/release.yml@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

### Gate CI on findings

```bash
trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed myapp:${GIT_SHA}
docker scout cves --exit-code --only-severity critical --only-fixed myapp:${GIT_SHA}
```

- A non-zero exit fails the pipeline step (topic 46). Start with **critical + fixable** only, so the gate is actionable instead of permanently red.
- Keep an **ignore file** (`.trivyignore`, Scout VEX statements) with expiry dates and reasons for accepted risks.

---

## Hands-on lab

1. Compare scans of `python:3.13`, `python:3.13-slim` and `gcr.io/distroless/python3-debian12` with Scout or Trivy. Record totals by severity.
2. Scan your `hello-flask` image. Find whether the findings come from the base image or your Python packages. Fix one by updating a dependency or the base, rebuild, rescan.
3. Generate an SBOM (`trivy image --format cyclonedx` or Syft) and find the exact version of `openssl` or `flask` in it.
4. Push an image with `--sbom=true --provenance=mode=max` to your local registry (`localhost:5000`) using a `docker-container` builder (topic 21) and inspect the attestations with `imagetools inspect`. (A registry on `localhost` works. If the builder container can't reach it, use `--network host` when creating the builder: `docker buildx create --driver-opt network=host ...`.)
5. Sign the pushed image's digest with a cosign key pair and verify it. Then push a new image to the same tag and try verifying the **tag**: it fails for the new, unsigned digest.

---

## Where you'll use it

- **CI pipelines:** build → SBOM + provenance → scan with a gate → sign → push (topic 46).
- **Deploy-time enforcement:** Kubernetes admission policies that require signatures from your CI identity and block critical CVEs.
- **Incident response:** "Are we affected by the new OpenSSL CVE?" → query stored SBOMs for every running image digest, in minutes instead of days.
- **Audits and customers:** providing SBOMs and signed artifacts on request.
- **Base image strategy:** measurable CVE reduction from minimal and hardened bases (topic 19).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| Scanner shows hundreds of CVEs | Full base image, old base, many unused packages | Slim/distroless/hardened base; rebuild regularly |
| Same CVE count after "updating" | Base tag cached locally; image not rebuilt | `docker build --pull`; check the base digest changed |
| CI gate always red, people ignore it | Gate on all severities, including unfixable | Gate on critical/high **with fixes**; documented ignores with expiry |
| Scanner misses a known-vulnerable binary | Installed without package metadata (curl into `/usr/local/bin`) | Install via package managers, or generate an SBOM at build time |
| `cosign verify`: `no signatures found` | You signed a different digest (tag moved) or never signed | Sign and verify **digests** |
| `cosign verify`: `none of the expected identities matched` | Signed by a different workflow/branch/issuer | Fix the identity regexp/issuer, or sign from the right workflow |
| Attestations missing in the registry | Built with the `docker` driver/old store, or `--load` instead of `--push` | Use a `docker-container` builder or the containerd store, and `--push` |
| Registry/tools show odd `unknown/unknown` platform entries | Attestation manifests in the index | Normal. Tools that don't understand them can ignore them |

---

## Quick recap

- **SBOM** (SPDX/CycloneDX) = what's inside. **Scanning** (Scout, Trivy) = SBOM matched against vulnerability databases. Results are only as good as the package metadata.
- Triage: critical/high **with fixes** and known-exploited first. Most fixes come from rebuilding on an updated, minimal base.
- BuildKit can attach **SBOM and SLSA provenance** attestations (`--sbom`, `--provenance=mode=max`) to images.
- **cosign** signs image **digests**. Keyless signing ties signatures to a CI identity via OIDC/Fulcio/Rekor. Docker Content Trust is legacy.
- Signatures matter only when **verified**: in deploy steps and Kubernetes admission policies.

**Interview questions**

1. **Q: How do you keep container images free of known vulnerabilities?**
   A: Minimal base images, pinned and automatically updated (Renovate/Dependabot), and regular rebuilds. Scan in CI (Scout or Trivy) with a gate on critical/high fixable issues, and keep re-scanning images that are already deployed, since new CVEs appear daily. Then prioritise by fix availability and exploitability.

2. **Q: What is an SBOM and why does it matter?**
   A: A Software Bill of Materials lists every component and version in an artifact (in SPDX or CycloneDX format). It lets you answer "are we affected?" quickly when a new CVE appears, and it's increasingly required for compliance.

3. **Q: How do you ensure only trusted images run in production?**
   A: Build in CI, attach provenance, and sign image digests with cosign (keyless, tied to the CI identity). Deploy by digest, and enforce signature verification (plus vulnerability policies) at deploy time, e.g. with Kubernetes admission controllers like Kyverno or the Sigstore policy-controller.
