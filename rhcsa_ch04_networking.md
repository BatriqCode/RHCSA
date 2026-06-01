# RHCSA RHEL 9 — COMPLETE EXAM PREPARATION COURSE
## Chapter 4: Network Management
### Senior Red Hat Linux Engineer · RHCSA/RHCE Instructor · 20+ Years Experience

---

> **Chapter 4 Priority:** Networking is tested on EVERY RHCSA exam. You will
> configure static IPs, hostnames, and DNS from the command line — no GUI.
> NetworkManager via nmcli is the RHEL 9 standard. Master nmcli cold.
> The exam gives you a specific IP, gateway, DNS, and hostname to set.
> If the network is wrong, remote storage and other tasks will also fail.

---

# ═══════════════════════════════════════════════════════════
# SECTION 1 — THEORY: FROM ZERO TO RHCSA
# ═══════════════════════════════════════════════════════════

## 1.1 Networking Fundamentals — The Concepts You Must Know

### IP Addressing Refresher

```
IPv4 Address:  192.168.1.100
               ─────────────
               Network  Host
               part     part
               (depends on subnet mask)

Subnet Mask (CIDR):  /24  =  255.255.255.0
                     /16  =  255.255.0.0
                     /8   =  255.0.0.0
                     /32  =  255.255.255.255 (single host)

CIDR Notation:  192.168.1.100/24
                Means: network is 192.168.1.0
                       hosts are 192.168.1.1 – 192.168.1.254
                       broadcast is 192.168.1.255

Gateway:  The router IP that forwards traffic outside your network
          Usually: 192.168.1.1  or  10.0.2.1

DNS:      Domain Name System — translates names to IPs
          8.8.8.8  = Google DNS
          1.1.1.1  = Cloudflare DNS
          192.168.1.1 = Local router (often does DNS too)
```

### Key Networking Concepts for RHCSA

| Concept | Definition | RHCSA Context |
|---|---|---|
| **IP Address** | Unique identifier for a network interface | Must set statically for exam tasks |
| **Subnet mask / CIDR** | Defines network vs host portion | `/24` most common on exam |
| **Default gateway** | Next-hop router for external traffic | Required for internet connectivity |
| **DNS server** | Resolves hostnames to IPs | Set for name resolution |
| **Hostname** | Human-readable name for the machine | Set with `hostnamectl` |
| **FQDN** | Fully Qualified Domain Name (host.domain.com) | `hostnamectl set-hostname` |
| **Network interface** | Physical or virtual NIC (ens3, eth0, etc.) | The device to configure |
| **Connection profile** | NetworkManager's config for an interface | Stored in `/etc/NetworkManager/system-connections/` |
| **MAC address** | Hardware address of NIC | Layer 2 identifier |
| **Loopback** | 127.0.0.1 — internal virtual interface | Always present, never modify |

---

## 1.2 The Linux Network Stack

```
╔══════════════════════════════════════════════════════════════════╗
║  RHEL 9 NETWORK STACK                                           ║
╠══════════════════════════════════════════════════════════════════╣
║  APPLICATION LAYER                                              ║
║  ping  ssh  curl  nfs  httpd  (use hostnames/IPs)              ║
╠══════════════════════════════════════════════════════════════════╣
║  NAME RESOLUTION                                                ║
║  /etc/hosts  →  checked first  (static hostname→IP mappings)   ║
║  /etc/resolv.conf  →  DNS server list (managed by NM)          ║
║  /etc/nsswitch.conf  →  resolution order (files dns myhostname) ║
╠══════════════════════════════════════════════════════════════════╣
║  NETWORK MANAGER (the boss)                                     ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  nmcli  nmtui  nm-connection-editor  (user interfaces)   │   ║
║  ├──────────────────────────────────────────────────────────┤   ║
║  │  NetworkManager daemon  (manages all connections)        │   ║
║  │  Connection Profiles:  /etc/NetworkManager/              │   ║
║  │  system-connections/*.nmconnection                       │   ║
║  └──────────────────────────────────────────────────────────┘   ║
╠══════════════════════════════════════════════════════════════════╣
║  KERNEL NETWORK LAYER                                           ║
║  Routing table  ARP cache  Netfilter/iptables  firewalld       ║
╠══════════════════════════════════════════════════════════════════╣
║  PHYSICAL / VIRTUAL LAYER                                       ║
║  ens3  ens33  eth0  ens160  enp3s0  (network interfaces)       ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 1.3 NetworkManager — The Central Authority

**NetworkManager (NM)** is the service that manages all network configuration
in RHEL 9. It replaced the older `network-scripts` and `/etc/sysconfig/network-scripts/`
approach. Everything goes through NM.

### Why NetworkManager?

- **Single point of control** — no more editing multiple config files
- **Profile-based** — connection configurations are stored as profiles
- **Event-driven** — reacts to hardware changes automatically
- **D-Bus integration** — `nmcli` and `nmtui` both talk to the NM daemon
- **Persistent** — profiles survive reboots without extra steps

### NM Components

```
┌─────────────────────────────────────────────────────────────────┐
│  NetworkManager Architecture                                    │
│                                                                 │
│  User tools:  nmcli  nmtui  nm-applet  cockpit                 │
│       ↕ D-Bus                                                   │
│  Daemon:   NetworkManager.service                               │
│       ↕                                                         │
│  Config:   /etc/NetworkManager/NetworkManager.conf              │
│            /etc/NetworkManager/system-connections/*.nmconnection│
│       ↕                                                         │
│  Runtime:  /run/NetworkManager/  (active state)                 │
│            /etc/resolv.conf     (DNS — NM writes this)          │
│       ↕                                                         │
│  Kernel:   ip commands, socket API                              │
└─────────────────────────────────────────────────────────────────┘
```

### Key Terminology: Device vs Connection

This is the **most common source of confusion** for RHCSA students:

```
DEVICE:     The physical/virtual network interface
            Examples: ens3, ens33, eth0, enp3s0
            Like: a physical port on your wall
            View with: nmcli device  or  ip link

CONNECTION: A configuration PROFILE applied to a device
            Examples: "Wired connection 1", "ens3-static"
            Like: the settings plugged into that port
            View with: nmcli connection  or  nmcli con

Relationship:
  One device can have MULTIPLE connection profiles
  But only ONE profile is ACTIVE at a time

Example:
  Device:  ens3
  Profile 1: "ens3-home"    (192.168.1.100/24, home gateway)
  Profile 2: "ens3-office"  (10.0.1.50/24, office gateway)
  Only one is active at any time
```

---

## 1.4 Network Interface Naming — Predictable Names

RHEL 9 uses **predictable interface names** based on hardware location.
Gone are the days of eth0, eth1 (usually). You'll see names like:

```
ens3     = Ethernet, Network card #3 (hotplug slot)
ens33    = Common in VMware VMs
enp3s0   = Ethernet, PCI bus 3, slot 0
eno1     = Ethernet, onboard NIC #1
eth0     = Old naming (still seen in some VMs)
lo       = Loopback (127.0.0.1 — always present)
virbr0   = Virtual bridge (KVM)
docker0  = Docker bridge
```

**Exam tip:** Always check interface names first with `ip link show` or `nmcli device`.
The exam might give you `ens33` or `eth0` — verify before configuring.

---

## 1.5 Connection Profile Storage

NetworkManager stores connection profiles here:

```
/etc/NetworkManager/system-connections/
├── ens3.nmconnection        (or Wired connection 1.nmconnection)
├── ens33.nmconnection
└── ...

Profile file format (INI-style):
[connection]
id=ens3-static
type=ethernet
interface-name=ens3

[ethernet]

[ipv4]
method=manual
addresses=192.168.1.100/24
gateway=192.168.1.1
dns=8.8.8.8;8.8.4.4;

[ipv6]
method=auto
```

> **Exam approach:** You almost never edit these files directly.
> Use `nmcli` to modify them — NM updates the files automatically.
> Understanding the file format helps when troubleshooting.

---

## 1.6 /etc/hosts — Static Name Resolution

`/etc/hosts` provides static name→IP mappings, checked **before** DNS.

```
# /etc/hosts format:
# IP_ADDRESS    FQDN              SHORT_NAME
127.0.0.1       localhost         localhost
::1             localhost         localhost
192.168.1.100   server1.lab.com   server1
192.168.1.200   client1.lab.com   client1

# Rules:
# - Each line: one IP, then one or more names
# - First name after IP is the official FQDN
# - Subsequent names are aliases
# - Comments start with #
```

---

## 1.7 /etc/resolv.conf — DNS Configuration

This file tells the system which DNS servers to query:

```
# /etc/resolv.conf — managed by NetworkManager in RHEL 9
nameserver 8.8.8.8          # Primary DNS
nameserver 8.8.4.4          # Secondary DNS
search lab.com example.com  # Domain search list
```

> **IMPORTANT:** In RHEL 9, NetworkManager **owns** `/etc/resolv.conf`.
> Manual edits are overwritten when NM restarts. Set DNS through `nmcli`.

---

## 1.8 Hostname in RHEL 9

RHEL 9 tracks three types of hostname:

```
STATIC hostname:    Stored in /etc/hostname
                    Persists across reboots
                    Set with: hostnamectl set-hostname NAME

TRANSIENT hostname: Kernel's current hostname (from NM or DHCP)
                    Temporary, lost on reboot

PRETTY hostname:    Free-form display name (optional)
                    e.g., "John's Development Server"

# View all three:
hostnamectl

# Sample output:
#  Static hostname: server1.lab.com
#  Transient hostname: (none if same as static)
#  Pretty hostname: Lab Server 1
#        Icon name: computer-vm
#          Chassis: vm 🖥
#       Machine ID: abc123...
#          Boot ID: def456...
#   Virtualization: kvm
# Operating System: Red Hat Enterprise Linux 9.3
#      CPE OS Name: cpe:/o:redhat:enterprise_linux:9
#           Kernel: Linux 5.14.0-362.8.1.el9_3.x86_64
#     Architecture: x86-64
```

---

## 1.9 DHCP vs Static IP — Exam Decision

```
DHCP (dynamic):
  method=auto
  IP assigned automatically by a DHCP server
  Changes between reboots (unreliable for servers)
  nmcli default: method=auto

STATIC (manual):
  method=manual
  You specify IP, prefix, gateway, DNS
  Survives reboots unchanged
  REQUIRED for exam tasks (servers always get static IPs)
```

---

## 1.10 Exam Traps and Common Networking Mistakes

1. **Wrong interface name** — always `ip link show` first; never assume ens33
2. **Forgetting `nmcli con up`** after changes — profile saved but not activated
3. **Setting IP without gateway** — host isolated from network
4. **Setting IP without DNS** — names don't resolve (breaks NFS, etc.)
5. **Editing `/etc/resolv.conf` directly** — NM overwrites it
6. **Forgetting `autoconnect yes`** — interface won't come up at boot
7. **Typo in IP address** — `192.168.1.10/24` vs `192.168.1.100/24`
8. **Using `ifconfig`** — deprecated in RHEL 9; use `ip addr`
9. **Wrong CIDR prefix** — `/24` vs `/32` means very different things
10. **Not verifying with `ping`** — always test after configuring
11. **Forgetting to check firewall** — firewalld may block connections
12. **Setting hostname with `hostname` command** — not persistent; use `hostnamectl`

---

# ═══════════════════════════════════════════════════════════
# SECTION 2 — COMMANDS REFERENCE (EXAM-LEVEL DETAIL)
# ═══════════════════════════════════════════════════════════

## 2.1 nmcli — The Primary Network Tool for RHCSA

`nmcli` is the command-line interface to NetworkManager. This is the
**primary tool** you will use on the RHCSA exam.

```
Syntax:  nmcli [OPTIONS] OBJECT ACTION [ARGS]

Objects:
  general (g)      NetworkManager general status
  connection (con,c) Connection profiles
  device (dev,d)   Network interfaces/devices
  radio            Wi-Fi/WWAN radio control
  monitor          Monitor NM events
  agent            Secret agent
  help             Help
```

---

## 2.2 nmcli general — System Status

```bash
# Overall NM status
nmcli general status
nmcli general                   # Alias

# Show hostname
nmcli general hostname

# Set hostname via NM
nmcli general hostname server1.lab.com

# Show logging level
nmcli general logging

# Sample output:
# STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
# connected  full          enabled  enabled  enabled  enabled
```

---

## 2.3 nmcli device — Interface Management

```bash
# List all network devices
nmcli device
nmcli dev
nmcli dev status                # Same as above (with STATUS column)

# Sample output:
# DEVICE  TYPE      STATE      CONNECTION
# ens3    ethernet  connected  Wired connection 1
# lo      loopback  unmanaged  --

# Show details of a specific device
nmcli device show ens3
nmcli dev show ens3

# Key fields in device show:
# GENERAL.DEVICE:    ens3
# GENERAL.TYPE:      ethernet
# GENERAL.STATE:     100 (connected)
# GENERAL.CONNECTION: Wired connection 1
# IP4.ADDRESS[1]:    192.168.1.100/24
# IP4.GATEWAY:       192.168.1.1
# IP4.DNS[1]:        8.8.8.8

# Connect/disconnect a device
nmcli device connect ens3       # Bring up
nmcli device disconnect ens3    # Bring down

# Show device details for all interfaces
nmcli dev show                  # All interfaces, full details
```

---

## 2.4 nmcli connection — Profile Management

This is the heart of network configuration:

```bash
# List all connection profiles
nmcli connection
nmcli con
nmcli con show                  # Same

# Sample output:
# NAME                UUID                 TYPE      DEVICE
# Wired connection 1  abc123-...           ethernet  ens3
# ens33               def456-...           ethernet  --

# Show details of a specific connection
nmcli con show "Wired connection 1"
nmcli con show ens3

# Activate a connection (bring it up)
nmcli con up "Wired connection 1"
nmcli con up ens3-static

# Deactivate a connection
nmcli con down "Wired connection 1"

# Delete a connection profile
nmcli con delete "Wired connection 1"
nmcli con delete UUID

# Reload all connection files from disk
nmcli con reload

# Modify an existing connection
nmcli con mod "connection name" SETTING.PROPERTY VALUE

# Apply modifications without full reconnect
nmcli con up "connection name"
```

---

## 2.5 nmcli con add — Create Connection Profiles

### Create a Static IP Connection (Most Common Exam Task)

```bash
# Full syntax for static Ethernet connection:
nmcli con add \
  type ethernet \
  con-name "ens3-static" \
  ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.dns-search "lab.com" \
  connection.autoconnect yes

# Explanation of each option:
# type ethernet          → connection type
# con-name "ens3-static" → profile name (your choice)
# ifname ens3            → bind to interface ens3
# ipv4.method manual     → static (not DHCP)
# ipv4.addresses         → IP and CIDR prefix
# ipv4.gateway           → default gateway
# ipv4.dns               → DNS servers (space or comma separated)
# ipv4.dns-search        → domain search list
# connection.autoconnect yes → bring up at boot
```

### Create a DHCP Connection

```bash
nmcli con add \
  type ethernet \
  con-name "ens3-dhcp" \
  ifname ens3 \
  ipv4.method auto \
  connection.autoconnect yes
```

### One-liner Versions (Exam Speed)

```bash
# Static IP — compact one-liner:
nmcli con add type ethernet con-name ens3 ifname ens3 ipv4.method manual ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8

# Even shorter — relies on defaults:
nmcli con add type eth con-name myconn ifname ens3 ip4 192.168.1.100/24 gw4 192.168.1.1
```

---

## 2.6 nmcli con mod — Modify Existing Connections

```bash
# General syntax:
nmcli con mod "CONNECTION_NAME" PROPERTY VALUE

# === IP Address ===
nmcli con mod ens3 ipv4.addresses 10.0.1.50/24
nmcli con mod ens3 ipv4.addresses "10.0.1.50/24 10.0.1.51/24"  # Multiple IPs

# === Gateway ===
nmcli con mod ens3 ipv4.gateway 10.0.1.1

# === DNS (REPLACE all DNS servers) ===
nmcli con mod ens3 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con mod ens3 ipv4.dns "8.8.8.8,8.8.4.4"   # Comma also works

# === DNS (ADD a DNS server) ===
nmcli con mod ens3 +ipv4.dns 1.1.1.1             # + prefix = ADD

# === DNS (REMOVE a DNS server) ===
nmcli con mod ens3 -ipv4.dns 8.8.4.4             # - prefix = REMOVE

# === Method (static vs DHCP) ===
nmcli con mod ens3 ipv4.method manual
nmcli con mod ens3 ipv4.method auto

# === Autoconnect ===
nmcli con mod ens3 connection.autoconnect yes
nmcli con mod ens3 connection.autoconnect no

# === DNS Search Domain ===
nmcli con mod ens3 ipv4.dns-search "lab.com example.com"

# === Change connection name ===
nmcli con mod ens3 connection.id "ens3-production"

# === AFTER ANY MODIFICATION — must reapply! ===
nmcli con up ens3                # Reconnect to apply changes
# OR
nmcli con down ens3 && nmcli con up ens3
```

---

## 2.7 nmcli Property Reference — Most Used on RHCSA

```bash
# Connection properties:
connection.id              → Profile name (what you call it)
connection.type            → ethernet, wifi, bridge, bond, etc.
connection.interface-name  → Device name (ens3, eth0, etc.)
connection.autoconnect     → yes/no — connect at boot

# IPv4 properties:
ipv4.method                → manual (static), auto (DHCP), disabled
ipv4.addresses             → "IP/prefix" or "IP/prefix gateway" (old way)
ipv4.gateway               → default route
ipv4.dns                   → "IP1 IP2" or "IP1,IP2"
ipv4.dns-search            → "domain1.com domain2.com"
ipv4.ignore-auto-dns       → yes/no — ignore DHCP-provided DNS
ipv4.never-default         → yes/no — don't use this as default route

# IPv6 properties:
ipv6.method                → auto, manual, ignore, disabled
ipv6.addresses             → IPv6 address/prefix
ipv6.gateway               → IPv6 gateway
ipv6.dns                   → IPv6 DNS

# Ethernet properties:
ethernet.mac-address       → Set MAC override
ethernet.mtu               → MTU size (default 1500)
```

---

## 2.8 nmcli Shorthand — Time-Saving Aliases

On the exam, every second counts. Use these abbreviations:

```bash
# Full → Short
nmcli connection show   →   nmcli con show   →   nmcli c s
nmcli device show       →   nmcli dev show   →   nmcli d s
nmcli connection up     →   nmcli con up     →   nmcli c up
nmcli connection mod    →   nmcli con mod    →   nmcli c mod
nmcli connection add    →   nmcli con add    →   nmcli c add
nmcli connection delete →   nmcli con del    →   nmcli c del
nmcli general status    →   nmcli g          
```

---

## 2.9 nmtui — Text UI for NetworkManager

`nmtui` provides a menu-driven interface — useful if you forget nmcli syntax.

```bash
# Launch nmtui
nmtui

# Opens a TUI (Text User Interface):
# ┌──────────────────────────────────────┐
# │  NetworkManager TUI                  │
# │                                      │
# │  ► Edit a connection                 │
# │    Activate a connection             │
# │    Set system hostname               │
# │                                      │
# │    Quit                              │
# └──────────────────────────────────────┘
#
# Navigate: Arrow keys, Tab, Space, Enter
# "Edit a connection" → select NIC → configure static IP
# "Activate a connection" → enable/disable connections

# Direct launch to specific function:
nmtui-edit                  # Go straight to edit a connection
nmtui-connect               # Go straight to activate/deactivate
nmtui-hostname              # Go straight to set hostname
```

> **EXAM STRATEGY:** `nmtui` is your backup if you blank on `nmcli` syntax.
> It's slower but reliable. Know both — use nmcli first for speed.

---

## 2.10 hostnamectl — Hostname Management

```bash
# View current hostname
hostnamectl
hostnamectl status          # Same

# Set static hostname (PERSISTENT — survives reboot)
hostnamectl set-hostname server1.lab.com
hostnamectl set-hostname server1          # Short name also valid

# Set pretty hostname (display name, optional)
hostnamectl set-hostname "Lab Server 1" --pretty

# Set transient hostname (temporary, lost on reboot)
hostnamectl set-hostname server1 --transient

# Verify
hostnamectl
cat /etc/hostname            # Should show new name
hostname                     # Current active hostname

# The exam task is always: set hostname to X — use:
hostnamectl set-hostname X
```

---

## 2.11 ip — The Modern Interface Tool

The `ip` command is used for **verification** — not configuration on RHCSA
(NM handles configuration). Know it for status checking and troubleshooting.

```bash
# === ip addr — IP addresses ===
ip addr                      # All interfaces with IPs
ip addr show                 # Same
ip addr show ens3            # Specific interface
ip a                         # Shorthand for ip addr
ip a s ens3                  # Shorthand for ip addr show ens3

# === ip link — Layer 2 (interfaces, no IPs) ===
ip link show                 # All interfaces, L2 info
ip link show ens3            # Specific interface
ip l                         # Shorthand

# Bring interface up/down:
ip link set ens3 up
ip link set ens3 down

# === ip route — Routing table ===
ip route                     # Show routing table
ip route show                # Same
ip r                         # Shorthand
ip route show default        # Show default gateway only

# Add/delete routes (temporary — for troubleshooting only):
ip route add 192.168.2.0/24 via 192.168.1.1
ip route del 192.168.2.0/24

# === ip neigh — ARP/neighbor table ===
ip neigh                     # ARP cache
ip neigh show                # Same

# Sample output of ip addr:
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
#     inet 127.0.0.1/8 scope host lo
# 2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     link/ether 52:54:00:xx:xx:xx brd ff:ff:ff:ff:ff:ff
#     inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic ens3
```

---

## 2.12 ss — Socket Statistics (Replaces netstat)

```bash
# Syntax
ss [OPTIONS]

# Common options:
-t          TCP sockets
-u          UDP sockets
-l          Listening sockets only
-a          All sockets (listening + established)
-n          Numeric (don't resolve names)
-p          Show process using socket
-e          Extended info
-s          Summary statistics

# Most used combinations:
ss -tlnp                     # TCP listening ports with process names
ss -ulnp                     # UDP listening ports
ss -tlnp | grep :22          # Is SSH listening?
ss -tlnp | grep :80          # Is HTTP listening?
ss -an | grep ESTABLISHED    # Active connections
ss -s                        # Summary of all sockets

# Sample output:
# State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# LISTEN  0       128     0.0.0.0:22         0.0.0.0:*          sshd
# LISTEN  0       128     [::]:22            [::]:*             sshd
```

---

## 2.13 ping — Connectivity Testing

```bash
# Syntax
ping [OPTIONS] DESTINATION

# Options:
-c N        Send N packets then stop
-i N        Interval between packets (seconds)
-W N        Timeout for each reply (seconds)
-s SIZE     Packet size in bytes
-4          Force IPv4
-6          Force IPv6

# Exam usage:
ping -c 4 192.168.1.1               # Test gateway
ping -c 4 8.8.8.8                   # Test internet
ping -c 4 server1.lab.com           # Test DNS resolution
ping -c 2 localhost                 # Test loopback (should always work)

# Quick one-packet test:
ping -c 1 192.168.1.1 && echo "REACHABLE" || echo "UNREACHABLE"
```

---

## 2.14 traceroute / tracepath — Path Tracing

```bash
# traceroute (may need: dnf install traceroute)
traceroute 8.8.8.8

# tracepath (built-in, no install needed)
tracepath 8.8.8.8
tracepath -4 8.8.8.8               # IPv4 only
```

---

## 2.15 nslookup / dig / host — DNS Testing

```bash
# nslookup (simple DNS query)
nslookup google.com                 # Forward lookup
nslookup 8.8.8.8                    # Reverse lookup

# dig (detailed DNS query — install with: dnf install bind-utils)
dig google.com                      # Forward lookup
dig @8.8.8.8 google.com            # Query specific DNS server
dig -x 8.8.8.8                     # Reverse lookup
dig +short google.com               # Just the IP

# host (simple)
host google.com
host 8.8.8.8
```

---

## 2.16 NetworkManager Service Management

```bash
# Check NM status
systemctl status NetworkManager

# Start/stop/restart NM
systemctl start NetworkManager
systemctl stop NetworkManager
systemctl restart NetworkManager

# Enable at boot
systemctl enable NetworkManager

# Reload NM config (softer than restart)
systemctl reload NetworkManager

# OR using nmcli:
nmcli general reload            # Reload config without restart

# Check if interface has connectivity
nmcli networking connectivity
# Options: full, limited, portal, none, unknown
```

---

## 2.17 Complete Static IP Configuration — The Full Workflow

This is the most common exam task. Memorize this entire workflow:

```bash
# THE COMPLETE STATIC IP WORKFLOW
# Assume: interface=ens3, IP=192.168.1.100/24, GW=192.168.1.1, DNS=8.8.8.8

# STEP 1: Identify the interface
ip link show                        # OR
nmcli device status

# STEP 2: Check current configuration
nmcli con show                      # See existing profiles
nmcli device show ens3              # Current IP/GW/DNS

# STEP 3A: Modify existing profile (if profile already exists)
CON=$(nmcli -t -f NAME,DEVICE con show --active | grep ens3 | cut -d: -f1)
nmcli con mod "$CON" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes

# STEP 3B: Create new profile (if starting fresh)
nmcli con add type ethernet con-name ens3-static ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes

# STEP 4: Activate the connection
nmcli con up ens3-static            # OR: nmcli con up "$CON"

# STEP 5: Verify everything
ip addr show ens3                   # IP assigned?
ip route show default               # Gateway set?
cat /etc/resolv.conf                # DNS set?
ping -c 2 192.168.1.1               # Gateway reachable?
ping -c 2 8.8.8.8                   # Internet reachable?
ping -c 2 google.com                # DNS working?

# STEP 6: Set hostname (usually part of same task)
hostnamectl set-hostname server1.lab.com
hostnamectl                         # Verify
```


# ═══════════════════════════════════════════════════════════
# SECTION 3 — PRACTICAL LABS
# ═══════════════════════════════════════════════════════════

---

## LAB 1 (Beginner) — Explore Your Network Configuration

### Objective
Understand the current network state without making any changes.

### Tasks

**Task 1.1 — See all interfaces and their state**
```bash
ip link show
ip addr show
nmcli device status
nmcli dev
```

**Task 1.2 — Identify your primary interface name**
```bash
# Find the interface that has an IP (not lo):
ip addr | grep -E "^[0-9]+:|inet " | grep -v "127.0.0.1"
# OR
nmcli device status | grep -v loopback
# Note the name (ens3, ens33, eth0, enp3s0, etc.)
```

**Task 1.3 — View full connection profile details**
```bash
nmcli con show                          # List profiles
nmcli con show "Wired connection 1"     # Adjust name to match your system
nmcli device show ens3                  # Replace ens3 with your interface
```

**Task 1.4 — Check routing**
```bash
ip route show
ip route show default                   # Just the default gateway
```

**Task 1.5 — Check DNS configuration**
```bash
cat /etc/resolv.conf
nmcli con show ens3 | grep DNS          # DNS in connection profile
```

**Task 1.6 — Check hostname**
```bash
hostname
hostnamectl
cat /etc/hostname
```

**Task 1.7 — Test connectivity**
```bash
ping -c 2 localhost
ping -c 2 $(ip route show default | awk '{print $3}')   # Ping gateway
ping -c 2 8.8.8.8
ping -c 2 google.com
```

**Task 1.8 — Check listening services**
```bash
ss -tlnp
ss -tlnp | grep :22   # SSH should be listening
```

### Expected Results
You can identify all interface names, IPs, gateways, DNS servers, and hostname.

---

## LAB 2 (Beginner) — Set Hostname

### Objective
Change the system hostname persistently using `hostnamectl`.

### Tasks

**Task 2.1 — Record current hostname**
```bash
hostnamectl
hostname
cat /etc/hostname
```

**Task 2.2 — Set new hostname**
```bash
hostnamectl set-hostname rhcsa-server.lab.com
```

**Task 2.3 — Verify the change**
```bash
hostnamectl
cat /etc/hostname              # Should show rhcsa-server.lab.com
hostname                       # Shows current active hostname
```

**Task 2.4 — Note: shell prompt may still show old name**
```bash
# The prompt updates when you open a NEW shell session
bash                           # New shell — shows new hostname in prompt
# OR
exec bash                      # Replace current shell
```

### Verification
```bash
hostnamectl | grep "Static hostname"   # Must show rhcsa-server.lab.com
cat /etc/hostname                      # Must show rhcsa-server.lab.com
```

---

## LAB 3 (Intermediate) — Configure Static IP with nmcli

### Objective
Configure a static IP address on your primary interface.

### Scenario
Configure interface `ens3` with:
- IP: `192.168.1.100/24`
- Gateway: `192.168.1.1`
- DNS: `8.8.8.8` and `8.8.4.4`
- Profile name: `ens3-static`

### Pre-Lab: Identify your interface
```bash
# Get your actual interface name:
IFACE=$(ip route show default | awk '{print $5}')
echo "Primary interface: $IFACE"
# Use this name in place of ens3 below
```

### Tasks

**Task 3.1 — Check existing profiles**
```bash
nmcli con show
# Note: an existing profile for your interface may already exist
```

**Task 3.2 — Option A: Create a new static profile**
```bash
nmcli con add type ethernet \
  con-name ens3-static \
  ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes
```

**Task 3.3 — Option B: Modify existing profile**
```bash
# If a profile already exists for your interface:
nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes
```

**Task 3.4 — Activate the profile**
```bash
nmcli con up ens3-static          # If you used Option A
# OR
nmcli con up "Wired connection 1" # If you used Option B
```

**Task 3.5 — Verify the configuration**
```bash
ip addr show ens3                 # Check IP address
ip route show default             # Check gateway
cat /etc/resolv.conf              # Check DNS
nmcli device show ens3            # All settings together
```

**Task 3.6 — Test connectivity**
```bash
ping -c 2 192.168.1.1             # Gateway
ping -c 2 8.8.8.8                 # Internet
ping -c 2 google.com              # DNS resolution
```

### Expected /etc/resolv.conf After Configuration
```
# Generated by NetworkManager
nameserver 8.8.8.8
nameserver 8.8.4.4
```

### Verification
```bash
ip addr show ens3 | grep "192.168.1.100"
ip route show default | grep "192.168.1.1"
cat /etc/resolv.conf | grep "8.8.8.8"
ping -c 1 8.8.8.8 && echo "PASS" || echo "FAIL"
```

---

## LAB 4 (Intermediate) — Modify an Existing Connection Profile

### Objective
Change individual settings on an existing connection without recreating it.

### Tasks

**Task 4.1 — Change DNS servers**
```bash
# Replace all DNS with new servers:
nmcli con mod ens3-static ipv4.dns "1.1.1.1 1.0.0.1"
nmcli con up ens3-static
cat /etc/resolv.conf                    # Verify
```

**Task 4.2 — Add a second IP address**
```bash
# Add (not replace) a second IP:
nmcli con mod ens3-static +ipv4.addresses 192.168.1.101/24
nmcli con up ens3-static
ip addr show ens3                       # Verify both IPs
```

**Task 4.3 — Remove the second IP**
```bash
nmcli con mod ens3-static -ipv4.addresses 192.168.1.101/24
nmcli con up ens3-static
ip addr show ens3                       # Only .100 remains
```

**Task 4.4 — Add a DNS search domain**
```bash
nmcli con mod ens3-static ipv4.dns-search "lab.com"
nmcli con up ens3-static
cat /etc/resolv.conf                    # Shows "search lab.com"
```

**Task 4.5 — Disable autoconnect**
```bash
nmcli con mod ens3-static connection.autoconnect no
nmcli con show ens3-static | grep autoconnect
# Re-enable:
nmcli con mod ens3-static connection.autoconnect yes
```

---

## LAB 5 (Intermediate) — nmtui Static IP Configuration

### Objective
Configure a static IP using the text UI (backup method for exam).

### Tasks

**Task 5.1 — Launch nmtui**
```bash
nmtui
```

**Task 5.2 — Navigate to Edit a Connection**
```
Select: ► Edit a connection
Select your interface (e.g., ens3)
Press: Edit...
```

**Task 5.3 — Configure static IP in the TUI**
```
IPv4 CONFIGURATION: <Manual>       ← Change from <Automatic> to <Manual>
Addresses:   192.168.1.100/24      ← Add
Gateway:     192.168.1.1           ← Add
DNS servers: 8.8.8.8               ← Add
             8.8.4.4               ← Add (press Enter to add another)
[✓] Automatically connect          ← Check this box
[✓] Available to all users         ← Check this box

Select: <OK>
```

**Task 5.4 — Activate the connection**
```
Back to main menu:
Select: Activate a connection
Select your connection
Press: Enter (to activate)
```

**Task 5.5 — Set hostname via nmtui**
```
Back to main menu:
Select: Set system hostname
Enter: rhcsa-server.lab.com
Select: <OK>
```

**Task 5.6 — Verify**
```bash
ip addr show ens3
hostnamectl
```

---

## LAB 6 (RHCSA Exam Level) — Complete Network Setup in Under 5 Minutes

### Scenario
The exam gives you:
- Interface: `ens3` (or check with `ip link show`)
- Static IP: `172.16.10.50/16`
- Gateway: `172.16.0.1`
- DNS: `172.16.0.1`
- Hostname: `node1.rhcsa.lab`

### Timed Solution

```bash
# STEP 1: Identify interface (5 seconds)
ip link show | grep -v "lo:" | grep "^[0-9]" | head -3

# STEP 2: Find existing profile name (5 seconds)
nmcli con show

# STEP 3: Configure (30 seconds)
# Option A — if profile exists, modify it:
PROFILE=$(nmcli -t -f NAME con show | head -1)
nmcli con mod "$PROFILE" \
  ipv4.method manual \
  ipv4.addresses 172.16.10.50/16 \
  ipv4.gateway 172.16.0.1 \
  ipv4.dns 172.16.0.1 \
  connection.autoconnect yes

# Option B — create new profile:
nmcli con add type ethernet con-name node1-static ifname ens3 \
  ipv4.method manual ipv4.addresses 172.16.10.50/16 \
  ipv4.gateway 172.16.0.1 ipv4.dns 172.16.0.1

# STEP 4: Activate (3 seconds)
nmcli con up "$PROFILE"  # or: nmcli con up node1-static

# STEP 5: Set hostname (5 seconds)
hostnamectl set-hostname node1.rhcsa.lab

# STEP 6: Verify (15 seconds)
ip addr show ens3 | grep 172.16.10.50
ip route show default | grep 172.16.0.1
cat /etc/resolv.conf
hostnamectl | grep Static
ping -c 1 172.16.0.1
```

---

## LAB 7 (Troubleshooting) — Network Diagnosis Workflow

### Scenario
A server has no network connectivity. Diagnose step by step.

### Systematic Diagnosis Procedure

```bash
# LAYER 1: Is the interface UP?
ip link show ens3
# Look for: UP or DOWN in the flags
# Fix if DOWN:
nmcli con up ens3

# LAYER 2: Does it have an IP?
ip addr show ens3
# If no IP: check if NM assigned one
nmcli device show ens3 | grep IP4

# LAYER 3: Is the connection profile active?
nmcli con show --active
nmcli device status

# LAYER 4: Can we reach the gateway?
ip route show default                           # What is gateway?
ping -c 2 $(ip route show default | awk '{print $3}')

# LAYER 5: Can we reach external IPs?
ping -c 2 8.8.8.8                              # Bypass DNS

# LAYER 6: Is DNS working?
ping -c 2 google.com                           # Uses DNS
cat /etc/resolv.conf                           # DNS configured?
dig google.com @8.8.8.8                        # Query specific DNS

# LAYER 7: Check NM service
systemctl status NetworkManager
journalctl -u NetworkManager -n 20

# LAYER 8: Check firewall (not blocking local traffic?)
systemctl status firewalld
firewall-cmd --list-all

# Common fixes:
nmcli con up ens3                              # Activate connection
nmcli con mod ens3 ipv4.gateway 192.168.1.1   # Fix wrong gateway
nmcli con mod ens3 ipv4.dns "8.8.8.8"         # Fix missing DNS
systemctl restart NetworkManager               # Last resort
```


# ═══════════════════════════════════════════════════════════
# SECTION 4 — 20 RHCSA EXAM SCENARIOS
# ═══════════════════════════════════════════════════════════

---

### SCENARIO 1
**Task:** Configure the network interface `ens3` with IP `192.168.1.100/24`,
gateway `192.168.1.1`, and DNS `8.8.8.8`. Make it persistent.

**Thinking:** modify or create profile → set method manual → set address/gw/dns → activate

**Full Solution:**
```bash
nmcli con add type ethernet con-name ens3-static ifname ens3 \
  ipv4.method manual ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8 \
  connection.autoconnect yes
nmcli con up ens3-static
```

**Verification:**
```bash
ip addr show ens3 | grep 192.168.1.100
ip route | grep default | grep 192.168.1.1
cat /etc/resolv.conf | grep 8.8.8.8
```

---

### SCENARIO 2
**Task:** Set the system hostname to `server1.example.com` permanently.

**Full Solution:**
```bash
hostnamectl set-hostname server1.example.com
```

**Verification:**
```bash
hostnamectl | grep "Static hostname"
cat /etc/hostname
```

---

### SCENARIO 3
**Task:** An existing connection `"Wired connection 1"` is using DHCP.
Convert it to a static IP `10.0.0.50/8` with gateway `10.0.0.1` and
DNS `10.0.0.1`. Keep using the same profile name.

**Full Solution:**
```bash
nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.50/8 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns 10.0.0.1 \
  connection.autoconnect yes
nmcli con up "Wired connection 1"
```

**Verification:**
```bash
nmcli device show ens3 | grep -E "IP4.ADDRESS|IP4.GATEWAY|IP4.DNS"
ip route show default
```

---

### SCENARIO 4
**Task:** Add a secondary DNS server `8.8.4.4` to an existing connection
without changing the primary DNS.

**Full Solution:**
```bash
nmcli con mod ens3-static +ipv4.dns 8.8.4.4     # + adds, doesn't replace
nmcli con up ens3-static
cat /etc/resolv.conf    # Should now show both 8.8.8.8 and 8.8.4.4
```

---

### SCENARIO 5
**Task:** Add a static entry to `/etc/hosts` so that `node2.lab.com`
resolves to `192.168.1.200`. Verify name resolution.

**Full Solution:**
```bash
echo "192.168.1.200  node2.lab.com  node2" >> /etc/hosts
cat /etc/hosts | grep node2
# Test:
ping -c 1 node2.lab.com
ping -c 1 node2
```

---

### SCENARIO 6
**Task:** List all active network connections and identify which interface
is carrying the default route.

**Full Solution:**
```bash
nmcli con show --active
ip route show default
# Output shows: default via 192.168.1.1 dev ens3 proto ...
#                                           ↑ this is your interface
```

---

### SCENARIO 7
**Task:** Configure a second interface `ens4` with IP `10.0.0.100/24`.
No gateway or DNS needed (internal network only).

**Full Solution:**
```bash
nmcli con add type ethernet con-name ens4-internal ifname ens4 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.100/24 \
  ipv4.never-default yes \
  connection.autoconnect yes
nmcli con up ens4-internal
ip addr show ens4
```

---

### SCENARIO 8
**Task:** Delete an unwanted connection profile named `"Old Connection"`.

**Full Solution:**
```bash
nmcli con show                          # Confirm it exists
nmcli con delete "Old Connection"
nmcli con show                          # Confirm it's gone
```

---

### SCENARIO 9
**Task:** Verify that the system can resolve the hostname `server2.lab.com`
and reach it. Diagnose any failure.

**Full Solution:**
```bash
# Step 1: Name resolution
ping -c 1 server2.lab.com 2>&1
nslookup server2.lab.com

# Step 2: If DNS fails, check resolv.conf:
cat /etc/resolv.conf

# Step 3: If not in DNS, add to /etc/hosts:
echo "192.168.1.200 server2.lab.com server2" >> /etc/hosts
ping -c 1 server2.lab.com   # Should work now
```

---

### SCENARIO 10
**Task:** Check which ports are currently listening on TCP. Save the output
to `/root/listening_ports.txt`.

**Full Solution:**
```bash
ss -tlnp > /root/listening_ports.txt
cat /root/listening_ports.txt
```

---

### SCENARIO 11
**Task:** Configure the interface to use DNS search domain `corp.example.com`
so that `ping webserver` resolves as `ping webserver.corp.example.com`.

**Full Solution:**
```bash
nmcli con mod ens3-static ipv4.dns-search "corp.example.com"
nmcli con up ens3-static
cat /etc/resolv.conf          # Shows "search corp.example.com"
# Test: ping webserver (tries webserver.corp.example.com automatically)
```

---

### SCENARIO 12
**Task:** Show the current NetworkManager daemon status and confirm it is
enabled for automatic startup.

**Full Solution:**
```bash
systemctl status NetworkManager
systemctl is-enabled NetworkManager     # Should output: enabled
systemctl is-active NetworkManager      # Should output: active
nmcli general status                    # NM connectivity info
```

---

### SCENARIO 13
**Task:** Temporarily bring down interface `ens3`, then bring it back up.

**Full Solution:**
```bash
# Bring down:
nmcli con down ens3-static
ip addr show ens3               # IP should be gone

# Bring back up:
nmcli con up ens3-static
ip addr show ens3               # IP should be back

# Alternative using device:
nmcli device disconnect ens3
nmcli device connect ens3
```

---

### SCENARIO 14
**Task:** The system currently has `ens3` configured with DHCP. You must
change it to static `192.168.2.50/24` with gateway `192.168.2.1` and
two DNS servers: `192.168.2.1` and `8.8.8.8`.

**Full Solution:**
```bash
# Find current DHCP profile name:
PROFILE=$(nmcli -t -f NAME,DEVICE con show | grep ens3 | cut -d: -f1)
echo "Profile: $PROFILE"

nmcli con mod "$PROFILE" \
  ipv4.method manual \
  ipv4.addresses 192.168.2.50/24 \
  ipv4.gateway 192.168.2.1 \
  ipv4.dns "192.168.2.1 8.8.8.8"

nmcli con up "$PROFILE"

# Verify:
ip addr show ens3
ip route show default
cat /etc/resolv.conf
```

---

### SCENARIO 15
**Task:** Show all connection profile details for interface `ens3` including
IP, gateway, DNS, and autoconnect setting.

**Full Solution:**
```bash
# Option 1: NM device view
nmcli device show ens3

# Option 2: Connection profile view
nmcli con show ens3-static | grep -E "ipv4\.|connection\.auto"

# Option 3: ip command (runtime state)
ip addr show ens3
ip route show
```

---

### SCENARIO 16
**Task:** Configure the hostname to `workstation.home.lab` and add an
entry to `/etc/hosts` mapping `127.0.0.1` to that hostname.

**Full Solution:**
```bash
hostnamectl set-hostname workstation.home.lab

# Add to /etc/hosts:
echo "127.0.0.1  workstation.home.lab  workstation" >> /etc/hosts

# Verify:
hostnamectl
cat /etc/hostname
grep workstation /etc/hosts
ping -c 1 workstation.home.lab
```

---

### SCENARIO 17
**Task:** The network connection is failing. Diagnose using only read-only
commands and identify the problem.

**Troubleshooting Script:**
```bash
#!/bin/bash
echo "=== Network Diagnostic Report ==="

echo -e "\n--- 1. Interface Status ---"
ip link show | grep -E "^[0-9]+:"

echo -e "\n--- 2. IP Addresses ---"
ip addr show | grep inet

echo -e "\n--- 3. Routing Table ---"
ip route show

echo -e "\n--- 4. Active NM Connections ---"
nmcli con show --active

echo -e "\n--- 5. DNS Config ---"
cat /etc/resolv.conf

echo -e "\n--- 6. Hostname ---"
hostnamectl | grep hostname

echo -e "\n--- 7. Ping Tests ---"
echo -n "Loopback: "; ping -c1 -W1 127.0.0.1 &>/dev/null && echo "OK" || echo "FAIL"
GW=$(ip route show default | awk '{print $3}' | head -1)
echo -n "Gateway ($GW): "; ping -c1 -W2 $GW &>/dev/null && echo "OK" || echo "FAIL"
echo -n "Internet (8.8.8.8): "; ping -c1 -W3 8.8.8.8 &>/dev/null && echo "OK" || echo "FAIL"
echo -n "DNS (google.com): "; ping -c1 -W3 google.com &>/dev/null && echo "OK" || echo "FAIL"

echo -e "\n--- 8. NM Service Status ---"
systemctl is-active NetworkManager
```

---

### SCENARIO 18
**Task:** Create a connection profile that uses DHCP for IPv4 but disables
IPv6 entirely.

**Full Solution:**
```bash
nmcli con add type ethernet con-name ens3-dhcp-noipv6 ifname ens3 \
  ipv4.method auto \
  ipv6.method disabled \
  connection.autoconnect yes
nmcli con up ens3-dhcp-noipv6
ip addr show ens3                   # Should show only IPv4 address
```

---

### SCENARIO 19
**Task:** Configure a static IP using `nmtui` (describe the steps).

**Answer:**
```bash
nmtui

# Step 1: Select "Edit a connection"
# Step 2: Select your interface (ens3)
# Step 3: Press <Edit...>
# Step 4: Change IPv4 CONFIGURATION from <Automatic> to <Manual>
# Step 5: In Addresses, add: 192.168.1.100/24
# Step 6: In Gateway, enter: 192.168.1.1
# Step 7: In DNS servers, add: 8.8.8.8  (press Enter for next)
# Step 8: Check [✓] Automatically connect
# Step 9: Press <OK> then <Back>
# Step 10: Select "Activate a connection"
# Step 11: Select your connection → Enter to activate
# Step 12: Press <Back> then <Quit>

# Verify:
ip addr show ens3
```

---

### SCENARIO 20
**Task:** Show all nmcli connection property names relevant to IPv4
(useful when you forget the exact property name during the exam).

**Full Solution:**
```bash
# Show all properties of a connection:
nmcli con show ens3-static | grep ipv4

# Key IPv4 properties you'll see:
# ipv4.method:          manual
# ipv4.dns:             8.8.8.8,8.8.4.4
# ipv4.dns-search:      lab.com
# ipv4.addresses:       192.168.1.100/24
# ipv4.gateway:         192.168.1.1
# ipv4.routes:          (empty usually)
# ipv4.never-default:   no
# ipv4.ignore-auto-dns: no

# If you forget a property name:
nmcli con show ens3-static | grep -i dns
nmcli con show ens3-static | grep -i gateway
nmcli con show ens3-static | grep -i address
```


# ═══════════════════════════════════════════════════════════
# SECTION 5 — TROUBLESHOOTING GUIDE
# ═══════════════════════════════════════════════════════════

---

## FAILURE 1: Interface Has No IP After nmcli con up

**Symptoms:** Command succeeded but `ip addr show` shows no IP.

**Diagnosis:**
```bash
nmcli con show --active                 # Is the connection active?
nmcli device status                     # What state is the device in?
journalctl -u NetworkManager -n 30      # NM log messages
ip link show ens3                       # Is the interface UP?
```

**Common Causes & Fixes:**
```bash
# Cause A: Interface is DOWN (cable issue in VMs or wrong interface name)
ip link set ens3 up
nmcli device connect ens3

# Cause B: Wrong interface name in profile
nmcli con show ens3-static | grep "interface-name"
# Fix: delete and recreate with correct ifname
nmcli con delete ens3-static
nmcli con add type ethernet con-name ens3-static ifname CORRECT_NAME ...

# Cause C: Conflicting profile with same interface
nmcli con show                          # More than one profile for ens3?
nmcli con down "Wired connection 1"     # Deactivate the other one
nmcli con up ens3-static

# Cause D: NetworkManager not running
systemctl start NetworkManager
nmcli con up ens3-static
```

---

## FAILURE 2: No Default Route (Cannot Reach Outside Network)

**Symptoms:** Can ping local IPs but not gateway or internet.

**Diagnosis:**
```bash
ip route show                           # Is default route present?
ip route show default                   # Shows: nothing OR wrong gateway
```

**Fix:**
```bash
# Fix gateway in profile:
nmcli con mod ens3-static ipv4.gateway 192.168.1.1
nmcli con up ens3-static

# Verify:
ip route show default
ping -c 2 192.168.1.1

# Temporary fix (not persistent):
ip route add default via 192.168.1.1 dev ens3
# (Make permanent via nmcli)
```

---

## FAILURE 3: DNS Not Working (Can Ping IPs but Not Names)

**Symptoms:** `ping 8.8.8.8` works, `ping google.com` fails.

**Diagnosis:**
```bash
cat /etc/resolv.conf                    # DNS servers listed?
dig google.com @8.8.8.8                # Direct query works?
nslookup google.com                    # Uses /etc/resolv.conf
```

**Fix:**
```bash
# Fix DNS in profile (NM will update /etc/resolv.conf):
nmcli con mod ens3-static ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con up ens3-static
cat /etc/resolv.conf                    # Should now have nameserver entries

# If /etc/resolv.conf is empty despite NM setting DNS:
# Check if it's a symlink issue:
ls -la /etc/resolv.conf
# Should be: /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
# If broken:
systemctl restart NetworkManager
systemctl restart systemd-resolved      # If using systemd-resolved
```

---

## FAILURE 4: Changes Not Persisting After Reboot

**Symptoms:** IP is correct now but wrong after reboot.

**Cause:** Profile has `connection.autoconnect no` or the active profile
is not the one you edited.

**Diagnosis:**
```bash
nmcli con show ens3-static | grep autoconnect
nmcli con show --active                 # Which profile is active?
```

**Fix:**
```bash
nmcli con mod ens3-static connection.autoconnect yes
# Verify:
nmcli con show ens3-static | grep autoconnect
```

---

## FAILURE 5: Hostname Not Changing / Reverting

**Symptoms:** `hostnamectl set-hostname` ran but hostname still shows old name.

**Diagnosis:**
```bash
cat /etc/hostname                       # Was file written?
hostnamectl                             # Static vs transient mismatch?
systemctl status NetworkManager         # NM overriding hostname from DHCP?
```

**Fix:**
```bash
# Fix A: NM getting hostname from DHCP — disable DHCP hostname:
nmcli con mod "Wired connection 1" ipv4.dhcp-hostname ""
nmcli con mod "Wired connection 1" ipv4.dhcp-send-hostname false

# Fix B: Manually write /etc/hostname:
echo "server1.lab.com" > /etc/hostname
hostnamectl                             # Verify

# Fix C: Restart after setting:
hostnamectl set-hostname server1.lab.com
systemctl restart systemd-hostnamed
```

---

## FAILURE 6: Two Connection Profiles Fight Over Same Interface

**Symptoms:** Interface keeps going up and down, or wrong IP appears.

**Diagnosis:**
```bash
nmcli con show | grep ens3             # Multiple profiles for ens3?
nmcli con show --active                # Which one is active?
```

**Fix:**
```bash
# Option A: Delete the old unwanted profile:
nmcli con delete "Wired connection 1"  # Delete old DHCP profile
nmcli con up ens3-static               # Use only the static one

# Option B: Disable autoconnect on the unwanted profile:
nmcli con mod "Wired connection 1" connection.autoconnect no
nmcli con up ens3-static
```

---

## FAILURE 7: "Error: Connection activation failed" When Running nmcli con up

**Symptoms:** nmcli con up returns an error.

**Diagnosis:**
```bash
journalctl -u NetworkManager -n 50     # Detailed error
nmcli con show ens3-static             # Profile syntax correct?
ip link show ens3                      # Interface exists?
```

**Common Causes:**
```bash
# Cause A: Interface name mismatch
nmcli con show ens3-static | grep interface-name
# Fix: edit ifname:
nmcli con mod ens3-static connection.interface-name CORRECT_NAME

# Cause B: IP already in use (ARP conflict)
arping -I ens3 192.168.1.100           # Check for conflict
# Fix: use a different IP

# Cause C: Invalid IP/prefix format
# Wrong: ipv4.addresses 192.168.1.100   (missing /prefix)
# Right: ipv4.addresses 192.168.1.100/24
nmcli con mod ens3-static ipv4.addresses 192.168.1.100/24
```

---

## FAILURE 8: /etc/resolv.conf Keeps Getting Overwritten

**Symptoms:** You edit `/etc/resolv.conf` manually but it changes back.

**Explanation:** This is correct behavior. NetworkManager owns `/etc/resolv.conf`.
Manual edits ARE overwritten. Always use `nmcli` to set DNS.

**Fix:**
```bash
# Set DNS through NM (permanent):
nmcli con mod ens3-static ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con up ens3-static
cat /etc/resolv.conf                   # NM wrote your DNS servers

# If you MUST have manual resolv.conf (bad practice):
# Make it immutable (breaks NM — avoid on exam):
chattr +i /etc/resolv.conf             # NOT recommended
```

---

## FAILURE 9: ping: Name or service not known

**Symptoms:** `ping hostname` fails with name resolution error.

**Diagnosis steps:**
```bash
ping -c1 8.8.8.8                       # Internet reachable by IP?
cat /etc/resolv.conf                   # DNS configured?
nslookup hostname 8.8.8.8             # Can reach DNS server?
cat /etc/hosts | grep hostname         # In /etc/hosts?
cat /etc/nsswitch.conf | grep hosts    # Resolution order?
```

**Fix:**
```bash
# If /etc/resolv.conf empty:
nmcli con mod ens3-static ipv4.dns "8.8.8.8"
nmcli con up ens3-static

# If hostname is private/local (not in public DNS):
echo "192.168.1.50  privatehostname.lab.com" >> /etc/hosts
```

---

## FAILURE 10: Interface Is UP but nmcli Shows "disconnected"

**Symptoms:** `ip link show ens3` shows UP, but `nmcli dev` shows disconnected.

**Cause:** NM isn't managing the interface (unmanaged state) or the profile was deleted.

**Fix:**
```bash
nmcli device status
# If "unmanaged":
nmcli device set ens3 managed yes

# If "disconnected" (managed but no profile):
# Create a new profile:
nmcli con add type ethernet con-name ens3-new ifname ens3 \
  ipv4.method manual ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8
nmcli con up ens3-new
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 6 — MEMORY AIDS
# ═══════════════════════════════════════════════════════════

## The Static IP Configuration Recipe (Memorize This)

```bash
nmcli con add type ethernet \
  con-name NAME \      ← what YOU call this profile
  ifname DEVICE \      ← ens3, eth0, ens33 etc
  ipv4.method manual \ ← static (not DHCP)
  ipv4.addresses IP/PREFIX \   ← e.g. 192.168.1.100/24
  ipv4.gateway GW \    ← e.g. 192.168.1.1
  ipv4.dns "DNS1 DNS2" ← e.g. "8.8.8.8 8.8.4.4"
  connection.autoconnect yes   ← survive reboots!

nmcli con up NAME
```

**Memory hook: "NAME Device Manual Address Gateway DNS Auto"**

---

## nmcli con mod — The + and - Prefix Rules

```
NO PREFIX    → REPLACE the value entirely
+ PREFIX     → ADD to existing value
- PREFIX     → REMOVE from existing value

Examples:
  nmcli con mod ens3 ipv4.dns "8.8.8.8"        # REPLACE → only 8.8.8.8
  nmcli con mod ens3 +ipv4.dns "8.8.4.4"       # ADD → now has both
  nmcli con mod ens3 -ipv4.dns "8.8.8.8"       # REMOVE 8.8.8.8
```

---

## Device vs Connection — Quick Recall

```
DEVICE  = Physical interface  (ens3, eth0)     → nmcli dev
          Like: the ethernet port on your server

CONNECTION = Configuration profile            → nmcli con
          Like: the settings file for that port

Commands on DEVICEs:    nmcli device connect/disconnect/status/show
Commands on CONNECTIONs: nmcli con up/down/add/mod/del/show
```

---

## /etc/resolv.conf — Who Owns What

```
OWNER: NetworkManager (in RHEL 9)
SET via: nmcli con mod PROFILE ipv4.dns "IP1 IP2"
VERIFY: cat /etc/resolv.conf
NEVER: manually edit (NM overwrites it)

Format:
nameserver 8.8.8.8
nameserver 8.8.4.4
search lab.com
```

---

## Hostname Cheat Sheet

```
View:    hostnamectl
Set:     hostnamectl set-hostname NAME.DOMAIN
File:    /etc/hostname
Temp:    hostname NAME   (NOT persistent — don't use!)
```

---

## Network Troubleshooting OSI Ladder

```
Problem Area    Test Command                  Fix
──────────────────────────────────────────────────────────────────
Layer 1/2  →  ip link show ens3            → nmcli dev connect ens3
  (interface)   (check: UP flag)

Layer 3    →  ip addr show ens3            → nmcli con mod ... ipv4.addresses
  (IP)          (check: inet line)

Layer 3    →  ip route show default        → nmcli con mod ... ipv4.gateway
  (gateway)     (check: via X.X.X.X)

Layer 7    →  ping 8.8.8.8                → Check gateway, routing
  (IP reach)    (bypass DNS)

Layer 7    →  ping google.com             → nmcli con mod ... ipv4.dns
  (DNS)         (uses /etc/resolv.conf)
```

---

## Top 15 Networking Commands for RHCSA (Priority Order)

```
Priority  Command                                    Purpose
 1        ip link show / ip addr show               See interfaces and IPs
 2        nmcli device status                        See NM device state
 3        nmcli con show                             List profiles
 4        nmcli con add type ethernet ...            Create static profile
 5        nmcli con mod PROFILE ipv4.xxx VALUE       Modify profile
 6        nmcli con up PROFILE                       Activate profile
 7        hostnamectl set-hostname NAME              Set hostname
 8        ip route show default                      Check gateway
 9        cat /etc/resolv.conf                       Check DNS
10        ping -c 2 IP_OR_NAME                       Test connectivity
11        nmcli device show IFACE                    Full device details
12        nmcli con show PROFILE | grep ipv4         Check profile settings
13        ss -tlnp                                   Check listening ports
14        cat /etc/hosts                             Check static names
15        journalctl -u NetworkManager -n 20        Debug NM errors
```

---

## Quick nmcli Syntax Reference Card

```
╔══════════════════════════════════════════════════════════════════════╗
║  NMCLI QUICK REFERENCE                                               ║
╠══════════════════════════════════════════════════════════════════════╣
║  STATUS                                                              ║
║    nmcli dev status              List all devices                    ║
║    nmcli con show                List all profiles                   ║
║    nmcli con show --active       Active profiles only                ║
║    nmcli device show ens3        Full interface details              ║
║    nmcli general                 Overall NM status                  ║
╠══════════════════════════════════════════════════════════════════════╣
║  CREATE STATIC PROFILE                                               ║
║    nmcli con add type ethernet con-name NAME ifname DEV \           ║
║      ipv4.method manual ipv4.addresses IP/PREFIX \                  ║
║      ipv4.gateway GW ipv4.dns "DNS1 DNS2"                           ║
╠══════════════════════════════════════════════════════════════════════╣
║  MODIFY PROFILE                                                      ║
║    nmcli con mod NAME ipv4.addresses IP/PREFIX    (replace)         ║
║    nmcli con mod NAME +ipv4.dns IP               (add)             ║
║    nmcli con mod NAME -ipv4.dns IP               (remove)          ║
║    nmcli con mod NAME ipv4.gateway GW             (change GW)      ║
║    nmcli con mod NAME ipv4.dns "IP1 IP2"         (replace DNS)     ║
║    nmcli con mod NAME ipv4.method auto            (switch to DHCP) ║
║    nmcli con mod NAME connection.autoconnect yes  (auto at boot)   ║
╠══════════════════════════════════════════════════════════════════════╣
║  ACTIVATE / DEACTIVATE                                               ║
║    nmcli con up NAME             Activate connection                 ║
║    nmcli con down NAME           Deactivate connection               ║
║    nmcli device connect ens3     Connect device                     ║
║    nmcli device disconnect ens3  Disconnect device                  ║
╠══════════════════════════════════════════════════════════════════════╣
║  DELETE                                                              ║
║    nmcli con delete NAME         Delete profile permanently         ║
╠══════════════════════════════════════════════════════════════════════╣
║  HOSTNAME                                                            ║
║    hostnamectl set-hostname NAME.DOMAIN                              ║
║    hostnamectl status                                                ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 7 — COMMAND COMPARISON TABLES
# ═══════════════════════════════════════════════════════════

## nmcli vs nmtui vs ip vs ifconfig

| Feature | nmcli | nmtui | ip | ifconfig |
|---|---|---|---|---|
| Interface | CLI | Text menu | CLI | CLI |
| Persistent config | **YES** | **YES** | NO (temporary) | NO |
| Script-friendly | **YES** | No | Yes | Yes |
| Exam primary tool | **YES** | Backup | Verification | DEPRECATED |
| Manages NM profiles | YES | YES | No | No |
| Read-only info | YES | Yes | YES | Yes |
| Available in RHEL 9 | YES | YES | YES | Optional install |
| RHCSA importance | **HIGHEST** | HIGH | HIGH | LOW |

---

## Network Information Commands

| Command | What It Shows | Best For |
|---|---|---|
| `ip addr show` | IP addresses on all interfaces | Quick IP check |
| `ip link show` | Interface state (UP/DOWN, MAC) | Layer 2 check |
| `ip route show` | Routing table | Gateway check |
| `nmcli device show` | Full NM view of interface | Complete details |
| `nmcli con show` | All NM profiles | Profile inventory |
| `ss -tlnp` | TCP listening ports + process | Service port check |
| `cat /etc/resolv.conf` | DNS servers | DNS config check |
| `hostnamectl` | All hostname info | Hostname check |
| `ping -c N HOST` | Connectivity test | End-to-end verify |
| `nslookup HOST` | DNS lookup | Name resolution test |

---

## Static vs DHCP — nmcli Settings Comparison

| Setting | Static (manual) | DHCP (auto) |
|---|---|---|
| `ipv4.method` | `manual` | `auto` |
| `ipv4.addresses` | `192.168.1.100/24` | (empty — DHCP assigns) |
| `ipv4.gateway` | `192.168.1.1` | (empty — DHCP assigns) |
| `ipv4.dns` | `8.8.8.8 8.8.4.4` | (empty — DHCP assigns) |
| Exam use | Required for server config | Default out of box |

---

## hostname Command Comparison

| Command | Persistence | Use |
|---|---|---|
| `hostname` | Shows current | Read-only display |
| `hostname newname` | **NOT persistent** | Temporary — avoid |
| `hostnamectl set-hostname` | **Persistent** | **Use this always** |
| `nmcli general hostname` | Persistent | Alternative |
| `echo name > /etc/hostname` | Persistent | Manual — acceptable |

---

# ═══════════════════════════════════════════════════════════
# SECTION 8 — HANDS-ON EXAM ENVIRONMENT
# ═══════════════════════════════════════════════════════════

## VM Network Setup for RHCSA Practice

```
Recommended VM Network Configuration:
┌─────────────────────────────────────────────────────────────┐
│  Adapter 1 (NAT):                                           │
│    Purpose: Internet access (dnf, updates)                  │
│    DHCP: Yes (auto-configured)                              │
│    Interface: ens3 or eth0 (first adapter)                  │
│                                                             │
│  Adapter 2 (Host-Only: 192.168.56.0/24):                   │
│    Purpose: Lab network (NFS, SSH practice)                 │
│    DHCP: No — configure static for practice                 │
│    Interface: ens4 or eth1 (second adapter)                 │
│    IP: 192.168.56.101/24                                    │
│    Gateway: 192.168.56.1 (host-only virtual router)         │
└─────────────────────────────────────────────────────────────┘

Server VM:   192.168.56.101   hostname: server1.lab.com
Client VM:   192.168.56.102   hostname: client1.lab.com
```

## Network Lab Setup Script

```bash
#!/bin/bash
# network_lab_setup.sh — Run on each VM as root

STATIC_IP="${1:-192.168.56.101}"
HOSTNAME_SET="${2:-server1.lab.com}"
IFACE="${3:-ens4}"

echo "Configuring: IP=$STATIC_IP, Hostname=$HOSTNAME_SET, Interface=$IFACE"

# Set hostname
hostnamectl set-hostname "$HOSTNAME_SET"

# Configure static IP on the lab interface
# First check if profile exists:
PROFILE=$(nmcli -t -f NAME,DEVICE con show | grep "$IFACE" | cut -d: -f1)

if [ -z "$PROFILE" ]; then
    nmcli con add type ethernet con-name "${IFACE}-lab" ifname "$IFACE" \
        ipv4.method manual \
        ipv4.addresses "${STATIC_IP}/24" \
        ipv4.gateway "192.168.56.1" \
        ipv4.dns "192.168.56.1 8.8.8.8" \
        connection.autoconnect yes
    nmcli con up "${IFACE}-lab"
else
    nmcli con mod "$PROFILE" \
        ipv4.method manual \
        ipv4.addresses "${STATIC_IP}/24" \
        ipv4.gateway "192.168.56.1" \
        ipv4.dns "192.168.56.1 8.8.8.8" \
        connection.autoconnect yes
    nmcli con up "$PROFILE"
fi

# Add /etc/hosts entries for lab
cat >> /etc/hosts << 'HOSTS'
192.168.56.101  server1.lab.com  server1
192.168.56.102  client1.lab.com  client1
HOSTS

echo "Setup complete."
ip addr show "$IFACE"
hostnamectl | grep "Static"
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 9 — EXAM CHECKLIST
# ═══════════════════════════════════════════════════════════

## Chapter 4: Network Management — Pre-Exam Checklist

### Conceptual Knowledge ✓
- [ ] Explain the difference between a NM Device and a Connection profile
- [ ] State what `ipv4.method manual` means vs `auto`
- [ ] Explain why `hostname NAME` command is not persistent
- [ ] State what file NM writes DNS servers to (/etc/resolv.conf)
- [ ] Explain the + and - prefix in `nmcli con mod`
- [ ] State the three types of hostname (static, transient, pretty)
- [ ] Explain the resolution order: /etc/hosts → DNS → myhostname
- [ ] Name the config file directory for NM profiles
- [ ] State what `connection.autoconnect yes` does

### Practical Skills ✓
- [ ] Find interface name: `ip link show` or `nmcli dev status`
- [ ] Create static profile: full `nmcli con add type ethernet ...` command
- [ ] Modify existing profile: `nmcli con mod PROFILE ipv4.xxx VALUE`
- [ ] Add DNS without replacing: `nmcli con mod PROFILE +ipv4.dns IP`
- [ ] Activate connection: `nmcli con up PROFILE`
- [ ] Set hostname: `hostnamectl set-hostname NAME.DOMAIN`
- [ ] Verify IP: `ip addr show IFACE`
- [ ] Verify gateway: `ip route show default`
- [ ] Verify DNS: `cat /etc/resolv.conf`
- [ ] Test connectivity: `ping -c 2 IP` and `ping -c 2 HOSTNAME`
- [ ] Use nmtui as backup method for static IP config
- [ ] Add /etc/hosts entry for static name resolution
- [ ] Check listening ports: `ss -tlnp`
- [ ] View NM profile in full: `nmcli con show PROFILE`
- [ ] Delete unwanted profile: `nmcli con delete NAME`
- [ ] Troubleshoot no gateway: `ip route show default` then fix with nmcli
- [ ] Troubleshoot DNS failure: `cat /etc/resolv.conf` then fix with nmcli
- [ ] Troubleshoot profile not activating at boot: check `connection.autoconnect`

---

# ═══════════════════════════════════════════════════════════
# SECTION 10 — CHAPTER 4 MASTER CHALLENGE (45 minutes)
# ═══════════════════════════════════════════════════════════

## Master Challenge: Full Network Configuration

**Time limit:** 45 minutes
**Environment:** RHEL 9 VM with two interfaces: `ens3` (NAT/DHCP) and `ens4` (lab network)

---

### Exam Statement

You are configuring a RHEL 9 server for a development environment.
Complete all network configuration tasks:

**Task 1:** Set the system hostname to `devserver.corp.example.com`.

**Task 2:** Configure interface `ens4` with the following static settings
(persistent across reboots):
- IP Address: `192.168.100.10/24`
- Default Gateway: `192.168.100.1`
- Primary DNS: `192.168.100.1`
- Secondary DNS: `8.8.8.8`
- DNS search domain: `corp.example.com`
- Connection profile name: `lab-static`

**Task 3:** Add static `/etc/hosts` entries:
- `192.168.100.1` → `gateway.corp.example.com` and `gateway`
- `192.168.100.20` → `appserver.corp.example.com` and `appserver`

**Task 4:** Verify that the name `appserver` resolves to `192.168.100.20`
using the `/etc/hosts` file.

**Task 5:** Ensure NetworkManager is enabled and starts automatically at boot.

**Task 6:** Create a diagnostic report file at `/root/network_report.txt`
containing the output of: IP addresses, routing table, DNS configuration,
hostname, and listening TCP ports.

---

### Full Solution

```bash
#!/bin/bash
# RHCSA Chapter 4 Network Management Master Challenge

set -e
echo "=== RHCSA Chapter 4: Network Management Master Challenge ==="
echo ""

# === IDENTIFY INTERFACE ===
echo "--- Identifying network interfaces ---"
ip link show
IFACE="ens4"    # Adjust to match your second interface
echo "Using interface: $IFACE"
echo ""

# === TASK 1: Set Hostname ===
echo "--- Task 1: Setting hostname ---"
hostnamectl set-hostname devserver.corp.example.com
hostnamectl | grep "Static"
echo "Task 1: DONE"
echo ""

# === TASK 2: Static IP on ens4 ===
echo "--- Task 2: Configuring static IP on $IFACE ---"

# Remove any existing profile for this interface first:
OLD=$(nmcli -t -f NAME,DEVICE con show | grep ":${IFACE}$" | cut -d: -f1)
[ -n "$OLD" ] && nmcli con delete "$OLD" 2>/dev/null && echo "Removed old profile: $OLD"

# Create new static profile:
nmcli con add \
  type ethernet \
  con-name lab-static \
  ifname "$IFACE" \
  ipv4.method manual \
  ipv4.addresses 192.168.100.10/24 \
  ipv4.gateway 192.168.100.1 \
  ipv4.dns "192.168.100.1 8.8.8.8" \
  ipv4.dns-search "corp.example.com" \
  connection.autoconnect yes

nmcli con up lab-static

# Verify:
ip addr show "$IFACE" | grep 192.168.100.10 && echo "IP set: OK" || echo "IP set: FAILED"
ip route show default | grep 192.168.100.1 && echo "Gateway: OK" || echo "Gateway: check needed"
grep "192.168.100" /etc/resolv.conf && echo "DNS: OK" || echo "DNS: check needed"

echo "Task 2: DONE"
echo ""

# === TASK 3: /etc/hosts entries ===
echo "--- Task 3: Adding /etc/hosts entries ---"

# Back up first:
cp /etc/hosts /etc/hosts.backup.ch4

# Remove any existing entries for these hosts (idempotent):
sed -i '/gateway.corp.example.com\|appserver.corp.example.com/d' /etc/hosts

# Add new entries:
cat >> /etc/hosts << 'HOSTS'
192.168.100.1   gateway.corp.example.com  gateway
192.168.100.20  appserver.corp.example.com  appserver
HOSTS

grep -E "gateway|appserver" /etc/hosts
echo "Task 3: DONE"
echo ""

# === TASK 4: Verify name resolution ===
echo "--- Task 4: Verifying name resolution ---"
ping -c 1 -W 2 appserver 2>/dev/null && echo "appserver resolves: OK" \
    || echo "appserver resolves via /etc/hosts (ping may fail if host unreachable)"
getent hosts appserver           # Should show 192.168.100.20
nslookup appserver 2>/dev/null | grep Address || echo "Check /etc/hosts manually"
echo "Task 4: DONE"
echo ""

# === TASK 5: Ensure NetworkManager is enabled ===
echo "--- Task 5: NetworkManager boot enable ---"
systemctl enable NetworkManager
systemctl is-enabled NetworkManager && echo "NM enabled: OK"
systemctl is-active NetworkManager && echo "NM active: OK"
echo "Task 5: DONE"
echo ""

# === TASK 6: Diagnostic Report ===
echo "--- Task 6: Creating diagnostic report ---"
cat > /root/network_report.txt << 'REPORT_START'
=== RHCSA Network Diagnostic Report ===
Generated: REPORT_START
echo "Generated: $(date)" >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== IP ADDRESSES ===" >> /root/network_report.txt
ip addr show >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== ROUTING TABLE ===" >> /root/network_report.txt
ip route show >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== DNS CONFIGURATION ===" >> /root/network_report.txt
cat /etc/resolv.conf >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== HOSTNAME ===" >> /root/network_report.txt
hostnamectl >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== LISTENING TCP PORTS ===" >> /root/network_report.txt
ss -tlnp >> /root/network_report.txt

echo "" >> /root/network_report.txt
echo "=== /etc/hosts ===" >> /root/network_report.txt
cat /etc/hosts >> /root/network_report.txt

ls -lh /root/network_report.txt && echo "Task 6: DONE"
echo ""

# === FINAL VERIFICATION ===
echo "═══════════════════════════════════════"
echo "      FINAL VERIFICATION REPORT"
echo "═══════════════════════════════════════"

echo ""
echo "--- Hostname ---"
hostnamectl | grep "Static hostname"
cat /etc/hostname

echo ""
echo "--- Interface $IFACE ---"
ip addr show "$IFACE" | grep "inet "

echo ""
echo "--- Routing ---"
ip route show default

echo ""
echo "--- DNS (resolv.conf) ---"
cat /etc/resolv.conf

echo ""
echo "--- DNS search domain ---"
nmcli con show lab-static | grep dns-search

echo ""
echo "--- /etc/hosts ---"
grep -E "gateway|appserver" /etc/hosts

echo ""
echo "--- Name resolution test ---"
getent hosts gateway
getent hosts appserver

echo ""
echo "--- NM service ---"
systemctl is-active NetworkManager
systemctl is-enabled NetworkManager

echo ""
echo "--- Connection profile lab-static ---"
nmcli con show lab-static | grep -E "ipv4\.|connection\.auto"

echo ""
echo "--- Report file ---"
ls -lh /root/network_report.txt

echo ""
echo "=== CHALLENGE COMPLETE ==="
```

---

### Grader Verification Commands

```bash
# All must succeed:
hostnamectl | grep "devserver.corp.example.com"
cat /etc/hostname | grep "devserver.corp.example.com"
nmcli con show lab-static | grep "192.168.100.10/24"
nmcli con show lab-static | grep "192.168.100.1" | grep gateway
ip addr show ens4 | grep "192.168.100.10/24"
ip route show default | grep "192.168.100.1"
cat /etc/resolv.conf | grep "192.168.100.1"
cat /etc/resolv.conf | grep "8.8.8.8"
cat /etc/resolv.conf | grep "corp.example.com"
nmcli con show lab-static | grep "autoconnect.*yes"
grep "192.168.100.1.*gateway" /etc/hosts
grep "192.168.100.20.*appserver" /etc/hosts
getent hosts appserver | grep "192.168.100.20"
systemctl is-enabled NetworkManager | grep enabled
ls -la /root/network_report.txt
grep "IP ADDRESSES" /root/network_report.txt
```

---

### Common Mistakes in This Challenge

1. **Wrong interface name** — using `ens3` instead of `ens4` (always check first!)
2. **Missing DNS search domain** — forgetting `ipv4.dns-search "corp.example.com"`
3. **Not running `nmcli con up`** after creating the profile
4. **Duplicate /etc/hosts entries** — add idempotently with `sed -i` to remove old ones first
5. **Setting hostname with `hostname` command** — not persistent, fails after reboot check
6. **Profile not set to autoconnect** — missing `connection.autoconnect yes`
7. **Wrong prefix length** — `/24` not `/32` or `/8`
8. **Not verifying with `ip addr`** after nmcli — never assume it worked
9. **Forgetting secondary DNS** — task specified both `192.168.100.1` and `8.8.8.8`
10. **Wrong report file location** — must be `/root/network_report.txt` not `/tmp/`

---

# ═══════════════════════════════════════════════════════════
# SELF-ASSESSMENT QUIZ — CHAPTER 4
# ═══════════════════════════════════════════════════════════

**Q1:** What is the difference between a NetworkManager Device and a Connection?

**Q2:** What command creates a static IP profile on interface `ens3` with IP `10.0.0.5/24`,
gateway `10.0.0.1`, and DNS `10.0.0.1`?

**Q3:** After running `nmcli con mod ens3-static ipv4.dns "1.1.1.1"`, what happens to
the previously configured DNS server `8.8.8.8`?

**Q4:** How do you ADD a DNS server without removing the existing ones?

**Q5:** A static IP was configured but after reboot the IP is gone. What is the most
likely cause and fix?

**Q6:** What file does NetworkManager write DNS server information to?
Can you safely edit this file manually?

**Q7:** What is the correct command to permanently change the hostname to `node1.rhcsa.lab`?

**Q8:** What does `connection.autoconnect yes` do in a NetworkManager profile?

**Q9:** You can `ping 8.8.8.8` but cannot `ping google.com`. What is the problem
and how do you fix it using nmcli?

**Q10:** What `nmcli` command shows all properties of the connection profile named `ens3-static`?

---

## Quiz Answers

**A1:** A **Device** is the physical or virtual network interface (hardware layer, e.g., `ens3`).
A **Connection** is a configuration profile that tells NM how to configure that device
(IP, gateway, DNS, etc.). Multiple profiles can exist for one device but only one is active.

**A2:**
```bash
nmcli con add type ethernet con-name ens3-static ifname ens3 \
  ipv4.method manual ipv4.addresses 10.0.0.5/24 \
  ipv4.gateway 10.0.0.1 ipv4.dns 10.0.0.1 \
  connection.autoconnect yes
nmcli con up ens3-static
```

**A3:** The `8.8.8.8` DNS server is **removed**. Without a `+` prefix, `nmcli con mod`
**replaces** the entire value. Only `1.1.1.1` will remain.

**A4:** Use the `+` prefix:
```bash
nmcli con mod ens3-static +ipv4.dns 8.8.4.4
```

**A5:** The profile has `connection.autoconnect no` (or defaults to no). Fix:
```bash
nmcli con mod ens3-static connection.autoconnect yes
```

**A6:** NetworkManager writes DNS to `/etc/resolv.conf`. You **cannot** safely edit
it manually — NM overwrites manual changes when the connection is reactivated.
Always set DNS with `nmcli con mod PROFILE ipv4.dns "IP1 IP2"`.

**A7:**
```bash
hostnamectl set-hostname node1.rhcsa.lab
```
This updates `/etc/hostname` and is immediately active. Do NOT use the bare
`hostname` command — it is not persistent.

**A8:** `connection.autoconnect yes` tells NetworkManager to automatically activate
this connection profile when the interface becomes available — including at system boot.
Without it, you must manually run `nmcli con up` after every reboot.

**A9:** The problem is **DNS not configured** (or wrong DNS server).
- `8.8.8.8` works because it's an IP address (no DNS needed)
- `google.com` fails because name-to-IP translation (DNS) is broken

Fix:
```bash
nmcli con mod ens3-static ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con up ens3-static
cat /etc/resolv.conf   # Verify nameserver entries
```

**A10:**
```bash
nmcli con show ens3-static
# For just IPv4 settings:
nmcli con show ens3-static | grep ipv4
```

---

# ═══════════════════════════════════════════════════════════
# END OF CHAPTER 4 — NETWORK MANAGEMENT
# ═══════════════════════════════════════════════════════════

**Chapter Status:** ✅ COMPLETE
**Previous:** Chapter 3 — Stratis
**Next:** Chapter 5 — Remote Storage (NFS & Autofs)

---

*Reply "Chapter 5" to begin the NFS & Autofs chapter.*
*Reply "NETWORK DRILL" for 10 additional exam scenarios.*

