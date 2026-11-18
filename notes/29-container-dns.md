# 29 · Container DNS and Service Discovery

> **Level:** ★★ intermediate
>
> **You will learn:** how containers find each other by name through Docker's **embedded DNS server at 127.0.0.11**, how it's wired into each container (a hidden NAT rule inside the container's namespace), how external names are resolved (and why `systemd-resolved` and corporate DNS cause trouble), network aliases and DNS round-robin, `/etc/resolv.conf` and `/etc/hosts` inside containers, and the `--dns`, `--dns-search`, `--dns-option` and `--add-host` options.
>
> **You need:** topic 27 (user-defined networks), topic 26 (NAT inside namespaces). From your Linux notes: DNS basics, `/etc/resolv.conf`, `/etc/hosts`.

---

## What it is

**Service discovery** = how one service finds the address of another. In Docker, it's done with **DNS**: on a user-defined network, a container can connect to `db:5432` and Docker's DNS server answers `db → 172.18.0.3`.

Two different setups:

| Network | `/etc/resolv.conf` in the container | Container names resolvable? |
|---|---|---|
| **User-defined** networks (and Compose) | `nameserver 127.0.0.11` (Docker's **embedded DNS**) | **Yes** |
| **Default bridge** | A **copy** of the host's resolv.conf (localhost entries removed) | No |
| `host` network | The host's resolv.conf | No (it's the host) |

---

## Why it exists

Container IPs change every time a container is recreated. Hard-coding IPs breaks constantly. Names are stable: `db` is always `db`, whatever its current IP. And for outside names (`api.github.com`), containers still need working DNS.

---

## Analogy

A **company phone directory operator**. Inside the office (the user-defined network), you dial "operator" (127.0.0.11) and ask for "accounting". The operator knows every extension in **this** office. For outside numbers, the operator forwards your call to the public directory (the host's DNS servers). On the default bridge there's no operator, just a copy of the public phone book.

---

## How it works inside

### 1. The embedded DNS server

- It isn't a separate container or process. **dockerd itself** answers DNS for containers.
- For each container on a user-defined network, Docker:
  1. writes `nameserver 127.0.0.11` into the container's `/etc/resolv.conf`,
  2. opens a listening UDP and TCP socket **inside the container's network namespace** on a random high port,
  3. adds **NAT rules inside the container's namespace** that redirect `127.0.0.11:53` to that socket.
- A query from the app → `127.0.0.11:53` → NATed inside the namespace → dockerd's socket → dockerd answers.

So `127.0.0.11` exists only inside each container. On the host, nothing listens there.

### 2. What the embedded DNS answers

For a container on network `appnet`, names resolve if they belong to containers **on a network this container shares**:
- the **container name** (`db`),
- **network aliases** (`--network-alias database`),
- in Compose, the **service name** (`db`) and `<container_name>`,
- the container's own hostname.

Anything else → forwarded to **upstream DNS servers** (the external ones).

### 3. Where upstream servers come from

dockerd decides the upstream (external) DNS servers when the container is created:
1. If you set `--dns` (or `"dns"` in `daemon.json`): those.
2. Otherwise: the **host's** `/etc/resolv.conf`.

The common trap: on Ubuntu with **systemd-resolved**, the host's `/etc/resolv.conf` says `nameserver 127.0.0.53`, a local stub resolver that's reachable only on the **host's** loopback, not from inside a container. Docker detects this and reads `/run/systemd/resolve/resolv.conf` instead (the real upstream servers). If it can't find usable servers, it falls back to public DNS (`8.8.8.8`, `8.8.4.4`), which is **blocked in many corporate networks**. The result: containers can't resolve anything, while the host resolves fine.

### 4. Default bridge DNS

- No embedded DNS. Docker copies the host's resolv.conf into the container, removing localhost nameservers. If none are left, it uses Google's public DNS.
- Containers can't resolve each other's names (topic 27).

### 5. `/etc/hosts` and `/etc/hostname`

Docker generates these files per container (in `/var/lib/docker/containers/<id>/`) and bind-mounts them in:

```
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
172.18.0.2  3f2a9c1b7e0d      ← the container's own hostname → its IP
```

- `--add-host name:ip` adds lines here (topic 28: `host.docker.internal:host-gateway`).
- Editing these files inside a running container works, but Docker may overwrite them, and changes are lost on recreate.

### 6. Aliases and DNS round-robin

Several containers can share an alias on the same network:

```bash
docker run -d --network appnet --network-alias search elasticsearch-1 ...
docker run -d --network appnet --network-alias search elasticsearch-2 ...
```

A lookup of `search` then returns **all** their IPs, in rotating order. That's **DNS round-robin**: a primitive form of load balancing. It's not health-aware (a dead container's IP is removed only once the container is stopped), and clients often cache the first answer. Real load balancing needs a proxy (nginx, HAProxy, Traefik) or an orchestrator (topic 53).

### 7. The resolv.conf options: `search` and `ndots`

```
nameserver 127.0.0.11
search corp.example.com
options ndots:0
```

- `search` = domains tried after short names: `db` → also tries `db.corp.example.com`.
- `ndots:N` = if the name has **fewer than N dots**, try the search domains **first**. High `ndots` (Kubernetes uses 5!) causes many extra queries for external names: `api.github.com` is first tried as `api.github.com.corp.example.com`, etc. This is a classic source of DNS latency (topic 53).
- `--dns-search` and `--dns-option` set these per container.

### 8. musl (Alpine) differences

Alpine's musl resolver (topic 19) sends queries to all nameservers **in parallel** and handles `search` slightly differently. Most setups don't notice, but odd DNS behaviour that only happens in Alpine containers is a known pattern.

---

## Commands / files

### See both setups

```bash
docker network create appnet
docker run -d --name db --network appnet alpine sleep 3600
docker run --rm --network appnet alpine cat /etc/resolv.conf
docker run --rm alpine cat /etc/resolv.conf
```

```
# Generated by Docker Engine.
# This file can be edited; Docker Engine will not make further changes once it
# has been modified.

nameserver 127.0.0.11
options ndots:0

# Based on host file: '/run/systemd/resolve/resolv.conf' (internal resolver)
# ExtServers: [192.168.1.1]
# Overrides: []
# Option ndots from: internal
nameserver 192.168.1.1
```

- The first file (user-defined network) points at the embedded DNS. The comments tell you **where Docker got the upstream servers** (`/run/systemd/resolve/resolv.conf`) and **which ones** (`ExtServers: [192.168.1.1]`). Very useful when debugging.
- The second (default bridge) contains the upstream server directly.

### Look up names

```bash
docker run --rm --network appnet alpine nslookup db
docker run --rm --network appnet alpine nslookup example.com
```

```
Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
Name:   db
Address: 172.18.0.2

Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
Name:   example.com
Address: 93.184.215.14
```

- Both went through `127.0.0.11`. The second was forwarded upstream.

Better tools (dig) in the `nicolaka/netshoot` debugging image:

```bash
docker run --rm --network appnet nicolaka/netshoot dig +short db
docker run --rm --network appnet nicolaka/netshoot dig example.com | grep -E 'SERVER|Query time'
```

```
172.18.0.2
;; Query time: 14 msec
;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
```

### See the hidden NAT rule inside a container

```bash
PID=$(docker inspect -f '{{.State.Pid}}' db)
sudo nsenter -t $PID -n iptables -t nat -S
sudo nsenter -t $PID -n ss -lunp
```

```
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER_OUTPUT
-N DOCKER_POSTROUTING
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:39421
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:51874
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 39421 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 51874 -j SNAT --to-source :53
State  Recv-Q Send-Q Local Address:Port   Peer Address:Port Process
UNCONN 0      0         127.0.0.11:51874       0.0.0.0:*     users:(("dockerd",pid=1290,fd=41))
```

- These rules live **in the container's namespace**, not the host's.
- Queries to `127.0.0.11:53` → port `51874` (UDP) / `39421` (TCP).
- `ss` shows **dockerd** (host PID 1290) owns that socket inside the container's network namespace.

### Aliases and round-robin

```bash
docker run -d --name s1 --network appnet --network-alias search alpine sleep 3600
docker run -d --name s2 --network appnet --network-alias search alpine sleep 3600
for i in 1 2 3; do docker run --rm --network appnet nicolaka/netshoot dig +short search | tr '\n' ' '; echo; done
```

```
172.18.0.3 172.18.0.4
172.18.0.4 172.18.0.3
172.18.0.3 172.18.0.4
```

- Both IPs, in changing order.

### Custom DNS settings

```bash
docker run --rm --dns 1.1.1.1 --dns-search corp.example.com --dns-option ndots:2 alpine cat /etc/resolv.conf
docker run --rm --add-host api.internal:10.0.0.42 alpine grep api /etc/hosts
```

```
nameserver 1.1.1.1
search corp.example.com
options ndots:2
10.0.0.42       api.internal
```

- On the default bridge, `--dns` goes straight into resolv.conf. On user-defined networks, the file still says `127.0.0.11`, and your `--dns` servers become the embedded DNS's **upstream** servers.

Daemon-wide defaults in `/etc/docker/daemon.json` (topic 34):

```json
{
  "dns": ["10.0.0.2", "10.0.0.3"],
  "dns-search": ["corp.example.com"]
}
```

---

## Hands-on lab

1. Run everything above. Read the comments Docker writes in resolv.conf.
2. **Break DNS on purpose:** run `docker run --rm --network appnet --dns 10.255.255.1 alpine nslookup example.com`. It times out (`;; connection timed out; no servers could be reached`), but `nslookup db` still works: internal names don't need upstream DNS.
3. **systemd-resolved check** (Ubuntu): `cat /etc/resolv.conf` (shows `127.0.0.53`) vs `cat /run/systemd/resolve/resolv.conf` (real servers). Compare with the `ExtServers` comment in a container.
4. **Name scoping:** create `othernet` with a container `x`. From `appnet`, `nslookup x` fails. Connect `x` to `appnet` too (`docker network connect appnet x`): now it resolves.
5. **ndots effect:** with `--dns-search example.org --dns-option ndots:5`, run `dig +search api.github.com` in netshoot and look at how it tries the search domain first.
6. Clean up: `docker rm -f db s1 s2 x 2>/dev/null; docker network rm appnet othernet`

---

## Where you'll use it

- **All multi-container apps:** connection strings use service names (`postgres://db:5432`, `redis://cache:6379`).
- **Corporate networks:** setting `dns` in `daemon.json` to internal DNS servers so containers resolve internal hostnames and aren't blocked from public DNS.
- **Incident triage:** "can't resolve" is one of the top container network problems (topic 30).
- **Kubernetes:** the same concept with CoreDNS and Service names (`db.default.svc.cluster.local`), plus the `ndots:5` latency issue (topic 53).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `bad address 'db'` / `Name or service not known` | Not on the same user-defined network, or a typo, or the target container isn't running | `docker network inspect`; put them on one network |
| Internal names work, external don't (`connection timed out; no servers could be reached`) | Upstream DNS unreachable from containers: often public DNS fallback blocked by a firewall, or a VPN DNS that containers can't reach | Set `"dns"` in `daemon.json` to reachable servers; check the `ExtServers` comment |
| Host resolves fine, containers don't | Host uses `127.0.0.53` (systemd-resolved) or a VPN's local resolver | Docker reads `/run/systemd/resolve/resolv.conf`. Make sure it has real servers, or set `dns` explicitly |
| Slow name resolution (seconds) | `ndots`/`search` causing many failed lookups, or one dead upstream server tried first | Fewer search domains, lower ndots, use FQDNs (Fully Qualified Domain Names, the complete name like `api.example.com.`) with a trailing dot, which skips the search list; remove dead servers |
| Old IP returned after recreating a service | The client app cached DNS (Java, Nginx `proxy_pass` resolves at startup) | Configure DNS TTL (time-to-live: how long an answer may be cached)/caching in the app; Nginx `resolver 127.0.0.11 valid=10s` + variables |
| Works on Debian image, odd results on Alpine | musl resolver behaviour | Test with a glibc image; simplify search/ndots |
| `/etc/resolv.conf` edits disappear | Docker regenerates it on recreate | Use `--dns*` options or `daemon.json` |

---

## Quick recap

- On user-defined networks, containers use **Docker's embedded DNS at 127.0.0.11**: dockerd answers through a socket and NAT rules **inside each container's namespace**.
- It resolves container names, aliases and Compose service names for containers on **shared networks**. Everything else goes to upstream servers (from `--dns`/`daemon.json`, or the host's resolv.conf, with special handling for systemd-resolved).
- The default bridge just copies the host's resolv.conf: no container names.
- Network aliases shared by several containers give **DNS round-robin** (not health-aware).
- Watch `search`/`ndots` for latency, and set corporate DNS servers in `daemon.json`.

**Interview questions**

1. **Q: How do containers resolve each other's names in Docker?**
   A: On user-defined networks, `/etc/resolv.conf` points to 127.0.0.11, Docker's embedded DNS. dockerd answers queries for container names, aliases and service names on shared networks, and forwards other queries to the configured upstream DNS servers.

2. **Q: Containers can reach IPs but can't resolve external names, while the host resolves fine. What do you check?**
   A: Which upstream servers Docker uses (the comments in the container's resolv.conf). The host may use systemd-resolved's 127.0.0.53 or a VPN resolver that containers can't reach, or Docker fell back to public DNS that's blocked. Set `dns` in daemon.json to reachable servers.

3. **Q: Is a network alias shared by several containers a load balancer?**
   A: Only DNS round-robin: it returns all IPs in rotating order. It isn't health-aware, and clients may cache answers. Use a real proxy or an orchestrator's service load balancing.
