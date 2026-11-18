# 41a · Seccomp, AppArmor/SELinux and What `--privileged` Turns Off

> **Level:** ★★★ advanced.
>
> **You will learn:** the second half of container confinement: **seccomp** (system-call filtering: Docker's default profile, custom profiles, and how to find what an app needs), **LSMs** (AppArmor's path-based `docker-default` profile and SELinux's `container_t` labels with MCS categories), **masked and read-only paths** in `/proc` and `/sys`, a method to find **which layer** denied an action, and exactly what `--privileged` switches off.
>
> **You need:** topic 41 (users, capabilities, no-new-privileges), topic 25 (SELinux labels on mounts).

---

## What it is

| Mechanism | Question it answers | Docker's default |
|---|---|---|
| **Seccomp** | Which **system calls** may the process make? | A default allowlist profile that blocks ~40+ dangerous calls |
| **LSM** (Linux Security Module): AppArmor or SELinux | Which **files and resources** may it touch, according to a central policy? | AppArmor `docker-default` (Ubuntu/Debian), or SELinux type `container_t` (RHEL/Fedora) |
| **Masked / read-only paths** | Which sensitive kernel interfaces in `/proc` and `/sys` are hidden or frozen? | A built-in list |

These work **independently** of capabilities (topic 41). An action must pass **all** of them.

---

## Why it exists

Capabilities control *privileged* actions. But many kernel exploits use **ordinary-looking system calls** in unusual ways (creating user namespaces, keyring operations, `bpf`, `userfaultfd`). Seccomp removes those entry points. LSMs add a **policy** that even root can't override: "processes of type `container_t` may never touch files labelled for the host". Together they've stopped several real container escapes that capabilities alone would not have.

---

## Analogy

Continuing the hotel badge from topic 41:
- **Seccomp** = a list of **actions** forbidden to all staff, whatever doors their badge opens ("nobody may switch off the fire alarm or open the gas main").
- **LSM** = the security office's **rulebook**, checked at every door: "kitchen staff never enter the vault", even if a kitchen worker somehow holds a key.
- **`--privileged`** = a master badge, no rulebook and no forbidden-actions list.

---

## How it works inside

### 1. Seccomp

**Seccomp** (secure computing mode) lets a process install a **filter** (a small BPF program) that the kernel runs on **every system call** the process makes. The filter decides: allow, return an error, kill the process, log, or notify a supervisor. The filter is inherited by children and can't be removed.

Docker's **default profile**:
- is an **allowlist**: around 300+ syscalls are allowed, and everything else returns `EPERM` ("operation not permitted"),
- blocks dangerous calls such as `mount`/`umount2`, `reboot`, `kexec_load`, `init_module`/`finit_module`/`delete_module` (kernel modules), `swapon`/`swapoff`, `clock_settime`/`settimeofday`, `acct`, `add_key`/`keyctl`/`request_key` (kernel keyring), `open_by_handle_at` (used in a famous early container escape), `bpf`, `perf_event_open`, `userfaultfd`, and `unshare`/`clone` with namespace flags (creating new namespaces, especially **user namespaces**). On older kernels it also blocks `ptrace`,
- **adapts to capabilities**: if you `--cap-add SYS_ADMIN`, the profile also allows the syscalls that capability needs.

Most apps never make these calls, but exploits often do. That's why seccomp is cheap protection with high value.

Options:
- `--security-opt seccomp=/path/profile.json` = a custom profile (usually the default with changes).
- `--security-opt seccomp=unconfined` = **no filter**. Avoid.
- `/proc/<pid>/status` → `Seccomp: 2` = a filter is active (topic 11).

Profile shape (shortened):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    { "names": ["read", "write", "openat", "close", "futex", "epoll_wait", "..."], "action": "SCMP_ACT_ALLOW" }
  ]
}
```

- `defaultAction: SCMP_ACT_ERRNO` + `defaultErrnoRet: 1` = anything not listed fails with error 1 (`EPERM`).
- To build a tight custom profile: record which syscalls the app really uses (`strace -f -c`, or tools like Inspektor Gadget's seccomp advisor), allow only those, test under real load, and keep the default profile as a fallback. Worth it for high-risk, internet-facing services.

### 2. LSMs: AppArmor and SELinux

An **LSM** is a framework for **mandatory access control** (MAC): the kernel enforces rules from a **policy**, regardless of file permissions or root. Usually only one of the two is active on a host. `docker info` shows which (`Security Options`).

**AppArmor** (Ubuntu, Debian, SUSE) is **path-based**:
- Docker loads a profile called **`docker-default`** into every container. Among other things, it denies writes to `/proc/sys/**` (except a few), `/proc/sysrq-trigger`, most of `/sys/**`, **mounting**, and access to `/proc/kcore`.
- Custom profile: write it, load it with `apparmor_parser -r -W profile`, then use `--security-opt apparmor=my-profile`. `apparmor=unconfined` disables it (avoid).
- Check a process: `cat /proc/<pid>/attr/current` → `docker-default (enforce)`.
- Denials appear in the kernel log: `dmesg | grep -i apparmor` → `apparmor="DENIED" operation="mount" profile="docker-default"`.

**SELinux** (RHEL, Fedora, CentOS Stream, Rocky) is **label-based**:
- Every process and file has a **label** (`user:role:type:level`). Containers run as type **`container_t`** with **MCS categories** (MCS = Multi-Category Security), such as `s0:c123,c456`, unique per container. Files they may use are labelled `container_file_t` with matching categories.
- Effect: even if two containers both run as root, container A (`c123,c456`) **can't read** container B's files (`c78,c910`), and neither can touch host files with other labels. That's why bind mounts need `:z`/`:Z` (topic 25).
- Options: `--security-opt label=type:my_container_t`, `label=level:s0:c100,c200`, `label=disable` (avoid).
- Denials: `ausearch -m avc -ts recent` (AVC = Access Vector Cache, SELinux's decision log).

### 3. Masked and read-only paths

Docker hides or write-protects sensitive kernel interfaces inside containers:
- **Masked** (replaced with an empty file or an empty tmpfs): `/proc/kcore`, `/proc/keys`, `/proc/timer_list`, `/proc/sched_debug`, `/proc/acpi`, `/sys/firmware`, `/proc/scsi`…
- **Read-only**: `/proc/sys`, `/proc/sysrq-trigger`, `/proc/irq`, `/proc/bus`, `/proc/fs`.
- `--security-opt systempaths=unconfined` removes this (avoid).

### 4. Which layer said no? A method

When something fails with "Operation not permitted" or "Permission denied", test one layer at a time, **on a lab machine only**:

| Test | If it now works, the blocker was… |
|---|---|
| Add the capability the action needs (`--cap-add X`) | A capability (topic 41) |
| `--security-opt seccomp=unconfined` | Seccomp |
| `--security-opt apparmor=unconfined` / `label=disable` | AppArmor / SELinux |
| Check `dmesg` / `ausearch` for DENIED/AVC lines | The LSM (no test needed) |
| Remove `--read-only` / check the path is in the masked list | Read-only rootfs or a masked path |

Then fix it **narrowly**: add one capability, a custom seccomp profile that allows one more syscall, or an LSM policy tweak. Never ship with `unconfined`.

### 5. What `--privileged` turns off

| Protection | Normal | `--privileged` |
|---|---|---|
| Capabilities | 14 | **All** |
| Seccomp | Default profile | **Off** |
| AppArmor/SELinux | Confined | **Off/unconfined** |
| Devices | Minimal `/dev` | **All host devices** |
| Masked/read-only paths | On | **Off** |
| `/sys` | Read-only | Read-write |

That's why `--privileged` = root on the host (topic 40). Use `--cap-add`/`--device` instead (topic 41).

---

## Commands / files

### Seccomp in action

```bash
docker run --rm alpine sh -c 'unshare -U id; echo "exit=$?"'
docker run --rm --security-opt seccomp=unconfined alpine sh -c 'unshare -U id; echo "exit=$?"'
docker run --rm alpine grep Seccomp /proc/self/status
```

```
unshare: unshare(0x10000000): Operation not permitted
exit=1
uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
exit=0
Seccomp:        2
Seccomp_filters:        1
```

- The default profile blocks creating a **user namespace** (`unshare -U`), a common first step in kernel exploits. Unconfined allows it. (On hosts whose AppArmor restricts unprivileged user namespaces, the second command may still fail. That's another layer saying no.)

Get Docker's default profile to customise:

```bash
curl -fsSLo default-seccomp.json https://raw.githubusercontent.com/moby/profiles/main/seccomp/default.json
grep -c '"names"' default-seccomp.json
```

Custom profile that additionally blocks `chmod` (a teaching example):

```bash
python3 - <<'EOF'
import json
p = json.load(open("default-seccomp.json"))
for rule in p["syscalls"]:
    rule["names"] = [n for n in rule["names"] if n not in ("chmod", "fchmod", "fchmodat", "fchmodat2")]
json.dump(p, open("noch.json", "w"))
EOF
docker run --rm --security-opt seccomp=./noch.json alpine chmod 777 /tmp
```

```
chmod: /tmp: Operation not permitted
```

### AppArmor / SELinux status

```bash
docker info --format '{{.SecurityOptions}}'
docker run -d --name conf nginx:1.27-alpine
PID=$(docker inspect -f '{{.State.Pid}}' conf)
sudo cat /proc/$PID/attr/current; echo
docker run --rm alpine sh -c 'echo 1 > /proc/sys/kernel/sysrq'
```

```
[name=apparmor name=seccomp,profile=builtin name=cgroupns]
docker-default (enforce)
sh: can't create /proc/sys/kernel/sysrq: Read-only file system
```

On SELinux hosts:

```bash
ps -eZ | grep container_t | head -2
```

```
system_u:system_r:container_t:s0:c123,c456 91234 ? 00:00:00 nginx
system_u:system_r:container_t:s0:c78,c910  91567 ? 00:00:00 postgres
```

- Different MCS categories per container.

### Layers blocking independently: `mount`

```bash
docker run --rm alpine sh -c 'mount -t tmpfs none /mnt && echo mounted'
docker run --rm --cap-add SYS_ADMIN alpine sh -c 'mount -t tmpfs none /mnt && echo mounted'
docker run --rm --cap-add SYS_ADMIN --security-opt apparmor=unconfined alpine sh -c 'mount -t tmpfs none /mnt && echo mounted'
sudo dmesg | grep -i 'apparmor="DENIED"' | tail -1
```

```
mount: permission denied (are you root?)
mount: permission denied (are you root?)
mounted
[ 9123.45] audit: type=1400 apparmor="DENIED" operation="mount" ... profile="docker-default" name="/mnt/" ...
```

- No capability → denied. Capability added, but AppArmor's `docker-default` still denies mounting. Only with AppArmor off does it work. Each layer blocks on its own. **Never run production like the third command.**

### Compose equivalents

```yaml
services:
  web:
    security_opt:
      - seccomp=./seccomp-web.json
      - apparmor=docker-default
      # SELinux: - label=type:my_container_t
```

---

## Hands-on lab

1. Run the seccomp `unshare -U` demo, and read `Seccomp:` in `/proc/self/status`.
2. Build the `noch.json` profile and confirm `chmod` is blocked.
3. Run the `mount` demo and find the AppArmor DENIED line in `dmesg`. (On SELinux hosts, look for AVC lines with `ausearch -m avc -ts recent`.)
4. Use the "which layer said no?" table on a failing command of your choice.
5. Compare `grep -E 'CapEff|Seccomp' /proc/self/status` in a normal container and a `--privileged` one.
6. Clean up: `docker rm -f conf; rm -f default-seccomp.json noch.json`

---

## Where you'll use it

- **Hardening baselines:** keep the default seccomp/AppArmor/SELinux on everywhere, with custom seccomp profiles for high-risk services.
- **Kubernetes:** `seccompProfile: { type: RuntimeDefault }` (required by the "restricted" Pod Security Standard), AppArmor and SELinux options in `securityContext`.
- **Troubleshooting:** "works on Ubuntu, fails on RHEL" is often SELinux; "fails only in Docker" is often seccomp/AppArmor.
- **Security reviews:** `seccomp=unconfined`, `apparmor=unconfined`, `label=disable` and `privileged: true` are red flags that need written justification.

---

## When it breaks

| Error | Usual layer | Fix |
|---|---|---|
| `Operation not permitted` on `mount`, `unshare`, `keyctl`, `bpf`, `perf_event_open` | Seccomp (and/or a missing capability) | Narrow custom profile; never `unconfined` in production |
| `Permission denied` writing `/proc/sys/...` or `/sys/...` | AppArmor / read-only paths | Use `--sysctl` for namespaced settings; host settings on the host |
| `Permission denied` on files even as root (RHEL/Fedora) | SELinux | `ausearch -m avc`; `:Z`/`:z` on mounts (topic 25) |
| `apparmor="DENIED"` lines in `dmesg` | AppArmor | Adjust/replace the profile for that container |
| App works with `--privileged` only | Some combination of the above | Use the layer-by-layer method; grant the minimum |
| Custom seccomp profile breaks the app under load | A syscall only used on some code paths | Test real traffic; use `SCMP_ACT_LOG` while profiling |

---

## Quick recap

- **Seccomp** filters system calls. Docker's default allowlist blocks mount, modules, keyring, kexec, bpf and new user namespaces. Customise it, don't disable it.
- **AppArmor** (`docker-default`, path rules) or **SELinux** (`container_t` + per-container MCS categories) confine even root, by policy.
- **Masked/read-only paths** protect `/proc` and `/sys`.
- To debug, find **which layer** denied the action (capability → seccomp → LSM → read-only/masked), then fix it narrowly.
- **`--privileged` turns all of this off** (plus all capabilities and devices) = host root.

**Interview questions**

1. **Q: What is seccomp in Docker?**
   A: A kernel feature that filters the system calls a process may make. Docker applies a default allowlist profile that blocks dangerous syscalls like mount, module loading, kexec, keyring operations and creating user namespaces. Custom profiles are applied with `--security-opt seccomp=`.

2. **Q: How do AppArmor and SELinux differ in how they confine containers?**
   A: AppArmor uses path-based profiles (Docker's `docker-default` denies mounts and writes to sensitive `/proc` and `/sys` paths). SELinux labels every process and file. Containers run as `container_t` with unique MCS categories, so they can't access each other's or the host's files even as root.

3. **Q: A container action fails with "Operation not permitted". How do you find the cause?**
   A: On a test machine, test each layer separately: add the specific capability, try seccomp unconfined, try the LSM unconfined, and check `dmesg`/`ausearch` for denials. Once the layer is identified, grant the narrowest possible exception.
