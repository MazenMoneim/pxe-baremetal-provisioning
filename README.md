<h1 align="center">
PXE Provisioning with MACs and Kickstart Files
</h1>  



<p align="center">
  <img src="https://github.com/user-attachments/assets/143367fb-07d3-4ed6-8be6-e4bf6c60d7bf" width="200" alt="PXE Logo" />
</p>


<p align="center">  
  <img src="https://img.shields.io/badge/-PXE%20Boot-green?style=flat" />  
  <img src="https://img.shields.io/badge/-TFTP-blue?style=flat" />  
  <img src="https://img.shields.io/badge/-DHCP-orange?style=flat" />  
  <img src="https://img.shields.io/badge/-Kickstart-red?style=flat" /> 
  <img src="https://img.shields.io/badge/-NFS-purple?style=flat" />
</p> 

<hr/>

> Demo for automating bare-metal provisioning of RHEL 9 servers via Zero Touch Provisioning (ZTP), PXE, and Kickstart.

Boot and install **any number of servers** with zero manual interaction. A single
control node hands out addresses, serves a network bootloader, and drives an
unattended RHEL 9 install through Kickstart — the same pattern used to provision
bare-metal fleets in production, built here from the individual, dedicated daemons
(rather than an all-in-one tool like dnsmasq) so every moving part is visible.

**Topics:** `pxe` `ztp` `kickstart` `rhel9` `bare-metal` `provisioning` `dhcp` `bind` `tftp` `automation`

---

## Table of contents

- [What is Zero Touch Provisioning here?](#what-is-zero-touch-provisioning-here)
- [Architecture](#architecture)
- [How the daemons hand off (mental model)](#how-the-daemons-hand-off-mental-model)
- [Prerequisites](#prerequisites)
- [Lab network layout](#lab-network-layout)
- [MAC / IP / name plan](#mac--ip--name-plan)
- [Setup](#setup)
  - [1. Install the packages](#1-install-the-packages)
  - [2. DHCP — ISC dhcpd](#2-dhcp--isc-dhcpd)
  - [3. DNS — BIND](#3-dns--bind)
  - [4. TFTP server](#4-tftp-server)
  - [5. Install tree + boot kernel/initrd (HTTP)](#5-install-tree--boot-kernelinitrd-http)
  - [6. PXELINUX per-MAC menus](#6-pxelinux-per-mac-menus)
  - [7. Kickstart files](#7-kickstart-files)
  - [8. Firewall](#8-firewall)
  - [9. Start and verify](#9-start-and-verify)
- [Booting a target](#booting-a-target)
- [Adding more nodes (any number of servers)](#adding-more-nodes-any-number-of-servers)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [ISC dhcpd end-of-life note](#isc-dhcpd-end-of-life-note)

---

## What is Zero Touch Provisioning here?

Zero Touch Provisioning (ZTP) means a machine goes from **powered-off bare metal** to a
**fully installed, configured, reachable OS** with no human touching it after the power
button. In this project that is achieved with three classic building blocks:

- **PXE** (Preboot eXecution Environment) — firmware on the target requests a network
  boot; a DHCP server tells it where to fetch a bootloader, and a TFTP server delivers it.
- **Kickstart** — RHEL's unattended-install answer file. It pre-answers every installer
  question (disks, users, packages, network) so Anaconda runs start to finish on its own.
- **Dedicated network services** — DHCP, DNS, TFTP, and HTTP each run as their own
  daemon, exactly as they would in production.

The result: rack a server (or clone a VM), set it to network-boot, power it on, and it
installs itself.

---

## Architecture

A single **control node** runs every service. Targets have no OS and only need a NIC on
the provisioning network and "network boot first" in their firmware.

| Role | Daemon | Package | Port |
|------|--------|---------|------|
| DHCP | ISC `dhcpd` | `dhcp-server` | UDP 67 |
| DNS  | BIND `named` | `bind` | UDP/TCP 53 |
| TFTP | `in.tftpd` (socket-activated) | `tftp-server` | UDP 69 |
| Install tree + Kickstart | `httpd` | `httpd` | TCP 80 |
| Bootloader files | (from `syslinux`) | `syslinux` | — |

```
                 ┌───────────────────────── control-plane (192.168.200.10) ───────────────────────────┐
                 │                                                                                    │
   power on      │   dhcpd ──► "here's your IP + next-server + pxelinux.0"                            │
  ┌────────┐     │   tftpd ──► pxelinux.0, per-MAC menu, vmlinuz, initrd.img                          │
  │ target │◄────┤   httpd ──► RHEL 9 install tree + ks-nodeN.cfg                                     │
  │ (no OS)│     │   named ──► forward/reverse DNS for lab.example.com                                │
  └────────┘     │                                                                                    │
   VMnet1        │   ens224 = 192.168.200.10  (Host-only, the provisioning network)                   │
 192.168.200.x   │   ens160 = 192.168.245.x   (NAT, internet for dnf only)                            │
                 └────────────────────────────────────────────────────────────────────────────────────┘
```

The control node has **two NICs on purpose**: `ens224` on the isolated provisioning
network where all PXE traffic happens, and `ens160` on NAT purely so the control node
itself can reach the internet to install packages. DHCP is deliberately bound to
`ens224` only, so it never leaks onto anything else.

---

## How the daemons hand off (mental model)

The whole point of using separate daemons instead of dnsmasq is that you can *see* the
handoff. The boot of one target goes:

1. **dhcpd** answers the client's DHCP request with an IP **plus** two PXE-specific
   options: `next-server` (where TFTP is) and `filename` (`pxelinux.0`). Those two lines
   are what turn ordinary DHCP into PXE.
2. The client TFTPs `pxelinux.0` from **tftpd**, which then requests a config file named
   after the client's own MAC (`01-<mac>`), then pulls `vmlinuz` + `initrd.img`.
3. The installer boots and uses its kernel arguments to fetch the **repo** and the
   **Kickstart** file from **httpd**.
4. **named** provides forward and reverse name resolution so the installed nodes have
   real FQDNs — the production touch a dnsmasq lab usually skips.

Each daemon does one job, is configured in its own file, logs to its own place, and can
be restarted independently. That separation is the lesson.

---

## Prerequisites

- A **control node**: RHEL 9 / Rocky 9 / CentOS Stream 9 VM (this is `control-plane`).
- One or more **target VMs** with **no OS**, on the same isolated virtual network.
- A **RHEL 9 DVD ISO** available to the control node (used to publish the install tree).
- The control node's provisioning NIC set to a **static** address (it is the DHCP
  server; it cannot itself be a DHCP client on the network it serves).

**Target VM requirements** (each one):
- NIC attached to the **provisioning network** (VMnet1 / Host-only in this lab).
- A **pinned static MAC** (so DHCP reservations and per-MAC Kickstart are stable).
- **Network / PXE first** in the boot order.
- Firmware set to **BIOS** (this guide uses BIOS; a UEFI variant is noted at the end).
- **At least 2 GB RAM during install** — the installer stages a ~750 MB root image
  (`install.img`) into a RAM disk; too little memory fails with "No space left on
  device". 4 GB is comfortable. (See troubleshooting.)
- A single disk is simplest. Note the disk **device name** — on VMware NVMe controllers
  it is `nvme0n1`, not `sda`. Your Kickstart must match it.

---

## Lab network layout

Two virtual networks on the hypervisor:

| Network | Type | Subnet | Purpose |
|---------|------|--------|---------|
| VMnet1 | Host-only | `192.168.200.0/24` | Isolated provisioning network (all PXE traffic) |
| VMnet8 | NAT | `192.168.245.0/24` | Internet for the control node only |

**Critical:** disable the hypervisor's own DHCP on the provisioning network (VMnet1),
because the control node runs the DHCP server. Two DHCP servers on one segment fight and
PXE fails intermittently. In VMware's Virtual Network Editor, uncheck *"Use local DHCP
service to distribute IP addresses to VMs"* for VMnet1.

Control node domain: **`lab.example.com`** on `192.168.200.0/24`.

---

## MAC / IP / name plan

| Target | Static MAC | IP | FQDN | Kickstart |
|--------|------------|----|------|-----------|
| node1 | `00:50:56:00:02:01` | 192.168.200.101 | node1.lab.example.com | ks-node1.cfg |
| node2 | `00:50:56:00:02:02` | 192.168.200.102 | node2.lab.example.com | ks-node2.cfg |
| node3 | `00:50:56:00:02:03` | 192.168.200.103 | node3.lab.example.com | ks-node3.cfg |

Control node itself: `controlplane.lab.example.com` → `192.168.200.10`.

Static MACs use VMware's static range `00:50:56:00:00:00`–`00:50:56:3F:FF:FF`. Set each
target's MAC in **VM Settings → Network Adapter → Advanced**, or in its `.vmx`:

```
ethernet0.addressType   = "static"
ethernet0.address       = "00:50:56:00:02:01"
ethernet0.connectionType = "hostonly"
ethernet0.vnet          = "VMnet1"
bios.bootOrder          = "ethernet0"
```

---

## Setup

### 1. Install the packages

The control node reaches the internet over NAT, so `dnf` works normally.

```bash
sudo dnf install -y dhcp-server bind bind-utils tftp-server syslinux httpd
```

`syslinux` isn't a service — it just provides the BIOS network bootloader files
(`pxelinux.0` and the `.c32` modules) that you'll copy into the TFTP root.

---

### 2. DHCP — ISC dhcpd

DHCP does double duty: it leases addresses **and** carries the PXE pointers. The two
lines that matter for PXE are `next-server` (the TFTP host) and `filename` (the boot
file). Fixed `host` reservations tie each target's MAC to a known IP, which is what makes
the rest of the pipeline deterministic.

`/etc/dhcp/dhcpd.conf`:

```
# Global
authoritative;
ddns-update-style none;
default-lease-time 600;
max-lease-time 7200;

option domain-name "lab.example.com";
option domain-name-servers 192.168.200.10;

# The PXE lab subnet
subnet 192.168.200.0 netmask 255.255.255.0 {
  range 192.168.200.50 192.168.200.99;        # dynamic pool for unknown clients
  option routers 192.168.200.10;              # optional on an isolated net
  option broadcast-address 192.168.200.255;

  # --- PXE / BIOS boot ---
  next-server 192.168.200.10;                 # TFTP server (this host)
  filename "pxelinux.0";                      # BIOS bootloader
}

# --- Fixed reservations keyed by MAC ---
host node1 { hardware ethernet 00:50:56:00:02:01; fixed-address 192.168.200.101; option host-name "node1"; }
host node2 { hardware ethernet 00:50:56:00:02:02; fixed-address 192.168.200.102; option host-name "node2"; }
host node3 { hardware ethernet 00:50:56:00:02:03; fixed-address 192.168.200.103; option host-name "node3"; }
```

Bind dhcpd to the provisioning NIC only, so it can never answer DHCP on the NAT side.
Edit `/etc/sysconfig/dhcpd`:

```
DHCPDARGS=ens224
```

Validate the config **before** starting — a syntax error stops the whole pipeline:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl enable --now dhcpd
```

---

### 3. DNS — BIND

DNS isn't strictly required to PXE-boot, but it's the production touch: nodes get real
forward and reverse names. BIND here is **authoritative** for `lab.example.com` and
**forwards** anything else to the NAT gateway so the control node can still resolve
internet names.

`/etc/named.conf` (key parts):

```
options {
    listen-on port 53 { 127.0.0.1; 192.168.200.10; };
    directory       "/var/named";
    allow-query     { localhost; 192.168.200.0/24; };
    recursion yes;
    forwarders { 192.168.245.2; };   # VMware NAT gateway = upstream resolver
    dnssec-validation no;            # simpler for an isolated lab
};

zone "lab.example.com" IN {
    type master;
    file "lab.example.com.zone";
    allow-update { none; };
};

zone "200.168.192.in-addr.arpa" IN {
    type master;
    file "200.168.192.rev";
    allow-update { none; };
};
```

Forward zone — `/var/named/lab.example.com.zone`:

```
$TTL 86400
@   IN SOA  controlplane.lab.example.com. admin.lab.example.com. (
        2025010101 ; serial   <-- bump on EVERY edit or changes won't load
        3600 1800 604800 86400 )
    IN NS   controlplane.lab.example.com.

controlplane IN A 192.168.200.10
node1        IN A 192.168.200.101
node2        IN A 192.168.200.102
node3        IN A 192.168.200.103
```

Reverse zone — `/var/named/200.168.192.rev`:

```
$TTL 86400
@   IN SOA  controlplane.lab.example.com. admin.lab.example.com. (
        2025010101 ; serial
        3600 1800 604800 86400 )
    IN NS   controlplane.lab.example.com.

10   IN PTR controlplane.lab.example.com.
101  IN PTR node1.lab.example.com.
102  IN PTR node2.lab.example.com.
103  IN PTR node3.lab.example.com.
```

Fix permissions, validate both zones, start, and point the control node at itself:

```bash
sudo chown root:named /var/named/lab.example.com.zone /var/named/200.168.192.rev
sudo named-checkconf
sudo named-checkzone lab.example.com /var/named/lab.example.com.zone
sudo named-checkzone 200.168.192.in-addr.arpa /var/named/200.168.192.rev
sudo systemctl enable --now named

sudo nmcli connection modify ens224 ipv4.dns 192.168.200.10
sudo nmcli connection up ens224

dig +short node1.lab.example.com @192.168.200.10   # forward test
dig +short -x 192.168.200.101 @192.168.200.10       # reverse test
```

> The single most common BIND mistake: editing a zone file but forgetting to **increment
> the serial**. `named` compares serials to decide whether to reload — same serial, no
> reload, and your change silently does nothing.

---

### 4. TFTP server

TFTP delivers the bootloader and the kernel/initrd. The `tftp-server` package ships a
systemd **socket** unit that starts the daemon on demand; you enable the socket, not a
service. Everything is served from `/var/lib/tftpboot`.

```bash
sudo mkdir -p /var/lib/tftpboot/pxelinux.cfg

# syslinux BIOS bootloader + menu modules
sudo cp /usr/share/syslinux/pxelinux.0   /var/lib/tftpboot/
sudo cp /usr/share/syslinux/menu.c32     /var/lib/tftpboot/
sudo cp /usr/share/syslinux/ldlinux.c32  /var/lib/tftpboot/
sudo cp /usr/share/syslinux/libutil.c32  /var/lib/tftpboot/

sudo systemctl enable --now tftp.socket
```

> If a `cp` fails, confirm the path with `rpm -ql syslinux | grep pxelinux.0` — the
> location can vary slightly by release.

---

### 5. Install tree + boot kernel/initrd (HTTP)

httpd serves two things: the full RHEL 9 package tree (so Anaconda can install packages)
and the Kickstart files. You also copy the boot kernel and initrd from the tree into the
TFTP root, because that's what PXELINUX loads.

```bash
sudo mkdir -p /var/www/html/rhel9
sudo mount -o loop /path/to/rhel-9-x86_64-dvd.iso /mnt
sudo cp -a /mnt/. /var/www/html/rhel9/

# kernel + initrd into the TFTP root
sudo cp /mnt/images/pxeboot/vmlinuz    /var/lib/tftpboot/
sudo cp /mnt/images/pxeboot/initrd.img /var/lib/tftpboot/
sudo umount /mnt

sudo restorecon -Rv /var/www/html          # fix SELinux labels so httpd can read it
sudo systemctl enable --now httpd
```

A valid tree has a `.treeinfo` file at its root and two variants, **BaseOS** and
**AppStream**, each with its own `repodata`. Verify:

```bash
curl -s http://192.168.200.10/rhel9/.treeinfo | grep -A3 variant
curl -s http://192.168.200.10/rhel9/BaseOS/repodata/    | head
curl -s http://192.168.200.10/rhel9/AppStream/repodata/ | head
```

> Both BaseOS **and** AppStream matter: the base environment groups pull packages from
> AppStream, so a Kickstart that only knows about BaseOS fails with "Error checking
> software selection". The Kickstart below declares both.

---

### 6. PXELINUX per-MAC menus

PXELINUX looks for a config file named after the booting client's MAC, prefixed with
`01-` (the ARP hardware type for Ethernet). This is how each target gets pointed at *its*
Kickstart. If no per-MAC file exists, it falls back to `default`.

Fallback `/var/lib/tftpboot/pxelinux.cfg/default` (boot local disk — protects already
installed machines from reinstalling):

```
default menu.c32
prompt 0
timeout 100
menu title RHEL 9 PXE Install

label local
  menu label Boot from local disk
  localboot 0
```

Per-node file, e.g. node1 → `/var/lib/tftpboot/pxelinux.cfg/01-00-50-56-00-02-01`:

```
default install
prompt 0
timeout 30
label install
  kernel vmlinuz
  append initrd=initrd.img inst.repo=http://192.168.200.10/rhel9 inst.ks=http://192.168.200.10/ks/ks-node1.cfg ip=dhcp
```

Duplicate for node2 / node3, changing the filename's MAC and the `ks-nodeN.cfg`.

---

### 7. Kickstart files

The Kickstart is the unattended answer file — it pre-decides mode, source, locale,
network, users, disks, packages, and post-install scripting. Store them under HTTP:

```bash
sudo mkdir -p /var/www/html/ks
```

`/var/www/html/ks/ks-node1.cfg`:

```
#version=RHEL9
# ============================================================================
# Every section is commented. Trim what you don't need.
# Validate before use:  ksvalidator ks-node1.cfg   (from the pykickstart package)
# ============================================================================

# ---------- Install mode ----------
text                        # text | graphical | cmdline

# ---------- Install source ----------
# Point url at the tree; declare BOTH repos explicitly so package resolution
# never fails on missing AppStream.
url --url="http://192.168.200.10/rhel9"
repo --name="BaseOS"    --baseurl="http://192.168.200.10/rhel9/BaseOS"
repo --name="AppStream" --baseurl="http://192.168.200.10/rhel9/AppStream"

# ---------- Localization ----------
lang en_US.UTF-8
keyboard --vckeymap=us --xlayouts='us'
timezone Africa/Cairo --utc

# ---------- Network ----------
# Pin to the target's known MAC so the static config lands on the RIGHT NIC
# (do NOT use --device=link if the VM has more than one adapter).
network --bootproto=static --device=00:50:56:00:02:01 --ip=192.168.200.101 --netmask=255.255.255.0 --gateway=192.168.200.10 --nameserver=192.168.200.10 --hostname=node1.lab.example.com --activate --onboot=yes
# DHCP alternative (single NIC): network --bootproto=dhcp --device=link --activate

# ---------- Security / identity ----------
# Generate a real hash with:  openssl passwd -6
rootpw --iscrypted REPLACE_WITH_HASH
# or lock root entirely:  rootpw --lock

# Admin user in wheel (sudo). Paste a real hash.
user --name=mazen --groups=wheel --gecos="Mazen Admin" --password=REPLACE_WITH_HASH --iscrypted

# Install the admin's SSH public key (paste your REAL key, not a placeholder).
sshkey --username=mazen "ssh-ed25519 REPLACE_WITH_YOUR_PUBLIC_KEY comment"

selinux --enforcing
firewall --enabled --service=ssh

# ---------- Disks & storage ----------
# IMPORTANT: match the real disk name. VMware NVMe controllers = nvme0n1 (not sda).
# Check from the installer shell (Ctrl+Alt+F2) with: lsblk
ignoredisk --only-use=nvme0n1
zerombr
clearpart --all --initlabel --drives=nvme0n1

# ---- Simple: let RHEL lay out LVM automatically (recommended for first run) ----
# autopart --type=lvm

# ---- Explicit manual layout (BIOS needs a biosboot partition; /boot stays plain) ----
part biosboot --fstype=biosboot --size=1    --ondisk=nvme0n1
part /boot    --fstype=xfs       --size=1024 --ondisk=nvme0n1
part pv.01    --size=1 --grow                --ondisk=nvme0n1
volgroup vg_system pv.01
logvol /     --vgname=vg_system --name=root --fstype=xfs  --size=8192
logvol /var  --vgname=vg_system --name=var  --fstype=xfs  --size=4096
logvol /app1 --vgname=vg_system --name=app1 --fstype=xfs  --size=2048
logvol swap  --vgname=vg_system --name=swap --fstype=swap --size=2048

bootloader --location=mbr --boot-drive=nvme0n1 --append="crashkernel=auto"

# ---------- Services & first boot ----------
services --enabled="sshd,chronyd,firewalld" --disabled="kdump"
firstboot --disable
reboot

# ============================================================================
%packages
@^minimal-environment
@core
openssh-server
chrony
vim-enhanced
bash-completion
tmux
%end

# ============================================================================
%pre --interpreter=/bin/bash --log=/tmp/ks-pre.log
echo "Starting install on $(date)"
lsblk -dno NAME,SIZE
%end

# ============================================================================
%post --interpreter=/bin/bash --log=/root/ks-post.log
# Passwordless sudo for wheel
echo '%wheel ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/wheel
chmod 0440 /etc/sudoers.d/wheel
echo "node1.lab.example.com - provisioned by PXE/Kickstart $(date)" > /etc/motd
%end

%post --nochroot --interpreter=/bin/bash
cp /tmp/ks-pre.log /mnt/sysroot/root/ks-pre.log 2>/dev/null || true
%end
```

Copy to `ks-node2.cfg` / `ks-node3.cfg`, changing the `--device` MAC, `--ip`, and
`--hostname`. **Always validate before serving:**

```bash
sudo dnf install -y pykickstart
ksvalidator /var/www/html/ks/ks-node1.cfg     # must print nothing = clean
```

---

### 8. Firewall

Open the four service ports. On an isolated lab network the default zone is fine.

```bash
sudo firewall-cmd --permanent --add-service=dhcp
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=tftp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

### 9. Start and verify

```bash
sudo systemctl enable --now dhcpd named tftp.socket httpd
systemctl --no-pager status dhcpd named tftp.socket httpd

# Watch the handoff live while a target boots:
sudo journalctl -u dhcpd -f                     # DHCPDISCOVER -> OFFER -> ACK
sudo journalctl -u named -f                     # DNS queries
sudo tail -f /var/log/messages | grep -i tftp   # TFTP file reads
sudo tail -f /var/log/httpd/access_log          # repo + ks fetches
```

---

## Booting a target

Confirm the target: **BIOS** firmware, NIC on **VMnet1**, **static MAC** set, **network
first** in boot order, and **≥2 GB RAM**. Power it on. Expected chain:

```
CLIENT MAC ADDR  →  DHCP lease from dhcpd  →  pxelinux.0 via TFTP
  →  per-MAC menu (01-<mac>)  →  vmlinuz + initrd.img via TFTP
  →  repo + Kickstart via HTTP  →  unattended install  →  reboot
```

After it reboots into the installed OS, from the control node:

```bash
ssh mazen@192.168.200.101 'hostname; id'
```

Because the Kickstart installed the control node's public key into `mazen`'s
`authorized_keys`, this logs in with **no password** and (thanks to the `%post` sudoers
rule) `mazen` has passwordless sudo. The nodes are now ready to be Ansible managed hosts.

---

## Adding more nodes (any number of servers)

Scaling to N servers is mechanical — three small additions per node:

1. **Reserve the MAC → IP in DHCP** (`/etc/dhcp/dhcpd.conf`):
   ```
   host node4 { hardware ethernet 00:50:56:00:02:04; fixed-address 192.168.200.104; option host-name "node4"; }
   ```
   then `sudo dhcpd -t && sudo systemctl restart dhcpd`.

2. **Add DNS records** (forward + reverse zones) and **bump both zone serials**, then
   `sudo systemctl reload named`.

3. **Create the per-MAC PXELINUX menu** and the **Kickstart**:
   ```
   /var/lib/tftpboot/pxelinux.cfg/01-00-50-56-00-02-04   ->  inst.ks=.../ks-node4.cfg
   /var/www/html/ks/ks-node4.cfg                          ->  --device MAC, --ip, --hostname
   ```

Give the target that MAC, set network-boot, power on. That's the "zero touch, any number
of servers" loop — a shell script or Ansible playbook can template these files from a
simple host list, which is the natural next step for this repo.

---

## Troubleshooting

Built from the failures actually hit while bringing this up — these are the ones you'll
most likely see.

| Symptom | Cause / fix |
|---------|-------------|
| Target gets no IP / no PXE | Hypervisor DHCP still enabled on VMnet1 (two DHCP servers fighting); `DHCPDARGS=ens224` not set; `dhcpd -t` not clean |
| IP but "no boot filename" | `next-server`/`filename` missing in dhcpd.conf, or `pxelinux.0` not in the TFTP root |
| TFTP timeouts | `tftp.socket` not enabled; firewall tftp closed; SELinux labels on `/var/lib/tftpboot` |
| PXELINUX loads but kernel/initrd fail | `vmlinuz`/`initrd.img` not copied into the TFTP root |
| **Install dies: "No space left on device", "Failed to find a root filesystem"** | **Target has too little RAM** — the ~750 MB `install.img` is staged in a RAM disk. Raise the VM to 2–4 GB. |
| **"Disk sda given in ignoredisk does not exist"** | **Wrong disk name.** VMware NVMe = `nvme0n1`. Check with `lsblk` on the installer shell (Ctrl+Alt+F2) and fix every disk reference in the Kickstart. |
| **"unrecognized arguments" / "Unknown command: --ip=..."** | **Broken line continuation / CRLF.** Flatten `network` and `user` to single lines; `sed -i 's/\r$//'`; re-run `ksvalidator`. |
| **"Error setting up software source" / "Error checking software selection"** | Install source can't resolve packages — declare **both** BaseOS and AppStream repos; verify `.treeinfo` and `repodata` are reachable. |
| SSH still prompts for a password despite a key | The `sshkey` line had a **placeholder** or wrong `--username`; the served Kickstart wasn't re-read (only applies on the *next* install). Fix now with `ssh-copy-id mazen@<ip>`. |
| Node has the **same IP on two NICs** | The VM has two adapters and `--device=link` grabbed the wrong one while DHCP filled the other. Give targets **one** NIC on VMnet1, and pin `network --device=<MAC>`. |
| Names don't resolve | Zone typo (`named-checkzone`), or the **serial wasn't bumped** after editing. |

**Host-side (VMware) note:** if the hypervisor host can't even reach the control node
(`Test-NetConnection` fails, adapter shows a `169.254.x.x` APIPA address), the host's
VMnet adapter lost its static address — VMware's NAT/DHCP services or the virtual
adapter need restarting (or a Virtual Network Editor *Restore Defaults*). Making guest
NICs **static** removes the dependency on that host-side DHCP entirely.


