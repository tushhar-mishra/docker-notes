# 55 · What to Learn Next

> **Level:** ★ foundation
>
> **You will learn:** where your Docker knowledge leads (Kubernetes first, then the surrounding DevOps/SRE skills), a suggested order, and **capstone projects** that prove your skills end to end and make good interview stories.
>
> **You need:** topics 01–54.

---

## What it is

A map of next steps, with the "why" for each, and six projects that tie Docker to real production practice.

## Why it exists

Docker is rarely the whole job. It's the **base layer** of modern platforms. Interviewers and real teams expect you to connect it to orchestration, infrastructure as code, CI/CD, observability, cloud and security. Knowing the order saves months.

## Analogy

You've learned to build and drive one excellent car (containers). Next you learn **traffic systems** (orchestration), **road building** (infrastructure as code), **logistics** (CI/CD), and **traffic monitoring** (observability).

---

## How it works inside: the learning path

### Stage 1: Kubernetes (the direct continuation)

In this order, each step building on topic 53:
1. **Core objects**: Pods, Deployments, ReplicaSets, Services, Ingress/Gateway API, ConfigMaps, Secrets, Namespaces.
2. **Workload types**: StatefulSets, DaemonSets, Jobs/CronJobs, init containers, sidecars.
3. **Scheduling and resources**: requests/limits, QoS classes, node selectors/affinity, taints/tolerations, HPA (Horizontal Pod Autoscaler).
4. **Storage**: PV, PVC, StorageClasses, CSI.
5. **Networking**: CNI (Calico/Cilium), Services and kube-proxy/eBPF, NetworkPolicy, CoreDNS.
6. **Security**: RBAC, ServiceAccounts, Pod Security Standards, securityContext, admission control (Kyverno), image policies.
7. **Packaging and delivery**: Helm, Kustomize, GitOps (Argo CD or Flux).
8. **Operations**: upgrades (cordon/drain), etcd backups, troubleshooting nodes with crictl/nsenter.

Certifications that match this path well: **CKA** (Certified Kubernetes Administrator), then **CKS** (Security). **CKAD** if you're more on the application side.

### Stage 2: Linux and networking depth (you have a head start)

- eBPF basics (Cilium, observability tools like bpftrace and Inspektor Gadget),
- nftables, conntrack, routing, TLS and certificates in depth,
- systemd advanced (units, cgroups via slices, journald).

### Stage 3: Infrastructure as Code

- **Terraform/OpenTofu**: create VMs, networks, registries, Kubernetes clusters (EKS/GKE/AKS) as code.
- **Ansible**: configure Docker hosts (`daemon.json`, packages, users) as code: exactly the "provisioning" sections of topics 04 and 34.

### Stage 4: CI/CD platforms

- GitHub Actions or GitLab CI in depth (reusable workflows, OIDC to cloud, environments, approvals). Topic 46 is your base.
- Release strategies: blue/green, canary, feature flags. Progressive delivery (Argo Rollouts).

### Stage 5: Observability and SRE practice

- Prometheus (PromQL), Alertmanager, Grafana; Loki; OpenTelemetry tracing.
- SLOs (Service Level Objectives), error budgets, incident management, post-mortems.

### Stage 6: Cloud

Pick one provider deeply (AWS is the most common in job ads): IAM, VPC networking, a managed container service (ECS/Fargate, then EKS), ECR, Secrets Manager, CloudWatch.

### Keep current with Docker

- Read the **Docker Engine release notes** (watch for deprecations, minimum API versions, default changes like the containerd image store or the nftables backend).
- Follow **runc/containerd security advisories**.
- Re-check sample outputs in these notes against your installed version from time to time. Docker changes fast.

---

## Commands / files: capstone projects

Each project is a portfolio piece: a public git repository with a README explaining **design choices** and **what you'd do differently in production**.

### Project 1: Production-grade single-host app

- A small API (any language) + PostgreSQL + Redis + Traefik.
- Multi-stage Dockerfile, non-root, distroless/slim, healthchecks, graceful shutdown, JSON logs.
- Compose with secrets, internal network for the DB, limits, log rotation, `up --wait`.
- `deploy.sh` with automatic rollback. Nightly `pg_dump` with a systemd timer, plus a tested restore.
- **Proves:** topics 14–20, 23–39, 44–45.

### Project 2: Secure CI/CD pipeline

- GitHub Actions: lint (`--check`, hadolint), build with GHA cache, test stage, Trivy gate, multi-arch push to GHCR, SBOM + provenance, keyless cosign signing.
- A deploy job that verifies the signature and deploys by digest to the Project 1 server over an SSH context.
- **Proves:** topics 21–22, 43, 46.

### Project 3: Observability stack

- Add cAdvisor, node_exporter, dockerd metrics, Prometheus + alert rules (OOM, memory vs limit, throttling, disk), Grafana dashboards, Loki + Fluent Bit/Alloy for logs.
- Trigger each alert on purpose and screenshot it.
- **Proves:** topics 31, 33, 47.

### Project 4: Break-fix runbook

- Script all 14 scenarios from topic 52 (setup + verify scripts), plus two of your own.
- Write a runbook entry for each (symptom, commands, root cause, fix, prevention).
- **Proves:** topics 50–52. Excellent interview material ("tell me about a time…").

### Project 5: Same app on Kubernetes

- Move Project 1 to a kind/k3d cluster: Deployment, StatefulSet (or a managed DB), Services, Ingress, probes, resources, securityContext (restricted), NetworkPolicy, external secrets (or Sealed Secrets), Helm or Kustomize, Argo CD.
- Repeat three break-fix scenarios in Kubernetes (OOMKilled, ImagePullBackOff, readiness failure) and document the differences.
- **Proves:** topic 53 and your first Kubernetes stage.

### Project 6: Hardening report

- Take Project 1 and produce a short security report: docker-bench-security before/after, capability drops, seccomp, read-only root, rootless or userns-remap trial, image CVE counts before/after base changes, secrets handling.
- **Proves:** topics 40–44.

---

## Hands-on lab

1. Choose **one** capstone to start this week (Project 1 is the natural first).
2. Create the repository with a README skeleton: Goal, Architecture diagram, How to run, Design decisions, Production gaps.
3. Set a schedule: one project every 2–3 weeks, Kubernetes basics in parallel.
4. After each project, add 2–3 entries to your own interview story list: problem, what you did, the result.

---

## Where you'll use it

- **Job applications:** link the repositories. They show practical depth that a certificate alone doesn't.
- **Interviews:** the capstones and break-fix runbooks become your concrete examples.
- **Work:** the same patterns apply directly to real services.

---

## When it breaks (learning traps)

| Trap | Better |
|---|---|
| Jumping into Kubernetes without container basics | You've done this right: Docker first |
| Tutorials only, no projects | Build, break, fix, document |
| Collecting tools instead of understanding mechanisms | Always ask "what happens underneath?" (namespaces, cgroups, iptables) |
| Not writing things down | READMEs, runbooks, post-mortems |
| Learning outdated practices from old blog posts | Check dates; prefer official docs and release notes |

---

## Quick recap

- Next: **Kubernetes** (objects → workloads → resources → storage → networking → security → Helm/GitOps → operations), with CKA/CKS as milestones.
- Around it: Linux/eBPF depth, Terraform/Ansible, CI/CD, observability/SRE, one cloud in depth.
- Prove it with **capstones**: production Compose app, secure CI/CD, observability, break-fix runbooks, Kubernetes migration, hardening report.
- Stay current with Docker/runc/containerd release notes and advisories.

**Interview questions**

1. **Q: What would you learn after Docker, and why?**
   A: Kubernetes, because it orchestrates the containers I already understand across many nodes. Then IaC (Terraform/Ansible), CI/CD and observability, since production platforms combine all of them.

2. **Q: Tell me about a project where you used Docker end to end.**
   A: (Use a capstone: the architecture, the key decisions such as distroless, non-root, health checks, digest deploys and signing, one incident you reproduced and fixed, and what you'd improve.)

3. **Q: How do you keep your container knowledge current?**
   A: Release notes for Docker Engine, containerd and runc, security advisories, testing changes in a lab before production, and revisiting defaults (image store, firewall backend, API versions) after upgrades.
