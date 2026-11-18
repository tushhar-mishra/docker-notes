# 01 · Why Containers

> **Level:** ★ foundation
>
> **You will learn:** the real problems that made containers necessary, how we got from physical servers to virtual machines to containers, and what Docker added that didn't exist before.
>
> **You need:** nothing. This is the starting point.

---

## What it is

A **container** is a way to package an application together with everything it needs to run: the code, the language runtime (like Python or Java), the libraries, the config files. You then run that package as an isolated process on a Linux machine.

**Docker** is the tool that made containers easy. It gives you one command to build the package, one command to share it, and one command to run it anywhere.

In this topic we don't touch any commands yet. We understand **why** this exists. Every later topic is easier when you know which problem a feature is solving.

---

## Why it exists

### Problem 1: "It works on my machine"

A developer writes a Python app on their laptop. It works. They send it to the operations team. On the server, it crashes.

Why? The laptop has Python 3.12. The server has Python 3.9. The laptop has the `libssl` library version 3. The server has version 1.1. The laptop has an environment variable set that nobody remembered. The code is the same, but the **environment** around it is different.

> **Environment** = everything around your code that it depends on: the operating system, installed libraries, language version, config files, environment variables.

### Problem 2: Dependency conflicts on one server

You have one server. App A needs Python 3.9. App B needs Python 3.12. App C needs a library that conflicts with App B's library. On a normal Linux server, there is one `/usr/lib`, one `/usr/bin/python3`. Installing App B's needs can break App A.

Teams used to solve this with "one app per server", which wastes money.

### Problem 3: Servers are slow to set up and drift over time

Setting up a server by hand takes hours: install packages, edit configs, create users. Someone logs in at 2 AM during an outage and changes a config file. Nobody writes it down. Six months later, server 1 and server 2 are "the same" on paper but behave differently. This is called **configuration drift**.

### Problem 4: Wasted hardware

A typical app uses maybe 5–15% of a server's CPU. If you run one app per server to avoid conflicts, you pay for 85% idle hardware.

### What we want instead

1. **Package the environment with the app**, so it runs the same everywhere.
2. **Isolate apps from each other** on the same machine, so they can't conflict.
3. **Start fast and be thrown away easily**, so we rebuild instead of patching by hand.
4. **Share hardware efficiently**, so we run many apps per machine.

---

## Analogy

Think of **shipping containers** (the steel boxes on cargo ships).

Before them, cargo was loaded piece by piece: sacks, barrels, crates of all shapes. Every port needed different handling. Loading a ship took weeks. Things broke or got lost.

The shipping container standardized the box. The crane doesn't care if the box holds bananas or car parts. The ship, the train and the truck all handle the same box the same way.

A software container does the same thing:
- The **box** is the container image (a package with your app and its environment).
- The **crane, ship, truck** are your laptop, the test server, the production server, the Kubernetes cluster.
- Nobody along the way needs to know what's inside. They just know how to run "a container".

---

## How it works inside: the journey from physical servers to containers

### Stage 1: Physical servers (bare metal)

```
+---------------------------------------+
|  App A    App B    App C               |
|  (all share the same libraries!)       |
+---------------------------------------+
|  Operating System (one Linux)          |
+---------------------------------------+
|  Physical hardware                     |
+---------------------------------------+
```

- All apps share one OS and one set of libraries, so they conflict.
- Or you buy one server per app, which wastes money.
- A new server takes days or weeks (order, rack, install).

### Stage 2: Virtual machines (VMs)

A **virtual machine** is a fake computer made in software. A program called a **hypervisor** (for example VMware ESXi, KVM, Hyper-V) splits one physical machine into several virtual ones. Each VM gets its own virtual CPU, virtual memory, virtual disk, and runs its **own complete operating system**, including its own **kernel**.

> **Kernel** = the core of the operating system. It is the one program that talks directly to the hardware. It manages processes, memory, files and the network. Every other program asks the kernel to do things for it, through **system calls** (requests like "open this file" or "send this network packet").

```
+-----------+  +-----------+  +-----------+
|  App A    |  |  App B    |  |  App C    |
|  libs     |  |  libs     |  |  libs     |
|  Guest OS |  |  Guest OS |  |  Guest OS |   <- full OS + kernel each
|  (kernel) |  |  (kernel) |  |  (kernel) |
+-----------+  +-----------+  +-----------+
|            Hypervisor                    |
+------------------------------------------+
|            Physical hardware             |
+------------------------------------------+
```

**What VMs fixed:**
- Strong isolation: each app has its own OS, so no conflicts.
- Many apps per physical server: better hardware use.

**What VMs didn't fix:**
- Each VM carries a full OS: gigabytes of disk and hundreds of MB of RAM just for the OS.
- A VM boots a whole kernel: it takes tens of seconds to minutes.
- "Works on my machine" is still there. The developer's laptop is not the same VM as production.
- VM images are huge and awkward to version and share.

### Stage 3: Containers

A container does **not** have its own kernel. All containers on a machine **share the host's Linux kernel**. The kernel itself is told to show each container a **separate view** of the system: its own list of processes, its own network, its own files. It also limits how much CPU and memory each container may use.

```
+-----------+  +-----------+  +-----------+
|  App A    |  |  App B    |  |  App C    |
|  libs     |  |  libs     |  |  libs     |   <- only the app + its libraries
+-----------+  +-----------+  +-----------+
|   Container runtime (e.g. Docker)        |
+------------------------------------------+
|   Host OS: ONE shared Linux kernel       |
+------------------------------------------+
|   Hardware (physical or a VM)            |
+------------------------------------------+
```

So a container is really **just a normal Linux process** (a running program) that the kernel has fenced off. You'll see this with your own eyes in topic 02 and topic 11.

**Why this is fast and light:**
- There is no second kernel to boot. Starting a container = starting a process. Milliseconds to a second.
- The package contains only the app and its libraries, not a whole OS kernel. Often tens of MB.
- Hundreds of containers can run on one machine.

**The trade-off (important for interviews and security):**
- Because all containers share one kernel, isolation is **weaker than a VM**. A kernel bug can affect every container on the host. That is why topic 40 onwards covers security in detail.
- A Linux container needs a Linux kernel. On Windows and macOS, Docker runs a small hidden Linux VM to get one (topic 04).

### Containers vs VMs side by side

| | Virtual machine | Container |
|---|---|---|
| What is isolated | A whole computer (virtual hardware) | A process (a fenced-off view of one kernel) |
| Has its own kernel | Yes | No, shares the host kernel |
| Typical size | GBs | MBs |
| Start time | Seconds to minutes | Milliseconds to seconds |
| Isolation strength | Strong (hardware-level) | Good, but weaker (kernel-level) |
| Density per host | Tens | Hundreds or more |
| Can run a different OS | Yes (Windows VM on Linux host) | No, a Linux container needs a Linux kernel |

**In real life they are combined.** In the cloud, your containers usually run *inside* VMs (for example AWS EC2 instances). The VM gives strong isolation between customers. Containers give fast, dense packaging inside it.

### Containers existed before Docker

The kernel features behind containers are old:
- **chroot** (1979): change what a process sees as the root folder `/`.
- **FreeBSD jails** (2000), **Solaris Zones** (2004).
- **Linux namespaces** (from 2002) and **cgroups** (2007, by Google engineers). You'll learn both in topic 02.
- **LXC** (Linux Containers, 2008): tools to create containers from those kernel features.

So Docker (2013) did **not** invent containers. Containers were hard to use, and Docker made them easy.

### What Docker actually added

1. **An image format**: a simple, layered, portable package of the app and its filesystem (topic 07).
2. **The Dockerfile**: a plain text recipe to build that image, repeatable by anyone (topic 14).
3. **Registries**: a place to push and pull images, like GitHub but for images (Docker Hub, topic 06).
4. **A simple CLI**: `docker build`, `docker push`, `docker run`. One workflow on a laptop and on a server.
5. **A standard**: Docker's formats became the **OCI** standards (OCI = Open Container Initiative, a group that keeps container formats vendor-neutral). This is why images built with Docker run on Kubernetes, Podman and others (topic 03).

### How containers fix each problem

| Problem | How containers fix it |
|---|---|
| Works on my machine | The image carries the whole user-space environment. The same image runs on the laptop, in CI and in production. |
| Dependency conflicts | Each container has its own filesystem with its own libraries. App A's Python 3.9 and App B's Python 3.12 never see each other. |
| Drift and slow setup | You don't patch containers by hand. You change the Dockerfile, build a new image and replace the container. This is called **immutable infrastructure**. |
| Wasted hardware | Many isolated containers share one host, each with CPU and memory limits. |

> **Immutable infrastructure** = you never change a running thing. You build a new version and replace the old one. If something is wrong, you roll back to the previous image.

---

## Commands / files

There are no Docker commands yet. Here is a Linux command you already know that proves the "shared kernel" idea, which you'll repeat in the lab once Docker is installed.

```bash
uname -r
```

Sample output on the host:
```
6.8.0-45-generic
```

- `uname -r` prints the **kernel release** (version) currently running.
- `6.8.0-45-generic` = kernel 6.8, Ubuntu's build number 45, the "generic" flavour.

Later, when you run `uname -r` **inside** a container, even an "Alpine" or "CentOS" container, you will get the **same number**. The container brings its own files, but never its own kernel.

---

## Hands-on lab

You don't have Docker yet (that's topic 04). Do this thought lab and a small real lab on your Linux notes knowledge.

**Lab 1: feel the conflict problem (on any Linux box or WSL)**

```bash
which python3
python3 --version
ls -l /usr/bin/python3*
```

Sample output:
```
/usr/bin/python3
Python 3.12.3
lrwxrwxrwx 1 root root      10 Apr  9 12:00 /usr/bin/python3 -> python3.12
-rwxr-xr-x 1 root root 8020928 Apr  9 12:00 /usr/bin/python3.12
```

What it shows: there is **one** `/usr/bin/python3` on the whole machine, and it points to 3.12. Any app on this machine that runs `python3` gets 3.12. If an app needs 3.9, you'd have to install it beside it and carefully point that app at it. Containers remove this whole problem: each container has its own `/usr/bin/python3`.

**Lab 2: write down the environment of an app you know**

Pick any app you've worked with. List: OS version, language version, libraries, config files, environment variables, ports. That list is exactly what a container image has to capture. Keep it. In topic 14 you'll turn a list like this into a Dockerfile.

**Lab 3 (after topic 04): prove the shared kernel**

```bash
uname -r
docker run --rm alpine uname -r
docker run --rm ubuntu uname -r
```

All three print the same kernel version.

---

## Where you'll use it

- **Interviews:** "What is the difference between a container and a VM?" is asked in almost every DevOps interview. Answer with the shared kernel, size, start time and isolation strength.
- **Architecture discussions:** choosing VMs for strong isolation (multi-tenant, untrusted code) versus containers for fast app deployment.
- **CI/CD** (Continuous Integration / Continuous Delivery: automated pipelines that build, test and deploy every change): the same image moves from build to test to production. Nobody rebuilds or reinstalls at each step.
- **Kubernetes:** Kubernetes runs containers. Everything you learn here is the foundation for it.

---

## When it breaks

At this stage, "breaking" means misunderstanding. These are the common wrong ideas:

| Wrong idea | Reality |
|---|---|
| "A container is a lightweight VM." | It's a process with a fenced-off view. No virtual hardware, no own kernel. |
| "Containers are fully secure sandboxes." | They share the kernel. A kernel exploit or a misconfiguration (like `--privileged`, a `docker run` flag that switches off most container protections) can escape. Topic 40. |
| "I can run a Windows app in a Linux container." | No. A Linux container needs a Linux kernel. (Windows containers exist, but they need a Windows kernel.) |
| "Containers fix all 'works on my machine' problems." | They fix the user-space environment. Differences in the **kernel** version, CPU architecture (amd64 vs arm64, topic 21), or external services (databases, DNS) can still bite you. |
| "Docker invented containers." | Docker made them easy and standard. The kernel features came first. |

---

## Quick recap

- Apps break when their environment differs between machines. Containers package the environment with the app.
- VMs virtualize hardware and run a full OS each. Containers are isolated processes sharing one kernel: smaller, faster, denser, but with weaker isolation.
- Docker didn't invent containers. It added an image format, the Dockerfile, registries and a simple CLI, and these became OCI standards.
- The usual approach is to replace containers rather than patch them (immutable infrastructure).

**Interview questions**

1. **Q: What is the main difference between a container and a VM?**
   A: A VM virtualizes hardware and runs its own kernel. A container is a process that shares the host kernel and is isolated by kernel features. So containers are smaller and start faster, but are less isolated.

2. **Q: Can a container run a different OS than the host?**
   A: It can run a different Linux *distribution* (an Ubuntu image on a RHEL host), because a distribution is just files. It cannot run a different *kernel*, because it uses the host's kernel.

3. **Q: Why do people say containers are "immutable"?**
   A: You don't patch a running container. You build a new image and replace the container. This removes configuration drift and makes rollback easy: just run the previous image.
