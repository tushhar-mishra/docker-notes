# 30 · Network Troubleshooting

> **Level:** ★★★ advanced
>
> **You will learn:** a step-by-step method to find **where** a container network problem is: in the app, the container's namespace, the bridge, the NAT rules, the host firewall, DNS, or the path outside. You'll learn what "connection refused", "timeout", "no route to host" and "connection reset" each tell you, how to use netshoot, `nsenter`, `tcpdump` on veth interfaces and conntrack, and how to recognise and fix **MTU** problems.
>
> **You need:** topics 26–29 (namespaces, veth, bridges, NAT, publishing, DNS), topic 11 (`nsenter`).

---

## What it is

A **method**, not a single command. Network problems feel random until you check each hop in order. Each hop has one or two commands that prove it works or doesn't.

```
 client ──► host firewall / cloud SG ──► host:port ──► DNAT / docker-proxy ──► FORWARD rules ──► docker0/br-xxx
        ──► veth ──► container eth0 ──► app listening on the right IP:port

 container ──► DNS (127.0.0.11 → upstream) ──► eth0 ──► veth ──► bridge ──► routing ──► MASQUERADE ──► host eth0 ──► internet / other networks
```

---

## Why it exists

"The container can't connect" can be caused by any of a dozen things, from the app listening on the wrong address to an MTU mismatch on a VPN. Guessing wastes hours. A fixed checklist finds it in minutes, and it's the same method you'll later use in Kubernetes.

---

## Analogy

A **plumber** finding a leak doesn't tear up the whole house. They check the water at the main valve, then each junction, moving along the pipe until the water stops. You do the same with packets.

---

## How it works inside: the method

### Step 0: read the error message precisely

| Error | What happened at the network level | Usually means |
|---|---|---|
| **`Connection refused`** | The packet **arrived**, but nothing listens on that IP:port. The kernel replied with a TCP RST | App not running / wrong port / listening on `127.0.0.1` only / docker-proxy with no backend |
| **`Connection timed out`** / hangs | Packets are **dropped** silently somewhere, and no reply comes | Firewall, security group, missing route, wrong IP, MTU (for large packets) |
| **`No route to host`** | The kernel (or a router) knows there's no path; an ICMP "unreachable" came back | Wrong subnet, interface down, firewall REJECT rule, ARP failure |
| **`Connection reset by peer`** | A connection was accepted, then torn down | App crashed or rejected the request, proxy with a dead backend, TLS mismatch |
| **`Name or service not known` / `bad address`** | DNS failed: no IP was found at all | Wrong network, typo, upstream DNS unreachable (topic 29) |
| **`Temporary failure in name resolution`** | No DNS server could be reached | resolv.conf/upstream problem |

This one distinction (refused vs timeout vs DNS) usually halves the search.

### Step 1: is the app running and listening on the right address?

```bash
docker ps --filter name=api
docker logs --tail 50 api
PID=$(docker inspect -f '{{.State.Pid}}' api)
sudo nsenter -t $PID -n ss -lntp
```

- Not running / restarting → it's an app problem, not a network one (topic 50).
- Listening on `127.0.0.1:8000` → **only reachable from inside the container itself**. Fix the app to bind `0.0.0.0` (the #1 cause of "refused" on published ports).
- Listening on a different port than you published → fix `-p` or the app config.

### Step 2: can you reach it from inside its own namespace?

```bash
sudo nsenter -t $PID -n curl -sS -m 3 http://127.0.0.1:8000/health
```

- Fails → the app itself is broken (or slow). Not the network.

### Step 3: from another container on the same network

```bash
docker run --rm --network appnet nicolaka/netshoot curl -sS -m 3 http://api:8000/health
```

- **netshoot** = a debugging image packed with network tools (curl, dig, tcpdump, ss, iperf, mtr, nmap…).
- Works → container networking and DNS are fine. The problem is in publishing or outside.
- DNS error → topic 29.
- Timeout/refused by IP too → check both containers really are on the same network: `docker network inspect appnet`.

### Step 4: through the published port, from the host

```bash
docker port api
curl -sS -m 3 http://127.0.0.1:8080/health
curl -sS -m 3 http://<host-LAN-IP>:8080/health
sudo ss -lntp | grep 8080
sudo iptables -t nat -L DOCKER -n -v | grep 8080
```

- No `docker port` output → not published.
- Works on 127.0.0.1 but not on the LAN IP → published on `127.0.0.1` only.
- The DNAT packet counter doesn't rise when you connect from outside → packets aren't reaching the host at all (step 5).

### Step 5: from outside the host

- Cloud **security group / network ACL** (AWS/Azure/GCP firewall around the VM) blocks the port? This is the most common cause of "works locally, timeout from outside".
- Rules in `DOCKER-USER` dropping it? `sudo iptables -S DOCKER-USER`.
- Watch whether the packets arrive at all:
  ```bash
  sudo tcpdump -ni eth0 tcp port 8080
  ```
  No packets → the problem is outside the host (SG, routing, wrong IP or DNS on the client side).

### Step 6: outbound (container → internet / another service)

```bash
docker run --rm --network appnet nicolaka/netshoot sh -c '
  ip route; ping -c1 -W2 172.18.0.1; ping -c1 -W2 1.1.1.1; dig +short example.com; curl -sS -m5 -o /dev/null -w "%{http_code}\n" https://example.com'
```

Interpretation in order:
- No default route → network setup problem.
- Can't ping the gateway (bridge IP) → veth/bridge problem.
- Gateway OK, `1.1.1.1` fails → host forwarding/NAT: `sysctl net.ipv4.ip_forward`, `iptables -t nat -S POSTROUTING | grep MASQ`, or the host itself has no internet.
- IP works, `dig` fails → DNS (topic 29).
- `dig` works, `curl` fails or hangs → **proxy required** (corporate networks), TLS interception, or **MTU**.

### Step 7: packet capture on the right interface

Find the host-side veth of a container:

```bash
IFINDEX=$(sudo nsenter -t $PID -n cat /sys/class/net/eth0/iflink)
VETH=$(grep -l "^$IFINDEX$" /sys/class/net/veth*/ifindex | cut -d/ -f5)
echo $VETH
sudo tcpdump -ni $VETH -c 20
```

- `iflink` (inside) = the interface index of the **peer** (the host-side veth). We find the host interface with that `ifindex`.
- Capture on the veth = exactly this container's traffic. On `docker0`/`br-xxx` = everything on that network. On `eth0` = what actually leaves/enters the host (after NAT).

Or capture **inside** the namespace with the host's tcpdump:

```bash
sudo nsenter -t $PID -n tcpdump -ni eth0 port 5432
```

Reading a capture:
- `Flags [S]` repeated, no `[S.]` reply → SYN sent, no answer → dropped (firewall/route).
- `Flags [S]` then `Flags [R.]` → refused (nothing listening).
- Handshake OK, then large packets retransmitted forever → MTU.

### Step 8: conntrack (NAT state)

```bash
sudo conntrack -L -p tcp --dport 8080 2>/dev/null | head
sudo conntrack -S | head -3
```

- Shows the original and translated addresses of each connection.
- `insert_failed` / `drop` counters rising, or `nf_conntrack: table full, dropping packet` in `dmesg` → the conntrack table is full under heavy load: raise `net.netfilter.nf_conntrack_max`.

### Step 9: MTU problems

**MTU** (Maximum Transmission Unit) = the biggest packet an interface sends without splitting it. Ethernet is normally **1500 bytes**.

When container traffic goes through a **tunnel** (VPN, VXLAN overlay, some cloud networks, GRE/IPsec), the tunnel adds headers, so the **real** path MTU is smaller (1450, 1400, sometimes less). But `docker0` and container `eth0` still use 1500. Big packets don't fit:
- normally an ICMP "fragmentation needed" message tells the sender to send smaller packets (**Path MTU Discovery**),
- but many firewalls block ICMP, so the big packets just **vanish**.

Classic symptoms:
- `ping` works, small API calls work, but **large responses hang** (downloads, big JSON, `git clone`, `apt-get update`, `docker pull` inside a container).
- **TLS handshakes hang** (the server's certificate chain is a big packet).
- Works from the host, fails from containers (the host may have the correct MTU, containers don't).

Test:

```bash
ip link show eth0 | grep -o 'mtu [0-9]*'
docker run --rm nicolaka/netshoot ping -c2 -M do -s 1472 1.1.1.1
docker run --rm nicolaka/netshoot ping -c2 -M do -s 1372 1.1.1.1
```

- `-M do` = "don't fragment". `-s 1472` + 28 bytes of headers = a 1500-byte packet.
- 1472 fails (`message too long` or no reply) but 1372 works → the path MTU is between 1400 and 1500.

Fix: set Docker's MTU to the path MTU:

```json
{ "mtu": 1400 }
```

in `/etc/docker/daemon.json` for the default bridge, and per network:

```bash
docker network create -o com.docker.network.driver.mtu=1400 appnet
```

(In Compose: `driver_opts: { com.docker.network.driver.mtu: 1400 }`.) Existing networks must be recreated.

### Step 10: corporate HTTP proxies

If the company requires an HTTP proxy for internet access:
- **The daemon** needs it for `docker pull` (systemd drop-in or `daemon.json` `proxies`; topic 34).
- **Builds** need it: `--build-arg HTTP_PROXY=…` (predefined ARGs; topic 18).
- **Containers** need it at runtime: `-e HTTP_PROXY=… -e HTTPS_PROXY=… -e NO_PROXY=localhost,127.0.0.1,.internal,db`, or configured once in `~/.docker/config.json` under `"proxies"` (the CLI then injects them into new containers).

---

## Commands / files

### Run netshoot attached to a no-tools container's network

```bash
docker run -d --name api --network appnet my-distroless-api:1.0
docker run -it --rm --network container:api nicolaka/netshoot
```

Inside netshoot you **share api's network namespace**: `ss -lntp` shows api's listeners, `curl localhost:8000` hits api, `tcpdump -i eth0` sees api's packets.

### A real "refused" diagnosis

```bash
docker run -d --name bad -p 8090:8000 python:3.13-slim python -m http.server 8000 --bind 127.0.0.1
curl -sS localhost:8090
PID=$(docker inspect -f '{{.State.Pid}}' bad)
sudo nsenter -t $PID -n ss -lnt
```

```
curl: (56) Recv failure: Connection reset by peer
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      5          127.0.0.1:8000      0.0.0.0:*
```

- `Recv failure / reset` (via docker-proxy) or `refused`, and `ss` shows `127.0.0.1:8000`: the app only listens on the container's loopback. Fix: `--bind 0.0.0.0`.

### Watching a container's traffic

```bash
docker run -d --name pinger alpine ping 1.1.1.1
PID=$(docker inspect -f '{{.State.Pid}}' pinger)
IFINDEX=$(sudo nsenter -t $PID -n cat /sys/class/net/eth0/iflink)
VETH=$(grep -l "^$IFINDEX$" /sys/class/net/veth*/ifindex | cut -d/ -f5)
sudo tcpdump -ni $VETH -c 2 icmp
sudo tcpdump -ni eth0 -c 2 icmp
```

```
13:40:01.100 IP 172.17.0.3 > 1.1.1.1: ICMP echo request, id 7, seq 12, length 64
13:40:01.112 IP 1.1.1.1 > 172.17.0.3: ICMP echo reply, id 7, seq 12, length 64
13:40:02.100 IP 192.168.1.50 > 1.1.1.1: ICMP echo request, id 7, seq 13, length 64
13:40:02.112 IP 1.1.1.1 > 192.168.1.50: ICMP echo reply, id 7, seq 13, length 64
```

- On the veth: the container's private IP. On `eth0`: the **host's** IP. That's MASQUERADE, visible.

---

## Hands-on lab: five broken setups

Diagnose each with the method, **before** reading the hint.

1. **Wrong bind address:** the `bad` container above. *(Hint: step 1.)*
2. **Different networks:**
   ```bash
   docker network create n1; docker network create n2
   docker run -d --name svc --network n1 nginx:1.27-alpine
   docker run --rm --network n2 nicolaka/netshoot curl -sS -m3 svc
   ```
   *(Hint: DNS fails → step 3, `docker network inspect`.)*
3. **Port published to localhost only:** `docker run -d --name lo -p 127.0.0.1:8095:80 nginx:1.27-alpine`, then curl the WSL/VM LAN IP on 8095. *(Hint: step 4, `docker port`.)*
4. **Broken upstream DNS:** `docker run --rm --dns 10.255.255.1 nicolaka/netshoot curl -sS -m5 https://example.com`. *(Hint: step 6; `dig` shows the timeout.)*
5. **MTU black hole (simulation):** `docker network create -o com.docker.network.driver.mtu=9000 bigmtu`, then from a container on it try `curl -sS -m10 https://example.com -o /dev/null`. Depending on your path, large packets may be dropped. Compare with `ping -M do -s 1472` and `-s 8972`. *(Hint: step 9. Fix by setting the MTU to the real path MTU.)*

Clean up: `docker rm -f bad svc lo pinger 2>/dev/null; docker network rm n1 n2 bigmtu 2>/dev/null`

---

## Where you'll use it

- **On-call:** the majority of "service X can't talk to Y" tickets on Docker hosts.
- **Cloud migrations and VPNs:** MTU and DNS problems appear when networks change.
- **Corporate environments:** proxy settings and internal DNS.
- **Kubernetes:** the same hops (pod netns → veth → CNI bridge/routes → NAT/kube-proxy → node). The same tools (`nsenter`, netshoot via `kubectl debug`, tcpdump on veth) apply.

---

## When it breaks (quick symptom table)

| Symptom | First thing to check |
|---|---|
| Refused on published port | App's listen address (`ss -lntp` in netns) and container port |
| Works from host, timeout from outside | Cloud security group / host firewall / `DOCKER-USER` / bound to 127.0.0.1 |
| Container → container by name fails | Same user-defined network? (`docker network inspect`) |
| Internal names OK, external names fail | Upstream DNS servers (`ExtServers` comment), corporate DNS |
| External IPs fail, gateway OK | `ip_forward`, MASQUERADE rule, host's own connectivity, proxy requirement |
| Small requests fine, big ones/TLS hang | MTU (`ping -M do -s`), set `mtu` in daemon.json / network options |
| Everything broke after a firewall change | Docker's iptables chains flushed → `systemctl restart docker` |
| Random drops under load | conntrack table full (`dmesg`, `conntrack -S`) |
| Client IP in app logs is always `172.17.0.1` | docker-proxy/hairpin NAT (topic 28) |

---

## Quick recap

- First, classify the error: **refused** (nothing listening), **timeout** (dropped), **no route**, **reset**, **DNS failure**.
- Walk the path: app listening on `0.0.0.0`? → reachable inside netns? → from a peer container? → via the published port on the host? → from outside (SG/firewall)? Outbound: route → gateway → NAT → DNS → proxy → MTU.
- Tools: `nsenter -t PID -n` with host tools, `nicolaka/netshoot` (also with `--network container:X`), `tcpdump` on the veth/bridge/eth0, `iptables -L -v` counters, `conntrack`.
- MTU mismatches cause "small works, big hangs". Fix with Docker's `mtu` setting.
- Proxies must be set separately for the daemon, builds and containers.

**Interview questions**

1. **Q: A service is published with `-p 8080:8080` but clients get "connection refused". What do you check?**
   A: That the container is running, and what it actually listens on inside its namespace (`nsenter -t PID -n ss -lntp`). It's commonly bound to 127.0.0.1 or a different port. Then `docker port` to confirm the mapping.

2. **Q: Containers can do small HTTP calls, but large downloads and TLS handshakes hang. Cause?**
   A: Most likely an MTU mismatch: the path (VPN, overlay or cloud) has a smaller MTU than the containers' 1500, and ICMP "fragmentation needed" is blocked. Confirm with `ping -M do -s`, and set Docker's MTU to match.

3. **Q: How do you capture one container's traffic from the host?**
   A: Find its host-side veth through the peer index (`cat /sys/class/net/eth0/iflink` inside the namespace, matched to `ifindex` on the host) and run `tcpdump -i <veth>`. Or run tcpdump inside its network namespace with `nsenter -t PID -n tcpdump -i eth0`.
