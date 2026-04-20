# Week 2 Exercise 2: RAID and Disk Management

**Difficulty:** Intermediate
**Time:** 25 minutes
**Prerequisites:** Lesson 2.2 (Block Devices & RAID)

---

## Problem 1: Analyze Block Device Layout

**Task 1A: View all block devices**
```bash
# Command:
lsblk

# Document:
# 1. How many physical disks?
# 2. How many partitions per disk?
# 3. What sizes?
# 4. Any RAID arrays visible?
```

**Task 1B: Get detailed device information**
```bash
# With filesystem info:
sudo lsblk -f

# With permissions:
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Questions:
# 1. What filesystem types are in use?
# 2. Are any NVMe drives present?
# 3. What's mounted where?
```

**Task 1C: Check partition table type**
```bash
# For each disk:
sudo fdisk -l /dev/sda | grep "Disklabel\|Device\|Start\|End"

# Questions:
# 1. MBR or GPT?
# 2. How many partitions?
# 3. Any unallocated space?
```

---

## Problem 2: Check RAID Status

**Task 2A: Detect RAID arrays**
```bash
# Check if any software RAID
sudo cat /proc/mdstat

# Expected outputs:
# - "No md devices" = no software RAID
# - "md0 : active raid1..." = RAID array present
```

**Task 2B: Get detailed RAID info (if present)**
```bash
# If output from 2A shows RAID:
sudo mdadm --detail /dev/md0

# Look for:
# - Array UUID
# - Raid Level
# - Array State
# - Device list with status
```

**Task 2C: Interpret RAID status**
```bash
# Key patterns to recognize:

# [UUU] = All devices Up
# [U_U] = One device Failed (missing)
# [UUU_] = One device being rebuilt

# State: clean, recovering, checking, degraded

# If anything other than [UUU] and "clean":
# Document the issue for analysis
```

---

## Problem 3: Diagnose Disk Health

**Task 3A: Check SMART status (if available)**
```bash
# Quick health check
sudo smartctl -H /dev/sda

# Expected output:
# === START OF READ SMART DATA SECTION ===
# SMART overall-health self-assessment test result: PASSED

# If it shows FAILED:
# That disk needs attention
```

**Task 3B: Get detailed SMART information**
```bash
# Full SMART report
sudo smartctl -a /dev/sda | head -50

# Look for:
# - Reallocated_Sector_Ct (should be low or 0)
# - Current_Pending_Sector (should be 0)
# - Offline_Uncorrectable (should be 0)

# High numbers = disk failing
```

**Task 3C: Check kernel error logs**
```bash
# Look for disk errors
dmesg | grep -i "error\|fail" | tail -20

# Or in journal
journalctl -p err --since "24 hours ago" | grep -i "disk\|ata\|sata"

# Document any errors found
```

---

## Problem 4: Simulate Failure Scenario

**Scenario:** Disk /dev/sdb is showing SMART errors. You need to prepare for replacement.

**Task 4A: Verify the problem disk**
```bash
# Check which partitions are on sdb
lsblk | grep sdb

# Check if in RAID
sudo cat /proc/mdstat | grep sdb

# Check mount points
mount | grep sdb

# Document:
# - Is sdb critical?
# - Is it in RAID?
# - Is it currently in use?
```

**Task 4B: Create replacement plan**
```bash
# For each partition on sdb, document:
# 1. Mount point (if any)
# 2. Filesystem type
# 3. Data importance
# 4. Whether RAID member

# Example plan:
# /dev/sdb1: /data, ext4, RAID5 member, critical
# /dev/sdb2: /backup, ext4, no RAID, non-critical
```

**Task 4C: Check backup status (in case needed)**
```bash
# For critical data on sdb:
# Is there a backup?
# When was it last updated?

# Example:
# /data is on sdb1
# Backup to: /backup/data/ (daily)
# Last backup: today 10:00 AM
```

---

## Problem 5: Capacity Planning

**Scenario:** You manage a storage array with multiple disks. Need to predict when it will fill up.

**Current state:**
```
Array: 10× 1TB disks, RAID 5 (66% usable = 6.6TB)
Current usage: 5TB
Growth rate: 100GB/month
```

**Task 5A: Calculate space remaining**
```bash
# Work through:
# 1. Total usable capacity: 10TB × 0.66 = 6.6TB
# 2. Currently used: 5TB
# 3. Remaining: 6.6TB - 5TB = 1.6TB
# 4. At 100GB/month growth: 1.6TB / 100GB = 16 months
```

**Task 5B: Calculate replacement timeline**
```bash
# If using RAID 5:
# - Average disk lifespan: 5 years
# - Rebuild time: ~24 hours per 1TB
# - This array: rebuild time = ~24 hours

# Question:
# Given current disk lifespan and failure rates,
# when should you plan for first disk replacement?
# (Typical: 3-5 years into service)
```

**Task 5C: Create upgrade timeline**
```bash
# At current growth rate:
# Month 0: 5TB used (76% full)
# Month 8: 5.8TB used (88% full) - Getting urgent
# Month 12: 6.2TB used (94% full) - Critical
# Month 16: 6.6TB used (100% full) - Out of space

# Recommendation:
# Plan upgrade by Month 8 (before 85% threshold)
```

---

## Problem 6: Partition vs No Partition

**Scenario:** You're given a new 2TB drive to add to the system. Should you partition it or use as raw PV for LVM?

**Task 6A: Evaluate partitioning**
```bash
# Option 1: Traditional partitions
# sudo fdisk /dev/sdc
# Create /dev/sdc1 and /dev/sdc2

# Pros:
# - Simple to understand
# - Legacy systems support

# Cons:
# - Fixed sizes (hard to resize)
# - Limited to 4 primary partitions
```

**Task 6B: Evaluate LVM (raw)**
```bash
# Option 2: Raw disk to LVM
# sudo pvcreate /dev/sdc
# sudo vgextend vg0 /dev/sdc

# Pros:
# - Flexible allocation
# - Easy to resize
# - Can add more disks to same VG

# Cons:
# - Requires LVM already in use
# - Less familiar to some admins
```

**Task 6C: Make a recommendation**
```bash
# Consider:
# 1. Is the system already using LVM?
# 2. How likely is it to need resizing?
# 3. What are the operational practices?

# Document your recommendation and why
```

---

## Challenge: Disk Monitoring Script

**Create a script that monitors disk health:**

```bash
#!/bin/bash

echo "=== Disk Health Report ==="
echo ""

echo "1. Block Device Summary:"
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

echo ""
echo "2. RAID Status:"
sudo cat /proc/mdstat 2>/dev/null || echo "No software RAID detected"

echo ""
echo "3. Disk Health (SMART):"
for disk in /dev/sd{a,b,c,d}; do
    if [ -e "$disk" ]; then
        echo "=== $disk ==="
        sudo smartctl -H "$disk" 2>/dev/null | grep -A1 "SMART overall"
    fi
done

echo ""
echo "4. Partition Table:"
sudo fdisk -l /dev/sda 2>/dev/null | grep "Disklabel\|Device\|Boot"

echo ""
echo "5. Alert Summary:"
echo "- Check for any FAILED SMART status above"
echo "- Check for any degraded RAID arrays"
echo "- Check for unallocated disk space"
```

**Your task:**
1. Create this script
2. Run it to generate a baseline report
3. Schedule to run monthly
4. Set up alerts for failures

---

## Self-Check

After completing all problems:

- [ ] Problem 1: Analyzed block devices and partitions
- [ ] Problem 2: Checked RAID arrays
- [ ] Problem 3: Evaluated disk health
- [ ] Problem 4: Created replacement plan for failing disk
- [ ] Problem 5: Calculated capacity and timeline
- [ ] Problem 6: Evaluated partition vs LVM strategy
- [ ] Challenge: Created monitoring script

---

## Key Skills You Practiced

- `lsblk` — View block device layout
- `sudo cat /proc/mdstat` — RAID status
- `sudo smartctl` — Disk health monitoring
- Capacity planning math
- Risk assessment for storage failures

---

## Next Steps

Compare your answers with `week2_solutions.md`.

Focus on:
1. Regular SMART monitoring is essential
2. RAID 6 is safer than RAID 5 for large drives
3. Capacity monitoring with headroom is critical

---

Powered by UQS v1.8.5-C
