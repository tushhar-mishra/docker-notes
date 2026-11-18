# 26 · Linux Networking Recap for Containers

> **Level:** ★★★ advanced
>
> **You will learn:** the five Linux networking building blocks Docker uses (**network namespaces, veth pairs, bridges, routing, and NAT with iptables/nftables**), and you'll build Docker's default network **by hand** with `ip` and `iptables`, so that every later networking topic is just "Docker automating what you already did".
>
> **You need:** topic 02 (namespaces), topic 11 (`nsenter -n`). From your Linux notes: IP addresses, subnets (CIDR like `/24`), routes, `ip addr`, `ip route`, basic iptables.

---

## What it is

| Building block | One-line meaning | Docker's use |
|---|---|---|
| **Network namespace** | A private copy of the network stack: interfaces, IPs, routes, firewall rules, ports | Each container gets one |
| **veth pair** | A virtual cable with two ends; what goes in one end comes out the other | Connects the container's namespace to the host |
| **Linux bridge** | A virtual **switch** inside the kernel | `docker0` and each user-defined network |
| **Routing + IP forwarding** | The kernel forwarding packets between interfaces | Host routes container traffic to the outside |
| **NAT (iptables/nftables)** | Rewriting packet addresses | Outbound: hide container IPs behind the host's IP (**masquerade**). Inbound: **port publishing** (DNAT) |

---

## Why it exists

Topic 02 showed that a new network namespace has **nothing** in it, just a `lo` interface that's down. A container needs to:
1. talk to other containers,
2. reach the internet,
3. be reachable from outside (published ports).

Docker builds this from standard Linux parts. When networking breaks in production, you'll debug **these** parts with **these** tools. Docker has no magic networking layer.

---

## Analogy

An **office building**:
- **Network namespace** = a separate office with its own phone extension list.
- **veth pair** = a network cable: one end plugged into the office's wall socket (`eth0` in the container), the other into the floor's switch.
- **Bridge** = the floor's network switch, connecting all office cables together.
- **Routing/IP forwarding** = the building's router that sends traffic between floors and out to the street.
- **NAT masquerade** = the reception desk: every outgoing letter gets the **building's** return address, and reception forwards the replies to the right office.
- **DNAT (port publishing)** = reception's rule: "deliveries for door 8080 go to office 3's door 80".

---

## How it works inside

### Network namespace

- Has its own: interfaces, IP addresses, routing table, iptables/nftables rules, listening ports, `/proc/sys/net` settings.
- Two namespaces can both listen on port 80, because they're separate stacks. That's how many containers can each run a web server on port 80 at once.
- Created with `ip netns add <name>` (which also makes a named handle in `/var/run/netns/`). Docker creates them without names (`/var/run/docker/netns/<id>`), so `ip netns list` doesn't show Docker's namespaces. You reach them through the container's PID instead (`nsenter -t PID -n`).

### veth pair

- Created as a **pair**: `veth-a` and `veth-b`. A packet sent into one immediately comes out of the other.
- You move **one end into a namespace** (it becomes the container's `eth0`) and keep the other end on the host (`vethXXXXXX`), plugged into a bridge.
- `eth0@if15` inside a container means "my peer is interface number 15 on the other side" (topic 11).

### Linux bridge

- A software **switch** (layer 2: it forwards **Ethernet frames** by **MAC address**, the hardware address of a network interface). It learns which MAC is behind which port.
- The bridge itself can also have an **IP address**. Then it acts as the **gateway** (the router address) for everything plugged in. `docker0` has `172.17.0.1`.
- Containers on the same bridge talk **directly through the bridge**: no NAT, no routing.

### Routing and IP forwarding

- Inside the container: `default via 172.17.0.1 dev eth0` = "anything not on my local subnet, send to the bridge's IP".
- On the host: the kernel must be willing to **forward** packets between interfaces (from `docker0` to `eth0`). That's the `net.ipv4.ip_forward=1` setting. **Docker turns it on.**

### NAT: iptables and nftables

**iptables** is the classic tool for the kernel's packet filter (**netfilter**). Rules live in **tables** (`filter` for allow/deny, `nat` for address rewriting) and **chains** (checkpoints a packet passes):

```
incoming packet
   │
   ▼
PREROUTING (nat: DNAT happens here: "port 8080 → 172.17.0.2:80")
   │
   ├── for this host? ──► INPUT ──► local process
   │
   └── to forward? ──► FORWARD (filter: allow/deny between interfaces) ──► POSTROUTING (nat: SNAT/MASQUERADE: "source = host's IP") ──► out
```

- **SNAT/MASQUERADE** (Source NAT): change the **source** address of outgoing packets to the host's IP. Container IPs (`172.17.x.x`) are private and unknown outside, so replies must come back to the host. MASQUERADE = SNAT using whatever IP the outgoing interface has.
- **DNAT** (Destination NAT): change the **destination** of incoming packets: `host:8080` → `172.17.0.2:80`. That's port publishing (topic 28).
- **conntrack** (connection tracking): the kernel remembers each NATed connection, so replies are translated back automatically. `conntrack -L` lists them.

**nftables** is the modern replacement for iptables (one tool, `nft`, faster, better rule sets). On most modern distributions, the `iptables` command is actually **`iptables-nft`**: it speaks iptables syntax but writes **nftables** rules underneath. Docker Engine 29 can also write native nftables rules (opt-in, topic 28).

### Docker's default network, in these terms

```
                                   HOST (root network namespace)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │   eth0 (192.168.1.50) ◄── internet                                              │
 │     ▲                                                                           │
 │     │ routing + ip_forward + iptables MASQUERADE (172.17.0.0/16 → host IP)      │
 │     │                                                                           │
 │   docker0 bridge (172.17.0.1/16)                                                │
 │     ├── vethA ─┐                       ├── vethB ─┐                             │
 └────────────────┼───────────────────────────────────┼─────────────────────────────┘
                  │ (veth pair)                       │
        ┌─────────┴─────────┐               ┌─────────┴─────────┐
        │ container 1 netns │               │ container 2 netns │
        │ eth0 172.17.0.2   │               │ eth0 172.17.0.3   │
        │ route: default    │               │ route: default    │
        │   via 172.17.0.1  │               │   via 172.17.0.1  │
        └───────────────────┘               └───────────────────┘
```

---

## Commands / files

These are the tools you'll use throughout Part 5:

| Command | Shows |
|---|---|
| `ip -br addr` | Interfaces and IPs, brief |
| `ip -br link` | Interfaces and state, brief |
| `ip route` | Routing table |
| `bridge link` | Which interfaces are plugged into which bridge |
| `ip -d link show docker0` | Details (type bridge, etc.) |
| `sysctl net.ipv4.ip_forward` | Is forwarding on? |
| `sudo iptables -t nat -S` | NAT rules in "command" form |
| `sudo iptables -S FORWARD` | Forwarding filter rules |
| `sudo nft list ruleset` | All nftables rules |
| `sudo conntrack -L` | Tracked (NATed) connections (package `conntrack`) |
| `ss -lntp` | Listening TCP ports and their processes |

Look at Docker's default setup on your host:

```bash
ip -br addr show docker0
bridge link
sysctl net.ipv4.ip_forward
sudo iptables -t nat -S POSTROUTING
```

```
docker0          UP             172.17.0.1/16 fe80::42:5aff:fe1c:2b3d/64
15: veth3a9c2f1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2
net.ipv4.ip_forward = 1
-P POSTROUTING ACCEPT
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

- `docker0 UP 172.17.0.1/16` = the bridge and its gateway IP.
- `veth3a9c2f1 ... master docker0` = one container's cable is plugged into docker0. (Seen only while a container runs.)
- `ip_forward = 1` = Docker enabled forwarding.
- `-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE` = "packets **from** the container subnet that are **leaving through any interface other than docker0** get the host's IP as their source". That's how containers reach the internet.

---

## Hands-on lab: build Docker's network by hand

Do this in your WSL2 Ubuntu or a Linux VM. You'll create two "containers" (namespaces), a bridge, connect them, and give them internet access. Then compare with Docker.

**Step 1: two network namespaces**

```bash
sudo ip netns add red
sudo ip netns add blue
ip netns list
sudo ip netns exec red ip -br addr
```

```
blue
red
lo               DOWN           
```

- `ip netns exec red <cmd>` = run a command inside the `red` namespace (like `nsenter -n`).
- Only `lo`, and it's DOWN.

**Step 2: a bridge (our "docker0")**

```bash
sudo ip link add br-lab type bridge
sudo ip addr add 10.10.0.1/24 dev br-lab
sudo ip link set br-lab up
```

**Step 3: two veth cables, one end in each namespace, the other on the bridge**

```bash
sudo ip link add veth-red type veth peer name veth-red-br
sudo ip link add veth-blue type veth peer name veth-blue-br

sudo ip link set veth-red netns red
sudo ip link set veth-blue netns blue

sudo ip link set veth-red-br master br-lab up
sudo ip link set veth-blue-br master br-lab up
```

- `type veth peer name X` = create a pair.
- `set veth-red netns red` = move that end into the namespace.
- `master br-lab` = plug the other end into the bridge.

**Step 4: IPs, interfaces up, default routes**

```bash
sudo ip netns exec red  ip addr add 10.10.0.2/24 dev veth-red
sudo ip netns exec blue ip addr add 10.10.0.3/24 dev veth-blue
for ns in red blue; do
  sudo ip netns exec $ns ip link set lo up
  sudo ip netns exec $ns ip link set veth-$ns up
  sudo ip netns exec $ns ip route add default via 10.10.0.1
done
sudo ip netns exec red ip route
```

```
default via 10.10.0.1 dev veth-red
10.10.0.0/24 dev veth-red proto kernel scope link src 10.10.0.2
```

**Step 5: container-to-container through the bridge**

```bash
sudo ip netns exec red ping -c 2 10.10.0.3
```

```
PING 10.10.0.3 (10.10.0.3) 56(84) bytes of data.
64 bytes from 10.10.0.3: icmp_seq=1 ttl=64 time=0.071 ms
64 bytes from 10.10.0.3: icmp_seq=2 ttl=64 time=0.052 ms
```

- Works with no NAT and no routing: same bridge, same subnet. `ttl=64` (not decreased) = no router hop.
- If this fails on a Docker host, it's usually because Docker set the iptables `FORWARD` policy to `DROP` (it does that to protect containers). Allow your lab bridge: `sudo iptables -I FORWARD -i br-lab -o br-lab -j ACCEPT`.

**Step 6: reach the host and the internet**

```bash
sudo ip netns exec red ping -c 1 10.10.0.1          # the bridge (host side): works
sudo ip netns exec red ping -c 1 -W 2 1.1.1.1       # internet: fails
```

To fix the internet, do what Docker does: forwarding + MASQUERADE + allow forwarding in the filter table.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 ! -o br-lab -j MASQUERADE
sudo iptables -I FORWARD -i br-lab ! -o br-lab -j ACCEPT
sudo iptables -I FORWARD -o br-lab -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo ip netns exec red ping -c 2 1.1.1.1
```

```
64 bytes from 1.1.1.1: icmp_seq=1 ttl=56 time=12.4 ms
```

- MASQUERADE rewrote the source from `10.10.0.2` to your host's IP. conntrack translates the replies back.
- The two FORWARD rules: let traffic **out** of the bridge, and let **replies** of existing connections back in. Docker creates equivalent rules for each network.

Watch the NAT entry:

```bash
sudo ip netns exec red ping -c 1 1.1.1.1 >/dev/null; sudo conntrack -L -p icmp 2>/dev/null | head -2
```

```
icmp     1 29 src=10.10.0.2 dst=1.1.1.1 type=8 code=0 id=4512 src=1.1.1.1 dst=192.168.1.50 type=0 code=0 id=4512 mark=0 use=1
```

- First half = the original packet (from `10.10.0.2`). Second half = what the reply looks like (to the **host's** IP `192.168.1.50`). That's MASQUERADE in action.

**Step 7: "publish a port" (preview of topic 28)**

```bash
sudo ip netns exec blue python3 -m http.server 80 >/dev/null 2>&1 &
sudo iptables -t nat -A PREROUTING -p tcp --dport 9090 -j DNAT --to-destination 10.10.0.3:80
sudo iptables -t nat -A OUTPUT -p tcp -d 127.0.0.1 --dport 9090 -j DNAT --to-destination 10.10.0.3:80
sudo sysctl -w net.ipv4.conf.all.route_localnet=1
sudo iptables -t nat -A POSTROUTING -s 127.0.0.1 -d 10.10.0.3 -j MASQUERADE
curl -s -o /dev/null -w '%{http_code}\n' localhost:9090
```

```
200
```

- `PREROUTING ... DNAT` = traffic from **other machines** to port 9090 → blue's port 80.
- `OUTPUT ... DNAT` = the same for traffic from **this host itself** (local traffic doesn't go through PREROUTING).
- `route_localnet` + the MASQUERADE rule = needed so packets from `127.0.0.1` can be routed to a namespace at all. (This hairpin case is why Docker also runs a userland `docker-proxy` process for localhost traffic; topic 28.)

**Step 8: compare with Docker**

```bash
docker run -d --name n1 nginx:1.27-alpine
PID=$(docker inspect -f '{{.State.Pid}}' n1)
sudo nsenter -t $PID -n ip -br addr
sudo nsenter -t $PID -n ip route
bridge link | grep docker0
```

Same structure: `eth0@ifN` with a `172.17.0.x` IP, default route via `172.17.0.1`, a veth on `docker0`.

**Step 9: clean up**

```bash
sudo pkill -f 'http.server 80'
sudo ip netns del red; sudo ip netns del blue        # deleting a namespace deletes the veth ends inside it (and so their peers)
sudo ip link del br-lab
sudo iptables -t nat -D POSTROUTING -s 10.10.0.0/24 ! -o br-lab -j MASQUERADE
sudo iptables -D FORWARD -i br-lab ! -o br-lab -j ACCEPT
sudo iptables -D FORWARD -o br-lab -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -D FORWARD -i br-lab -o br-lab -j ACCEPT 2>/dev/null
sudo iptables -t nat -D PREROUTING -p tcp --dport 9090 -j DNAT --to-destination 10.10.0.3:80
sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 9090 -j DNAT --to-destination 10.10.0.3:80
sudo iptables -t nat -D POSTROUTING -s 127.0.0.1 -d 10.10.0.3 -j MASQUERADE
docker rm -f n1
```

---

## Where you'll use it

- **Debugging container networking** (topic 30): you'll check each piece: namespace interfaces, veth/bridge membership, routes, forwarding, NAT rules, conntrack.
- **Kubernetes:** CNI plugins (CNI = Container Network Interface, the standard for container network plugins) such as Flannel, Calico and Cilium build exactly these pieces (veth pairs, bridges or routes, NAT or eBPF) for every Pod. **eBPF** (extended Berkeley Packet Filter) = small, verified programs loaded into the running kernel. They're used for fast packet handling (replacing long iptables chains), tracing and security monitoring.
- **Firewall incidents:** Docker's iptables rules interacting with ufw/firewalld (topic 28).
- **Interviews:** "How does a container reach the internet?" → veth → bridge → routing → MASQUERADE.

---

## When it breaks

| Symptom | Likely broken piece | Check |
|---|---|---|
| Container has no IP / only `lo` | Network namespace not connected (e.g. `--network none`) | `nsenter -t PID -n ip -br addr` |
| Containers on the same bridge can't ping each other | Bridge/veth down, or FORWARD rules blocking, or `icc` disabled | `bridge link`, `iptables -S FORWARD` |
| Container can ping the gateway but not the internet | `ip_forward=0`, missing MASQUERADE, or the host itself has no route/DNS | `sysctl net.ipv4.ip_forward`, `iptables -t nat -S POSTROUTING` |
| All Docker networking broke after a firewall reload | `firewalld`/`iptables-restore`/`ufw` flushed Docker's rules | Restart Docker to recreate them (topic 28) |
| `ip_forward` keeps getting reset | A sysctl config or a security tool sets it to 0 | `grep -r ip_forward /etc/sysctl.*`; Docker re-enables it when it starts |
| Rules shown by `iptables` and `nft` don't match | Mixed iptables-legacy and iptables-nft backends | `iptables -V` shows `(nf_tables)` or `(legacy)`; use one |

---

## Quick recap

- **Network namespace** = a private network stack. **veth pair** = a virtual cable. **Bridge** = a virtual switch with the gateway IP.
- Containers on one bridge talk directly. To leave the host, packets are **routed** (`ip_forward=1`) and **masqueraded** (SNAT to the host IP); conntrack handles the replies.
- **DNAT** in `PREROUTING`/`OUTPUT` rewrites host ports to container IP:port. That's port publishing.
- Modern `iptables` is usually `iptables-nft` writing nftables rules.
- You built `docker0` by hand: Docker automates exactly these steps.

**Interview questions**

1. **Q: How does a Docker container reach the internet?**
   A: Its `eth0` is one end of a veth pair. The other end is attached to the `docker0` bridge, whose IP is the container's default gateway. The host forwards the packet (`ip_forward=1`), and an iptables MASQUERADE rule rewrites the source IP to the host's. conntrack translates the replies back.

2. **Q: What is a veth pair?**
   A: A pair of connected virtual Ethernet interfaces: packets sent into one come out the other. One end is placed in a container's network namespace and the other stays on the host (usually attached to a bridge).

3. **Q: Why can two containers both listen on port 80?**
   A: Each has its own network namespace with its own interfaces and port space, so there's no conflict. Only publishing them on the **same host port** would conflict.
