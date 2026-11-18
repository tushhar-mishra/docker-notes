# 27 · Docker Network Drivers

> **Level:** ★★ intermediate
>
> **You will learn:** every way a container can be networked: the default `bridge` (`docker0`) and why you should use **user-defined bridges** instead, `host`, `none`, `container:` mode (sharing another container's network), `macvlan`/`ipvlan` (containers directly on your LAN), `overlay` (multi-host, Swarm), connecting a container to several networks, IPv6, and how Docker's chosen subnets can **clash with your company network**.
>
> **You need:** topic 26 (namespaces, veth, bridges, NAT).

---

## What it is

A **network driver** decides how a container's network namespace is wired up. `docker network ls` shows the networks; each uses a driver.

| Driver | What the container gets | Typical use |
|---|---|---|
| `bridge` (default network `bridge` / `docker0`) | Own namespace, veth to `docker0`, NAT to outside | Legacy default. Avoid for real apps |
| `bridge` (**user-defined**) | Same, but on its own bridge `br-xxxx`, **with DNS by container name** | **The standard for single-host apps** (Compose uses this) |
| `host` | **No** network namespace: uses the host's network directly | Max performance, network tools, many ports |
| `none` | Own namespace with only `lo` | Batch jobs with no network, security isolation |
| `container:<name>` | **Shares another container's** network namespace | Sidecars, debugging (topic 51), how Kubernetes Pods work |
| `macvlan` | Its own **MAC and IP on your physical LAN** | Apps that must look like real machines on the LAN |
| `ipvlan` | Own IP on the LAN, sharing the host's MAC | Like macvlan where switches limit MACs (cloud, some Wi-Fi) |
| `overlay` | Network spanning **several hosts** (VXLAN tunnels) | Docker Swarm |

---

## Why it exists

Different apps need different trade-offs:
- **Isolation** (bridge, none) vs **performance and simplicity** (host).
- **Name-based discovery** between services (user-defined bridge).
- **Being a real host on the LAN** (macvlan/ipvlan) for legacy systems, DHCP servers or licence servers.
- **Spanning hosts** (overlay).

---

## Analogy

Ways to give an employee a phone line:
- **Default bridge** = a shared office phone switch where people can only dial by **extension number** (IP), and the numbers change when people move desks.
- **User-defined bridge** = a team switch with a **directory**: you dial "accounting" by name.
- **host** = the employee uses the company's main reception line directly. No switch, but also no privacy.
- **none** = no phone at all.
- **container:** = two people sharing **one** desk phone and number.
- **macvlan** = the employee gets their own direct external phone number from the phone company.

---

## How it works inside

### The default `bridge` network (`docker0`)

Containers started **without** `--network` land here.
- Subnet usually `172.17.0.0/16`, gateway `172.17.0.1` on `docker0`.
- **No DNS by container name.** Containers can reach each other only by IP (which changes on restart). The old `--link` option added `/etc/hosts` entries, but it's **deprecated**. Don't use it.
- All containers that use it share one flat network, so unrelated apps can reach each other.

### User-defined bridge networks

`docker network create appnet` creates:
- a new Linux bridge `br-<12 hex chars of the network ID>`,
- a new subnet (by default the next free one from Docker's **address pools**, e.g. `172.18.0.0/16`, `172.19.0.0/16`, …),
- iptables/nftables rules for NAT and **isolation between networks** (containers on `appnet` can't reach containers on `othernet`),
- **DNS by name**: Docker's embedded DNS server answers container names and aliases (topic 29).

Containers on a user-defined bridge can reach each other on **all ports**. Publishing (`-p`) is only needed for access **from outside** the host network.

### `host` network

- The container skips the network namespace. It sees the host's interfaces, IPs and ports. Listening on port 80 = listening on the host's port 80.
- `-p` is **ignored** (there's nothing to forward).
- No NAT, so the best network performance and real client IPs.
- Downsides: port conflicts with the host and other host-network containers, and **no network isolation**.
- On Docker Desktop, "host" means the VM's network (it now also supports host networking with some limits).

### `none`

- Network namespace with only `lo`. No way in or out. Good for jobs that process local files, or as a security measure.

### `container:<name>` (shared network namespace)

```bash
docker run -d --name app nginx
docker run --rm --network container:app nicolaka/netshoot ss -lntp
```

- The second container **joins** the first one's network namespace: same `eth0`, same IP, same ports. They can talk over `localhost`.
- **This is exactly how a Kubernetes Pod works**: all containers in a Pod share one network namespace, held open by the "pause" container (topic 11, 53).
- Great for debugging: attach a toolbox container to a no-tools app's network (topic 30, 51).

### macvlan and ipvlan

- **macvlan**: Docker creates sub-interfaces on a physical NIC (e.g. `eth0`), each with **its own MAC address**. Containers get IPs **from your LAN's subnet** and are reachable directly, without NAT or port publishing.
  - Needs the network/switch to allow several MACs on one port (often blocked in clouds and on Wi-Fi).
  - Catch: by Linux design, the **host can't talk to its own macvlan containers** directly (the parent interface and macvlan children are isolated). A workaround is a macvlan sub-interface on the host too.
- **ipvlan** (L2 mode): like macvlan, but all containers share the host's MAC. Works where extra MACs are forbidden. (L3 mode routes instead of switching.)
- You must manage IP ranges so Docker doesn't hand out addresses your DHCP server also uses (`--ip-range`).

### overlay

- Connects containers on **several Docker hosts** as if they were on one network, using **VXLAN** (Virtual Extensible LAN: Ethernet frames wrapped in UDP packets, port 4789) between hosts.
- Requires **Swarm mode** (Docker's built-in orchestrator; topic 53). In Kubernetes, the CNI plugin does this job instead.

### Multiple networks per container

A container can be on several networks at once, with one `eth` interface per network:

```bash
docker network connect backend api
```

Typical design: `frontend` (proxy + api) and `backend` (api + db). The proxy can't reach the db directly. Only the api is on both.

### IPv6

- Off by default for new networks. Enable per network: `docker network create --ipv6 --subnet 2001:db8:1::/64 v6net`.
- For the default bridge, set `"ipv6": true` and `"fixed-cidr-v6"` in `daemon.json` (topic 34).
- Since Docker 27, IPv6 NAT is supported (`ip6tables` on by default), so containers can use private IPv6 ranges like IPv4.

### Address pools and subnet clashes (a real production problem)

Docker picks subnets for new networks from its **default address pools**: `172.17.0.0/16` to `172.31.0.0/16`, then `192.168.0.0/16` split into `/20` blocks.

If your company network, VPN or cloud VPC (Virtual Private Cloud, your private network in the cloud) **also uses** one of those ranges (for example `172.18.0.0/16` for an office, or `192.168.x.x` for a VPN), then:
- the host adds a route for the Docker network that **captures traffic meant for the real network**,
- the server suddenly can't reach that office, VPN or database, or SSH from there to the host drops,
- it often happens "randomly", when a new Compose project creates a new network.

Fix: set non-clashing pools in `/etc/docker/daemon.json` (topic 34):

```json
{
  "bip": "10.200.0.1/24",
  "default-address-pools": [
    { "base": "10.201.0.0/16", "size": 24 }
  ]
}
```

- `bip` = the `docker0` bridge IP and subnet.
- `default-address-pools` = where new networks get their subnets: here, `/24` networks carved from `10.201.0.0/16` (256 networks of 254 addresses).

### Inter-container communication on the default bridge (`icc`)

`"icc": false` in `daemon.json` blocks container-to-container traffic on the **default** bridge. For user-defined networks, use separate networks (or the network option `com.docker.network.bridge.enable_icc=false`).

---

## Commands / files

### List and inspect networks

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
b2c3d4e5f6a7   host      host      local
c3d4e5f6a7b8   none      null      local
```

- Three networks always exist. `SCOPE local` = this host only (`swarm` for overlay).

```bash
docker network create appnet
docker network inspect appnet --format '{{json .IPAM.Config}} {{.Options}}'
ip -br link | grep br-
```

```
[{"Subnet":"172.18.0.0/16","Gateway":"172.18.0.1"}] map[]
br-9f8e7d6c5b4a  UP  ...
```

- **IPAM** = IP Address Management (which subnet and gateway, and which container gets which IP).
- The bridge name `br-9f8e7d6c5b4a` = `br-` + the network ID's first 12 characters.

Create with explicit settings:

```bash
docker network create --driver bridge \
  --subnet 10.201.5.0/24 --gateway 10.201.5.1 \
  --opt com.docker.network.bridge.name=br-payments \
  --label team=payments payments-net
```

### Default bridge: no names. User-defined: names work

```bash
docker run -d --name a1 alpine sleep 600
docker run --rm alpine ping -c1 -W2 a1
docker run -d --name b1 --network appnet alpine sleep 600
docker run --rm --network appnet alpine ping -c1 b1
```

```
ping: bad address 'a1'
PING b1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.089 ms
```

### Isolation between networks

```bash
docker network create othernet
docker run --rm --network othernet alpine ping -c1 -W2 172.18.0.2
```

```
1 packets transmitted, 0 packets received, 100% packet loss
```

- Docker's rules drop traffic between different bridge networks.

### Connect a container to a second network

```bash
docker network connect othernet b1
docker exec b1 ip -br addr
docker inspect b1 -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}={{$v.IPAddress}} {{end}}'
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0@if21        UP             172.18.0.2/16
eth1@if24        UP             172.19.0.2/16
appnet=172.18.0.2 othernet=172.19.0.2
```

- One veth/interface per network.

### host and none

```bash
docker run --rm --network host alpine ip -br addr | head -3
docker run --rm --network none alpine ip -br addr
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.1.50/24
docker0          UP             172.17.0.1/16
lo               UNKNOWN        127.0.0.1/8 ::1/128
```

- With `host`, the container sees the host's `eth0` and `docker0`.

### container: mode

```bash
docker run -d --name web --network appnet nginx:1.27-alpine
docker run --rm --network container:web alpine wget -qO- localhost | head -4
```

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

- `localhost` in the second container is the **same** network stack as `web`.

### macvlan

```bash
docker network create -d macvlan \
  --subnet 192.168.1.0/24 --gateway 192.168.1.1 \
  --ip-range 192.168.1.192/27 \
  -o parent=eth0 lan
docker run -d --name lanweb --network lan nginx:1.27-alpine
docker inspect -f '{{.NetworkSettings.Networks.lan.IPAddress}}' lanweb
```

```
192.168.1.192
```

- `parent=eth0` = the physical NIC. `--ip-range` = only hand out `.192`–`.223` (keep them outside your DHCP range).
- Other machines on the LAN can now browse `http://192.168.1.192` directly. (This won't work in WSL2 or most cloud VMs: their virtual networks don't pass extra MACs.)

### Remove networks

```bash
docker network rm othernet
```

```
Error response from daemon: error while removing network: network othernet has active endpoints (name:"b1" id:"…")
```

- Disconnect or remove its containers first (`docker network disconnect othernet b1`).

---

## Hands-on lab

1. Run the default-bridge vs user-defined-bridge name test above.
2. Build a **three-tier isolation** setup:
   ```bash
   docker network create frontend; docker network create backend
   docker run -d --name db --network backend -e POSTGRES_PASSWORD=pw postgres:17-alpine
   docker run -d --name api --network backend alpine sleep 3600
   docker network connect frontend api
   docker run -d --name proxy --network frontend alpine sleep 3600
   docker exec api   sh -c 'nc -w2 db 5432 </dev/null && echo "db: port open"'
   docker exec proxy sh -c 'nc -w2 db 5432 </dev/null && echo "db: port open"'
   docker exec proxy ping -c1 api
   ```
   `api → db` prints `port open`. `proxy → db` fails with `nc: bad address 'db'`: the name isn't even resolvable from the frontend network. `proxy → api` resolves and answers.
   (We use `nc host port </dev/null` instead of `nc -z`, because the `-z` flag isn't compiled into every BusyBox build. For full tools, run `nicolaka/netshoot` on the same network.)
3. Look at the plumbing: `ip -br link | grep br-`, `bridge link`, and `docker network inspect backend` (look at `Containers`).
4. **Subnet clash simulation:** `ip route | grep -E '172\.(1[7-9]|2[0-9]|3[01])'` lists the routes Docker added. Imagine your VPN uses `172.18.0.0/16`: which of these routes would steal its traffic?
5. Clean up: `docker rm -f a1 b1 web db api proxy lanweb 2>/dev/null; docker network prune -f`

---

## Where you'll use it

- **Every Compose project** gets its own user-defined bridge (topic 35). Services find each other by name.
- **Segmentation:** frontend/backend networks so databases aren't reachable from edge services.
- **Performance-critical or network-tool containers** (monitoring agents, load balancers) use `host`.
- **Corporate environments:** fixing `default-address-pools` to avoid clashing with office/VPN/VPC ranges is very common day-one work on a new Docker host.
- **Kubernetes understanding:** `container:` mode = how Pods share a network namespace.

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| `ping: bad address 'db'` | Not on the same **user-defined** network (or on the default bridge, which has no DNS) | Put both on a user-defined network |
| Host suddenly can't reach an office/VPN range after starting a project | A Docker network took an overlapping subnet | `ip route`; set `default-address-pools` and `bip`, recreate networks |
| `Error response from daemon: Pool overlaps with other one on this address space` | You asked for a subnet already used by another Docker network | Pick another subnet; `docker network inspect` to find the existing one |
| `could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network` | All default pools used up (many Compose projects) | `docker network prune`; larger/more pools in `daemon.json` |
| `-p` does nothing with `--network host` | Ports aren't published in host mode | The app listens on the host's port directly |
| Port already in use with `--network host` | Something on the host already uses that port | `ss -lntp` on the host |
| macvlan container unreachable from the Docker host itself | Linux macvlan parent/child isolation | Add a macvlan interface on the host, or reach it from another machine |
| macvlan works at home but not in the cloud/WSL | The virtual network blocks extra MACs | Use ipvlan or normal bridge + publishing |
| `network ... has active endpoints` on rm | Containers still connected | Disconnect/remove them first |

---

## Quick recap

- The **default bridge** (`docker0`) has no name resolution, and `--link` is deprecated. Use **user-defined bridges**: their own subnet, DNS by name, isolation from other networks.
- `host` = no network namespace (fast, no isolation, `-p` ignored). `none` = only loopback. `container:X` = share X's network namespace (how Pods work).
- `macvlan`/`ipvlan` put containers directly on the LAN. `overlay` spans hosts (Swarm).
- A container can join several networks (one interface each), which is useful for segmentation.
- Set `bip` and `default-address-pools` so Docker's subnets don't clash with company/VPN/VPC ranges.

**Interview questions**

1. **Q: Why use a user-defined bridge instead of the default bridge?**
   A: User-defined bridges provide automatic DNS resolution by container name, isolate the app from unrelated containers, and allow connecting and disconnecting containers on the fly. The default bridge only supports IP-based communication (or the deprecated `--link`).

2. **Q: What's the difference between bridge and host networking?**
   A: Bridge gives the container its own network namespace, a private IP and NAT, and you publish ports to expose it. Host networking shares the host's network namespace: no NAT, best performance, no isolation, and no port mapping.

3. **Q: After deploying a Compose project, the server lost connectivity to part of the corporate network. Why?**
   A: Docker allocated the new network's subnet from its default pools, and it overlapped a corporate range. The host route for the Docker bridge captured that traffic. Configure non-overlapping `default-address-pools` (and `bip`) in `daemon.json`.
