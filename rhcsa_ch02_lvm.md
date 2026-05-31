# RHCSA RHEL 9 — COMPLETE EXAM PREPARATION COURSE
## Chapter 2: LVM — Logical Volume Manager
### Senior Red Hat Linux Engineer · RHCSA/RHCE Instructor · 20+ Years Experience

---

> **Chapter 2 Priority:** LVM is the MOST heavily tested storage topic on the RHCSA. Expect 2–4 LVM tasks on the real exam. Every task must be done from memory. Master this chapter before exam day.

---

# ═══════════════════════════════════════════════════════════
# SECTION 1 — THEORY: FROM ZERO TO RHCSA
# ═══════════════════════════════════════════════════════════

## 1.1 Why LVM Exists — The Problem It Solves

In Chapter 1 you learned standard partitioning. Standard partitions have a fatal limitation:
**they cannot be resized online and have rigid, fixed boundaries.**

Imagine you create a 20GB `/var` partition. Six months later it's 95% full. With standard
partitions, your only options are:
- Backup → delete → recreate larger → restore (downtime)
- Add a new disk and move data manually (complex)

**LVM solves this entirely.** With LVM:
- Add a new disk → extend the volume group → extend the logical volume → grow the filesystem
- All this happens **online, with zero downtime**
- You can even shrink volumes (ext4), take snapshots, and do thin provisioning

This is why every RHEL production server uses LVM by default.

---

## 1.2 LVM Architecture — The Three Layers

```
╔══════════════════════════════════════════════════════════════════════╗
║  LVM ARCHITECTURE — THREE LAYERS                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  LAYER 3: LOGICAL VOLUMES (LV) — What you use                       ║
║  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 ║
║  │  lv_root    │  │  lv_home    │  │  lv_data    │  ← /dev/vg0/... ║
║  │  /dev/vg0/  │  │  /dev/vg0/  │  │  /dev/vg0/  │                 ║
║  │  lv_root    │  │  lv_home    │  │  lv_data    │                 ║
║  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 ║
║         └────────────────┴─────────────────┘                        ║
║                          │                                           ║
╠══════════════════════════╪═══════════════════════════════════════════╣
║  LAYER 2: VOLUME GROUP (VG) — The pool                              ║
║  ┌───────────────────────┴──────────────────────────────────────┐   ║
║  │  vg0  (40GB total pool of Physical Extents)                  │   ║
║  │  ████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │   ║
║  │  [used by LVs above]        [free PEs available for growth]  │   ║
║  └───────────────────┬──────────────────────────────────────────┘   ║
║                      │                                               ║
╠══════════════════════╪═══════════════════════════════════════════════╣
║  LAYER 1: PHYSICAL VOLUMES (PV) — The raw storage                   ║
║  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        ║
║  │  /dev/sdb      │  │  /dev/sdc1     │  │  /dev/sdd      │        ║
║  │  (whole disk)  │  │  (partition)   │  │  (whole disk)  │        ║
║  └────────────────┘  └────────────────┘  └────────────────┘        ║
║  ← Physical disks or partitions marked as LVM PVs ──────────────── ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Layer 1: Physical Volumes (PV)

A **Physical Volume** is the lowest layer — it is a raw disk or partition that has been
initialized for use by LVM. The `pvcreate` command writes LVM metadata to the first few
sectors of the device.

```
/dev/sdb  →  pvcreate /dev/sdb  →  PV: /dev/sdb
/dev/sdc1 →  pvcreate /dev/sdc1 →  PV: /dev/sdc1
/dev/sdd  →  pvcreate /dev/sdd  →  PV: /dev/sdd
```

**PV Internal Structure:**
```
/dev/sdb (Physical Volume):
┌─────────────────────────────────────────────────────────────┐
│ LVM Label (sector 1)                                        │
│ ┌───────────────────────────────────────────────────────┐   │
│ │  PE 0  │  PE 1  │  PE 2  │  PE 3  │  ...  │  PE N   │   │
│ │ 4MB    │ 4MB    │ 4MB    │ 4MB    │       │ 4MB     │   │
│ └───────────────────────────────────────────────────────┘   │
│ Each block = Physical Extent (PE) = default 4MB             │
└─────────────────────────────────────────────────────────────┘
```

### Layer 2: Volume Groups (VG)

A **Volume Group** combines one or more PVs into a single storage pool. It presents a
unified pool of **Physical Extents (PEs)** to the logical layer above.

```
vgcreate vgdata /dev/sdb /dev/sdc1

Result:
vgdata = /dev/sdb (10GB = 2560 PEs) + /dev/sdc1 (5GB = 1280 PEs)
       = 15GB = 3840 Physical Extents of 4MB each
```

### Layer 3: Logical Volumes (LV)

A **Logical Volume** is carved out of VG free space. It appears to the system as a block
device at `/dev/VGNAME/LVNAME` and `/dev/mapper/VGNAME-LVNAME`.

```
lvcreate -L 5G -n lvdata vgdata

Creates: /dev/vgdata/lvdata  (5GB LV from vgdata pool)
Also at: /dev/mapper/vgdata-lvdata
```

---

## 1.3 Physical Extents (PE) — The Fundamental Unit

Everything in LVM is allocated in units called **Physical Extents (PEs)**.

```
Default PE size: 4 MiB

A 10GB disk becomes:
10 * 1024 = 10240 MiB / 4 MiB = 2560 PEs

A 5GB Logical Volume needs:
5 * 1024 = 5120 MiB / 4 MiB = 1280 PEs

PE allocation for LV:
┌─────────────────────────────────────────────────────────────┐
│  VG: vgdata (3840 PEs total)                                │
│  ┌───────────────────────────┐  ┌──────────────────────┐   │
│  │  lv_home (1280 PEs = 5G) │  │  FREE (2560 PEs = 10G)│   │
│  └───────────────────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

PE size can be changed with `vgcreate -s` but default 4MiB is almost always correct.

---

## 1.4 LVM Data Path — How a Read/Write Actually Works

```
Application writes to /data/file.txt
           ↓
VFS layer → LV: /dev/vgdata/lvdata
           ↓
LVM maps Logical Extent (LE) → Physical Extent (PE)
  LE 0 → PE 500 on /dev/sdb
  LE 1 → PE 501 on /dev/sdb
  LE 2 → PE 100 on /dev/sdc1   ← LVM can stripe across PVs
           ↓
Device Mapper translates → actual disk I/O
           ↓
Physical disk writes
```

The **Device Mapper** kernel subsystem handles the LE→PE translation transparently.
This is why LVs can span multiple disks seamlessly.

---

## 1.5 LVM Device Naming — Two Paths, One Device

Every LV appears at TWO paths simultaneously (both work in fstab):

```
/dev/VGNAME/LVNAME           ← Symlink (user-friendly)
/dev/mapper/VGNAME-LVNAME    ← Device mapper path (real device)

Example:
/dev/vgdata/lvhome           → symlink to →
/dev/mapper/vgdata-lvhome    ← actual dm device

Both paths point to the same block device!

In /etc/fstab, prefer:
/dev/mapper/vgdata-lvhome    or    /dev/vgdata/lvhome
(Both are stable and don't change like /dev/sdX does)
```

---

## 1.6 LVM Snapshots

An **LVM snapshot** is a point-in-time copy of a logical volume. It uses
**Copy-on-Write (CoW)**: only changed blocks are copied to the snapshot.

```
Before snapshot:
┌───────────────────────────────────────┐
│  lv_data: Block A, Block B, Block C   │
└───────────────────────────────────────┘

After snapshot of lv_data:
┌───────────────────────────────────────┐
│  lv_data (origin): A, B, C            │
└───────────────────────────────────────┘
┌───────────────────────────────────────┐
│  lv_data_snap: [metadata only at first]│
└───────────────────────────────────────┘

After modifying Block B in lv_data:
┌───────────────────────────────────────┐
│  lv_data (origin): A, B_new, C        │
│  lv_data_snap: B_old (CoW copy kept)  │
└───────────────────────────────────────┘

The snapshot still shows original B.
This is how live backups work without stopping services.
```

---

## 1.7 LVM Thin Provisioning

**Thin Provisioning** allows you to allocate more storage than physically exists,
based on the assumption that not all space will be used simultaneously.

```
Physical Storage: 10GB
Thin Pool: 10GB
Thin Volumes:
  lv_vm1: "20GB" (only uses 3GB actually written)
  lv_vm2: "20GB" (only uses 2GB actually written)
  lv_vm3: "15GB" (only uses 1GB actually written)
Total allocated: 55GB from 10GB pool (5.5:1 overcommit)
Actual usage: 6GB (fine — under the 10GB limit)
```

> **RHCSA Note:** Thin provisioning is a bonus topic. Understand it conceptually.
> The exam focuses on standard thick provisioning.

---

## 1.8 Advantages and Disadvantages of LVM

### Advantages
| Advantage | Description |
|---|---|
| **Online resize** | Extend LVs without unmounting (XFS) |
| **Disk spanning** | One LV across multiple physical disks |
| **Snapshots** | Point-in-time backups without downtime |
| **Flexible** | No wasted space — pool shares all disks |
| **Striping/Mirroring** | RAID-like features built in |
| **Thin provisioning** | Overcommit storage |

### Disadvantages
| Disadvantage | Description |
|---|---|
| **Complexity** | More commands to learn |
| **Recovery harder** | Corrupted LVM metadata → complex fix |
| **Overhead** | Minor performance overhead vs raw partition |
| **Not for /boot** | GRUB needs direct access (usually) |

---

## 1.9 LVM vs Standard Partitioning — When to Use Each

| Scenario | Standard Partition | LVM |
|---|---|---|
| `/boot` partition | **YES** | No (GRUB compatibility) |
| `/boot/efi` | **YES** | No |
| Everything else | Possible | **Preferred** |
| Need to resize online | No | **YES** |
| Multiple disks as one | No | **YES** |
| Snapshots needed | No | **YES** |
| Simple single-disk | OK | Preferred anyway |
| RHEL 9 default install | Uses LVM for `/` and `/home` | |

---

## 1.10 Exam Traps and Common LVM Mistakes

1. **`pvcreate` on a partition, not a disk** — both are valid, but exam often uses whole disks
2. **Forgetting `pvcreate`** before `vgcreate` — you'll get "not a physical volume" error
3. **Not having free PE** in VG before extending LV — add another PV first
4. **XFS cannot shrink** — trying to reduce an XFS LV will corrupt data
5. **Forgetting to grow the filesystem** after extending the LV — the LV grows but the FS stays same size
6. **Wrong path in fstab** — use `/dev/vgname/lvname` or `/dev/mapper/vgname-lvname`
7. **Removing PV that has active data** — always `pvmove` first
8. **Snapshot overflow** — if snapshot fills up, it becomes invalid
9. **lvcreate -l vs -L** — lowercase `-l` is in PEs/percentage, uppercase `-L` is in size (G, M, T)
10. **Not updating fstab** after renaming a VG/LV — old path won't mount

---

# ═══════════════════════════════════════════════════════════
# SECTION 2 — COMMANDS REFERENCE (EXAM-LEVEL DETAIL)
# ═══════════════════════════════════════════════════════════

## 2.1 Physical Volume Commands

### pvcreate — Initialize a PV

```bash
# Syntax
pvcreate [OPTIONS] DEVICE [DEVICE...]

# Options
-f, --force       Force (even if filesystem detected)
--dataalignment N Align data to N bytes
-u, --uuid UUID   Restore a specific UUID
-v                Verbose

# Examples
pvcreate /dev/sdb                # Whole disk as PV (preferred)
pvcreate /dev/sdb1               # Partition as PV (also valid)
pvcreate /dev/sdb /dev/sdc       # Multiple PVs in one command
pvcreate -f /dev/sdb             # Force overwrite existing data

# What it does internally:
# Writes LVM label to sector 1 of the device
# Creates PV metadata in a metadata area (usually 1MB)
```

### pvdisplay — Detailed PV Information

```bash
# Syntax
pvdisplay [OPTIONS] [PV...]

# Options
-s, --short       Short format
-C, --colon       Colon-separated output (scriptable)
-m, --maps        Show PE allocation map
-v                Verbose

# Examples
pvdisplay                       # All PVs (verbose)
pvdisplay /dev/sdb              # Specific PV
pvdisplay -m /dev/sdb           # With PE allocation map

# Sample output:
#   --- Physical volume ---
#   PV Name               /dev/sdb
#   VG Name               vgdata
#   PV Size               10.00 GiB / not usable 4.00 MiB
#   Allocatable           yes
#   PE Size               4.00 MiB
#   Total PE              2559
#   Free PE               1279
#   Allocated PE          1280
#   PV UUID               abc123-...
```

### pvs — Concise PV Summary

```bash
# Syntax
pvs [OPTIONS] [PV...]

# Options
-o FIELDS         Select output fields
-v                More verbose
--noheadings      No header line (for scripts)
--separator SEP   Field separator

# Examples
pvs                             # Quick summary of all PVs
pvs /dev/sdb                    # Specific PV
pvs -o pv_name,pv_size,pv_free  # Custom columns
pvs -o +pv_used                 # Add pv_used to default columns

# Sample output:
#   PV         VG     Fmt  Attr PSize   PFree
#   /dev/sdb   vgdata lvm2 a--  10.00g  5.00g
#   /dev/sdc   vgdata lvm2 a--  10.00g 10.00g
```

### pvscan — Scan for PVs

```bash
pvscan                          # Scan all devices for PVs
pvscan --cache                  # Update LVM device cache (after adding disk)
```

### pvremove — Remove LVM Label from PV

```bash
pvremove /dev/sdb               # Remove LVM label (device leaves LVM)
pvremove -f /dev/sdb            # Force
# WARNING: Only use on PVs that have been removed from all VGs first!
```

### pvmove — Move PV Data to Another PV

```bash
# Move all data off /dev/sdb (before removing it from VG)
pvmove /dev/sdb                        # Move all extents to other PVs in VG
pvmove /dev/sdb /dev/sdc               # Move specifically to /dev/sdc
pvmove --abort                         # Abort an in-progress pvmove
```

### pvresize — Resize a PV (After Disk Growth)

```bash
pvresize /dev/sdb               # Resize PV to match underlying device
pvresize --setphysicalvolumesize 15G /dev/sdb  # Set explicit size
```

---

## 2.2 Volume Group Commands

### vgcreate — Create a Volume Group

```bash
# Syntax
vgcreate [OPTIONS] VGNAME PV [PV...]

# Options
-s, --physicalextentsize SIZE   PE size (default 4MiB; use M, G, etc.)
-p, --maxphysicalvolumes N      Max PVs allowed in VG
-l, --maxlogicalvolumes N       Max LVs allowed in VG

# Examples
vgcreate vgdata /dev/sdb                        # One PV
vgcreate vgdata /dev/sdb /dev/sdc               # Two PVs
vgcreate -s 8M vgdata /dev/sdb                  # 8MiB PE size
vgcreate vgsales /dev/sdb1 /dev/sdc1 /dev/sdd1  # Three partitions

# Naming convention: prefix with 'vg' (vgdata, vgweb, vgsales)
```

### vgdisplay — Detailed VG Information

```bash
vgdisplay                       # All VGs
vgdisplay vgdata                # Specific VG

# Key fields to know:
#   VG Size          — Total size of VG
#   PE Size          — Physical Extent size (default 4MiB)
#   Total PE         — Total number of PEs
#   Alloc PE / Size  — Used PEs / size
#   Free  PE / Size  — Available PEs / size ← most important!
```

### vgs — Concise VG Summary

```bash
vgs                             # All VGs, quick summary
vgs vgdata                      # Specific VG
vgs -o vg_name,vg_size,vg_free  # Custom columns
vgs -o +vg_tags                 # Add tags to output

# Sample output:
#   VG     #PV #LV #SN Attr   VSize   VFree
#   vgdata   2   3   0 wz--n- 19.99g  9.99g
```

### vgscan — Scan for Volume Groups

```bash
vgscan                          # Scan for VGs
vgscan --mknodes                # Recreate /dev entries
```

### vgextend — Add PV to Existing VG

```bash
# Syntax
vgextend VGNAME PV [PV...]

# IMPORTANT: Run pvcreate on new disk FIRST!
pvcreate /dev/sdc               # Step 1: initialize new disk
vgextend vgdata /dev/sdc        # Step 2: add to VG

# Verify
vgs vgdata                      # VFree should increase
pvs                             # /dev/sdc now shows VG name
```

### vgreduce — Remove PV from VG

```bash
# IMPORTANT: Move data off the PV first!
pvmove /dev/sdb                 # Step 1: move data away
vgreduce vgdata /dev/sdb        # Step 2: remove PV from VG
pvremove /dev/sdb               # Step 3: clean LVM label (optional)
```

### vgrename — Rename a Volume Group

```bash
vgrename vgdata vgnewname
# WARNING: Update /etc/fstab after renaming!
```

### vgchange — Activate/Deactivate VG

```bash
vgchange -a y vgdata            # Activate all LVs in vgdata
vgchange -a n vgdata            # Deactivate all LVs in vgdata
vgchange -a y                   # Activate ALL VGs
# Needed after importing VGs from another system
```

### vgremove — Delete an Entire VG

```bash
# Remove all LVs first, then remove VG
lvremove /dev/vgdata/lv1
lvremove /dev/vgdata/lv2
vgremove vgdata
pvremove /dev/sdb
```

### vgmerge / vgsplit — Merge or Split VGs

```bash
# Merge vg2 into vg1 (vg2 must have no active LVs)
vgmerge vg1 vg2

# Split: move specified LVs/PVs from vgdata to vgnew
vgsplit vgdata vgnew /dev/sdc
```

---

## 2.3 Logical Volume Commands

### lvcreate — Create a Logical Volume

```bash
# Syntax
lvcreate [OPTIONS] VGNAME

# Key Options:
-L SIZE        Size using units (G, M, T, %)
               Examples: -L 5G  -L 500M  -L 1T
-l EXTENTS     Size in PEs or percentage
               Examples: -l 1280  -l 100%FREE  -l 50%VG
-n NAME        LV name (REQUIRED)
-i STRIPES     Number of stripes (RAID-like performance)
-I STRIPE_SIZE Stripe size
-m MIRRORS     Number of mirrors (for redundancy)
-s, --snapshot Create snapshot LV
-p r/rw        Permissions (r=read-only, rw=read-write)

# Standard LV creation examples:
lvcreate -L 5G -n lvdata vgdata              # 5GB named lvdata
lvcreate -L 500M -n lvlogs vgweb             # 500MB LV
lvcreate -l 1280 -n lvhome vgdata            # 1280 PEs (= 5GB with 4MiB PE)
lvcreate -l 100%FREE -n lvdata vgdata        # All remaining free space
lvcreate -l 50%VG -n lvdata vgdata           # 50% of total VG size
lvcreate -l 50%FREE -n lvdata vgdata         # 50% of FREE space

# Striped LV (performance):
lvcreate -i 2 -L 10G -n lvstripe vgdata      # Stripe across 2 PVs

# Snapshot:
lvcreate -s -L 1G -n lvdata_snap /dev/vgdata/lvdata

# Thin pool and thin LV:
lvcreate -L 10G --thinpool pool1 vgdata
lvcreate -V 20G --thin -n thin1 vgdata/pool1
```

### SIZE vs EXTENTS — Critical Exam Knowledge

```bash
# -L (uppercase) = human-readable size
lvcreate -L 5G -n lv1 vg0      # 5 Gigabytes
lvcreate -L 500M -n lv2 vg0    # 500 Megabytes
lvcreate -L 1T -n lv3 vg0      # 1 Terabyte

# -l (lowercase) = logical extents or percentage
lvcreate -l 1280 -n lv1 vg0    # 1280 extents × 4MiB = 5GiB
lvcreate -l 100%FREE -n lv1 vg0  # ALL free space
lvcreate -l 50%VG -n lv1 vg0   # Half of total VG
lvcreate -l 50%FREE -n lv1 vg0 # Half of free space
```

### lvdisplay — Detailed LV Information

```bash
lvdisplay                           # All LVs
lvdisplay /dev/vgdata/lvdata        # Specific LV
lvdisplay -m /dev/vgdata/lvdata     # With segment/PE map

# Key fields:
#   LV Path       /dev/vgdata/lvdata
#   LV Name       lvdata
#   VG Name       vgdata
#   LV Size       5.00 GiB
#   Current LE    1280       ← logical extents
#   Segments      1
#   Allocation    inherit
#   Read ahead sectors  auto
```

### lvs — Concise LV Summary

```bash
lvs                             # All LVs, quick summary
lvs vgdata                      # LVs in specific VG
lvs -o lv_name,lv_size,vg_name  # Custom columns
lvs -o +lv_path                 # Add LV path to output
lvs --all                       # Include hidden/internal LVs

# Sample output:
#   LV     VG     Attr       LSize   Pool Origin Data%
#   lvdata vgdata -wi-ao----  5.00g
#   lvhome vgdata -wi-ao----  3.00g
```

### lvscan — Scan for Logical Volumes

```bash
lvscan                          # Scan all LVs
lvscan --cache                  # Update LVM device list

# Sample output:
#   ACTIVE            '/dev/vgdata/lvdata' [5.00 GiB] inherit
#   ACTIVE            '/dev/vgdata/lvhome' [3.00 GiB] inherit
```

### lvextend — Extend a Logical Volume

```bash
# Syntax
lvextend [OPTIONS] LV [PV...]

# Options
-L SIZE           Extend to absolute size (use +SIZE to add)
-l EXTENTS        Extend by extents or percentage
-r, --resizefs    Also resize the filesystem automatically!
+SIZE             Add to current size (use + prefix)

# Examples — CRITICAL for exam!

# Extend LV only (must resize FS separately):
lvextend -L +2G /dev/vgdata/lvdata         # Add 2GB
lvextend -L 10G /dev/vgdata/lvdata         # Extend TO 10GB total
lvextend -l +512 /dev/vgdata/lvdata        # Add 512 PEs
lvextend -l +100%FREE /dev/vgdata/lvdata   # Add all remaining free space

# BEST PRACTICE — extend LV AND resize filesystem in one command:
lvextend -r -L +2G /dev/vgdata/lvdata      # -r = --resizefs
lvextend -r -l +100%FREE /dev/vgdata/lvdata

# Without -r, you must resize the FS manually:
# For XFS (online):
lvextend -L +2G /dev/vgdata/lvdata
xfs_growfs /data                           # Uses mount point!

# For ext4 (online):
lvextend -L +2G /dev/vgdata/lvdata
resize2fs /dev/vgdata/lvdata               # Uses device path!
```

### lvreduce — Shrink a Logical Volume (DANGEROUS)

```bash
# ONLY works with ext4 (XFS CANNOT shrink)
# ALWAYS unmount first!

# Safe procedure for ext4 shrink:
umount /dev/vgdata/lvdata               # Step 1: unmount
e2fsck -f /dev/vgdata/lvdata            # Step 2: filesystem check
resize2fs /dev/vgdata/lvdata 3G         # Step 3: shrink FS first
lvreduce -L 3G /dev/vgdata/lvdata       # Step 4: shrink LV
mount /dev/vgdata/lvdata /data          # Step 5: remount

# Combined (still need FS check):
e2fsck -f /dev/vgdata/lvdata
lvreduce -r -L 3G /dev/vgdata/lvdata    # -r shrinks FS and LV together

# EXAM WARNING: Always shrink FS BEFORE LV. Never the other way!
# If you shrink the LV first, the FS is truncated → DATA LOSS!
```

### lvresize — Resize in Either Direction

```bash
lvresize -L 8G /dev/vgdata/lvdata       # Resize to 8G (grow or shrink)
lvresize -r -L 8G /dev/vgdata/lvdata    # With filesystem resize
lvresize -r -L +2G /dev/vgdata/lvdata   # Add 2G to current size
```

### lvremove — Delete a Logical Volume

```bash
# Must unmount first!
umount /dev/vgdata/lvdata
lvremove /dev/vgdata/lvdata             # Interactive (asks Y/N)
lvremove -f /dev/vgdata/lvdata          # Force (no prompt)
```

### lvrename — Rename a Logical Volume

```bash
lvrename vgdata lvold lvnew
lvrename /dev/vgdata/lvold /dev/vgdata/lvnew
# Update /etc/fstab after renaming!
```

### lvchange — Change LV Attributes

```bash
lvchange -a y /dev/vgdata/lvdata        # Activate LV
lvchange -a n /dev/vgdata/lvdata        # Deactivate LV
lvchange -p r /dev/vgdata/lvdata        # Set read-only
lvchange -p rw /dev/vgdata/lvdata       # Set read-write
```

---

## 2.4 The Complete LVM Status Commands — Side by Side

```bash
# Three levels of detail for each layer:

# PHYSICAL VOLUMES:
pvs                     # Quick: one line per PV
pvdisplay               # Detailed: all metadata
pvscan                  # Find PVs on system

# VOLUME GROUPS:
vgs                     # Quick: one line per VG
vgdisplay               # Detailed: all metadata
vgscan                  # Find VGs on system

# LOGICAL VOLUMES:
lvs                     # Quick: one line per LV
lvdisplay               # Detailed: all metadata
lvscan                  # Find LVs on system

# ALL LVM AT ONCE:
lvmdiskscan             # Show all disks and LVM status
```

---

## 2.5 Filesystem Operations on LVs

After creating/extending an LV, you still need to manage the filesystem:

```bash
# FORMAT a new LV:
mkfs.xfs /dev/vgdata/lvdata
mkfs.ext4 /dev/vgdata/lvdata

# MOUNT an LV:
mkdir -p /data
mount /dev/vgdata/lvdata /data

# PERMANENT MOUNT in /etc/fstab:
# Two valid options (both stable, unlike /dev/sdX):
echo "/dev/vgdata/lvdata  /data  xfs  defaults  0  2" >> /etc/fstab
# OR using device mapper path:
echo "/dev/mapper/vgdata-lvdata  /data  xfs  defaults  0  2" >> /etc/fstab

# GROW filesystem after lvextend:
xfs_growfs /data                       # XFS: use mount point, online
resize2fs /dev/vgdata/lvdata           # ext4: use device path, online ok

# CHECK filesystem:
xfs_repair /dev/vgdata/lvdata          # XFS (unmounted)
e2fsck -f /dev/vgdata/lvdata           # ext4 (unmounted)
```

---

## 2.6 LVM Snapshot Commands

```bash
# Create a snapshot of lvdata (1GB CoW space)
lvcreate -s -L 1G -n lvdata_snap /dev/vgdata/lvdata

# List snapshots
lvs -a | grep snap
lvdisplay /dev/vgdata/lvdata_snap

# Mount snapshot (read-only by default)
mkdir /mnt/snap
mount -o ro /dev/vgdata/lvdata_snap /mnt/snap

# Restore from snapshot (DESTRUCTIVE — overwrites origin)
umount /data
umount /mnt/snap
lvconvert --merge /dev/vgdata/lvdata_snap
# Reboot or reactivate LV to complete merge

# Remove snapshot (when no longer needed)
umount /mnt/snap
lvremove /dev/vgdata/lvdata_snap
```

---

## 2.7 Full LVM Workflow — The PVCVLMF Pattern

```
P — pvcreate  (Physical Volume)
V — vgcreate  (Volume Group)
C — lvcreate  (Logical Volume — "C" = Create LV)
L — (format with mkfs)
M — mkdir + mount
F — /etc/fstab (Persistent)
```

Memory hook: **"Pretty Vixens Can Lure More Fans"**

```bash
# The 6-step LVM workflow every exam task follows:
pvcreate /dev/sdb                          # 1. Create PV
vgcreate vgdata /dev/sdb                   # 2. Create VG
lvcreate -L 5G -n lvdata vgdata            # 3. Create LV
mkfs.xfs /dev/vgdata/lvdata               # 4. Format (L = mkfs)
mkdir -p /data && mount /dev/vgdata/lvdata /data  # 5. Mount
echo "/dev/vgdata/lvdata /data xfs defaults 0 2" >> /etc/fstab  # 6. fstab
```

CHAPTER_START
echo "Section 1 and 2 written"
# ═══════════════════════════════════════════════════════════
# SECTION 3 — PRACTICAL LABS
# ═══════════════════════════════════════════════════════════

---

## LAB 1 (Beginner) — Build Your First LVM Stack

### Objective
Create a complete LVM stack: PV → VG → LV → format → mount → fstab.

### Initial Situation
RHEL 9 VM with `/dev/sdb` (10GB, empty, no partitions).

### Pre-Lab Check
```bash
lsblk /dev/sdb              # Confirm empty
pvs                         # Confirm no existing PVs
vgs                         # Confirm no existing VGs
```

### Tasks

**Task 1.1 — Create Physical Volume**
```bash
pvcreate /dev/sdb
pvs                         # Verify: /dev/sdb shows up with no VG
pvdisplay /dev/sdb          # Full details
```

**Task 1.2 — Create Volume Group**
```bash
vgcreate vglab /dev/sdb
vgs                         # Verify: vglab with ~10GB free
vgdisplay vglab             # Full details
```

**Task 1.3 — Create Logical Volume (3GB)**
```bash
lvcreate -L 3G -n lvlab /dev/vglab  # Wait — should be lvcreate -L 3G -n lvlab vglab
lvcreate -L 3G -n lvlab vglab
lvs                         # Verify: lvlab 3.00g
lvdisplay /dev/vglab/lvlab  # Full details
lsblk                       # Should now show /dev/vglab/lvlab
```

**Task 1.4 — Format as XFS**
```bash
mkfs.xfs /dev/vglab/lvlab
lsblk -f /dev/vglab/lvlab   # Verify: FSTYPE=xfs
```

**Task 1.5 — Mount and configure fstab**
```bash
mkdir -p /labdata
mount /dev/vglab/lvlab /labdata
echo "/dev/vglab/lvlab  /labdata  xfs  defaults  0  2" >> /etc/fstab
mount | grep labdata        # Verify mount
df -hT /labdata             # Verify size and type
```

**Task 1.6 — Create test data**
```bash
echo "LVM Lab 1 Success" > /labdata/test.txt
cat /labdata/test.txt
```

### Verification
```bash
pvs                         # PV: /dev/sdb in vglab
vgs                         # VG: vglab ~7G free
lvs                         # LV: lvlab 3.00g active
df -hT /labdata             # XFS, ~3GB
mount | grep labdata        # Mounted
cat /labdata/test.txt       # Data visible
```

### Expected Results
```
  PV         VG    Fmt  Attr PSize   PFree
  /dev/sdb   vglab lvm2 a--  <10.00g <7.00g

  VG    #PV #LV #SN Attr   VSize   VFree
  vglab   1   1   0 wz--n- <10.00g <7.00g

  LV    VG    Attr       LSize
  lvlab vglab -wi-ao----  3.00g

Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/mapper/vglab-lvlab xfs  3.0G   68M  2.9G   3% /labdata
```

---

## LAB 2 (Intermediate) — Extend a VG and LV

### Objective
Add a second disk to the VG, then extend the LV to use the new space.

### Initial Situation
VG `vglab` exists on `/dev/sdb`. A new disk `/dev/sdc` (10GB) has been added.

### Tasks

**Task 2.1 — Prepare new disk as PV**
```bash
lsblk /dev/sdc              # Confirm new disk
pvcreate /dev/sdc           # Initialize as PV
pvs                         # See both /dev/sdb and /dev/sdc
```

**Task 2.2 — Extend VG with new PV**
```bash
vgextend vglab /dev/sdc
vgs vglab                   # VFree should jump to ~17GB
vgdisplay vglab             # See "Total PE" and "Free PE" increase
```

**Task 2.3 — Extend the LV**
```bash
# Check current size
df -h /labdata              # Currently 3GB

# Extend LV by 5GB (note the + prefix):
lvextend -L +5G /dev/vglab/lvlab

# Verify LV grew but filesystem still shows old size:
lvs vglab                   # LV is now 8GB
df -h /labdata              # Still shows 3GB! FS not yet resized
```

**Task 2.4 — Grow the filesystem**
```bash
# XFS: use mount point, must be MOUNTED:
xfs_growfs /labdata

# Verify filesystem grew:
df -h /labdata              # Now shows ~8GB
xfs_info /labdata           # Confirms new block count
```

**Task 2.5 — Verify test data survived**
```bash
cat /labdata/test.txt       # Should still say "LVM Lab 1 Success"
ls -la /labdata/
```

### Alternative — Extend + Resize in One Command
```bash
# The -r flag does both lvextend AND filesystem resize:
lvextend -r -L +2G /dev/vglab/lvlab
# No separate xfs_growfs needed!
```

### Verification
```bash
pvs                         # Two PVs in vglab
vgs vglab                   # Larger total size
lvs vglab                   # LV shows 8G
df -hT /labdata             # Filesystem shows ~8G, type xfs
```

---

## LAB 3 (Intermediate) — Create ext4 LV and Resize

### Objective
Create an ext4 LV, extend it, and then (carefully) shrink it.

### Tasks

**Task 3.1 — Create ext4 LV**
```bash
lvcreate -L 2G -n lvext4 vglab
mkfs.ext4 /dev/vglab/lvext4
mkdir -p /ext4data
mount /dev/vglab/lvext4 /ext4data
echo "/dev/vglab/lvext4  /ext4data  ext4  defaults  0  2" >> /etc/fstab
df -hT /ext4data
```

**Task 3.2 — Extend ext4 LV**
```bash
lvextend -r -L +1G /dev/vglab/lvext4   # -r auto-resizes ext4 too
df -hT /ext4data                        # Now shows ~3GB
```

**Task 3.3 — Shrink ext4 LV (careful procedure)**
```bash
# CRITICAL ORDER: FS first, then LV

# Step 1: Unmount (REQUIRED for shrink)
umount /ext4data

# Step 2: Check filesystem
e2fsck -f /dev/vglab/lvext4

# Step 3: Shrink filesystem FIRST to 2GB
resize2fs /dev/vglab/lvext4 2G

# Step 4: Shrink LV to same size
lvreduce -L 2G /dev/vglab/lvext4

# Step 5: Remount
mount /ext4data
df -hT /ext4data            # Should show ~2GB
```

---

## LAB 4 (RHCSA Exam Level) — Full LVM Exam Simulation

### Objective
Simulate the most common RHCSA LVM task format.

### Scenario
"A new disk `/dev/sdb` is available. Create a 5GB XFS logical volume named `lvdata` in a volume group named `vgdata`. Mount it permanently under `/data`."

### Solution (Timed — must complete in under 4 minutes)

```bash
#!/bin/bash
# Timer starts now

# 1. Verify disk
lsblk /dev/sdb

# 2. PV
pvcreate /dev/sdb

# 3. VG
vgcreate vgdata /dev/sdb

# 4. LV
lvcreate -L 5G -n lvdata vgdata

# 5. Format
mkfs.xfs /dev/vgdata/lvdata

# 6. Mount point
mkdir -p /data

# 7. fstab (IMPORTANT: use device mapper path or /dev/vgname/lvname)
echo "/dev/vgdata/lvdata  /data  xfs  defaults  0  2" >> /etc/fstab

# 8. Mount
mount -a
systemctl daemon-reload

# 9. Verify
df -hT /data
lvs vgdata
mount | grep /data
```

---

## LAB 5 (RHCSA Exam Level) — Extend Existing LV on Exam

### Scenario
"The logical volume `lvdata` in volume group `vgdata` needs to be extended to 8GB.
A new disk `/dev/sdc` is available if needed. The filesystem must be accessible
immediately after extension with all data intact."

### Solution

```bash
# Step 1: Check current state
lvs vgdata                  # lvdata current size?
vgs vgdata                  # How much free space in VG?
df -h /data                 # Current filesystem usage

# Step 2: Check if VG has enough free space
# If vgdata has 3G+ free: just extend LV
# If vgdata is full: add /dev/sdc first

# --- If VG needs more space: ---
pvcreate /dev/sdc
vgextend vgdata /dev/sdc

# --- Extend LV to 8GB (absolute size) ---
lvextend -r -L 8G /dev/vgdata/lvdata   # -r resizes XFS automatically

# Alternative: extend by adding 3GB:
lvextend -r -L +3G /dev/vgdata/lvdata

# Verify
lvs vgdata                  # LSize = 8.00g
df -hT /data                # Shows ~8GB XFS
```

---

## LAB 6 (Troubleshooting) — LVM Volume Not Mounting

### Scenario
A system reboots and `/data` is not mounted. Diagnose and fix.

### Diagnostic Procedure

```bash
# Step 1: Check if LVM is visible
lvscan                      # Are LVs active?
lvs                         # Do they show up?

# Step 2: Check if VG is active
vgs                         # VG status?
vgchange -a y vgdata        # Force activate VG

# Step 3: Check fstab
cat /etc/fstab | grep data
mount -a 2>&1               # Try to mount, read errors

# Step 4: Check the LV device exists
ls -la /dev/vgdata/lvdata
ls /dev/mapper/ | grep vgdata

# Step 5: Try manual mount
mount /dev/vgdata/lvdata /data 2>&1

# Step 6: Check filesystem
xfs_repair -n /dev/vgdata/lvdata   # Dry run check

# Common fixes:
vgchange -a y               # Activate all VGs
mount -a                    # Mount all fstab entries
```

---

## LAB 7 (Troubleshooting) — Filesystem Not Growing After lvextend

### Scenario
`lvextend` ran successfully but `df` still shows the old size.

```bash
# Diagnosis:
lvs                         # LV is 8G
df -h /data                 # Filesystem still 5G  ← THE PROBLEM

# Root cause: lvextend was run without -r

# Fix for XFS:
xfs_growfs /data            # Must use mount point!

# Fix for ext4:
resize2fs /dev/vgdata/lvdata  # Can use device path

# Prevention: always use -r with lvextend:
lvextend -r -L +2G /dev/vgdata/lvdata
```


# ═══════════════════════════════════════════════════════════
# SECTION 4 — 20+ RHCSA EXAM SCENARIOS
# ═══════════════════════════════════════════════════════════

---

### SCENARIO 1
**Task:** Create a 5GB logical volume named `lvdata` in a new volume group `vgdata`
on disk `/dev/sdb`. Format as XFS and mount permanently at `/data`.

**Thinking Process:** PV → VG → LV → mkfs → mkdir → fstab → mount -a

**Full Solution:**
```bash
pvcreate /dev/sdb
vgcreate vgdata /dev/sdb
lvcreate -L 5G -n lvdata vgdata
mkfs.xfs /dev/vgdata/lvdata
mkdir -p /data
echo "/dev/vgdata/lvdata  /data  xfs  defaults  0  2" >> /etc/fstab
mount -a
```

**Verification:**
```bash
lvs vgdata
df -hT /data
mount | grep /data
```

---

### SCENARIO 2
**Task:** Extend the logical volume `lvdata` in `vgdata` from 5GB to 8GB.
The filesystem must reflect the new size immediately.

**Full Solution:**
```bash
# Check free space first
vgs vgdata
# If VG has free space:
lvextend -r -L 8G /dev/vgdata/lvdata      # Absolute: extend TO 8G
# OR
lvextend -r -L +3G /dev/vgdata/lvdata     # Relative: ADD 3G

# Verify
df -hT /data                               # Should show ~8G
lvs vgdata
```

---

### SCENARIO 3
**Task:** A volume group `vgdata` is running out of space. A new disk `/dev/sdc`
is available. Add it to `vgdata`.

**Full Solution:**
```bash
pvcreate /dev/sdc
vgextend vgdata /dev/sdc
vgs vgdata                  # Verify VFree increased
pvs                         # /dev/sdc now shows vgdata
```

---

### SCENARIO 4
**Task:** Create a 2GB swap logical volume in volume group `vgswap` on `/dev/sdb`.
Activate it permanently.

**Full Solution:**
```bash
pvcreate /dev/sdb
vgcreate vgswap /dev/sdb
lvcreate -L 2G -n lvswap vgswap
mkswap /dev/vgswap/lvswap
swapon /dev/vgswap/lvswap
echo "/dev/vgswap/lvswap  swap  swap  defaults  0  0" >> /etc/fstab
```

**Verification:**
```bash
swapon --show
free -h
lvs vgswap
```

---

### SCENARIO 5
**Task:** Use exactly 100% of the free space in `vgdata` to create a logical
volume named `lvfull`. Format as XFS. Mount at `/full`.

**Full Solution:**
```bash
lvcreate -l 100%FREE -n lvfull vgdata
mkfs.xfs /dev/vgdata/lvfull
mkdir -p /full
echo "/dev/vgdata/lvfull  /full  xfs  defaults  0  2" >> /etc/fstab
mount -a
vgs vgdata                  # VFree should be 0
df -hT /full
```

---

### SCENARIO 6
**Task:** Create a logical volume using exactly 1280 physical extents named
`lvpe` in volume group `vgdata`. Mount at `/pedata` as XFS.

**Thinking:** 1280 PEs × 4MB = 5120MB = 5GB

**Full Solution:**
```bash
lvcreate -l 1280 -n lvpe vgdata
mkfs.xfs /dev/vgdata/lvpe
mkdir -p /pedata
echo "/dev/vgdata/lvpe  /pedata  xfs  defaults  0  2" >> /etc/fstab
mount -a
lvs vgdata                  # Should show lvpe 5.00g
```

---

### SCENARIO 7
**Task:** Create a snapshot of `/dev/vgdata/lvdata` named `lvdata_backup` with 1GB of CoW space.

**Full Solution:**
```bash
lvcreate -s -L 1G -n lvdata_backup /dev/vgdata/lvdata
lvs -a vgdata               # Snapshot visible
lvdisplay /dev/vgdata/lvdata_backup  # Check Data% usage

# Mount snapshot read-only:
mkdir -p /mnt/backup
mount -o ro /dev/vgdata/lvdata_backup /mnt/backup
ls /mnt/backup              # Same content as /data at snapshot time
```

---

### SCENARIO 8
**Task:** List all physical volumes and their free space. Identify which PV
has the most free space.

**Full Solution:**
```bash
pvs -o pv_name,pv_size,pv_free --sort -pv_free
# OR
pvdisplay | grep -E "PV Name|Free PE"
```

---

### SCENARIO 9
**Task:** Remove logical volume `lvold` from `vgdata`. The mount point is `/old`.

**Full Solution:**
```bash
# Step 1: Unmount
umount /old

# Step 2: Remove fstab entry
sed -i '/\/old/d' /etc/fstab

# Step 3: Remove LV
lvremove /dev/vgdata/lvold      # Answer 'y' or use -f

# Verify
lvs vgdata                      # lvold gone
vgs vgdata                      # VFree increased
```

---

### SCENARIO 10
**Task:** The root filesystem is on an LVM LV. Check how much free space
the volume group has for future expansion.

**Full Solution:**
```bash
# Find which VG holds root
df /
lvs | grep -v "^$"
vgs                             # Shows all VGs with VSize and VFree

# More detailed:
vgdisplay | grep -E "VG Name|Free PE|VG Size"

# Or scriptable:
vgs --noheadings -o vg_name,vg_free
```

---

### SCENARIO 11
**Task:** Rename the logical volume `lvdata` to `lvproduction` in `vgdata`.
Ensure the system still mounts it correctly.

**Full Solution:**
```bash
# Step 1: Unmount
umount /data

# Step 2: Rename
lvrename vgdata lvdata lvproduction

# Step 3: Update /etc/fstab
sed -i 's|/dev/vgdata/lvdata|/dev/vgdata/lvproduction|g' /etc/fstab

# Step 4: Remount
mount -a
mount | grep production     # Verify new path mounted
lvs vgdata                  # Shows lvproduction
```

---

### SCENARIO 12
**Task:** Create a volume group `vgmulti` spanning two disks `/dev/sdb`
and `/dev/sdc`. Create a 12GB LV named `lvbig` using space from both disks.

**Full Solution:**
```bash
pvcreate /dev/sdb /dev/sdc
vgcreate vgmulti /dev/sdb /dev/sdc
lvcreate -L 12G -n lvbig vgmulti        # LVM spans both PVs automatically
mkfs.xfs /dev/vgmulti/lvbig
mkdir -p /bigdata
echo "/dev/vgmulti/lvbig  /bigdata  xfs  defaults  0  2" >> /etc/fstab
mount -a
df -hT /bigdata             # Shows 12GB XFS
pvs                         # Both PVs contribute
```

---

### SCENARIO 13
**Task:** Display the physical extent usage map showing which PEs on `/dev/sdb`
are used by which logical volumes.

**Full Solution:**
```bash
pvdisplay -m /dev/sdb
# Shows which PEs map to which LVs

# Also useful:
pvs -o pv_name,pv_pe_count,pv_pe_alloc_count
```

---

### SCENARIO 14
**Task:** Check if a logical volume `lvdata` exists. If not, create it
(500MB, XFS, mounted at `/data`). Write this as a safe script.

**Full Solution:**
```bash
#!/bin/bash
VG="vgdata"
LV="lvdata"
MOUNT="/data"

if ! lvs /dev/${VG}/${LV} &>/dev/null; then
    echo "LV not found, creating..."
    pvcreate /dev/sdb 2>/dev/null
    vgcreate $VG /dev/sdb 2>/dev/null
    lvcreate -L 500M -n $LV $VG
    mkfs.xfs /dev/${VG}/${LV}
    mkdir -p $MOUNT
    echo "/dev/${VG}/${LV}  ${MOUNT}  xfs  defaults  0  2" >> /etc/fstab
    mount -a
    echo "Done"
else
    echo "LV exists: $(lvs /dev/${VG}/${LV})"
fi
```

---

### SCENARIO 15
**Task:** Extend the LV `/dev/vgdata/lvdata` to use 60% of its volume group.

**Full Solution:**
```bash
lvextend -r -l 60%VG /dev/vgdata/lvdata
df -hT /data
lvs vgdata
vgs vgdata                  # Compare LV size to VG size
```

---

### SCENARIO 16
**Task:** Deactivate a volume group `vgbackup`, then reactivate it.

**Full Solution:**
```bash
# Deactivate (unmount LVs first!)
umount /backup
vgchange -a n vgbackup
lvscan | grep vgbackup      # Shows INACTIVE

# Reactivate
vgchange -a y vgbackup
mount -a
lvscan | grep vgbackup      # Shows ACTIVE
```

---

### SCENARIO 17
**Task:** Move all data from physical volume `/dev/sdb` to `/dev/sdc` within
the same volume group, then remove `/dev/sdb` from the VG.

**Full Solution:**
```bash
# Verify both PVs are in same VG
pvs

# Move data (can take time with real data)
pvmove /dev/sdb /dev/sdc

# Verify /dev/sdb has no PEs allocated
pvs                         # /dev/sdb PFree = PSize

# Remove from VG
vgreduce vgdata /dev/sdb

# Optional: remove LVM label
pvremove /dev/sdb

# Verify
pvs                         # Only /dev/sdc in vgdata
vgs vgdata                  # Size reduced
```

---

### SCENARIO 18
**Task:** Create a thin-provisioned pool and two thin volumes from it.

**Full Solution:**
```bash
# Create thin pool (10GB)
lvcreate -L 10G --thinpool thinpool1 vgdata

# Create thin volumes (overcommitted — 20GB each from 10GB pool)
lvcreate -V 20G --thin -n lv_thin1 vgdata/thinpool1
lvcreate -V 20G --thin -n lv_thin2 vgdata/thinpool1

# Format and mount
mkfs.xfs /dev/vgdata/lv_thin1
mkfs.xfs /dev/vgdata/lv_thin2
mkdir -p /thin1 /thin2
mount /dev/vgdata/lv_thin1 /thin1
mount /dev/vgdata/lv_thin2 /thin2

# Check actual usage
lvs -a vgdata               # Shows Data% for pool usage
```

---

### SCENARIO 19
**Task:** A logical volume `lvdata` has a PE size of 4MiB. Calculate: if the VG
has 512 free PEs, how many GB can you add to the LV?

**Answer and Verification:**
```bash
# 512 PEs × 4MiB = 2048 MiB = 2 GiB

# Verify via vgs:
vgs -o vg_name,vg_free_count,vg_extent_size vgdata

# Extend by that exact amount:
lvextend -l +512 /dev/vgdata/lvdata
# = lvextend -L +2G /dev/vgdata/lvdata
```

---

### SCENARIO 20
**Task:** Create a complete LVM setup for a web server:
- VG `vgweb` on `/dev/sdb`
- LV `lv_www` (5G, XFS) mounted at `/var/www`
- LV `lv_logs` (2G, XFS) mounted at `/var/log/httpd`
- LV `lv_db` (3G, ext4) mounted at `/var/lib/mysql`

**Full Solution:**
```bash
pvcreate /dev/sdb
vgcreate vgweb /dev/sdb

lvcreate -L 5G -n lv_www vgweb
lvcreate -L 2G -n lv_logs vgweb
lvcreate -L 3G -n lv_db vgweb

mkfs.xfs /dev/vgweb/lv_www
mkfs.xfs /dev/vgweb/lv_logs
mkfs.ext4 /dev/vgweb/lv_db

mkdir -p /var/www /var/log/httpd /var/lib/mysql

cat >> /etc/fstab << 'EOF'
/dev/vgweb/lv_www    /var/www          xfs  defaults  0  2
/dev/vgweb/lv_logs   /var/log/httpd    xfs  defaults  0  2
/dev/vgweb/lv_db     /var/lib/mysql    ext4 defaults  0  2
EOF

mount -a
systemctl daemon-reload

# Verify all three:
df -hT /var/www /var/log/httpd /var/lib/mysql
lvs vgweb
```

---

### SCENARIO 21 — Bonus (Advanced)
**Task:** The VG `vgdata` was created on a disk that is being decommissioned.
Export the VG so it can be imported on a new server.

**Full Solution:**
```bash
# On OLD server:
umount /data                            # Unmount all LVs
vgchange -a n vgdata                    # Deactivate VG
vgexport vgdata                         # Export VG
# Now physically move disk to new server

# On NEW server:
pvscan                                  # Find imported PV
vgimport vgdata                         # Import VG
vgchange -a y vgdata                    # Activate LVs
mount /dev/vgdata/lvdata /data          # Mount
```


# ═══════════════════════════════════════════════════════════
# SECTION 5 — TROUBLESHOOTING GUIDE
# ═══════════════════════════════════════════════════════════

---

## FAILURE 1: "Device /dev/sdb not found" After pvcreate

**Symptoms:** `pvcreate /dev/sdb` gives "Device not found" or `pvs` shows nothing.

**Diagnosis:**
```bash
lsblk                       # Is disk detected by kernel?
ls /dev/sdb*                # Does device file exist?
dmesg | tail -30            # Kernel disk detection messages
```

**Causes & Fixes:**
```bash
# Cause 1: Kernel not updated after adding disk
echo 1 > /sys/bus/scsi/drivers/sd/rescan
udevadm settle

# Cause 2: Wrong disk name (it may be /dev/vdb or /dev/nvme0n1)
lsblk -d                    # See ALL disks

# Cause 3: Disk not physically connected
# → Check VM settings or physical cabling
```

---

## FAILURE 2: "Can't initialize physical volume... already in volume group"

**Symptoms:** `pvcreate /dev/sdb` fails with "already a physical volume"
or "already in volume group".

**Diagnosis:**
```bash
pvs /dev/sdb                # Which VG owns it?
vgs                         # Is that VG active?
```

**Fix:**
```bash
# If old VG is no longer needed:
vgchange -a n old_vg_name
vgremove old_vg_name
pvremove /dev/sdb
# Now pvcreate will work:
pvcreate /dev/sdb

# If you want to FORCE overwrite (data destructive!):
pvcreate -f /dev/sdb        # Only do this if you're sure
```

---

## FAILURE 3: "Insufficient free space" During lvcreate

**Symptoms:** `lvcreate -L 5G -n lvdata vgdata` fails.

**Diagnosis:**
```bash
vgs vgdata                  # What is VFree?
vgdisplay vgdata | grep Free
```

**Fix:**
```bash
# Option A: Use smaller LV
lvcreate -L 3G -n lvdata vgdata   # Use less space

# Option B: Add another PV to the VG
pvcreate /dev/sdc
vgextend vgdata /dev/sdc
lvcreate -L 5G -n lvdata vgdata   # Now should work

# Option C: Use exact free space
lvcreate -l 100%FREE -n lvdata vgdata
```

---

## FAILURE 4: LV Extended But df Shows Old Size

**Symptoms:** `lvextend -L +2G` succeeded but `df -h` shows old size.

**Explanation:** LV and filesystem are separate. `lvextend` only grows
the block device. The filesystem still sees the old boundary.

**Fix:**
```bash
# For XFS (online, use mount point):
xfs_growfs /data
# OR
xfs_growfs /dev/vgdata/lvdata

# For ext4 (online, use device path):
resize2fs /dev/vgdata/lvdata

# Prevention: Use -r with lvextend:
lvextend -r -L +2G /dev/vgdata/lvdata   # Does both!
```

---

## FAILURE 5: "Filesystem has unsupported features" or XFS Won't Mount

**Symptoms:** XFS LV won't mount after extending or repair.

**Diagnosis:**
```bash
dmesg | tail -20             # XFS error messages
xfs_repair -n /dev/vgdata/lvdata  # Check state
mount -t xfs /dev/vgdata/lvdata /data 2>&1
```

**Fix:**
```bash
# Try mounting with recovery option:
mount -t xfs -o recovery /dev/vgdata/lvdata /data

# If dirty/unclean shutdown:
xfs_repair /dev/vgdata/lvdata   # Must be UNMOUNTED

# Last resort — replay journal:
mount -t xfs -o logbsize=32k /dev/vgdata/lvdata /data
```

---

## FAILURE 6: Cannot Unmount LV — "Target is Busy"

**Symptoms:** `umount /data` gives "target is busy".

**Fix:**
```bash
fuser -mv /data             # Find which processes
lsof /data                  # List open files
fuser -km /data             # Kill processes using it
umount /data

# Lazy unmount (last resort):
umount -l /data
```

---

## FAILURE 7: LVM Commands Not Found (Package Missing)

**Symptoms:** `pvcreate: command not found`

**Fix:**
```bash
dnf install lvm2            # Install LVM tools
which pvcreate              # Verify: /sbin/pvcreate or /usr/sbin/pvcreate
```

---

## FAILURE 8: LVs Not Showing After Reboot

**Symptoms:** After reboot, `lvs` shows nothing or LVs are INACTIVE.

**Diagnosis:**
```bash
pvscan                      # Scan for PVs
vgscan                      # Scan for VGs
lvscan                      # Are LVs showing as INACTIVE?
```

**Fix:**
```bash
# Activate all VGs and LVs:
vgchange -a y
lvscan                      # Should now show ACTIVE

# If VG UUID conflict (rare):
vgimport vgdata             # Import if VG was exported
vgchange -a y vgdata

# Check LVM services:
systemctl enable lvm2-lvmpolld
systemctl start lvm2-lvmpolld
```

---

## FAILURE 9: Snapshot Overflow — "Snapshot is Invalidated"

**Symptoms:** Snapshot became invalid because CoW space filled up.

**Diagnosis:**
```bash
lvs -a | grep snap          # Data% shows 100%
# Once invalidated, snapshot cannot be used
```

**Fix:**
```bash
# Remove invalidated snapshot:
lvremove /dev/vgdata/lvdata_snap

# Create new snapshot with MORE space:
lvcreate -s -L 5G -n lvdata_snap /dev/vgdata/lvdata

# Monitor snapshot usage:
lvs /dev/vgdata/lvdata_snap  # Watch Data% column
```

---

## FAILURE 10: Wrong Path in /etc/fstab After LV Rename

**Symptoms:** After `lvrename`, system fails to mount at boot.

**Fix:**
```bash
# Check old vs new name
lvs

# Update fstab:
# Old: /dev/vgdata/lvold
# New: /dev/vgdata/lvnew
sed -i 's|/dev/vgdata/lvold|/dev/vgdata/lvnew|g' /etc/fstab

# Also check device mapper:
ls /dev/mapper/ | grep vgdata
# If old: /dev/mapper/vgdata-lvold → update fstab

mount -a                    # Test
```

---

## FAILURE 11: "Physical volume /dev/sdb is already in volume group"

**Symptoms:** Trying to add a PV to a second VG fails.

**Explanation:** A PV can only belong to ONE VG at a time.

**Fix:**
```bash
pvs /dev/sdb               # Which VG currently?
# Either use that VG, or use a different disk, or
# Remove from current VG first (pvmove + vgreduce)
```


# ═══════════════════════════════════════════════════════════
# SECTION 6 — MEMORY AIDS
# ═══════════════════════════════════════════════════════════

## The LVM 3-Layer Stack (Memorize This Visual)

```
┌─────────────────────────────────────────────────────────┐
│  LV (Logical Volume)  → What you format and mount        │
│  VG (Volume Group)    → The pool that LVs are carved from│
│  PV (Physical Volume) → Real disks / partitions          │
└─────────────────────────────────────────────────────────┘
PV → VG → LV → mkfs → mount → fstab
```

---

## The PVCVLMF Mnemonic — Full LVM Build Workflow

```
P  →  pvcreate /dev/sdX           (Physical Volume)
V  →  vgcreate vgNAME /dev/sdX    (Volume Group)
C  →  lvcreate -L Ng -n lvNAME vgNAME   (Create LV)
 L →  mkfs.xfs /dev/vgNAME/lvNAME       (Lay filesystem)
  M→  mkdir /mnt && mount /dev/... /mnt  (Mount)
   F→  echo "..." >> /etc/fstab          (Fstab)
```

Rhyme: **"Pretty Vixens Can Lay More Files"**

---

## lvcreate Size Flags — Don't Mix Them Up

```
UPPERCASE  -L  →  Human SIZE       -L 5G  -L 500M  -L +2G
lowercase  -l  →  EXTENTS / %      -l 1280  -l 100%FREE  -l 50%VG
```

Golden rule: **Capital L for Size (GB, MB). Lower l for logical (PEs, %).**

---

## The Extend Workflow Cheat Sheet

```
Add disk to VG:         pvcreate NEW_DISK → vgextend VG NEW_DISK
Extend LV (best):       lvextend -r -L +Ng /dev/VG/LV
Extend LV (manual XFS): lvextend -L +Ng /dev/VG/LV → xfs_growfs /mountpoint
Extend LV (manual ext4):lvextend -L +Ng /dev/VG/LV → resize2fs /dev/VG/LV
```

---

## The Shrink Workflow (ext4 ONLY — NEVER XFS)

```
Step 1: umount /mountpoint
Step 2: e2fsck -f /dev/VG/LV
Step 3: resize2fs /dev/VG/LV TARGET_SIZE  ← FS first!
Step 4: lvreduce -L TARGET_SIZE /dev/VG/LV  ← LV second!
Step 5: mount /mountpoint
```

**DANGER ORDER:** If you do steps 4 then 3, you LOSE DATA.

---

## Quick Status Command Guide

```bash
pvs          # PVs: one line each (PV name, VG, size, free)
vgs          # VGs: one line each (VG name, #PVs, #LVs, size, free)
lvs          # LVs: one line each (LV name, VG, size, status)
pvdisplay    # PV full details
vgdisplay    # VG full details (most important: Free PE and size)
lvdisplay    # LV full details (path, size, LE count)
lvscan       # LVs with ACTIVE/INACTIVE status
```

---

## LVM Device Paths — Exam Cheat

```
SAME device, TWO paths:
  /dev/vgdata/lvdata           ← symlink (exam-friendly)
  /dev/mapper/vgdata-lvdata    ← real device mapper path

BOTH work in:
  mount /dev/vgdata/lvdata /data
  echo "/dev/vgdata/lvdata /data xfs defaults 0 2" >> /etc/fstab

NEVER use:
  /dev/dm-0                    ← unstable, changes between boots
```

---

## Top 15 LVM Commands for RHCSA (Priority Order)

```
Priority  Command                           Why
 1        lsblk / pvs / vgs / lvs          See current state first
 2        pvcreate /dev/sdX                Step 1: make PV
 3        vgcreate VGNAME /dev/sdX         Step 2: create pool
 4        lvcreate -L Ng -n lvNAME VGNAME  Step 3: carve LV
 5        mkfs.xfs /dev/VGNAME/lvNAME      Step 4: format
 6        mkdir -p /mountpoint             Step 5: create dir
 7        /etc/fstab edit                  Step 6: make permanent
 8        mount -a                         Step 7: test fstab
 9        pvcreate + vgextend              Expand VG with new disk
10        lvextend -r -L +Ng /dev/VG/LV   Grow LV + FS at once
11        xfs_growfs /mountpoint           Grow XFS (if forgot -r)
12        e2fsck -f + resize2fs + lvreduce Shrink ext4 LV
13        lvcreate -s -L 1G -n snap /dev/VG/LV  Snapshot
14        pvmove /dev/sdX                  Move data before removing PV
15        vgchange -a y                    Reactivate all VGs after issues
```

---

## fstab Entries for LVM — Template

```
# XFS LV (preferred format):
/dev/vgdata/lvdata     /data     xfs    defaults    0  2

# Using device mapper (also correct):
/dev/mapper/vgdata-lvdata  /data  xfs  defaults  0  2

# ext4 LV:
/dev/vgdata/lvext4     /data2    ext4   defaults    0  2

# Swap LV:
/dev/vgdata/lvswap     swap      swap   defaults    0  0
```

---

## LVM Attribute Flags — What the Columns Mean

```
From: lvs
LV    VG   Attr        LSize
lvdata vgdata -wi-ao----  5.00g

Attr column = six characters:
  Position 1: Volume type
    - = normal LV
    s = snapshot
    t = thin volume
    T = thin pool
    m = mirror

  Position 2: Permissions
    w = writable
    r = read-only

  Position 3: Allocation policy
    i = inherit (from VG)
    c = contiguous
    n = normal

  Position 4: fixed minor number
    - = no

  Position 5: State
    a = ACTIVE
    - = inactive
    s = suspended

  Position 6: Device open
    o = open (mounted)
    - = not open

So: -wi-ao---- = normal, writable, inherit, active, open
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 7 — COMMAND COMPARISON TABLES
# ═══════════════════════════════════════════════════════════

## LVM vs Standard Partitioning vs Stratis

| Feature | Standard Partition | LVM | Stratis |
|---|---|---|---|
| Online resize | NO | YES | YES |
| Span multiple disks | NO | YES | YES |
| Snapshots | NO | YES | YES |
| Thin provisioning | NO | YES | YES |
| Learning curve | Low | Medium | Medium |
| RHCSA importance | HIGH | **HIGHEST** | MEDIUM |
| Default RHEL 9 setup | No | YES | No |
| Commands | fdisk/parted | pv/vg/lvcreate | stratis |
| Filesystem support | Any | Any | XFS only |
| Configuration tool | fdisk/parted | LVM commands | stratisd daemon |

---

## LVM Info Commands Compared

| Command | Detail Level | Best For | Fields Shown |
|---|---|---|---|
| `pvs` | Low (1 line) | Quick scan | PV, VG, size, free |
| `pvdisplay` | High (block) | Full details | All PV metadata |
| `pvscan` | Medium | Finding PVs | PV paths + status |
| `vgs` | Low (1 line) | Quick scan | VG, PV count, LV count, size, free |
| `vgdisplay` | High (block) | Full details | All VG metadata |
| `vgscan` | Medium | Finding VGs | VG names |
| `lvs` | Low (1 line) | Quick scan | LV, VG, attrs, size |
| `lvdisplay` | High (block) | Full details | All LV metadata |
| `lvscan` | Medium | Finding LVs | LV paths + ACTIVE status |

---

## lvextend vs resize2fs vs xfs_growfs

| Command | Target | Online? | Use For |
|---|---|---|---|
| `lvextend` | Block device (LV) | YES | Growing the LV only |
| `lvextend -r` | LV + filesystem | YES | Growing BOTH at once |
| `xfs_growfs /mountpoint` | XFS filesystem | YES (mounted) | After lvextend without -r |
| `resize2fs /dev/VG/LV` | ext4 filesystem | YES (online grow) | After lvextend without -r |
| `resize2fs /dev/VG/LV SIZE` | ext4 filesystem | NO (unmounted for shrink) | Shrinking ext4 |
| `lvreduce -L SIZE` | Block device (LV) | NO (unmounted for shrink) | Shrinking the LV |

---

## Extend +SIZE vs Absolute SIZE

| Syntax | Meaning | Example |
|---|---|---|
| `lvextend -L 8G` | Extend TO 8GB total | If LV is 5G → becomes 8G |
| `lvextend -L +3G` | Add 3GB to current | If LV is 5G → becomes 8G |
| `lvextend -l 100%FREE` | Use all free PEs | Fills VG |
| `lvextend -l +512` | Add 512 PEs (2GB default) | Precise PE control |

> **EXAM TIP:** The `+` prefix ADDS to current size. No `+` means set ABSOLUTE size.
> The exam may say "extend to 8GB" (no +) or "add 3GB" (use +).

---

# ═══════════════════════════════════════════════════════════
# SECTION 8 — HANDS-ON EXAM ENVIRONMENT
# ═══════════════════════════════════════════════════════════

## VM Setup for LVM Practice

```
Recommended Storage Configuration:
┌────────────────────────────────────────────────────────────┐
│  /dev/sda  →  20GB  →  System disk (OS, /boot, swap, /)   │
│  /dev/sdb  →  10GB  →  LVM Practice Disk 1                │
│  /dev/sdc  →  10GB  →  LVM Practice Disk 2 (VG expansion) │
│  /dev/sdd  →  10GB  →  Stratis/Other (Chapter 3)          │
└────────────────────────────────────────────────────────────┘
```

## LVM Practice Reset Script

```bash
#!/bin/bash
# reset_lvm_lab.sh — WIPE ALL LVM ON sdb AND sdc — NEVER run on production!
# Run as root

set -e

echo "WARNING: This will destroy all LVM on /dev/sdb and /dev/sdc"
read -p "Type YES to continue: " confirm
[ "$confirm" != "YES" ] && exit 1

# Unmount any mounts from these disks
for mp in /data /labdata /ext4data /bigdata /pedata /full /thin1 /thin2; do
    umount $mp 2>/dev/null || true
done

# Deactivate and remove VGs
for vg in vglab vgdata vgweb vgmulti vgswap; do
    vgchange -a n $vg 2>/dev/null || true
    vgremove -f $vg 2>/dev/null || true
done

# Remove LVM labels
pvremove -f /dev/sdb 2>/dev/null || true
pvremove -f /dev/sdc 2>/dev/null || true

# Wipe disks
wipefs -a /dev/sdb
wipefs -a /dev/sdc

echo "Lab environment reset. Ready for new exercises."
lsblk /dev/sdb /dev/sdc
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 9 — EXAM CHECKLIST
# ═══════════════════════════════════════════════════════════

## Chapter 2: LVM — Pre-Exam Checklist

### Conceptual Knowledge ✓
- [ ] Draw the 3-layer LVM architecture from memory (PV → VG → LV)
- [ ] Explain what a Physical Extent (PE) is and its default size (4MiB)
- [ ] Explain the difference between `-L` and `-l` in lvcreate
- [ ] State why XFS cannot shrink but ext4 can
- [ ] Explain what `lvextend -r` does (both LV and filesystem)
- [ ] Describe the correct shrink order (FS before LV — never reverse!)
- [ ] Explain how LVM snapshots work (Copy-on-Write)
- [ ] State two valid paths to access an LV (/dev/VG/LV and /dev/mapper/VG-LV)

### Practical Skills ✓
- [ ] Create a PV: `pvcreate /dev/sdX`
- [ ] Create a VG: `vgcreate VGNAME /dev/sdX`
- [ ] Create an LV by size: `lvcreate -L 5G -n lvNAME VGNAME`
- [ ] Create an LV using %: `lvcreate -l 100%FREE -n lvNAME VGNAME`
- [ ] Format as XFS: `mkfs.xfs /dev/VGNAME/lvNAME`
- [ ] Format as ext4: `mkfs.ext4 /dev/VGNAME/lvNAME`
- [ ] Mount and add to fstab correctly
- [ ] Check LVM state: `pvs`, `vgs`, `lvs`
- [ ] Add disk to existing VG: `pvcreate` then `vgextend`
- [ ] Extend LV and FS together: `lvextend -r -L +Ng`
- [ ] Extend FS manually (XFS): `xfs_growfs /mountpoint`
- [ ] Extend FS manually (ext4): `resize2fs /dev/VG/LV`
- [ ] Shrink ext4 LV safely (5-step procedure from memory)
- [ ] Create a snapshot: `lvcreate -s -L 1G -n snap /dev/VG/LV`
- [ ] Remove a snapshot: `lvremove`
- [ ] Troubleshoot: LV not appearing → `vgchange -a y`
- [ ] Troubleshoot: FS not growing → `xfs_growfs` or `resize2fs`

---

# ═══════════════════════════════════════════════════════════
# SECTION 10 — CHAPTER 2 MASTER CHALLENGE (60 minutes)
# ═══════════════════════════════════════════════════════════

## Master Challenge: Full LVM Configuration

**Time limit:** 60 minutes
**Environment:** RHEL 9 with `/dev/sdb` (10GB) and `/dev/sdc` (10GB), both empty

---

### Exam Statement

You are the Linux administrator for a growing application server.
Your manager has provided two additional disks and requires the following:

**Task 1:** Initialize `/dev/sdb` and `/dev/sdc` as physical volumes, then
create a single volume group named `vgapps` spanning both disks.

**Task 2:** Create a logical volume named `lv_appdata` in `vgapps` using
exactly 8GB. Format as XFS and mount permanently at `/appdata`.
The mount point must not allow SUID programs to run.

**Task 3:** Create a logical volume named `lv_logs` using 3GB. Format as
ext4 with the label `APPLOGS`. Mount permanently at `/applogs`.

**Task 4:** Create an additional 2GB swap logical volume named `lv_swap`
in `vgapps`. Make it permanently active with priority 10.

**Task 5:** After completing tasks 2-4, simulate growth: extend `lv_appdata`
by 3GB (total 11GB). The filesystem must be accessible immediately.

**Task 6:** Create a snapshot of `lv_logs` named `lv_logs_snap` with 500MB
of snapshot space. Mount the snapshot read-only at `/mnt/logs_snap`.

**Task 7:** Write a verification file `/appdata/server_config.txt` containing:
```
SERVER: apps01
VG: vgapps
PVS: /dev/sdb /dev/sdc
```

---

### Full Solution

```bash
#!/bin/bash
# RHCSA Chapter 2 LVM Master Challenge Solution

set -e  # Exit on error

echo "=== RHCSA Chapter 2 Master Challenge ==="
echo ""

# === TASK 1: PVs and VG ===
echo "--- Task 1: Creating PVs and VG ---"
pvcreate /dev/sdb /dev/sdc
vgcreate vgapps /dev/sdb /dev/sdc

pvs                         # Verify both PVs
vgs vgapps                  # Verify VG (~20GB)
echo "Task 1: DONE"
echo ""

# === TASK 2: lv_appdata (8GB XFS at /appdata, nosuid) ===
echo "--- Task 2: lv_appdata ---"
lvcreate -L 8G -n lv_appdata vgapps
mkfs.xfs /dev/vgapps/lv_appdata
mkdir -p /appdata

# Add to fstab with nosuid
echo "/dev/vgapps/lv_appdata  /appdata  xfs  defaults,nosuid  0  2" >> /etc/fstab
mount -a
echo "Task 2: DONE"
echo ""

# === TASK 3: lv_logs (3GB ext4, label APPLOGS, at /applogs) ===
echo "--- Task 3: lv_logs ---"
lvcreate -L 3G -n lv_logs vgapps
mkfs.ext4 -L "APPLOGS" /dev/vgapps/lv_logs
mkdir -p /applogs

echo "/dev/vgapps/lv_logs  /applogs  ext4  defaults  0  2" >> /etc/fstab
mount -a
echo "Task 3: DONE"
echo ""

# === TASK 4: lv_swap (2GB swap, priority 10) ===
echo "--- Task 4: lv_swap ---"
lvcreate -L 2G -n lv_swap vgapps
mkswap /dev/vgapps/lv_swap

# Activate with priority 10
swapon -p 10 /dev/vgapps/lv_swap

# fstab with priority
echo "/dev/vgapps/lv_swap  swap  swap  defaults,pri=10  0  0" >> /etc/fstab
echo "Task 4: DONE"
echo ""

# === TASK 5: Extend lv_appdata from 8G to 11G ===
echo "--- Task 5: Extending lv_appdata to 11GB ---"
lvextend -r -L 11G /dev/vgapps/lv_appdata  # -r resizes XFS automatically
df -hT /appdata                              # Verify
echo "Task 5: DONE"
echo ""

# === TASK 6: Snapshot of lv_logs ===
echo "--- Task 6: Snapshot lv_logs ---"
lvcreate -s -L 500M -n lv_logs_snap /dev/vgapps/lv_logs
mkdir -p /mnt/logs_snap
mount -o ro /dev/vgapps/lv_logs_snap /mnt/logs_snap
echo "Task 6: DONE"
echo ""

# === TASK 7: Verification file ===
echo "--- Task 7: Writing verification file ---"
cat > /appdata/server_config.txt << 'EOF'
SERVER: apps01
VG: vgapps
PVS: /dev/sdb /dev/sdc
EOF

cat /appdata/server_config.txt
echo "Task 7: DONE"
echo ""

# === FINAL VERIFICATION ===
echo "==============================="
echo "=== FULL VERIFICATION REPORT ==="
echo "==============================="
echo ""
echo "--- PVs ---"
pvs

echo ""
echo "--- VG ---"
vgs vgapps

echo ""
echo "--- LVs ---"
lvs vgapps

echo ""
echo "--- Mounts ---"
df -hT | grep -E "appdata|applogs|logs_snap|Filesystem"

echo ""
echo "--- Swap ---"
swapon --show

echo ""
echo "--- fstab entries ---"
grep -E "vgapps|lv_" /etc/fstab

echo ""
echo "--- Mount options (nosuid check) ---"
findmnt /appdata

echo ""
echo "--- Snapshot status ---"
lvs -a vgapps | grep snap

echo ""
echo "--- Verification file ---"
cat /appdata/server_config.txt

echo ""
echo "=== CHALLENGE COMPLETE ==="
```

---

### Grader Verification Commands

```bash
# These all must succeed for full marks:
vgs vgapps | grep -E "20|19"               # ~20GB VG
lvs vgapps | grep lv_appdata | grep 11     # 11GB LV
df -T /appdata | grep xfs                  # XFS mounted
mount | grep appdata | grep nosuid         # nosuid option
df -T /applogs | grep ext4                 # ext4 mounted
tune2fs -l /dev/vgapps/lv_logs | grep APPLOGS  # label
swapon --show | grep lv_swap               # swap active
swapon --show | grep lv_swap | grep 10    # priority 10
df -h /appdata | grep "11G\|10.9\|10G"   # ~11GB filesystem
mount | grep logs_snap | grep ro           # snapshot read-only
cat /appdata/server_config.txt | grep apps01  # file content
grep vgapps /etc/fstab | wc -l            # fstab has 3+ entries
```

---

### Common Mistakes in This Challenge

1. **Forgetting `pvcreate` before `vgcreate`** → "not a PV"
2. **Using `lvcreate -L 8g` (lowercase g)** → works but confusing; use `G`
3. **Not using `-r` with `lvextend`** → LV grows but `df` shows old size
4. **Snapshot of wrong LV** → must snapshot lv_logs not lv_appdata
5. **Priority syntax wrong** → fstab: `pri=10`, swapon: `-p 10`
6. **Mounting snapshot as read-write** → must use `-o ro`
7. **Wrong label syntax** → `mkfs.ext4 -L "APPLOGS"` not `--label`
8. **Forgetting `systemctl daemon-reload`** after fstab edit
9. **Checking `df` before `mount -a`** → filesystem not yet mounted
10. **Using `/dev/dm-0` path in fstab** → unstable, use `/dev/vgapps/lv_appdata`

---

# ═══════════════════════════════════════════════════════════
# SELF-ASSESSMENT QUIZ — CHAPTER 2
# ═══════════════════════════════════════════════════════════

**Q1:** What are the three layers of LVM from bottom to top?

**Q2:** What does `lvcreate -l 100%FREE -n lvdata vgdata` do differently
from `lvcreate -L 10G -n lvdata vgdata`?

**Q3:** After running `lvextend -L +5G /dev/vgdata/lvdata` on an XFS
filesystem, what additional command is required?

**Q4:** What is the correct safe order to shrink an ext4 LV from 10GB to 5GB?
(List all 5 steps)

**Q5:** What does `pvmove /dev/sdb` do and when would you use it?

**Q6:** You run `lvextend -r -L +2G /dev/vgdata/lvdata`. What does the `-r`
flag do?

**Q7:** A VG shows 0 free PE. You need to add more space to an LV. What do you do?

**Q8:** What is the difference between `pvs` and `pvdisplay`?

**Q9:** Why is `/dev/mapper/vgdata-lvdata` more reliable in `/etc/fstab`
than `/dev/sdb1`?

**Q10:** A snapshot `lv_logs_snap` shows `Data%` at 95%. What action
should you take immediately?

---

## Quiz Answers

**A1:** Physical Volume (PV) → Volume Group (VG) → Logical Volume (LV)

**A2:** `-l 100%FREE` uses every remaining free Physical Extent in the VG
(no space wasted). `-L 10G` allocates exactly 10GB (fails if less than 10GB is free).

**A3:** `xfs_growfs /mountpoint` — the LV block device grew but XFS filesystem
boundaries haven't been updated yet. For XFS, use the mount point (not device path).

**A4:**
1. `umount /mountpoint`
2. `e2fsck -f /dev/vgdata/lvdata`
3. `resize2fs /dev/vgdata/lvdata 5G` (shrink FS FIRST)
4. `lvreduce -L 5G /dev/vgdata/lvdata` (then shrink LV)
5. `mount /mountpoint`

**A5:** `pvmove` migrates all Physical Extents (data) from `/dev/sdb` to
other PVs in the same VG. Used before removing a PV from a VG (`vgreduce`).

**A6:** `-r` (or `--resizefs`) automatically resizes the filesystem after
extending the LV. Without it, only the LV block device grows; the filesystem
remains at the old size and must be grown manually.

**A7:** Add a new physical disk:
1. `pvcreate /dev/sdX`
2. `vgextend VGNAME /dev/sdX`
3. Now extend the LV: `lvextend -r -L +Ng /dev/VGNAME/lvNAME`

**A8:** `pvs` shows a one-line summary per PV (quick status). `pvdisplay`
shows full metadata per PV (UUID, PE size, total/free PE, VG membership).

**A9:** LVM device names (`/dev/mapper/vgdata-lvdata`) are stable because they
are based on VG and LV names which you control. `/dev/sdb1` can change to
`/dev/sdc1` if disks are added/removed or reordered at boot.

**A10:** Expand the snapshot immediately: `lvextend -L +1G /dev/vgdata/lv_logs_snap`.
At 100%, the snapshot becomes invalidated (useless). Monitor with `lvs` regularly.

---

# ═══════════════════════════════════════════════════════════
# END OF CHAPTER 2 — LVM (LOGICAL VOLUME MANAGER)
# ═══════════════════════════════════════════════════════════

**Chapter Status:** ✅ COMPLETE
**Previous:** Chapter 1 — Partitioning
**Next:** Chapter 3 — Stratis

---

*Reply "Chapter 3" to begin the Stratis chapter.*
*Reply "DRILL CH2" for 10 additional LVM exam scenarios.*

