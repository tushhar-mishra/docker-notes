# 28 · Publishing Ports

> **Level:** ★★★ advanced
>
> **You will learn:** every form of `-p`, what Docker creates underneath (DNAT rules + the `docker-proxy` process), how to bind to `127.0.0.1` only, why **published ports bypass ufw and firewalld** (a classic security incident) and how to control them with the `DOCKER-USER` chain, Docker's newer nftables backend, `EXPOSE` vs `-p`, and how containers reach services on the host (`host.docker.internal`).
>
> **You need:** topic 26 (DNAT, conntrack, chains), topic 27 (bridge networks).

---

## What it is

**Publishing** a port makes a container port reachable through a **port on the host**:

```bash
docker run -d -p 8080:80 nginx
#              │    └─ container port (where the app listens inside)
#              └─ host port (what clients connect to)
```

Clients connect to `host-ip:8080`; Docker forwards the traffic to `container-ip:80`.

| Form | Meaning |
|---|---|
| `-p 8080:80` | Host port 8080 on **all** host IPs (IPv4 `0.0.0.0` and IPv6 `::`) → container 80/tcp |
| `-p 127.0.0.1:8080:80` | Only on the host's **loopback**: reachable from the host itself, not from the network |
| `-p 10.0.0.5:8080:80` | Only on one specific host IP |
| `-p 8080:80/udp` | UDP instead of TCP |
| `-p 53:53/tcp -p 53:53/udp` | Both protocols need separate flags |
| `-p 80` | Container port 80 → a **random** free host port (from 32768–60999) |
| `-p 8000-8010:8000-8010` | A range |
| `-P` | Publish **every** `EXPOSE`d port to random host ports |

`--mount`-style long syntax: `--publish published=8080,target=80,protocol=tcp,host_ip=127.0.0.1`.

---

## Why it exists

Containers on a bridge network have **private IPs** (`172.17.x.x`) that only the host knows how to reach (topic 26). Clients on other machines can't connect to them. Publishing creates the path **from the host's real IP** into the container.

---

## Analogy

A **hotel reception desk** (the host). Guests' rooms (containers) aren't reachable from the street. Reception has a rule: "anyone asking at **door 8080** is sent to **room 3, door 80**". Binding to `127.0.0.1` means the rule only applies to people **already inside the building**. The firewall bypass problem: the hotel's security guard checks people at the **front entrance** (INPUT chain), but the receptionist sends door-8080 visitors down a **side corridor** (FORWARD chain) that the guard never watches.

---

## How it works inside

When you run `docker run -d --name web -p 8080:80 nginx`, dockerd sets up three things:

### 1. A DNAT rule (the main path)

In the `nat` table, a chain called `DOCKER` gets:

```
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

- "TCP packets to port 8080, **not** coming from `docker0`, get their destination rewritten to `172.17.0.2:80`."
- The `DOCKER` chain is jumped to from `PREROUTING` (traffic from other machines) and `OUTPUT` (traffic from the host itself to its own IPs).

### 2. A filter rule allowing the forwarded traffic

```
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
```

- After DNAT, the packet isn't for the host anymore. It's **forwarded** to `docker0`, so it passes through the `FORWARD` chain, where this rule lets it through.
- Docker also sets the `FORWARD` chain's default policy to `DROP`, and since Docker 28 it explicitly **drops traffic to container ports that weren't published**, even from the local LAN.

(The exact helper chain names, like `DOCKER-FORWARD`, `DOCKER-CT`, `DOCKER-BRIDGE` and `DOCKER-ISOLATION-*`, differ between Docker versions. `DOCKER` and `DOCKER-USER` are the stable ones to know.)

### 3. `docker-proxy` (the userland proxy)

```
root  48211  1290  /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 80
```

- A small process per published port (per IP family) that **listens on the host port** and copies bytes to the container.
- Why, if DNAT already works? Some paths can't be DNATed easily: connections from the host to `127.0.0.1:8080` (loopback traffic doesn't go through `PREROUTING`), and "hairpin" connections (a container calling the host's published port and coming back to itself). The proxy handles those. It also **reserves the port**, so nothing else on the host can take it.
- Downside: connections through the proxy show the **proxy's** IP as the client IP, not the real client.
- `"userland-proxy": false` in `daemon.json` disables it. Docker then relies on extra NAT rules (and `route_localnet`) for loopback. Faster and fewer processes, but it changes some edge-case behaviour.

### Why ufw/firewalld don't block published ports

**ufw** (Uncomplicated Firewall, Ubuntu's default) and **firewalld** (RHEL's) mostly filter traffic **to the host** in the `INPUT` chain.

```
packet to host:8080
  │
  PREROUTING ── DNAT to 172.17.0.2:80 ──► now it's "forwarded" traffic
  │
  FORWARD chain  ◄── Docker's ACCEPT rule is here
  │                  ufw's "deny 8080" rule is in INPUT → never consulted
  ▼
container
```

So `ufw deny 8080` does **nothing** for a published container port, and your "internal" database published with `-p 5432:5432` is **open to the internet**. This is a real, frequent incident (exposed Redis/Elasticsearch/MongoDB instances are found by internet scanners within hours).

**Fixes:**
1. **Don't publish what doesn't need to be public.** Containers on the same network talk directly, with no `-p` needed.
2. **Bind to loopback**: `-p 127.0.0.1:5432:5432` (or a private interface IP), and put a reverse proxy in front for public services.
3. **Use the `DOCKER-USER` chain**: Docker creates it, jumps to it **before** its own rules, and **never touches your rules in it**. Example: only allow the office network to reach published ports from outside:
   ```bash
   sudo iptables -I DOCKER-USER -i eth0 ! -s 203.0.113.0/24 -j DROP
   ```
   (Make the rules persistent with your distribution's iptables-persistent / service, because they're lost on reboot.)
4. **Cloud security groups / external firewalls** in front of the host (they filter before the packet ever reaches the host).
5. **firewalld**: Docker adds its bridges to a `docker` zone. Use firewalld **policies** or direct rules, or bind to specific IPs.

### The nftables backend (Engine 29)

- Since Docker 29 you can set `"firewall-backend": "nftables"` in `daemon.json` (still **experimental**). Docker then writes **native nftables** rules into its own tables (`ip docker-bridges`, `ip6 docker-bridges`) instead of iptables chains.
- With nftables there's **no `DOCKER-USER` chain**. You create your **own nftables table and base chain** with a priority that runs **before** Docker's (nftables lets several tables hook the same point, ordered by priority).
- You'll still see the iptables backend on most production hosts for now.

### `EXPOSE` vs `-p`

- `EXPOSE 80` in a Dockerfile = **documentation** in the image config. It publishes nothing.
- It's used by `-P` (publish all exposed ports to random host ports) and by tools that read metadata.
- Containers on the same network can reach **any** port the app listens on, exposed or not.

### Reaching the host from a container

Containers often need to call something running **on the host** (a database installed directly on the host, a dev server):

```bash
docker run --add-host=host.docker.internal:host-gateway myapp
```

- `host-gateway` is a special value: Docker replaces it with the host's IP as seen from the container (the bridge gateway, e.g. `172.17.0.1`). It adds `host.docker.internal` to the container's `/etc/hosts`.
- **Docker Desktop** provides `host.docker.internal` automatically.
- The host service must listen on an address reachable from the bridge (not only `127.0.0.1`), and the host firewall must allow it.

---

## Commands / files

### Publish and check

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
docker port web
curl -sI localhost:8080 | head -1
```

```
80/tcp -> 0.0.0.0:8080
80/tcp -> [::]:8080
HTTP/1.1 200 OK
```

- Two entries: IPv4 and IPv6.

### See the rules Docker created

```bash
sudo iptables -t nat -S DOCKER
sudo iptables -S DOCKER
sudo iptables -t nat -S PREROUTING
sudo iptables -t nat -S OUTPUT
```

```
-N DOCKER
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
-N DOCKER
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
...
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
```

- `--dst-type LOCAL` = "destined to any of this host's own IPs". So any host IP on port 8080 is DNATed.
- `OUTPUT ! -d 127.0.0.0/8` = host-originated traffic to the host's **non-loopback** IPs is DNATed. Traffic to `127.0.0.1` is left to `docker-proxy`.

Watch the counters go up as you send requests:

```bash
sudo iptables -t nat -L DOCKER -n -v
```

```
Chain DOCKER (2 references)
 pkts bytes target     prot opt in      out     source      destination
    0     0 RETURN     0    --  docker0 *       0.0.0.0/0   0.0.0.0/0
    3   180 DNAT       6    --  !docker0 *      0.0.0.0/0   0.0.0.0/0   tcp dpt:8080 to:172.17.0.2:80
```

- `pkts 3` = three connections were DNATed (only the first packet of each connection hits the nat table; conntrack handles the rest).

### See the proxy

```bash
ps -ef | grep docker-proxy | grep -v grep
sudo ss -lntp | grep 8080
```

```
root  48211  1290  0 13:10 ?  00:00:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 80 -use-listen-fd
LISTEN 0  4096  0.0.0.0:8080  0.0.0.0:*  users:(("docker-proxy",pid=48211,fd=4))
LISTEN 0  4096     [::]:8080     [::]:*  users:(("docker-proxy",pid=48230,fd=4))
```

- `ss` shows `docker-proxy` as the listener on 8080. Remember this when you look for "what's using port 8080?" on a Docker host.

### Bind to localhost only

```bash
docker run -d --name db -p 127.0.0.1:5432:5432 -e POSTGRES_PASSWORD=pw postgres:17-alpine
docker port db
sudo ss -lntp | grep 5432
```

```
5432/tcp -> 127.0.0.1:5432
LISTEN 0  4096  127.0.0.1:5432  0.0.0.0:*  users:(("docker-proxy",...))
```

- From another machine: connection refused. From the host: works.

### Random ports and `-P`

```bash
docker run -d --name r -P nginx:1.27-alpine
docker port r
```

```
80/tcp -> 0.0.0.0:32768
80/tcp -> [::]:32768
```

### Prove the ufw bypass (on a test VM with ufw)

```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status
# from ANOTHER machine:
curl -sI http://<host-ip>:8080 | head -1
```

```
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
HTTP/1.1 200 OK
```

- ufw says only 22 is allowed, but 8080 answers. Then fix it with `DOCKER-USER`:

```bash
sudo iptables -I DOCKER-USER -i eth0 -p tcp -m conntrack --ctorigdstport 8080 -j DROP
sudo iptables -S DOCKER-USER
```

```
-N DOCKER-USER
-A DOCKER-USER -i eth0 -p tcp -m conntrack --ctorigdstport 8080 -j DROP
-A DOCKER-USER -j RETURN
```

- `--ctorigdstport 8080` matches the **original** destination port (before DNAT changed it to 80). In `DOCKER-USER` the packet has **already** been DNATed, so `--dport` would be `80`, not `8080`.

### host.docker.internal

```bash
python3 -m http.server 9999 --bind 0.0.0.0 >/dev/null 2>&1 &
docker run --rm --add-host=host.docker.internal:host-gateway alpine sh -c 'grep host.docker /etc/hosts; wget -qO- host.docker.internal:9999 | head -2'
kill %1
```

```
172.17.0.1      host.docker.internal
<!DOCTYPE HTML>
<html lang="en">
```

---

## Hands-on lab

1. Publish nginx three ways (`-p 8080:80`, `-p 127.0.0.1:8081:80`, `-P`). From your **Windows browser** (WSL forwards localhost), open each one. Then run `ip -br addr show eth0` in WSL, and from Windows try the WSL IP instead of `localhost`: the `127.0.0.1` one fails.
2. Read the DNAT rule for each one in `iptables -t nat -S DOCKER`, and the listeners with `ss -lntp`.
3. Send 5 requests and watch the DNAT packet counter rise (`iptables -t nat -L DOCKER -n -v`).
4. **Port conflict:** run a second container with `-p 8080:80`:
   ```
   docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint ...: Bind for 0.0.0.0:8080 failed: port is already allocated
   ```
5. If you have a Linux VM with ufw, reproduce the bypass and fix it with `DOCKER-USER` as shown above.
6. Clean up: `docker rm -f web db r`

---

## Where you'll use it

- **Every service that must be reached from outside the host**: usually only a reverse proxy/load balancer (ports 80/443) is published, and everything else talks on internal networks.
- **Security reviews:** `docker ps` / `ss -lntp` to find unintended `0.0.0.0` bindings of databases and admin UIs.
- **Firewall design** on Docker hosts: `DOCKER-USER` rules, cloud security groups, binding to private IPs.
- **Development:** `-p 127.0.0.1:...` for local databases. `host.docker.internal` to reach host services.
- **Kubernetes mapping:** `-p` ≈ a Service of type NodePort/LoadBalancer. Kube-proxy uses the same DNAT idea (topic 53).

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Another container or `docker-proxy` already has that host port | `docker ps --filter publish=8080`; pick another port |
| `listen tcp4 0.0.0.0:80: bind: address already in use` | A **host** process (nginx, apache) has the port | `sudo ss -lntp \| grep :80` |
| `curl: (7) Failed to connect ... Connection refused` on the published port | The app inside listens on `127.0.0.1` only, or on another port | `nsenter -t PID -n ss -lntp` (topic 11); make the app listen on `0.0.0.0` |
| `curl: (56) Recv failure: Connection reset by peer` | docker-proxy accepted, but nothing listens in the container (app still starting, crashed, or wrong container port) | Check app logs; check the container port in `-p` |
| Works on the host, not from other machines | Bound to `127.0.0.1`, or a cloud security group/host firewall blocks it | `docker port`; check security groups and `DOCKER-USER` |
| Port reachable from the internet despite ufw/firewalld | The documented bypass | Bind to `127.0.0.1`/private IP, `DOCKER-USER` rules, external firewall |
| All published ports stopped working after `firewall-cmd --reload` / `iptables -F` / `ufw reload` | Docker's chains were flushed | `sudo systemctl restart docker` to recreate them (with `live-restore`, containers keep running) |
| App logs show every client IP as `172.17.0.1` | Traffic came through docker-proxy or a NAT hairpin | Use a reverse proxy passing `X-Forwarded-For`, `--network host` for the edge proxy, or disable the userland proxy |

---

## Quick recap

- `-p [host-ip:]host-port:container-port[/proto]`. The default binds **all** host IPs (v4 and v6). `-P` publishes all EXPOSEd ports to random ports. `EXPOSE` alone publishes nothing.
- Underneath: a **DNAT** rule in the `DOCKER` nat chain, an ACCEPT rule for the forwarded traffic, and a **`docker-proxy`** process for loopback/hairpin traffic.
- Published traffic goes through **FORWARD**, not INPUT, so **ufw/firewalld don't filter it**. Bind to `127.0.0.1` or private IPs, use **`DOCKER-USER`** (match with `--ctorigdstport`), or filter upstream.
- Engine 29 offers an experimental **nftables** backend (no DOCKER-USER; use your own higher-priority table).
- `--add-host=host.docker.internal:host-gateway` lets containers reach the host.

**Interview questions**

1. **Q: How does `-p 8080:80` work under the hood?**
   A: Docker adds an iptables DNAT rule that rewrites packets to any host IP on port 8080 to the container's IP on port 80, plus a FORWARD accept rule. A docker-proxy process also listens on 8080 for loopback and hairpin connections. conntrack handles the return traffic.

2. **Q: You enabled ufw and denied a port, but a container's published port is still reachable. Why?**
   A: Docker's DNAT happens in PREROUTING, so the traffic is forwarded to the container through the FORWARD chain and never hits ufw's INPUT rules. Fix it by binding to 127.0.0.1 or a private IP, adding rules to the DOCKER-USER chain, or using an external firewall.

3. **Q: What does EXPOSE do?**
   A: It only documents which ports the app uses, in the image metadata. It doesn't publish anything. Publishing needs `-p` (or `-P`, which publishes all exposed ports to random host ports).
