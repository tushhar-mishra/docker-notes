# 54a · Senior-Level Scenario Interviews

> **Level:** ★★★ advanced. Do this after topic 54.
>
> **You will learn:** how senior DevOps/SRE/Platform interviews test containers: through **open-ended scenarios** instead of definitions. You get 18 realistic problems (8 production incidents, 5 system designs, 5 judgement calls) plus 2 hands-on exercises. Each has what the interviewer is really testing, the clarifying questions to ask, a model answer, follow-ups with answers, and the **red flags** that cost candidates the offer.
>
> **You need:** everything, especially topics 12, 28–33, 39a, 40–47, 50–53.

---

## What it is

Senior interviews ask "**what would you do**", not "what is". There's rarely one right answer. They're scoring:
- **Method**: do you narrow problems systematically, or guess?
- **Depth**: can you go from symptom to kernel mechanism (cgroups, conntrack, signals) when needed?
- **Judgement**: do you weigh trade-offs (cost, risk, team skills, time), and say what you would *not* do?
- **Operational maturity**: do you think about blast radius, rollback, observability, security and people (on-call, post-mortems)?
- **Communication**: clear structure, explicit assumptions, and checking in with the interviewer.

## Why it exists

Junior candidates recite. Senior candidates **reason under uncertainty** and own outcomes. A senior engineer who knows every flag but restarts things blindly, or designs a platform without thinking about rollback, won't pass. These problems train the shape of a senior answer.

## Analogy

A junior pilot is tested on the controls. A senior pilot (captain) is tested in the **simulator with an engine fire at take-off**: does she follow the procedure, communicate with the crew, choose the safest option, and debrief afterwards?

---

## How it works inside: the answer frameworks

### For incidents: **S.C.O.P.E.**

1. **S**tabilise first: mitigate user impact (roll back, shift traffic, scale out, raise a limit) before the perfect diagnosis. Say it explicitly.
2. **C**larify: what changed (deploy, config, traffic, infra)? When did it start? Who/what is affected (all users, one region, one host)?
3. **O**bserve: data before hypotheses. Name the exact signals: metrics, logs, events, `docker inspect`, cgroup files, packet captures.
4. **P**rove: one hypothesis at a time, each tested with a specific command. Explain what result would confirm or reject it.
5. **E**liminate and prevent: root cause fix, then prevention (alert, test, default config, runbook), then a blameless post-mortem.

### For design: **R.A.D.I.O.S.**

1. **R**equirements: functional + non-functional (scale, availability target/SLO, security/compliance, team size, budget). Ask; then state assumptions.
2. **A**rchitecture: draw the main components and data flow.
3. **D**eep-dive: pick the 2–3 hardest parts and go deep (that's where seniority shows).
4. **I**ssues: failure modes, scaling limits, security threats.
5. **O**perations: deploy, rollback, observability, on-call, upgrades, cost.
6. **S**ummary: trade-offs you made, and what you'd do in phase 2.

### For judgement questions

"**It depends**" is only acceptable if you say **on what**, give a **default**, and name **when you'd choose differently**.

---

## Commands / files: the problems

## Part A: Production incidents

### A1. "Random 502s, a few per minute, no deploys happening"

**Context given:** Containers behind a load balancer (an AWS ALB (Application Load Balancer, AWS's managed HTTP load balancer), or an nginx proxy with upstream keepalive). The app is Node.js. About 0.1% of requests return 502. Backends look healthy and CPU is low.

**Testing:** do you know the **keepalive timeout race**, and can you prove it?

**Clarify:** Are 502s tied to time-after-idle or to load? Are they on the proxy's logs with "upstream prematurely closed connection" / "connection reset by peer"? What are the proxy's idle timeout and the app server's keepalive timeout?

**Model answer:**
- 502 = the proxy got no valid response from the backend (topic 39a). With no deploys and healthy backends, a classic cause is connection reuse: the proxy keeps idle connections to the backend open for **60 s** (ALB default idle timeout), but Node's HTTP server closes idle keep-alive connections after **5 s** (`server.keepAliveTimeout` default). If the proxy sends a new request on a connection at the exact moment the backend closes it, the request fails → 502.
- **Prove it:** proxy logs show `upstream prematurely closed connection` / reset; 502s correlate with low-traffic periods (more idle connections); `tcpdump` on the backend's veth (topic 30) shows the backend sending FIN, and the proxy sending a request on that connection right after.
- **Fix:** backend keepalive timeout must be **longer** than the proxy's idle timeout (e.g. Node `keepAliveTimeout = 65s`, `headersTimeout` slightly higher), or lower the proxy's upstream keepalive timeout. Optionally enable retries for idempotent requests at the proxy.
- **Prevent:** a platform default per runtime, a dashboard for 5xx by upstream, and a test.

**Follow-ups:**
- *"Same symptom, but only during deploys?"* → Not the keepalive race. It's shutdown/readiness: the container is removed from the load balancer **after** it already stopped. Fix: fail readiness first, delay before shutdown (`preStop` sleep in Kubernetes), drain in-flight requests, exec-form PID 1 (topics 12, 45).
- *"Why not just retry everything?"* → Retrying non-idempotent requests (POST payments) can duplicate side effects. Retry only idempotent requests, or use idempotency keys.

**Red flags:** "Restart the containers." "Increase the timeout everywhere" without understanding which side must be longer.

---

### A2. "p99 latency doubled after we added CPU limits / moved to Kubernetes. Average CPU is 30%."

**Testing:** CFS throttling, and runtimes that ignore container limits.

**Clarify:** Which runtime (Go, Java, Node)? What are the CPU limit and the number of threads? Did the image or runtime version change at the same time?

**Model answer:**
- Average CPU hides **bursts**. With `cpu.max` = 1 CPU (100 ms per 100 ms period), a multi-threaded app with 8 busy threads uses the whole quota in ~12 ms and is then **frozen for ~88 ms** (topic 31). Requests landing in that window get +88 ms.
- **Prove it:** `cpu.stat` → `nr_throttled/nr_periods` high, `throttled_usec` rising (or `container_cpu_cfs_throttled_periods_total` in Prometheus). Correlate with p99 on the same graph.
- Check thread sizing: Go **before 1.25** sets `GOMAXPROCS` = host cores, not the limit; JVM GC/JIT threads; Node's libuv pool; gunicorn workers sized from `nproc` (which shows host CPUs).
- **Fixes, in order of preference:** size threads to the limit (`GOMAXPROCS`, Go 1.25+, `automaxprocs`, JVM `-XX:ActiveProcessorCount`), raise the limit, or remove CPU limits and rely on requests/weights, with good node capacity management (many teams do this for latency-sensitive services).
- **Prevent:** an alert on throttling ratio > 25%, and a load test with production limits.

**Follow-ups:**
- *"Would you remove CPU limits everywhere?"* → Not blindly. Without limits, a runaway pod can starve neighbours on the node. Keep requests accurate, keep memory limits always, use CPU limits for batch/noisy workloads, and monitor. Have a reasoned policy, not a dogma.
- *"Why doesn't memory behave the same way?"* → Memory can't be throttled like CPU. Over the limit means reclaim, then OOM kill.

**Red flags:** "CPU is 30%, so it's not CPU." Not knowing what throttling is.

---

### A3. "Every few hours, some requests take exactly 5 seconds longer. Mostly in Alpine-based services."

**Testing:** DNS internals: conntrack races, resolver timeouts, musl vs glibc, `ndots`.

**Clarify:** Is it exactly 5 s (the resolver's default timeout)? Only outbound calls by hostname? Kubernetes or plain Docker? Which images?

**Model answer:**
- An exact 5 s delay = a **DNS query timed out and was retried** (`options timeout:5` default).
- Known cause: resolvers send **A and AAAA queries in parallel from the same UDP socket**. Under load, a race in the kernel's **conntrack** insert for the two packets (same 5-tuple) can drop one of them. That query times out after 5 s. musl (Alpine) also queries all nameservers in parallel and doesn't support glibc's workaround options.
- Amplifier: Kubernetes `ndots:5` → each external lookup becomes several queries (search domains first), multiplying the chances (topic 29).
- **Prove it:** `conntrack -S` → `insert_failed` counter rising; `tcpdump -i any port 53` shows a query with no answer and a retry 5 s later; latency histogram spikes at +5 s.
- **Fixes:** glibc images: `options single-request-reopen` (or `use-vc` for TCP) via `--dns-option`/`dnsConfig`; in Kubernetes, **NodeLocal DNSCache** (a per-node cache, which avoids the conntrack path); reduce `ndots`/use FQDNs with a trailing dot; app-level DNS caching; move hot services off Alpine if needed; newer kernels reduced the race.
- **Prevent:** DNS latency SLI (Service Level Indicator, the metric an SLO is measured on) and alerts on `insert_failed`.

**Follow-ups:** *"How would you confirm it's not the DNS server itself?"* → Query the server directly from the host with `dig` in a loop; check server-side metrics; compare from a glibc image with `single-request-reopen`.

**Red flags:** "DNS is flaky, add retries." Not linking "exactly 5 s" to a resolver timeout.

---

### A4. "A Java service is OOMKilled every ~6 hours. Heap dashboards show 60% heap usage."

**Testing:** container memory accounting versus the runtime's view, and native memory.

**Clarify:** Container limit vs `-Xmx`/`MaxRAMPercentage`? Exit 137 with `OOMKilled: true`? JDK version? Does the app use NIO/Netty, many threads, or JNI?

**Model answer:**
- The cgroup counts **everything** (topic 31): heap + metaspace + code cache + thread stacks (1 MB each × threads) + direct/NIO buffers + glibc malloc arenas + GC structures + page cache from tmpfs/shm. The heap is just one part.
- **Prove it:** `memory.stat` (anon growing, file stable); `jcmd <pid> VM.native_memory summary` with `-XX:NativeMemoryTracking=summary` to see non-heap areas; thread count over time; RSS vs heap gap growing. A known culprit: **glibc malloc arenas** in multi-threaded apps (many 64 MB arenas) → `MALLOC_ARENA_MAX=2`, or jemalloc.
- **Fix:** leave 25–30% headroom (`MaxRAMPercentage=70`), cap direct memory (`-XX:MaxDirectMemorySize`), fix thread/buffer leaks, set `MALLOC_ARENA_MAX`, or raise the limit based on the measured steady state.
- **Prevent:** alert on working set / limit > 90% (topic 47); dashboards for RSS and heap together.

**Follow-ups:** *"Why every ~6 hours?"* → A slow native leak, or growth to a steady state just above the limit. The trend line tells you which. *"How do you capture evidence before the kill?"* → Lower thresholds for a heap dump on a canary, take periodic NMT diffs, and use `memory.events` / `oom_kill`.

**Red flags:** "Increase the memory" with no measurement. Thinking `-Xmx` = the container's memory usage.

---

### A5. "After a deploy, a whole host became unresponsive. SSH barely works."

**Testing:** noisy-neighbour isolation, host resource exhaustion, blast radius.

**Clarify:** What did the deploy change? Do containers have memory/PID limits? Is it CPU, memory, disk I/O, PIDs or logs? Is dockerd responsive?

**Model answer:**
- **Stabilise:** roll back or stop the new containers (`docker update --restart no` first so a restart policy doesn't fight you), from the console if SSH is too slow.
- **Observe:** `top`/`vmstat 1` (run queue, swap, iowait), `free -m`, `dmesg` (global OOM? conntrack table full?), `df -h`/`df -i` (logs filling the disk), `ps -eLf | wc -l` (thread/PID exhaustion), `journalctl -u docker`.
- Typical causes: no memory limits → **host-level** OOM and swapping (thrashing); a fork/thread explosion without `pids.max`; a log flood (with a blocking driver, apps stall; disk fills, journald stops); a CPU spin without limits.
- **Fix:** limits on every container (memory always, PIDs always, CPU by policy), log rotation, the Docker data dir on a separate disk, and reserved host resources (systemd slices: give `system.slice` and `user.slice` CPU/memory weight so SSH stays usable).
- **Prevent:** admission/CI check that every service defines limits (Compose lint / OPA), canary deploys (one host first), alerts.

**Red flags:** Rebooting immediately (losing evidence) without containment. Not thinking about blast radius.

---

### A6. "Image works in staging, fails in production with `exec format error` / `no such file or directory`."

**Testing:** architecture and libc reasoning, and promotion discipline.

**Model answer:**
- `exec format error` → wrong CPU architecture: production is arm64 (Graviton) or amd64 and the image isn't. Check `docker image inspect -f '{{.Architecture}}'` vs `uname -m`. Fix: multi-platform builds (topic 21).
- `no such file or directory` for a binary that exists → dynamic loader missing (glibc binary on Alpine/scratch), or a script with CRLF line endings / a bad shebang (topics 15, 17, 19).
- Process smell: **if staging and production run different bytes, promotion is broken.** Build once, promote the **same digest** (topic 22). Ask: was the production image rebuilt? Was it pulled by tag that moved?

**Red flags:** "Rebuild it on the production server."

---

### A7. "Our cloud bill shows huge outbound traffic from one host, and the CPU is pegged."

**Testing:** incident response for a compromise (topic 44a).

**Model answer (short form):** Treat it as a security incident. Identify the container (per-container network/CPU metrics, `docker stats`, `nsenter ss -tanp` for remote IPs). **Contain:** disconnect its networks, block egress at the security group, pause it, stop restarts, and serve traffic from fresh containers on a clean host. **Preserve:** inspect, logs, diff, `/proc/PID/exe`, sockets, export/commit/save, hashes, off-host. **Investigate:** entry point (vulnerable dependency? exposed Docker API 2375? leaked credentials?), lateral movement, host escape (privileged/socket/host mounts → rebuild the host). **Eradicate:** patch, **rotate every secret reachable** (including instance role credentials via the metadata endpoint), rebuild from CI. **Recover**, then a post-mortem with new detections (Falco, egress alerts) and prevention (read-only root, egress allowlists, IMDSv2).

**Red flags:** "Delete the container and redeploy" (no evidence, no rotation, the attacker comes back). Forgetting credentials.

---

### A8. "Containers on one host can't reach the database VPC since this morning. Other hosts are fine."

**Testing:** Docker network internals on the host.

**Model answer:** Check whether a **new Docker network** was created whose subnet overlaps the database VPC range (`ip route get <db-ip>` shows `dev br-xxxx`) (topic 27). Other common host-specific causes: Docker's iptables rules flushed by a firewall reload (`iptables -t nat -S | grep -c DOCKER`), MTU after a VPN/overlay change (small queries work, large results hang), or DNS upstream changes. Fix the overlap by removing/recreating the network and setting `default-address-pools` fleet-wide via config management. **Prevent:** the address plan documented, and a CI check for networks.

---

## Part B: System design

### B1. "Design the container build and release platform for a company with 50 microservices and 15 teams."

**Clarify:** Cloud/on-prem? Kubernetes already? Compliance (SOC 2, PCI)? Languages? Release frequency? Multi-arch needed?

**Strong answer outline:**
- **Golden paths:** a small set of approved base images per language (slim/distroless/hardened), rebuilt nightly by a platform pipeline, with `docker init`-style templates and a reusable CI workflow so teams don't reinvent Dockerfiles (topics 19–20).
- **CI** (reusable workflow): lint (`--check`, hadolint) → build with BuildKit and a shared registry cache (or remote builders) → unit tests in a test stage → integration tests with Compose `--wait` → scan gate (critical/high fixable) → push with SHA + semver tags → SBOM + SLSA provenance → keyless cosign signing via OIDC (topics 43, 46).
- **Registry:** one private registry (ECR/Artifact Registry/Harbor) with immutable tags, pull-through cache for Docker Hub, retention policies (keep releases, expire branch builds after N days), replication per region.
- **Promotion:** the same digest dev → staging → prod, recorded in git (GitOps). Production admission verifies signature + provenance + no critical CVEs (Kyverno/policy-controller).
- **Dependency hygiene:** Renovate for base digests and libraries, with auto-merge on green for patch updates.
- **Builders:** ephemeral VM runners or rootless BuildKit. No privileged DinD on shared runners. Separate deploy credentials from build runners.
- **Observability of the platform:** build duration, cache hit rate, scan findings trend, mean time to patch critical CVEs, deploy frequency, change failure rate (DORA metrics = the four DevOps Research and Assessment delivery metrics: deploy frequency, lead time, change failure rate, time to restore).
- **Trade-offs:** central control vs team autonomy (templates are opt-out with review), strict gates vs velocity (start with critical-only gates, tighten over time), registry cost vs retention.

**Follow-ups:**
- *"A critical OpenSSL CVE drops on Friday. How fast can you patch all 50 services, and how?"* → Query SBOMs for affected digests running in prod → the base image pipeline rebuilds the bases → Renovate PRs (or forced rebuilds) for all services → automated tests → progressive rollout → track by dashboard until 100%. Target hours, not weeks.
- *"How do you stop a team from bypassing the pipeline?"* → Admission policy only admits images signed by the CI identity; registry write access only for CI.

---

### B2. "We have 12 VMs running Docker Compose. Deploys cause downtime and one host died last month. What do you propose?"

**Strong answer:**
- Clarify the SLO, team size and skills, workload types (stateless vs databases), and budget.
- **Short term (weeks):** load balancer in front of ≥2 hosts per service; health checks; deploy one host at a time (drain from the LB → `compose pull && up -d --wait` → health → re-add); `live-restore`; limits/log rotation; move databases to a managed service or a properly replicated setup; backups with restore tests (topic 39).
- **Medium term:** adopt an orchestrator if the requirements justify it: managed Kubernetes (EKS/GKE/AKS) for self-healing, rolling updates, autoscaling and an ecosystem; or **ECS/Fargate / Cloud Run** for less operational load; or **Nomad** if the team wants simplicity. Explain the choice against team skills and the cost of running Kubernetes.
- **Migration plan:** stateless services first, using the concept map (topic 53); dual-run behind the LB; observability first; databases last (or never: keep them managed).
- **Red flags:** "Just move everything to Kubernetes" with no reasoning; ignoring the stateful parts; no rollback plan.

---

### B3. "Design CI runners that safely build and test code from external contributors."

**Strong answer:** Untrusted code = assume malicious. Use **ephemeral VM per job** (Firecracker microVMs, cloud VMs, or GitHub-hosted runners), or at least gVisor/Kata sandboxes. **No** privileged DinD, **no** host Docker socket, **no** secrets for fork PRs (only after maintainer approval), egress restrictions (block the metadata endpoint, allowlisted registries), resource limits and timeouts, rootless BuildKit for image builds, and a cache that's **read-only** for untrusted jobs (to avoid cache poisoning). Separate trusted pipelines (main branch, signing, deploy) on different runners with OIDC-scoped credentials. Monitor runners with Falco-type rules. (Topics 40, 42, 46, 44a.)

**Follow-up:** *"How do you keep builds fast if every job gets a fresh VM?"* → Pre-baked runner images with warm base layers, a remote BuildKit cache (registry-backed; read-only for untrusted jobs), dependency proxies.

---

### B4. "Design logging and monitoring for 600 containers on 40 Docker hosts."

**Strong answer:** Per host: cAdvisor + node_exporter + dockerd metrics + a log shipper (Fluent Bit/Vector/Alloy) reading rotated local logs. Central: Prometheus (or a scalable backend like Mimir/Thanos/VictoriaMetrics, since 600 containers × hundreds of series adds up), Loki/Elasticsearch for logs, Alertmanager routing to on-call, Grafana dashboards. Labels from Compose/Docker (project, service, team, env) for ownership routing. **Control cardinality** (drop unused cAdvisor metrics; never put user IDs or request IDs in metric labels). Alerting: SLO-based pages (error rate, latency), cause-based warnings (OOM events, throttling, restart loops, disk/inodes, conntrack, certificate expiry). Retention tiers (hot 15 days, cold longer), sampling for debug logs, PII filtering in the shipper. Meta-monitoring: alert if a host stops reporting. (Topics 33, 47.)

---

### B5. "Reduce the critical/high CVEs across our images by 80% in one quarter."

**Strong answer:** Measure first (scan everything in the registry and running in prod; group findings by **base image** vs **app dependency** vs **unfixable**). Most findings come from fat base images: move to slim/distroless/hardened bases via golden images + Renovate. Multi-stage builds to drop build tools. Automated weekly rebuilds even without code changes. Gates in CI for **new** criticals with fixes (don't block on the existing backlog on day one). VEX/ignore lists with expiry for not-affected cases. A dashboard by team with targets, and a "patch SLA" (critical within 7 days). Report progress weekly. (Topics 19, 20, 43.)

---

## Part C: Judgement calls

### C1. "Alpine or distroless as the company default base?"
**Answer:** Default to **distroless** (or slim/hardened glibc images) for runtime images: glibc compatibility, minimal attack surface, no shell. Alpine is fine for static binaries and small tools, but musl differences (DNS behaviour, malloc performance, Python wheels) cause real incidents. Pair distroless with a debugging story (`docker debug`, sidecars, `kubectl debug`) so "no shell" isn't a blocker. (Topics 19, 51.)

### C2. "Should we set CPU limits?"
**Answer:** Always set CPU **requests/weights** and memory limits. CPU **limits** cause throttling and latency (A2). For latency-sensitive services on well-managed capacity, many teams don't set them. For batch, multi-tenant or noisy workloads, they do. Monitor throttling either way, and have a written policy with the reasoning.

### C3. "Should we run our PostgreSQL in containers?"
**Answer:** Containers aren't the problem. **Operations** are: backups and restore tests, replication/failover, upgrades, storage performance and data durability. On a single host with Compose: possible for small or non-critical systems, with named volumes, logical backups shipped off-host and tested restores. For critical production: prefer a **managed database** (RDS/Cloud SQL), or Kubernetes with a mature operator (CloudNativePG) and a team that knows it. Decide by the team's ability to run it at 3 AM.

### C4. "We're a 5-person startup. Compose or Kubernetes?"
**Answer:** Start with **managed containers** (Cloud Run, ECS Fargate, Fly/Render) or Compose on 2+ VMs behind a load balancer, with good CI and backups. Kubernetes brings real power and real operational cost. Adopt it when the requirements (many services, autoscaling, multi-team, HA) and skills justify it. Make the move easier later by building OCI images properly, with 12-factor config, health endpoints, and IaC from day one.

### C5. "A team wants to mount the Docker socket into their monitoring/CI container. Approve?"
**Answer:** Not as-is: socket = root on the host (topic 40). Ask what they actually need. Read-only container metadata → a **socket proxy** that allows only specific GET endpoints. Building images → rootless BuildKit or a remote builder. Metrics → cAdvisor/the daemon's metrics endpoint. If there's truly no alternative: a dedicated host, rootless Docker, auditing, and a time-boxed exception with an owner. The senior skill shown here: saying "no, but here's how" instead of a flat no.

---

## Part D: Hands-on exercises (live or take-home)

### D1. "Review this Dockerfile" (find at least 10 issues)

```dockerfile
FROM node:latest
MAINTAINER dev@company.com
ENV NPM_TOKEN=npm_abc123
COPY . /app
WORKDIR /app
RUN npm install
RUN apt-get update && apt-get install -y curl vim
EXPOSE 80
HEALTHCHECK CMD curl localhost
CMD npm start
```

**Expected findings:**
1. `node:latest`: unpinned and mutable → pin the version (+ digest), slim base.
2. `MAINTAINER` is deprecated → `LABEL org.opencontainers.image.authors`.
3. **Secret in `ENV`** → it's in the image config forever. Rotate the token now; use `RUN --mount=type=secret` (topic 18).
4. `COPY . /app` before installing dependencies → cache-busting on every change. Copy `package*.json` first (topic 16).
5. No `.dockerignore` mentioned → `node_modules`, `.git`, `.env` get copied in.
6. `npm install` instead of `npm ci` (not reproducible), and dev dependencies are shipped → multi-stage + `npm ci --omit=dev` (topic 17).
7. `apt-get install` without `--no-install-recommends`, no cleanup of lists in the same `RUN`, and `vim` in production.
8. Runs as **root** → `USER node` (or a numeric UID).
9. Port 80 → use 8080 for portability as non-root (topic 20).
10. `HEALTHCHECK CMD curl localhost` → curl output isn't checked (`-f` missing), there's no interval/timeout/start-period, and it depends on curl being installed.
11. Shell-form `CMD npm start` → PID 1 is `sh`, and `npm` as a process manager doesn't forward SIGTERM well → `CMD ["node", "server.js"]` (topic 12).
12. No multi-stage, no labels for source/revision, no `# syntax=` line or `--check` in CI.

### D2. "This Compose stack doesn't work. Fix it." (narrate your method)

```yaml
version: "3.8"
services:
  api:
    image: myorg/api:latest
    ports: ["8080:8080"]
    environment:
      DB_URL: postgres://app:${DB_PASSWORD}@localhost:5432/app
    depends_on: [db]
  db:
    image: postgres
    ports: ["5432:5432"]
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

**Expected findings:** obsolete `version`; `latest` tags; **`localhost` inside the api container refers to the api container itself**, so it must be `db` (topic 29); `depends_on` without a health condition → a startup race (topic 37); Postgres published on `0.0.0.0:5432` → exposed to the internet, bypassing ufw (topic 28); no named volume → data lost on recreate, plus anonymous volumes from `VOLUME` (topic 24); password in env rather than Compose secrets with `POSTGRES_PASSWORD_FILE` (topic 36/44); no limits, restart policies, log rotation or health checks (topic 39); `DB_PASSWORD` may be unset → use `${DB_PASSWORD:?required}`.

---

## Hands-on lab

1. **Mock interview:** have someone read you A1–A8 one at a time. Answer out loud using S.C.O.P.E., with a 10-minute timer each. Record yourself and check: did you **stabilise first**, name **exact commands**, state **how to confirm**, and finish with **prevention**?
2. **Design drill:** whiteboard B1 and B4 in 35 minutes each with R.A.D.I.O.S. Draw the diagram first. List three failure modes and three trade-offs.
3. **Reproduce two incidents** in your lab: A2 (topic 52, scenario 13) and A5 (a container without limits doing `tail /dev/zero` on a small VM, watching the host struggle; use a disposable VM!).
4. **Do D1 and D2** in writing, then compare with the expected findings.
5. Write **two personal stories** from your own work or labs in STAR format (Situation, Task, Action, Result), e.g. "how I diagnosed an OOM loop", "how I reduced image CVEs".

---

## Where you'll use it

- **Senior DevOps/SRE/Platform interviews:** incident and design rounds.
- **Real on-call and architecture reviews:** the same frameworks work at work.
- **Mentoring:** these problems make good training exercises for juniors.

---

## When it breaks (common senior-interview failures)

| Failure | Why it hurts | Instead |
|---|---|---|
| Jumping to a fix without clarifying | Looks like guessing | Ask 2–3 sharp questions, state assumptions |
| No mitigation step | User impact continues while you investigate | "First I'd roll back / shift traffic" |
| Naming tools, not signals | Shallow | "I'd check `nr_throttled` in `cpu.stat`", not "I'd check monitoring" |
| Designs with no failure modes | Unrealistic | Always cover "what if X fails", rollback and on-call |
| Dogma ("never use X") | Poor judgement | Default + conditions to deviate |
| Forgetting security and secrets in incidents | Repeat compromise | Rotate credentials, assume breach scope |
| Monologue without check-ins | Communication score drops | "Does that level of detail work, or should I go deeper on X?" |

---

## Quick recap

- Senior interviews score method, depth, judgement, operational maturity and communication.
- Incidents: **S.C.O.P.E.**: stabilise → clarify → observe → prove → eliminate/prevent.
- Design: **R.A.D.I.O.S.**: requirements → architecture → deep-dive → issues → operations → summary.
- Know the classic senior incidents: keepalive 502s, CPU throttling, 5-second DNS, native memory OOM, host exhaustion, arch/libc mismatch, compromise response, subnet overlap.
- Judgement answers need a default, the conditions for deviating, and a "no, but here's how" when saying no.

**Interview questions**

1. **Q: What separates a senior answer from a mid-level one in an incident question?**
   A: The senior candidate mitigates user impact first, asks what changed, names precise signals and commands, tests one hypothesis at a time, and ends with prevention and a post-mortem, while considering blast radius and security.

2. **Q: How do you approach a system design question about a container platform?**
   A: Clarify requirements and constraints, sketch the architecture, deep-dive the hardest parts (supply chain, deploy safety, multi-tenancy), cover failure modes and operations, and summarise the trade-offs and next phases.

3. **Q: What's your default stance on running databases in containers?**
   A: The concern is operations, not containers. For critical production I prefer managed databases or a mature operator with a skilled team. For small or non-critical systems, containers with named volumes, tested backups and restores are acceptable.
