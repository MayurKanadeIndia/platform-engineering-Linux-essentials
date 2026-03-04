# 🏭 LVM Phase 3 — Real-World Production Patterns

> **Learning Path:** Linux Fundamentals → LVM Foundations → LVM on GCP → Production LVM  
> **Level:** Advanced — Senior Infrastructure Engineer  
> **Goal:** Understand how real companies architect LVM at scale. Learn production patterns for databases, containers, and CI/CD. Build incident runbooks. Develop the mindset of a staff engineer managing storage in production.

---

## 📚 Table of Contents

1. [The Mental Shift](#the-mental-shift)
2. [How Big Tech Uses LVM](#how-big-tech-uses-lvm)
3. [LVM and Databases — The Critical Pattern](#lvm-and-databases--the-critical-pattern)
4. [LVM Thin Provisioning](#lvm-thin-provisioning)
5. [LVM Striping — Performance Engineering](#lvm-striping--performance-engineering)
6. [LVM Mirroring — Built-In Redundancy](#lvm-mirroring--built-in-redundancy)
7. [Production Incident Runbooks](#production-incident-runbooks)
8. [LVM in the Container World](#lvm-in-the-container-world)
9. [LVM Troubleshooting Guide](#lvm-troubleshooting-guide)
10. [The Production Mindset](#the-production-mindset)
11. [Storage Design Principles](#storage-design-principles)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
13. [Interview Questions](#interview-questions)
14. [Common Senior-Level Mistakes](#common-senior-level-mistakes)

---

## The Mental Shift

```
PHASE 1: "How does LVM work?"
  → Architecture and commands

PHASE 2: "How do I build and recover LVM on GCP?"
  → Hands-on construction, failure simulation, automation

PHASE 3: "How do real companies design storage at scale?"
  → Production patterns for databases, containers, CI/CD
  → Incident runbooks written before the incident happens
  → Performance engineering: striping, thin provisioning
  → The mindset of a staff engineer owning storage in production
```

---

## How Big Tech Uses LVM

### The Honest Reality

```
Storage architecture depends on WHERE the workload runs:

LAYER 4: Application (MySQL, Cassandra, PostgreSQL, Redis)
         │
LAYER 3: Storage Abstraction
         ├── Cloud-native: Kubernetes PVCs / CSI drivers
         └── Traditional: LVM directly
         │
LAYER 2: Block Storage
         ├── Cloud: GCP Persistent Disk / AWS EBS
         └── Bare-metal: NVMe / SAS drives
         │
LAYER 1: Physical Hardware
         Real servers in real data centers
```

**Where LVM is actively used in 2024-2026:**

| Environment                         | LVM Usage           | Why                                                        |
| ----------------------------------- | ------------------- | ---------------------------------------------------------- |
| Bare-metal data centers             | Essential           | No cloud abstraction. Physical drives need management.     |
| Cloud VMs (GCP/AWS)                 | Optional but common | Layer multiple cloud disks. Live resize without API calls. |
| Kubernetes bare-metal nodes         | Heavy use           | TopoLVM provisions LVs as PersistentVolumes.               |
| Kubernetes on cloud                 | Less common         | CSI drivers + cloud storage often sufficient.              |
| Database servers (all environments) | Universal           | LVM snapshots for consistent DB backups.                   |
| CI/CD build servers                 | Common              | Snapshot-based clean build environments.                   |

### Production Pattern 1: Database Server

```
┌────────────────────────────────────────────────────────────┐
│  DATABASE SERVER (MySQL / PostgreSQL)                      │
│                                                            │
│  /dev/sdb (SSD) ─┐                                        │
│  /dev/sdc (SSD) ─┼→ vg_mysql_data (SSD pool)             │
│                                                            │
│  /dev/sdd (HDD) ──→ vg_mysql_logs (HDD pool)             │
│                                                            │
│  LVs:                                                     │
│  lv_data   (xfs)  → /var/lib/mysql          (InnoDB)     │
│  lv_tmp    (xfs)  → /var/lib/mysql-tmp      (temp)       │
│  lv_binlogs(xfs)  → /var/lib/mysql-binlogs  (binlogs)    │
│  lv_backup (ext4) → /backup/mysql           (staging)    │
│                                                            │
│  WHY separate VGs?                                        │
│  Data disk full → binlogs still write → DB stays alive   │
│  Binlog disk full → data unaffected, just lose binlogs   │
│  Each concern independently extensible and snapshotable  │
└────────────────────────────────────────────────────────────┘
```

### Production Pattern 2: Kubernetes Worker Node

```
┌────────────────────────────────────────────────────────────┐
│  KUBERNETES WORKER NODE (bare-metal)                      │
│                                                            │
│  /dev/nvme0 (1TB) ─┐                                      │
│  /dev/nvme1 (1TB) ─┼→ vg_k8s (2TB pool)                  │
│                                                            │
│  lv_containers (1TB)  → /var/lib/docker    (images)      │
│  lv_kubelet   (200GB) → /var/lib/kubelet   (k8s state)   │
│  lv_logs      (300GB) → /var/log           (pod logs)    │
│  lv_etcd       (50GB) → /var/lib/etcd      (cluster DB)  │
│  lv_tmp       (450GB) → /tmp               (scratch)     │
│                                                            │
│  WHY LVM on Kubernetes nodes?                             │
│  Container images grow unpredictably                     │
│  Extend lv_containers without draining the node          │
│  Snapshot before OS upgrades                             │
│  etcd gets its own fast isolated storage                 │
└────────────────────────────────────────────────────────────┘
```

### Production Pattern 3: CI/CD Build Server

```
┌────────────────────────────────────────────────────────────┐
│  JENKINS / GITHUB ACTIONS RUNNER                          │
│                                                            │
│  vg_builds (500GB)                                        │
│  ├── lv_workspace  (100GB) → /builds/workspace            │
│  ├── lv_cache      (200GB) → /builds/cache (maven/npm)   │
│  ├── lv_artifacts  (150GB) → /builds/artifacts           │
│  └── lv_docker      (50GB) → /builds/docker-layers       │
│                                                            │
│  LVM snapshot per build:                                  │
│  → Clean build environment every run (snapshot rollback) │
│  → Parallel builds on snapshot clones                    │
│  → Failed build → reset to clean snapshot instantly      │
└────────────────────────────────────────────────────────────┘
```

---

## LVM and Databases — The Critical Pattern

### Why XFS for Databases

| Feature                | ext4                     | XFS                     |
| ---------------------- | ------------------------ | ----------------------- |
| Can shrink             | ✅ Yes                   | ❌ No                   |
| Allocation groups      | 1                        | Multiple (parallel I/O) |
| Large file performance | Good                     | Excellent               |
| Direct I/O (O_DIRECT)  | Good                     | Excellent               |
| Scale limit            | 50 TB                    | 8 EB                    |
| Best for               | General use, backup dirs | Databases, large files  |

> **Rule:** Use XFS for all database data directories. Use ext4 for backup directories where you may need to shrink.

### MySQL LVM Setup

```bash
# Create database-grade disks on GCP
gcloud compute disks create mysql-data-1 \
  --size=100GB --zone=us-central1-a --type=pd-ssd

gcloud compute disks create mysql-data-2 \
  --size=100GB --zone=us-central1-a --type=pd-ssd

gcloud compute disks create mysql-logs-1 \
  --size=50GB --zone=us-central1-a --type=pd-balanced

# Inside VM: Build separate VGs for data and logs
sudo pvcreate /dev/sdb /dev/sdc   # SSD → data VG
sudo pvcreate /dev/sdd             # Balanced → logs VG

sudo vgcreate vg_mysql_data /dev/sdb /dev/sdc
sudo vgcreate vg_mysql_logs /dev/sdd

# Backup metadata immediately
sudo vgcfgbackup vg_mysql_data
sudo vgcfgbackup vg_mysql_logs

# Create LVs
sudo lvcreate -L 150G -n lv_data    vg_mysql_data
sudo lvcreate -L 30G  -n lv_tmp     vg_mysql_data
sudo lvcreate -L 20G  -n lv_data_bk vg_mysql_data
sudo lvcreate -L 40G  -n lv_binlogs vg_mysql_logs

# Format: XFS for performance, ext4 for backup
sudo mkfs.xfs  -L mysql_data    /dev/vg_mysql_data/lv_data
sudo mkfs.xfs  -L mysql_tmp     /dev/vg_mysql_data/lv_tmp
sudo mkfs.ext4 -L mysql_backup  /dev/vg_mysql_data/lv_data_bk
sudo mkfs.xfs  -L mysql_binlogs /dev/vg_mysql_logs/lv_binlogs

# Mount with database-optimized options
# noatime: skip access-time updates (saves IOPS on every read)
sudo mount -o noatime,nodiratime \
  /dev/vg_mysql_data/lv_data    /var/lib/mysql

sudo mount -o noatime,nodiratime \
  /dev/vg_mysql_data/lv_tmp     /var/lib/mysql-tmp

sudo mount -o noatime,nodiratime \
  /dev/vg_mysql_logs/lv_binlogs /var/lib/mysql-binlogs

sudo mount /dev/vg_mysql_data/lv_data_bk /backup/mysql

sudo chown -R mysql:mysql /var/lib/mysql \
                           /var/lib/mysql-tmp \
                           /var/lib/mysql-binlogs
```

### Consistent MySQL Backup via LVM Snapshot

```bash
sudo tee /usr/local/bin/mysql-lvm-backup.sh << 'EOF'
#!/bin/bash
# MySQL Consistent Backup via LVM Snapshot
# Write-pause window: < 1 second
set -euo pipefail

BACKUP_DIR="/backup/mysql"
SNAP_SIZE="20G"
SNAP_NAME="lv_data_snap_$(date +%Y%m%d_%H%M%S)"
LOG="/var/log/mysql-backup.log"

echo "$(date): MySQL LVM backup started" >> "$LOG"

# FLUSH + LOCK + SNAPSHOT + UNLOCK in one atomic operation
# Tables are locked for the duration of lvcreate only (<1 second)
mysql -u backup_user -p"${MYSQL_BACKUP_PASSWORD}" << MYSQL_EOF
FLUSH TABLES WITH READ LOCK;
SYSTEM sudo lvcreate -s -L $SNAP_SIZE -n $SNAP_NAME /dev/vg_mysql_data/lv_data;
UNLOCK TABLES;
MYSQL_EOF

echo "$(date): Snapshot $SNAP_NAME created. Tables unlocked." >> "$LOG"

# Record binlog position for point-in-time recovery
mysql -u backup_user -p"${MYSQL_BACKUP_PASSWORD}" \
  -e "SHOW MASTER STATUS\G" \
  > "$BACKUP_DIR/binlog_pos_$(date +%Y%m%d).txt"

# Mount snapshot and backup (MySQL continues writing normally)
sudo mkdir -p /mnt/mysql_snap
sudo mount -o ro,noatime /dev/vg_mysql_data/$SNAP_NAME /mnt/mysql_snap

sudo tar -czf "$BACKUP_DIR/mysql-$(date +%Y%m%d-%H%M%S).tar.gz" \
  -C /mnt/mysql_snap .

# Cleanup
sudo umount /mnt/mysql_snap
sudo lvremove -f /dev/vg_mysql_data/$SNAP_NAME

# Upload to GCS for disaster recovery
gsutil cp "$BACKUP_DIR/mysql-$(date +%Y%m%d)*.tar.gz" \
  gs://YOUR-BACKUP-BUCKET/mysql/

echo "$(date): Backup completed and uploaded to GCS" >> "$LOG"
EOF

sudo chmod +x /usr/local/bin/mysql-lvm-backup.sh

# Schedule daily at 2 AM
(crontab -l 2>/dev/null; \
  echo "0 2 * * * /usr/local/bin/mysql-lvm-backup.sh") | crontab -
```

---

## LVM Thin Provisioning

### What It Is

**Thick provisioning (standard LVM):** Reserve 100GB immediately, even if only 5GB is written.

**Thin provisioning:** Volumes _claim_ space but only consume it when actually written. You can overcommit — create more virtual space than physical space.

```
THICK:  lvcreate -L 100G → 100GB immediately consumed from VG
THIN:   Create 30GB pool → provision 3 × 20G thin volumes (60GB virtual)
        Real usage: only what is actually written to disk
```

### When to Use

| Use Thin Provisioning         | Avoid Thin Provisioning              |
| ----------------------------- | ------------------------------------ |
| Development environments      | Production databases                 |
| Test/staging clusters         | High-write production workloads      |
| Docker storage pools          | Systems where full disk = data loss  |
| Lab and learning environments | Any system without active monitoring |

### Building a Thin Provisioned Setup

```bash
# Step 1: Create the thin pool (real physical storage)
sudo lvcreate -L 30G -T -n thin_pool vg_production

# Step 2: Create thin volumes larger than the pool
sudo lvcreate -V 20G -T -n lv_dev1 vg_production/thin_pool
sudo lvcreate -V 20G -T -n lv_dev2 vg_production/thin_pool
sudo lvcreate -V 20G -T -n lv_dev3 vg_production/thin_pool
# 3 × 20GB = 60GB virtual from a 30GB pool — intentional overcommit

# Verify — note Data% starts at 0%
sudo lvs

# Format and use like normal LVs
sudo mkfs.ext4 /dev/vg_production/lv_dev1
sudo mkdir -p /dev1
sudo mount /dev/vg_production/lv_dev1 /dev1

# CRITICAL: Monitor thin pool usage
# If thin pool reaches 100% → ALL thin volumes FREEZE
sudo lvs vg_production/thin_pool  # Watch Data%
```

### Auto-Extend Thin Pool

```bash
# Configure LVM to auto-extend the thin pool in /etc/lvm/lvm.conf
# thin_pool_autoextend_threshold = 70   (extend when 70% full)
# thin_pool_autoextend_percent = 20     (extend by 20%)

# Or extend manually when monitoring alerts:
sudo lvextend -L +10G vg_production/thin_pool
```

---

## LVM Striping — Performance Engineering

### What Striping Does

```
NORMAL LVM (linear):
  Write → goes to /dev/sdb first, then /dev/sdc when sdb fills
  Throughput = ONE disk speed

STRIPED LVM:
  Write is split across all disks simultaneously
  Throughput ≈ NUMBER_OF_DISKS × SINGLE_DISK_SPEED

  Writing a 512KB block with 128KB stripe across 4 disks:
  Disk 1: [128KB chunk 1]
  Disk 2: [128KB chunk 2]  ← all happening in parallel
  Disk 3: [128KB chunk 3]
  Disk 4: [128KB chunk 4]
```

### Creating a Striped LV

```bash
# -i = number of stripes (disks to stripe across)
# -I = stripe size in KB (128K is optimal for most DB workloads)

sudo lvcreate -L 30G \
  -n lv_db_striped \
  -i 3 \
  -I 128 \
  vg_production

# Verify stripe configuration
sudo lvdisplay /dev/vg_production/lv_db_striped | grep -E "Stripe|LV Size"

# Format and benchmark
sudo mkfs.xfs /dev/vg_production/lv_db_striped
sudo mkdir -p /var/lib/db-fast
sudo mount /dev/vg_production/lv_db_striped /var/lib/db-fast

# Write benchmark (direct I/O — bypasses cache for real numbers)
sudo dd if=/dev/zero of=/var/lib/db-fast/test bs=1M count=2000 \
  oflag=direct 2>&1 | grep -E "MB|GB"
```

### Striping on Cloud vs Bare Metal

```
ON GCP PERSISTENT DISKS:
  VM network bandwidth is the bottleneck, not individual disk speed.
  e2-standard-2: max ~800 MB/s total.
  Adding more pd-balanced disks does NOT exceed this limit.
  Striping benefit: marginal (reduces per-disk IOPS pressure).

ON BARE METAL NVMe:
  Each NVMe: ~7 GB/s sequential
  4 NVMe striped: ~28 GB/s
  MASSIVE performance gain.

USE STRIPING ON:
  ✅ Physical bare-metal servers with multiple NVMe/SAS drives
  ✅ GCP when per-disk IOPS (not bandwidth) is the bottleneck

SKIP STRIPING ON:
  ❌ Single disk environments
  ❌ When bandwidth cap dominates (most cloud VMs)
  ❌ When reliability is more important than performance
```

---

## LVM Mirroring — Built-In Redundancy

### What Mirroring Does

```
WITHOUT MIRRORING:
  lv_critical data → /dev/sdb only
  /dev/sdb fails → DATA LOST 💥

WITH LVM MIRRORING (-m 1):
  lv_critical data → /dev/sdb AND /dev/sdc simultaneously
  /dev/sdb fails → /dev/sdc has complete copy → NO DATA LOST ✅
  This is RAID-1 implemented entirely in software via LVM.
```

### Creating a Mirrored LV

```bash
# -m 1 = create 1 mirror (total: 2 copies = original + mirror)
sudo lvcreate -L 10G -m 1 -n lv_critical vg_production

# Check which PVs each copy landed on
sudo lvs -a -o lv_name,devices | grep critical
# lv_critical_mimage_0: /dev/sdb  ← copy 1
# lv_critical_mimage_1: /dev/sdc  ← copy 2
# lv_critical_mlog:     /dev/sdd  ← mirror log (sync state tracking)
```

### When to Use LVM Mirroring

| Scenario                               | Recommendation                                             |
| -------------------------------------- | ---------------------------------------------------------- |
| Bare-metal server with physical drives | Use LVM mirroring or hardware RAID                         |
| GCP Persistent Disk                    | **Skip** — GCP already replicates data 3x internally       |
| AWS EBS                                | **Skip** — EBS already replicates across Availability Zone |
| Any cloud block storage                | **Skip** — cloud providers handle redundancy               |
| Critical data on physical drives       | Use LVM mirroring                                          |

> **On GCP and AWS:** The cloud provider replicates your data transparently at the infrastructure level. You cannot see the individual drives. LVM mirroring adds cost and complexity without adding real redundancy on cloud.

---

## Production Incident Runbooks

A runbook is written **before** the incident happens so you execute calmly instead of improvising at 3 AM.

### Runbook 001 — VG Almost Full (80%+ Alert)

```bash
#!/bin/bash
# RUNBOOK-001: VG Almost Full | Severity: Warning | SLA: 4 hours

echo "=== RUNBOOK-001: VG Almost Full | $(date) ==="

# Step 1: Assess current state
sudo vgs
sudo lvs
df -hT | grep vg_production

# Step 2: Find the biggest consumers
sudo lvs --sort -lv_size | head -10
sudo du -sh /app/data/* 2>/dev/null | sort -rh | head -10

# Step 3: Quick cleanup — safe to delete
sudo find /var/log/app -name "*.gz" -mtime +30 -delete
sudo find /app/data/tmp -mtime +1 -delete 2>/dev/null
df -hT /app/data   # Check if cleanup freed enough space

# Step 4: If cleanup not enough — add new disk
# On LOCAL machine:
# gcloud compute disks create emergency-disk-$(date +%Y%m%d) \
#   --size=20GB --zone=us-central1-a --type=pd-balanced
# gcloud compute instances attach-disk lvm-production \
#   --disk=emergency-disk-$(date +%Y%m%d) --zone=us-central1-a

# Step 5: Expand VG and extend fullest LV
# sudo pvcreate /dev/sde
# sudo vgextend vg_production /dev/sde
# sudo lvextend -L +15G -r /dev/vg_production/lv_appdata

# Step 6: Verify and document
sudo vgs
df -hT | grep vg_production
echo "$(date): RUNBOOK-001 completed by $(whoami)" \
  >> /var/log/lvm-changes.log
```

### Runbook 002 — Disk Failure / Partial VG

```bash
#!/bin/bash
# RUNBOOK-002: Disk Failure | Severity: Critical | SLA: 1 hour

echo "=== RUNBOOK-002: Disk Failure | $(date) ==="

# Step 1: Identify what failed
sudo pvs 2>&1 | grep -E "WARNING|unknown|a-m"
sudo vgs 2>&1   # Look for 'p' in Attr = partial

# Step 2: Check which LVs are affected
sudo lvs -a -o lv_name,lv_attr,devices 2>&1

# Step 3: Check data accessibility
df -hT | grep vg_production
ls /app/data 2>&1 | head -5

# Step 4a: Disk still partially alive — evacuate now
# sudo pvdisplay --maps /dev/FAILING_DISK
# sudo pvmove /dev/FAILING_DISK          # Move data off
# sudo vgreduce vg_production /dev/FAILING_DISK
# sudo pvremove /dev/FAILING_DISK

# Step 4b: Disk completely gone — force remove
# sudo vgchange -ay --partial vg_production
# sudo vgreduce --removemissing --force vg_production

# Step 5: Restore from GCP snapshot
# On LOCAL machine:
# gcloud compute snapshots list --filter="name:lvm-data"
# gcloud compute disks create restored-disk \
#   --source-snapshot=SNAPSHOT_NAME --zone=us-central1-a
# gcloud compute instances attach-disk lvm-production \
#   --disk=restored-disk --zone=us-central1-a
# Inside VM:
# sudo pvscan && sudo vgchange -ay && sudo mount -a

# Step 6: Verify and document
sudo vgs
df -hT | grep vg_production
echo "$(date): RUNBOOK-002: Disk failure recovery by $(whoami)" \
  >> /var/log/lvm-changes.log
```

### Runbook 003 — Emergency Space Recovery (15-Minute SLA)

```bash
#!/bin/bash
# RUNBOOK-003: /LV is 100% full. App crashing. | SLA: 15 minutes

echo "=== RUNBOOK-003: Emergency Space | $(date) ==="
START=$(date +%s)

# Minutes 1-2: Immediate relief — safe deletes
sudo find /var/log/app -name "*.gz" -mtime +7 -delete 2>/dev/null
sudo find /app/data/tmp -mtime +1 -delete 2>/dev/null
df -hT /app/data

# Minutes 3-5: Check VG for free space
FREE=$(sudo vgs --noheadings --units g -o vg_free vg_production | tr -d ' g')
echo "VG free space: ${FREE}GB"

# Minutes 5-10: Extend from VG if space available
if (( $(echo "$FREE > 2" | bc -l) )); then
  echo "Extending lv_appdata by 10GB from existing VG free space"
  sudo lvextend -L +10G -r /dev/vg_production/lv_appdata
  df -hT /app/data
else
  echo "VG is full. Must add new GCP disk."
  echo "Run this on LOCAL machine NOW:"
  echo "gcloud compute disks create emergency-$(date +%Y%m%d%H%M) \\"
  echo "  --size=20GB --zone=us-central1-a --type=pd-balanced && \\"
  echo "gcloud compute instances attach-disk lvm-production \\"
  echo "  --disk=emergency-$(date +%Y%m%d%H%M) --zone=us-central1-a"
  echo "Then: sudo pvcreate /dev/sde && sudo vgextend vg_production /dev/sde"
  echo "Then: sudo lvextend -L +15G -r /dev/vg_production/lv_appdata"
fi

END=$(date +%s)
echo "Completed in $((END - START)) seconds"
echo "$(date): RUNBOOK-003 emergency recovery by $(whoami)" \
  >> /var/log/lvm-changes.log
```

---

## LVM in the Container World

### LVM at the Kubernetes Node Level

The modern pattern for Kubernetes on bare metal is **TopoLVM** — a CSI driver that provisions LVM Logical Volumes as Kubernetes PersistentVolumes.

```
HOW TopoLVM WORKS:

Node admin prepares:
  sudo vgcreate vg_k8s /dev/nvme0n1 /dev/nvme1n1

Developer submits a PVC:
  kind: PersistentVolumeClaim
  spec:
    storageClassName: topolvm-provisioner
    resources:
      requests:
        storage: 10Gi

TopoLVM receives the claim → runs:
  lvcreate -L 10G -n pvc-abc123 vg_k8s

Developer sees: A 10GB volume mounted in their pod
Admin sees: An LV named pvc-abc123 in vg_k8s

Benefits:
  ✅ Local NVMe performance (faster than network storage)
  ✅ LVM flexibility for ops team
  ✅ Standard Kubernetes API for developers
  ✅ LV created/deleted automatically with pod lifecycle
```

```bash
# Node admin perspective — check what TopoLVM created
sudo lvs | grep pvc-
# pvc-abc123  vg_k8s  10.00g  ← PostgreSQL pod's storage
# pvc-def456  vg_k8s   5.00g  ← Redis pod's storage
# pvc-ghi789  vg_k8s  50.00g  ← MySQL pod's storage
```

### Docker Storage Driver (Legacy Context)

```
Docker storage drivers:
  overlay2    → Default. Uses host FS. Simple. Modern choice.
  devicemapper → Uses LVM thin provisioning. More complex.

Current state (2024-2026):
  overlay2 is the standard.
  devicemapper (LVM-backed) used only in legacy environments.
  Kubernetes CSI drivers have replaced both for PV workloads.

LVM matters at the NODE level regardless:
  /var/lib/docker lives on an LV → extend without draining node
  /var/lib/kubelet lives on an LV → independent growth control
  /var/log lives on an LV → logs cannot kill the OS partition
```

---

## LVM Troubleshooting Guide

### Issue 1: df Shows Old Size After lvextend

```
SYMPTOM: lvextend ran successfully but df -h shows old size

ROOT CAUSE: lvextend grows the block device.
            The filesystem is a separate layer — it must be told.

FIX:
  sudo resize2fs /dev/vg_production/lv_appdata  # ext4
  sudo xfs_growfs /app/data                     # XFS (must be mounted)

PREVENT: Always use -r flag:
  sudo lvextend -L +10G -r /dev/vg_production/lv_appdata
```

### Issue 2: umount — Target Is Busy

```
SYMPTOM: sudo umount /app/data → "target is busy"

ROOT CAUSE: A process has an open file inside the mount.

FIX:
  sudo lsof /app/data      # Find the process
  sudo fuser -mv /app/data # Detailed process list
  sudo fuser -mk /app/data # Kill blocking processes (use carefully)
  cd / && sudo umount /app/data  # If it's just your terminal's cwd
```

### Issue 3: VG Not Found After Reboot

```
SYMPTOM: After reboot, vgs shows nothing or VG inactive.

ROOT CAUSE: LVM services didn't start, or fstab mount order issue.

FIX:
  sudo pvscan --cache            # Rescan all devices
  sudo vgchange -ay              # Activate all VGs
  sudo mount -a                  # Mount all fstab entries

PREVENT:
  sudo systemctl enable lvm2-monitor
  sudo systemctl enable lvm2-lvmetad
```

### Issue 4: Snapshot Invalidated During Backup

```
SYMPTOM: Snap% = 100.00. Snapshot shows as 'Invalid'. Backup corrupt.

ROOT CAUSE: Snapshot space exhausted before backup completed.
            Too many writes to original LV during the backup window.

FIX:
  sudo lvremove /dev/vg_production/lv_appdata_snap
  sudo lvcreate -s -L 15G -n lv_appdata_snap \
    /dev/vg_production/lv_appdata
  # Redo backup immediately

PREVENT:
  snapshot_size = write_rate_per_hour × backup_hours × 3
  Monitor snap% during backup: watch -n 30 "sudo lvs | grep snap"
```

### Issue 5: pvmove Very Slow or Stuck

```
SYMPTOM: pvmove running but barely progressing.

ROOT CAUSE: High I/O on the system. pvmove runs at low priority.

FIX:
  # Safely pause pvmove (no data loss)
  sudo lvconvert --abort /dev/vg_production/pvmove0

  # Resume when system is less busy (2 AM)
  sudo pvmove /dev/sdb

  # Run in background with progress monitoring
  sudo pvmove -b /dev/sdb
  sudo pvmove --poll /dev/sdb
```

### Issue 6: VG Metadata Inconsistent

```
SYMPTOM: "WARNING: VG vg_production metadata is inconsistent"

ROOT CAUSE: Metadata on different PVs got out of sync.
            Usually happens after crash during metadata write.

FIX:
  # Check if backup exists (should always exist)
  cat /etc/lvm/backup/vg_production

  # Restore from backup
  sudo vgcfgrestore vg_production

  # If no backup — repair
  sudo vgck vg_production
  sudo vgscan --mknodes

PREVENT:
  sudo vgcfgbackup vg_production  # Run before EVERY LVM operation
```

### Issue 7: Extending LV Fails — "Insufficient free space"

```
SYMPTOM: lvextend -L +10G fails with "Insufficient free space"

ROOT CAUSE: VG does not have 10GB of free PEs available.

FIX:
  # Check available free space
  sudo vgs vg_production

  # If VG is full: add a new PV first
  # (Attach new GCP disk → pvcreate → vgextend)
  sudo pvcreate /dev/sde
  sudo vgextend vg_production /dev/sde
  sudo lvextend -L +10G -r /dev/vg_production/lv_appdata

  # Extend to use ALL remaining free space instead:
  sudo lvextend -l +100%FREE -r /dev/vg_production/lv_appdata
```

---

## The Production Mindset

### Questions a Staff Engineer Asks Before ANY LVM Change

```
1. "What is the blast radius if this goes wrong?"
   → Which applications use this LV?
   → What breaks if the LV is briefly unavailable?
   → Can I limit the impact to one LV?

2. "Do I have a tested rollback plan?"
   → GCP snapshot taken in the last hour?
   → vgcfgbackup done?
   → Exact commands ready to undo the change?

3. "Have I done this EXACT operation in the lab first?"
   → Never test on production what you haven't done in lab.
   → Our GCP lab exists exactly for this purpose.

4. "Is there a better maintenance window?"
   → Off-peak traffic period?
   → Application team notified?
   → On-call engineer standing by?

5. "Has a second engineer reviewed this change?"
   → Four-eyes principle for production storage changes.
   → Write the commands in a change ticket before executing.

6. "What does success look like?"
   → Before: screenshot of pvs, vgs, lvs, df -hT
   → After: same screenshot, compare both
   → Document in change log with ticket reference
```

### The Change Audit Log

```bash
# Add this function to /etc/profile.d/lvm-audit.sh
# Every engineer's terminal will have access to it

sudo tee /etc/profile.d/lvm-audit.sh << 'EOF'
lvm_log() {
  local ACTION="$1"
  local DETAILS="${2:-no details provided}"
  echo "$(date '+%Y-%m-%d %H:%M:%S') | $(whoami) | $ACTION | $DETAILS" \
    | sudo tee -a /var/log/lvm-changes.log
}
export -f lvm_log
EOF

# Usage in daily work:
lvm_log "lvextend" "lv_appdata 25GB→35GB. OPS-1234. Approved by: lead"
lvm_log "pvmove"   "Evacuating /dev/sdb SMART errors. OPS-1235"
lvm_log "snapshot" "Pre-migration snapshot lv_db_snap. OPS-1236"
lvm_log "vgextend" "Added lvm-data-4 to vg_production. OPS-1237"
```

---

## Storage Design Principles

These are the principles senior infrastructure engineers apply when designing storage for any new system.

### Principle 1: Separate Concerns Into Separate LVs

```
NEVER put OS, application data, logs, and backups on the same LV.
Each has different growth patterns. Isolation prevents one from killing the others.

BAD:  One 500GB LV for everything
GOOD: /          → 20GB  (OS — never needs to grow much)
      /app/data  → 100GB (application — grows with business)
      /var/log   → 50GB  (logs — grows with traffic, rotated)
      /backup    → 100GB (backups — grows with retention policy)
      VG free    → 230GB (headroom to extend any of the above)
```

### Principle 2: Overprovision the Pool, Underprovision the LVs

```
BAD:  60GB VG, one 55GB LV — no room to extend anything
GOOD: 60GB VG, four 10GB LVs + 20GB free — can extend any LV

You want the VG to be your safety buffer.
LVs should start smaller than their theoretical maximum.
The free space in the VG IS your flexibility.
```

### Principle 3: Always Have Two Layers of Backup

```
Layer 1: LVM snapshot   → fast, in-place, for backup window
Layer 2: GCP snapshot   → offsite, for disaster recovery

LVM snapshot alone:      disk fails → snapshot is gone too
GCP snapshot alone:      no consistent state during DB backup
Both together:           you are protected at every layer ✅
```

### Principle 4: Alert Early, Act Before Crisis

```
Monitor VG free space as a primary metric.

Alert thresholds:
  70% VG used  → Warning. Plan expansion this week.
  80% VG used  → Start expansion now. Do not wait.
  90% VG used  → Emergency. Drop everything.
  100% VG used → System impact. Apps crashing.

Never reach 90%. Never reach 100%.
The 20% buffer between your alert threshold and 100%
is your execution window.
```

### Principle 5: Test Recovery, Not Just Backup

```
Taking a snapshot is WORTHLESS if you cannot restore from it.

Schedule quarterly recovery drills:
  1. Create a test VM in GCP
  2. Restore from your oldest retained snapshot
  3. Verify data integrity (checksums, row counts, app smoke test)
  4. Document actual recovery time → this is your real RTO
  5. Fix any gaps found during the drill

If you have never successfully restored, you do not have a backup.
```

### Principle 6: Document Everything

```
Every VG and LV should answer these questions in documentation:
  → What application uses this?
  → What happens if it fills up?
  → Who is responsible for it?
  → What is the backup/recovery procedure?
  → How do you extend it safely?

Your documentation is for the engineer at 3 AM
who has never seen this system before.
Write it for them.
```

---

## Quick Reference Cheat Sheet

```bash
# ── THIN PROVISIONING ────────────────────────────────────────
sudo lvcreate -L 30G -T -n thin_pool vg_production     # Create pool
sudo lvcreate -V 20G -T -n lv_dev1 vg_production/thin_pool  # Thin LV
sudo lvs vg_production/thin_pool    # Monitor Data% (alert at 70%)
sudo lvextend -L +10G vg_production/thin_pool  # Extend pool when needed

# ── STRIPING ─────────────────────────────────────────────────
sudo lvcreate -L 30G -n lv_striped -i 3 -I 128 vg_production
# -i 3 = stripe across 3 PVs | -I 128 = 128KB stripe size

# ── MIRRORING ────────────────────────────────────────────────
sudo lvcreate -L 10G -m 1 -n lv_mirror vg_production
# -m 1 = 1 mirror = 2 total copies

# ── MYSQL BACKUP ─────────────────────────────────────────────
# 1. FLUSH + LOCK + SNAPSHOT + UNLOCK (tables locked < 1 second)
# 2. Mount snapshot read-only
# 3. tar/rsync from snapshot (app writes continue on original)
# 4. Unmount and remove snapshot
# 5. Upload to GCS / S3

# ── RUNBOOK: VG > 80% ─────────────────────────────────────────
df -hT && sudo vgs && sudo lvs      # Assess
sudo find /var/log -name "*.gz" -mtime +30 -delete  # Quick cleanup
# If not enough: add disk → pvcreate → vgextend → lvextend -r

# ── RUNBOOK: DISK FAILURE ─────────────────────────────────────
sudo vgs    # Look for 'p' in Attr
sudo pvs    # Look for [unknown]
# If disk readable: sudo pvmove /dev/FAILING_DISK
# If completely gone: sudo vgreduce --removemissing --force vg_production
# Restore: gcloud disks create --source-snapshot=SNAP && pvscan && mount -a

# ── CHANGE LOG ────────────────────────────────────────────────
lvm_log "ACTION" "DETAILS. Ticket: OPS-XXXX. Approved by: ENGINEER"

# ── MORNING HEALTH CHECK ──────────────────────────────────────
sudo pvs && sudo vgs && sudo lvs
df -hT | grep -E "(vg_|Filesystem)"
sudo grep -i "lvm\|device-mapper" /var/log/syslog | tail -20
cat /var/log/lvm-changes.log | tail -20
```

---

## Interview Questions

**Intermediate Level:**

- Why would you create two separate VGs for a database server instead of putting everything in one?
- Explain what thin provisioning is and what happens if the thin pool reaches 100%.
- Why is XFS preferred over ext4 for database data directories?
- Walk me through a consistent MySQL backup using LVM snapshots. What is the write-pause window?
- What is the difference between LVM mirroring and RAID? When would you choose each?

**Senior Level:**

- Design the LVM storage architecture for a MySQL primary database server with 4-hour RPO. Show VGs, LVs, disk types, backup strategy.
- Your pvmove has been running for 6 hours and the disk might fail before it completes. What do you do?
- A thin pool has reached 95% usage on a production system. Walk through your response step by step.
- How does TopoLVM integrate LVM with Kubernetes? What problem does it solve?
- You need to migrate a 2TB LV from pd-balanced to pd-ssd with zero downtime. Walk through the procedure.

**Staff Engineer Level:**

- How do you design LVM storage for a Kubernetes cluster with mixed workloads (databases, stateless apps, log aggregation)?
- What are the failure modes of LVM thin provisioning that are NOT present in thick provisioning?
- Your company is moving from bare-metal servers with LVM to GCP. Which LVM capabilities does GCP provide natively? Which do you still need LVM for?
- Design a quarterly DR drill procedure that validates your LVM + GCP snapshot recovery process and measures actual RTO.
- A storage migration is planned during a 2-hour maintenance window. The LV is 500GB and contains a live database. Walk through your plan including rollback decision points.

---

## Common Senior-Level Mistakes

**1. Using one VG for everything on a database server**
Separate VGs for data and logs. If data fills, logs still write. If logs fill, data is unaffected. One VG for everything means one problem takes everything down.

**2. Using thin provisioning for production databases**
Thin pools can fill up and freeze all writes instantly. For databases, thick provisioning guarantees storage. Always monitor thin pool usage if you must use it.

**3. Choosing stripe count higher than number of PVs**
`lvcreate -i 4` on a VG with only 3 PVs fails. Stripes must be equal to or fewer than available PVs in the VG.

**4. Skipping recovery drills**
Having a backup and having a working backup are different things. If you have never successfully restored, you do not have a backup. Schedule and execute quarterly recovery drills.

**5. Not monitoring thin pool separately from VG usage**
VG usage and thin pool fill% are different metrics. A VG can have 40% free while the thin pool inside it is 95% full. Monitor them independently.

**6. Running pvmove on a VG with active snapshots**
pvmove and active LVM snapshots on the same LV can cause unexpected behavior. Remove snapshots before running pvmove.

**7. Designing with no VG headroom**
If your VG is 100% allocated to LVs, you cannot extend any LV without adding hardware. Always leave 20-30% of VG free as operational headroom.

**8. No written runbooks before production changes**
Executing from memory under pressure at 3 AM leads to mistakes. Every significant LVM operation should have a written procedure with rollback steps reviewed before execution.

---

## Summary — What You Now Know

```
Phase 1:  LVM Architecture — PV, VG, LV, Extents, Metadata
Phase 2:  LVM on GCP — live extension, pvmove, snapshots, cost
Phase 3:  Production patterns — databases, containers, CI/CD,
          thin provisioning, striping, mirroring, runbooks,
          troubleshooting, and the staff engineer mindset

─────────────────────────────────────────────────────────────

Real-world patterns:
  Database server  → Separate VGs, XFS, LVM snapshot backup
  Kubernetes node  → LVM per node, TopoLVM for PV provisioning
  CI/CD server     → Snapshot-based clean build environments

Advanced features:
  Thin provisioning  → Overcommit storage. Monitor Data% obsessively.
  Striping           → Parallel I/O. Best on bare-metal NVMe.
  Mirroring          → RAID-1 in software. Skip on cloud (handled by provider).

Production discipline:
  Two backup layers   → LVM snapshot + GCP snapshot always
  Alert at 70%        → Never reach 90%
  vgcfgbackup first   → Before every operation
  Change log          → Every action documented with ticket number
  Runbooks            → Written before the incident, not during
  Recovery drills     → Quarterly. Measured RTO.
  Four-eyes review    → Every production storage change

The engineer who has:
  → done this in the lab (Phase 1 + 2)
  → understands WHY (Phase 3)
  → has written runbooks before the incident
  → has successfully restored from backup at least once

...is the engineer who gets called when it matters most.
```

---
