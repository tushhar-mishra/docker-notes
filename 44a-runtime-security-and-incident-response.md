# 44a · Runtime Security and Incident Response

> **Level:** ★★★ advanced.
>
> **You will learn:** how to **detect** attacks in running containers (not just prevent them), using runtime monitoring with **Falco** (eBPF-based system-call rules), **drift detection** (new or changed binaries in a container), auditing who uses the **Docker API**, and how to **respond** to a compromised container step by step: detect → triage → contain → preserve evidence → investigate → eradicate → recover → learn. You'll also learn when to treat the **host** as compromised.
>
> **You need:** topics 40–44 (security model, capabilities, seccomp/LSMs, rootless, supply chain, secrets), topic 11 (host-side inspection), topics 50–51 (debugging method and tools), topic 26 (eBPF definition).

---

## What it is

Topics 40–44 are about **prevention**: make attacks hard. This topic is about **detection and response**: assume something will eventually get through (a zero-day in your app, a stolen token, a malicious dependency), notice it quickly, limit the damage and recover cleanly.

| Phase | Question | Tools |
|---|---|---|
| **Detect** | Is something suspicious happening right now? | Falco, drift detection, audit logs, metrics anomalies (CPU spikes from miners), egress monitoring |
| **Respond** | How do we stop it without destroying evidence? | Network isolation, pause, evidence collection, forensics |
| **Recover** | How do we return to a known-good state? | Rebuild from source, rotate secrets, redeploy by digest |

---

## Why it exists

- Prevention is never perfect: vulnerable dependencies, new CVEs, misconfigurations and leaked credentials happen.
- Attackers who get into a container typically: open a shell, download tools or a crypto-miner, read secrets (env, mounted files, cloud metadata endpoints), scan the internal network, and try to escape to the host.
- The difference between a minor incident and a major breach is often **how fast you notice** and **how well you contain it** without wiping the evidence you need to understand it.
- Interviews for senior SRE/security-minded roles almost always include "a container is compromised, what do you do?"

---

## Analogy

**Burglar alarms and a police procedure.** Locks (prevention) stop most burglars. Motion sensors (Falco) catch the ones who get in anyway. When the alarm goes off, you don't bulldoze the house. You lock the doors so the burglar can't move to other rooms (containment), photograph everything before touching it (evidence), find how they got in (investigation), change all the locks and keys (eradication, secret rotation), and install a sensor on that window (lessons learned).

---

## How it works inside

### 1. Runtime detection with Falco

**Falco** (a CNCF project) watches **system calls** of every process on the host, using an **eBPF** probe (topic 26) or a kernel module. It enriches each event with container information (container name, image, Kubernetes Pod) and checks it against **rules**. When a rule matches, it emits an alert (stdout, file, syslog, webhook, or **Falcosidekick** to Slack/SIEM; a SIEM = Security Information and Event Management system that collects and correlates security logs).

Example of a rule (Falco's YAML rule language):

```yaml
- rule: Terminal shell in container
  desc: A shell was spawned in a container with an attached terminal
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name container=%container.name image=%container.image.repository
     cmdline=%proc.cmdline)
  priority: NOTICE
```

- `condition` = a filter over syscall events (`spawned_process` = an `execve` happened; `container` = not on the host).
- `output` = the alert text, with fields filled from the event.

Default rules cover the typical attack steps:

| Attack step | Example default rule |
|---|---|
| Interactive shell in a container | Terminal shell in container |
| Downloading and running new binaries | Drop and execute new binary in container (drift) |
| Reading secrets | Read sensitive file untrusted (e.g. `/etc/shadow`) |
| Privilege escalation attempts | Set setuid or setgid bit; PTRACE attached to process |
| Container escape attempts | Mount launched in privileged container; namespace changes (`setns`) |
| Tampering with the host | Write below binary dir / etc; Clear log activities |
| Suspicious network | Outbound connection to known mining pools; unexpected inbound connection |

Falco itself needs high privileges to watch the kernel (it's a host-level security agent, a deliberate trade-off like cAdvisor in topic 47). Run it as a hardened, monitored system service or container, and pin its version.

Alternatives and complements: **Tetragon** (Cilium's eBPF security observability and enforcement, which can kill processes), **Tracee** (Aqua), and commercial runtime-security platforms (Sysdig, Aqua, Prisma, Wiz/Defender sensors).

### 2. Drift detection

**Drift** = a running container's filesystem or processes differ from its image. In an immutable setup, **any** new executable in a running container is suspicious.
- **Prevent it**: `--read-only` + `noexec` tmpfs (topic 25). Downloaded binaries can't be written or executed.
- **Detect it manually**: `docker diff <c>` (topic 07/10). New files in `/tmp`, `/dev/shm`, `/usr/local/bin`, or changed binaries are red flags.
- **Detect it automatically**: Falco's drift rules; runtime platforms compare running executables against the image's SBOM.

### 3. Auditing the Docker API and host

Every `docker` command is an API call to a root daemon (topic 03). You need to know **who did what**:
- **dockerd logs** (`journalctl -u docker`). With debug logging, requests are logged in detail, but that's noisy.
- **auditd** (the Linux kernel audit system): watch Docker's files and socket. The CIS Benchmark recommends rules such as:
  ```
  -w /usr/bin/dockerd -k docker
  -w /var/lib/docker -k docker
  -w /etc/docker -k docker
  -w /usr/lib/systemd/system/docker.service -k docker
  -w /run/docker.sock -k docker
  ```
  Then `ausearch -k docker` shows which user and process touched them.
- **Authorization plugins** (e.g. Open Policy Agent's Docker authz plugin) can **block** dangerous API calls (`--privileged`, host mounts) and log every request.
- **Socket proxies** (topic 39) limit what tools that "need the socket" can do.
- **Docker `events`** (topic 10) shipped to your log system: `exec_create`, `exec_start` (someone ran `docker exec`), `container create` with unusual options.

### 4. Indicators of compromise (what to look for)

- New processes not in the image's normal command (`xmrig`, `nc`, `curl | sh`, `python -c`, shells).
- CPU at 100% in a normally idle container (crypto-mining).
- Outbound connections to unknown IPs/ports (mining pools often use ports like 3333/4444/14444), or DNS lookups of odd domains.
- Requests to the **cloud metadata endpoint** `169.254.169.254` from app containers (stealing the instance's IAM credentials).
- New files in `/tmp`, `/dev/shm`, hidden folders (`/tmp/.x`), modified `crontab`, `authorized_keys`, `.bashrc`.
- `docker exec` events nobody can explain, or new containers with `--privileged` / host mounts.

### 5. Incident response, step by step

**Step 1: Detect and triage (minutes).**
Confirm it's real: which container, image digest, host, since when, what alert. Decide severity: is data at risk? Is a host escape possible (privileged, socket mounted, root + host mounts)?

**Step 2: Contain (without destroying evidence).**
- **Isolate the network**: `docker network disconnect <net> <c>` for each network. The attacker loses its command-and-control link and can't move laterally, but processes and memory stay for analysis.
- Block egress for the host or service at the firewall/security group if needed.
- **Freeze**: `docker pause <c>` (cgroup freezer, topic 08) stops the attacker's processes while keeping memory.
- **Stop restart policies from "fixing" it**: `docker update --restart no <c>`.
- Restore service with **new** containers from a known-good image (different name), on a clean host if escape is possible.
- **Don't** `docker rm` or reboot yet: that erases the writable layer, memory and logs.

**Step 3: Preserve evidence.**
Collect before changing anything else (extends topic 50's `collect.sh`):
- `docker inspect`, `docker logs -t`, `docker events` history, `docker diff`,
- the image digest, and a copy of the image (`docker save`),
- the filesystem: `docker export` (merged view) and `docker commit` (writable layer as an image),
- process tree and binaries: `ps -ef --forest` on the host, `ls -l /proc/<pid>/exe`, `cp /proc/<pid>/exe` (recovers a binary **even if the attacker deleted it**),
- open network connections: `nsenter -t <pid> -n ss -tanp`,
- memory of suspicious processes: `gcore <pid>` (from the gdb package), or a full checkpoint with CRIU (experimental `docker checkpoint`),
- host logs: `journalctl`, `dmesg`, auditd, Falco alerts.
Hash the files (`sha256sum`) and store them somewhere the attacker can't reach. Note times in UTC.

**Step 4: Investigate.**
- **Entry point**: which vulnerability or credential? (App logs, WAF logs (WAF = Web Application Firewall, which filters malicious HTTP requests in front of the app), the image's CVEs from the SBOM (topic 43), recently leaked tokens.)
- **Actions**: what did they run, read and connect to? (Falco timeline, diff, shell history if any, egress logs.)
- **Scope**: other containers with the same image or vulnerability? Same credentials used elsewhere? **Did they reach the host?** Check for host-level changes: new users, SSH keys, systemd units, cron jobs, kernel modules, unexpected containers.
- If a host escape is plausible (privileged container, Docker socket, host root mounts, unpatched kernel/runc CVE), **treat the host as compromised**: rebuild it from scratch rather than cleaning it.

**Step 5: Eradicate.**
- Fix the root cause: patch the dependency/base image, close the misconfiguration, remove the socket mount.
- **Rotate every secret** the container could access: env vars, mounted secrets, cloud credentials (including instance role credentials obtainable from the metadata endpoint), database passwords, API tokens (topic 44).
- Rebuild images from source in CI (never `commit` an "cleaned" container into production).
- Hunt across the fleet for the same indicators.

**Step 6: Recover.**
Redeploy by **digest** from the fixed build, on clean hosts. Watch closely (extra alerts) for recurrence.

**Step 7: Learn.**
Write a **blameless post-mortem**: timeline, impact, root cause, what detection worked or failed, and action items (a new Falco rule, read-only rootfs, egress restrictions, faster patching, IMDSv2 (version 2 of AWS's Instance Metadata Service, which requires a session token; combined with a response hop limit of 1, containers behind Docker NAT can't reach it)/metadata blocking, etc.).

### 6. Hardening that makes response easier

- **Read-only root + noexec tmpfs**: attackers can't drop tools.
- **Egress control**: containers that only talk to their database shouldn't reach the internet (internal networks, topic 36; egress firewall/proxy).
- **Block the cloud metadata endpoint** from containers that don't need it (on AWS, require **IMDSv2** with a hop limit of 1 so containers can't fetch instance credentials).
- **Non-root, drop all capabilities, no-new-privileges, seccomp/LSM** (topics 41, 41a): fewer escape paths.
- **Central, tamper-resistant logs**: attackers often clear local logs.

---

## Commands / files

### Run Falco (lab)

Copy the exact `docker run` command for your Falco version from Falco's documentation (the required mounts change between versions). It looks like this:

```bash
docker run --rm -it --name falco --privileged \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v /proc:/host/proc:ro \
  -v /etc:/host/etc:ro \
  falcosecurity/falco:<version>
```

- `--privileged` and the host mounts are needed because Falco loads an eBPF probe into the host kernel and reads host process information. Run this only on a lab VM, or follow Falco's production install guide (package + systemd service).

In another terminal, behave like an attacker:

```bash
docker run -d --name victim nginx:1.27-alpine
docker exec -it victim sh
# inside:
cat /etc/shadow
cp /bin/busybox /tmp/.miner && chmod +x /tmp/.miner && /tmp/.miner echo hi
exit
```

Falco prints alerts similar to:

```
Notice A shell was spawned in a container with an attached terminal (evt_type=execve user=root process=sh parent=containerd-shim command=sh terminal=34816 container_id=7a1b2c3d4e5f container_name=victim image=nginx)
Warning Sensitive file opened for reading by non-trusted program (file=/etc/shadow process=cat command=cat /etc/shadow container_name=victim image=nginx)
Critical Executing binary not part of base image (proc_exe=/tmp/.miner proc_cmdline=.miner echo hi container_name=victim image=nginx)
```

(Exact rule names and fields depend on your Falco version and enabled rulesets. Some rules, like the drift rule, may need to be enabled explicitly.)

### Drift check by hand

```bash
docker diff victim
```

```
C /tmp
A /tmp/.miner
C /root
A /root/.ash_history
```

- `A /tmp/.miner` = a new file in a container that should never change. `.ash_history` = someone used an interactive shell.

### Audit who uses Docker

```bash
sudo apt-get install -y auditd
echo '-w /run/docker.sock -k docker
-w /etc/docker -k docker
-w /usr/bin/dockerd -k docker' | sudo tee /etc/audit/rules.d/docker.rules
sudo augenrules --load
docker ps >/dev/null
sudo ausearch -k docker -i | tail -6
```

```
type=SYSCALL msg=audit(11/18/2026 10:42:01.123:812) : arch=x86_64 syscall=connect success=yes exit=0 ... uid=tushar gid=tushar euid=tushar comm=docker exe=/usr/bin/docker key=docker
```

- Shows **which user** (`uid=tushar`) and **which program** (`exe=/usr/bin/docker`) connected to the socket, and when.

Watch `docker exec` usage live:

```bash
docker events --filter event=exec_start --format '{{.Time}} exec in {{.Actor.Attributes.name}}: {{.Actor.Attributes.execID}}'
```

### Containment and evidence (the response sequence)

```bash
C=victim
TS=$(date -u +%Y%m%dT%H%M%SZ); D=ir-$C-$TS; mkdir -p $D
# 1. contain
for n in $(docker inspect $C -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'); do docker network disconnect $n $C; done
docker update --restart no $C
# 2. preserve (processes still alive, network cut)
PID=$(docker inspect -f '{{.State.Pid}}' $C)
docker inspect $C > $D/inspect.json
docker logs -t $C > $D/logs.txt 2>&1
docker diff $C > $D/diff.txt
ps -ef --forest > $D/host-ps.txt
sudo nsenter -t $PID -n ss -tanp > $D/sockets.txt
for p in $(cat /sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)/cgroup.procs); do
  sudo cp /proc/$p/exe $D/exe-$p 2>/dev/null; sudo cat /proc/$p/cmdline | tr '\0' ' ' > $D/cmdline-$p.txt; done
docker pause $C
docker export $C | gzip > $D/fs.tar.gz
docker commit -m "IR $TS" $C forensics/$C:$TS
docker save forensics/$C:$TS | gzip > $D/forensic-image.tar.gz
sha256sum $D/* > $D/SHA256SUMS
echo "evidence in $D"
```

- Order matters: **cut the network first**, then **capture live state** (processes, sockets, binaries), then **freeze** and **image** the filesystem.
- `cp /proc/<pid>/exe` saves the running binary even if the attacker deleted the file on disk.
- Move `$D` off the host (to a secured bucket) before you rebuild anything.

---

## Hands-on lab

1. Run Falco on a lab VM, then perform the "attacker" actions in `victim`. Match each action to an alert.
2. Enable `--read-only --tmpfs /tmp:rw,noexec,nosuid` on a new `victim2` and repeat the attack. Which steps now fail outright?
3. Set up the auditd rules and identify which user ran `docker exec`.
4. Run the full containment + evidence script on `victim`. Then write a 10-line incident report (timeline, entry point (pretend), actions, containment, evidence, eradication steps including **which secrets you would rotate**).
5. Write a custom Falco rule that alerts when any container process connects to `169.254.169.254` (the cloud metadata endpoint). Hint: a condition on `evt.type=connect and fd.sip="169.254.169.254" and container`.
6. Clean up: `docker rm -f victim victim2 falco; docker rmi forensics/victim:*`

---

## Where you'll use it

- **Security operations:** runtime alerts feeding a SIEM, with on-call runbooks for "container compromised".
- **Compliance:** PCI DSS, SOC 2 and ISO 27001 auditors ask for runtime monitoring, audit trails and incident procedures.
- **Kubernetes:** Falco/Tetragon run as DaemonSets. The same response steps apply with `kubectl` (cordon the node, NetworkPolicy to isolate the Pod, `kubectl debug`, node-level forensics with crictl).
- **Senior interviews:** "walk me through responding to a compromised container" (see topic 54a).

---

## When it breaks

| Problem | Cause | Fix |
|---|---|---|
| Falco fails to start: driver/probe errors | Kernel/driver mismatch, missing BTF (BPF Type Format: kernel type information that modern eBPF programs need; check for /sys/kernel/btf/vmlinux), or not enough privileges | Use the driver your kernel supports (modern eBPF needs a recent kernel with BTF); follow the version's install docs |
| Alert flood (shells from your own debug sessions) | Default rules match normal admin work | Tune with exceptions (specific images, users, CI runners); don't disable whole rules |
| No alerts at all during a test | Rule disabled, wrong ruleset, or the event wasn't in a container | Check loaded rules (`falco --list`); test with the exact actions above |
| Evidence lost | Container removed or host rebooted before collection | Contain first (network disconnect, pause), collect, then rebuild |
| Attacker returns after cleanup | Credentials not rotated, root cause not fixed, or persistence on the host | Rotate everything reachable; patch; rebuild hosts if escape was possible |
| Can't tell who ran `docker exec` | No auditing; shared accounts | auditd rules, named accounts, events shipped centrally, authz plugin |

---

## Quick recap

- Prevention isn't enough. Add **runtime detection** (Falco/Tetragon: syscall rules for shells, drift, secret reads, escapes, mining) and **auditing** (auditd on the socket and config, Docker events, authz plugins).
- **Drift** = new executables in a running container: prevent it with read-only + noexec, detect it with `docker diff`/Falco.
- Response order: **detect → triage → contain (disconnect network, pause, stop restarts) → preserve (inspect, logs, diff, `/proc/PID/exe`, sockets, export/commit/save, hashes) → investigate → eradicate (patch, rotate all reachable secrets, rebuild) → recover by digest → post-mortem.**
- If escape to the host is plausible, **rebuild the host**.
- Egress control, metadata-endpoint protection and tamper-resistant central logs make attacks smaller and investigations easier.

**Interview questions**

1. **Q: A container is running a crypto-miner. Walk me through your response.**
   A: Confirm and scope it (which image digest, which hosts, since when). Contain by disconnecting the container's networks and pausing it, set its restart policy to no, and serve traffic from fresh containers. Preserve evidence: inspect, logs, diff, process binaries from `/proc/PID/exe`, sockets, export/commit, hashes, moved off-host. Investigate the entry point and whether the host was reached. Then patch, rotate every credential the container could access, rebuild from source, redeploy by digest, hunt fleet-wide, and write a post-mortem with new detections.

2. **Q: How do you detect attacks inside running containers?**
   A: Runtime security tools like Falco or Tetragon watch syscalls via eBPF and alert on shells, new executables (drift), sensitive file reads, privilege escalation and suspicious connections. Add auditd on the Docker socket and config, ship Docker events centrally, and watch metrics for anomalies like CPU spikes or unusual egress.

3. **Q: Why not just delete the compromised container immediately?**
   A: Deleting it destroys the writable layer, process memory and in-container evidence you need to find the entry point and scope. Contain it first (network disconnect, pause), collect evidence, then remove it and rebuild.
