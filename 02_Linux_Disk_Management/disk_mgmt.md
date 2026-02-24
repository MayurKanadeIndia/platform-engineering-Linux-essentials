# 💽 Disk Management

> **Learning Path:** Linux Fundamentals → Infrastructure Engineering → Cloud Engineering  
> **Level:** Beginner to Intermediate  
> **Goal:** Understand how Linux manages disks — from dividing them into partitions, to formatting with filesystems, to mounting them so your applications can actually use them.

---

## 📚 Table of Contents

1. [What We Will Learn](#what-we-will-learn)
2. [The Big Picture](#the-big-picture)
3. [Partitions](#partitions)
4. [MBR — Master Boot Record](#mbr--master-boot-record)
5. [GPT — GUID Partition Table](#gpt--guid-partition-table)
6. [MBR vs GPT Comparison](#mbr-vs-gpt-comparison)
7. [Device Naming in Linux](#device-naming-in-linux)
8. [fdisk — Creating Partitions](#fdisk--creating-partitions)
9. [Filesystems](#filesystems)
10. [Creating Filesystems with mkfs](#creating-filesystems-with-mkfs)
11. [Mount Points](#mount-points)
12. [Mounting and Unmounting](#mounting-and-unmounting)
13. [Persistent Mounts with /etc/fstab](#persistent-mounts-with-etcfstab)
14. [SWAP Space](#swap-space)
15. [UUIDs and Labels](#uuids-and-labels)
16. [Complete Workflow](#complete-workflow)
17. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
18. [Interview Questions](#interview-questions)
19. [Common Beginner Mistakes](#common-beginner-mistakes)

---

## What We Will Learn

- What partitions are and why we use them
- The difference between MBR and GPT partition tables
- How Linux names disks and partitions (`/dev/sda`, `/dev/sdb1`, etc.)
- How to create and manage partitions with `fdisk`
- What filesystems are and how to create them with `mkfs`
- How to mount and unmount disks to folders
- How to make mounts permanent using `/etc/fstab`
- How to set up SWAP space
- What UUIDs are and why they matter

---

## The Big Picture

Here is the complete journey from a raw disk to usable storage:

```
RAW DISK (just delivered, empty)
         │
         │ Step 1: fdisk
         │ Divide disk into partitions
         ▼
PARTITIONS (/dev/sdb1, /dev/sdb2, /dev/sdb3)
         │
         │ Step 2: mkfs
         │ Build a filesystem on each partition
         ▼
FORMATTED PARTITIONS (ext4, xfs, swap)
         │
         │ Step 3: mount
         │ Attach each partition to a folder
         ▼
MOUNTED FILESYSTEMS (/home, /opt, /var/log)
         │
         │ Step 4: /etc/fstab
         │ Make mounts automatic on every reboot
         ▼
PERSISTENT STORAGE — Ready for production ✅
```

---

## Partitions

### What is a Partition?

A partition is a **logically separated section of a disk**. Think of it like dividing a notebook into sections — each section is independent of the others.

```
ONE PHYSICAL DISK (500 GB)
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  Partition 1 │  Partition 2 │  Partition 3 │  Partition 4 │
│   /boot      │      /       │    /home     │     SWAP     │
│   (1 GB)     │   (50 GB)    │   (400 GB)   │   (8 GB)     │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### Why Use Partitions?

**1. Isolate the OS from user data**

```
WITHOUT partitions:
┌──────────────────────────────────────┐
│  OS + Apps + User Files — All mixed  │
│  User fills disk with movies...      │
│  Disk 100% FULL                      │
│  OS can't write logs → CRASH 💥      │
└──────────────────────────────────────┘

WITH partitions:
┌─────────────────┬────────────────────┐
│  OS Partition   │   User Partition   │
│  (always safe)  │   FULL 😱          │
│  OS still runs  │   Only user data   │
│  normally ✅    │   is affected      │
└─────────────────┴────────────────────┘
```

**2. Protect system stability from application outages**

If a website's log partition fills up completely, the web server stops writing logs but the **operating system and other services keep running normally**. Without partitioning, a full disk crashes the entire system.

**3. Separate concerns for security and management**

- Mount `/home` with `noexec` option → users cannot execute programs from their home directory
- Mount `/tmp` with `noexec` and `nosuid` → prevents certain attack vectors
- Easier to backup specific partitions independently

### Typical Partition Scheme

**Server:**

```
/boot    →  1 GB      (boot files — kernel, initrd, GRUB)
/        →  20–50 GB  (OS and system files)
/var     →  50+ GB    (logs, databases, spool)
/home    →  remaining (user files)
swap     →  2–8 GB    (RAM overflow)
```

**Desktop:**

```
/boot    →  1 GB      (boot files)
/        →  50 GB     (OS, apps)
/home    →  remaining (user files, documents)
swap     →  equal to RAM size
```

---

## MBR — Master Boot Record

### What is MBR?

MBR stands for **Master Boot Record**. It is a small section of data stored at the **very beginning of a disk** (first 512 bytes). It contains:

1. A small boot program (used by BIOS to start the OS)
2. The **partition table** (describes how the disk is divided)

```
DISK /dev/sda:
┌──────────────────────────────────────────────────────────┐
│  First 512 bytes: MBR                                    │
│  ┌──────────────────────┬─────────────────────────────┐  │
│  │  Boot code (446 B)   │  Partition Table (64 B)     │  │
│  │  Loaded by BIOS      │  Max 4 entries              │  │
│  │  to start bootloader │  Each = one primary part.   │  │
│  └──────────────────────┴─────────────────────────────┘  │
│                                                           │
│  Rest of disk: your actual partition data                 │
└──────────────────────────────────────────────────────────┘
```

### MBR Limitations

| Limitation         | Detail                                         |
| ------------------ | ---------------------------------------------- |
| Maximum disk size  | **2 TB** — cannot address larger disks         |
| Maximum partitions | **4 primary partitions** only                  |
| Backup of table    | **None** — if MBR corrupts, disk is unreadable |
| Partition ID size  | 32-bit only                                    |

### The Extended Partition Workaround

Since MBR allows only 4 primary partitions, a workaround was invented: use one primary partition as a **container** for unlimited logical partitions inside it.

```
MBR Partition Layout with Extended Partition:

┌──────────┬──────────┬──────────┬─────────────────────────────────┐
│Primary 1 │Primary 2 │Primary 3 │  Extended Partition (container) │
│  /boot   │    /     │   /var   │  ┌──────┬──────┬──────┬──────┐ │
│          │          │          │  │  L1  │  L2  │  L3  │  L4  │ │
│          │          │          │  │/home │ /opt │ /tmp │/data │ │
│          │          │          │  └──────┴──────┴──────┴──────┘ │
└──────────┴──────────┴──────────┴─────────────────────────────────┘

Primary partitions: 3  (real partitions)
Extended partition: 1  (just a container, not directly usable)
Logical partitions: 4  (inside the container — unlimited)
```

> **Note:** This is a workaround/hack. GPT eliminates the need for this entirely.

---

## GPT — GUID Partition Table

### What is GPT?

GPT stands for **GUID Partition Table**. It is the modern replacement for MBR and part of the **UEFI standard**.

GUID stands for **Globally Unique Identifier** — every partition gets a unique ID that no other partition in the world shares, like a fingerprint.

### GPT Improvements Over MBR

| Feature                     | GPT (New)                                          |
| --------------------------- | -------------------------------------------------- |
| Maximum disk size           | **9.4 ZB** (9.4 Zettabytes — 9.4 trillion GB)      |
| Maximum partitions          | **128 partitions**                                 |
| Partition table backup      | **Yes** — stored at both beginning AND end of disk |
| Partition IDs               | 128-bit GUIDs (unique worldwide)                   |
| No primary/extended concept | All partitions are equal                           |
| Error detection             | CRC32 checksums on partition table                 |

### GPT Disk Layout

```
GPT DISK /dev/sda:
┌──────────────────────────────────────────────────────────────┐
│  Protective MBR (for backward compatibility, first 512 B)    │
├──────────────────────────────────────────────────────────────┤
│  GPT Header (primary copy) + Partition Table (128 entries)   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Partition 1   Partition 2   Partition 3  ...  Partition 128 │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  GPT Header (backup copy) + Partition Table (backup copy)    │
└──────────────────────────────────────────────────────────────┘
                                                      ↑
                                    Backup at end of disk!
                                    If beginning corrupts,
                                    recover from backup.
```

### What is UEFI?

**UEFI** = Unified Extensible Firmware Interface

It is the **modern replacement for BIOS**. Just as GPT replaces MBR for partition tables, UEFI replaces BIOS for firmware.

```
Old system:  BIOS  + MBR  (1980s technology)
New system:  UEFI  + GPT  (modern standard)
```

---

## MBR vs GPT Comparison

| Feature                | MBR                               | GPT                   |
| ---------------------- | --------------------------------- | --------------------- |
| Maximum disk size      | 2 TB                              | 9.4 ZB                |
| Maximum partitions     | 4 primary (+ logical in extended) | 128                   |
| Partition table backup | No                                | Yes (at end of disk)  |
| Error detection        | No                                | Yes (CRC32 checksums) |
| Works with BIOS        | Yes                               | Sometimes             |
| Works with UEFI        | Yes (legacy mode)                 | Yes (native)          |
| Used today             | Fading out (legacy hardware)      | Modern standard       |
| Supports large disks   | No (>2TB fails)                   | Yes                   |

> **Rule of thumb for 2024+:** Always use **GPT** unless you have a specific reason to use MBR (e.g., very old hardware or legacy OS compatibility requirements).

---

## Device Naming in Linux

### Everything is a File

Linux treats **everything as a file** — including your disks. All hardware devices are represented as files in the `/dev` directory.

### Disk Naming Convention

```
/dev/sda    ← First disk  (sd = SCSI/SATA disk, a = first)
/dev/sdb    ← Second disk (b = second)
/dev/sdc    ← Third disk  (c = third)
/dev/sdd    ← Fourth disk (d = fourth)
...and so on alphabetically
```

### Partition Naming Convention

```
/dev/sda    ← The entire first disk
/dev/sda1   ← Partition 1 on the first disk
/dev/sda2   ← Partition 2 on the first disk
/dev/sda3   ← Partition 3 on the first disk

/dev/sdb    ← The entire second disk
/dev/sdb1   ← Partition 1 on the second disk
/dev/sdb2   ← Partition 2 on the second disk
```

### Real Example (Google Cloud VM)

```bash
lsblk
```

```
NAME    SIZE  TYPE  MOUNTPOINT
sda      20G  disk             ← Boot disk
└─sda1   20G  part  /          ← Root partition, mounted at /
sdb      10G  disk             ← Attached persistent disk 1
sdc      10G  disk             ← Attached persistent disk 2
sdd      10G  disk             ← Attached persistent disk 3
```

---

## fdisk — Creating Partitions

### What is fdisk?

`fdisk` is the standard Linux tool for **creating, modifying, and deleting partitions**. It supports both MBR and GPT (modern versions).

**Alternatives to fdisk:**

| Tool     | Best For                              |
| -------- | ------------------------------------- |
| `fdisk`  | General use — MBR and GPT             |
| `gdisk`  | GPT-specific work                     |
| `parted` | Scripting, large disks, more features |

### Basic fdisk Commands

```bash
# List all disks and their partitions (read-only, safe to run)
sudo fdisk -l

# List partitions on a specific disk
sudo fdisk -l /dev/sdb

# Start interactive partitioning on a disk
sudo fdisk /dev/sdb
```

### fdisk Interactive Menu

When you run `sudo fdisk /dev/sdb`, you enter an interactive session:

```
Welcome to fdisk (util-linux 2.36).
Changes will remain in memory only, until you decide to write them.

Command (m for help): _
```

**Key commands inside fdisk:**

| Command | Action                           |
| ------- | -------------------------------- |
| `m`     | Show help menu                   |
| `p`     | Print current partition table    |
| `n`     | Create a new partition           |
| `d`     | Delete a partition               |
| `t`     | Change partition type            |
| `g`     | Create new GPT partition table   |
| `o`     | Create new MBR partition table   |
| `w`     | **Write (save) changes to disk** |
| `q`     | Quit WITHOUT saving (safe exit)  |

> **Critical:** Nothing is permanent until you press `w`. If you make a mistake, press `q` to quit safely without any changes being written.

### Creating a New Partition (Step by Step)

```bash
sudo fdisk /dev/sdb

# Inside fdisk:
Command: g          # Create a new GPT partition table (fresh start)
Command: n          # New partition
Partition number: 1 # First partition
First sector: (press Enter for default — start of disk)
Last sector: +5G    # Make it 5GB  (or +500M for 500MB, or just Enter for rest of disk)
Command: p          # Print to verify it looks right
Command: w          # Write/save the changes
```

### Checking Results

```bash
# See the new partition
lsblk /dev/sdb

# Detailed view
sudo fdisk -l /dev/sdb
```

---

## Filesystems

### What is a Filesystem?

A filesystem is the **organizational system that manages how files are stored and retrieved on a partition**. Without a filesystem, a partition is just raw unorganized space — data exists but cannot be found or used.

```
WITHOUT a filesystem:
┌─────────────────────────────────────────────┐
│  101001110100101010010101010011...          │
│  (raw bits — you can write data             │
│   but you can never find it again)          │
└─────────────────────────────────────────────┘

WITH a filesystem (ext4):
┌─────────────────────────────────────────────┐
│  /home/john/photos/birthday.jpg   ← findable│
│  /home/john/documents/resume.pdf  ← findable│
│  /var/log/syslog                  ← findable│
│  (organized, indexed, usable)               │
└─────────────────────────────────────────────┘
```

### Linux Filesystem Types

| Filesystem   | Full Name       | Key Characteristics                     | Best For                       |
| ------------ | --------------- | --------------------------------------- | ------------------------------ |
| **ext2**     | 2nd Extended FS | No journaling — risky on power loss     | Old systems only               |
| **ext3**     | 3rd Extended FS | Added journaling over ext2              | Older, still common            |
| **ext4**     | 4th Extended FS | Faster, larger files, better journaling | Most common — default choice   |
| **XFS**      | Extended FS     | Very high performance, no shrink        | Large files, databases         |
| **Btrfs**    | B-Tree FS       | Built-in snapshots, checksums           | Modern systems, still maturing |
| **ZFS**      | Zettabyte FS    | Enterprise-grade, self-healing          | Enterprise storage             |
| **ReiserFS** | Reiser FS       | Good for small files                    | Legacy, being replaced         |

### What is Journaling?

A journal is a **pre-write log** that records what the filesystem is about to do before doing it.

```
WITHOUT journaling (ext2):
  System writing file → Power cut mid-write
  Result: Partial write, corrupted data, inconsistent filesystem
  Fix: Must run full disk scan (fsck) — takes hours on large disks

WITH journaling (ext3, ext4):
  System writes intention to journal FIRST → Then writes actual data
  Power cut → On restart, reads journal → Completes or rolls back cleanly
  Result: No corruption, fast recovery
```

> **Rule of thumb:** Always use **ext4** for general use, **XFS** for high-performance database or large file workloads.

---

## Creating Filesystems with mkfs

### What is mkfs?

`mkfs` (**Make FileSystem**) is the command that **formats a partition** — it builds the filesystem structure (catalog, indexes, metadata) on the raw partition, making it ready to store files.

```bash
# General syntax
mkfs -t TYPE DEVICE

# These two commands do EXACTLY the same thing:
mkfs -t ext4 /dev/sdb1
mkfs.ext4 /dev/sdb1           # Shortcut — directly calls the ext4 formatter

# Other filesystem examples
mkfs.ext3 /dev/sdb2           # Create ext3 filesystem
mkfs.xfs  /dev/sdb3           # Create XFS filesystem
mkfs.ext4 /dev/sdc1           # Create ext4 on another disk's partition
```

### Verifying the Filesystem

```bash
# See what filesystem is on each partition
lsblk -f

# Detailed filesystem information
sudo blkid /dev/sdb1
```

> **Warning:** Running `mkfs` on a partition **destroys all existing data** on it. This is irreversible. Always confirm you're targeting the correct device.

---

## Mount Points

### What is a Mount Point?

A **mount point** is a directory that serves as the **access point (door) into a disk or partition**.

```
BEFORE mounting:
/dev/sdb1  ← disk partition exists, has data, but NO door to enter

AFTER running: mount /dev/sdb1 /app/data
/app/data  ← this folder IS NOW the door into /dev/sdb1
            Open /app/data → you're reading from /dev/sdb1
            Save to /app/data → data goes onto /dev/sdb1
```

### The Root Mount Point /

The `/` (root) mount point is the most important mount point in Linux. **Everything in Linux lives under `/`**.

```
/                ← Root of everything (always a mount point)
├── boot         ← Boot files
├── home         ← User home directories
├── var          ← Variable data (logs, mail, databases)
│   └── log      ← Log files
├── etc          ← Configuration files
├── bin          ← Essential system commands
├── dev          ← Hardware device files (/dev/sda, etc.)
├── tmp          ← Temporary files
└── opt          ← Optional/third-party software
```

### Mounting Over Existing Data

An important and often confusing behavior: mounting a partition onto a folder **hides** any existing data in that folder.

```bash
# Example:
mkdir /home/sarah           # Create a folder

mount /dev/sdb2 /home       # Mount a disk onto /home

# Result: /home/sarah is now hidden (covered by the new mount)
# It still EXISTS but you cannot see or access it
# It's like placing a rug over something on the floor

umount /home                # Remove the mount
# Now /home/sarah is visible again — it was never deleted!
```

---

## Mounting and Unmounting

### The mount Command

```bash
# Mount syntax
mount DEVICE MOUNTPOINT

# Examples
sudo mount /dev/sdb1 /app/data          # Mount partition to folder
sudo mount /dev/vg_data/lv_app /app     # Mount an LVM logical volume

# View all currently mounted filesystems
mount

# Better: view only real disk filesystems (not virtual ones)
df -h
```

### The df Command

```bash
df -h
```

```
Filesystem               Size  Used Avail Use%  Mounted on
/dev/sda1                 20G  4.2G   15G  23%  /
/dev/sdb1                 10G  1.1G  8.9G  11%  /app/data
/dev/sdc1                 10G  512M  9.5G   5%  /var/log
tmpfs                    1.0G     0  1.0G   0%  /dev/shm  (virtual, RAM-based)
```

> `mount` shows everything including RAM-based virtual filesystems (used internally by Linux). `df -h` shows only real storage filesystems — much cleaner for disk monitoring.

### The umount Command

```bash
# Unmount syntax (note: it's umount, NOT unmount)
umount MOUNTPOINT
# or
umount DEVICE

# Examples
sudo umount /app/data        # By mount point folder
sudo umount /dev/sdb1        # By device name
```

> **Common error:** `target is busy` — means a process has a file open inside that mounted partition. Find and stop the process before unmounting.

```bash
# Find what is using a mount point
lsof /app/data              # List Open Files at that mount point
fuser -m /app/data          # Show processes using the mount point
```

### Important Rule: Manual Mounts Do NOT Persist

```
You run: sudo mount /dev/sdb1 /app/data
The mount works perfectly.

Server reboots...

/app/data is now empty. The mount is gone.
Linux forgot about it.

Solution: Add it to /etc/fstab
```

---

## Persistent Mounts with /etc/fstab

### What is /etc/fstab?

`/etc/fstab` (**File System Table**) is a configuration file that lists **all filesystems to be automatically mounted at boot**.

```bash
# View your fstab
cat /etc/fstab
```

### The 6 Fields of /etc/fstab

Every line in `/etc/fstab` has exactly 6 fields separated by spaces or tabs:

```
┌──────────────────┬─────────────┬──────┬──────────┬──────┬──────┐
│  FIELD 1         │  FIELD 2    │  3   │  FIELD 4 │  5   │  6   │
│  Device          │ Mount Point │ Type │ Options  │ Dump │ fsck │
├──────────────────┼─────────────┼──────┼──────────┼──────┼──────┤
│ UUID=abc123...   │     /       │ xfs  │ defaults │  0   │  1   │
│ UUID=def456...   │    swap     │ swap │ defaults │  0   │  0   │
│ UUID=ghi789...   │  /app/data  │ ext4 │ defaults │  0   │  2   │
└──────────────────┴─────────────┴──────┴──────────┴──────┴──────┘
```

**Field 1 — Device:** Which disk/partition to mount

```bash
UUID=550e8400-e29b-41d4-a716-446655440000   # Recommended (by UUID)
/dev/sdb1                                    # Not recommended (name can change)
LABEL=appdata                                # Also works (by label)
```

**Field 2 — Mount Point:** Which folder to mount it to

```
/               Root filesystem
/home           Home directories
/app/data       Custom application data
swap            Special keyword for swap space (no folder needed)
```

**Field 3 — Filesystem Type:**

```
ext4    xfs    ext3    swap    tmpfs    nfs    vfat
```

**Field 4 — Mount Options:**

```
defaults    = Standard read/write mount (use this 99% of the time)
ro          = Read-only
noexec      = Cannot execute programs from this mount
nosuid      = Ignore setuid bits (security hardening)
noauto      = Do not mount at boot (mount manually only)
```

**Field 5 — Dump (0 or 1):**
Controls whether the old `dump` backup utility backs up this filesystem.

```
0  → Do not back up (almost always use 0 — dump is rarely used today)
1  → Back up
```

**Field 6 — fsck Order (0, 1, or 2):**
Controls filesystem check order at boot.

```
0  → Never check (use for swap and network filesystems)
1  → Check FIRST (use for / root partition ONLY)
2  → Check after root (use for all other partitions)
```

### Example /etc/fstab File

```bash
# /etc/fstab — filesystem table
# <device>                                <mount>     <type>  <options>  <dump>  <fsck>
UUID=1a2b3c4d-5e6f-7890-abcd-ef1234567890  /           xfs     defaults    0       1
UUID=2b3c4d5e-6f70-8901-bcde-f12345678901  swap        swap    defaults    0       0
UUID=3c4d5e6f-7081-9012-cdef-012345678902  /app/data   ext4    defaults    0       2
UUID=4d5e6f70-8192-0123-def0-123456789003  /var/log    ext4    defaults    0       2
UUID=5e6f7081-9203-1234-ef01-234567890104  /backup     ext4    defaults    0       2
```

### Testing fstab Without Rebooting

```bash
# Mount everything in fstab that isn't already mounted
sudo mount -a

# If no errors appear — your fstab is correct
# If errors appear — fix them before rebooting!
```

> **Always test with `mount -a` before rebooting. A mistake in fstab can prevent the server from booting.**

---

## SWAP Space

### What is SWAP?

SWAP is disk space used as **overflow when RAM is full**.

```
RAM is full (8GB used / 8GB total):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │ P8 │  ← All 8GB used
└────┴────┴────┴────┴────┴────┴────┴────┘

New process P9 needs memory:
P1 (least recently used) → moved to SWAP partition on disk
P9 fits in RAM now

┌────┬────┬────┬────┬────┬────┬────┬────┐
│ P9 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │ P8 │  ← P9 in RAM
└────┴────┴────┴────┴────┴────┴────┴────┘
SWAP: [P1]   ← P1 waiting on disk
```

SWAP is **much slower than RAM** (disk vs memory) but prevents out-of-memory crashes.

### Setting Up SWAP

```bash
# Step 1: Create a partition with fdisk (set type to Linux swap)
sudo fdisk /dev/sdb

# Step 2: Prepare the partition as swap (NOT mkfs)
sudo mkswap /dev/sdb2

# Step 3: Activate the swap partition
sudo swapon /dev/sdb2

# Step 4: Verify swap is active
swapon -s
# or
free -h
```

### Swap in /etc/fstab

```bash
# Add to /etc/fstab for automatic activation at boot:
UUID=xxxx-xxxx-xxxx   swap   swap   defaults   0   0
#                     ↑       ↑
#              mount point   type (both say "swap")
```

---

## UUIDs and Labels

### The Problem with Device Names

```
Today:
/dev/sda = your boot disk
/dev/sdb = your data disk

You add a new USB disk and reboot.
Linux detects disks in a different order.

After reboot:
/dev/sda = your boot disk (same)
/dev/sdb = the NEW USB disk  ← changed!
/dev/sdc = your data disk    ← shifted!

Your fstab still points /data to /dev/sdb...
→ It now mounts the USB disk as /data instead!
→ DISASTER 💥
```

### The Solution: UUIDs

A **UUID (Universally Unique Identifier)** is a permanent unique ID **written directly onto the disk** during formatting. It never changes regardless of what order disks are detected.

```
UUID = 550e8400-e29b-41d4-a716-446655440000

It looks complex but:
✅ Unique to this specific disk — no two disks share it
✅ Never changes on reboot or disk reassignment
✅ Stable even if you add/remove other disks
```

### Finding UUIDs

```bash
# See filesystem type, label AND UUID for all partitions
lsblk -f

# Output:
# NAME   FSTYPE  LABEL   UUID                                   MOUNTPOINT
# sda
# └─sda1 xfs             1234-5678-9abc-def0-123456789abc       /
# sdb
# └─sdb1 ext4    mydata  abcd-ef12-3456-7890-abcdef123456       /app/data

# See ONLY UUIDs (simpler output)
sudo blkid

# Output:
# /dev/sda1: UUID="1234-5678-9abc-def0-123456789abc" TYPE="xfs"
# /dev/sdb1: UUID="abcd-ef12-3456-7890-abcdef123456" TYPE="ext4" LABEL="mydata"

# Find UUID for a specific partition
sudo blkid /dev/sdb1
```

### Labels

A label is a **human-readable name** you can assign to a partition.

```bash
# Create a label when formatting
sudo mkfs.ext4 -L mydata /dev/sdb1

# Or add a label after formatting
sudo e2label /dev/sdb1 mydata          # For ext filesystems
sudo xfs_admin -L mydata /dev/sdc1     # For XFS
```

**In /etc/fstab, you can use UUID or LABEL:**

```bash
# Using UUID (RECOMMENDED — always unique):
UUID=abcd-ef12-3456-7890-abcdef123456  /app/data  ext4  defaults  0  2

# Using LABEL (easier to read, but labels can accidentally duplicate):
LABEL=mydata  /app/data  ext4  defaults  0  2

# Using device name (NOT recommended — can change on reboot):
/dev/sdb1  /app/data  ext4  defaults  0  2
```

> **Best practice:** Always use **UUID** in `/etc/fstab`. It is the safest and most reliable identifier.

---

## Complete Workflow

Here is the **full end-to-end process** of adding a new disk and making it usable:

```bash
# ── STEP 1: Verify disk is visible to OS ──────────────────
lsblk
# Confirm new disk appears (e.g., /dev/sdb)

# ── STEP 2: Create partition with fdisk ───────────────────
sudo fdisk /dev/sdb
# Inside fdisk: g (GPT table) → n (new partition) → w (save)

# Verify partition was created
lsblk /dev/sdb

# ── STEP 3: Create filesystem ─────────────────────────────
sudo mkfs.ext4 /dev/sdb1

# ── STEP 4: Create mount point directory ──────────────────
sudo mkdir -p /app/data

# ── STEP 5: Get the UUID ──────────────────────────────────
sudo blkid /dev/sdb1
# Copy the UUID value

# ── STEP 6: Mount temporarily to test ─────────────────────
sudo mount /dev/sdb1 /app/data

# Verify it works
df -h /app/data

# ── STEP 7: Add to /etc/fstab for permanent mounting ──────
sudo nano /etc/fstab

# Add this line (use YOUR actual UUID):
# UUID=your-uuid-here  /app/data  ext4  defaults  0  2

# ── STEP 8: Test fstab entry ──────────────────────────────
sudo umount /app/data          # Unmount first
sudo mount -a                  # Mount everything in fstab

# Verify it mounted correctly
df -h /app/data

# ── STEP 9: Verify it survives reboot ─────────────────────
sudo reboot
# After reboot: df -h /app/data should show the disk mounted
```

---

## Quick Reference Cheat Sheet

```bash
# ── DISK INSPECTION ───────────────────────────────────────
lsblk                          # List all disks and partitions
lsblk -f                       # Include filesystem type, UUID, label
sudo fdisk -l                  # Detailed partition info for all disks
sudo fdisk -l /dev/sdb         # Info for one specific disk
df -h                          # Disk usage (mounted filesystems only)
sudo blkid                     # Show UUIDs and filesystem types
sudo blkid /dev/sdb1           # UUID for a specific partition

# ── PARTITIONING ──────────────────────────────────────────
sudo fdisk /dev/sdb            # Start interactive partitioning
# Inside fdisk:
#   g  = create GPT table    n = new partition
#   d  = delete partition    p = print table
#   w  = write and exit      q = quit without saving

# ── FILESYSTEM CREATION ────────────────────────────────────
sudo mkfs.ext4 /dev/sdb1       # Create ext4 filesystem
sudo mkfs.xfs  /dev/sdb2       # Create XFS filesystem
sudo mkfs.ext4 -L mydata /dev/sdb1  # With a label

# ── MOUNTING ──────────────────────────────────────────────
sudo mount /dev/sdb1 /app/data       # Mount partition
sudo mount -a                         # Mount everything in fstab
mount                                 # Show all mounted filesystems
df -h                                 # Show disk usage + mounts

# ── UNMOUNTING ────────────────────────────────────────────
sudo umount /app/data                 # By mount point
sudo umount /dev/sdb1                 # By device name
lsof /app/data                        # Find what's using the mount

# ── SWAP ──────────────────────────────────────────────────
sudo mkswap /dev/sdb2                 # Prepare swap partition
sudo swapon /dev/sdb2                 # Activate swap
sudo swapoff /dev/sdb2                # Deactivate swap
swapon -s                             # Show active swap
free -h                               # Show RAM and swap usage

# ── LABELS ────────────────────────────────────────────────
sudo e2label /dev/sdb1 mydata         # Set label on ext4 partition
sudo xfs_admin -L mydata /dev/sdc1    # Set label on XFS partition
```

---

## Interview Questions

**Beginner Level:**

- What is a partition and why do we use them?
- What is the difference between MBR and GPT?
- What does `mkfs` do and when do you need to run it?
- What is a mount point and what does mounting mean?
- Why do manual mounts not survive a reboot?

**Intermediate Level:**

- You have a Linux server where the `/home` partition is full but `/` has lots of free space. Without LVM, what are your options?
- What is the difference between `mount` and `df`? When would you use each?
- Explain the 6 fields of `/etc/fstab`. What happens if you set the last field to 1 for two different partitions?
- Why should you always use UUID instead of `/dev/sdb1` in fstab?
- What is the purpose of SWAP? When would a production server need swap?

**Senior Level:**

- A new disk is attached to a running server. Walk me through every command you'd run to make it available for production use.
- A server fails to boot after you edited `/etc/fstab`. How do you fix it?
- What is the difference between ext4 and XFS in a production context? When would you choose each?
- How does the filesystem journal protect against data corruption?
- You run `lsblk` and see a disk but it has no partitions. What are the next steps?

---

## Common Beginner Mistakes

**1. Running mkfs on the wrong device**
`mkfs.ext4 /dev/sda` instead of `mkfs.ext4 /dev/sda1` will format your entire first disk, destroying the OS. Always double-check the device name before running mkfs.

**2. Using /dev/sdb1 in /etc/fstab instead of UUID**
Device names can change on reboot when you add or remove disks. UUID never changes. Always use UUID.

**3. Forgetting to test /etc/fstab with `mount -a` before rebooting**
A typo in fstab can prevent the server from booting. Always run `sudo mount -a` first — if there are errors, fix them before rebooting.

**4. Forgetting to create the mount point directory**
`mount /dev/sdb1 /app/data` will fail if `/app/data` does not exist yet. Always run `mkdir -p /app/data` first.

**5. Confusing mkfs with mount**
`mkfs` builds the filesystem structure on a partition (done once). `mount` connects it to a folder (done every reboot). These are two completely different operations.

**6. Choosing MBR for a new disk larger than 2TB**
MBR cannot address more than 2TB. Always use GPT for modern disks.

**7. Not understanding that mounting hides existing data in that folder**
If you mount a partition onto `/home` and users already have files in `/home`, those files become inaccessible (not deleted, just hidden) until you unmount.

---

## Summary

```
Disk          → Physical storage device (/dev/sda, /dev/sdb)
Partition     → A divided section of a disk (/dev/sda1, /dev/sda2)
MBR           → Old partition table: max 2TB, max 4 partitions
GPT           → Modern partition table: 9.4ZB, 128 partitions, backup copy
fdisk         → Tool to create/delete/modify partitions
Filesystem    → Organizational system for storing files (ext4, xfs)
ext4          → Most common Linux filesystem — default choice
XFS           → High-performance filesystem — cannot shrink
mkfs          → Format a partition with a filesystem (done once)
mount         → Attach a partition to a folder (done every boot)
umount        → Detach a partition from a folder
/etc/fstab    → Auto-mount configuration file (6 fields per entry)
UUID          → Permanent unique identifier — always use in fstab
SWAP          → Disk space used as RAM overflow
blkid         → Command to show UUIDs of all partitions
lsblk -f      → Show partition tree with filesystem and UUID info
```

---

![alt text](images/disk_mgmt_summary.png)
