# 🗄️ LVM Phase 1 — Foundations

> **Learning Path:** Linux Fundamentals → Disk Management → LVM → Production Storage Engineering  
> **Level:** Beginner to Intermediate  
> **Goal:** Understand LVM from the ground up — what problem it solves, how its three-layer architecture works internally, and how to build a complete LVM setup on Google Cloud Platform from scratch.

---

## 📚 Table of Contents

1. [What Problem Does LVM Solve](#what-problem-does-lvm-solve)
2. [The LVM Architecture — Three Layers](#the-lvm-architecture--three-layers)
3. [Core Concept 1 — Physical Volume (PV)](#core-concept-1--physical-volume-pv)
4. [Core Concept 2 — Volume Group (VG)](#core-concept-2--volume-group-vg)
5. [Core Concept 3 — Logical Volume (LV)](#core-concept-3--logical-volume-lv)
6. [Core Concept 4 — Extents (PE and LE)](#core-concept-4--extents-pe-and-le)
7. [Core Concept 5 — LVM Metadata](#core-concept-5--lvm-metadata)
8. [Traditional Partitions vs LVM](#traditional-partitions-vs-lvm)
9. [Complete Command Reference](#complete-command-reference)
10. [Hands-On Lab — GCP Setup](#hands-on-lab--gcp-setup)
11. [The Two-Step Resize Rule](#the-two-step-resize-rule)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
13. [Interview Questions](#interview-questions)
14. [Common Beginner Mistakes](#common-beginner-mistakes)
15. [What Differentiates a Senior Engineer](#what-differentiates-a-senior-engineer)

---

## What Problem Does LVM Solve

Traditional disk partitions have one fatal limitation: **their boundaries are fixed forever.**

```
TRADITIONAL PARTITION PROBLEM:

Server disk (500GB) divided into partitions:
┌──────────────┬──────────────┬──────────────┬──────────────┐
│   /boot      │      /       │   /var/log   │    /home     │
│    1 GB      │    50 GB     │    20 GB     │   429 GB     │
└──────────────┴──────────────┴──────────────┴──────────────┘

6 months later:
  /var/log → 100% FULL 😱   (logs grew massively)
  /home    → Only 40% used   (200 GB sitting empty, doing nothing)

Can you move free space from /home to /var/log?
→ NO. The partition line is drawn in stone. You are stuck.
```

**LVM eliminates this problem entirely** by introducing a virtualization layer between physical disks and filesystems. Storage becomes a flexible pool — you add to it, carve from it, and resize it while the system runs.

| Capability                             | Traditional Partitions | LVM                           |
| -------------------------------------- | ---------------------- | ----------------------------- |
| Resize a volume online                 | Impossible             | `lvextend -r` — zero downtime |
| Span multiple physical disks           | Impossible             | Native feature                |
| Add a new disk to existing mount       | Impossible             | `vgextend` + `lvextend`       |
| Take consistent point-in-time snapshot | Not available          | Built-in                      |
| Move data off a failing disk           | Requires full downtime | `pvmove` — online             |
| Recover from metadata corruption       | No metadata to recover | VG metadata on every PV       |

---

## The LVM Architecture — Three Layers

LVM works in three distinct layers. Always read this diagram **bottom to top** — that is the order you build it.

```
╔══════════════════════════════════════════════════════════════════╗
║                      LAYER 3: FILESYSTEM                        ║
║              ext4 / xfs — what your OS and apps see            ║
║                                                                  ║
║       /app/data (ext4)    /var/log (ext4)    /backup (xfs)     ║
╠══════════════════════════════════════════════════════════════════╣
║                   LAYER 2: LOGICAL VOLUMES (LV)                 ║
║                                                                  ║
║         lv_appdata (15GB)    lv_logs (10GB)    lv_backup (5GB) ║
║                                                                  ║
║    What you FORMAT with mkfs and MOUNT to folders               ║
║    Behaves exactly like a traditional partition to the OS       ║
╠══════════════════════════════════════════════════════════════════╣
║                    LAYER 2: VOLUME GROUP (VG)                   ║
║                                                                  ║
║                       vg_production                             ║
║                           30 GB                                  ║
║                                                                  ║
║         One unified storage pool — the heart of LVM's power    ║
╠══════════════════════════════════════════════════════════════════╣
║                   LAYER 1: PHYSICAL VOLUMES (PV)                ║
║                                                                  ║
║      /dev/sdb (10GB)    /dev/sdc (10GB)    /dev/sdd (10GB)     ║
║                                                                  ║
║             Real disks registered with LVM                      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Build Order (Always Bottom to Top)

```
Step 1: pvcreate  → Register real disks as Physical Volumes
Step 2: vgcreate  → Pool all PVs together into a Volume Group
Step 3: lvcreate  → Carve Logical Volumes from the pool
Step 4: mkfs      → Format each LV with a filesystem (ext4/xfs)
Step 5: mount     → Attach each LV to a folder
Step 6: fstab     → Make mounts persist across reboots
```

---

## Core Concept 1 — Physical Volume (PV)

### What it is

A Physical Volume is a **real disk or partition that has been registered with LVM** by writing LVM metadata to it.

### What pvcreate Actually Does

```bash
sudo pvcreate /dev/sdb
```

Internally, LVM writes a metadata header to the **first sectors** of `/dev/sdb`:

```
BEFORE pvcreate:
┌─────────────────────────────────────────────┐
│  /dev/sdb — 10GB                            │
│  Raw disk. No structure. LVM unaware.       │
└─────────────────────────────────────────────┘

AFTER pvcreate /dev/sdb:
┌─────────────────────────────────────────────┐
│  /dev/sdb — 10GB                            │
│  ┌────────────────────────────────────┐     │
│  │  LVM METADATA HEADER               │     │
│  │  UUID:    abc123-def456-789...     │     │
│  │  Size:    10 GB                    │     │
│  │  PE Size: 4 MB                     │     │
│  │  VG Name: (none yet)               │     │
│  │  Total PE: 2559                    │     │
│  └────────────────────────────────────┘     │
│  Remaining space: available for LVM data    │
└─────────────────────────────────────────────┘
```

### Key pvdisplay Fields to Understand

```bash
sudo pvdisplay /dev/sdb
```

```
PV Name               /dev/sdb
VG Name                              ← Empty until assigned to a VG
PV Size               10.00 GiB
PE Size               4.00 MiB       ← Fundamental unit of LVM storage
Total PE              2559            ← 10GB ÷ 4MB = 2559 extents
Free PE               2559            ← All free (no LVs allocated yet)
Allocated PE          0
PV UUID               abc123-...
```

---

## Core Concept 2 — Volume Group (VG)

### What it is

A Volume Group is a **single unified storage pool** built from one or more Physical Volumes. Once created, you no longer think about individual disks — you only see one big pool of storage.

```
THREE SEPARATE PVs:
┌──────────┐    ┌──────────┐    ┌──────────┐
│ /dev/sdb │    │ /dev/sdc │    │ /dev/sdd │
│  10 GB   │    │  10 GB   │    │  10 GB   │
└──────────┘    └──────────┘    └──────────┘
      │                │                │
      └────────────────┼────────────────┘
                       │  vgcreate
                       ▼
       ┌───────────────────────────────────┐
       │          vg_production            │
       │             30 GB pool            │
       │  All PEs from all 3 disks pooled  │
       └───────────────────────────────────┘
```

### Growing the Pool — Adding a New Disk

```bash
# 6 months later: pool is almost full
# Solution: add a new GCP disk in seconds

# On GCP (outside VM):
gcloud compute disks create new-disk --size=10GB --zone=us-central1-a
gcloud compute instances attach-disk lvm-lab --disk=new-disk --zone=us-central1-a

# Inside the VM:
sudo pvcreate /dev/sde
sudo vgextend vg_production /dev/sde

# Pool grew from 30GB to 40GB. Zero downtime. Zero app impact.
```

### Key vgdisplay Fields to Understand

```bash
sudo vgdisplay vg_production
```

```
VG Name               vg_production
VG Size               29.98 GiB      ← Slightly under 30GB (metadata overhead)
PE Size               4.00 MiB
Total PE              7677
Alloc PE / Size       3840 / 15.00 GiB  ← PEs assigned to LVs
Free  PE / Size       3837 / 14.99 GiB  ← PEs still available
```

> **Production Monitoring Rule:** Alert when `Free PE` drops below 20% of `Total PE`. Never let a VG fill up completely — you lose the ability to extend without first adding a new disk.

---

## Core Concept 3 — Logical Volume (LV)

### What it is

A Logical Volume is a **usable slice carved from the Volume Group**. To the operating system, it looks and behaves exactly like a traditional partition. The difference: it can be resized, snapshotted, and span multiple physical disks.

```
FROM THE 30GB VOLUME GROUP:

sudo lvcreate -L 15G -n lv_appdata vg_production
sudo lvcreate -L 10G -n lv_logs    vg_production
sudo lvcreate -l 100%FREE -n lv_backup vg_production

┌─────────────────────────────────────────────────────────────┐
│                   vg_production (30GB)                      │
├────────────────────┬───────────────────┬────────────────────┤
│   lv_appdata       │    lv_logs        │    lv_backup       │
│     15 GB          │     10 GB         │     ~5 GB          │
│                    │                   │                    │
│  mkfs.ext4         │  mkfs.ext4        │  mkfs.xfs          │
│  mount /app/data   │  mount /var/log   │  mount /backup     │
└────────────────────┴───────────────────┴────────────────────┘
```

### LV Device Path

LVs are accessible at two equivalent paths:

```
/dev/vg_production/lv_appdata        ← Symlink (friendly name)
/dev/mapper/vg_production-lv_appdata ← Real device-mapper path

Both refer to the same LV. Both work with mkfs, mount, resize2fs, etc.
```

---

## Core Concept 4 — Extents (PE and LE)

Extents are the **fundamental unit of storage allocation** in LVM. Understanding extents separates engineers who know commands from engineers who understand the system.

### Physical Extents (PE)

When a VG is created, all storage is divided into equal-sized chunks called Physical Extents:

```
DEFAULT PE SIZE = 4 MB

vg_production (30GB) divided into PEs:
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐  ...
│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│PE│
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘  ...
 4M  4M  4M  4M  ...

Total PEs = 30,720 MB ÷ 4 MB = 7,680 PEs
```

### Logical Extents (LE)

When an LV is created, LVM assigns PEs from the free pool to that LV. Assigned PEs are called Logical Extents (LE):

```
lvcreate -L 15G -n lv_appdata vg_production

LVM assigns 3,840 PEs (15GB ÷ 4MB) to lv_appdata
These PEs are now called Logical Extents of lv_appdata

Free pool: 7,680 - 3,840 = 3,840 PEs remaining
```

### Why Extends Enable Zero-Downtime Resize

```
lvextend -L +3G /dev/vg_production/lv_appdata

What LVM actually does:
1. Find 768 free PEs (3GB ÷ 4MB) in the VG free pool
2. Mark them as assigned to lv_appdata
3. Update VG metadata on all PVs
4. Kernel updates the block device size

No data moves. No filesystem operations yet.
The block device is now 18GB.
Run resize2fs to tell ext4 about the new space.
```

### PE Size Guidelines

| Storage Size | Recommended PE Size | Command                            |
| ------------ | ------------------- | ---------------------------------- |
| Under 1 TB   | 4 MB (default)      | `vgcreate vg_name /dev/sdb`        |
| 1 TB – 10 TB | 16 MB               | `vgcreate -s 16M vg_name /dev/sdb` |
| Over 10 TB   | 32 MB or 64 MB      | `vgcreate -s 32M vg_name /dev/sdb` |

> **Warning:** PE size cannot be changed after VG creation. Choose carefully upfront.

---

## Core Concept 5 — LVM Metadata

### Where LVM's Memory Lives

LVM stores its entire configuration — PV list, VG layout, LV allocations, PE maps — as **metadata written on EVERY PV in the VG.**

```
vg_production = /dev/sdb + /dev/sdc + /dev/sdd

Each PV stores a COMPLETE COPY of VG metadata:
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  /dev/sdb    │   │  /dev/sdc    │   │  /dev/sdd    │
│ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
│ │METADATA  │ │   │ │METADATA  │ │   │ │METADATA  │ │
│ │ copy 1   │ │   │ │ copy 2   │ │   │ │ copy 3   │ │
│ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │
│  data...     │   │  data...     │   │  data...     │
└──────────────┘   └──────────────┘   └──────────────┘
```

This redundancy means if one PV loses its header, LVM reads metadata from the surviving PVs and can recover automatically.

### Backing Up and Restoring Metadata

```bash
# ALWAYS run this before any risky LVM operation
sudo vgcfgbackup vg_production
# Saves to: /etc/lvm/backup/vg_production

# View the backup (plain text — readable!)
cat /etc/lvm/backup/vg_production

# Restore metadata from backup (recovery scenario)
sudo vgcfgrestore vg_production
```

> **Senior Engineer Habit:** Run `vgcfgbackup` before every LVM change. It takes one second and can save hours of recovery work.

---

## Traditional Partitions vs LVM

```
SCENARIO: /var/log is full. You must add 10GB. Zero downtime required.

TRADITIONAL PARTITIONS:
  Step 1:  Stop all services writing to /var/log       (downtime starts)
  Step 2:  Unmount /var/log
  Step 3:  Backup all log data
  Step 4:  Delete /var/log partition with fdisk
  Step 5:  Create new larger partition
           (ONLY POSSIBLE if adjacent unallocated space exists!)
  Step 6:  Format with mkfs.ext4
  Step 7:  Restore log data from backup
  Step 8:  Remount and restart services               (downtime ends)

  Time: 2-4 hours minimum
  Risk: Data loss if backup/restore fails
  Downtime: GUARANTEED
  Cross-disk expansion: IMPOSSIBLE

LVM:
  # If VG has free space:
  sudo lvextend -L +10G -r /dev/vg_production/lv_logs

  # If VG is also full, add a new disk first:
  sudo pvcreate /dev/sde
  sudo vgextend vg_production /dev/sde
  sudo lvextend -L +10G -r /dev/vg_production/lv_logs

  Time: 30 seconds
  Risk: Zero
  Downtime: ZERO — filesystem stays mounted throughout
  Cross-disk expansion: WORKS NATIVELY
```

---

## Complete Command Reference

### PV Commands

```bash
# Register disk(s) with LVM
sudo pvcreate /dev/sdb
sudo pvcreate /dev/sdb /dev/sdc /dev/sdd     # Multiple at once

# Display information
sudo pvdisplay                                 # Detailed — all PVs
sudo pvdisplay /dev/sdb                        # Detailed — one PV
sudo pvdisplay --maps /dev/sdb                 # Show PE allocation map
sudo pvs                                       # Short summary table
sudo pvscan                                    # Scan for LVM signatures

# Remove LVM header (un-register from LVM)
# Only if PV is not part of any VG
sudo pvremove /dev/sdb
```

### VG Commands

```bash
# Create a Volume Group
sudo vgcreate vg_production /dev/sdb                    # One PV
sudo vgcreate vg_production /dev/sdb /dev/sdc /dev/sdd  # Multiple PVs
sudo vgcreate -s 32M vg_large /dev/sdb                  # Custom PE size

# Expand the pool (add new disk)
sudo vgextend vg_production /dev/sde

# Shrink the pool (remove a disk — must be empty)
sudo vgreduce vg_production /dev/sdb

# Display information
sudo vgdisplay vg_production                  # Detailed
sudo vgs                                      # Short summary table

# Activate / deactivate
sudo vgchange -ay vg_production               # Activate all LVs
sudo vgchange -an vg_production               # Deactivate all LVs

# Metadata backup and restore
sudo vgcfgbackup vg_production                # Backup to /etc/lvm/backup/
sudo vgcfgrestore vg_production               # Restore from backup

# Rename
sudo vgrename vg_production vg_prod
```

### LV Commands

```bash
# Create a Logical Volume
sudo lvcreate -L 15G   -n lv_appdata vg_production   # By size (absolute)
sudo lvcreate -l 100%FREE -n lv_app  vg_production   # Use all free space
sudo lvcreate -l 50%VG    -n lv_app  vg_production   # Use 50% of VG total
sudo lvcreate -l 3840     -n lv_app  vg_production   # By number of extents

# Display information
sudo lvdisplay /dev/vg_production/lv_appdata          # Detailed
sudo lvs                                              # Short summary table

# Extend (grow) a Logical Volume
sudo lvextend -L +10G /dev/vg_production/lv_appdata  # Add 10GB more
sudo lvextend -L 25G  /dev/vg_production/lv_appdata  # Set total to 25GB
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata  # Extend + resize FS

# Reduce (shrink) — DANGEROUS, see warning below
sudo lvreduce -L -5G /dev/vg_production/lv_appdata

# Rename
sudo lvrename vg_production lv_appdata lv_application

# Remove (DELETE — destroys all data)
sudo lvremove /dev/vg_production/lv_appdata
```

### Filesystem Resize Commands

```bash
# Resize ext4 filesystem (after lvextend)
# Can be done LIVE while mounted
sudo resize2fs /dev/vg_production/lv_appdata

# Resize XFS filesystem (after lvextend)
# XFS must be MOUNTED to resize
# XFS can ONLY grow — NEVER shrink
sudo xfs_growfs /app/data

# RECOMMENDED: Use -r flag on lvextend to do both at once
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata
# -r automatically calls resize2fs or xfs_growfs as appropriate
```

---

## The Two-Step Resize Rule

This is the most common source of confusion with LVM. Memorize this:

```
LVM operates at the BLOCK DEVICE layer.
The FILESYSTEM sits ON TOP of the block device.
They are TWO SEPARATE LAYERS with TWO SEPARATE sizes.

STEP 1: Grow the block device
sudo lvextend -L +10G /dev/vg_production/lv_appdata

  LVM layer:  18 GB  ✅  (block device is now bigger)
  FS layer:   15 GB  ❌  (filesystem still thinks it's 15GB)
  df -h shows: 15GB      ← NEW SPACE IS NOT USABLE YET

STEP 2: Grow the filesystem
sudo resize2fs /dev/vg_production/lv_appdata

  LVM layer:  18 GB  ✅
  FS layer:   18 GB  ✅  (filesystem now knows about all 18GB)
  df -h shows: 18GB  ✅  ← NEW SPACE IS NOW USABLE

SHORTCUT: Use the -r flag — does BOTH in one command:
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata
```

### Shrinking — Extra Steps Required

```
SHRINKING is the REVERSE and more DANGEROUS:
You MUST shrink the filesystem FIRST, then the LV.

# WRONG ORDER (causes data corruption):
sudo lvreduce -L -5G /dev/vg_production/lv_appdata  ← CORRUPTS DATA ❌
sudo resize2fs /dev/vg_production/lv_appdata

# CORRECT ORDER:
# Step 1: Unmount (required for ext4 shrink)
sudo umount /app/data

# Step 2: Check filesystem integrity first
sudo e2fsck -f /dev/vg_production/lv_appdata

# Step 3: Shrink filesystem FIRST
sudo resize2fs /dev/vg_production/lv_appdata 10G   # Shrink FS to 10GB

# Step 4: Shrink LV AFTER filesystem
sudo lvreduce -L 10G /dev/vg_production/lv_appdata

# Step 5: Remount
sudo mount /dev/vg_production/lv_appdata /app/data

# IMPORTANT: XFS CANNOT BE SHRUNK. EVER.
# If you need to shrink an XFS LV: backup → reformat → restore.
```

---

## Hands-On Lab — GCP Setup

### Prerequisites

- Google Cloud account with billing enabled
- `gcloud` CLI installed and authenticated
- A GCP project with Compute Engine API enabled

### Step 1: Create the VM

```bash
gcloud compute instances create lvm-lab \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard
```

### Step 2: Create and Attach Three Persistent Disks

```bash
# Create three 10GB disks (your future Physical Volumes)
for i in 1 2 3; do
  gcloud compute disks create lvm-disk-$i \
    --size=10GB --zone=us-central1-a --type=pd-standard
done

# Attach all three to the VM
for i in 1 2 3; do
  gcloud compute instances attach-disk lvm-lab \
    --disk=lvm-disk-$i --zone=us-central1-a
done
```

### Step 3: SSH and Verify

```bash
gcloud compute ssh lvm-lab --zone=us-central1-a

# Inside VM — verify all disks visible
lsblk
# sdb, sdc, sdd should appear as 10G unpartitioned disks
```

### Step 4: Install LVM Tools

```bash
sudo apt-get update && sudo apt-get install -y lvm2
```

### Step 5: Create Physical Volumes

```bash
sudo pvcreate /dev/sdb /dev/sdc /dev/sdd
sudo pvs       # Verify all three PVs created
```

### Step 6: Create Volume Group

```bash
sudo vgcreate vg_production /dev/sdb /dev/sdc /dev/sdd
sudo vgs       # Verify 30GB pool created
```

### Step 7: Create Logical Volumes

```bash
sudo lvcreate -L 15G        -n lv_appdata vg_production
sudo lvcreate -L 10G        -n lv_logs    vg_production
sudo lvcreate -l 100%FREE   -n lv_backup  vg_production
sudo lvs       # Verify all three LVs created
```

### Step 8: Format and Mount

```bash
# Format
sudo mkfs.ext4 /dev/vg_production/lv_appdata
sudo mkfs.ext4 /dev/vg_production/lv_logs
sudo mkfs.ext4 /dev/vg_production/lv_backup

# Create mount points
sudo mkdir -p /app/data /var/log/app /backup

# Mount
sudo mount /dev/vg_production/lv_appdata /app/data
sudo mount /dev/vg_production/lv_logs    /var/log/app
sudo mount /dev/vg_production/lv_backup  /backup

# Verify
df -hT
```

### Step 9: Make Mounts Persistent

```bash
# Get UUIDs
sudo blkid /dev/vg_production/lv_appdata
sudo blkid /dev/vg_production/lv_logs
sudo blkid /dev/vg_production/lv_backup

# Add to fstab (replace UUIDs with your actual values)
sudo tee -a /etc/fstab << 'EOF'
UUID=YOUR-UUID-1  /app/data     ext4  defaults  0  2
UUID=YOUR-UUID-2  /var/log/app  ext4  defaults  0  2
UUID=YOUR-UUID-3  /backup       ext4  defaults  0  2
EOF

# Test fstab without rebooting
sudo umount /app/data /var/log/app /backup
sudo mount -a
df -h    # Verify all three mounted correctly
```

### Step 10: Live Extension Demo

```bash
# Write test data to simulate a full filesystem
sudo dd if=/dev/zero of=/app/data/testfile bs=1M count=2000
df -h /app/data

# Extend lv_appdata by 3GB — WHILE MOUNTED AND IN USE
sudo lvextend -L +3G -r /dev/vg_production/lv_appdata

# Verify — no unmount, no service restart, no downtime
df -h /app/data   # Shows 18GB now ✅
sudo lvs          # lv_appdata shows 18.00g ✅
```

### Step 11: Snapshot Before Moving On

```bash
# Exit the VM
exit

# Take a GCP snapshot — your safety net for Phase 2
gcloud compute disks snapshot lvm-lab \
  --snapshot-names=phase1-complete \
  --zone=us-central1-a

# Verify snapshot
gcloud compute snapshots list
```

---

## Quick Reference Cheat Sheet

```bash
# ── INSPECT EVERYTHING ───────────────────────────────────────
lsblk                                   # Block device tree
sudo pvs                                # PV summary
sudo vgs                                # VG summary
sudo lvs                                # LV summary
sudo pvdisplay                          # PV detailed info
sudo vgdisplay vg_production            # VG detailed info
sudo lvdisplay /dev/vg_production/lv_appdata  # LV detailed info
sudo pvdisplay --maps /dev/sdb          # Show PE allocation map
df -hT                                  # Disk usage (mounted FS)

# ── CREATE LVM STACK (bottom to top) ─────────────────────────
sudo pvcreate /dev/sdb /dev/sdc /dev/sdd
sudo vgcreate vg_production /dev/sdb /dev/sdc /dev/sdd
sudo lvcreate -L 15G -n lv_appdata vg_production
sudo mkfs.ext4 /dev/vg_production/lv_appdata
sudo mkdir -p /app/data
sudo mount /dev/vg_production/lv_appdata /app/data

# ── EXTEND LVM STACK ──────────────────────────────────────────
# Add new disk to VG pool:
sudo pvcreate /dev/sde
sudo vgextend vg_production /dev/sde

# Grow an LV and filesystem:
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata  # By +10GB
sudo lvextend -L 25G  -r /dev/vg_production/lv_appdata  # To 25GB total
sudo lvextend -l +100%FREE -r /dev/vg_production/lv_appdata  # Use all free

# ── SHRINK (ext4 only — XFS cannot shrink) ────────────────────
sudo umount /app/data
sudo e2fsck -f /dev/vg_production/lv_appdata
sudo resize2fs /dev/vg_production/lv_appdata 10G  # Shrink FS first
sudo lvreduce -L 10G /dev/vg_production/lv_appdata  # Then shrink LV
sudo mount /dev/vg_production/lv_appdata /app/data

# ── METADATA BACKUP ───────────────────────────────────────────
sudo vgcfgbackup vg_production            # Backup VG metadata
sudo vgcfgrestore vg_production           # Restore VG metadata
cat /etc/lvm/backup/vg_production         # Read the backup file

# ── GCP DISK OPERATIONS ───────────────────────────────────────
# Create disk:
gcloud compute disks create lvm-disk-1 \
  --size=10GB --zone=us-central1-a --type=pd-standard

# Attach disk to VM:
gcloud compute instances attach-disk lvm-lab \
  --disk=lvm-disk-1 --zone=us-central1-a

# Take snapshot:
gcloud compute disks snapshot lvm-lab \
  --snapshot-names=my-snapshot --zone=us-central1-a
```

---

## Interview Questions

**Beginner Level:**

- What problem does LVM solve that traditional partitions cannot?
- Explain the three layers of LVM in simple terms.
- What is a Physical Extent and what is the default PE size?
- What is the difference between `pvs`, `vgs`, and `lvs`?
- What does the `-r` flag do when used with `lvextend`?

**Intermediate Level:**

- Your `/app/data` filesystem is 100% full at 3 AM. Walk through every command to fix it with zero downtime.
- What happens internally when you run `lvextend`? Describe the PE allocation process step by step.
- Why does LVM store metadata on every PV in the VG? What does this enable?
- What is the difference between `lvextend -L +10G` and `lvextend -L 10G`?
- You ran `lvextend -L +5G` but `df -h` still shows the old size. Why and how do you fix it?
- Why can XFS not be shrunk? What must you do if you need to reduce an XFS LV?

**Senior Level:**

- A PV in your production VG goes offline due to hardware failure. The VG is in an inconsistent state. Walk through your full recovery procedure.
- What are the performance tradeoffs between a large and small PE size? When would you increase it?
- Explain how VG metadata redundancy works and how you would recover from VG metadata corruption using `vgcfgrestore`.
- When would you use LVM striping across multiple PVs? What are the risks vs benefits?
- Your company is moving from bare-metal servers with LVM to GCP Persistent Disks. What LVM concepts directly map to GCP storage abstractions?

---

## Common Beginner Mistakes

**1. Running `lvextend` without `-r` and wondering why space did not increase**
`lvextend` grows the block device (LVM layer). The filesystem layer must also be resized with `resize2fs` (ext4) or `xfs_growfs` (XFS). Use `lvextend -r` to do both automatically.

**2. Trying to shrink an XFS filesystem**
XFS supports growing only. It cannot shrink. If you need to reduce an XFS LV, you must: backup data → unmount → `lvreduce` → reformat with `mkfs.xfs` → restore data.

**3. Shrinking the LV before shrinking the filesystem**
If you run `lvreduce` before `resize2fs`, the filesystem extends beyond the new LV boundary and becomes corrupted. Always: resize FS first → then resize LV.

**4. Using device paths instead of UUIDs in `/etc/fstab`**
`/dev/vg_production/lv_appdata` works but UUIDs are safer. Always use `sudo blkid` to get UUIDs and use those in fstab.

**5. Not backing up VG metadata before risky operations**
`sudo vgcfgbackup vg_production` takes one second. Skip it and a mistake can make your entire VG unrecoverable.

**6. Creating VG with default PE size for very large storage**
With 4MB PE size on a 10TB disk, you get 2.6 million PEs. Metadata becomes large and operations slow. Use `vgcreate -s 32M` for large storage environments.

**7. Forgetting to test fstab with `mount -a` before rebooting**
A typo in `/etc/fstab` can prevent the server from booting. Always run `sudo mount -a` after editing fstab. If errors appear, fix them before rebooting.

**8. Confusing `-L` and `-l` flags on `lvcreate`**
`-L` takes a size: `-L 10G`. `-l` takes extents or percentage: `-l 100%FREE`, `-l 2560`. These are different flags with different argument formats.

---

## What Differentiates a Senior Engineer

```
Junior Engineer:
  → Knows the commands
  → Runs them when instructed
  → Verifies with df -h

Mid-level Engineer:
  → Understands the two-layer resize rule
  → Uses -r flag automatically
  → Checks pvs, vgs, lvs before and after every change
  → Knows the difference between -L and -l flags

Senior Engineer:
  → Thinks in layers: always asks "what happens if this disk
    disappears right now?"
  → Runs vgcfgbackup before EVERY LVM operation
  → Takes GCP snapshots before any destructive change
  → Uses pvdisplay --maps to see exact PE allocation
  → Monitors VG free PE as a key production metric
    (alert at 80% full — never let it hit 100%)
  → Scripts ALL LVM operations with pre/post validation
  → Peer-reviews LVM changes before production execution
  → Knows how to recover from VG metadata corruption
  → Understands PE size implications for large storage
  → Plans LV sizes conservatively with room to grow
```

---

## Summary

```
LVM        → Logical Volume Manager — storage virtualization layer
PV         → Physical Volume — real disk registered with LVM
             Command: pvcreate /dev/sdb
VG         → Volume Group — unified storage pool from multiple PVs
             Command: vgcreate vg_name /dev/sdb /dev/sdc
LV         → Logical Volume — usable slice of the VG pool
             Command: lvcreate -L 15G -n lv_name vg_name
PE         → Physical Extent — fundamental 4MB unit of LVM storage
LE         → Logical Extent — PE assigned to an LV
Metadata   → Stored on EVERY PV in the VG — built-in redundancy
Resize     → TWO steps: lvextend (block device) + resize2fs (filesystem)
             OR one step with: lvextend -r (recommended)
XFS        → Can only grow. Can NEVER shrink.
ext4       → Can grow AND shrink (unmount required to shrink)
vgcfgbackup→ Always run before risky LVM operations
-r flag    → The most important flag: resizes LV and filesystem together
```

---
