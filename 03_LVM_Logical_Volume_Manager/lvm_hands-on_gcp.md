# 🌩️ LVM Phase 2 — Hands-On in Google Cloud

> **Learning Path:** Linux Fundamentals → LVM Foundations → LVM Production on GCP  
> **Level:** Intermediate to Advanced  
> **Goal:** Build, extend, recover, snapshot, and monitor a production-grade LVM setup on Google Cloud Platform. Simulate real failure scenarios and learn to respond like a senior infrastructure engineer.

---

## 📚 Table of Contents

1. [What We Will Cover](#what-we-will-cover)
2. [Production Architecture](#production-architecture)
3. [Section 1 — Full Production Setup](#section-1--full-production-setup)
4. [Section 2 — Live Volume Extension Zero Downtime](#section-2--live-volume-extension-zero-downtime)
5. [Section 3 — Disk Failure Simulation and Recovery](#section-3--disk-failure-simulation-and-recovery)
6. [Section 4 — LVM Snapshots](#section-4--lvm-snapshots)
7. [Section 5 — GCP Snapshot Strategy](#section-5--gcp-snapshot-strategy)
8. [Section 6 — Cost Optimization on GCP](#section-6--cost-optimization-on-gcp)
9. [Section 7 — Monitoring VG Health](#section-7--monitoring-vg-health)
10. [Final Architecture Diagram](#final-architecture-diagram)
11. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
12. [Interview Questions](#interview-questions)
13. [Common Beginner Mistakes](#common-beginner-mistakes)
14. [What Differentiates a Senior Engineer](#what-differentiates-a-senior-engineer)

---

## What We Will Cover

- Build a complete production LVM stack on GCP from scratch
- Extend a mounted volume live (zero downtime) while an app is writing
- Simulate disk failure and perform both proactive and reactive recovery
- Use `pvmove` to evacuate a dying disk online
- Create and use LVM snapshots for zero-impact backups
- Design a GCP snapshot strategy for disaster recovery
- Optimize GCP persistent disk costs with LVM
- Write production monitoring and automation scripts

---

## Production Architecture

```
GCP PROJECT: your-project-id
ZONE: us-central1-a

┌────────────────────────────────────────────────────────────────┐
│                  GCE VM: lvm-production                        │
│                  Machine: e2-standard-2                        │
│                                                                 │
│  BOOT DISK:  /dev/sda (20GB pd-standard)  → /                 │
│                                                                 │
│  LVM DISKS:                                                    │
│  /dev/sdb → lvm-data-1 (20GB pd-balanced) ─┐                  │
│  /dev/sdc → lvm-data-2 (20GB pd-balanced) ─┼─→ vg_production  │
│  /dev/sdd → lvm-data-3 (20GB pd-balanced) ─┘                  │
│  /dev/sde → lvm-data-4 (20GB pd-balanced) ← added live        │
│                                                                 │
│  LOGICAL VOLUMES:                                              │
│  lv_appdata (35GB ext4) → /app/data       ← extended live     │
│  lv_logs    (15GB ext4) → /var/log/app                        │
│  lv_backup  (10GB ext4) → /backup                             │
│  lv_db       (5GB xfs)  → /var/lib/db                         │
│                                                                 │
│  AUTOMATION:                                                   │
│  gcp-snapshot.sh     → runs daily at 2 AM via cron            │
│  lvm-health-check.sh → runs every 5 minutes via cron          │
└────────────────────────────────────────────────────────────────┘
```

---

## Section 1 — Full Production Setup

### Step 1.1 — Create VM and Data Disks

```bash
# ── On your LOCAL machine ──────────────────────────────────────

# Set your project
gcloud config set project YOUR_PROJECT_ID

# Create the VM
gcloud compute instances create lvm-production \
  --zone=us-central1-a \
  --machine-type=e2-standard-2 \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard \
  --boot-disk-device-name=boot-disk

# Create 3 data disks
for i in 1 2 3; do
  gcloud compute disks create lvm-data-$i \
    --size=20GB --zone=us-central1-a --type=pd-balanced
done

# Attach all three
for i in 1 2 3; do
  gcloud compute instances attach-disk lvm-production \
    --disk=lvm-data-$i --zone=us-central1-a --device-name=lvm-data-$i
done
```

### Step 1.2 — SSH and Verify

```bash
gcloud compute ssh lvm-production --zone=us-central1-a

# Verify disks
lsblk
# sdb, sdc, sdd should appear as 20G unpartitioned disks

# Install LVM
sudo apt-get update -y && sudo apt-get install -y lvm2
```

### Step 1.3 — Build the LVM Stack (Bottom to Top)

```bash
# ── LAYER 1: Physical Volumes ──────────────────────────────────
sudo pvcreate /dev/sdb /dev/sdc /dev/sdd
sudo pvs

# ── LAYER 2: Volume Group ──────────────────────────────────────
sudo vgcreate vg_production /dev/sdb /dev/sdc /dev/sdd

# IMMEDIATELY backup metadata (senior engineer habit)
sudo vgcfgbackup vg_production

sudo vgdisplay vg_production

# ── LAYER 3: Logical Volumes ───────────────────────────────────
sudo lvcreate -L 25G -n lv_appdata vg_production
sudo lvcreate -L 15G -n lv_logs    vg_production
sudo lvcreate -L 10G -n lv_backup  vg_production
sudo lvcreate -L 5G  -n lv_db      vg_production
sudo lvs
```

### Step 1.4 — Format, Mount, and Persist

```bash
# Format
sudo mkfs.ext4 -L appdata  /dev/vg_production/lv_appdata
sudo mkfs.ext4 -L logs     /dev/vg_production/lv_logs
sudo mkfs.ext4 -L backup   /dev/vg_production/lv_backup
sudo mkfs.xfs  -L database /dev/vg_production/lv_db

# Mount points
sudo mkdir -p /app/data /var/log/app /backup /var/lib/db

# Mount
sudo mount /dev/vg_production/lv_appdata /app/data
sudo mount /dev/vg_production/lv_logs    /var/log/app
sudo mount /dev/vg_production/lv_backup  /backup
sudo mount /dev/vg_production/lv_db      /var/lib/db

# Verify
df -hT | grep vg_production

# Persist in fstab (use UUIDs from blkid)
sudo blkid /dev/vg_production/lv_appdata \
           /dev/vg_production/lv_logs \
           /dev/vg_production/lv_backup \
           /dev/vg_production/lv_db

# Add to fstab:
# UUID=YOUR-UUID-1  /app/data     ext4  defaults  0  2
# UUID=YOUR-UUID-2  /var/log/app  ext4  defaults  0  2
# UUID=YOUR-UUID-3  /backup       ext4  defaults  0  2
# UUID=YOUR-UUID-4  /var/lib/db   xfs   defaults  0  2

# Test fstab
sudo umount /app/data /var/log/app /backup /var/lib/db
sudo mount -a
df -hT | grep vg_production
```

---

## Section 2 — Live Volume Extension Zero Downtime

### The Scenario

```
PRODUCTION ALERT:
"/app/data is 86% full. Threshold: 80%"

Requirements:
  → Add 10GB to /app/data
  → Zero downtime — application must NOT stop
  → Verify before and after
```

### Step 2.1 — Pre-Extension Checklist

```bash
# 1. Confirm the problem
df -hT /app/data

# 2. Check VG free space
sudo vgs vg_production

# 3. Check PV allocation detail
sudo pvs

# 4. Backup VG metadata BEFORE any changes
sudo vgcfgbackup vg_production
echo "$(date): Pre-extension backup done by $(whoami)" \
  >> /var/log/lvm-changes.log
```

### Step 2.2 — Add New Disk to VG (if VG is too full)

```bash
# ── On LOCAL machine ──────────────────────────────────────────
gcloud compute disks create lvm-data-4 \
  --size=20GB --zone=us-central1-a --type=pd-balanced

# Hot-attach to running VM (no reboot needed)
gcloud compute instances attach-disk lvm-production \
  --disk=lvm-data-4 --zone=us-central1-a

# ── Back inside VM ────────────────────────────────────────────
lsblk          # /dev/sde should appear

sudo pvcreate /dev/sde
sudo vgextend vg_production /dev/sde
sudo vgs       # VG now has ~25GB more free space
```

### Step 2.3 — Extend LV While App is Running

```bash
# Prove zero-downtime: start background writer
sudo bash -c '
  while true; do
    echo "$(date): app writing" >> /app/data/activity.log
    sleep 1
  done
' &
WRITER_PID=$!

# Extend lv_appdata by 10GB while app writes
# -r flag: resizes block device AND filesystem in one command
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata

# Verify new size — no interruption to app
df -hT /app/data

# Verify app wrote continuously through the extension
tail -20 /app/data/activity.log

# Stop the test writer
kill $WRITER_PID

# Document the change
echo "$(date): lv_appdata extended +10GB by $(whoami)" \
  >> /var/log/lvm-changes.log
```

### What Happens Internally

```
BEFORE lvextend:
  lv_appdata = 25GB = 6400 PEs

DURING lvextend -L +10G -r:
  1. LVM finds 2560 free PEs in VG (10GB ÷ 4MB)
  2. Assigns them to lv_appdata → now 8960 PEs = 35GB
  3. Updates VG metadata on ALL PVs (redundancy)
  4. Kernel device-mapper updates block device size in real time
  5. -r flag calls resize2fs automatically
  6. ext4 filesystem expands to fill new space

No data moved. No service interrupted. OS saw the block
device grow live while files were being written to it.

AFTER:
  lv_appdata = 35GB
  df -h shows 35GB
  Application never paused
```

---

## Section 3 — Disk Failure Simulation and Recovery

### Scenario A — PROACTIVE: Evacuate a Disk Before It Fails

**Use when:** Monitoring shows SMART errors, reallocated sectors increasing, or disk flagged as failing.

```bash
# Step 1: Check what's on the disk you want to evacuate
sudo pvdisplay --maps /dev/sdb

# Step 2: Move ALL extents off /dev/sdb while everything runs
# pvmove works ONLINE — no unmount, no service restart
sudo pvmove /dev/sdb
# Watch progress: /dev/sdb: Moved: 0.4% → 100.0%

# Step 3: Verify /dev/sdb is now empty
sudo pvdisplay /dev/sdb | grep "Allocated PE"
# Allocated PE    0  ← All extents moved away

# Step 4: Remove empty PV from VG
sudo vgreduce vg_production /dev/sdb

# Step 5: Remove LVM header from disk
sudo pvremove /dev/sdb

# Step 6: Detach from GCP (disk is safe to physically remove)
# gcloud compute instances detach-disk lvm-production \
#   --disk=lvm-data-1 --zone=us-central1-a

# Verify VG is healthy
sudo vgs
df -hT | grep vg_production  # All LVs still mounted ✅
```

### How pvmove Works Internally

```
BEFORE pvmove /dev/sdb:
  sdb: [lv_appdata PEs 0-6399][lv_logs PEs 0-2559]
  sdc: [lv_appdata PEs 6400+][...]
  sde: [free space]

DURING pvmove (for each PE on sdb):
  1. Copy data from source PE (sdb) → destination PE (sdc/sde)
  2. Update PE allocation map in VG metadata
  3. Kernel device-mapper updates LV mapping in real time
  4. Mark source PE as free
  Application reads/writes continue uninterrupted

AFTER pvmove:
  sdb: [free][free][free]  ← Safe to remove
  sdc: [data moved from sdb + original data]
  sde: [more data moved from sdb]
```

### Scenario B — REACTIVE: Disk Already Failed

```bash
# ── Simulate: detach disk suddenly (LOCAL machine) ─────────────
gcloud compute instances detach-disk lvm-production \
  --disk=lvm-data-2 --zone=us-central1-a

# ── Inside VM: detect and recover ────────────────────────────
sudo vgs
# 'p' flag in Attr column = PARTIAL (degraded VG)

sudo pvs
# [unknown] entry with 'a-m' attr = missing PV

# Force-activate VG in partial mode
sudo vgchange -ay --partial vg_production

# Remove missing PV from VG (accept data loss on that disk)
sudo vgreduce --removemissing --force vg_production

# VG is consistent again
sudo vgs
# No more 'p' flag

# Check filesystem integrity on all LVs
sudo fsck -n /dev/vg_production/lv_appdata
sudo fsck -n /dev/vg_production/lv_logs

# Remount everything
sudo mount -a
df -hT | grep vg_production
```

> **Critical lesson:** Data that existed ONLY on the failed disk is permanently lost. This is why proactive `pvmove` when SMART errors appear is essential. This is also why GCP snapshots matter — restore the disk from yesterday's snapshot.

### Recovery from GCP Snapshot After Disk Failure

```bash
# ── On LOCAL machine ──────────────────────────────────────────

# Restore the lost disk from its last snapshot
gcloud compute disks create lvm-data-2-restored \
  --source-snapshot=lvm-data-2-YYYYMMDD \
  --zone=us-central1-a \
  --type=pd-balanced

# Attach restored disk
gcloud compute instances attach-disk lvm-production \
  --disk=lvm-data-2-restored --zone=us-central1-a

# ── Inside VM ─────────────────────────────────────────────────
# LVM auto-detects the restored PV (it still has LVM metadata)
sudo pvscan
sudo vgchange -ay vg_production
sudo mount -a
df -hT | grep vg_production  # Fully restored ✅
```

---

## Section 4 — LVM Snapshots

### What an LVM Snapshot Is

```
ORIGINAL LV: lv_appdata (35GB) — apps writing to it

Create snapshot:
  sudo lvcreate -s -L 5G -n lv_appdata_snap \
    /dev/vg_production/lv_appdata

RESULT: Two views of storage

  Original LV:        Apps continue writing normally
  Snapshot LV:        Frozen view at the moment of creation
                      Uses Copy-on-Write (CoW) to track changes
```

### Copy-on-Write Explained

```
At snapshot creation: NO DATA IS COPIED. Zero overhead.
The snapshot records: "remember the state right now."

When original LV changes:
  BEFORE overwriting a block → copy OLD block to snapshot
  THEN write new data to original

Result:
  Original: has current data (apps work normally)
  Snapshot: has data as it was at creation time

If snapshot fills up (Snap% = 100%):
  Snapshot is INVALIDATED — it can no longer track changes
  The snapshot becomes unusable
  Solution: Use a large enough snapshot, monitor Snap%
```

### Step 4.1 — Create, Mount, and Use Snapshot

```bash
# Create snapshot — 5GB space for changes during backup window
sudo lvcreate -s -L 5G \
  -n lv_appdata_snap \
  /dev/vg_production/lv_appdata

# Verify (Attr 's' = snapshot, Snap% = % of snapshot space used)
sudo lvs

# Mount snapshot READ-ONLY for backup
sudo mkdir -p /mnt/snapshot
sudo mount -o ro /dev/vg_production/lv_appdata_snap /mnt/snapshot

# Backup FROM snapshot (zero impact on running app)
sudo tar -czf /backup/appdata-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C /mnt/snapshot .

# Monitor snapshot usage while original LV changes
sudo lvs /dev/vg_production/lv_appdata_snap

# Clean up when done
sudo umount /mnt/snapshot
sudo lvremove /dev/vg_production/lv_appdata_snap
```

### Snapshot Size Guidelines

```
Formula: snapshot_size = write_rate_per_hour × backup_hours × 2

Example:
  Application writes 500MB/hour
  Backup takes 2 hours
  Changes = 1GB

  Recommended snapshot size: 1GB × 2 = 2GB minimum
  Safe choice: 3GB (3x buffer)

Too small snapshot → fills up → INVALIDATED → backup corrupted
```

### Snapshot for Safe Testing

```bash
# Create snapshot before a risky change (like a DB migration)
sudo lvcreate -s -L 5G \
  -n lv_db_before_migration \
  /dev/vg_production/lv_db

# Make your change...
# If it fails:

# Merge snapshot back (ROLLBACK)
sudo lvconvert --merge /dev/vg_production/lv_db_before_migration
# System will revert lv_db to its state at snapshot creation
# Requires unmounting lv_db first (or reboot for root LVs)
```

---

## Section 5 — GCP Snapshot Strategy

### LVM Snapshot vs GCP Snapshot

| Feature               | LVM Snapshot                 | GCP Snapshot                  |
| --------------------- | ---------------------------- | ----------------------------- |
| Storage location      | Same physical disk           | GCP object storage            |
| Survives disk failure | ❌ No                        | ✅ Yes                        |
| Speed                 | Instant (CoW)                | Minutes                       |
| Cost                  | Free (uses your LV space)    | ~$0.026/GB/month              |
| Cross-region restore  | ❌ No                        | ✅ Yes                        |
| Max retention         | While VG exists              | Forever                       |
| Best for              | Backup windows, safe testing | Disaster recovery, compliance |

### Step 5.1 — Automated GCP Snapshot Script

```bash
sudo tee /usr/local/bin/gcp-snapshot.sh << 'EOF'
#!/bin/bash
# Production GCP Snapshot Script — runs daily via cron
set -euo pipefail

PROJECT=$(curl -sf \
  "http://metadata.google.internal/computeMetadata/v1/project/project-id" \
  -H "Metadata-Flavor: Google")
ZONE=$(curl -sf \
  "http://metadata.google.internal/computeMetadata/v1/instance/zone" \
  -H "Metadata-Flavor: Google" | awk -F/ '{print $NF}')
DATE=$(date +%Y%m%d)
RETENTION_DAYS=7
LOG="/var/log/gcp-snapshots.log"

echo "$(date): Snapshot job started" >> "$LOG"

# Snapshot all LVM data disks
for DISK in lvm-data-1 lvm-data-2 lvm-data-3 lvm-data-4; do
  gcloud compute disks snapshot "$DISK" \
    --snapshot-names="${DISK}-${DATE}" \
    --zone="$ZONE" \
    --project="$PROJECT" \
    --description="Auto snapshot $(date)" 2>> "$LOG" && \
  echo "$(date): Created snapshot ${DISK}-${DATE}" >> "$LOG"
done

# Delete snapshots older than RETENTION_DAYS
gcloud compute snapshots list \
  --project="$PROJECT" \
  --filter="name:lvm-data AND creationTimestamp < $(
    date -d "${RETENTION_DAYS} days ago" +%Y-%m-%dT%H:%M:%SZ)" \
  --format="value(name)" | while read SNAP; do
    gcloud compute snapshots delete "$SNAP" \
      --project="$PROJECT" --quiet 2>> "$LOG"
    echo "$(date): Deleted old snapshot $SNAP" >> "$LOG"
done

echo "$(date): Snapshot job completed" >> "$LOG"
EOF

sudo chmod +x /usr/local/bin/gcp-snapshot.sh

# Schedule: daily at 2 AM
(crontab -l 2>/dev/null; \
  echo "0 2 * * * /usr/local/bin/gcp-snapshot.sh") | crontab -

# Test run
sudo /usr/local/bin/gcp-snapshot.sh
```

### Step 5.2 — Disaster Recovery from GCP Snapshot

```bash
# ── On LOCAL machine ──────────────────────────────────────────

# List available snapshots
gcloud compute snapshots list \
  --filter="name:lvm-data" \
  --format="table(name, diskSizeGb, creationTimestamp, status)"

# Restore disk from snapshot
gcloud compute disks create lvm-data-2-restored \
  --source-snapshot=lvm-data-2-YYYYMMDD \
  --zone=us-central1-a \
  --type=pd-balanced

# Attach restored disk
gcloud compute instances attach-disk lvm-production \
  --disk=lvm-data-2-restored \
  --zone=us-central1-a

# ── Inside VM ─────────────────────────────────────────────────
sudo pvscan                      # LVM finds restored PV automatically
sudo vgchange -ay vg_production  # Activate VG
sudo mount -a                    # Mount all filesystems
df -hT | grep vg_production      # Verify everything restored ✅
```

---

## Section 6 — Cost Optimization on GCP

### GCP Persistent Disk Pricing Reference

| Disk Type     | Cost/GB/Month | IOPS             | Best For                     |
| ------------- | ------------- | ---------------- | ---------------------------- |
| `pd-standard` | ~$0.040       | Low (HDD-backed) | Backups, cold storage, logs  |
| `pd-balanced` | ~$0.100       | 3,000 cap        | General purpose, most VMs    |
| `pd-ssd`      | ~$0.170       | 30 IOPS/GB       | Databases, high-performance  |
| `pd-extreme`  | ~$0.300       | Up to 350K       | SAP HANA, top-tier databases |
| Snapshots     | ~$0.026       | N/A              | Only charged for used data   |

### Strategy 1: Start Small, Grow with LVM

```bash
# WRONG: Overprovision "just in case"
# 500GB disk at $50/month even though you use 50GB

# RIGHT: Start small, extend as needed
gcloud compute disks create lvm-data-1 \
  --size=20GB --type=pd-balanced --zone=us-central1-a
# $2.00/month

# 3 months later, need more space:
gcloud compute disks create lvm-data-extra \
  --size=10GB --type=pd-balanced --zone=us-central1-a

gcloud compute instances attach-disk lvm-production \
  --disk=lvm-data-extra --zone=us-central1-a

sudo pvcreate /dev/sde
sudo vgextend vg_production /dev/sde
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata
# Pay only for what you use. Grow on demand.
```

### Strategy 2: Right Disk Type Per Workload

```bash
# Assign disk types based on I/O requirements:

# Backup disk — sequential writes, low IOPS needed
gcloud compute disks create lvm-backup-disk \
  --size=50GB --type=pd-standard   # $2.00/month

# Application data — general purpose
gcloud compute disks create lvm-app-disk \
  --size=50GB --type=pd-balanced   # $5.00/month

# Database — random I/O, high IOPS needed
gcloud compute disks create lvm-db-disk \
  --size=50GB --type=pd-ssd        # $8.50/month

# Use LVM to route each LV to its optimal disk
# lv_backup  → pd-standard (cheap)
# lv_appdata → pd-balanced (balanced)
# lv_db      → pd-ssd      (fast)
```

### Strategy 3: Stop VMs When Not in Use

```bash
# Stop VM — CPU/RAM billing stops. Disk billing continues.
gcloud compute instances stop lvm-production --zone=us-central1-a

# Start VM when needed again
gcloud compute instances start lvm-production --zone=us-central1-a

# For the lab:
# Study 4 hours/day × 20 days = 80 hours/month
# e2-standard-2: ~$0.067/hour × 80 hours = $5.36/month
# Disks (4 × 20GB pd-balanced): $8.00/month
# Snapshots (~80GB used data): $2.08/month
# TOTAL: ~$15.44/month
# New GCP accounts get $300 free credit = 3 months free
```

### Strategy 4: Snapshot Lifecycle Management

```bash
# Only keep snapshots as long as required
# 7-day retention for lab: reasonable
# 30-day retention for production: compliance requirement

# Calculate snapshot cost before creating:
# 100GB LV with 40GB actual data
# Snapshot cost = 40GB × $0.026 = $1.04/month per snapshot
# 7 snapshots = $7.28/month — reasonable

# Delete old snapshots to control costs
gcloud compute snapshots list \
  --filter="creationTimestamp < 2026-02-01" \
  --format="value(name)" | \
  xargs -I{} gcloud compute snapshots delete {} --quiet
```

---

## Section 7 — Monitoring VG Health

### Production Health Check Script

```bash
sudo tee /usr/local/bin/lvm-health-check.sh << 'SCRIPT'
#!/bin/bash
# LVM Health Check — run every 5 minutes via cron
ALERT_THRESHOLD=80
CRITICAL_THRESHOLD=90
LOG="/var/log/lvm-health.log"

echo "$(date): Health check started" >> "$LOG"

# Check VG usage
sudo vgs --noheadings --units g \
  -o vg_name,vg_size,vg_free | \
while read VG SIZE FREE; do
  USED=$(echo "$SIZE $FREE" | \
    awk '{printf "%.0f", (($1-$2)/$1)*100}')
  echo "VG $VG: ${USED}% used (${FREE}g free)" >> "$LOG"

  if [ "$USED" -ge "$CRITICAL_THRESHOLD" ]; then
    logger -p local0.crit -t LVM-HEALTH \
      "CRITICAL: $VG is ${USED}% full on $(hostname)"
  elif [ "$USED" -ge "$ALERT_THRESHOLD" ]; then
    logger -p local0.warning -t LVM-HEALTH \
      "WARNING: $VG is ${USED}% full on $(hostname)"
  fi
done

# Check for partial/degraded VGs
if sudo vgs --noheadings -o vg_attr | grep -q "p"; then
  logger -p local0.crit -t LVM-HEALTH \
    "CRITICAL: Partial VG detected on $(hostname) - disk failure!"
  echo "CRITICAL: PARTIAL VG DETECTED" >> "$LOG"
fi

# Check snapshot usage
sudo lvs --noheadings -o lv_name,snap_percent 2>/dev/null | \
  grep -v "^$" | while read LV PCT; do
    PCT_INT=${PCT%.*}
    if [ "${PCT_INT:-0}" -ge 80 ]; then
      logger -p local0.warning -t LVM-HEALTH \
        "WARNING: Snapshot $LV is ${PCT}% full"
    fi
  done

echo "$(date): Health check completed" >> "$LOG"
SCRIPT

sudo chmod +x /usr/local/bin/lvm-health-check.sh

# Run every 5 minutes
(crontab -l 2>/dev/null; \
  echo "*/5 * * * * /usr/local/bin/lvm-health-check.sh") \
  | crontab -
```

### Daily Morning Check Commands

```bash
# Run these every morning in production
echo "=== $(date) ==="
echo "=== PV STATUS ===" && sudo pvs
echo "=== VG STATUS ===" && sudo vgs
echo "=== LV STATUS ===" && sudo lvs
echo "=== DISK USAGE ===" && df -hT | grep -E "(vg_|Filesystem)"

# Check for any LVM warnings or errors
sudo grep -i "lvm\|vg_production\|device-mapper" \
  /var/log/syslog | tail -20

# View the change log
cat /var/log/lvm-changes.log
```

---

## Final Architecture Diagram

```
GOOGLE CLOUD PROJECT
│
├── ZONE: us-central1-a
│
├── VM: lvm-production (e2-standard-2)
│   │
│   ├── Boot Disk: /dev/sda (20GB pd-standard) → /
│   │
│   ├── PHYSICAL VOLUMES:
│   │   ├── /dev/sdb → lvm-data-1 (20GB pd-balanced)
│   │   ├── /dev/sdc → lvm-data-2 (20GB pd-balanced)
│   │   ├── /dev/sdd → lvm-data-3 (20GB pd-balanced)
│   │   └── /dev/sde → lvm-data-4 (20GB pd-balanced) [added live]
│   │
│   ├── VOLUME GROUP: vg_production (80GB pool)
│   │
│   ├── LOGICAL VOLUMES:
│   │   ├── lv_appdata (35GB ext4) → /app/data      [extended live]
│   │   ├── lv_logs    (15GB ext4) → /var/log/app
│   │   ├── lv_backup  (10GB ext4) → /backup
│   │   └── lv_db       (5GB xfs)  → /var/lib/db
│   │
│   └── AUTOMATION:
│       ├── gcp-snapshot.sh      (cron: daily 2 AM)
│       └── lvm-health-check.sh  (cron: every 5 min)
│
└── GCP SNAPSHOTS (7-day retention):
    ├── lvm-data-1-YYYYMMDD
    ├── lvm-data-2-YYYYMMDD
    ├── lvm-data-3-YYYYMMDD
    └── lvm-data-4-YYYYMMDD
```

---

## Quick Reference Cheat Sheet

```bash
# ── FULL STATUS CHECK ─────────────────────────────────────────
sudo pvs && sudo vgs && sudo lvs
df -hT | grep vg_production
sudo pvdisplay --maps /dev/sdb     # Show PE allocation on a PV

# ── ADD DISK TO RUNNING SYSTEM ────────────────────────────────
# 1. Create and attach (GCP):
gcloud compute disks create new-disk \
  --size=20GB --zone=us-central1-a --type=pd-balanced
gcloud compute instances attach-disk lvm-production \
  --disk=new-disk --zone=us-central1-a

# 2. Register and extend (inside VM):
sudo pvcreate /dev/sde
sudo vgextend vg_production /dev/sde

# ── EXTEND LV LIVE ────────────────────────────────────────────
sudo lvextend -L +10G -r /dev/vg_production/lv_appdata  # +10GB
sudo lvextend -L 50G  -r /dev/vg_production/lv_appdata  # to 50GB
sudo lvextend -l +100%FREE -r /dev/vg_production/lv_appdata  # use all free

# ── EVACUATE FAILING DISK ──────────────────────────────────────
sudo pvdisplay --maps /dev/sdb     # Check what's on it
sudo pvmove /dev/sdb               # Move everything off (online)
sudo vgreduce vg_production /dev/sdb  # Remove from VG
sudo pvremove /dev/sdb             # Remove LVM header

# ── RECOVER DEGRADED VG ───────────────────────────────────────
sudo vgs                                       # Look for 'p' in Attr
sudo vgchange -ay --partial vg_production      # Force-activate
sudo vgreduce --removemissing --force vg_production  # Remove missing PV
sudo fsck -n /dev/vg_production/lv_appdata     # Check integrity

# ── LVM SNAPSHOTS ─────────────────────────────────────────────
sudo lvcreate -s -L 5G -n snap_name /dev/vg_production/lv_appdata
sudo mount -o ro /dev/vg_production/snap_name /mnt/snapshot
sudo umount /mnt/snapshot
sudo lvremove /dev/vg_production/snap_name

# ── GCP SNAPSHOTS ─────────────────────────────────────────────
gcloud compute disks snapshot lvm-data-1 \
  --snapshot-names=lvm-data-1-$(date +%Y%m%d) \
  --zone=us-central1-a

gcloud compute snapshots list

gcloud compute disks create restored-disk \
  --source-snapshot=lvm-data-1-YYYYMMDD \
  --zone=us-central1-a --type=pd-balanced

# ── METADATA BACKUP ───────────────────────────────────────────
sudo vgcfgbackup vg_production    # Before EVERY risky operation
sudo vgcfgrestore vg_production   # Recovery from backup

# ── VM LIFECYCLE ──────────────────────────────────────────────
gcloud compute instances stop  lvm-production --zone=us-central1-a
gcloud compute instances start lvm-production --zone=us-central1-a
```

---

## Interview Questions

**Beginner Level:**

- What is the difference between an LVM snapshot and a GCP snapshot?
- What does `pvmove` do and when would you use it?
- What does the `p` flag mean in `vgs` output?
- What is Copy-on-Write in the context of LVM snapshots?
- Why do you use `lvextend -r` instead of separate `lvextend` and `resize2fs`?

**Intermediate Level:**

- Walk me through recovering a VG after a PV goes offline suddenly.
- A developer reports that their app paused briefly during an LVM extension. Why could this happen and how do you prevent it?
- Your LVM snapshot invalidated halfway through a backup. What happened and how do you prevent it?
- Explain the difference between `pd-standard`, `pd-balanced`, and `pd-ssd`. When would you choose each for an LVM PV?
- How does GCP hot-attach allow adding a disk to a running VM without rebooting?

**Senior Level:**

- Design a complete LVM + GCP snapshot strategy for a MySQL database with 4-hour RPO and 1-hour RTO requirements.
- A `pvmove` operation is taking too long and you need to cancel it safely. Walk through the procedure.
- Your production VG metadata is corrupted and `vgchange` fails. Walk through full recovery using `vgcfgrestore`.
- How would you design an LVM architecture to ensure that your most critical LV (`lv_db`) never spans a disk that also contains `lv_appdata`?
- Explain the cost implications of snapshotting a 500GB LV vs a 500GB GCP Persistent Disk. Which is more efficient and why?

---

## Common Beginner Mistakes

**1. Making snapshot too small and watching it invalidate**
If your snapshot fills to 100%, it is invalidated and your backup is corrupted. Always calculate: `write_rate × backup_duration × 2` and use that as your snapshot size minimum.

**2. Not running pvmove when SMART errors first appear**
Engineers wait until the disk fails completely. By then, data on that disk is lost. Run `pvmove` the moment monitoring shows degradation.

**3. Using `vgreduce --removemissing --force` without understanding data loss**
This command permanently removes the missing PV and accepts any data loss. Always check `pvdisplay --maps` first to understand what data was on the missing disk and whether you have a GCP snapshot to restore from.

**4. Forgetting to reload rsyslog or app services after the LV extends**
Some applications cache the filesystem size at startup. After extending, verify the application sees the new size. Most modern apps do not need this, but legacy apps sometimes do.

**5. Snapshotting the LV but forgetting to snapshot the GCP disk**
LVM snapshots live on the same physical disk. If that disk fails, the snapshot is gone too. Always pair LVM snapshots (for backup windows) with GCP snapshots (for disaster recovery).

**6. Not monitoring `Snap%` during long backup jobs**
If your backup takes 4 hours and your snapshot fills up at hour 3 — your entire backup is invalid. Add snapshot monitoring to your health check script.

**7. Leaving stopped VMs with all disks attached**
A stopped VM does not incur CPU/RAM costs but still incurs disk costs. If you are not using the lab for a week, detach and delete unused disks. Keep only what you need.

---

## What Differentiates a Senior Engineer

```
Junior Engineer:
  → Follows the steps when told what to do
  → Knows lvextend, pvmove commands
  → Can extend a volume successfully

Mid-level Engineer:
  → Proactively monitors VG free space (alerts at 80%)
  → Runs pvmove when SMART errors first appear
  → Takes GCP snapshots before every risky operation
  → Uses -r flag on lvextend automatically

Senior Engineer:
  → Designs storage architecture BEFORE deployment
    (right disk type per workload, right LV sizes with headroom)
  → Writes automation for snapshots and health checks
  → Has runbooks for every failure scenario BEFORE it happens
  → Considers RPO/RTO requirements and designs backup strategy around them
  → Pairs LVM snapshots with GCP snapshots at every layer
  → Calculates cost per GB and optimizes disk types per LV purpose
  → Peer-reviews all LVM changes with a second engineer
  → Maintains a /var/log/lvm-changes.log audit trail
  → Knows that pvmove can be paused and resumed safely
  → Can recover from VG metadata corruption using vgcfgrestore
  → Monitors snapshot Snap% as a key operational metric
```

---

## Summary

```
pvmove          → Evacuate a PV online. No unmount. No downtime.
                  Use when: SMART errors, planned disk retirement
vgreduce        → Remove a PV from VG (must be empty first)
--removemissing → Remove a dead/missing PV (accepts data loss)
LVM Snapshot    → CoW point-in-time copy. Fast. For backup windows.
                  Invalidates if full. Lives on same disk.
GCP Snapshot    → Full copy stored in GCP object storage.
                  Survives disk failure. For disaster recovery.
pd-standard     → HDD-backed, $0.040/GB. Backups, cold storage.
pd-balanced     → SSD-backed, $0.100/GB. General purpose.
pd-ssd          → SSD-backed, $0.170/GB. Databases, high IOPS.
Hot-attach      → GCP can add a disk to a running VM. No reboot.
Health check    → Monitor VG free space. Alert at 80%. Critical at 90%.
Snapshot%       → Monitor snapshot fill %. Alert before it hits 100%.
vgcfgbackup     → Run before EVERY LVM operation. Takes 1 second.
lvm-changes.log → Audit trail. Every change logged with who and when.
```

---
