# RHCSA RHEL 9 — COMPLETE EXAM PREPARATION COURSE
## Chapter 1: Partitioning
### Senior Red Hat Linux Engineer · RHCSA/RHCE Instructor · 20+ Years Experience

---

> **How to use this chapter:** Read Section 1 once for theory. Study Section 2 commands actively (type every command). Complete every lab in Section 3 in order. Then drill the 20 exam scenarios until you can do them from memory in under 5 minutes each. The exam is 100% hands-on — speed and accuracy win.

---

# ═══════════════════════════════════════════════════════════
# SECTION 1 — THEORY: FROM ZERO TO RHCSA
# ═══════════════════════════════════════════════════════════

## 1.1 What Is a Partition? (Start Here)

A **partition** is a logically defined region of a physical storage device (HDD, SSD, NVMe). The operating system treats each partition as an independent unit with its own filesystem, mount point, and metadata.

Think of a raw disk like an empty land lot. Partitioning is dividing it into plots. Formatting each plot (creating a filesystem) is like building a structure on each plot. Mounting is connecting a road (path) to that structure so people (processes) can access it.

### Why Partition At All?

| Reason | Explanation |
|---|---|
| Isolation | A full `/var` log won't crash `/` (root) |
| Security | Mount partitions with `noexec`, `nosuid` flags |
| Performance | Separate I/O workloads per disk |
| Backup | Easier to snapshot individual partitions |
| Multi-boot | Each OS lives on its own partition |
| Quota | Limit space per filesystem |

---

## 1.2 Linux Storage Architecture — The Full Stack

```
╔══════════════════════════════════════════════════════╗
║                  USER SPACE                          ║
║   Applications (ls, cp, vim, Apache...)              ║
╠══════════════════════════════════════════════════════╣
║                 KERNEL SPACE                         ║
║  VFS (Virtual File System Layer)                     ║
║  ┌──────────┐  ┌──────────┐  ┌──────────┐           ║
║  │   XFS    │  │  ext4    │  │  tmpfs   │  ...       ║
║  └──────────┘  └──────────┘  └──────────┘           ║
║  Block Layer (I/O scheduler, request queues)         ║
║  Device Mapper (LVM, dm-crypt, RAID)                 ║
╠══════════════════════════════════════════════════════╣
║               HARDWARE LAYER                         ║
║  ┌─────────────────────────────────────────────┐     ║
║  │  Physical Disk (HDD / SSD / NVMe)           │     ║
║  │  ┌────────┬────────┬────────┬────────┐      │     ║
║  │  │  MBR/  │  Part1 │  Part2 │  Part3 │      │     ║
║  │  │  GPT   │  /boot │  /     │  swap  │      │     ║
║  │  └────────┴────────┴────────┴────────┘      │     ║
║  └─────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════╝
```

The kernel exposes block devices through the `/dev` virtual filesystem. Everything you do with `fdisk`, `parted`, `mkfs`, and `mount` goes through this stack.

---

## 1.3 Block Device Naming Conventions

Understanding device names is CRITICAL for the exam. One wrong device name and you wipe the wrong disk.

### SATA/SAS Disks (most common in exam VMs)

```
/dev/sda        → First SATA disk
/dev/sda1       → First partition on first SATA disk
/dev/sda2       → Second partition on first SATA disk
/dev/sdb        → Second SATA disk
/dev/sdb1       → First partition on second SATA disk
```

### NVMe Disks (modern SSDs)

```
/dev/nvme0      → First NVMe controller
/dev/nvme0n1    → First namespace (disk) on first controller
/dev/nvme0n1p1  → First partition on that disk
/dev/nvme0n1p2  → Second partition
/dev/nvme1n1    → Second NVMe disk
```

### Virtual Disks (KVM/QEMU — common in exam VMs)

```
/dev/vda        → First VirtIO disk
/dev/vda1       → First partition
/dev/vdb        → Second VirtIO disk
```

### Device Mapper (LVM, RAID)

```
/dev/mapper/vgname-lvname   → LVM logical volume via device mapper
/dev/dm-0                   → Device mapper device 0
```

### Key Rule: Never Assume — Always Verify

```bash
lsblk           # Always run this first to see the current disk layout
```

---

## 1.4 Partition Tables: MBR vs GPT

### MBR — Master Boot Record (Legacy, 1983)

```
Disk Layout with MBR:
┌──────────────────────────────────────────────────────────────────┐
│  Sector 0 (512 bytes)                                            │
│  ┌──────────────────────┬───────────────────────────────────┐    │
│  │  Bootstrap Code      │  Partition Table (4 entries max)  │    │
│  │  (446 bytes)         │  (64 bytes)    + 2 byte signature │    │
│  └──────────────────────┴───────────────────────────────────┘    │
│                                                                  │
│  ┌──────────┬──────────┬──────────┬──────────────────────────┐   │
│  │ Primary  │ Primary  │ Primary  │ Extended (contains       │   │
│  │ Part 1   │ Part 2   │ Part 3   │ logical partitions 5+)  │   │
│  └──────────┴──────────┴──────────┴──────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

**MBR Limitations (memorize these):**
- Max **4 primary** partitions
- Max disk size: **2 TiB**
- Extended partition = workaround (one primary holds many logical)
- Logical partitions start at **partition 5**
- Single point of failure: lose sector 0, lose everything

### GPT — GUID Partition Table (Modern, UEFI)

```
Disk Layout with GPT:
┌─────────────────────────────────────────────────────────────────┐
│  Sector 0: Protective MBR (backward compatibility)             │
├─────────────────────────────────────────────────────────────────┤
│  Sector 1: Primary GPT Header                                   │
├─────────────────────────────────────────────────────────────────┤
│  Sectors 2-33: Partition Table Entries (128 partitions max)     │
│  Each entry: 128 bytes, stores GUID, name, start/end LBA        │
├──────────┬──────────┬──────────┬──────────┬────────────────────┤
│ Part 1   │ Part 2   │ Part 3   │ Part 4   │  ... up to 128     │
├──────────┴──────────┴──────────┴──────────┴────────────────────┤
│  ... Data Area ...                                              │
├─────────────────────────────────────────────────────────────────┤
│  Backup Partition Table (mirror at end of disk)                 │
│  Backup GPT Header                                              │
└─────────────────────────────────────────────────────────────────┘
```

**GPT Advantages:**
- Up to **128 partitions** (no extended/logical concept)
- Disk size: **9.4 ZiB** (effectively unlimited)
- **Redundant headers** (backup at end of disk)
- CRC32 checksum per entry (detects corruption)
- Required for UEFI boot

### MBR vs GPT — Exam Decision Table

| Feature | MBR | GPT |
|---|---|---|
| Max partitions | 4 primary (+ logical) | 128 |
| Max disk size | 2 TiB | 9.4 ZiB |
| Boot mode | BIOS/Legacy | UEFI (preferred) |
| Redundancy | None | Backup header at end |
| RHCSA default | Older VMs | **RHEL 9 default** |
| Tool to use | `fdisk` | `gdisk` or `parted` |

> **EXAM TRAP:** On RHEL 9, new disks default to GPT. If you use `fdisk` on a fresh disk, it creates MBR by default. Use `parted` or `gdisk` for GPT. The exam expects you to know both.

---

## 1.5 Filesystem Types — What You Must Know

A partition is just raw space. A **filesystem** is the structure that organizes files within that space. You create a filesystem with `mkfs`.

### XFS — Default Filesystem in RHEL 9

```
XFS Architecture:
┌─────────────────────────────────────────────────────┐
│  XFS Filesystem                                     │
│  ┌──────────────┬──────────────┬──────────────┐     │
│  │  AG 0        │  AG 1        │  AG N        │     │
│  │ (Allocation  │ (Allocation  │ (Allocation  │     │
│  │  Group)      │  Group)      │  Group)      │     │
│  │ ┌──────────┐ │ ┌──────────┐ │ ┌──────────┐ │     │
│  │ │ Superblk │ │ │ Superblk │ │ │ Superblk │ │     │
│  │ │ Free map │ │ │ Free map │ │ │ Free map │ │     │
│  │ │ Inodes   │ │ │ Inodes   │ │ │ Inodes   │ │     │
│  │ │ Data blks│ │ │ Data blks│ │ │ Data blks│ │     │
│  │ └──────────┘ │ └──────────┘ │ └──────────┘ │     │
│  └──────────────┴──────────────┴──────────────┘     │
│  Journal (Write-ahead log for crash recovery)        │
└─────────────────────────────────────────────────────┘
```

- Default on RHEL 7, 8, 9
- **Cannot shrink** — only grow (exam trap!)
- Excellent performance for large files and parallel I/O
- 64-bit addressing, up to 8 EiB per filesystem
- Journal ensures filesystem consistency after crash

### ext4 — Extended Filesystem 4

- Widely compatible, stable
- **Can shrink** (with `resize2fs` offline)
- Max filesystem: 1 EiB; max file: 16 TiB
- Less performant than XFS for large parallel workloads
- Used when compatibility or shrink capability needed

### swap — Not a Real Filesystem

- Not mounted as a directory
- Used as virtual RAM overflow
- Managed by `mkswap`, `swapon`, `swapoff`
- Appears as `[SWAP]` in `lsblk`

### vfat/FAT32 — EFI System Partition

- Required for `/boot/efi` on UEFI systems
- Cross-platform compatible
- No permissions, no ownership

### Filesystem Comparison Table

| Feature | XFS | ext4 | vfat |
|---|---|---|---|
| RHEL 9 default | **YES** | No | No |
| Max size | 8 EiB | 1 EiB | 2 TiB |
| Shrink support | **NO** | YES | N/A |
| Grow support | YES | YES | N/A |
| Journal | YES | YES | NO |
| mkfs command | `mkfs.xfs` | `mkfs.ext4` | `mkfs.vfat` |
| RHCSA format | Most common | Common | Rare |

---

## 1.6 /etc/fstab — The Permanent Mount Configuration

`/etc/fstab` tells the system what to mount at boot. **This file is critical for the RHCSA exam.** A syntax error here can prevent the system from booting.

### Anatomy of /etc/fstab

```
# /etc/fstab — Field breakdown:
#
# <device>          <mount-point>  <fstype>  <options>          <dump>  <pass>
#
UUID=1234-abcd      /              xfs       defaults           0       1
UUID=5678-efgh      /boot          xfs       defaults           0       2
UUID=9abc-ijkl      swap           swap      defaults           0       0
/dev/sdb1           /data          xfs       defaults           0       2
//server/share      /mnt/nfs       nfs       defaults,_netdev   0       0
```

### Field-by-Field Explanation

**Field 1 — Device Identifier**

| Method | Example | Notes |
|---|---|---|
| UUID | `UUID=abc123` | **Preferred** — survives device rename |
| LABEL | `LABEL=mydata` | Human-readable, must be unique |
| Device path | `/dev/sdb1` | Risky — device name can change |
| Device mapper | `/dev/mapper/vg0-lv0` | For LVM volumes |

> **EXAM RULE:** Always use UUID or device mapper path in `/etc/fstab`. Never use `/dev/sdX` — it can change between boots.

**Field 2 — Mount Point**
The directory where the filesystem will be accessible. Must exist before mounting.

**Field 3 — Filesystem Type**
`xfs`, `ext4`, `swap`, `nfs`, `nfs4`, `vfat`, `tmpfs`, `auto`

**Field 4 — Mount Options (Comma-separated)**

| Option | Meaning |
|---|---|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `ro` | Read-only |
| `rw` | Read-write |
| `noexec` | Cannot execute binaries |
| `nosuid` | Ignore SUID/SGID bits |
| `nodev` | No device files |
| `noauto` | Don't mount at boot |
| `user` | Allow non-root to mount |
| `_netdev` | Wait for network (NFS, CIFS) |
| `nofail` | Don't fail boot if device missing |

**Field 5 — Dump (Backup)**
- `0` = No dump backup (use this almost always)
- `1` = Enable dump (legacy, rarely used)

**Field 6 — fsck Pass Order**
- `0` = Skip fsck check
- `1` = Check first (use for root `/`)
- `2` = Check after root (use for other local filesystems)

> **EXAM TRAP:** Set `/` to pass=1, all other local filesystems to pass=2, swap and network filesystems to pass=0. A wrong pass value won't break boot but will fail exam grading.

---

## 1.7 How Mounting Works Internally

```
mount /dev/sdb1 /data

Kernel steps:
1. Read partition table → locate /dev/sdb1 start/end LBA
2. Read superblock of filesystem on /dev/sdb1
3. Validate filesystem type and journal state
4. Register filesystem in VFS mount table
5. Bind /data directory inode to filesystem root inode
6. All paths under /data now resolve through /dev/sdb1

Result in /proc/mounts (runtime mount table):
/dev/sdb1 /data xfs rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota 0 0
```

The kernel maintains two mount tables:
- `/proc/mounts` — current runtime mounts (real state)
- `/etc/fstab` — persistent configuration (what to do at boot)

---

## 1.8 Swap Space — Extended Memory

Swap extends RAM using disk space. The kernel's memory manager (OOM killer) moves inactive memory pages to swap when RAM is low.

```
Memory Pressure Flow:
┌─────────┐     High RAM usage      ┌──────────────┐
│   RAM   │ ──────────────────────► │  Swap Space  │
│  (Fast) │ ◄────────────────────── │  (Disk/SSD)  │
└─────────┘     Pages recalled      └──────────────┘

Cold pages move to swap.
Hot pages stay in RAM.
If swap also fills → OOM killer terminates processes.
```

**Swap sizing guidelines (Red Hat official):**
| RAM | Recommended Swap |
|---|---|
| < 2 GB | 2x RAM |
| 2–8 GB | Equal to RAM |
| 8–64 GB | At least 4 GB |
| > 64 GB | At least 4 GB |

**Multiple swap spaces are additive and share load.**

---

## 1.9 Exam Traps and Common Mistakes in Partitioning

1. **Forgetting `partprobe`** after fdisk/gdisk changes — kernel doesn't see new partitions without it
2. **Wrong device name** — always `lsblk` first
3. **Not creating mount point** before `mount` — directory must exist
4. **Using `/dev/sdX` in fstab** — use UUID
5. **Forgetting `systemctl daemon-reload`** after editing fstab — required for systemd to re-read
6. **XFS cannot shrink** — never try `xfs_growfs` to shrink
7. **Pass field 0 for root** — root must be pass=1
8. **Not running `mkswap` before `swapon`** — the swap signature must exist
9. **Wrong fstype in fstab** — `xfs` not `XFS`, `swap` not `SWAP`
10. **Creating partition without writing changes** — must type `w` in fdisk before quitting

---

# ═══════════════════════════════════════════════════════════
# SECTION 2 — COMMANDS REFERENCE (EXAM-LEVEL DETAIL)
# ═══════════════════════════════════════════════════════════

## 2.1 lsblk — List Block Devices

**Your first command on any exam task involving storage.**

```bash
# Syntax
lsblk [OPTIONS] [DEVICE...]
```

| Option | Meaning | Example |
|---|---|---|
| (none) | Tree view of all block devices | `lsblk` |
| `-f` | Show filesystem info (UUID, type, label) | `lsblk -f` |
| `-o` | Choose output columns | `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID` |
| `-p` | Show full device path | `lsblk -p` |
| `-l` | List format (no tree) | `lsblk -l` |
| `-d` | Show disks only (no partitions) | `lsblk -d` |
| `-n` | No headers | `lsblk -n` |
| `-b` | Show size in bytes | `lsblk -b` |
| `-S` | Show SCSI devices | `lsblk -S` |

**Sample output:**
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   20G  0 disk
├─sda1        8:1    0    1G  0 part /boot
├─sda2        8:2    0    2G  0 part [SWAP]
└─sda3        8:3    0   17G  0 part /
sdb           8:16   0   10G  0 disk
```

**With -f (shows filesystem details):**
```
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1 xfs          boot  a1b2c3d4-...                        700M    30%    /boot
├─sda2 swap   1           e5f6a7b8-...                                       [SWAP]
└─sda3 xfs                9c0d1e2f-...                        14G     17%    /
```

---

## 2.2 blkid — Block Device Identifier

Used to get UUIDs and filesystem types for use in `/etc/fstab`.

```bash
# Syntax
blkid [OPTIONS] [DEVICE]
```

| Option | Meaning |
|---|---|
| (none) | Show all block devices with UUID and type |
| `/dev/sdb1` | Show only specific device |
| `-U UUID` | Find device by UUID |
| `-L LABEL` | Find device by label |
| `-t TYPE=xfs` | Filter by filesystem type |
| `-o value` | Output only the field value |
| `-s UUID` | Show only UUID field |

**Examples:**
```bash
blkid                         # All devices
blkid /dev/sdb1               # One device
blkid -U "abc123-..."         # Find device by UUID
blkid -s UUID -o value /dev/sdb1  # Get only the UUID string
```

**Sample output:**
```
/dev/sda1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="xfs" PARTUUID="..."
/dev/sdb1: UUID="11223344-5566-7788-9900-aabbccddeeff" TYPE="xfs"
```

---

## 2.3 fdisk — MBR Partition Editor (Interactive)

`fdisk` is an interactive partition editor, primarily for MBR disks. It **does not write changes until you type `w`** — this is your safety net.

```bash
# Start fdisk on a disk (NEVER on a partition)
fdisk /dev/sdb

# Non-interactive: just print partition table
fdisk -l /dev/sdb
fdisk -l             # All disks
```

### fdisk Interactive Commands

| Key | Action |
|---|---|
| `m` | **Help menu** (use this if you forget) |
| `p` | **Print** current partition table |
| `n` | **New** partition |
| `d` | **Delete** partition |
| `t` | Change partition **type** |
| `l` | **List** known partition types |
| `w` | **Write** changes and exit (COMMITS CHANGES) |
| `q` | **Quit** WITHOUT saving (safe exit) |
| `g` | Create new empty **GPT** partition table |
| `o` | Create new empty **MBR** partition table |
| `i` | Print info about a partition |
| `v` | **Verify** the partition table |

### fdisk Full Workflow — Creating a Partition

```bash
[root@server ~]# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.4).
...

Command (m for help): p          # Print current table

Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
...
Device   Boot  Start    End  Sectors  Size  Id  Type
(empty)

Command (m for help): n          # New partition
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p            # Primary partition

Partition number (1-4, default 1): 1   # Partition number
First sector (2048-20971519, default 2048): [ENTER]   # Accept default
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519): +5G

Created a new partition 1 of type 'Linux' and of size 5 GiB.

Command (m for help): p          # Verify

Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 10487807 10485760   5G 83 Linux

Command (m for help): w          # Write! (MANDATORY)
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### Partition Types (common for exam)

| Hex Code | Type | Use |
|---|---|---|
| `83` | Linux | Regular Linux partition (default) |
| `82` | Linux swap | Swap partition |
| `8e` | Linux LVM | LVM physical volume |
| `fd` | Linux RAID | Software RAID |
| `ef` | EFI (FAT-12/16/32) | EFI system partition |

---

## 2.4 gdisk — GPT Partition Editor (Interactive)

`gdisk` is the GPT equivalent of `fdisk`. Interface is nearly identical.

```bash
gdisk /dev/sdb
gdisk -l /dev/sdb    # List partition table (non-interactive)
```

### gdisk Interactive Commands

| Key | Action |
|---|---|
| `?` | Help menu |
| `p` | Print partition table |
| `n` | New partition |
| `d` | Delete partition |
| `t` | Change partition type |
| `l` | List GUID partition types |
| `w` | Write and exit |
| `q` | Quit without saving |
| `i` | Show info about a partition |
| `v` | Verify disk integrity |
| `z` | Zap (destroy) GPT data |

### GPT Partition Types (common)

| Type Code | Description |
|---|---|
| `8300` | Linux filesystem (default) |
| `8200` | Linux swap |
| `8e00` | Linux LVM |
| `8301` | Linux reserved |
| `ef00` | EFI System Partition |
| `fd00` | Linux RAID |

### gdisk Workflow Example

```bash
[root@server ~]# gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.8

Command (? for help): n          # New partition
Partition number (1-128, default 1): 1
First sector (default 2048): [ENTER]
Last sector (+sectors or +size{K,M,G,T,P}): +5G
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): [ENTER]

Command (? for help): p          # Print to verify
Command (? for help): w          # Write
Do you want to proceed? (Y/N): Y
```

---

## 2.5 parted — Non-Interactive / Scriptable Partition Editor

`parted` works with both MBR and GPT and can be used non-interactively (great for scripts and exam speed).

```bash
# Syntax
parted [OPTIONS] [DEVICE] [COMMAND [ARGUMENT...]]
```

| Option | Meaning |
|---|---|
| `-l` | List all partition tables |
| `-s` | Script mode (no prompts) — use in exam scripts |
| `--align optimal` | Align partitions for performance |

### parted Commands

| Command | Syntax | Description |
|---|---|---|
| `print` | `parted /dev/sdb print` | Show partition table |
| `mklabel` | `parted /dev/sdb mklabel gpt` | Create GPT table |
| `mklabel` | `parted /dev/sdb mklabel msdos` | Create MBR table |
| `mkpart` | `parted /dev/sdb mkpart primary xfs 1MiB 5GiB` | Create partition |
| `rm` | `parted /dev/sdb rm 1` | Delete partition 1 |
| `resizepart` | `parted /dev/sdb resizepart 1 10GiB` | Resize partition |
| `name` | `parted /dev/sdb name 1 'mydata'` | Name GPT partition |
| `set` | `parted /dev/sdb set 1 lvm on` | Set partition flag |

### parted Full Example — GPT + Partition in One Script

```bash
# Create GPT table and 5GB partition in one command sequence
parted -s /dev/sdb mklabel gpt \
       mkpart primary xfs 1MiB 5GiB

# Create swap partition starting right after
parted -s /dev/sdb mkpart primary linux-swap 5GiB 7GiB

# Verify
parted /dev/sdb print
```

### parted Alignment Explanation

Always start partitions at 1MiB (not sector 0 or 1) for proper alignment:
```bash
# CORRECT alignment
parted -s /dev/sdb mkpart primary xfs 1MiB 5GiB

# WRONG — misaligned, performance penalty
parted -s /dev/sdb mkpart primary xfs 0 5GiB
```

---

## 2.6 partprobe — Inform Kernel of Partition Table Changes

After any fdisk/gdisk/parted operation, the kernel's in-memory partition table may be stale. `partprobe` forces a re-read without rebooting.

```bash
# Syntax
partprobe [DEVICE]

# Examples
partprobe              # Update all devices
partprobe /dev/sdb     # Update specific disk
```

> **EXAM RULE:** Always run `partprobe` after creating/modifying partitions. If it fails, reboot (but that's a last resort on the exam).

Alternative:
```bash
udevadm settle         # Wait for udev to process events
udevadm trigger        # Trigger udev events
```

---

## 2.7 mkfs — Create Filesystems

```bash
# General syntax
mkfs [-t TYPE] [OPTIONS] DEVICE
```

### mkfs.xfs — Create XFS Filesystem

```bash
# Syntax
mkfs.xfs [OPTIONS] DEVICE

# Options
-f          # Force overwrite (if filesystem already exists)
-L LABEL    # Set filesystem label (max 12 chars for XFS)
-b size=N   # Block size (512, 1024, 2048, 4096)
-d file     # Use a file as filesystem (rare)
-i size=N   # Inode size
```

**Examples:**
```bash
mkfs.xfs /dev/sdb1               # Basic XFS filesystem
mkfs.xfs -L "mydata" /dev/sdb1   # With label
mkfs.xfs -f /dev/sdb1            # Force overwrite existing filesystem
```

### mkfs.ext4 — Create ext4 Filesystem

```bash
# Syntax
mkfs.ext4 [OPTIONS] DEVICE

# Options
-L LABEL    # Set filesystem label
-b SIZE     # Block size (1024, 2048, 4096)
-m N        # Reserved blocks percentage (default 5%)
-j          # Create with journal (default for ext4)
```

**Examples:**
```bash
mkfs.ext4 /dev/sdb2               # Basic ext4
mkfs.ext4 -L "backup" /dev/sdb2   # With label
mkfs.ext4 -m 0 /dev/sdb2          # No reserved blocks (for data partitions)
```

### mkswap — Prepare Swap Partition

```bash
# Syntax
mkswap [OPTIONS] DEVICE

# Options
-L LABEL    # Set swap label
-U UUID     # Use specific UUID

# Example
mkswap /dev/sdb3
mkswap -L "swap1" /dev/sdb3
```

---

## 2.8 mount / umount — Mount and Unmount Filesystems

### mount

```bash
# Syntax
mount [OPTIONS] DEVICE MOUNTPOINT
mount [OPTIONS] MOUNTPOINT          # Mount from /etc/fstab

# Options
-t TYPE          # Filesystem type (often auto-detected)
-o OPTIONS       # Mount options (rw, ro, noexec, etc.)
-a               # Mount all filesystems in /etc/fstab
-r               # Read-only
-w               # Read-write
-v               # Verbose
--bind           # Bind mount (share a directory)
-l               # List all mounts
```

**Examples:**
```bash
# Manual mount
mount /dev/sdb1 /data
mount -t xfs /dev/sdb1 /data
mount -o ro /dev/sdb1 /data          # Read-only
mount -o remount,rw /data            # Remount as read-write

# Mount from fstab
mount -a                              # Mount all fstab entries
mount /data                          # Mount specific fstab entry

# Show all current mounts
mount | grep /data
mount -l

# Bind mount (make directory accessible at another path)
mount --bind /source /destination
```

### umount

```bash
# Syntax
umount [OPTIONS] DEVICE|MOUNTPOINT

# Options
-l          # Lazy unmount (detach now, cleanup later)
-f          # Force unmount (for NFS)
-a          # Unmount all (except proc, sysfs, etc.)

# Examples
umount /data
umount /dev/sdb1
umount -l /data     # Use when "device is busy"
```

**"Device or resource busy" error fix:**
```bash
# Find what's using the filesystem
fuser -mv /data          # Show processes using /data
lsof /data               # List open files on /data
fuser -k /data           # Kill processes (careful!)
cd /                     # Make sure you're not IN the mount point
umount /data
```

---

## 2.9 swapon / swapoff — Manage Swap Space

```bash
# Syntax
swapon [OPTIONS] [DEVICE|FILE]
swapoff [OPTIONS] [DEVICE|FILE]

# swapon Options
-a          # Activate all swap in /etc/fstab
-s          # Summary of current swap usage (same as 'free -h')
-p PRIORITY # Set priority (higher = used first, -1 to 32767)
-v          # Verbose
--show      # Show swap with headers

# Examples
mkswap /dev/sdb3           # Step 1: prepare (do this first!)
swapon /dev/sdb3           # Step 2: activate
swapon -a                  # Activate all fstab swap
swapon --show              # Show active swap
swapoff /dev/sdb3          # Deactivate
swapoff -a                 # Deactivate all

# Verify
free -h                    # Shows Mem and Swap totals
cat /proc/swaps            # Shows active swap devices
```

---

## 2.10 df / du — Disk Usage

```bash
# df — Disk Free (filesystem level)
df [OPTIONS] [FILE...]
df -h                      # Human-readable sizes (KB, MB, GB)
df -H                      # Human-readable (powers of 1000)
df -T                      # Show filesystem type
df -i                      # Show inode usage
df -x tmpfs                # Exclude filesystem type
df /data                   # Show only the filesystem of /data
df -h --output=source,size,used,avail,pcent,target  # Custom columns

# du — Disk Usage (file/directory level)
du [OPTIONS] [FILE...]
du -h /data                # Human-readable, recursive
du -sh /data               # Summary only (total)
du -sh /*                  # Top-level directory sizes
du -h --max-depth=1 /var   # One level deep
du -ah /data               # Show all files
du -c /data /logs          # Show total of multiple paths
```

---

## 2.11 xfs_info / xfs_admin / xfs_repair

```bash
# xfs_info — Display XFS filesystem parameters
xfs_info /dev/sdb1         # Must be mounted
xfs_info /data             # Can use mount point

# Sample output:
# meta-data=/dev/sdb1    isize=512   agcount=4, agsize=327680 blks
# data     =             bsize=4096  blocks=1310720, imaxpct=25
# naming   =version 2    bsize=4096  ascii-ci=0, ftype=1
# log      =internal log bsize=4096  blocks=2560, version=2
# realtime =none         extsz=4096  blocks=0, rtextents=0

# xfs_admin — Modify XFS filesystem attributes (unmounted)
xfs_admin -l /dev/sdb1             # Show label
xfs_admin -L "newlabel" /dev/sdb1  # Change label (unmounted)
xfs_admin -u /dev/sdb1             # Show UUID

# xfs_repair — Repair damaged XFS filesystem (unmounted)
xfs_repair /dev/sdb1               # Repair
xfs_repair -n /dev/sdb1            # Dry run (check only, no changes)
```

---

## 2.12 tune2fs / e2fsck — ext4 Filesystem Management

```bash
# tune2fs — Adjust ext4 parameters
tune2fs -l /dev/sdb2               # List filesystem info
tune2fs -L "datalabel" /dev/sdb2   # Set label
tune2fs -c 50 /dev/sdb2            # Max mounts before fsck check
tune2fs -m 2 /dev/sdb2             # Reduce reserved space to 2%

# e2fsck — Check/repair ext4 (unmounted only)
e2fsck /dev/sdb2                   # Interactive check
e2fsck -f /dev/sdb2                # Force check even if clean
e2fsck -p /dev/sdb2                # Auto-fix (preen mode)
e2fsck -n /dev/sdb2                # Dry run (no changes)
```

---

## 2.13 resize2fs / xfs_growfs — Grow Filesystems

```bash
# resize2fs — Resize ext4 (can grow online or shrink offline)
resize2fs /dev/sdb2                # Grow to fill partition
resize2fs /dev/sdb2 8G             # Resize to exact size
resize2fs /dev/sdb2 +2G            # Add 2G (needs e2fsck first if shrinking)

# To SHRINK ext4 (must unmount first):
umount /dev/sdb2
e2fsck -f /dev/sdb2
resize2fs /dev/sdb2 4G             # Shrink to 4G
# Then shrink the partition itself with fdisk/parted

# xfs_growfs — Grow XFS (online, filesystem must be mounted)
xfs_growfs /data                   # Grow to fill expanded partition
xfs_growfs -D 2097152 /data        # Grow to specific block count
# NOTE: XFS CANNOT SHRINK — this is a permanent limitation
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 3 — PRACTICAL LABS
# ═══════════════════════════════════════════════════════════

---

## LAB 1 (Beginner) — Identify and Explore Your Disks

### Objective
Understand the current disk layout without modifying anything.

### Initial Situation
A RHEL 9 VM with one system disk (`/dev/sda`) and one empty disk (`/dev/sdb`, 10GB).

### Tasks

**Task 1.1 — List all block devices and understand the output**
```bash
lsblk
lsblk -f
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID
```

**Task 1.2 — Get UUID and filesystem type of each partition**
```bash
blkid
blkid /dev/sda1
blkid -s UUID -o value /dev/sda1   # Get just the UUID
```

**Task 1.3 — Inspect the current partition table**
```bash
fdisk -l /dev/sda
fdisk -l /dev/sdb
parted /dev/sda print
parted -l
```

**Task 1.4 — Check current mounts and disk usage**
```bash
mount | grep -v "cgroup\|proc\|sys\|tmpfs"   # Real mounts only
df -hT                                         # Disk usage with types
cat /etc/fstab
cat /proc/mounts
```

**Task 1.5 — Check swap**
```bash
swapon --show
free -h
cat /proc/swaps
```

### Expected Results
You can identify all disks, their partitions, filesystem types, UUIDs, mount points, and swap usage without running any commands you don't understand.

### Verification Steps
```bash
# You should be able to answer:
# - How many disks does this system have?
lsblk -d
# - What filesystem is on /boot?
df -T /boot
# - What is the UUID of /dev/sda1?
blkid /dev/sda1
```

---

## LAB 2 (Beginner) — Create Your First Partition

### Objective
Create a 2GB Linux partition on `/dev/sdb` using `fdisk`.

### Initial Situation
`/dev/sdb` is a 10GB empty disk with no partitions.

### Pre-Lab Verification
```bash
lsblk /dev/sdb   # Confirm it's empty
```

### Tasks

**Task 2.1 — Create a 2GB partition using fdisk**
```bash
fdisk /dev/sdb
# Inside fdisk:
# n → new partition
# p → primary
# 1 → partition number 1
# [ENTER] → accept default first sector
# +2G → 2 gigabytes
# p → print and verify
# w → write and exit
```

**Task 2.2 — Inform kernel of new partition table**
```bash
partprobe /dev/sdb
```

**Task 2.3 — Verify the new partition**
```bash
lsblk /dev/sdb
fdisk -l /dev/sdb
```

**Task 2.4 — Format as XFS**
```bash
mkfs.xfs /dev/sdb1
```

**Task 2.5 — Create mount point and mount**
```bash
mkdir -p /mnt/mydata
mount /dev/sdb1 /mnt/mydata
```

**Task 2.6 — Verify the mount**
```bash
mount | grep mydata
df -h /mnt/mydata
lsblk -f /dev/sdb1
```

**Task 2.7 — Create a test file and verify persistence**
```bash
echo "test data" > /mnt/mydata/testfile.txt
cat /mnt/mydata/testfile.txt
ls -la /mnt/mydata/
```

### Solution
```bash
# Complete one-block solution:
fdisk /dev/sdb << 'EOF'
n
p
1

+2G
w
EOF
partprobe /dev/sdb
mkfs.xfs /dev/sdb1
mkdir -p /mnt/mydata
mount /dev/sdb1 /mnt/mydata
echo "Lab 2 complete" > /mnt/mydata/test.txt
df -h /mnt/mydata
```

### Expected Result
`/dev/sdb1` is 2GB, formatted as XFS, mounted at `/mnt/mydata`, and contains a test file.

---

## LAB 3 (Intermediate) — Permanent Mount with UUID

### Objective
Make the partition from Lab 2 mount automatically at boot using UUID in `/etc/fstab`.

### Tasks

**Task 3.1 — Get the UUID**
```bash
blkid /dev/sdb1
# OR
lsblk -f /dev/sdb1
```

**Task 3.2 — Add entry to /etc/fstab**
```bash
# First, backup /etc/fstab
cp /etc/fstab /etc/fstab.backup

# Get UUID (copy the output)
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /mnt/mydata  xfs  defaults  0  2" >> /etc/fstab

# Verify the line was added correctly
tail -3 /etc/fstab
```

**Task 3.3 — Test the fstab entry without rebooting**
```bash
umount /mnt/mydata           # Unmount first
mount -a                      # Mount all fstab entries
mount | grep mydata          # Verify it mounted
```

**Task 3.4 — Use systemd to validate fstab**
```bash
systemctl daemon-reload
# If this fails, fstab has a syntax error!

# Test with findmnt
findmnt --verify
```

**Task 3.5 — Reboot test (optional but thorough)**
```bash
reboot
# After reboot:
mount | grep mydata
df -h /mnt/mydata
```

### Solution (complete)
```bash
cp /etc/fstab /etc/fstab.backup
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /mnt/mydata  xfs  defaults  0  2" | tee -a /etc/fstab
umount /mnt/mydata
mount -a
mount | grep mydata && echo "SUCCESS" || echo "FAILED - check fstab"
```

### Verification
```bash
grep mydata /etc/fstab
findmnt /mnt/mydata
```

---

## LAB 4 (Intermediate) — Swap Partition

### Objective
Create, format, activate, and permanently configure a 1GB swap partition.

### Tasks

**Task 4.1 — Create 1GB partition for swap**
```bash
fdisk /dev/sdb
# n → p → 2 → [ENTER] → +1G → w
# Then change type to swap:
# t → 2 → 82 → w
partprobe /dev/sdb
```

**Task 4.2 — Format as swap**
```bash
mkswap /dev/sdb2
# Note: Do NOT use mkfs.swap — it doesn't exist!
```

**Task 4.3 — Activate swap**
```bash
swapon /dev/sdb2
swapon --show      # Verify
free -h            # Check Swap row
```

**Task 4.4 — Add to /etc/fstab for persistence**
```bash
UUID=$(blkid -s UUID -o value /dev/sdb2)
echo "UUID=${UUID}  swap  swap  defaults  0  0" >> /etc/fstab
```

**Task 4.5 — Test**
```bash
swapoff /dev/sdb2
swapon -a           # Activates from fstab
swapon --show
```

### Expected Result
```
NAME       TYPE      SIZE USED PRIO
/dev/sda2  partition   2G   0B   -2
/dev/sdb2  partition   1G   0B   -3
```

---

## LAB 5 (RHCSA Exam Level) — Full Disk Configuration Workflow

### Objective
Simulate a complete RHCSA partitioning task from scratch.

### Scenario
"A new 10GB disk `/dev/sdb` has been added to the system. Create:
- A 3GB XFS partition mounted at `/data`, permanently
- A 1GB swap partition, permanently active
- A 2GB ext4 partition mounted at `/backup` with label 'BACKUP', permanently"

### Solution

```bash
# STEP 1: Verify the disk
lsblk /dev/sdb
fdisk -l /dev/sdb

# STEP 2: Create partitions
fdisk /dev/sdb << 'EOF'
g
n
1

+3G
n
2

+1G
t
2
19
n
3

+2G
p
w
EOF

# g = GPT, partition type 19 = Linux swap in GPT (or use parted)

partprobe /dev/sdb
lsblk /dev/sdb

# STEP 3: Format each partition
mkfs.xfs /dev/sdb1
mkswap /dev/sdb2
mkfs.ext4 -L "BACKUP" /dev/sdb3

# STEP 4: Create mount points
mkdir -p /data /backup

# STEP 5: Get UUIDs
UUID1=$(blkid -s UUID -o value /dev/sdb1)
UUID2=$(blkid -s UUID -o value /dev/sdb2)
UUID3=$(blkid -s UUID -o value /dev/sdb3)

# STEP 6: Add to /etc/fstab
cat >> /etc/fstab << EOF
UUID=${UUID1}  /data    xfs   defaults  0  2
UUID=${UUID2}  swap     swap  defaults  0  0
UUID=${UUID3}  /backup  ext4  defaults  0  2
EOF

# STEP 7: Activate everything
mount -a
swapon -a
systemctl daemon-reload

# STEP 8: Verify
df -hT /data /backup
swapon --show
mount | grep -E "data|backup"
lsblk -f /dev/sdb
```

---

## LAB 6 (Troubleshooting) — Debug a Failed Mount

### Scenario
After adding an entry to `/etc/fstab`, the system won't mount `/data`. Diagnose and fix.

### The Problem (intentionally broken fstab entry)
```
# Broken entries to diagnose:
/dev/sdx1  /data    xfs  defaults  0  2   # Wrong device
UUID=wrong-uuid  /data  xfs  defaults  0  2  # Wrong UUID
/dev/sdb1  /data   xfs  defaults  0  2   # No directory /data
/dev/sdb1  /data   XFS  defaults  0  2   # Wrong fstype (capital)
```

### Diagnosis Steps

```bash
# Step 1: Try to mount and read the error
mount /data 2>&1

# Step 2: Verify the device exists
ls -la /dev/sdb1
lsblk

# Step 3: Verify UUID is correct
blkid | grep sdb1
grep data /etc/fstab

# Step 4: Check mount point exists
ls -la /data || echo "Directory missing!"

# Step 5: Test mount manually
mount /dev/sdb1 /data   # Bypass fstab

# Step 6: Validate fstab syntax
findmnt --verify
mount --fake -av          # Dry run all mounts
```

### Fix Procedure

```bash
# Fix 1: Create missing mount point
mkdir -p /data

# Fix 2: Correct the UUID in fstab
CORRECT_UUID=$(blkid -s UUID -o value /dev/sdb1)
# Edit /etc/fstab with correct UUID
sed -i "s/UUID=wrong-uuid/UUID=${CORRECT_UUID}/" /etc/fstab

# Fix 3: Fix wrong fstype (change XFS to xfs)
sed -i 's/ XFS / xfs /' /etc/fstab

# Validate
mount -a
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 4 — 20 RHCSA EXAM SCENARIOS
# ═══════════════════════════════════════════════════════════

---

### SCENARIO 1
**Task:** Create a 1GB partition on `/dev/sdb`, format it as XFS, and mount it permanently at `/data`. The mount must survive a reboot.

**Thinking Process:**
1. What disk? → `/dev/sdb`
2. Size? → 1GB
3. Format? → XFS → `mkfs.xfs`
4. Mount point? → `/data` → must create directory
5. Permanent? → `/etc/fstab` with UUID

**Full Solution:**
```bash
fdisk /dev/sdb           # n,p,1,[ENTER],+1G,w
partprobe /dev/sdb
mkfs.xfs /dev/sdb1
mkdir -p /data
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /data  xfs  defaults  0  2" >> /etc/fstab
mount -a
```

**Verification:**
```bash
df -hT /data              # Shows xfs, ~1GB
findmnt /data             # Shows UUID source
grep data /etc/fstab      # Entry exists
```

---

### SCENARIO 2
**Task:** Add 2GB of swap space using a new partition on `/dev/sdb`. Make it permanent.

**Full Solution:**
```bash
# Assumes /dev/sdb has space or is a fresh disk
fdisk /dev/sdb           # n,p,2,[ENTER],+2G, t,2,82, w
partprobe /dev/sdb
mkswap /dev/sdb2
swapon /dev/sdb2
UUID=$(blkid -s UUID -o value /dev/sdb2)
echo "UUID=${UUID}  swap  swap  defaults  0  0" >> /etc/fstab
```

**Verification:**
```bash
swapon --show
free -h
```

---

### SCENARIO 3
**Task:** Create a 500MB ext4 partition with label `ARCHIVE` on `/dev/sdc`. Mount it permanently at `/archive` with the `nosuid` and `noexec` options.

**Full Solution:**
```bash
fdisk /dev/sdc           # n,p,1,[ENTER],+500M,w
partprobe /dev/sdc
mkfs.ext4 -L "ARCHIVE" /dev/sdc1
mkdir -p /archive
UUID=$(blkid -s UUID -o value /dev/sdc1)
echo "UUID=${UUID}  /archive  ext4  defaults,nosuid,noexec  0  2" >> /etc/fstab
mount -a
```

**Verification:**
```bash
mount | grep archive    # Shows nosuid,noexec options
df -hT /archive
tune2fs -l /dev/sdc1 | grep "Filesystem volume"  # Shows label
```

---

### SCENARIO 4
**Task:** A partition `/dev/sdb1` has been created but isn't mounting at boot. Find and fix the issue in `/etc/fstab`.

**Solution approach:**
```bash
blkid /dev/sdb1                    # Get real UUID
grep sdb1 /etc/fstab               # Compare with fstab
findmnt --verify                   # Check syntax
# Fix UUID mismatch:
REAL=$(blkid -s UUID -o value /dev/sdb1)
sed -i "s|UUID=.*  /data|UUID=${REAL}  /data|" /etc/fstab
mount -a
```

---

### SCENARIO 5
**Task:** Extend the root filesystem. A new disk `/dev/sdb` is available. (Note: On RHCSA, root extension usually requires LVM — but this tests partitioning knowledge.) Create a separate `/home` partition and move data to it.

**Solution:**
```bash
fdisk /dev/sdb           # Create partition covering available space
partprobe /dev/sdb
mkfs.xfs /dev/sdb1
mkdir /mnt/newhome
mount /dev/sdb1 /mnt/newhome
rsync -av /home/ /mnt/newhome/
umount /mnt/newhome
# Backup /etc/fstab
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /home  xfs  defaults  0  2" >> /etc/fstab
mount -a
```

---

### SCENARIO 6
**Task:** Using `parted`, create a GPT disk with a 4GB XFS partition on `/dev/sdb`.

**Full Solution:**
```bash
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary xfs 1MiB 4097MiB
partprobe /dev/sdb
mkfs.xfs /dev/sdb1
mkdir -p /data4
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /data4  xfs  defaults  0  2" >> /etc/fstab
mount -a
```

---

### SCENARIO 7
**Task:** Verify the filesystem on `/dev/sdb1` (ext4) for errors without making any changes.

**Full Solution:**
```bash
umount /dev/sdb1 2>/dev/null       # Must be unmounted
e2fsck -n /dev/sdb1                # -n = no changes (dry run)
# For XFS:
xfs_repair -n /dev/sdb1
```

---

### SCENARIO 8
**Task:** Change the label of an XFS filesystem on `/dev/sdb1` to `NEWLABEL`.

**Full Solution:**
```bash
# XFS: use xfs_admin (filesystem must be unmounted for label change)
umount /dev/sdb1
xfs_admin -L "NEWLABEL" /dev/sdb1
mount /dev/sdb1 /data
xfs_admin -l /dev/sdb1             # Verify: label = "NEWLABEL"
# Or with lsblk:
lsblk -f /dev/sdb1
```

---

### SCENARIO 9
**Task:** Show all currently mounted filesystems and their sizes in a human-readable format. Save the output to `/root/mounts.txt`.

**Full Solution:**
```bash
df -hT > /root/mounts.txt
# Also:
mount > /root/mounts.txt
findmnt >> /root/mounts.txt
cat /root/mounts.txt
```

---

### SCENARIO 10
**Task:** Create a swap file of 512MB on an already partitioned system (no free partitions available).

**Full Solution:**
```bash
# Create the file
dd if=/dev/zero of=/swapfile bs=1M count=512
# OR faster with fallocate:
fallocate -l 512M /swapfile

# Secure it
chmod 600 /swapfile

# Format as swap
mkswap /swapfile

# Activate
swapon /swapfile

# Make permanent
echo "/swapfile  swap  swap  defaults  0  0" >> /etc/fstab
```

**Verification:**
```bash
swapon --show
free -h
ls -lh /swapfile
```

---

### SCENARIO 11
**Task:** Determine which filesystem uses the most space and which has the most inodes used.

**Full Solution:**
```bash
# Most space:
df -h --output=source,size,used,pcent | sort -k3 -rh | head -5

# Most inodes:
df -i --output=source,itotal,iused,ipcent | sort -k3 -rn | head -5
```

---

### SCENARIO 12
**Task:** A disk was removed and `/etc/fstab` still has its entry, causing boot failure. Fix without booting into recovery mode.

**Understanding:** If the system is already failing at boot, use emergency/rescue mode or `nofail` option.

**Fix (if you can still log in):**
```bash
# Remove the broken entry
sed -i '/\/dev\/sdb1/d' /etc/fstab
# OR comment it out
sed -i 's|^UUID=abc123|#UUID=abc123|' /etc/fstab
mount -a                           # Test remaining entries
```

**Prevention — use `nofail`:**
```bash
echo "UUID=abc123  /data  xfs  defaults,nofail  0  2" >> /etc/fstab
```

---

### SCENARIO 13
**Task:** Create a 2GB partition, format as ext4, mount at `/reports`. The partition must only be readable by all users but not executable.

**Full Solution:**
```bash
fdisk /dev/sdb   # Create partition
partprobe /dev/sdb
mkfs.ext4 /dev/sdb1
mkdir -p /reports
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /reports  ext4  defaults,noexec  0  2" >> /etc/fstab
mount -a
# Verify:
mount | grep reports    # Should show noexec
```

---

### SCENARIO 14
**Task:** Display detailed XFS filesystem information for the partition mounted at `/data`.

**Full Solution:**
```bash
xfs_info /data              # Mounted filesystem info
# OR
xfs_info /dev/sdb1          # By device
# Key output to understand:
# bsize=4096 → 4KB block size
# blocks=1310720 → total blocks
# agcount=4 → 4 allocation groups
```

---

### SCENARIO 15
**Task:** List all filesystems of type XFS on the system.

**Full Solution:**
```bash
lsblk -f | grep xfs
blkid -t TYPE=xfs
df -T | grep xfs
findmnt -t xfs
```

---

### SCENARIO 16
**Task:** A partition `/dev/sdb1` is mounted at `/data`. Without unmounting it, grow its XFS filesystem after the underlying partition was extended.

**Solution (assumes partition was already extended via fdisk/parted):**
```bash
# After extending partition:
partprobe /dev/sdb
# Grow XFS online (no unmount needed):
xfs_growfs /data
# Verify:
df -h /data
xfs_info /data
```

---

### SCENARIO 17
**Task:** Mount `/dev/sdb1` read-only, then verify that you cannot write to it.

**Full Solution:**
```bash
mount -o ro /dev/sdb1 /data
# Verify:
mount | grep /data         # Shows 'ro'
touch /data/test 2>&1      # Should fail: "Read-only file system"
```

---

### SCENARIO 18
**Task:** Find all mounted filesystems larger than 5GB.

**Full Solution:**
```bash
df -BG --output=source,size,target | awk 'NR>1 && $2+0 > 5 {print}'
# Simpler:
df -h | awk 'NR>1 {print}' | sort -k2 -rh
```

---

### SCENARIO 19
**Task:** Create a partition table backup and restore from it.

**Full Solution:**
```bash
# Backup MBR (includes partition table for MBR disk)
dd if=/dev/sdb of=/root/sdb_mbr.bak bs=512 count=1

# For GPT:
sgdisk --backup=/root/sdb_gpt.bak /dev/sdb

# Restore GPT:
sgdisk --load-backup=/root/sdb_gpt.bak /dev/sdb
partprobe /dev/sdb
```

---

### SCENARIO 20
**Task:** Configure a partition to mount at `/cache` with `noatime` option for better performance. Make it persistent.

**Full Solution:**
```bash
# noatime: don't update access time on reads (big I/O improvement)
mkdir -p /cache
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID}  /cache  xfs  defaults,noatime  0  2" >> /etc/fstab
mount -a
mount | grep cache         # Verify noatime in options
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 5 — TROUBLESHOOTING GUIDE
# ═══════════════════════════════════════════════════════════

## FAILURE 1: "mount: /data: special device /dev/sdb1 does not exist"

**Cause:** Partition doesn't exist, or kernel hasn't been notified.

**Diagnosis:**
```bash
lsblk                      # Is sdb1 listed?
ls /dev/sdb*               # Check device files
dmesg | tail -20           # Kernel messages about disk
```

**Fix:**
```bash
partprobe /dev/sdb         # Re-read partition table
udevadm settle             # Wait for udev
# If still missing — reboot or check physical connection
```

---

## FAILURE 2: "mount: /data: wrong fs type, bad option, bad superblock"

**Cause:** Wrong filesystem type in fstab, or corrupted superblock.

**Diagnosis:**
```bash
blkid /dev/sdb1            # What type does the system think it is?
grep data /etc/fstab       # What type is in fstab?
```

**Fix:**
```bash
# Fix type mismatch in fstab:
sed -i 's|/data  ext4|/data  xfs|' /etc/fstab

# Fix corrupted XFS:
xfs_repair /dev/sdb1

# Fix corrupted ext4:
e2fsck -f /dev/sdb1
```

---

## FAILURE 3: System Won't Boot After /etc/fstab Change

**Cause:** Syntax error in `/etc/fstab` or missing device.

**Prevention:**
```bash
# Always test before rebooting:
mount -a                   # Test all entries
findmnt --verify           # Syntax check
systemctl daemon-reload    # Check systemd sees it
```

**Recovery (if system won't boot):**
1. Boot to RHEL rescue/emergency mode
2. At `dracut` prompt or emergency shell:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
vi /etc/fstab              # Fix the error
# OR
cp /etc/fstab.backup /etc/fstab   # Restore backup
exit
reboot
```

Or at GRUB: append `systemd.unit=emergency.target` to kernel line.

---

## FAILURE 4: "device is busy" When Unmounting

**Cause:** A process is using a file or directory on the filesystem.

**Diagnosis:**
```bash
fuser -mv /data            # Which processes are using /data?
lsof +D /data              # List open files
lsof /data                 # Also works
```

**Fix:**
```bash
fuser -km /data            # Kill all processes using /data
umount /data               # Then unmount

# OR softer approach:
umount -l /data            # Lazy unmount (detaches immediately)
```

---

## FAILURE 5: UUID Mismatch in /etc/fstab

**Cause:** Disk was replaced, reformatted, or UUID was changed.

**Diagnosis:**
```bash
blkid | grep sdb1          # Real UUID
grep UUID /etc/fstab       # fstab UUID
# If they don't match → won't mount
```

**Fix:**
```bash
# Get correct UUID
REAL_UUID=$(blkid -s UUID -o value /dev/sdb1)
# Edit /etc/fstab to use correct UUID
sed -i "s|UUID=old-uuid-here|UUID=${REAL_UUID}|" /etc/fstab
mount -a
```

---

## FAILURE 6: Swap Not Active After Reboot

**Cause:** Missing or incorrect fstab entry, or mkswap not run.

**Diagnosis:**
```bash
swapon --show              # Is it active?
blkid /dev/sdb2            # Is it typed as swap?
grep swap /etc/fstab       # Is entry in fstab?
```

**Fix:**
```bash
# If not formatted:
mkswap /dev/sdb2

# If formatted but not in fstab:
UUID=$(blkid -s UUID -o value /dev/sdb2)
echo "UUID=${UUID}  swap  swap  defaults  0  0" >> /etc/fstab

# Activate without reboot:
swapon -a
```

---

## FAILURE 7: "No space left on device" But df Shows Free Space

**Cause:** Inode exhaustion — out of inodes even though data space exists.

**Diagnosis:**
```bash
df -h /data    # Shows space available
df -i /data    # Shows 100% inode usage
```

**Fix:**
```bash
# Find directories with many small files:
for dir in /data/*; do echo "$dir: $(find "$dir" | wc -l)"; done | sort -t: -k2 -rn

# Delete unnecessary files or move them
# For XFS: resize isn't needed — just delete files
# For ext4: would need reformat with more inodes (destructive)
```

---

## FAILURE 8: "XFS filesystem has duplicate UUID"

**Cause:** Disk was cloned — both copies have same UUID.

**Diagnosis:**
```bash
blkid | sort    # Two devices with same UUID
```

**Fix:**
```bash
# Assign new UUID to one of them (unmount first)
umount /dev/sdb1
xfs_admin -U generate /dev/sdb1    # Generate new UUID
blkid /dev/sdb1                     # Confirm new UUID
# Update fstab with new UUID
```

---

## FAILURE 9: Partition Created But Not Visible in /dev

**Cause:** Kernel partition table not updated.

**Diagnosis:**
```bash
cat /proc/partitions         # Kernel's view
ls /dev/sdb*                 # Device files
```

**Fix:**
```bash
partprobe /dev/sdb
udevadm settle
# If still not visible:
echo 1 > /sys/block/sdb/device/rescan
# Last resort:
reboot
```

---

## FAILURE 10: "mount.nfs: No route to host" (Mentioned as Bonus)

This applies to NFS chapter, but if a partition is labeled as _netdev and network is down:
```bash
systemctl status NetworkManager
# Remove _netdev from non-NFS entry in fstab if this was a mistake
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 6 — MEMORY AIDS
# ═══════════════════════════════════════════════════════════

## The PCFM Workflow (Every Partition Task)

```
P — Partition    → fdisk / gdisk / parted
C — Create FS    → mkfs.xfs / mkfs.ext4 / mkswap
F — Fstab        → UUID, mount point, type, options, dump, pass
M — Mount        → mkdir mount_point; mount -a
```

**Memory hook:** "Please Create Files Mounted"

---

## /etc/fstab Field Mnemonic

```
"Devices Must Format Options Doubtfully Passionately"
  D      M       F       O       D        P
  Device Mount  Format  Options Dump     Pass
```

Pass values: 0=none, 1=root, 2=others

---

## Partition Tool Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║  PARTITIONING QUICK REFERENCE                                    ║
╠══════════╦══════════════╦═════════════════════════════════════╣
║  Tool    ║  Table Type  ║  Best For                           ║
╠══════════╬══════════════╬═════════════════════════════════════╣
║  fdisk   ║  MBR         ║  Interactive, MBR disks             ║
║  gdisk   ║  GPT         ║  Interactive, GPT disks             ║
║  parted  ║  MBR + GPT   ║  Scripts, both types                ║
╚══════════╩══════════════╩═════════════════════════════════════╝
```

---

## Filesystem Cheat Sheet

```
╔═══════════════════════════════════════════════════════════════╗
║  FILESYSTEM QUICK REFERENCE                                   ║
╠══════════╦══════════════╦══════════════╦════════════════════╣
║  FS Type ║  mkfs cmd    ║  Check cmd   ║  Resize cmd        ║
╠══════════╬══════════════╬══════════════╬════════════════════╣
║  XFS     ║  mkfs.xfs    ║  xfs_repair  ║  xfs_growfs (grow) ║
║  ext4    ║  mkfs.ext4   ║  e2fsck      ║  resize2fs (both)  ║
║  swap    ║  mkswap      ║  N/A         ║  N/A               ║
║  vfat    ║  mkfs.vfat   ║  fsck.vfat   ║  N/A               ║
╚══════════╩══════════════╩══════════════╩════════════════════╝
```

---

## fstab Pass Values — Quick Table

```
Pass 0 → Skip fsck (swap, tmpfs, NFS)
Pass 1 → Root filesystem only /
Pass 2 → All other local filesystems
```

---

## Mount Options — Most Common on Exam

```
defaults  = rw,suid,dev,exec,auto,nouser,async (6 options in one)
ro        = Read only
rw        = Read/write
noexec    = Cannot run executables (security)
nosuid    = SUID/SGID bits ignored (security)
nodev     = Device files ignored (security)
noatime   = Don't update access time (performance)
_netdev   = Wait for network (NFS, CIFS)
nofail    = Don't fail boot if missing (resilience)
```

---

## UUID vs LABEL vs Device — Decision Rule

```
UUID   → ALWAYS use in fstab (survives renames, reboots)
LABEL  → OK if set, must be unique
/dev/  → NEVER in fstab for persistent mounts
```

---

## Top 10 Exam Commands for Partitioning (In Priority Order)

```
1.  lsblk -f                  ← START HERE every time
2.  blkid                     ← Get UUIDs
3.  fdisk /dev/sdX            ← Create partitions (MBR)
4.  gdisk /dev/sdX            ← Create partitions (GPT)
5.  partprobe /dev/sdX        ← Update kernel
6.  mkfs.xfs /dev/sdX1        ← Format as XFS
7.  mkswap /dev/sdX2          ← Format swap
8.  mkdir -p /mountpoint      ← Create mount point
9.  /etc/fstab (edit)         ← Permanent mount
10. mount -a && swapon -a     ← Test without reboot
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 7 — COMMAND COMPARISON TABLES
# ═══════════════════════════════════════════════════════════

## Partitioning Tools Comparison

| Feature | fdisk | gdisk | parted |
|---|---|---|---|
| Partition table | MBR | GPT | Both |
| Interface | Interactive | Interactive | Both |
| Scriptable | Limited | Limited | YES (`-s`) |
| Max partitions | 4 primary | 128 | 128 (GPT) |
| Max disk size | 2TiB | 9.4ZiB | 9.4ZiB |
| RHEL 9 default | No | No | Preferred |
| Exam use | Common | Common | Less common |
| RHCSA importance | HIGH | HIGH | MEDIUM |

## Filesystem Tools Comparison

| Command | Filesystem | Creates | Checks | Grows | Shrinks |
|---|---|---|---|---|---|
| `mkfs.xfs` | XFS | YES | No | No | No |
| `mkfs.ext4` | ext4 | YES | No | No | No |
| `mkswap` | swap | YES | No | No | No |
| `xfs_repair` | XFS | No | YES | No | No |
| `e2fsck` | ext4 | No | YES | No | No |
| `xfs_growfs` | XFS | No | No | YES | NO |
| `resize2fs` | ext4 | No | No | YES | YES |
| `xfs_admin` | XFS | No | No | No | No |
| `tune2fs` | ext4 | No | No | No | No |

## Storage Information Commands

| Command | What It Shows | RHCSA Use |
|---|---|---|
| `lsblk` | Block device tree | PRIMARY — use first |
| `lsblk -f` | + filesystem info | PRIMARY |
| `blkid` | UUID, type, label | For fstab UUID |
| `fdisk -l` | Partition table | Verification |
| `parted -l` | Partition table (GPT) | Verification |
| `df -hT` | Mounted space usage | Verification |
| `df -i` | Inode usage | Troubleshooting |
| `du -sh` | Directory size | Space analysis |
| `mount` | Current mounts | Verification |
| `findmnt` | Mount tree | Verification |
| `cat /proc/mounts` | Kernel mount table | Deep debug |
| `swapon --show` | Swap usage | Swap verify |
| `free -h` | RAM + Swap total | Quick check |

## Partition Table Types

| Aspect | MBR | GPT |
|---|---|---|
| Full name | Master Boot Record | GUID Partition Table |
| Year introduced | 1983 | 2000s |
| Partition limit | 4 primary + logicals | 128 |
| Max disk | 2 TiB | 9.4 ZiB |
| Redundancy | None | Backup at disk end |
| CRC check | No | Yes |
| Boot | BIOS/Legacy | UEFI |
| Tool | fdisk | gdisk / parted |
| RHEL 9 default | No | YES |

---

# ═══════════════════════════════════════════════════════════
# SECTION 8 — HANDS-ON EXAM ENVIRONMENT SETUP
# ═══════════════════════════════════════════════════════════

## Recommended VM Specifications

```
┌─────────────────────────────────────────────────────────┐
│  RHCSA Practice VM — Recommended Specs                  │
├─────────────────────────────────────────────────────────┤
│  Hypervisor: VirtualBox 7+ / VMware Workstation / KVM   │
│  OS: RHEL 9 / Rocky Linux 9 / AlmaLinux 9              │
│                                                         │
│  CPU: 2 vCPUs (min 1)                                  │
│  RAM: 2 GB (min 1 GB)                                  │
│  Primary Disk: 20 GB (sda) — system disk               │
│  Practice Disks:                                        │
│    /dev/sdb — 10 GB (partitioning practice)            │
│    /dev/sdc — 10 GB (LVM practice)                     │
│    /dev/sdd — 10 GB (Stratis practice)                 │
│  Network: NAT + Host-Only (or Bridged)                 │
├─────────────────────────────────────────────────────────┤
│  Download links:                                        │
│  Rocky 9: https://rockylinux.org/download              │
│  Alma 9:  https://almalinux.org/get-almalinux/         │
│  RHEL 9:  developers.redhat.com (free developer acct)  │
└─────────────────────────────────────────────────────────┘
```

## Network Setup

```bash
# VM should have:
# Adapter 1: NAT (internet access for dnf)
# Adapter 2: Host-Only (192.168.56.0/24 for NFS/SSH practice)

# Verify network
ip addr show
ping 8.8.8.8
```

## Storage Setup Script (Run After VM Install)

```bash
#!/bin/bash
# rhcsa_lab_setup.sh — Run as root
# Adds practice disks if using VirtualBox CLI

# Check if additional disks are present
DISKS=$(lsblk -d -o NAME,SIZE | grep -v "sda\|NAME")
echo "Available practice disks:"
echo "$DISKS"

# Wipe practice disks (NEVER run on production!)
for disk in sdb sdc sdd; do
    if lsblk /dev/$disk &>/dev/null; then
        echo "Wiping /dev/$disk..."
        wipefs -a /dev/$disk
        dd if=/dev/zero of=/dev/$disk bs=512 count=100
        echo "/dev/$disk ready"
    fi
done

echo "Lab environment ready."
lsblk
```

## Recommended Snapshots

```
Snapshot 1: "Clean Install — Pre-Lab"      ← After OS install, before any changes
Snapshot 2: "After Partitioning Lab"       ← After Chapter 1 labs
Snapshot 3: "After LVM Lab"               ← After Chapter 2 labs
Snapshot 4: "After Stratis Lab"           ← After Chapter 3 labs
Snapshot 5: "Full Exam Ready"             ← Final state for exam simulation
```

**Always restore to Snapshot 1 before each chapter's labs.**

---

# ═══════════════════════════════════════════════════════════
# SECTION 9 — EXAM CHECKLIST
# ═══════════════════════════════════════════════════════════

## Chapter 1: Partitioning — Pre-Exam Checklist

Before your RHCSA exam, you must be able to do ALL of the following from memory, without notes, in under 5 minutes per task:

### Knowledge Checks ✓
- [ ] Explain the difference between MBR and GPT
- [ ] State the maximum partitions for each (4/128)
- [ ] State the max disk size for each (2TiB/9.4ZiB)
- [ ] List all 6 fields of /etc/fstab and what each means
- [ ] Explain pass values (0, 1, 2) and when to use each
- [ ] Name the default filesystem for RHEL 9 (XFS)
- [ ] State that XFS cannot shrink
- [ ] State that ext4 CAN shrink (offline)
- [ ] Recite the PCFM workflow (Partition, Create FS, Fstab, Mount)

### Practical Skills ✓
- [ ] Run `lsblk -f` and interpret every field
- [ ] Create a partition with `fdisk` (n, p, number, size, w)
- [ ] Create a partition with `gdisk`
- [ ] Create a partition with `parted -s`
- [ ] Run `partprobe` to update kernel
- [ ] Format a partition as XFS: `mkfs.xfs`
- [ ] Format a partition as ext4: `mkfs.ext4`
- [ ] Prepare a swap partition: `mkswap`
- [ ] Create a mount point: `mkdir -p`
- [ ] Get UUID with `blkid` command
- [ ] Add correct /etc/fstab entry with UUID
- [ ] Test fstab entry with `mount -a`
- [ ] Activate swap: `swapon /dev/sdX` and `swapon -a`
- [ ] Verify mount: `df -hT`, `mount`, `findmnt`
- [ ] Verify swap: `swapon --show`, `free -h`
- [ ] Troubleshoot "device busy": `fuser -mv`
- [ ] Fix wrong UUID in fstab
- [ ] Use `xfs_growfs` to grow XFS online
- [ ] Explain why `xfs_repair` requires filesystem to be unmounted

---

# ═══════════════════════════════════════════════════════════
# SECTION 10 — CHAPTER 1 MASTER CHALLENGE (45 minutes)
# ═══════════════════════════════════════════════════════════

## Master Challenge: Full Disk Configuration

**Time limit:** 45 minutes
**Environment:** RHEL 9 / Rocky Linux 9 with `/dev/sdb` (10GB, empty)

---

### Exam Statement

You are a Linux system administrator. A new 10GB disk `/dev/sdb` has been added to the server. Your manager requires the following configuration:

1. **Partition 1 (3GB):** Mount permanently at `/production` as XFS filesystem. Must not allow execution of SUID programs.

2. **Partition 2 (2GB):** Mount permanently at `/development` as ext4 filesystem with label `DEV`. Must be read-write and not allow execution of programs.

3. **Partition 3 (1GB):** Configure as additional swap space with label `SWAP2`. Must be active immediately and on every reboot.

4. **Partition 4 (2GB):** Mount permanently at `/archive` as XFS filesystem. Mount must NOT fail at boot if the partition is temporarily unavailable.

5. **Verification:** After configuration, create a file `/production/status.txt` containing the text "PRODUCTION READY", and verify all mounts survive a simulated remount (`mount -a`).

---

### Solution (Full)

```bash
#!/bin/bash
# RHCSA Chapter 1 Master Challenge Solution

# === STEP 1: VERIFY DISK ===
lsblk /dev/sdb
fdisk -l /dev/sdb

# === STEP 2: PARTITION THE DISK ===
fdisk /dev/sdb << 'FDISK'
g
n
1

+3G
n
2

+2G
n
3

+1G
n
4

+2G
p
w
FDISK

# Inform kernel
partprobe /dev/sdb
lsblk /dev/sdb

# === STEP 3: FORMAT FILESYSTEMS ===
mkfs.xfs /dev/sdb1                      # production (XFS)
mkfs.ext4 -L "DEV" /dev/sdb2           # development (ext4, labeled)
mkswap -L "SWAP2" /dev/sdb3            # swap
mkfs.xfs /dev/sdb4                      # archive (XFS)

# === STEP 4: GET UUIDs ===
UUID1=$(blkid -s UUID -o value /dev/sdb1)
UUID2=$(blkid -s UUID -o value /dev/sdb2)
UUID3=$(blkid -s UUID -o value /dev/sdb3)
UUID4=$(blkid -s UUID -o value /dev/sdb4)

echo "UUIDs:"
echo "  sdb1 (production): $UUID1"
echo "  sdb2 (development): $UUID2"
echo "  sdb3 (swap): $UUID3"
echo "  sdb4 (archive): $UUID4"

# === STEP 5: CREATE MOUNT POINTS ===
mkdir -p /production /development /archive

# === STEP 6: CONFIGURE /etc/fstab ===
# Backup first!
cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)

# Add entries
cat >> /etc/fstab << EOF
# RHCSA Chapter 1 Master Challenge
UUID=${UUID1}  /production   xfs   defaults,nosuid        0  2
UUID=${UUID2}  /development  ext4  defaults,nosuid,noexec 0  2
UUID=${UUID3}  swap          swap  defaults               0  0
UUID=${UUID4}  /archive      xfs   defaults,nofail        0  2
EOF

# === STEP 7: ACTIVATE EVERYTHING ===
mount -a
swapon -a
systemctl daemon-reload

# === STEP 8: CREATE TEST FILE ===
echo "PRODUCTION READY" > /production/status.txt
cat /production/status.txt

# === STEP 9: VERIFY ===
echo ""
echo "=== VERIFICATION ==="
echo ""
echo "--- Block device layout ---"
lsblk -f /dev/sdb

echo ""
echo "--- Mounted filesystems ---"
df -hT | grep -E "production|development|archive"

echo ""
echo "--- Active swap ---"
swapon --show

echo ""
echo "--- fstab entries ---"
grep -E "production|development|archive|SWAP2" /etc/fstab

echo ""
echo "--- Mount options ---"
findmnt /production
findmnt /development
findmnt /archive

echo ""
echo "--- Test file content ---"
cat /production/status.txt

echo ""
echo "--- Mount remount test ---"
umount /production /development /archive
swapoff /dev/sdb3
mount -a
swapon -a
df -hT | grep -E "production|development|archive"
swapon --show

echo ""
echo "=== CHALLENGE COMPLETE ==="
```

---

### Verification Commands (Grader View)

```bash
# These commands should all succeed:
df -T /production | grep xfs
df -T /development | grep ext4
mount | grep production | grep nosuid
mount | grep development | grep noexec
swapon --show | grep sdb3
mount | grep archive | grep nofail
cat /production/status.txt              # PRODUCTION READY
blkid /dev/sdb2 | grep DEV             # Check label
grep production /etc/fstab             # fstab entry exists
```

---

### Common Mistakes in This Challenge

1. **Forgetting `partprobe`** → partitions not visible
2. **Not creating directories** → mount fails silently
3. **Using `/dev/sdb1` instead of UUID in fstab** → wrong (use UUID)
4. **Wrong pass value** → `1` for root only, `2` for others
5. **Not running `swapon -a`** after fstab entry
6. **Forgetting `-L` flag** for ext4 label
7. **Not backing up fstab** before editing
8. **Using `XFS` instead of `xfs`** in fstab (case-sensitive!)
9. **Missing `nofail` for archive** → boot could hang if disk removed
10. **Confusing `nosuid` and `noexec`**: nosuid = no SUID bits, noexec = no execution

---

# ═══════════════════════════════════════════════════════════
# SELF-ASSESSMENT QUIZ — CHAPTER 1
# ═══════════════════════════════════════════════════════════

Answer these from memory. Answers at the end.

**Q1:** What is the maximum number of primary partitions on an MBR disk?

**Q2:** Which command formats a partition as XFS?

**Q3:** What does `partprobe` do?

**Q4:** In `/etc/fstab`, what does pass value `2` mean?

**Q5:** You created `/dev/sdb1` as swap. What two commands do you need to activate it permanently?

**Q6:** What option do you add to fstab to prevent boot failure if a partition is missing?

**Q7:** Can you shrink an XFS filesystem? What about ext4?

**Q8:** Which command shows you filesystem UUIDs?

**Q9:** You type `mount /data` and get "mount point does not exist". What happened and how do you fix it?

**Q10:** What is the command to test all fstab entries without rebooting?

**Q11:** You need to grow an XFS filesystem after extending its partition. What command do you use, and does the filesystem need to be unmounted?

**Q12:** What is wrong with this fstab entry?
`/dev/sdb1  /data  XFS  defaults  0  1`

---

## Quiz Answers

**A1:** 4 primary partitions maximum (logical partitions exist inside 1 extended partition)

**A2:** `mkfs.xfs /dev/sdXN`

**A3:** Tells the kernel to re-read the partition table after changes (without rebooting)

**A4:** Check this filesystem with fsck after root is checked (pass=1); use for non-root local filesystems

**A5:** `mkswap /dev/sdb1` (first), then add to `/etc/fstab`, then `swapon -a`

**A6:** `nofail` option in the options field

**A7:** XFS **CANNOT** shrink. ext4 CAN shrink but must be unmounted and run `e2fsck` first

**A8:** `blkid` (also `lsblk -f`)

**A9:** The directory `/data` doesn't exist. Fix: `mkdir -p /data`

**A10:** `mount -a`

**A11:** `xfs_growfs /mountpoint` — does NOT need to be unmounted (online operation)

**A12:** THREE errors: (1) uses `/dev/sdb1` instead of UUID, (2) `XFS` should be lowercase `xfs`, (3) pass value `1` should be `2` (only root gets pass=1)

---

# ═══════════════════════════════════════════════════════════
# END OF CHAPTER 1 — PARTITIONING
# ═══════════════════════════════════════════════════════════

**Chapter Status:** ✅ COMPLETE
**Next Chapter:** Chapter 2 — LVM (Logical Volume Manager)

---

*Reply "NEXT" or "Chapter 2" to begin the LVM chapter.*
*Reply "DRILL" to get 10 more exam scenarios for this chapter.*
*Reply "QUIZ" for a timed 15-question self-test.*
