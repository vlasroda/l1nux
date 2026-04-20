# Week 2 Exercise 3: LVM and Storage Monitoring

**Difficulty:** Intermediate
**Time:** 25 minutes
**Prerequisites:** Lesson 2.3 (LVM and Storage Monitoring)

---

## Problem 1: Analyze LVM Setup (if present)

**Task 1A: Check if LVM is in use**
```bash
# List logical volumes
sudo lvs 2>/dev/null || echo "LVM not configured"

# List volume groups
sudo vgs 2>/dev/null || echo "LVM not configured"

# List physical volumes
sudo pvs 2>/dev/null || echo "LVM not configured"
```

**Task 1B: If LVM is present, document the structure**
```bash
# Show LV details
sudo lvs -o lv_name,lv_size,vg_name,lv_status

# Show VG details
sudo vgs -o vg_name,vg_size,vg_free,lv_count

# Show PV details
sudo pvs -o pv_name,vg_name,pv_size,pv_free

# Document:
# 1. How many VGs?
# 2. How many LVs per VG?
# 3. How much free space in each VG?
```

**Task 1C: Check for snapshots**
```bash
# Look for snapshot LVs
sudo lvs -o lv_name,lv_type,lv_size

# If you see "snapshot" or "snap" in the output:
# - How many snapshots?
# - How old are they?
# - What are they snapshots of?

# Snapshots consume space and should be removed when no longer needed
```

---

## Problem 2: Expand a Logical Volume (Simulated)

**Scenario:** /home is at 85% capacity and needs more space. How would you expand it?

**Task 2A: Check current size**
```bash
# Show home filesystem
df -h /home

# Example output:
# /dev/mapper/vg0-lv_home  50G   43G   7G  86%

# Also check:
sudo lvs /dev/vg0/lv_home
```

**Task 2B: Check VG free space**
```bash
# How much free space in VG?
sudo vgs vg0

# Example:
# VG   #PV #LV #SN VSize  VFree
# vg0    3   5   0 300G 100G

# Interpretation: 100G free to allocate
```

**Task 2C: Create expansion plan**
```bash
# Plan the steps:
echo "1. Check free space in VG: sudo vgs vg0"
echo "2. Extend LV by 30G: sudo lvextend -L +30G /dev/vg0/lv_home"
echo "3. Grow filesystem: sudo resize2fs /dev/vg0/lv_home"
echo "4. Verify: df -h /home"

# Document:
# - How much space to add?
# - Are there any risks?
# - Will the VG have enough?
```

**Task 2D: Verify the plan would work**
```bash
# Check:
# 1. Is filesystem ext4 or XFS?
sudo lsblk -f | grep lv_home

# 2. Is it mounted?
mount | grep lv_home

# 3. Can you extend without unmounting?
# - ext4: Yes (online resize)
# - XFS: Yes (online resize)
# - Some others: Might need unmount

# Document findings
```

---

## Problem 3: Monitor Storage Growth

**Scenario:** You need to track storage usage to predict when arrays will fill up.

**Task 3A: Establish baseline**
```bash
# Current snapshot:
date > /tmp/storage_baseline.txt
echo "=== Filesystem Usage ===" >> /tmp/storage_baseline.txt
df -h >> /tmp/storage_baseline.txt
echo "" >> /tmp/storage_baseline.txt
echo "=== LVM Status ===" >> /tmp/storage_baseline.txt
sudo vgs >> /tmp/storage_baseline.txt
sudo lvs >> /tmp/storage_baseline.txt

# Save for comparison
cat /tmp/storage_baseline.txt
```

**Task 3B: Identify fast-growing filesystems**
```bash
# Find directories with high growth
du -sh /var/*  2>/dev/null | sort -rh

# Questions:
# 1. What's the largest?
# 2. What's second-largest?
# 3. Are any growing unexpectedly?
# 4. When was /var/log last rotated?
```

**Task 3C: Calculate growth rate (simulated)**
```bash
# If you had measurements from 30 days ago:
# /var/log: 10GB
# /var/log: 15GB (now)
# Growth: 5GB in 30 days = 166MB/day

# At this rate:
# If /var is 100GB and 85GB used:
# 15GB remaining at 166MB/day = 90 days until full

# Document expected fill-up date
```

**Task 3D: Identify cleanup opportunities**
```bash
# Old log files
find /var/log -type f -mtime +30 | head -20

# Temporary files
ls -lh /tmp/ | tail -20

# Old cache
du -sh /var/cache/

# Package cache
du -sh /var/cache/apt/ 2>/dev/null

# Document what could be safely cleaned
```

---

## Problem 4: Handle Disk Full Emergency

**Scenario:** A critical filesystem is 98% full and applications are failing. You need to free space NOW.

**Task 4A: Identify the culprit**
```bash
# Quick identification
du -sh /* | sort -rh

# Zoom in on largest
du -sh /var/* | sort -rh

# Go deeper
find /var -type f -size +100M -exec ls -lh {} \; | awk '{print $5, $NF}' | sort -rh
```

**Task 4B: Prioritize cleanup (ranked by safety)**
```bash
# Safest (definitely OK to delete):
# - Old log files (>30 days old)
# - Temp files (in /tmp)
# - Downloaded packages (apt cache)

# Safe (usually OK):
# - Backup files (*.bak, *.old)
# - Previous package versions
# - Old journal entries

# Risky (need to verify first):
# - Application data
# - User files
# - System files
```

**Task 4C: Execute cleanup**
```bash
# Example sequence (VERIFY FIRST!):

# 1. Rotate and compress logs
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;

# 2. Delete empty log files
find /var/log -empty -delete

# 3. Clean package cache
sudo apt-get clean 2>/dev/null

# 4. Clear temp files (older than 7 days)
find /tmp -atime +7 -delete

# Verify space freed
df -h
```

**Task 4D: Document the cleanup**
```bash
# What was deleted?
# How much space was freed?
# How much space remains?
# What's the new urgent level?

# Example:
echo "Cleaned 15GB from /var/log"
echo "Freed 2GB from /tmp"
echo "/var now 65% full (safe)"
```

---

## Problem 5: Snapshot and Backup Scenario

**Scenario:** You need to backup a live database on /data without stopping it.

**Task 5A: Check if LVM available**
```bash
# Is /data on LVM?
df /data | grep mapper

# If yes:
# Get the LV name
mount | grep " /data "
# Shows something like: /dev/mapper/vg0-lv_data on /data

# LV path: /dev/vg0/lv_data
```

**Task 5B: Plan snapshot backup**
```bash
# Determine snapshot size:
# Rule of thumb: 10-20% of LV size for snapshot

# If /data is 50GB and 40GB used:
# Snapshot size: 50G × 0.2 = 10GB

# Document plan:
echo "1. Create 10G snapshot: sudo lvcreate -L 10G -s -n lv_data_snap /dev/vg0/lv_data"
echo "2. Mount snapshot: sudo mount /dev/vg0/lv_data_snap /mnt/snap"
echo "3. Backup: tar czf backup.tar.gz /mnt/snap/"
echo "4. Unmount and remove: sudo umount /mnt/snap && sudo lvremove /dev/vg0/lv_data_snap"
```

**Task 5C: Create the backup command**
```bash
# Full backup command (non-destructive):
SNAP_NAME="lv_data_snap_$(date +%Y%m%d_%H%M%S)"
SNAP_SIZE="10G"
BACKUP_DIR="/backups"

# (Don't run unless you have LVM and want to backup)
echo "Commands to execute:"
echo "sudo lvcreate -L $SNAP_SIZE -s -n $SNAP_NAME /dev/vg0/lv_data"
echo "sudo mount /dev/vg0/$SNAP_NAME /mnt/snap"
echo "sudo tar czf $BACKUP_DIR/data_$(date +%Y%m%d).tar.gz /mnt/snap/"
echo "sudo umount /mnt/snap"
echo "sudo lvremove /dev/vg0/$SNAP_NAME"
```

---

## Challenge: Storage Management Dashboard

**Create a comprehensive monitoring script:**

```bash
#!/bin/bash

echo "=== Storage Management Dashboard ==="
echo "Generated: $(date)"
echo ""

echo "1. FILESYSTEM CAPACITY:"
df -h | awk 'NR==1 {printf "%-30s %10s %10s %10s\n", $1, "Size", "Used", "Use%"}
NR>1 {if ($5+0 > 80) {printf ">>> "} printf "%-30s %10s %10s %10s\n", $1, $2, $3, $5}'

echo ""
echo "2. INODE USAGE:"
df -i | awk 'NR>1 && $5+0 > 80 {printf "%-30s %10s\n", $1, $5}'

echo ""
echo "3. LVM STATUS:"
sudo vgs -o vg_name,vg_size,vg_free --units G 2>/dev/null || echo "LVM not available"

echo ""
echo "4. TOP SPACE CONSUMERS:"
du -sh /* 2>/dev/null | sort -rh | head -5

echo ""
echo "5. ALERTS:"
FULL_FS=$(df -h | awk '$5+0 > 90 {print $1}')
if [ -n "$FULL_FS" ]; then
    echo "⚠️  CRITICAL: Following filesystems over 90%: $FULL_FS"
else
    echo "✓ No critical filesystem alerts"
fi

echo ""
echo "6. RAID STATUS:"
sudo cat /proc/mdstat 2>/dev/null | grep -E "^md|recovery" || echo "No RAID detected or issues"
```

**Your task:**
1. Save as `/usr/local/bin/storage_dashboard.sh`
2. Make executable: `chmod +x`
3. Run monthly: `crontab -e` → Add: `0 2 1 * * /usr/local/bin/storage_dashboard.sh > /tmp/storage_report.txt`
4. Review reports monthly

---

## Self-Check

After completing all problems:

- [ ] Problem 1: Analyzed LVM setup (if present)
- [ ] Problem 2: Created plan to expand LV
- [ ] Problem 3: Established baseline and growth rate
- [ ] Problem 4: Handled emergency cleanup
- [ ] Problem 5: Planned snapshot-based backup
- [ ] Challenge: Created monitoring dashboard

---

## Key Skills You Practiced

- `sudo lvs/vgs/pvs` — LVM status
- `sudo lvextend` — LV expansion
- `sudo resize2fs` — Filesystem growth
- `du -sh` — Directory space analysis
- Capacity planning and growth prediction
- Emergency cleanup procedures

---

## Next Steps

Compare your answers with `week2_solutions.md`.

Focus on:
1. Regular monitoring prevents emergencies
2. LVM snapshots are powerful backup tools
3. Always have a cleanup plan before disk fills

---

Powered by UQS v1.8.5-C
