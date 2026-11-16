---
title: A Linux machine gave away its whole setup through its files
slug: linux-file-system-hunting
subtitle: Ten findings in /proc, /etc, /dev and /var/log, read from a live Ubuntu 24.04 system
tags: linux, devops, networking, security
domain: jaydeepkanzariya.hashnode.dev
cover: https://cdn.hashnode.com/uploads/covers/69f3509a909e64ad07ce72d6/7f96fa1f-bf58-4d3e-bff3-15fcfbea58ef.png
seoTitle: Linux file system hunting: 10 findings in /proc and /etc
seoDescription: Ten findings from /proc, /etc, /dev and /var/log: a routing table in backwards hex, deleted files you can still read, and an empty /boot.
enableToc: true
saveAsDraft: true
---

The default gateway on this machine is `010200C0`. That is what the kernel's routing table says, and it isn't a typo. It's `192.0.2.1`, written backwards.

I checked a live Linux VM using read-only commands and worked out what each file was telling me. The assignment was to investigate the file system like a system investigator instead of practising commands, so for every file I asked four things: what does it do, why does it exist, what problem does it solve, and what did it give away?

> **Note** — Every output below comes from one Linux VM: Ubuntu 24.04.5 LTS, kernel 6.18.44, 1 vCPU, about 3.9 GB of RAM, booted by a Firecracker init process. It is a minimal image, not a desktop install. Famous files like `auth.log` and `/boot/grub` are missing, and the absence turned out to be a finding too. Every output comes from that one machine.

## 1. The routing table is a text file, written backwards

**What it is:** `/proc/net/route` is the kernel's IPv4 routing table, generated on every read. `ip route` is a formatter on top of it.

**Why it exists, and what it solves:** programs need to ask "where do packets go?" without a dedicated API. As a file, even a shell script can ask.

```text
Iface  Destination  Gateway   Flags  Mask
eth0   00000000     010200C0  0003   00000000
eth0   000200C0     00000000  0001   00FFFFFF
```

The values are hex, and they look like garbage until you notice the byte order. The kernel prints a raw 32-bit number in the CPU's native order, which is little-endian on x86. Reverse the bytes and it reads normally:

- Gateway `010200C0` → bytes `01 02 00 C0` → `C0.00.02.01` → **192.0.2.1**
- Destination `000200C0` with mask `00FFFFFF` → **192.0.2.0/24**, the directly connected subnet
- Destination `00000000` with mask `00000000` → **0.0.0.0/0**, the default route
- Flags `0003` = `RTF_UP | RTF_GATEWAY`; `0001` = `RTF_UP` only, so no gateway needed

**What surprised me:** the kernel doesn't translate anything. It dumps its internal representation and leaves presentation to user space, a pattern that shows up across `/proc`. Also, 192.0.2.0/24 is TEST-NET-1, a range reserved for documentation (RFC 5737). I infer the network is synthetic; the file alone doesn't prove it.

**When not to use it:** in scripts, prefer `ip -j route`, which gives JSON and doesn't depend on byte-order tricks.

## 2. `/proc/net/tcp` ties an open port to its owner

**What it is:** every TCP socket the kernel knows about, one per line. Filtering for state `0A` (LISTEN) leaves one row:

```text
local_address  rem_address    st
00000000:07E8  00000000:0000  0A
```

IPs use the same little-endian quirk as finding 1, but ports are plain hex: `07E8` is **2024**. So something listens on `0.0.0.0:2024`. Reading `/proc/1/cmdline` names it:

```text
/process_api --firecracker-init --addr 0.0.0.0:2024 --max-ws-buffer-size 32768 ...
```

PID 1 owns the socket. That ties a kernel table entry to a process by reading two files, with no `ss` or `netstat`.

**Why it exists, and what it solves:** one source of truth for "what is listening here?", available even where `ss` isn't installed. Security audits start with exactly this question.

**What surprised me:** PID 1 is a custom init named `process_api`, not systemd. I come back to that in finding 10.

## 3. DNS is a two-step decision, and one file is rewritten at boot

**What they do:** `/etc/resolv.conf` says which DNS servers to ask. `/etc/nsswitch.conf` says whether to ask DNS at all, and in what order.

```text
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
options timeout:2 attempts:3

# /etc/nsswitch.conf
hosts:          files dns
```

**Why it exists, and what it solves:** name resolution is a pluggable chain. `files dns` means `/etc/hosts` is checked first and DNS only if nothing matches, so a machine can override DNS locally for testing, blocking or air-gapped use. The `options` line caps the cost of a dead server at two seconds per attempt.

**What surprised me:** `/etc/resolv.conf` is a plain 67-byte file with a modification time of today (Nov 13), while `/etc/passwd` is dated Oct 30. I infer it's rewritten at VM start. On a desktop Ubuntu this file is normally a symlink to a `systemd-resolved` stub; here there is no resolver daemon and `resolvectl` doesn't exist. `/etc/hosts` also pins one external API hostname to a fixed IP, which is the `files` entry in `nsswitch.conf` doing its job: that name never reaches DNS.

## 4. Deleted files aren't gone while a process holds them

**What it is:** `/proc/<pid>/fd` is a directory of symlinks, one per file descriptor a process has open. The experiment: create a file, open it, delete it, read it back.

```bash
echo "still here" > /tmp/ghost.txt
exec 3< /tmp/ghost.txt
rm /tmp/ghost.txt
ls -l /proc/$$/fd/3
cat /proc/$$/fd/3
```

```text
lr-x------ 1 root root 64 Nov 13 10:56 /proc/1310/fd/3 -> /tmp/ghost.txt (deleted)
still here
```

**Why it works:** `rm` removes a name, a directory entry. The inode and its data blocks are freed only when the link count hits zero and no process still has the file open.

**What problem it solves:** this is the classic "disk is full but `du` shows nothing" mystery. A service keeps writing to a log someone deleted, and `/proc/*/fd` shows the culprit. You can recover the contents with `cp`.

**Two related surprises:** `/proc/self` is a symlink that resolves to the PID of whichever process reads it (`-> 1343`, then `-> 1344` on the next call), which is why `/dev/stdout` can be `-> /proc/self/fd/1`. And `/proc/self/status` shows the process's privileges:

```text
CapEff: 000001fffeffffff
CapBnd: 000001fffeffffff
```

A full mask on this kernel would be `000001ffffffffff` (capability bits 0 to 40). Compare the two and exactly one bit is cleared: bit 24, `CAP_SYS_RESOURCE`. Root here isn't one all-powerful switch. It's a set of capabilities, and the environment removed one from the bounding set so it can't be regained.

## 5. Why passwords moved out of a world-readable file

```text
-rw-r--r-- 1 root root    1111 /etc/passwd
-rw-r----- 1 root shadow   609 /etc/shadow
```

**What they do:** `passwd` lists accounts (name, UID, home, shell). `shadow` holds password hashes.

**Why two files:** hashes used to live in `passwd`. But `passwd` must be world-readable, because every `ls -l` uses it to turn UIDs into names, so anyone could copy the hashes and crack them offline. Moving them to a file only root and the `shadow` group can read closes that hole.

**What surprised me:**

- Of 23 accounts, 19 have `nologin` or `false` as their shell. They are service users, there so daemons run with minimal privilege.
- root's entry in `shadow` is `root:*`. No password can ever match `*`, so root has no password login at all.
- The `UID_MIN 1000` line in `/etc/login.defs` explains why human UIDs start at 1000.

That leaves a puzzle. If `shadow` is unreadable, how does a normal user change their own password? The answer is the SUID bit, which makes a program run as its owner:

```text
$ find /usr/bin -perm -4000 -type f
fusermount3  chfn  chsh  gpasswd  mount  newgrp  passwd  su  umount
```

`passwd` runs as root, edits `shadow`, and only lets you change your own entry. Every SUID binary is a deliberate privilege escalation, so audits start by listing them. This list is short and standard, which is what you want to see. `/tmp` is `drwxrwxrwt`: the trailing `t` is the sticky bit, so everyone can write there but only delete their own files.

## 6. `/dev` holds devices that don't exist

```text
crw-rw-rw- 1 root root 1, 3 /dev/null
crw-rw-rw- 1 root root 1, 5 /dev/zero
crw-rw-rw- 1 root root 1, 8 /dev/random
crw-rw-rw- 1 root root 1, 9 /dev/urandom
```

**What it is:** `c` means character device. There is no file size, only two numbers: major `1` (the kernel's memory-devices driver) and minor `3`, `5`, `8` or `9` (which sub-device). Opening `/dev/null` never touches a disk. The kernel routes the call to that driver.

**Why it exists, and what it solves:** one interface (open, read, write, close) for disks, terminals, randomness and the void. A program needs no special code to discard output or read random bytes.

**What surprised me:** from `/proc/mounts`:

```text
devtmpfs /dev devtmpfs rw,relatime,size=2030944k,nr_inodes=507736,mode=755 0 0
/dev/vda / ext4 rw,relatime,discard,... 0 0
```

`/dev` is a RAM-backed filesystem the kernel fills at boot, not something stored on disk. And the root disk is `vda`, not the usual `sda`. The `vd*` naming is what virtio block devices get, so this is a virtual machine.

## 7. `/proc/sys` is a writable control panel

Unlike most of `/proc`, the files under `/proc/sys` accept writes, and a write changes kernel behaviour immediately.

```text
net/ipv4/ip_forward = 0
net/ipv4/ip_local_port_range = 32768  60999
kernel/randomize_va_space = 2
kernel/pid_max = 32768
```

**What problem it solves:** you can tune a running kernel with no recompile and no reboot.

- `ip_forward = 0`: this machine won't forward packets between interfaces, so it isn't a router. Setting it to `1` makes it one, a step Docker and VPN software take.
- `ip_local_port_range`: outgoing connections get a source port in 32768–60999, about 28,000 ports. That caps simultaneous connections from one source IP to the same destination IP and port, not connections overall.
- `randomize_va_space = 2`: full ASLR. Stack, heap and libraries load at randomised addresses each run. `cat`'s first mapping started at `0x55fa18524000` in `/proc/self/maps`, and the documented behaviour is that this changes per run.
- `pid_max = 32768`: process IDs wrap after this.

**When not to use it:** writes here vanish at reboot. For a permanent setting, put it in `/etc/sysctl.d/` and let `sysctl` apply it at boot.

## 8. Network and cgroup files describe the machine

Each network interface is a directory of one-value files under `/sys/class/net`:

```text
eth0  mtu=1400   mac=02:fc:00:00:00:01  state=up
lo    mtu=65536  mac=00:00:00:00:00:00  state=unknown
ifb0  mtu=1500   state=down
ifb1  mtu=1500   state=down
```

**What problem it solves:** a script can read one value (`cat .../mtu`) without parsing command output.

**What surprised me:**

- **MTU 1400, not 1500.** I infer the traffic passes through a tunnel or overlay that needs room for its own headers. The file doesn't prove it.
- **A MAC starting with `02:`.** That sets the locally administered bit, so the address was assigned by software, not burned into hardware.
- **`ifb0` and `ifb1`.** Intermediate Functional Block devices, used for shaping incoming traffic.

`/proc/self/cgroup` adds a layer:

```text
8:pids:/process_api
4:memory:/process_api/50afe65228bafa4b48a84cc5fd93990b
```

Control groups let the kernel cap CPU, memory and process counts for a group of processes, so a runaway workload hurts only its own group. The numbered per-controller lines, plus `9:name=systemd:/`, are the older cgroup v1 layout; v2 would show a single `0::` line. Here both appear, so the system is mixing them (my reading of the output, not something I confirmed in the kernel config).

Put findings 2, 6 and 8 together and the architecture reconstructs itself without documentation: a Firecracker-style microVM with a custom PID 1, virtio disk, software MAC and a reduced MTU, with its workload in its own cgroup.

## 9. The logs read like a build diary

```text
-rw-r--r-- 1 root root  60K Oct 24 14:05 bootstrap.log
-rw-r--r-- 1 root root 581K Oct 30 01:07 dpkg.log
-rw-rw---- 1 root utmp    0 Oct 24 14:05 btmp
-rw-rw-r-- 1 root utmp    0 Oct 24 14:05 wtmp
drwxr-sr-x 2 root systemd-journal 4.0K Oct 30 01:04 journal
```

**What they do:** `bootstrap.log` and `dpkg.log` record how the image was built and which packages were installed. `wtmp` and `btmp` record successful and failed logins, and `last` reads them. `journal/` is where `systemd-journald` stores its binary logs.

**Why they exist:** after something goes wrong you need a timeline of who did what, and when.

**What surprised me:** the first line of `bootstrap.log` shows a package index being fetched from an internal mirror into `/build/chroot` on Oct 24, so the image was bootstrapped in a chroot. The last `dpkg.log` entry, `ca-certificates-java`, is dated Oct 30, so a second layer of packages came six days later. Meanwhile `wtmp`, `btmp` and `lastlog` are all zero bytes, and `last` can only say `wtmp begins Sat Oct 24 14:05:10 2026`. Nobody has ever logged in interactively; programs drive this machine.

The `journal/` directory exists but holds no journal files, because the journal service never ran. And there is no `/var/log/auth.log` at all. That file is written by `rsyslog`, and `dpkg -l rsyslog` here finds no such package. Without it, auth events live only in the journal, which is empty on this VM.

## 10. `/boot` is empty, and systemd is configured but not running

```text
$ ls -lh /boot
total 0
$ systemctl cat cron
System has not been booted with systemd as init system (PID 1). Can't operate.
```

**What `/boot` normally holds:** the compressed kernel (`vmlinuz-*`), an initial RAM filesystem (`initrd.img-*`), the kernel build config and GRUB's menu. The initramfs solves a chicken-and-egg problem: the driver needed to read the root disk may live on that disk, so a tiny RAM filesystem carries only enough drivers to mount the real root.

**What surprised me:** none of it is here. The running kernel is `6.18.44-fc-v445`, but there is no `/boot/config-6.18.44-fc-v445` and no bootloader. As I understand Firecracker's boot model, the host loads the kernel image directly into the guest, so the guest filesystem never needs a bootloader. That comes from Firecracker's documented design, not from anything I could verify inside the VM.

The systemd side is the mirror image: the files are all there and the engine is missing. Unit files sit in `/lib/systemd/system/`, and "enabled" units are symlinks in `/etc/systemd/system/*.wants/`:

```text
timers.target.wants/apt-daily.timer -> /lib/systemd/system/apt-daily.timer
multi-user.target.wants/e2scrub_reap.service -> /lib/systemd/system/e2scrub_reap.service
```

Enabling a service is creating a symlink, and disabling it is removing one. That is also why `/etc` is the place for overrides: `systemd.unit(5)` documents that `/etc/systemd/system` is searched before `/lib`, so your changes win and survive package upgrades. `apt-daily.timer` shows another design detail:

```ini
[Timer]
OnCalendar=*-*-* 6,18:00
RandomizedDelaySec=12h
Persistent=true
```

The random delay of up to 12 hours exists so millions of Ubuntu machines don't hit the update mirrors at the same second. On this VM the timer never fires, because PID 1 is `process_api`.

## Smaller finds

- **A fully commented config means "all defaults."** After stripping comments, `/etc/systemd/journald.conf` contains only `[Journal]`. In systemd config files the commented lines document the compiled-in defaults, and you uncomment one to change it.
- **`/etc/shells` is a whitelist.** It lists seven shells (`sh`, `bash`, `rbash`, `dash` and their `/usr/bin` aliases). `chsh` only accepts shells from that list, and some services refuse users whose shell isn't on it.
- **`/dev/shm` is shared memory in RAM.** It's a `tmpfs` of 3.9 GB, equal to this VM's RAM size, and its contents disappear on reboot.
- **The timezone is one symlink:** `/etc/localtime -> /usr/share/zoneinfo/Etc/UTC`.

## What the files had in common

Almost every question I asked, the kernel answered with a file: routes, sockets, privileges, devices, tuning knobs, resource limits. Commands like `ip`, `ss`, `sysctl` and `systemctl` are mostly friendly readers for those files. And the absences were as informative as the contents: no `/boot` content, no journal, no `auth.log`, no running systemd, each one a clue about how the image was built.

The limitation is real. All of this comes from one minimal VM, so I can't say how a stock Ubuntu desktop differs beyond what the docs describe. On yours, what does `/boot` hold, and does `grep CONFIG_EXT4_FS /boot/config-$(uname -r)` print `=y` or `=m`? I'd like to compare.
