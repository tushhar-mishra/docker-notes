# 33 · Logging

> **Level:** ★★ intermediate
>
> **You will learn:** how a container's stdout/stderr travel to a **logging driver**, what the `json-file` log file looks like on disk, why the default setup **can fill your disk**, how to set rotation (`max-size`, `max-file`) and why the `local` driver is a better default, the remote drivers (`journald`, `syslog`, `fluentd`, `gelf`, `awslogs`…), dual logging, blocking vs non-blocking delivery, and common app-side mistakes (buffered output, logging to files).
>
> **You need:** topic 03 (shim keeps stdio), topic 10 (`docker logs`), topic 13 (log files and disk).

---

## What it is

The container way to log is simple: **the app writes to stdout and stderr**, and the platform collects it. The app doesn't manage log files.

```
app (PID 1) ──stdout/stderr──► containerd-shim ──► dockerd ──► logging driver ──► destination
                                                                 │
                                                                 ├─ json-file → /var/lib/docker/containers/<id>/<id>-json.log
                                                                 ├─ local     → /var/lib/docker/containers/<id>/local-logs/container.log
                                                                 ├─ journald  → systemd journal
                                                                 ├─ syslog / fluentd / gelf / awslogs / gcplogs / splunk → remote systems
                                                                 └─ none      → discarded
```

---

## Why it exists

- **Separation of concerns:** apps shouldn't need to know about log rotation, file paths or log servers. The "twelve-factor app" principle: treat logs as event streams.
- **Uniformity:** every container's logs are collected the same way, whatever language it's written in.
- **Ephemeral containers:** files inside a container vanish with it (topic 23). The stdout stream is captured outside the container.

The danger: by default, Docker writes these logs to disk **forever, with no size limit**.

---

## Analogy

Every room in a hotel has a **pneumatic tube** to the front desk (stdout/stderr). Guests just drop notes in the tube. The front desk (dockerd) files them, either in a local filing cabinet (`json-file`/`local`) or by forwarding them to head office (remote drivers). If nobody ever throws old notes away (no rotation), the filing cabinet eventually fills the whole lobby.

---

## How it works inside

### The path of one log line

1. The app writes `"GET /health 200\n"` to file descriptor 1 (stdout).
2. That fd is a **pipe** (or a PTY if `-t`), whose other end is held by the **shim** (topic 03).
3. The shim forwards the bytes to dockerd, through a FIFO (a named pipe file) that dockerd reads.
4. dockerd splits the stream into **messages** at newlines (lines longer than **16 KB** are split into "partial" messages), adds a timestamp and the stream name (stdout/stderr), and hands each message to the container's **logging driver**.
5. The driver writes or sends it.

### `json-file` (the default)

One JSON object per line in `/var/lib/docker/containers/<id>/<id>-json.log`:

```json
{"log":"GET /health 200\n","stream":"stdout","time":"2026-11-17T08:40:01.123456789Z"}
```

- `log` = the line, including its newline. `stream` = stdout or stderr. `time` = when dockerd received it (UTC).
- **Default: no rotation.** One file grows forever, until the disk is full (topic 13's 6.2 GB log).
- Overhead: every line is JSON-wrapped (often 2× the raw size).

### `local` (recommended local driver)

- Stores logs in an efficient binary format (protobuf) with **rotation on by default**: 20 MB per file, 5 files, compressed. About **100 MB per container** at most.
- `docker logs` works the same.
- Not meant to be read by other tools directly. If a log shipper reads Docker's JSON files from disk, keep `json-file` (with rotation).

### Rotation options (json-file and local)

| Option | Meaning |
|---|---|
| `max-size` | Rotate when the file reaches this size (`10m`, `100m`) |
| `max-file` | Keep this many files (current + rotated) |
| `compress` | gzip rotated files (`true`/`false`) |

Set the **default** for new containers in `/etc/docker/daemon.json` (topic 34):

```json
{
  "log-driver": "local",
  "log-opts": { "max-size": "20m", "max-file": "5" }
}
```

or for json-file:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3", "compress": "true" }
}
```

**Important:** daemon defaults apply only to containers **created after** the change (and after restarting dockerd). Existing containers keep their old log settings until they're **recreated** (not just restarted).

### Remote drivers

| Driver | Sends to | Notes |
|---|---|---|
| `journald` | systemd journal on the host | Read with `journalctl CONTAINER_NAME=web`. Journal has its own size limits/rotation |
| `syslog` | A syslog server (local or remote, UDP/TCP/TLS) | Classic Unix logging |
| `fluentd` | A Fluentd/Fluent Bit collector (forward protocol) | Common in log pipelines |
| `gelf` | Graylog / Logstash (GELF = Graylog Extended Log Format) | UDP/TCP |
| `awslogs` | AWS CloudWatch Logs | Needs AWS credentials for **dockerd** |
| `gcplogs` | Google Cloud Logging | |
| `splunk` | Splunk HTTP Event Collector | |
| `none` | Nowhere | `docker logs` shows nothing |

### Dual logging (the local cache)

Since Docker 20.10, when you use a remote driver, Docker **also** keeps a small local copy (the "cache", using the `local` format with rotation), so `docker logs` still works. Control with log options `cache-disabled`, `cache-max-size`, `cache-max-file`.

### Blocking vs non-blocking delivery

- **`mode=blocking`** (default): the app's write to stdout **waits** until dockerd/the driver has taken the message. If the log destination is slow or down (a remote syslog server, CloudWatch throttling, a full disk), **the app's writes block and the app freezes**.
- **`mode=non-blocking`** + `max-buffer-size=4m`: messages go into an in-memory ring buffer. If the buffer is full, **messages are dropped** instead of blocking the app.
- Choose: blocking = never lose logs (but risk stalls). Non-blocking = app availability first (but may drop logs in bursts). For remote drivers, non-blocking is often the safer production choice.

### The common way to ship logs

Many teams keep `json-file` (with rotation) or `journald` locally, and run a **log shipper agent** (Fluent Bit, Vector, Promtail/Alloy, the Datadog/Elastic agent) as a container that reads the log files or the journal and sends them to a central system (Loki, Elasticsearch, Splunk, CloudWatch). That keeps the app independent of the log backend and avoids blocking (topic 47).

### App-side mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Logging to a file inside the container | `docker logs` empty; file fills the writable layer; lost on `rm` | Log to stdout/stderr; or symlink the file to `/dev/stdout` (the nginx image does `ln -sf /dev/stdout /var/log/nginx/access.log`) |
| **Output buffering** (Python, some C programs) | Logs appear in big delayed chunks, or are lost on crash | Python: `PYTHONUNBUFFERED=1` or `python -u`; flush in code |
| Multi-line stack traces | Each line becomes a separate message | Log as JSON (structured logging) so one event = one line |
| Logging secrets / personal data | They end up in every log system | Filter at the app level |
| Huge debug volume in production | Disk, cost, and blocking pressure | Log levels; sampling |

---

## Commands / files

### Which driver does a container use?

```bash
docker info --format '{{.LoggingDriver}}'
docker inspect -f '{{.HostConfig.LogConfig.Type}} {{.HostConfig.LogConfig.Config}}' web
```

```
json-file
json-file map[]
```

- `map[]` = no options, so **no rotation**.

### Look at the raw json-file

```bash
docker run -d --name logdemo alpine sh -c 'i=0; while true; do echo "line $i"; echo "error $i" >&2; i=$((i+1)); sleep 1; done'
sleep 3
sudo tail -3 $(docker inspect -f '{{.LogPath}}' logdemo)
```

```
{"log":"error 2\n","stream":"stderr","time":"2026-11-17T08:45:03.002Z"}
{"log":"line 3\n","stream":"stdout","time":"2026-11-17T08:45:04.003Z"}
{"log":"error 3\n","stream":"stderr","time":"2026-11-17T08:45:04.003Z"}
```

### Per-container rotation

```bash
docker run -d --name rotated --log-opt max-size=1m --log-opt max-file=3 \
  alpine sh -c 'while true; do head -c 2000 /dev/urandom | base64; done'
sleep 10
sudo ls -lh /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' rotated)/
```

```
-rw-r----- 1 root root 1000K Nov 17 14:50 3f2a…-json.log
-rw-r----- 1 root root  977K Nov 17 14:50 3f2a…-json.log.1
-rw-r----- 1 root root  977K Nov 17 14:50 3f2a…-json.log.2
...
```

- Never more than 3 files of ~1 MB. `docker logs` reads across all of them.

### The `local` driver

```bash
docker run -d --name loc --log-driver local alpine sh -c 'while true; do date; sleep 1; done'
docker logs --tail 2 loc
sudo ls -lh /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' loc)/local-logs/
```

```
Mon Nov 17 09:20:11 UTC 2026
Mon Nov 17 09:20:12 UTC 2026
-rw-r----- 1 root root 1.2K Nov 17 14:50 container.log
```

### journald

```bash
docker run -d --name jd --log-driver journald alpine sh -c 'echo "hello journal"; sleep 600'
journalctl CONTAINER_NAME=jd --no-pager -o verbose | grep -E 'MESSAGE=|CONTAINER_ID=|CONTAINER_NAME='
```

```
    MESSAGE=hello journal
    CONTAINER_ID=4b5c6d7e8f90
    CONTAINER_NAME=jd
```

- Each message is a journal entry with fields you can filter on (`CONTAINER_NAME`, `CONTAINER_ID`, `IMAGE_NAME`, `CONTAINER_TAG`).

### Non-blocking mode

```bash
docker run -d --log-driver syslog \
  --log-opt syslog-address=udp://10.0.0.5:514 \
  --log-opt mode=non-blocking --log-opt max-buffer-size=4m \
  --log-opt tag='{{.Name}}/{{.ID}}' \
  myapp:1.0
```

- `tag` = a template for the identifier attached to each message (handy for finding the container in the remote system).

### Emergency: a huge log file right now

```bash
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail -3
sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log
```

Then fix it permanently: set rotation in `daemon.json` and **recreate** the container.

---

## Hands-on lab

1. Run `logdemo`, read its raw JSON log, and run `docker logs logdemo 2>/dev/null | head -3` (stdout only) vs `docker logs logdemo 2>&1 >/dev/null | head -3` (stderr only).
2. **Buffering:**
   ```bash
   docker run -d --name buf python:3.13-slim python -c "import time
   for i in range(100): print('tick', i); time.sleep(1)"
   sleep 5; docker logs buf | wc -l
   docker run -d --name unbuf -e PYTHONUNBUFFERED=1 python:3.13-slim python -c "import time
   for i in range(100): print('tick', i); time.sleep(1)"
   sleep 5; docker logs unbuf | wc -l
   ```
   `buf` shows 0 lines (output buffered, because stdout isn't a terminal). `unbuf` shows ~5.
3. **Rotation:** run `rotated`, watch the files cycle, and confirm total disk use stays capped.
4. **Daemon default:** set `"log-driver": "local"` with `max-size` in `/etc/docker/daemon.json`, `sudo systemctl restart docker`, start a new container, and check `docker inspect -f '{{.HostConfig.LogConfig}}'`. An **old** container still shows `json-file` until you recreate it.
5. Clean up: `docker rm -f logdemo rotated loc jd buf unbuf`

---

## Where you'll use it

- **Day one on any Docker host:** set log rotation in `daemon.json`. Disk-full from logs is one of the most common Docker incidents.
- **Central logging:** journald/json-file + a shipper (Fluent Bit, Vector, Alloy) → Loki/Elasticsearch/Splunk/CloudWatch.
- **App reviews:** "does it log to stdout, unbuffered, in structured JSON?"
- **Kubernetes:** the same model. kubelet/containerd write container stdout to `/var/log/pods/...` with rotation, and node agents ship them (topic 53).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| Disk full, a `*-json.log` is many GB | json-file without rotation | `truncate -s 0` now; set `max-size`/`max-file` (or `local`) and recreate containers |
| `docker logs` shows nothing | App logs to files, or output is buffered, or driver is `none` | Log to stdout; unbuffer; check driver |
| `Error response from daemon: configured logging driver does not support reading` | Remote driver with dual logging cache disabled | Re-enable the cache, or read from the remote system |
| App freezes or slows when the log server is down | Blocking mode with a remote driver | `mode=non-blocking` + `max-buffer-size`, or a local driver + shipper |
| Lines missing in the remote system during bursts | Non-blocking buffer overflowed | Bigger `max-buffer-size`, fewer logs, or a local driver + shipper |
| Stack traces split into many entries | Multi-line output | Structured JSON logging, or multi-line parsing in the shipper |
| Changed daemon log settings, but containers still use the old ones | Settings are fixed at container creation | Recreate the containers |
| dockerd fails to start after editing log settings | JSON syntax error, or an invalid option in `daemon.json` | `journalctl -u docker`; validate with `dockerd --validate --config-file /etc/docker/daemon.json` |

---

## Quick recap

- Apps log to **stdout/stderr**. The shim and dockerd pass each line to the container's **logging driver**.
- Default `json-file` has **no rotation**: set `max-size`/`max-file`, or use the **`local`** driver (rotation on by default). Settings apply to newly created containers.
- Remote drivers (journald, syslog, fluentd, gelf, awslogs…) send logs elsewhere. Dual logging keeps `docker logs` working.
- `mode=blocking` (default) can freeze apps if the destination stalls. `non-blocking` can drop logs.
- App side: unbuffered output, structured JSON, no log files inside the container.

**Interview questions**

1. **Q: How do container logs work in Docker, and where are they stored by default?**
   A: The app writes to stdout/stderr, the shim and dockerd capture the streams, and the logging driver stores them. The default `json-file` driver writes JSON lines to `/var/lib/docker/containers/<id>/<id>-json.log`, without rotation unless configured.

2. **Q: A Docker host keeps running out of disk because of logs. What's the permanent fix?**
   A: Set rotation defaults in `/etc/docker/daemon.json` (`log-opts` `max-size`/`max-file`, or the `local` driver), restart dockerd, and recreate existing containers. Truncate current huge files to recover space immediately.

3. **Q: What's the risk of the default blocking log mode with a remote logging driver?**
   A: If the log destination is slow or unavailable, writes to stdout block, which can stall the application. Non-blocking mode avoids that with an in-memory buffer, at the cost of possibly dropping messages.
