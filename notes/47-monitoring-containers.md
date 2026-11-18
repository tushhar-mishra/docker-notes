# 47 · Monitoring Containers

> **Level:** ★★ intermediate
>
> **You will learn:** what to measure on a Docker host and where each number comes from (cgroups, the daemon, the host), the built-in tools (`docker stats`, `docker events`, `docker system df`), a standard Prometheus stack (**cAdvisor** for container metrics, **node_exporter** for the host, **dockerd's own metrics endpoint**, Grafana), shipping logs centrally, and a practical list of **alerts** that catch real incidents.
>
> **You need:** topic 31 (cgroup files), topic 32 (health), topic 33 (logging), topic 34 (`metrics-addr`), topic 10 (`stats`, `events`).

---

## What it is

Monitoring answers three questions continuously, without anyone typing commands:
1. **Is it up and healthy?** (state, health checks, restarts)
2. **Is it about to break?** (memory near limit, CPU throttling, disk filling, errors rising)
3. **What happened?** (events, logs, history for the post-mortem)

Three kinds of data:

| Signal | Source on a Docker host | Typical tools |
|---|---|---|
| **Metrics** (numbers over time) | cgroup files per container, `/proc` and `/sys` for the host, dockerd's `/metrics`, the app's `/metrics` | cAdvisor, node_exporter, Prometheus, Grafana |
| **Logs** (text events) | Container stdout/stderr via the logging driver | Fluent Bit / Vector / Grafana Alloy → Loki / Elasticsearch / cloud logging |
| **Events** (lifecycle) | `docker events` (die, oom, kill, health_status, restart…) | Event exporters, alerting |

(Traces, from OpenTelemetry instrumentation in apps, are the fourth signal. They're app-level, not Docker-specific.)

---

## Why it exists

Containers fail quietly: a child process OOM-killed while the container stays "Up" (topic 31), a service throttled to 50% speed, a crash loop restarting every minute with `unless-stopped`, logs filling the disk at 2 GB/day. Without monitoring, you find out from users. With it, you get an alert **before** or **as** it happens, plus the history to understand why.

---

## Analogy

A **hospital ward's monitoring system**:
- **Metrics** = heart rate, blood pressure and oxygen graphs for every bed, kept for weeks.
- **Events** = the ward log: "Bed 4 admitted 10:02, alarm 10:15, discharged 11:30".
- **Logs** = each patient's own diary.
- **Alerts** = the alarm that rings at the nurses' station when oxygen drops, **before** the patient collapses.
- **cAdvisor** = the bedside monitor attached to every patient automatically. **node_exporter** = the monitor for the building's power and water.

---

## How it works inside

### Where container metrics come from

Everything per-container comes from the **cgroup files** you read by hand in topics 11 and 31, plus the network namespace's counters:

| Metric | cgroup v2 source | cAdvisor metric name |
|---|---|---|
| CPU used | `cpu.stat` `usage_usec` | `container_cpu_usage_seconds_total` |
| CPU throttling | `cpu.stat` `nr_throttled`, `throttled_usec` | `container_cpu_cfs_throttled_periods_total`, `..._seconds_total` |
| Memory (working set) | `memory.current` − inactive file (`memory.stat`) | `container_memory_working_set_bytes` |
| Memory limit | `memory.max` | `container_spec_memory_limit_bytes` |
| OOM kills | `memory.events` `oom_kill` | `container_oom_events_total` |
| Network in/out | netns interface counters | `container_network_receive_bytes_total`, `..._transmit_bytes_total` |
| Disk I/O | `io.stat` | `container_fs_reads_bytes_total`, `container_fs_writes_bytes_total` |
| Processes | `pids.current` | `container_processes`/`container_threads` |

**cAdvisor** (Container Advisor, from Google) reads these for every container on the host, adds the container name, image and labels (from the Docker API), and exposes them at `/metrics` in **Prometheus format**. Kubernetes' kubelet has cAdvisor built in, so the same metric names work there.

**Working set** is the memory number that matters for OOM risk: usage minus cache the kernel can easily drop. Alert on `working_set / limit`, not raw usage.

### dockerd's own metrics

With `"metrics-addr": "127.0.0.1:9323"` in `daemon.json` (topic 34), dockerd exposes Prometheus metrics about **itself**:
- `engine_daemon_container_states_containers{state="running|paused|stopped"}`,
- `engine_daemon_container_actions_seconds` (how long create/start/delete take; slow = daemon or storage trouble),
- `engine_daemon_health_checks_failed_total`, `engine_daemon_health_checks_total`,
- `engine_daemon_events_total`, `engine_daemon_network_actions_seconds`,
- Go runtime/process metrics (`go_goroutines`, `process_resident_memory_bytes` of dockerd).

### The host (node_exporter)

Container problems often come from the **host**: the disk holding `/var/lib/docker` filling, inodes running out, host memory pressure, conntrack table full (topic 30), clock drift. **node_exporter** exposes host metrics (`node_filesystem_avail_bytes`, `node_memory_MemAvailable_bytes`, `node_nf_conntrack_entries`, `node_load1`, …).

### Prometheus and Grafana in one sentence each

- **Prometheus** = a time-series database that **scrapes** (pulls over HTTP) `/metrics` endpoints every N seconds, stores the numbers, evaluates **alert rules**, and sends alerts to **Alertmanager** (which routes them to Slack, email, PagerDuty…).
- **Grafana** = dashboards on top of Prometheus (and Loki for logs).

### Logs, centrally

Pattern from topic 33: keep a local driver with rotation (`json-file`/`local`/`journald`), and run a **shipper** container that reads the logs and sends them to a central store:
- **Fluent Bit / Vector / Grafana Alloy** reading `/var/lib/docker/containers/*/*-json.log` (json-file) or the journal,
- adding labels (container name, image, Compose project/service from Docker labels),
- sending to **Loki**, Elasticsearch/OpenSearch, Splunk, or a cloud service.

### Events

`docker events` is a live stream that dockerd only keeps briefly. To keep history or alert on it, have a small agent forward events (some log shippers and monitoring agents do this), or rely on the metrics they affect (`container_oom_events_total`, restart counters, health metrics).

### What to alert on (a solid starter set)

(**PromQL** = Prometheus Query Language. `rate(x[5m])` = per-second increase over 5 minutes; `increase(x[10m])` = total increase over 10 minutes. **SLO** = Service Level Objective, a reliability target such as "99.9% of requests succeed in under 300 ms".)

| Alert | Expression idea (PromQL) | Why |
|---|---|---|
| Container down / missing | Expected container absent from `container_last_seen` for > 2 min | Service gone |
| **Restart loop** | Restart count increasing (via an exporter or events), or `time() - container_start_time_seconds < 60` repeatedly | Crash loops hide behind `unless-stopped` |
| **OOM kill** | `increase(container_oom_events_total[10m]) > 0` | Includes child-process OOMs |
| **Memory near limit** | `container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9` for 10m | OOM soon |
| **CPU throttling** | `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25` | Latency spikes |
| **Unhealthy** | `engine_daemon_health_checks_failed_total` rising, or container health status from an exporter | Broken but running |
| **Docker disk filling** | `node_filesystem_avail_bytes{mountpoint="/var/lib/docker"} / size < 0.15` (or the root FS if not separate) | Everything breaks at 100% |
| **Inodes low** | `node_filesystem_files_free / node_filesystem_files < 0.1` | "No space left" with free blocks (topic 48) |
| **dockerd down / slow** | Scrape of `:9323` failing; `engine_daemon_container_actions_seconds` p99 high | Daemon trouble |
| **conntrack nearly full** | `node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8` | Random connection drops |
| **App SLOs** | Error rate and latency from the app's own metrics | What users actually feel |

Alert on **symptoms users feel** (errors, latency) for paging, and on **causes** (memory near limit, disk filling) as warnings. That avoids alert fatigue.

---

## Commands / files

### Built-in quick looks

```bash
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.PIDs}}'
docker events --since 1h --filter event=die --filter event=oom --filter event=health_status \
  --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}} {{.Actor.Attributes.exitCode}}'
docker ps --format '{{.Names}} {{.Status}}' | grep -E 'Restarting|unhealthy'
docker system df
```

### A monitoring stack with Compose

`monitoring/compose.yaml`:

```yaml
name: monitoring

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    privileged: true                     # reads cgroups/devices of all containers (an agent trade-off)
    devices: ["/dev/kmsg"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    restart: unless-stopped

  node-exporter:
    image: quay.io/prometheus/node-exporter:v1.9.1
    command: ["--path.rootfs=/host"]
    pid: host
    network_mode: host                   # sees the host's network stats; listens on host :9100
    volumes:
      - /:/host:ro,rslave
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v3.5.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
      - promdata:/prometheus
    ports: ["127.0.0.1:9090:9090"]
    extra_hosts: ["host.docker.internal:host-gateway"]
    restart: unless-stopped

  grafana:
    image: grafana/grafana:12.1.0
    ports: ["127.0.0.1:3000:3000"]
    volumes: [grafanadata:/var/lib/grafana]
    restart: unless-stopped

volumes:
  promdata:
  grafanadata:
```

(Check each project's current release and pin it. The versions above are examples.)

`monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

rule_files: [/etc/prometheus/alerts.yml]

scrape_configs:
  - job_name: cadvisor
    static_configs: [{ targets: ["cadvisor:8080"] }]
  - job_name: node
    static_configs: [{ targets: ["host.docker.internal:9100"] }]
  - job_name: dockerd
    static_configs: [{ targets: ["host.docker.internal:9323"] }]
```

- dockerd's metrics must listen where Prometheus can reach them: with `metrics-addr` on `127.0.0.1`, a container can't reach it. Use the docker0 gateway IP (e.g. `"metrics-addr": "172.17.0.1:9323"`) or run Prometheus with host networking. **Never** expose `:9323` publicly.

`monitoring/alerts.yml`:

```yaml
groups:
  - name: containers
    rules:
      - alert: ContainerOOMKilled
        expr: increase(container_oom_events_total{name!=""}[10m]) > 0
        labels: { severity: warning }
        annotations:
          summary: "OOM kill in {{ $labels.name }}"

      - alert: ContainerMemoryNearLimit
        expr: |
          (container_memory_working_set_bytes{name!=""}
            / on(id) container_spec_memory_limit_bytes{name!=""} > 0) > 0.9
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.name }} at {{ $value | humanizePercentage }} of its memory limit"

      - alert: ContainerCPUThrottled
        expr: |
          rate(container_cpu_cfs_throttled_periods_total{name!=""}[5m])
            / rate(container_cpu_cfs_periods_total{name!=""}[5m]) > 0.25
        for: 15m
        labels: { severity: warning }

      - alert: DockerDiskFilling
        expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.15
        for: 10m
        labels: { severity: critical }
```

- `{name!=""}` = only real containers (cAdvisor also reports cgroups that aren't containers).
- `> 0` in the memory rule filters out containers **without** a limit (limit = 0).

### Query a few metrics directly

```bash
docker compose -f monitoring/compose.yaml up -d
curl -s 'http://127.0.0.1:9090/api/v1/query?query=container_memory_working_set_bytes{name="counter-web-1"}' | python3 -m json.tool | grep -A2 '"value"'
```

```
                "value": [
                    1763371234.5,
                    "48234496"
```

- About 46 MiB working set for `counter-web-1`.

### dockerd metrics

```bash
curl -s 172.17.0.1:9323/metrics | grep -E '^engine_daemon_container_states_containers'
```

```
engine_daemon_container_states_containers{state="paused"} 0
engine_daemon_container_states_containers{state="running"} 7
engine_daemon_container_states_containers{state="stopped"} 2
```

### Ship logs with Fluent Bit (sketch)

```yaml
  fluent-bit:
    image: fluent/fluent-bit:4.0
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./fluent-bit.yaml:/fluent-bit/etc/fluent-bit.yaml:ro
    command: ["-c", "/fluent-bit/etc/fluent-bit.yaml"]
```

```yaml
# fluent-bit.yaml
pipeline:
  inputs:
    - name: tail
      path: /var/lib/docker/containers/*/*-json.log
      parser: docker
      tag: docker.*
  outputs:
    - name: loki
      match: "*"
      host: loki.internal
      labels: job=docker
```

(This needs the `json-file` driver, since the `local` driver's files aren't meant for external readers.)

---

## Hands-on lab

1. Bring up the monitoring stack. Set `metrics-addr` in `daemon.json` to the docker0 gateway IP and reload Docker.
2. Open Prometheus (`http://localhost:9090` from Windows) → Status → Targets: all three should be **UP**.
3. Graph `rate(container_cpu_usage_seconds_total{name!=""}[1m])` and start a CPU burner (`docker run -d --name burn --cpus 0.5 alpine sh -c 'while :; do :; done'`). Then graph the throttling ratio for `burn`.
4. Trigger an OOM (topic 31's example with `--restart on-failure`) and watch `ContainerOOMKilled` go **pending → firing** in Prometheus → Alerts.
5. In Grafana (`http://localhost:3000`, admin/admin), add Prometheus as a data source (`http://prometheus:9090`) and import a community "Docker / cAdvisor" dashboard by ID from grafana.com.
6. Clean up: `docker rm -f burn; docker compose -f monitoring/compose.yaml down -v`

---

## Where you'll use it

- **Every production Docker host** needs at least: container metrics (cAdvisor), host metrics (node_exporter), daemon metrics, central logs, and the alerts above.
- **Capacity planning:** right-sizing limits from working-set and throttling history.
- **Incident response and post-mortems:** "memory climbed for 3 days, then OOM": the graph tells the story.
- **Kubernetes:** the same cAdvisor metric names (via kubelet), node_exporter, Prometheus/Grafana (often via kube-prometheus-stack), and the same alert ideas (topic 53).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| cAdvisor shows no container names, only IDs | It can't reach the Docker API socket | Mount `/var/run` (read-only) as shown |
| cAdvisor metrics missing for some containers (rootless/other runtimes) | Different cgroup paths | Run cAdvisor for that environment, or use the runtime's metrics |
| Prometheus target `dockerd` DOWN | `metrics-addr` on 127.0.0.1, unreachable from the container | Bind to the docker0 gateway IP, or use host networking |
| Memory alert for containers with no limit | Division by 0 limit | Filter `limit > 0` |
| High Prometheus memory/disk | Too many series (high-cardinality labels) | Drop unused cAdvisor metrics (`--disable_metrics`, `--docker_only`), relabel |
| No OOM alerts although containers were OOM-killed | Alert only on container exit codes | Use `container_oom_events_total` / `memory.events` |
| Logs missing centrally | Shipper reading the wrong driver's files, or log rotation outpacing it | Match the driver; buffer settings; check shipper logs |
| Alert fatigue | Paging on causes rather than symptoms | Page on user-facing SLOs; warnings for causes |

---

## Quick recap

- Container metrics come from **cgroup files**. **cAdvisor** exports them (CPU, throttling, working-set memory, OOM events, network, I/O) in Prometheus format, with the same names Kubernetes uses.
- Add **node_exporter** (host: disk, inodes, memory, conntrack) and **dockerd's `metrics-addr`** (daemon health), scraped by **Prometheus**, shown in **Grafana**, routed by **Alertmanager**.
- Ship logs centrally with a shipper (Fluent Bit/Vector/Alloy → Loki/Elastic/cloud). Keep local rotation.
- Starter alerts: OOM kills, memory > 90% of limit, CPU throttling, restart loops, unhealthy, Docker disk/inodes, daemon down, conntrack.
- Page on what users feel. Warn on causes.

**Interview questions**

1. **Q: How do you monitor Docker containers in production?**
   A: Run cAdvisor for per-container metrics from cgroups, node_exporter for host metrics, and enable dockerd's Prometheus endpoint. Prometheus scrapes them, Grafana shows dashboards, and Alertmanager sends alerts. Logs are shipped by Fluent Bit/Vector to Loki or Elasticsearch, plus app-level metrics and traces.

2. **Q: Which memory metric should you alert on for OOM risk, and why?**
   A: The working set (`container_memory_working_set_bytes`) relative to the limit. It excludes easily reclaimable cache, so it reflects what the kernel's OOM decision is based on. Raw usage includes cache and overstates the risk.

3. **Q: How would you detect a container that keeps getting OOM-killed but never shows as stopped?**
   A: Child-process OOM kills don't stop the container, so watch the cgroup's `memory.events` oom_kill counter, exposed as `container_oom_events_total` by cAdvisor, plus `oom` events from `docker events` and kernel logs.
