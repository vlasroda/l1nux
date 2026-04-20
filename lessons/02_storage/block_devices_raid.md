# Lesson 2.2: Block Devices and RAID (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Storage systems are built on block devices and RAID. Understanding this layer is essential for diagnosing storage issues
**Skills:** View block devices, understand RAID levels, check RAID status, diagnose failing drives

---

## Quick Review

### The Storage Stack

```
Application (your code)
    ↓
Filesystem (ext4, XFS, etc.)
    ↓
Block Device Layer (RAID, LVM, partitions)
    ↓
Hardware (disks, SSDs, controllers)
```

Most issues originate at the hardware/RAID level but show up as filesystem problems.

---

## Key Concepts

### 1. Block Devices

**What is a block device?**
A device that stores data in fixed-size blocks (typically 4KB), accessed by block number rather than sequentially.

```
Block device: /dev/sda (entire disk)
├── Partition 1: /dev/sda1 (partition on sda)
├── Partition 2: /dev/sda2
└── Partition 3: /dev/sda3

RAID array: /dev/md0 (virtual block device made from multiple disks)
```

**Types of block devices:**
```
/dev/sda          - Physical disk
/dev/sda1         - Partition on disk
/dev/sdb          - Another disk
/dev/md0          - Software RAID array
/dev/nvme0n1      - NVMe SSD
/dev/nvme0n1p1    - Partition on NVMe
/dev/vda          - Virtual block device (VM)
```

### 2. Partitions

**Partition table types:**

**MBR (Master Boot Record)** — Old standard
```
Limitations:
  - Max 4 primary partitions (or 3 + extended)
  - Max 2TB per disk
  - No UUID support

Still used for: Legacy systems, boot partitions
```

**GPT (GUID Partition Table)** — Modern standard
```
Advantages:
  - 128+ partitions per disk
  - Disks larger than 2TB
  - UUID for each partition
  - Checksums for data integrity

Use for: Any new system, large drives
```

**View partitions:**
```bash
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0  100G  0 disk
├─sda1    8:1    0    1G  0 part /boot
├─sda2    8:2    0   99G  0 part /
nvme0n1 259:0    0 500G  0 disk
└─nvme0n1p1 259:1  0 500G  0 part /var
```

### 3. RAID Levels

**RAID 0 (Striping)**
```
Data split across multiple disks
Disks: [A1 A3] [A2 A4]
Performance: Excellent (parallel reads/writes)
Redundancy: NONE - if any disk fails, entire array lost
Use case: Caches, temp storage (not critical data)
```

**RAID 1 (Mirroring)**
```
Exact copy on two disks
Disk 1: [A1 A2 A3]
Disk 2: [A1 A2 A3] (identical)
Performance: Good reads (both disks), writes slower (2 writes)
Redundancy: High - survives 1 disk failure
Use case: OS disks, critical databases
```

**RAID 5 (Striping with Parity)**
```
Data striped across 3+ disks, parity on all
Disks: [A1 P1] [A2 P2] [A3 P3]
       Where P = parity (allows reconstruction if 1 disk lost)
Performance: Good (multiple disks), parity writes cost
Redundancy: Medium - survives 1 disk failure
Capacity: 66% usable (1/3 for parity)
Recovery time: Hours to days (rebuilding with heavy I/O)
Use case: General-purpose storage arrays
```

**RAID 6 (Dual Parity)**
```
Like RAID 5, but 2 parity blocks
Survives: 2 simultaneous disk failures
Capacity: 50% usable (2/3 for parity)
Recovery time: Slower than RAID 5 (more calculations)
Use case: Large capacity drives (lower rebuild risk)
```

**RAID 10 (1+0, Mirrored Stripe)**
```
Striped mirrors: stripe across multiple 2-disk mirrors
Disk 1: [A1 A3] ---|
Disk 2: [A1 A3] ---|  (Mirror pair)
Disk 3: [A2 A4] ---|
Disk 4: [A2 A4] ---|  (Mirror pair)
Performance: Excellent (stripe + mirror benefits)
Redundancy: High - survives 1 failure per mirror
Capacity: 50% usable
Use case: High-performance arrays (databases, storage clusters)
```

### 4. RAID Recovery

**Rebuild Process:**
```
Disk fails
↓
Replace with new disk
↓
RAID controller rebuilds parity from remaining disks
↓
Data recovered from parity
↓
Normal operation

Time: Hours to days depending on disk size
Risk: During rebuild, if another disk fails, data may be lost
↑ This is why RAID 6 (dual parity) is common for large arrays
```

**URE (Unrecoverable Read Error):**
```
Risk during RAID rebuild: One in ~10^15 bits (typical for large disks)

Scenario:
- 4TB disk fails, triggers rebuild
- Controller reads all 3TB of data from other disks
- Chance of URE: ~4TB × 8 bits/byte / 10^15 = ~3% risk
- If URE occurs during rebuild, data lost permanently

Solution: RAID 6 (dual parity) + good disk health monitoring
```

### 5. Software vs Hardware RAID

**Software RAID (md, dmraid, LVM):**
```
Managed by: Linux kernel (md = multiple disk)
Advantages:
  - Platform independent (take disks to any Linux system)
  - No special controller needed
  - Full visibility into RAID status

Disadvantages:
  - CPU overhead (checksums, parity calculations)
  - Slower than hardware RAID
  - If OS doesn't load, can't recover disks

Use case: General-purpose servers, smaller arrays
```

**Hardware RAID (RAID controller in disk enclosure):**
```
Managed by: Dedicated RAID controller
Advantages:
  - Fast (dedicated hardware for parity)
  - Survives OS failure
  - Hot spares manage automatic replacement

Disadvantages:
  - Proprietary (must use same controller brand)
  - More expensive
  - Less visibility into details

Use case: Enterprise storage systems, DDN systems
```

---

## Essential Commands

### View Block Devices

```bash
# Simple block device list
lsblk                      # Tree view of devices
lsblk -o NAME,SIZE,TYPE    # Custom columns

# Detailed device info
sudo fdisk -l              # All partitions and sizes
blkid                      # UUID and filesystem info
sudo lsblk -f              # Filesystem info with lsblk

# Example output:
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0  100G  0 disk
├─sda1    8:1    0    1G  0 part /boot
├─sda2    8:2    0   99G  0 part /
sdb       8:16   0  500G  0 disk
└─sdb1    8:17   0  500G  0 part /data
```

### Check RAID Status

```bash
# Show RAID arrays
sudo cat /proc/mdstat         # All md RAID arrays

# Example output:
$ cat /proc/mdstat
Personalities : [raid1] [raid6] [raid5] [raid4]
md0 : active raid5 sda1[0] sdb1[1] sdc1[2]
      1048576 blocks super 1.2 level 5, 64k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/2 pages [0 bytes], 0KB in-use

md127 : active raid1 sdd1[0] sde1[1]
       524288 blocks super 1.2 [2/2] [UU]
       resync=DELAYED

# Interpretation:
# [UUU] = 3 disks, all Up
# [U_U] = 3 disks, one Failed (missing)
# resync=DELAYED = rebuild not started yet
```

**Detailed RAID array info:**
```bash
# Check specific RAID array
sudo mdadm --detail /dev/md0

# Example output:
$ mdadm --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : Jan 19 2024 10:00:00 UTC
     Raid Level : raid5
     Array UUID : abc123def456:abc123def456
       Raid Devices : 3
      Total Devices : 3
        Persistence : Superblock is persistent

        Update Time : Jan 19 2024 11:30:00 UTC
              State : clean
     Active Devices : 3
    Working Devices : 3
     Failed Devices : 0
      Spare Devices : 0

Consistency Policy : resync

           Name : host:0
           UUID : abc123def456:abc123def456
         Events : 1234

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda1
       1       8       16        1      active sync   /dev/sdb1
       2       8       32        2      active sync   /dev/sdc1
```

### Check Disk Health

```bash
# Check SMART status
sudo smartctl -a /dev/sda        # All SMART info
sudo smartctl -H /dev/sda         # Quick health check

# Monitor for errors
sudo smartctl -t short /dev/sda   # Run quick self-test
sudo smartctl -t long /dev/sda    # Run extended test

# Watch test progress
sudo smartctl -a /dev/sda | grep "Self-test execution"
```

### Diagnose Failing Disk

```bash
# Check for read errors in kernel logs
dmesg | grep -i "ata\|sata\|error"

# Check system journal
journalctl -p err --since "1 hour ago" | grep -i "disk\|error"

# Monitor real-time I/O errors
sudo watch -n 1 'cat /proc/diskstats | grep sda'

# Check RAID array for issues
sudo cat /proc/mdstat
# Look for [U_U] pattern (failed device)
```

### Replace Failed Disk (Software RAID)

```bash
# Identify failed device from /proc/mdstat output
# Say we have: md0 : active raid5 sda1[0] sdb1[1] sdc1[F]
# (sdc1 shows [F] = failed)

# Mark as failed
sudo mdadm --manage /dev/md0 --fail /dev/sdc1

# Remove from array
sudo mdadm --manage /dev/md0 --remove /dev/sdc1

# (Physically replace disk)

# Add new disk to array
sudo mdadm --manage /dev/md0 --add /dev/sdc1

# Check rebuild progress
watch -n 1 'cat /proc/mdstat'
```

---

## Interactive Exercise: Diagnose RAID Status

**Task 1: View your block devices**
```bash
lsblk

# Note:
# - How many disks?
# - Are they partitioned?
# - What's mounted where?
```

**Task 2: Check for RAID arrays**
```bash
sudo cat /proc/mdstat

# If output shows md devices:
# - Note the level (raid0, raid1, raid5, raid6)
# - Check [UUU] status (all up)
```

**Task 3: Get filesystem info**
```bash
sudo lsblk -f

# Shows filesystem type for each device
# Note which use ext4 vs XFS vs other
```

**Task 4: Simulate a disk failure scenario**
```bash
# Create a test file to show what happens
dd if=/dev/zero of=/tmp/disk_test bs=1M count=100

# Check I/O performance before and after
# (We'll do more I/O testing in Week 4)
```

**Task 5: Check disk health (if available)**
```bash
# If SMART support available
sudo smartctl -H /dev/sda

# Expected output:
# === START OF READ SMART DATA SECTION ===
# SMART overall-health self-assessment test result: PASSED
```

---

## Challenge Questions

**Answer without looking them up:**

1. **RAID Level Choice:** You're designing a storage array for a video production company. Large sequential writes are critical. Performance > redundancy. Which RAID level?
   ```
   A) RAID 0 (Striping)
   B) RAID 1 (Mirroring)
   C) RAID 5 (Striping + Parity)
   D) RAID 6 (Dual Parity)
   ```

2. **Rebuild Risk:** Why is RAID 6 preferred over RAID 5 for large modern disks?
   ```
   (Explain in 2-3 sentences)
   ```

3. **RAID 5 Capacity:** You have 4 disks of 1TB each in RAID 5. What's the usable capacity?
   ```
   A) 4TB
   B) 3TB
   C) 2TB
   D) 2.5TB
   ```

4. **Failed Disk Detection:** You see this in /proc/mdstat:
   ```
   md0 : active raid5 sda1[0] sdb1[F] sdc1[2]
   ```
   What does [F] mean and what should you do?
   ```
   (Explain and suggest action)
   ```

5. **Partition Table:** You need to create a 10TB filesystem. Should you use MBR or GPT?
   ```
   A) MBR (more stable)
   B) GPT (supports > 2TB)
   C) Either works the same
   D) Depends on filesystem type
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Storage array degraded"

```bash
# Check RAID status
sudo cat /proc/mdstat

# Output shows:
# md0 : active raid5 sda1[0] sdb1[F] sdc1[2]
# [F] = failed disk

# Action:
# 1. Alert on-call team
# 2. Check SMART status of failing disk
sudo smartctl -H /dev/sdb

# 3. If failing, prepare replacement
# 4. Don't wait - rebuild during failure is risky
```

### Scenario 2: "High I/O latency on one disk"

```bash
# Check I/O stats
iostat -x 1 5

# Look for one disk with high await time
# Disk1: await=5ms (normal)
# Disk2: await=150ms (slow!)

# Suspect:
# - Disk failing (check SMART)
# - Disk overloaded (check load)
# - Cache not working (check controller logs)
```

### Scenario 3: "Can't add new partition"

```bash
# Check partition table type
sudo fdisk -l /dev/sda | grep "Disklabel"

# If "Disklabel type: dos" (MBR)
# and you have 4 partitions:
# Can't add more unless using extended partition

# Solution:
# 1. Backup data
# 2. Convert to GPT
# 3. Repartition

sudo gdisk /dev/sda
# (interactive, complex - usually done during maintenance)
```

### Scenario 4: "RAID rebuild taking forever"

```bash
# Check rebuild progress
cat /proc/mdstat

# Output shows:
# md0 : active raid5 sda1[0] sdb1[1] sdc1[2]
#       1048576 blocks super 1.2 level 5, 64k chunk, algorithm 2 [3/3] [UUU]
#       resync = 45.2% (473858/1048576) finish=87.3min speed=1234K/sec

# Normal for large disks
# Time estimate: Hours

# DON'T interrupt - let it complete
# Interruption = data loss risk
```

### Scenario 5: "Storage cluster performance drop"

```bash
# If using hardware RAID:
# 1. Check controller logs
# 2. Check if rebuild happening
# 3. Check if all disks present

# If using software RAID (md):
cat /proc/mdstat

# If rebuild happening:
# Performance will suffer during rebuild
# This is expected
```

---

## Key Takeaways

1. **RAID level determines redundancy vs performance** — choose wisely based on workload
2. **RAID rebuild is risky** — especially RAID 5 with large disks (URE risk)
3. **RAID 6 is safer** — dual parity for large disk arrays
4. **Monitor disk health proactively** — SMART can predict failures
5. **Failed disk replacement** — must be done quickly to restore redundancy
6. **Partition table matters** — use GPT for modern systems
7. **Software vs hardware RAID** — software is flexible, hardware is fast

---

## How Ready Are You?

Can you explain these?
- [ ] The difference between RAID 0, 1, 5, 6
- [ ] Why RAID 6 is preferred for large disks
- [ ] What [UUU] vs [U_U] means in /proc/mdstat
- [ ] How to check disk health
- [ ] The difference between MBR and GPT

If you checked all boxes, you're ready for Lesson 2.3.

---

Powered by UQS v1.8.5-C
