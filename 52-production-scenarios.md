# 52 · Production Scenarios (Break-Fix Labs)

> **Level:** ★★★ advanced
>
> **You will learn:** to apply everything under realistic incident conditions. Fourteen scenarios you can **break on purpose** in your lab and then **diagnose and fix**: full disk, OOM loop, wrong port binding, DNS failure, permission denied on a volume, zombie PID 1, subnet clash, slow deploys, lost data, exposed database, firewall reload, wrong CPU architecture, CPU throttling, and a config change that "doesn't apply". Each one has the setup, the symptom as a user would report it, the investigation, the root cause, the fix, and how to prevent it.
>
> **You need:** all previous topics, especially 50 (the method) and 51 (tools).

---

## What it is

A **break-fix lab** is a practice incident: you create a realistic failure, then solve it as if you'd been paged. This is how SREs build intuition, and it's what practical DevOps interviews test ("here's a broken container, fix it").

How to use this topic:
1. Run the **Setup** commands. Don't read further.
2. Read only the **Symptom** (what the "user" reports).
3. Investigate using topic 50's method. Write down your hypothesis.
4. Then read **Investigate → Root cause → Fix → Prevent** and compare.

---

## Why it exists

Reading about mechanisms is not the same as recognising them at 3 AM. Practising each failure once makes the real one familiar, and you'll know which command to type first.

---

## Analogy

**Fire drills.** Nobody learns to evacuate a building from a manual. You practise on a calm day so that on the real day your hands know what to do.

---

## How it works inside: the incident loop

```
Symptom → Triage (60-second commands) → Hypothesis → Test it (one command) → Root cause → Mitigate (restore service) → Fix → Prevent (monitoring/config/process) → Write it down
```

- **Mitigate first, then fix.** Restoring service (truncate a log, raise a limit, roll back) comes before the perfect fix.
- **Keep evidence** (topic 50's `collect.sh`) before destructive steps.
- **Prevent** = the step that turns an incident into a lesson: an alert, a default in `daemon.json`, a CI check.

---

## Commands / files: the scenarios

### Scenario 1: "The site is down, and deploys fail with 'no space left on device'"

**Setup**
```bash
docker run -d --name chatty alpine sh -c 'while :; do echo "$(date) INFO request handled id=$RANDOM user=someone path=/api/v1/things status=200"; done'
sleep 60
```

**Symptom:** pulls/builds fail with `no space left on device`. (In the lab the disk may not fill completely. Watch the log file grow instead.)

**Investigate**
```bash
df -h /var/lib/docker; df -i /var/lib/docker
sudo du -xh --max-depth=1 /var/lib/docker | sort -h | tail -5
sudo ls -lh $(docker inspect -f '{{.LogPath}}' chatty)
docker inspect chatty -f '{{.HostConfig.LogConfig}}'
```

**Root cause:** `json-file` logging with no rotation (`map[]`), and a chatty app.

**Fix**
```bash
sudo truncate -s 0 $(docker inspect -f '{{.LogPath}}' chatty)          # mitigate now
# permanent: daemon.json log defaults, then recreate the container
docker rm -f chatty
docker run -d --name chatty --log-opt max-size=10m --log-opt max-file=3 alpine sh -c 'while :; do echo log; done'
```

**Prevent:** `log-driver: local` (or json-file with `max-size`/`max-file`) in `daemon.json` on every host; disk usage alerts (topics 33, 34, 47); a separate Docker disk (topic 48).

---

### Scenario 2: "The API restarts every few minutes"

**Setup**
```bash
docker run -d --name api2 --restart unless-stopped --memory 128m python:3.13-slim python -c "
import time
cache = []
while True:
    cache.append(bytearray(5*1024*1024))   # 'cache' that never evicts
    time.sleep(1)"
sleep 60
```

**Symptom:** intermittent 502s. Dashboards show the service "up".

**Investigate**
```bash
docker ps --filter name=api2 --format '{{.Status}}'
docker inspect api2 -f 'Restarts={{.RestartCount}} Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Limit={{.HostConfig.Memory}}'
sudo dmesg -T | grep -i 'memory cgroup out of memory' | tail -2
docker stats --no-stream api2
```

**Root cause:** an unbounded in-memory cache. Every ~25 s it hits the 128 MB limit → OOM kill (137) → restart policy brings it back. "Up X seconds" hides the loop.

**Fix:** bound the cache (LRU = least recently used: when full, evict the entry unused for longest; plus a max size) or move it to Redis. If the real working set is legitimately bigger, raise the limit **and** size runtime heaps below it.

**Prevent:** alerts on `container_oom_events_total` and on restart counts (topic 47); load tests with production limits.

---

### Scenario 3: "Container is running, but the port refuses connections"

**Setup**
```bash
docker run -d --name web3 -p 8093:5000 python:3.13-slim sh -c 'pip -q install flask >/dev/null && python -c "
from flask import Flask
app=Flask(__name__)
@app.get(\"/\")
def i(): return \"hi\"
app.run()"'
sleep 15
```

**Symptom:** `curl localhost:8093` → `Connection reset by peer` / `Empty reply`.

**Investigate**
```bash
docker logs web3 | tail -3
PID=$(docker inspect -f '{{.State.Pid}}' web3); sudo nsenter -t $PID -n ss -lntp
docker port web3
```

**Root cause:** Flask's default `app.run()` listens on **`127.0.0.1:5000`** (the logs say `Running on http://127.0.0.1:5000`). Only the container's loopback can reach it. Published traffic arrives on `eth0`.

**Fix:** `app.run(host="0.0.0.0")` (or `flask run --host 0.0.0.0`), and recreate.

**Prevent:** a standard: apps bind `0.0.0.0` and read the port from config; a health check that hits the real interface.

---

### Scenario 4: "Container can't reach any external API, but the host can"

**Setup**
```bash
docker network create dnsfail
docker run -d --name dns4 --network dnsfail --dns 10.255.255.1 alpine sh -c 'while :; do wget -q -T 3 -O /dev/null https://example.com && echo ok || echo FAIL; sleep 5; done'
sleep 20
```

**Symptom:** logs show `FAIL`. Internal calls between containers work.

**Investigate**
```bash
docker logs --tail 3 dns4
docker exec dns4 cat /etc/resolv.conf
docker exec dns4 nslookup example.com
docker exec dns4 ping -c1 -W2 1.1.1.1 >/dev/null && echo "IP connectivity works"
```

**Root cause:** the resolv.conf comment `ExtServers: [10.255.255.1]` shows the upstream DNS server is unreachable. IP connectivity is fine, only name resolution fails.

**Fix:** set reachable DNS servers (`--dns`, or `"dns"` in `daemon.json` for all containers). In real life the usual causes are a corporate firewall blocking Docker's public-DNS fallback, or a systemd-resolved/VPN resolver that containers can't reach (topic 29).

**Prevent:** `dns` set explicitly in `daemon.json` on corporate hosts; synthetic checks from inside a container.

---

### Scenario 5: "After moving to the new image, the app crashes with permission denied on its data folder"

**Setup**
```bash
sudo mkdir -p /srv/app5 && sudo chown 1000:1000 /srv/app5
docker run -d --name app5 -u 10001 -v /srv/app5:/data alpine sh -c 'echo hello > /data/state.txt && sleep 3600'
sleep 2
```

**Symptom:** container exits. Logs: `sh: can't create /data/state.txt: Permission denied`.

**Investigate**
```bash
docker logs app5
docker inspect app5 -f 'User={{.Config.User}}'
ls -ldn /srv/app5
```

**Root cause:** the new image runs as UID **10001** (it was root before), and the host folder is owned by **1000** with mode `755`.

**Fix:** `sudo chown -R 10001:10001 /srv/app5` (matching the image's documented UID), then `docker start app5`. On SELinux hosts, also check `:Z` (topic 25).

**Prevent:** document image UIDs; use named volumes (pre-filled with correct ownership; topic 24); test upgrades with real data.

---

### Scenario 6: "The service stops being able to start subprocesses after a day"

**Setup**
```bash
cat > zomb6.py <<'EOF'
import subprocess, time
while True:
    subprocess.Popen(["sh", "-c", "sleep 0.1 &"])   # grandchildren get orphaned to PID 1
    time.sleep(0.05)
EOF
docker run -d --name z6 --pids-limit 100 -v "$PWD/zomb6.py:/z.py" python:3.13-slim python /z.py
sleep 20
```

**Symptom:** logs start showing `BlockingIOError: [Errno 11] Resource temporarily unavailable`.

**Investigate**
```bash
docker logs --tail 3 z6
docker top z6 -o pid,stat,args | awk '$2 ~ /Z/' | wc -l
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' z6).scope/pids.current
```

**Root cause:** orphaned processes are re-parented to PID 1 (python), which never reaps them → **zombies** pile up and fill `pids.max` (topic 12). (`Popen` objects that are never waited on add zombies too.)

**Fix:** run with `--init` (tini reaps orphans), and fix the code to `wait()` for its children.

**Prevent:** `init: true` by default in Compose; PID limits plus an alert on `pids.current` near `pids.max`.

---

### Scenario 7: "Since this morning, the server can't reach the office network / VPN"

**Setup** (simulation: pretend your office uses `172.30.0.0/16`)
```bash
docker network create --subnet 172.30.0.0/16 projectx_default
ip route | grep 172.30
```

**Symptom:** SSH from the office to the server hangs. The server can't reach office hosts in `172.30.x.x`.

**Investigate**
```bash
ip route get 172.30.5.10
docker network ls; docker network inspect projectx_default -f '{{json .IPAM.Config}}'
```

```
172.30.5.10 dev br-1a2b3c4d5e6f src 172.30.0.1 uid 1000
```

**Root cause:** a new Docker network took a subnet that overlaps the office range. The kernel now routes office traffic into the Docker bridge.

**Fix:** `docker network rm projectx_default` (after stopping its containers), set `default-address-pools` and `bip` in `daemon.json` to unused ranges (topic 27/34), restart Docker, recreate the networks.

**Prevent:** set address pools on day one for every host; document reserved ranges.

---

### Scenario 8: "Every deploy causes 10 seconds of errors"

**Setup**
```bash
docker run -d --name slowstop python:3.13-slim sh -c 'python -m http.server 8000'
time docker stop slowstop
docker inspect slowstop -f '{{.State.ExitCode}}'
```

**Symptom:** each deploy takes 10 s longer than needed and logs exit 137.

**Investigate**
```bash
docker start slowstop; docker top slowstop -o pid,args
```

**Root cause:** shell-form start (`sh -c`): `sh` is PID 1 and ignores SIGTERM. After 10 s, SIGKILL hits the app in the middle of requests (topic 12).

**Fix:** exec form (`["python","-m","http.server","8000"]`), or `exec` in the script, or `--init`. Plus graceful drain in the app (topic 45).

**Prevent:** `docker build --check` (`JSONArgsRecommended`), reviews, stop-time metrics.

---

### Scenario 9: "We upgraded Redis and all the data is gone"

**Setup**
```bash
docker run -d --name r9 redis:7-alpine
docker exec r9 redis-cli SET important yes; docker exec r9 redis-cli SAVE
docker rm -f r9
docker run -d --name r9 redis:7.4-alpine
docker exec r9 redis-cli GET important
```

**Symptom:** `(nil)`.

**Investigate**
```bash
docker inspect r9 -f '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{end}}'
docker volume ls -f dangling=true
```

**Root cause:** the image declares `VOLUME /data`. No named volume was mounted, so each container got a **new anonymous volume**. The old data is in the dangling volume (topic 24).

**Fix (recovery):**
```bash
OLD=$(docker volume ls -qf dangling=true | head -1)   # check its contents first!
docker rm -f r9
docker run -d --name r9 -v $OLD:/data redis:7.4-alpine
docker exec r9 redis-cli GET important
```
Then migrate to a **named** volume.

**Prevent:** named volumes for every stateful service; never `volume prune` blindly (topic 13).

---

### Scenario 10: "Security found our database open to the internet, but ufw blocks it!"

**Setup** (on a VM with ufw)
```bash
sudo ufw default deny incoming; sudo ufw allow 22/tcp; sudo ufw --force enable
docker run -d --name pg10 -p 5432:5432 -e POSTGRES_PASSWORD=pw postgres:17-alpine
```

**Symptom:** an external scanner reaches port 5432.

**Investigate**
```bash
docker port pg10
sudo iptables -t nat -S DOCKER | grep 5432
sudo ufw status
```

**Root cause:** published ports are DNATed and pass through the FORWARD chain. ufw's INPUT rules never see them (topic 28).

**Fix:** don't publish the database at all (apps reach it on the internal network), or `-p 127.0.0.1:5432:5432`. Plus `DOCKER-USER` rules / cloud security groups as a second layer. **Rotate the database password** (assume it was exposed).

**Prevent:** a review rule "no `0.0.0.0` publishing of data stores"; an external port scan in CI or monitoring.

---

### Scenario 11: "After the security team reloaded the firewall, all containers lost connectivity"

**Setup** (lab VM only)
```bash
docker run -d --name w11 -p 8111:80 nginx:1.27-alpine
curl -sI localhost:8111 | head -1
sudo iptables -t nat -F        # simulate a firewall tool flushing NAT rules
curl -sI -m 3 localhost:8111 | head -1; docker run --rm alpine wget -q -T 3 -O /dev/null https://example.com || echo "egress broken"
```

**Symptom:** published ports and outbound traffic broken. Containers show as running.

**Investigate**
```bash
sudo iptables -t nat -S | grep -c DOCKER
sudo iptables -t nat -S POSTROUTING
```

**Root cause:** Docker's NAT chains (DNAT for ports, MASQUERADE for egress) were flushed (topic 26/28). firewalld reloads, `iptables-restore` of a saved ruleset, or ufw reloads can do this.

**Fix:** `sudo systemctl restart docker` (it recreates all its rules; with live-restore, containers keep running).

**Prevent:** integrate firewall tooling with Docker (firewalld's docker zone, put custom rules in `DOCKER-USER`, never flush the nat table); post-change checks.

---

### Scenario 12: "The image works on my Mac, but on the server: exec format error"

**Setup**
```bash
docker buildx build --platform linux/arm64 -t archdemo:1 --load - <<'EOF'
FROM alpine:3.22
CMD ["uname", "-m"]
EOF
docker run --rm archdemo:1
```

(If binfmt/QEMU is installed (topic 21), this may actually run through emulation and print `aarch64`. On a server **without** emulation, you get the error below.)

**Symptom:** `exec /bin/uname: exec format error`.

**Investigate**
```bash
docker image inspect archdemo:1 -f '{{.Os}}/{{.Architecture}}'
uname -m
```

**Root cause:** the image was built for **arm64** (Apple Silicon default), and the server is **amd64**.

**Fix:** build multi-platform (`--platform linux/amd64,linux/arm64`) in CI and push an index (topic 21).

**Prevent:** CI builds the images (not laptops); multi-arch builds; deploy by digest.

---

### Scenario 13: "p99 latency is terrible, but CPU usage is only 40%"

**Setup**
```bash
docker run -d --name t13 --cpus 0.5 python:3.13-slim python -c "
import threading, time
def burn():
    while True: sum(range(10000))
for _ in range(4): threading.Thread(target=burn, daemon=True).start()
while True:
    t=time.time(); time.sleep(0.01); print('tick latency ms', round((time.time()-t)*1000), flush=True); time.sleep(0.5)"
sleep 15
```

**Symptom:** `tick latency` spikes far above 10 ms. `docker stats` shows ~50% CPU (below 100%, so it "looks fine").

**Investigate**
```bash
docker logs --tail 5 t13
CG=/sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' t13).scope
cat $CG/cpu.max; grep -E 'nr_periods|nr_throttled|throttled_usec' $CG/cpu.stat
```

**Root cause:** CFS quota throttling. Busy threads use up the 50 ms-per-100 ms quota early in each period, then **everything** in the container (including the latency-sensitive thread) pauses for the rest of the period (topic 31).

**Fix:** raise `--cpus`, reduce busy threads to match the quota, or remove the hard limit and use shares/requests.

**Prevent:** alert on the throttling ratio (topic 47); size thread pools from `cpu.max`.

---

### Scenario 14: "I changed the config file on the host, restarted nothing, and the app doesn't see the change"

**Setup**
```bash
echo "feature_x=false" > ~/app14.conf
docker run -d --name c14 -v ~/app14.conf:/etc/app.conf alpine sh -c 'while :; do cat /etc/app.conf; sleep 2; done'
sed -i 's/false/true/' ~/app14.conf
sleep 5; docker logs --tail 2 c14
```

**Symptom:** the logs keep printing `feature_x=false`.

**Investigate**
```bash
ls -i ~/app14.conf; docker exec c14 ls -i /etc/app.conf
```

**Root cause:** single-file bind mount + `sed -i` replaced the file (new inode). The container still has the old inode (topic 25).

**Fix:** `docker restart c14` (picks up the new inode). Long-term: mount the **directory**, and make the app reload on a signal or a file watch.

**Prevent:** mount config directories, not single files; config changes = a new deploy.

---

### Clean up all scenarios

```bash
docker rm -f chatty api2 web3 dns4 app5 z6 slowstop r9 pg10 w11 t13 c14 2>/dev/null
docker network rm dnsfail projectx_default 2>/dev/null
docker volume prune -f
sudo rm -rf /srv/app5; rm -f zomb6.py ~/app14.conf
```

---

## Hands-on lab

1. Do all 14 scenarios in order, timing yourself from "Symptom" to "Root cause". Target: under 5 minutes each.
2. For each one, write a **one-paragraph incident note**: impact, root cause, mitigation, prevention. (This is how real post-mortems are written.)
3. Pair up with someone (or yourself a week later): one person sets up a scenario **without** saying which; the other diagnoses it.
4. Invent two new scenarios from earlier topics (ideas: MTU black hole (topic 30), healthcheck missing curl (topic 32), `.env` not reaching the container (topic 36), `docker` group abuse (topic 40), `live-restore` buffer blocking (topic 49)).

---

## Where you'll use it

- **On-call readiness**, and **practical interviews** ("debug this broken Docker setup").
- **Team training:** game days / chaos drills.
- **Runbooks:** each scenario's Investigate/Fix/Prevent becomes a runbook entry.

---

## When it breaks (meta: when your diagnosis goes wrong)

| Trap | Avoid by |
|---|---|
| Fixing the symptom only (truncating logs every week) | Always do the Prevent step |
| Changing several things at once | One hypothesis, one change, observe |
| Destroying evidence (`rm`, restarts) | Collect first (topic 50) |
| Trusting "Up" status | Check RestartCount, health, `memory.events` |
| Assuming the network when it's the app (or the reverse) | Classify the error first (refused/timeout/DNS) |

---

## Quick recap

- Practise failures on purpose: setup → symptom → investigate → root cause → mitigate → fix → prevent.
- The 14 classics: log-filled disk, OOM restart loop, `127.0.0.1` binding, upstream DNS, UID mismatch on volumes, zombie PID 1, subnet overlap, SIGTERM ignored, anonymous-volume data "loss", published DB bypassing ufw, flushed iptables rules, arm64 vs amd64, CPU throttling, stale single-file mounts.
- Mitigate first, keep evidence, change one thing at a time, and always finish with prevention (defaults, alerts, CI checks).

**Interview questions**

1. **Q: A container shows "Up 30 seconds" every time you look, and users see intermittent errors. What do you suspect?**
   A: A restart loop hidden by a restart policy. Check `RestartCount`, `ExitCode`, `OOMKilled` and `docker events` for die/oom events. A commonly found cause is OOM kills from a memory leak or an undersized limit.

2. **Q: Users can't reach a containerised app on its published port, but the container is healthy inside. Your first two checks?**
   A: Which address and port the app listens on inside its network namespace (`nsenter -t PID -n ss -lntp`; 127.0.0.1 is the classic mistake), and the port mapping with `docker port` (correct container port, and bound on the right host IP).

3. **Q: After a new Compose project was deployed, the server lost connectivity to part of the network. Why, and how do you prevent it?**
   A: Docker allocated a bridge subnet that overlaps a real network range, so the host routes that traffic into the bridge. Remove or recreate the network, and configure `default-address-pools`/`bip` in daemon.json with reserved non-overlapping ranges.
