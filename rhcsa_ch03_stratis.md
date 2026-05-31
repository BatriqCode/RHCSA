# RHCSA RHEL 9 — COMPLETE EXAM PREPARATION COURSE
## Chapter 3: Stratis — Advanced Storage Management
### Senior Red Hat Linux Engineer · RHCSA/RHCE Instructor · 20+ Years Experience

---

> **Chapter 3 Priority:** Stratis is a confirmed RHCSA exam objective for RHEL 9.
> It appears less frequently than LVM but IS tested. The exam typically asks you
> to create a pool, create a filesystem, and mount it permanently. Know those
> steps cold — and know the critical fstab difference from standard XFS.

---

# ═══════════════════════════════════════════════════════════
# SECTION 1 — THEORY: FROM ZERO TO RHCSA
# ═══════════════════════════════════════════════════════════

## 1.1 What Is Stratis and Why Does It Exist?

LVM is powerful but demands deep expertise. Managing PVs, VGs, LVs, and then layering
filesystems on top with separate tools is complex. Red Hat created **Stratis** as a
higher-level storage management solution that hides this complexity while still
delivering enterprise-grade features.

Think of the difference like this:

```
LVM approach:        pvcreate → vgcreate → lvcreate → mkfs.xfs → mount
                     (5+ commands, many options to remember)

Stratis approach:    stratis pool create → stratis filesystem create → mount
                     (2 commands + mount, fewer decisions)
```

Stratis was introduced in RHEL 8 and is a fully supported feature in RHEL 9.
It is implemented as:
- `stratisd`   — a background daemon (written in Rust) managing all storage operations
- `stratis`    — the command-line interface you interact with
- D-Bus        — communication channel between CLI and daemon

---

## 1.2 Stratis Architecture — The Three Concepts

Stratis has only THREE concepts you must understand (simpler than LVM's three layers):

```
╔══════════════════════════════════════════════════════════════════════╗
║  STRATIS ARCHITECTURE                                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  LAYER 3: FILESYSTEMS — What you mount                              ║
║  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ║
║  │   fs_www         │  │   fs_data        │  │   fs_logs        │  ║
║  │  (XFS, thin)     │  │  (XFS, thin)     │  │  (XFS, thin)     │  ║
║  │  /dev/stratis/   │  │  /dev/stratis/   │  │  /dev/stratis/   │  ║
║  │  pool1/fs_www    │  │  pool1/fs_data   │  │  pool1/fs_logs   │  ║
║  └──────────┬───────┘  └──────────┬───────┘  └──────────┬───────┘  ║
║             └────────────────┬────┘                      │          ║
╠══════════════════════════════╪═══════════════════════════╪══════════╣
║  LAYER 2: POOL — The storage container                  │          ║
║  ┌──────────────────────────┴───────────────────────────┴───────┐  ║
║  │  pool1  (20GB — thin-provisioned, XFS lives here)           │  ║
║  │  ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  ║
║  │  [used by filesystems]        [free pool space]              │  ║
║  └───────────────────────────┬──────────────────────────────────┘  ║
║                              │                                       ║
╠══════════════════════════════╪═══════════════════════════════════════╣
║  LAYER 1: BLOCKDEVS — The raw storage                               ║
║  ┌──────────────────┐  ┌──────────────────┐                        ║
║  │   /dev/sdb       │  │   /dev/sdc       │                        ║
║  │  (whole disk)    │  │  (whole disk)    │                        ║
║  └──────────────────┘  └──────────────────┘                        ║
║  Block devices added to the pool (no pvcreate needed!)              ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Concept 1: Block Devices (blockdev)

Raw disks or partitions added directly to a Stratis pool. Stratis requires:
- **Whole disks** or **unformatted partitions** — no existing filesystem
- No existing LVM metadata

Unlike LVM, there is no separate `pvcreate` step. You hand the disk directly to
`stratis pool create` or `stratis pool add-data`.

### Concept 2: Pool

A **pool** is Stratis's equivalent of LVM's Volume Group. It:
- Aggregates one or more block devices into a single storage container
- Manages a **thin-provisioning metadata layer** automatically
- Maintains an internal **XFS metadata area**
- Located at `/dev/stratis/POOLNAME/`
- Managed entirely by the `stratisd` daemon

Key difference from LVM VG: **Stratis pools are always thin-provisioned.**
You never allocate extents manually. Stratis handles all internal bookkeeping.

### Concept 3: Filesystem

A Stratis **filesystem** is what you actually mount. Each filesystem is:
- An **XFS filesystem** created on a thin volume within the pool
- Always XFS — you cannot choose ext4 or other types
- Accessible at `/dev/stratis/POOLNAME/FSNAME`
- Also accessible via UUID (preferred for fstab)
- Initially allocated **1 TiB** of thin-provisioned virtual space
  (actual physical usage is much less — grows as data is written)
- Can be **snapshotted** at any point

---

## 1.3 How Stratis Works Internally

```
stratis filesystem create pool1 fs_data
         │
         ▼
stratisd daemon (D-Bus)
         │
         ▼
Creates thin-provisioned LV internally (hidden from user)
         │
         ▼
Creates XFS filesystem on that thin volume
         │
         ▼
Exposes device: /dev/stratis/pool1/fs_data
                 (symlink to /dev/dm-X)
         │
         ▼
User mounts: mount /dev/stratis/pool1/fs_data /data
```

Stratis manages the LVM thin pool internally. You never interact with it
directly — that's the point. The `stratisd` daemon is the only process that
touches the underlying LVM/device-mapper layers.

---

## 1.4 Thin Provisioning in Stratis — What "1 TiB" Means

```
stratis filesystem create pool1 myfs
→ Creates a filesystem that appears to be 1 TiB

BUT: The pool itself is only 10GB
AND: Only the data you actually write uses real space

Example:
  Pool physical size:    10 GB  (real disk space)
  fs1 virtual size:     1 TiB  (what df shows!)
  fs2 virtual size:     1 TiB
  fs3 virtual size:     1 TiB
  Total "allocated":    3 TiB from 10GB pool

  fs1 actual usage:     500 MB (data written so far)
  fs2 actual usage:     300 MB
  fs3 actual usage:     200 MB
  Total physical used:  1 GB   (fine — pool is 10GB)
```

This is why `df -h /mnt/fs1` shows **1 TiB** for a Stratis filesystem even
if the physical pool is only 10GB. Do NOT be alarmed — this is by design.

> **EXAM TRAP:** `df` output on a Stratis filesystem shows 1 TiB.
> The real pool size is shown by `stratis pool list`.
> The exam may ask you to check actual pool usage — always use `stratis pool list`.

---

## 1.5 Stratis vs LVM — Feature Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│  STRATIS vs LVM — SIDE BY SIDE                                      │
├────────────────────────┬───────────────────────┬────────────────────┤
│  Feature               │  Stratis              │  LVM               │
├────────────────────────┼───────────────────────┼────────────────────┤
│  Complexity            │  LOW                  │  MEDIUM-HIGH       │
│  Thin provisioning     │  Always (automatic)   │  Optional          │
│  Filesystem choice     │  XFS only             │  Any               │
│  Shrink filesystem     │  NO                   │  YES (ext4)        │
│  Grow filesystem       │  Automatic            │  Manual (xfs_growfs│
│  Snapshots             │  YES                  │  YES               │
│  Encryption            │  YES (Clevis/LUKS)    │  YES (dm-crypt)    │
│  GUI/daemon managed    │  YES (stratisd)        │  NO (direct CLI)  │
│  Commands to learn     │  ~8 key commands       │  ~15+ commands    │
│  fstab special option  │  YES (x-systemd.req.) │  NO               │
│  Exam weight           │  MEDIUM               │  HIGH              │
│  RHEL 9 maturity       │  Stable/supported     │  Production proven │
│  Redundancy (RAID)     │  In development        │  YES              │
└────────────────────────┴───────────────────────┴────────────────────┘
```

---

## 1.6 Stratis Filesystem — The /dev/stratis Path Structure

```
After creating pool1 with fs_data and fs_logs:

/dev/stratis/
└── pool1/
    ├── fs_data   → /dev/dm-3  (device mapper device)
    └── fs_logs   → /dev/dm-4

Also accessible by UUID (for fstab):
  blkid /dev/stratis/pool1/fs_data
  UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="xfs"
```

---

## 1.7 The Critical fstab Difference for Stratis

This is the **#1 exam trap** for Stratis. A Stratis filesystem fstab entry
**must** include a special option that tells systemd to wait for the
`stratisd` daemon before attempting to mount:

```
# STANDARD XFS (partitions or LVM):
UUID=abc123  /data  xfs  defaults  0  2

# STRATIS XFS — CRITICAL DIFFERENCE:
UUID=abc123  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0
                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^    ^
                                   Must have this option!               Pass=0!
```

Two things differ from standard XFS:
1. **`x-systemd.requires=stratisd.service`** — tells systemd to start stratisd
   before mounting this filesystem. Without this, the mount fails at boot because
   stratisd isn't running yet when systemd tries to mount it.
2. **Pass field = `0`** not `2` — Stratis manages its own filesystem checks
   internally. Do NOT set pass to 1 or 2 for Stratis.

---

## 1.8 Stratis Snapshots

A **Stratis snapshot** is a read-write point-in-time copy of a filesystem.
Unlike LVM snapshots (which use CoW), Stratis snapshots are independent
filesystems that share origin data via thin provisioning.

```
Original:  pool1/fs_data    (read-write)
Snapshot:  pool1/fs_data_snap  (read-write, independent copy at snapshot time)

After snapshot:
- Both fs_data and fs_data_snap are fully functional filesystems
- They share data blocks that haven't changed
- Changes to either one don't affect the other
- Both appear as 1 TiB thin volumes
```

This differs from LVM snapshots:
- Stratis snapshots are **read-write** (LVM snapshots default read-only)
- Stratis snapshot space is shared via pool (LVM needs dedicated CoW space)
- Stratis snapshot can be mounted independently and modified

---

## 1.9 Stratis Pool Expansion

You can add block devices to an existing pool to increase its capacity:

```bash
# Initial pool:
stratis pool create pool1 /dev/sdb         # 10GB pool

# Pool running low:
stratis pool add-data pool1 /dev/sdc       # Add 10GB → now 20GB
stratis pool list                          # See expanded size
```

Stratis manages the expansion transparently — no manual LV extension or
filesystem grow needed.

---

## 1.10 Exam Traps and Common Stratis Mistakes

1. **stratisd not running** → ALL stratis commands fail; always enable + start first
2. **Missing `x-systemd.requires=stratisd.service`** in fstab → fails at boot
3. **Pass value = 2** instead of `0` in fstab → wrong for Stratis
4. **Trying to use a disk that has data/filesystem** → pool create fails
5. **Using a partition instead of whole disk** → Stratis prefers whole disks
6. **Confusing pool size with filesystem size** → df shows 1TiB, pool is smaller
7. **Trying to choose ext4** → Stratis only supports XFS internally
8. **Not using UUID in fstab** → use UUID (blkid), not /dev/stratis/... path
9. **Forgetting the pool name** in stratis commands → always: stratis OBJECT ACTION POOL [FS]
10. **Trying to run mkfs on Stratis device** → Stratis manages XFS internally

---

# ═══════════════════════════════════════════════════════════
# SECTION 2 — COMMANDS REFERENCE (EXAM-LEVEL DETAIL)
# ═══════════════════════════════════════════════════════════

## 2.1 Installation and Service Management

Stratis requires two packages and one service:

```bash
# Install packages
dnf install stratisd stratis-cli

# Enable and start the daemon (REQUIRED before any stratis command)
systemctl enable --now stratisd

# Verify service is running
systemctl status stratisd
systemctl is-active stratisd

# If you run stratis commands without stratisd:
# Error: "stratisd must be running for this command to be successful"
```

---

## 2.2 stratis — The Master Command

```
Syntax:  stratis [OPTIONS] OBJECT ACTION [ARGUMENTS]

Objects:
  pool          Manage pools
  filesystem    Manage filesystems (alias: fs)
  blockdev      Manage block devices
  key           Manage encryption keys
  report        Generate reports
  daemon        Manage the daemon
```

---

## 2.3 Pool Commands

### stratis pool create — Create a New Pool

```bash
# Syntax
stratis pool create POOL_NAME BLOCKDEV [BLOCKDEV...]

# Options
--key-desc KEY     Encrypt pool with key (advanced)
--no-overprovision Don't overprovision (cap filesystems at pool size)

# Examples
stratis pool create pool1 /dev/sdb                    # One device
stratis pool create pool1 /dev/sdb /dev/sdc           # Two devices
stratis pool create mypool /dev/nvme0n1               # NVMe disk

# IMPORTANT: Device must be completely clean
# If it has a filesystem or LVM metadata, use:
wipefs -a /dev/sdb    # Clear all signatures first
```

### stratis pool list — List All Pools

```bash
stratis pool list

# Sample output:
# Name    Total Physical   Properties                                    UUID
# pool1   10 GiB / 2.2 GiB in use   ~Ca,~Cr,~Op  abc123-...-def456

# Columns:
# Name          → pool name
# Total Physical → total size of all block devices in pool
# Properties:
#   ~Ca = Not encrypted (Ca = encrypted)
#   ~Cr = No cache (Cr = has cache)
#   ~Op = Not overprovisioned cap
# UUID          → pool identifier
```

### stratis pool add-data — Add Block Device to Pool

```bash
# Syntax
stratis pool add-data POOL_NAME BLOCKDEV [BLOCKDEV...]

# Example
stratis pool add-data pool1 /dev/sdc      # Expand pool1 by adding /dev/sdc
stratis pool list                          # Verify increased total size
```

### stratis pool rename — Rename a Pool

```bash
stratis pool rename OLD_NAME NEW_NAME
# Update /etc/fstab if using /dev/stratis/POOLNAME paths
```

### stratis pool destroy — Delete a Pool

```bash
# Must destroy all filesystems first!
stratis filesystem destroy pool1 fs1
stratis filesystem destroy pool1 fs2
stratis pool destroy pool1

# Or use -f to force (still destroys filesystems first internally)
```

---

## 2.4 Filesystem Commands

### stratis filesystem create — Create a Filesystem

```bash
# Syntax
stratis filesystem create POOL_NAME FS_NAME

# Example
stratis filesystem create pool1 fs_data
stratis filesystem create pool1 fs_logs
stratis filesystem create pool1 fs_www

# Device appears at:
#   /dev/stratis/pool1/fs_data
#   /dev/stratis/pool1/fs_logs
# Also as device mapper:
#   /dev/dm-X (check with ls -la /dev/stratis/pool1/fs_data)
```

### stratis filesystem list — List Filesystems

```bash
# Syntax
stratis filesystem list [POOL_NAME]

stratis filesystem list           # All filesystems in all pools
stratis filesystem list pool1     # Only filesystems in pool1

# Alias: stratis fs list

# Sample output:
# Pool    Filesystem   Total    Used      Created              Device
# pool1   fs_data      1 TiB    546 MiB   2024-01-15 10:30:00  /dev/stratis/pool1/fs_data
# pool1   fs_logs      1 TiB    120 MiB   2024-01-15 10:30:05  /dev/stratis/pool1/fs_logs

# Key: "Total" = 1 TiB (thin virtual), "Used" = actual physical data
```

### stratis filesystem rename — Rename a Filesystem

```bash
stratis filesystem rename pool1 OLD_FS_NAME NEW_FS_NAME
# Device path changes: /dev/stratis/pool1/NEW_FS_NAME
# UUID stays the same → fstab UUID entries still work
```

### stratis filesystem snapshot — Create a Snapshot

```bash
# Syntax
stratis filesystem snapshot POOL_NAME ORIGIN_FS SNAPSHOT_NAME

# Example
stratis filesystem snapshot pool1 fs_data fs_data_snap_20240115
stratis filesystem list pool1     # Both fs_data and snapshot listed

# Mount snapshot (read-write):
mkdir -p /mnt/snapshot
mount /dev/stratis/pool1/fs_data_snap_20240115 /mnt/snapshot
```

### stratis filesystem destroy — Delete a Filesystem

```bash
# Must unmount first!
umount /data
stratis filesystem destroy pool1 fs_data
stratis filesystem list pool1     # fs_data gone
```

---

## 2.5 Block Device Commands

```bash
# List all block devices managed by Stratis
stratis blockdev list
stratis blockdev list pool1          # Only devices in pool1

# Sample output:
# Pool    Device Node   Physical Size  Tier  UUID
# pool1   /dev/sdb      10 GiB         DATA  abc123-...
# pool1   /dev/sdc      10 GiB         DATA  def456-...
```

---

## 2.6 Mounting Stratis Filesystems

### Temporary Mount (No Reboot Survival)

```bash
# Method 1: Device path
mount /dev/stratis/pool1/fs_data /data

# Method 2: UUID (find it first)
blkid /dev/stratis/pool1/fs_data
mount UUID="abc123-..." /data
```

### Permanent Mount via /etc/fstab (Exam-Critical)

```bash
# Step 1: Get UUID
blkid /dev/stratis/pool1/fs_data

# Step 2: Create mount point
mkdir -p /data

# Step 3: Add to fstab — THE CRITICAL FORMAT:
UUID=abc123-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0
#                                                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^ ^
#                                                      MANDATORY special option                    | |
#                                                      Tells systemd: start stratisd first         | |
#                                                                                          dump=0 ─┘ |
#                                                                                          pass=0 ───┘

# Step 4: Reload systemd and test
systemctl daemon-reload
mount -a

# Step 5: Verify
mount | grep /data
df -h /data              # Will show 1 TiB — normal for Stratis!
```

### The Full fstab Line — Copy This Exactly

```
UUID=YOUR_UUID_HERE  /your/mount  xfs  defaults,x-systemd.requires=stratisd.service  0  0
```

Differences from regular XFS fstab:
- Option: `x-systemd.requires=stratisd.service` — required
- Dump (field 5): `0` — same
- Pass (field 6): `0` — NOT 2 like regular XFS!

---

## 2.7 Getting Pool and Filesystem Information

```bash
# Overall pool status
stratis pool list

# Filesystem details
stratis filesystem list

# Block device details
stratis blockdev list

# Detailed report (advanced)
stratis report

# Daemon version
stratis daemon version
stratis daemon redundancy

# Check stratisd logs
journalctl -u stratisd -f        # Follow daemon logs
journalctl -u stratisd --since "1 hour ago"
```

---

## 2.8 Wiping a Device Before Using with Stratis

Stratis refuses to use devices that have existing signatures:

```bash
# Check what's on the device
blkid /dev/sdb
lsblk -f /dev/sdb

# Wipe ALL signatures (filesystem, LVM, partition table)
wipefs -a /dev/sdb

# Or use dd to zero the beginning:
dd if=/dev/zero of=/dev/sdb bs=1M count=10

# Verify it's clean
blkid /dev/sdb         # Should return nothing
lsblk -f /dev/sdb      # No FSTYPE shown
```

---

## 2.9 Complete Stratis Workflow — The PSMC Pattern

```
P  →  pool create       (Create the pool from block device)
S  →  stratis fs create (Create filesystem in pool)
M  →  mkdir + mount     (Create mount point and mount)
C  →  fstab Config      (Make it permanent with correct options)
```

Rhyme: **"Pools Store My Content"**

```bash
# The 4-step Stratis workflow:
systemctl enable --now stratisd                         # Prerequisite!
stratis pool create pool1 /dev/sdb                     # P: pool
stratis filesystem create pool1 fs_data                # S: filesystem
mkdir -p /data && mount /dev/stratis/pool1/fs_data /data  # M: mount
# Get UUID, add to fstab with x-systemd.requires      # C: config
UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs_data)
echo "UUID=${UUID}  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0" >> /etc/fstab
```


# ═══════════════════════════════════════════════════════════
# SECTION 3 — PRACTICAL LABS
# ═══════════════════════════════════════════════════════════

---

## LAB 1 (Beginner) — Install Stratis and Explore the CLI

### Objective
Install stratisd and stratis-cli, start the service, and explore available commands.

### Initial Situation
Fresh RHEL 9 / Rocky Linux 9 installation. No Stratis installed.

### Tasks

**Task 1.1 — Check if Stratis is installed**
```bash
rpm -q stratisd stratis-cli
which stratis
systemctl status stratisd 2>&1 | head -5
```

**Task 1.2 — Install Stratis packages**
```bash
dnf install -y stratisd stratis-cli
rpm -q stratisd stratis-cli   # Verify installed
```

**Task 1.3 — Enable and start stratisd**
```bash
systemctl enable --now stratisd
systemctl status stratisd     # Should show: active (running)
systemctl is-active stratisd  # Should output: active
```

**Task 1.4 — Explore the CLI**
```bash
stratis --help                # All commands
stratis pool --help           # Pool subcommands
stratis filesystem --help     # Filesystem subcommands
stratis blockdev --help       # Blockdev subcommands
stratis pool list             # No pools yet
stratis filesystem list       # No filesystems yet
stratis blockdev list         # No block devices yet
```

**Task 1.5 — Check stratisd daemon version**
```bash
stratis daemon version
```

### Expected Result
Stratis is installed, daemon is running, and CLI responds to commands.

---

## LAB 2 (Beginner) — Create Your First Stratis Pool and Filesystem

### Objective
Create a Stratis pool, create a filesystem, and mount it temporarily.

### Initial Situation
`stratisd` running. `/dev/sdb` (10GB) is empty and clean.

### Pre-Lab Verification
```bash
systemctl is-active stratisd    # Must be: active
lsblk /dev/sdb                  # Must show no partitions
blkid /dev/sdb                  # Must return nothing (clean device)
```

### Tasks

**Task 2.1 — Create a pool**
```bash
stratis pool create pool1 /dev/sdb
stratis pool list
```

Expected output:
```
Name    Total Physical          Properties   UUID
pool1   10 GiB / 615 MiB in use  ~Ca,~Cr,~Op  abc123-...
```

**Task 2.2 — Inspect the pool's block device**
```bash
stratis blockdev list
stratis blockdev list pool1
lsblk /dev/sdb       # Note: sdb is now "claimed" by Stratis
```

**Task 2.3 — Create a filesystem**
```bash
stratis filesystem create pool1 fs_data
stratis filesystem list
stratis fs list pool1   # 'fs' is alias for 'filesystem'
```

**Task 2.4 — Check the device path**
```bash
ls -la /dev/stratis/pool1/
ls -la /dev/stratis/pool1/fs_data
# It's a symlink to a /dev/dm-X device
```

**Task 2.5 — Mount the filesystem**
```bash
mkdir -p /data
mount /dev/stratis/pool1/fs_data /data
```

**Task 2.6 — Check what df reports**
```bash
df -hT /data
# SIZE WILL SHOW 1 TiB — this is correct and expected!
# Stratis uses thin provisioning
```

**Task 2.7 — Create test data**
```bash
echo "Stratis Lab 2 Complete" > /data/test.txt
ls -la /data/
cat /data/test.txt
```

### Verification
```bash
stratis pool list          # pool1 exists
stratis filesystem list    # fs_data exists, device shown
mount | grep /data         # Mounted
df -T /data | grep xfs     # XFS filesystem type
cat /data/test.txt         # Data accessible
```

---

## LAB 3 (Intermediate) — Permanent Mount with fstab

### Objective
Configure a Stratis filesystem to mount automatically at boot using the correct fstab syntax.

### Tasks

**Task 3.1 — Get the UUID of the Stratis filesystem**
```bash
blkid /dev/stratis/pool1/fs_data
# Output: /dev/stratis/pool1/fs_data: UUID="..." TYPE="xfs"

# Get JUST the UUID:
blkid -s UUID -o value /dev/stratis/pool1/fs_data
```

**Task 3.2 — Backup fstab and add Stratis entry**
```bash
cp /etc/fstab /etc/fstab.backup

# Capture UUID
UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs_data)
echo "UUID=${UUID}  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0" >> /etc/fstab

# Verify the entry
tail -3 /etc/fstab
```

**Task 3.3 — Test the fstab entry**
```bash
umount /data                    # Unmount first
systemctl daemon-reload         # Reload systemd unit files
mount -a                        # Mount all fstab entries
mount | grep /data              # Verify mounted
```

**Task 3.4 — Verify filesystem behavior**
```bash
df -hT /data                    # Shows 1 TiB (thin provisioned)
cat /data/test.txt              # Data persisted
findmnt /data                   # Shows UUID source
```

**Task 3.5 — Check stratisd dependency in systemd**
```bash
systemctl list-dependencies --reverse stratisd.service
# Should show your mount unit depending on stratisd
```

### Critical fstab Comparison
```bash
# Show the difference between regular and Stratis entries:
echo "=== Regular XFS ==="
grep " xfs " /etc/fstab | grep -v stratisd

echo "=== Stratis XFS ==="
grep stratisd /etc/fstab
```

---

## LAB 4 (Intermediate) — Add Storage to Existing Pool

### Objective
Expand a Stratis pool by adding a second block device.

### Initial Situation
`pool1` exists on `/dev/sdb`. A new disk `/dev/sdc` (10GB) is available.

### Tasks

**Task 4.1 — Check current pool size**
```bash
stratis pool list               # Note current Total Physical
stratis blockdev list pool1     # One device: /dev/sdb
```

**Task 4.2 — Verify /dev/sdc is clean**
```bash
blkid /dev/sdc                  # Should be empty
lsblk -f /dev/sdc               # No filesystem shown
wipefs -a /dev/sdc              # Wipe if needed
```

**Task 4.3 — Add /dev/sdc to pool1**
```bash
stratis pool add-data pool1 /dev/sdc
```

**Task 4.4 — Verify the expansion**
```bash
stratis pool list               # Total Physical should now be ~20 GiB
stratis blockdev list pool1     # Both /dev/sdb and /dev/sdc listed
```

**Task 4.5 — Create a second filesystem on the expanded pool**
```bash
stratis filesystem create pool1 fs_logs
stratis filesystem list pool1   # Two filesystems
mkdir -p /logs
mount /dev/stratis/pool1/fs_logs /logs
df -hT /logs                    # Shows 1 TiB
```

### Expected Result
```
Pool    Total Physical         Block Devices
pool1   20 GiB / 1.2 GiB      /dev/sdb, /dev/sdc
```

---

## LAB 5 (Intermediate) — Stratis Snapshots

### Objective
Create, mount, use, and manage a Stratis snapshot.

### Tasks

**Task 5.1 — Create origin data**
```bash
for i in {1..5}; do echo "Data file $i" > /data/file$i.txt; done
ls /data/
```

**Task 5.2 — Create a snapshot**
```bash
stratis filesystem snapshot pool1 fs_data fs_data_snap
stratis filesystem list pool1
# Both fs_data and fs_data_snap are listed
```

**Task 5.3 — Mount the snapshot**
```bash
mkdir -p /mnt/snap
mount /dev/stratis/pool1/fs_data_snap /mnt/snap
ls /mnt/snap/           # Same files as /data
cat /mnt/snap/file1.txt # Content matches
```

**Task 5.4 — Modify the origin, verify snapshot is independent**
```bash
echo "NEW DATA" > /data/file_new.txt
ls /data/               # 6 files now
ls /mnt/snap/           # Still 5 files — snapshot preserved
```

**Task 5.5 — Remove snapshot when done**
```bash
umount /mnt/snap
stratis filesystem destroy pool1 fs_data_snap
stratis filesystem list pool1   # Only fs_data remains
```

---

## LAB 6 (RHCSA Exam Level) — Full Stratis Exam Simulation

### Scenario
"Install and configure Stratis. Create a pool named `stratis_pool` using `/dev/sdb`.
Create a filesystem named `stratis_fs` in that pool. Mount it permanently at
`/stratis_data`. Ensure it mounts at boot."

### Timed Solution (target: under 5 minutes)

```bash
# STEP 1: Install and start (if not already done)
dnf install -y stratisd stratis-cli
systemctl enable --now stratisd
systemctl is-active stratisd    # Confirm: active

# STEP 2: Wipe disk (insurance)
wipefs -a /dev/sdb

# STEP 3: Create pool
stratis pool create stratis_pool /dev/sdb
stratis pool list               # Verify

# STEP 4: Create filesystem
stratis filesystem create stratis_pool stratis_fs
stratis filesystem list         # Verify; note device path

# STEP 5: Mount point
mkdir -p /stratis_data

# STEP 6: Get UUID and add to fstab
UUID=$(blkid -s UUID -o value /dev/stratis/stratis_pool/stratis_fs)
echo "UUID=${UUID}  /stratis_data  xfs  defaults,x-systemd.requires=stratisd.service  0  0" >> /etc/fstab

# STEP 7: Mount and verify
systemctl daemon-reload
mount -a
mount | grep stratis_data
df -hT /stratis_data

# STEP 8: Create a test file (exam often asks for this)
echo "Stratis configured" > /stratis_data/status.txt
cat /stratis_data/status.txt
```

### Verification Commands
```bash
stratis pool list | grep stratis_pool
stratis filesystem list | grep stratis_fs
mount | grep stratis_data
grep stratisd /etc/fstab
cat /stratis_data/status.txt
```

---

## LAB 7 (Troubleshooting) — Stratis Common Failures

### Failure Scenario A — stratisd Not Running

```bash
# Symptom: stratis pool list returns error
# Error: "stratisd must be running for this command to be successful"

# Diagnosis:
systemctl status stratisd

# Fix:
systemctl start stratisd
systemctl enable stratisd     # Prevent recurrence
```

### Failure Scenario B — Device Not Clean

```bash
# Symptom: stratis pool create pool1 /dev/sdb fails
# Error: "Device /dev/sdb is not a valid pool member device"

# Diagnosis:
blkid /dev/sdb                # Has existing filesystem?
lsblk -f /dev/sdb             # LVM metadata?

# Fix:
wipefs -a /dev/sdb
# Then retry pool create
```

### Failure Scenario C — Missing x-systemd.requires in fstab

```bash
# Symptom: System hangs or /data not mounted at boot
# stratisd starts AFTER systemd tries to mount fstab entries

# Diagnosis:
grep /data /etc/fstab         # Check for x-systemd.requires

# Fix:
# Edit /etc/fstab:
# Change:
UUID=abc  /data  xfs  defaults  0  0
# To:
UUID=abc  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0
mount -a
```

### Failure Scenario D — Pool Shows "in use" But No Filesystem Exists

```bash
# Pool reports physical usage but filesystems are gone
stratis pool list         # Shows data usage
stratis filesystem list   # Empty

# Fix: Pool overhead is normal even with no filesystems
# Stratis stores ~615 MiB metadata even in empty pool
# This is expected behavior, not a problem
```


# ═══════════════════════════════════════════════════════════
# SECTION 4 — 20 RHCSA EXAM SCENARIOS
# ═══════════════════════════════════════════════════════════

---

### SCENARIO 1
**Task:** Install Stratis, enable the daemon, and create a pool named `mypool`
on `/dev/sdb`. Create a filesystem named `myfs` and mount it at `/mydata`.
Make it permanent.

**Thinking:** Install → start service → pool → filesystem → mount → fstab (with x-systemd option)

**Full Solution:**
```bash
dnf install -y stratisd stratis-cli
systemctl enable --now stratisd
wipefs -a /dev/sdb
stratis pool create mypool /dev/sdb
stratis filesystem create mypool myfs
mkdir -p /mydata
UUID=$(blkid -s UUID -o value /dev/stratis/mypool/myfs)
echo "UUID=${UUID}  /mydata  xfs  defaults,x-systemd.requires=stratisd.service  0  0" >> /etc/fstab
systemctl daemon-reload
mount -a
```

**Verification:**
```bash
stratis pool list
stratis filesystem list
mount | grep mydata
grep stratisd /etc/fstab
```

---

### SCENARIO 2
**Task:** Check whether Stratis is properly installed and the daemon is active.
If not, fix it.

**Full Solution:**
```bash
rpm -q stratisd stratis-cli || dnf install -y stratisd stratis-cli
systemctl is-active stratisd || systemctl enable --now stratisd
systemctl status stratisd
stratis pool list
```

---

### SCENARIO 3
**Task:** A Stratis pool `prodpool` exists on `/dev/sdb`. Add `/dev/sdc`
to expand it.

**Full Solution:**
```bash
stratis pool list                      # Verify prodpool exists
blkid /dev/sdc || wipefs -a /dev/sdc  # Clean device
stratis pool add-data prodpool /dev/sdc
stratis pool list                      # Total Physical increased
stratis blockdev list prodpool         # Both devices shown
```

---

### SCENARIO 4
**Task:** List all Stratis filesystems and their actual disk usage.

**Full Solution:**
```bash
stratis filesystem list
# Output shows Used column (actual physical) vs Total (1 TiB virtual)

# For pool-level physical usage:
stratis pool list
# Shows: Total Physical / X GiB in use

# For more detail:
stratis report
```

---

### SCENARIO 5
**Task:** Create a snapshot of Stratis filesystem `fs_prod` in pool `prodpool`.
Name it `fs_prod_bak`. Mount it read-only at `/mnt/backup`.

**Full Solution:**
```bash
# Create snapshot (Stratis snapshots are mounted read-write by default)
stratis filesystem snapshot prodpool fs_prod fs_prod_bak
stratis filesystem list prodpool      # Both listed

# Mount (use -o ro for read-only)
mkdir -p /mnt/backup
mount -o ro /dev/stratis/prodpool/fs_prod_bak /mnt/backup
mount | grep backup                   # Verify ro mount
ls /mnt/backup/                       # Same content as fs_prod at snapshot time
```

---

### SCENARIO 6
**Task:** Show the UUID of a Stratis filesystem `fs_data` in pool `pool1`.

**Full Solution:**
```bash
blkid /dev/stratis/pool1/fs_data
blkid -s UUID -o value /dev/stratis/pool1/fs_data  # UUID only
# OR:
stratis filesystem list pool1
# Device column shows path; use blkid to get UUID
```

---

### SCENARIO 7
**Task:** A Stratis filesystem is mounted at `/appdata` but fails to mount
after reboot. Diagnose and fix.

**Thinking:** Check fstab for missing `x-systemd.requires` option and wrong pass value.

**Full Solution:**
```bash
# Diagnosis
grep appdata /etc/fstab
# Probably missing x-systemd.requires=stratisd.service
# Or pass value is 2 instead of 0

# Get device UUID
blkid /dev/stratis/pool1/fs_app

# Fix fstab (remove wrong line, add correct line):
grep -n appdata /etc/fstab              # Find line number
sed -i '/appdata/d' /etc/fstab          # Remove bad line
UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs_app)
echo "UUID=${UUID}  /appdata  xfs  defaults,x-systemd.requires=stratisd.service  0  0" >> /etc/fstab

# Test
systemctl daemon-reload
mount -a
mount | grep appdata
```

---

### SCENARIO 8
**Task:** Destroy a Stratis filesystem `fs_old` in pool `prodpool`.
The filesystem is currently mounted at `/old`.

**Full Solution:**
```bash
# Step 1: Unmount
umount /old

# Step 2: Remove fstab entry
sed -i '/\/old/d' /etc/fstab

# Step 3: Destroy filesystem
stratis filesystem destroy prodpool fs_old
stratis filesystem list prodpool       # fs_old gone

# Step 4: Verify pool space reclaimed
stratis pool list                      # "in use" should decrease
```

---

### SCENARIO 9
**Task:** Create two filesystems in the same pool (`pool1`): `fs_web` and `fs_db`.
Mount them at `/var/www` and `/var/lib/db` permanently.

**Full Solution:**
```bash
stratis filesystem create pool1 fs_web
stratis filesystem create pool1 fs_db
mkdir -p /var/www /var/lib/db

UUID_WEB=$(blkid -s UUID -o value /dev/stratis/pool1/fs_web)
UUID_DB=$(blkid -s UUID -o value /dev/stratis/pool1/fs_db)

cat >> /etc/fstab << EOF
UUID=${UUID_WEB}  /var/www      xfs  defaults,x-systemd.requires=stratisd.service  0  0
UUID=${UUID_DB}   /var/lib/db   xfs  defaults,x-systemd.requires=stratisd.service  0  0
EOF

systemctl daemon-reload
mount -a
df -hT /var/www /var/lib/db
```

---

### SCENARIO 10
**Task:** Show what block devices are being used by all Stratis pools
on the system.

**Full Solution:**
```bash
stratis blockdev list                   # All block devices managed by Stratis
stratis blockdev list pool1             # For specific pool

# With lsblk to see kernel perspective:
lsblk | grep -A2 "sdb\|sdc"
```

---

### SCENARIO 11
**Task:** Rename the Stratis pool `old_pool` to `prod_pool`.

**Full Solution:**
```bash
stratis pool list                       # Confirm old_pool exists
stratis pool rename old_pool prod_pool
stratis pool list                       # Confirm prod_pool now exists

# Device paths change:
ls /dev/stratis/                        # prod_pool directory now

# UUID stays same → fstab UUID entries still work
# But if fstab uses /dev/stratis/old_pool/... (not UUID), update it:
sed -i 's|/dev/stratis/old_pool|/dev/stratis/prod_pool|g' /etc/fstab
```

---

### SCENARIO 12
**Task:** Confirm that a Stratis filesystem is truly XFS underneath.

**Full Solution:**
```bash
# Method 1: blkid
blkid /dev/stratis/pool1/fs_data
# Output: TYPE="xfs"

# Method 2: df
df -T /data
# Shows: xfs

# Method 3: xfs_info
xfs_info /data
# Shows XFS superblock info

# Method 4: mount
mount | grep /data
# Shows: type xfs
```

---

### SCENARIO 13
**Task:** Check how much physical space the Stratis pool `pool1` is actually using.

**Full Solution:**
```bash
stratis pool list
# Output: pool1  10 GiB / 2.1 GiB in use

# The "X GiB in use" is the real physical consumption
# NOT the 1 TiB shown by df

# Detailed breakdown per filesystem:
stratis filesystem list pool1
# Shows "Used" per filesystem
```

---

### SCENARIO 14
**Task:** Completely remove Stratis pool `testpool` and all its filesystems.
The pool uses `/dev/sdb`.

**Full Solution:**
```bash
# Step 1: Unmount all filesystems in the pool
umount /test1 /test2 2>/dev/null

# Step 2: Remove fstab entries
sed -i '/testpool/d' /etc/fstab

# Step 3: Destroy filesystems
stratis filesystem list testpool
stratis filesystem destroy testpool fs1
stratis filesystem destroy testpool fs2

# Step 4: Destroy pool
stratis pool destroy testpool

# Step 5: Verify disk is free
stratis pool list                  # testpool gone
blkid /dev/sdb                     # Stratis metadata still there
wipefs -a /dev/sdb                 # Optional: clean for reuse
```

---

### SCENARIO 15
**Task:** Create a Stratis pool that spans two disks for more capacity.

**Full Solution:**
```bash
# Method A: Create pool with both disks at once
wipefs -a /dev/sdb /dev/sdc
stratis pool create bigpool /dev/sdb /dev/sdc
stratis pool list                  # Shows ~20 GiB total

# Method B: Create pool with one disk, then add second
stratis pool create bigpool /dev/sdb
stratis pool add-data bigpool /dev/sdc
```

---

### SCENARIO 16
**Task:** After `stratis pool add-data pool1 /dev/sdc`, verify that the
device is being used correctly.

**Full Solution:**
```bash
stratis blockdev list pool1
# Both /dev/sdb and /dev/sdc should appear as DATA tier

stratis pool list
# Total Physical should show combined size

lsblk /dev/sdb /dev/sdc
# Both show as part of Stratis device mapper
```

---

### SCENARIO 17
**Task:** Write a shell script that:
1. Checks if stratisd is running (if not, starts it)
2. Creates pool `autopool` on `/dev/sdb` (if it doesn't exist)
3. Creates filesystem `autofs` in that pool (if it doesn't exist)
4. Mounts at `/autodata`

**Full Solution:**
```bash
#!/bin/bash
POOL="autopool"
FS="autofs"
DEVICE="/dev/sdb"
MOUNT="/autodata"

# Ensure stratisd is running
if ! systemctl is-active --quiet stratisd; then
    echo "Starting stratisd..."
    systemctl enable --now stratisd
fi

# Create pool if missing
if ! stratis pool list | grep -q "^${POOL}"; then
    echo "Creating pool ${POOL}..."
    wipefs -a ${DEVICE}
    stratis pool create ${POOL} ${DEVICE}
fi

# Create filesystem if missing
if ! stratis filesystem list ${POOL} | grep -q " ${FS} "; then
    echo "Creating filesystem ${FS}..."
    stratis filesystem create ${POOL} ${FS}
fi

# Mount if not already mounted
mkdir -p ${MOUNT}
if ! mount | grep -q "${MOUNT}"; then
    mount /dev/stratis/${POOL}/${FS} ${MOUNT}
fi

# Verify
stratis pool list
stratis filesystem list ${POOL}
mount | grep ${MOUNT}
echo "Done."
```

---

### SCENARIO 18
**Task:** Explain what happens when you run `df -h` on a Stratis filesystem.
What size will it show and why?

**Answer + Demonstration:**
```bash
stratis pool create demo /dev/sdb      # /dev/sdb = 10GB
stratis filesystem create demo myfs
mount /dev/stratis/demo/myfs /tmp/demo
df -h /tmp/demo

# Output: Size = 1.0T (1 TiB)
# WHY: Stratis uses thin provisioning. Each filesystem is created on a
# thin volume with 1 TiB of virtual space. Physical usage grows as you
# write data. The pool's actual physical limit is 10GB.

# To see ACTUAL physical usage:
stratis pool list
# Shows: 10 GiB / 615 MiB in use
```

---

### SCENARIO 19
**Task:** Create a Stratis filesystem, write 100MB of data to it, then
create a snapshot. Delete the original data. Verify snapshot still has data.

**Full Solution:**
```bash
# Setup
stratis pool create pool1 /dev/sdb
stratis filesystem create pool1 origin_fs
mkdir -p /origin
mount /dev/stratis/pool1/origin_fs /origin

# Write 100MB
dd if=/dev/urandom of=/origin/testfile.dat bs=1M count=100
ls -lh /origin/testfile.dat

# Snapshot
stratis filesystem snapshot pool1 origin_fs origin_snap
mkdir -p /snapshot
mount /dev/stratis/pool1/origin_snap /snapshot
ls /snapshot/                          # testfile.dat is here

# Delete from origin
rm /origin/testfile.dat
ls /origin/                            # Gone from origin

# Verify snapshot preserved it
ls /snapshot/                          # Still there!
ls -lh /snapshot/testfile.dat          # 100MB still intact
```

---

### SCENARIO 20
**Task:** Configure systemd to ensure `/data` (a Stratis filesystem) mounts
only after `stratisd.service` starts. Verify the dependency.

**Full Solution:**
```bash
# The fstab option handles this automatically:
UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs_data)
echo "UUID=${UUID}  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0" \
     >> /etc/fstab

systemctl daemon-reload

# Verify the dependency chain was created:
systemctl cat data.mount 2>/dev/null || \
    systemctl show -p Requires,After $(systemctl list-units --type=mount | grep data | awk '{print $1}')

# Check the dependency tree:
systemctl list-dependencies --reverse stratisd.service | head -10
# Should show the data.mount unit depending on stratisd
```


# ═══════════════════════════════════════════════════════════
# SECTION 5 — TROUBLESHOOTING GUIDE
# ═══════════════════════════════════════════════════════════

---

## FAILURE 1: "stratisd must be running for this command to be successful"

**Cause:** The `stratisd` daemon is not running. Every stratis command requires it.

**Diagnosis:**
```bash
systemctl status stratisd
systemctl is-active stratisd      # Should output "active"
```

**Fix:**
```bash
systemctl start stratisd
systemctl enable stratisd         # Prevent at next boot
systemctl is-active stratisd      # Confirm active
```

---

## FAILURE 2: "stratis pool create" Fails — Device Not Clean

**Error message:** something like "Error: ... signatures present"
or "device is not a valid pool member"

**Cause:** `/dev/sdb` has an existing filesystem, LVM metadata, or partition table.

**Diagnosis:**
```bash
blkid /dev/sdb                    # Any filesystem signature?
lsblk -f /dev/sdb                 # Filesystem type shown?
pvs /dev/sdb 2>/dev/null          # LVM PV?
```

**Fix:**
```bash
wipefs -a /dev/sdb                # Remove ALL signatures
# Double-check:
blkid /dev/sdb                    # Should return nothing now
# Then retry:
stratis pool create pool1 /dev/sdb
```

---

## FAILURE 3: Stratis Filesystem Not Mounting at Boot

**Cause A:** Missing `x-systemd.requires=stratisd.service` option in fstab.
**Cause B:** Pass field set to 2 (should be 0).
**Cause C:** stratisd not enabled for boot.

**Diagnosis:**
```bash
grep /data /etc/fstab             # Check options and pass field
systemctl is-enabled stratisd     # Should be "enabled"
journalctl -b -u stratisd         # Check boot logs
```

**Fix:**
```bash
# Fix stratisd enable:
systemctl enable stratisd

# Fix fstab — get UUID first:
UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs_data)

# Remove wrong entry and add correct one:
sed -i '/\/data/d' /etc/fstab
echo "UUID=${UUID}  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0" \
    >> /etc/fstab

systemctl daemon-reload
mount -a
```

---

## FAILURE 4: "df" Shows 1 TiB — Is This an Error?

**This is NOT an error.** This is expected Stratis behavior.

**Explanation:**
```bash
df -h /data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/dm-4       1.0T  7.0M  1.0T   1% /data
# ← The 1.0T is normal! Stratis thin-provisions 1 TiB per filesystem.

# Real physical usage:
stratis pool list
# Name   Total Physical          ...
# pool1  10 GiB / 650 MiB in use  ← THIS is reality
```

**To check real space:**
```bash
stratis pool list           # Pool's actual physical usage
stratis filesystem list     # Per-filesystem actual usage
```

---

## FAILURE 5: Cannot Add Block Device to Pool — Already in Pool

**Error:** "Block device already owned by Stratis"

**Diagnosis:**
```bash
stratis blockdev list               # Which pool owns the device?
stratis pool list                   # Confirm pool name
```

**Fix:**
```bash
# A block device can only belong to ONE Stratis pool
# Either use a different device, or remove from current pool first:
stratis filesystem list POOL_NAME   # Destroy filesystems first
stratis filesystem destroy POOL_NAME FS_NAME
stratis pool destroy POOL_NAME
wipefs -a /dev/sdb                  # Then clean and reuse
```

---

## FAILURE 6: Stratis Pool Showing "Stopped" or Unavailable After Reboot

**Cause:** Stratisd took too long to start, or device mapper devices not ready.

**Diagnosis:**
```bash
systemctl status stratisd
journalctl -u stratisd -b          # Boot-time logs
lsblk | grep dm                    # Device mapper devices visible?
```

**Fix:**
```bash
systemctl restart stratisd
stratis pool list                  # Should reappear
vgchange -a y 2>/dev/null          # Activate any hidden dm devices
mount -a                           # Remount fstab entries
```

---

## FAILURE 7: /dev/stratis/pool1/fs_data Does Not Exist

**Cause:** Pool or filesystem not active, or stratisd not running.

**Diagnosis:**
```bash
systemctl is-active stratisd
stratis pool list
stratis filesystem list
ls /dev/stratis/
```

**Fix:**
```bash
# Start stratisd if stopped:
systemctl start stratisd

# Activate the pool:
stratis pool list                  # Should appear now
ls /dev/stratis/pool1/fs_data      # Should exist now

# If still missing:
stratisd --version                 # Check daemon is functional
journalctl -u stratisd | tail -20  # Read recent errors
```

---

## FAILURE 8: Stratis Filesystem Shows 100% Full

**Cause:** Physical pool space exhausted. All thin space consumed.

**Diagnosis:**
```bash
stratis pool list
# Shows:  10 GiB / 10 GiB in use  ← 100%!
```

**Fix:**
```bash
# Add more physical storage to the pool:
pvcreate /dev/sdc 2>/dev/null       # Not needed — Stratis handles it
wipefs -a /dev/sdc
stratis pool add-data pool1 /dev/sdc
stratis pool list                   # Now 20 GiB, less % used
```

---

## FAILURE 9: stratis Command Not Found After Install

**Cause:** Package installed but PATH not including /sbin.

**Diagnosis:**
```bash
which stratis
rpm -ql stratis-cli | grep bin
```

**Fix:**
```bash
# Full path:
/usr/sbin/stratis pool list

# Or fix PATH:
export PATH=$PATH:/usr/sbin
# Add to ~/.bashrc for permanence
```

---

## FAILURE 10: Snapshot Shows Same Size as Origin (1 TiB) — Is It Full?

**This is normal.** Each Stratis filesystem (including snapshots) shows 1 TiB.

```bash
stratis filesystem list pool1
# fs_data       1 TiB   500 MiB  ← real usage
# fs_data_snap  1 TiB   500 MiB  ← real usage (shared blocks)

# Both show 1 TiB because they are BOTH thin volumes
# The shared blocks count toward pool usage only once
# Real pool usage = what stratis pool list shows
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 6 — MEMORY AIDS
# ═══════════════════════════════════════════════════════════

## The Stratis PSMC Workflow (4 steps)

```
P  →  pool create        stratis pool create POOL DEVICE
S  →  filesystem create  stratis filesystem create POOL FSNAME
M  →  mkdir + mount      mkdir -p /mnt && mount /dev/stratis/POOL/FS /mnt
C  →  fstab Config       UUID=...  /mnt  xfs  defaults,x-systemd.requires=stratisd.service  0  0
```

**Rhyme:** "Pools Store My Content"

---

## The #1 Exam Differentiator — fstab Comparison

```
═══════════════════════════════════════════════════════════
  STANDARD XFS (partition or LVM):
  UUID=abc  /data  xfs  defaults  0  2

  STRATIS XFS — TWO DIFFERENCES:
  UUID=abc  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0
                                  ↑                                       ↑
                            EXTRA OPTION                           PASS = 0
═══════════════════════════════════════════════════════════
```

**Memory hook:** "Stratis needs ZERO pass and REQUIRES its daemon"

---

## Stratis Command Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║  STRATIS COMMAND QUICK REFERENCE                                 ║
╠══════════════════════════════════════════════════════════════════╣
║  SERVICE                                                         ║
║    systemctl enable --now stratisd      Start + enable daemon    ║
╠══════════════════════════════════════════════════════════════════╣
║  POOL                                                            ║
║    stratis pool create NAME DEVICE      Create pool              ║
║    stratis pool list                    List all pools           ║
║    stratis pool add-data NAME DEVICE    Add device to pool       ║
║    stratis pool rename OLD NEW          Rename pool              ║
║    stratis pool destroy NAME            Delete pool              ║
╠══════════════════════════════════════════════════════════════════╣
║  FILESYSTEM                                                      ║
║    stratis filesystem create POOL NAME  Create filesystem        ║
║    stratis filesystem list [POOL]       List filesystems         ║
║    stratis filesystem snapshot POOL FS SNAP  Create snapshot     ║
║    stratis filesystem rename POOL OLD NEW    Rename FS           ║
║    stratis filesystem destroy POOL NAME      Delete FS           ║
╠══════════════════════════════════════════════════════════════════╣
║  BLOCK DEVICES                                                   ║
║    stratis blockdev list [POOL]         List block devices       ║
╠══════════════════════════════════════════════════════════════════╣
║  ALIASES                                                         ║
║    stratis fs = stratis filesystem      Shorter alias            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## The 1 TiB Reminder — Don't Panic!

```
df -h /stratis_mount → shows 1.0T → NORMAL (thin provisioned virtual size)
stratis pool list    → shows real physical pool size and actual usage
stratis fs list      → shows per-filesystem real usage
```

---

## Key Fact Table — Stratis for Exam Day

```
╔═══════════════════════════════════════════════════════════════╗
║  STRATIS EXAM FACTS — MEMORIZE THESE                         ║
╠═══════════════════════════════════════════════════════════════╣
║  Packages:    stratisd  stratis-cli                          ║
║  Daemon:      stratisd.service                               ║
║  FS type:     XFS only (always)                              ║
║  Thin prov:   Always (1 TiB virtual per FS)                  ║
║  Device path: /dev/stratis/POOLNAME/FSNAME                   ║
║  fstab option: x-systemd.requires=stratisd.service          ║
║  fstab pass:  0 (not 2!)                                     ║
║  fstab UUID:  Use blkid to get UUID                          ║
║  Snapshots:   Read-WRITE (unlike LVM default read-only)      ║
║  Pool expand: stratis pool add-data                          ║
║  No pvcreate: No need — hand disk directly to stratis        ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Stratis vs LVM Comparison Table (Exam Quick-Recall)

| Question | Stratis Answer | LVM Answer |
|---|---|---|
| First step with disk? | `stratis pool create` | `pvcreate` |
| Storage container name? | Pool | Volume Group (VG) |
| Usable unit name? | Filesystem | Logical Volume (LV) |
| Filesystem type? | XFS only | Any |
| Thin provisioning? | Always automatic | Optional, manual |
| fstab special option? | `x-systemd.requires=stratisd.service` | None |
| fstab pass value? | `0` | `2` |
| Snapshot type? | Read-write | Read-only (default) |
| Daemon required? | YES (`stratisd`) | No |
| Grow filesystem? | Automatic | Manual (`xfs_growfs`) |
| CLI tool? | `stratis` | `pvcreate/vgcreate/lvcreate` |
| `df` filesystem size? | 1 TiB (thin virtual) | Actual size |

---

## Top 10 Stratis Commands for RHCSA

```
 1. systemctl enable --now stratisd      ← ALWAYS FIRST
 2. wipefs -a /dev/sdX                   ← Clean device before use
 3. stratis pool create POOL /dev/sdX    ← Create pool
 4. stratis filesystem create POOL FS   ← Create filesystem
 5. stratis pool list                   ← Verify pool
 6. stratis filesystem list             ← Verify filesystem
 7. blkid /dev/stratis/POOL/FS          ← Get UUID for fstab
 8. /etc/fstab with x-systemd.requires  ← Make permanent
 9. stratis pool add-data POOL /dev/sdY ← Expand pool
10. stratis filesystem snapshot POOL FS SNAP ← Snapshot
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 7 — COMMAND COMPARISON TABLES
# ═══════════════════════════════════════════════════════════

## Complete Storage Solution Comparison

| Feature | Standard Partition | LVM | Stratis |
|---|---|---|---|
| Setup commands | 3–5 | 5–7 | 2–3 |
| Requires daemon | No | No | YES (stratisd) |
| Online resize | No | YES | Auto |
| Spans disks | No | YES | YES |
| Snapshots | No | YES | YES |
| Snapshot type | — | Read-only (default) | **Read-write** |
| Filesystem choice | Any | Any | XFS only |
| Thin provisioning | No | Optional | Always |
| fstab special option | None | None | `x-systemd.requires=stratisd.service` |
| fstab pass value | 1 or 2 | 2 | **0** |
| `df` size accuracy | Exact | Exact | Shows 1 TiB (thin) |
| RHCSA exam weight | HIGH | **HIGHEST** | MEDIUM |
| Complexity | Low | High | Medium |

---

## stratis Commands Mapped to LVM Equivalents

| Operation | Stratis Command | LVM Equivalent |
|---|---|---|
| Initialize disk | (none needed) | `pvcreate /dev/sdX` |
| Create storage pool | `stratis pool create POOL DEVICE` | `vgcreate VG DEVICE` |
| Show pool info | `stratis pool list` | `vgs` |
| Expand pool | `stratis pool add-data POOL DEVICE` | `pvcreate` + `vgextend` |
| Create usable unit | `stratis filesystem create POOL NAME` | `lvcreate -L Ng -n NAME VG` |
| Show filesystem | `stratis filesystem list` | `lvs` |
| Create snapshot | `stratis filesystem snapshot POOL FS SNAP` | `lvcreate -s -L Ng -n SNAP /dev/VG/LV` |
| Delete filesystem | `stratis filesystem destroy POOL NAME` | `lvremove /dev/VG/LV` |
| Delete pool | `stratis pool destroy POOL` | `vgremove VG` + `pvremove` |

---

## fstab Entry Format Comparison

| Storage Type | Device Field | fstype | Options | dump | pass |
|---|---|---|---|---|---|
| Standard XFS | `UUID=...` | `xfs` | `defaults` | `0` | `2` |
| ext4 | `UUID=...` | `ext4` | `defaults` | `0` | `2` |
| LVM XFS | `/dev/VG/LV` | `xfs` | `defaults` | `0` | `2` |
| Stratis XFS | `UUID=...` | `xfs` | `defaults,x-systemd.requires=stratisd.service` | `0` | **`0`** |
| swap | `UUID=...` | `swap` | `defaults` | `0` | `0` |
| NFS | `server:/share` | `nfs` | `defaults,_netdev` | `0` | `0` |

---

# ═══════════════════════════════════════════════════════════
# SECTION 8 — HANDS-ON EXAM ENVIRONMENT
# ═══════════════════════════════════════════════════════════

## VM Storage for Stratis Practice

```
Disk Layout for Complete Chapter 3 Practice:
┌──────────────────────────────────────────────────────────┐
│  /dev/sda  →  System disk (OS) — do not touch           │
│  /dev/sdb  →  10 GB — Primary Stratis disk               │
│  /dev/sdc  →  10 GB — Secondary Stratis disk (expansion) │
│  /dev/sdd  →  10 GB — Reserved for other chapters        │
└──────────────────────────────────────────────────────────┘
```

## Stratis Lab Reset Script

```bash
#!/bin/bash
# reset_stratis_lab.sh — DESTRUCTIVE — removes all Stratis on sdb/sdc
# Run as root

echo "WARNING: This will destroy all Stratis on /dev/sdb and /dev/sdc"
read -p "Type YES to continue: " confirm
[ "$confirm" != "YES" ] && exit 1

# Unmount any Stratis mounts
for mp in /data /logs /appdata /mydata /stratis_data /autodata /var/www /var/lib/db; do
    umount $mp 2>/dev/null || true
done
umount /mnt/snap 2>/dev/null || true
umount /mnt/backup 2>/dev/null || true
umount /snapshot 2>/dev/null || true

# Get all pools and destroy
systemctl is-active stratisd || systemctl start stratisd

for pool in $(stratis pool list --no-headers 2>/dev/null | awk '{print $1}'); do
    echo "Destroying pool: $pool"
    for fs in $(stratis filesystem list $pool --no-headers 2>/dev/null | awk '{print $2}'); do
        echo "  Destroying filesystem: $fs"
        stratis filesystem destroy $pool $fs 2>/dev/null || true
    done
    stratis pool destroy $pool 2>/dev/null || true
done

# Wipe disks
wipefs -a /dev/sdb 2>/dev/null
wipefs -a /dev/sdc 2>/dev/null

echo "Stratis lab environment reset complete."
stratis pool list
lsblk /dev/sdb /dev/sdc
```

---

# ═══════════════════════════════════════════════════════════
# SECTION 9 — EXAM CHECKLIST
# ═══════════════════════════════════════════════════════════

## Chapter 3: Stratis — Pre-Exam Checklist

### Conceptual Knowledge ✓
- [ ] Name the two Stratis packages (stratisd, stratis-cli)
- [ ] Name the three Stratis concepts (blockdev, pool, filesystem)
- [ ] Explain why `df` shows 1 TiB for a Stratis filesystem
- [ ] Recite the fstab option for Stratis from memory (`x-systemd.requires=stratisd.service`)
- [ ] State the correct pass value for Stratis in fstab (`0`)
- [ ] Explain why Stratis snapshots are read-write (vs LVM read-only)
- [ ] State which filesystem Stratis always uses (XFS)
- [ ] Explain how to check real pool space usage (`stratis pool list`)

### Practical Skills ✓
- [ ] Install Stratis: `dnf install stratisd stratis-cli`
- [ ] Enable and start daemon: `systemctl enable --now stratisd`
- [ ] Wipe a device: `wipefs -a /dev/sdX`
- [ ] Create a pool: `stratis pool create NAME DEVICE`
- [ ] Add device to pool: `stratis pool add-data NAME DEVICE`
- [ ] Create filesystem: `stratis filesystem create POOL NAME`
- [ ] List pools: `stratis pool list`
- [ ] List filesystems: `stratis filesystem list`
- [ ] List block devices: `stratis blockdev list`
- [ ] Get UUID: `blkid /dev/stratis/POOL/FS`
- [ ] Mount temporarily: `mount /dev/stratis/POOL/FS /mountpoint`
- [ ] Write correct fstab entry with x-systemd.requires and pass=0
- [ ] Create snapshot: `stratis filesystem snapshot POOL FS SNAP`
- [ ] Mount snapshot read-only: `mount -o ro /dev/stratis/POOL/SNAP /mnt`
- [ ] Destroy filesystem: `stratis filesystem destroy POOL NAME`
- [ ] Destroy pool: `stratis pool destroy NAME`
- [ ] Troubleshoot: stratisd not running → `systemctl start stratisd`
- [ ] Troubleshoot: device not clean → `wipefs -a`
- [ ] Troubleshoot: boot failure → check fstab for x-systemd option

---

# ═══════════════════════════════════════════════════════════
# SECTION 10 — CHAPTER 3 MASTER CHALLENGE (45 minutes)
# ═══════════════════════════════════════════════════════════

## Master Challenge: Full Stratis Configuration

**Time limit:** 45 minutes
**Environment:** RHEL 9 with `/dev/sdb` (10GB) and `/dev/sdc` (10GB), both clean

---

### Exam Statement

You are deploying a new application server. The storage team has provided
two new disks. Configure Stratis storage as follows:

**Task 1:** Install Stratis. Ensure the daemon starts automatically on boot.

**Task 2:** Create a Stratis pool named `apppool` using `/dev/sdb` as the
initial block device.

**Task 3:** Create two filesystems in `apppool`:
- `fs_application` — mount permanently at `/application`
- `fs_database` — mount permanently at `/database`
Both must mount automatically at boot without causing boot failures.

**Task 4:** Add `/dev/sdc` to `apppool` to increase its capacity.
Verify the total pool size after expansion.

**Task 5:** Create a point-in-time snapshot of `fs_database` named
`fs_database_snap`. Mount the snapshot at `/mnt/db_snap`.

**Task 6:** Create the following files:
- `/application/app.conf` containing the text: `APP_ENV=production`
- `/database/db.conf` containing the text: `DB_ENGINE=postgresql`

**Task 7:** Verify that after simulating a remount (`umount` then `mount -a`),
all data persists and all mounts come back correctly.

---

### Full Solution

```bash
#!/bin/bash
# RHCSA Chapter 3 Stratis Master Challenge — Full Solution

set -e

echo "=== RHCSA Chapter 3: Stratis Master Challenge ==="
echo ""

# === PRE-CHECK ===
echo "--- Pre-flight checks ---"
lsblk /dev/sdb /dev/sdc         # Confirm both disks exist
blkid /dev/sdb || true
blkid /dev/sdc || true

# === TASK 1: Install Stratis and enable daemon ===
echo ""
echo "--- Task 1: Installing Stratis ---"
dnf install -y stratisd stratis-cli
systemctl enable --now stratisd
systemctl is-active stratisd && echo "stratisd: ACTIVE" || { echo "FAILED"; exit 1; }

# === TASK 2: Create pool ===
echo ""
echo "--- Task 2: Creating pool apppool on /dev/sdb ---"
wipefs -a /dev/sdb
stratis pool create apppool /dev/sdb
stratis pool list
echo "Pool creation: DONE"

# === TASK 3: Create filesystems and permanent mounts ===
echo ""
echo "--- Task 3: Creating filesystems ---"
stratis filesystem create apppool fs_application
stratis filesystem create apppool fs_database
stratis filesystem list apppool

mkdir -p /application /database

# Get UUIDs
UUID_APP=$(blkid -s UUID -o value /dev/stratis/apppool/fs_application)
UUID_DB=$(blkid -s UUID -o value /dev/stratis/apppool/fs_database)

echo "fs_application UUID: ${UUID_APP}"
echo "fs_database    UUID: ${UUID_DB}"

# Add to fstab with CORRECT Stratis options
cp /etc/fstab /etc/fstab.backup.ch3

cat >> /etc/fstab << FSTAB
# Stratis Chapter 3 Challenge
UUID=${UUID_APP}  /application  xfs  defaults,x-systemd.requires=stratisd.service  0  0
UUID=${UUID_DB}   /database     xfs  defaults,x-systemd.requires=stratisd.service  0  0
FSTAB

systemctl daemon-reload
mount -a
mount | grep -E "application|database"
echo "Filesystem mounts: DONE"

# === TASK 4: Add /dev/sdc to pool ===
echo ""
echo "--- Task 4: Expanding apppool with /dev/sdc ---"
wipefs -a /dev/sdc
stratis pool add-data apppool /dev/sdc
stratis pool list               # Should show ~20 GiB
stratis blockdev list apppool   # Both /dev/sdb and /dev/sdc
echo "Pool expansion: DONE"

# === TASK 5: Snapshot of fs_database ===
echo ""
echo "--- Task 5: Creating snapshot of fs_database ---"
stratis filesystem snapshot apppool fs_database fs_database_snap
stratis filesystem list apppool   # Snapshot visible

mkdir -p /mnt/db_snap
mount /dev/stratis/apppool/fs_database_snap /mnt/db_snap
mount | grep db_snap
echo "Snapshot: DONE"

# === TASK 6: Create required files ===
echo ""
echo "--- Task 6: Creating config files ---"
echo "APP_ENV=production" > /application/app.conf
echo "DB_ENGINE=postgresql" > /database/db.conf

cat /application/app.conf
cat /database/db.conf
echo "Config files: DONE"

# === TASK 7: Simulate remount ===
echo ""
echo "--- Task 7: Simulating remount test ---"
umount /application /database
mount -a

echo "Post-remount verification:"
mount | grep -E "application|database"
cat /application/app.conf         # Must show APP_ENV=production
cat /database/db.conf             # Must show DB_ENGINE=postgresql

# === FINAL VERIFICATION REPORT ===
echo ""
echo "════════════════════════════════════════"
echo "        FINAL VERIFICATION REPORT"
echo "════════════════════════════════════════"
echo ""
echo "--- Stratis service ---"
systemctl is-active stratisd

echo ""
echo "--- Pools ---"
stratis pool list

echo ""
echo "--- Block Devices ---"
stratis blockdev list apppool

echo ""
echo "--- Filesystems ---"
stratis filesystem list apppool

echo ""
echo "--- Mount Status ---"
df -hT | grep -E "application|database"

echo ""
echo "--- fstab entries ---"
grep stratisd /etc/fstab

echo ""
echo "--- Snapshot ---"
mount | grep db_snap

echo ""
echo "--- Config file contents ---"
echo "app.conf: $(cat /application/app.conf)"
echo "db.conf:  $(cat /database/db.conf)"

echo ""
echo "=== CHALLENGE COMPLETE ==="
```

---

### Grader Verification Commands

```bash
# Every one of these must succeed:
systemctl is-active stratisd                                        # active
stratis pool list | grep apppool                                    # pool exists
stratis blockdev list apppool | grep sdb                            # sdb in pool
stratis blockdev list apppool | grep sdc                            # sdc in pool
stratis pool list | grep apppool | grep "19\|20\|18"               # ~20 GiB
stratis filesystem list apppool | grep fs_application               # FS exists
stratis filesystem list apppool | grep fs_database                  # FS exists
stratis filesystem list apppool | grep fs_database_snap             # snapshot exists
mount | grep /application                                           # mounted
mount | grep /database                                              # mounted
mount | grep /mnt/db_snap                                           # snapshot mounted
grep stratisd /etc/fstab | grep application                        # fstab entry
grep stratisd /etc/fstab | grep database                           # fstab entry
grep "0  0$" /etc/fstab | grep stratisd | wc -l                   # pass=0 entries
cat /application/app.conf | grep "APP_ENV=production"              # file content
cat /database/db.conf | grep "DB_ENGINE=postgresql"                # file content
```

---

### Common Mistakes in This Challenge

1. **Starting without enabling stratisd** → daemon stops after reboot
2. **Not wiping `/dev/sdb` or `/dev/sdc`** → pool create fails silently or with error
3. **Wrong fstab option** — missing `x-systemd.requires=stratisd.service`
4. **Pass value = 2** → should be `0` for Stratis; breaks boot behavior
5. **Using `/dev/stratis/apppool/fs_application` in fstab** instead of UUID
6. **Not running `systemctl daemon-reload`** after editing fstab
7. **Mounting snapshot read-only with `-o ro`** when task doesn't specify
   (minor, but know when required)
8. **Forgetting `stratis pool add-data`** still needs a clean device
9. **Checking `df` size and panicking at 1 TiB** — this is correct
10. **Confusing pool destroy order** — must destroy filesystems before pool

---

# ═══════════════════════════════════════════════════════════
# SELF-ASSESSMENT QUIZ — CHAPTER 3
# ═══════════════════════════════════════════════════════════

**Q1:** What two packages must be installed to use Stratis?

**Q2:** Before running any `stratis` command, what service must be running?

**Q3:** Write the complete fstab entry for a Stratis filesystem with
UUID `abc123`, mounted at `/data`.

**Q4:** Why does `df -h /data` show 1 TiB for a Stratis filesystem
on a 10GB pool?

**Q5:** What command do you use to check the REAL physical usage of a Stratis pool?

**Q6:** `stratis pool create pool1 /dev/sdb` fails. What is the most likely
cause and how do you fix it?

**Q7:** How do you add a second disk `/dev/sdc` to an existing pool `pool1`?

**Q8:** How do Stratis snapshots differ from LVM snapshots?

**Q9:** What happens to Stratis filesystems at boot if the fstab entry
does not include `x-systemd.requires=stratisd.service`?

**Q10:** List the 4-step Stratis setup workflow (commands only).

---

## Quiz Answers

**A1:** `stratisd` and `stratis-cli`

**A2:** `stratisd.service` — start with `systemctl enable --now stratisd`

**A3:**
```
UUID=abc123  /data  xfs  defaults,x-systemd.requires=stratisd.service  0  0
```

**A4:** Stratis always creates thin-provisioned filesystems. Each filesystem
receives a 1 TiB virtual allocation. Data only consumes physical space as it is
written. The real limit is the pool's physical size (10GB).

**A5:** `stratis pool list` — the "Total Physical" column shows real size
and "in use" shows actual physical consumption.

**A6:** `/dev/sdb` has existing filesystem signatures or metadata.
Fix: `wipefs -a /dev/sdb`, then retry `stratis pool create`.

**A7:**
```bash
wipefs -a /dev/sdc
stratis pool add-data pool1 /dev/sdc
```

**A8:** Stratis snapshots are **read-write** independent filesystems sharing
original data via thin provisioning. LVM snapshots are read-only by default
and use a Copy-on-Write mechanism that requires dedicated snapshot space.

**A9:** The filesystem will fail to mount at boot because systemd tries to
mount it before `stratisd` has started. The option tells systemd to start
stratisd first. Without it, you'll see mount errors in `journalctl -b`.

**A10:**
```bash
systemctl enable --now stratisd           # (prerequisite)
stratis pool create POOL /dev/sdb         # 1. Pool
stratis filesystem create POOL NAME      # 2. Filesystem
mkdir -p /mnt && mount /dev/stratis/POOL/NAME /mnt  # 3. Mount
# Add UUID to fstab with x-systemd.requires  # 4. Config
```

---

# ═══════════════════════════════════════════════════════════
# END OF CHAPTER 3 — STRATIS
# ═══════════════════════════════════════════════════════════

**Chapter Status:** ✅ COMPLETE
**Previous:** Chapter 2 — LVM
**Next:** Chapter 4 — Network Management

---

*Reply "Chapter 4" to begin the Network Management chapter.*
*Reply "STORAGE REVIEW" for a combined Partitioning + LVM + Stratis drill.*

