# Lesson 2.3: LVM and Storage Monitoring (Refresher)

**Time:** 25-30 minutes
**Why it matters:** LVM adds flexibility to storage management; monitoring prevents full disks and performance issues
**Skills:** Understand LVM structure, manage logical volumes, monitor storage utilization and performance

---

## Quick Review

### The Problem LVM Solves

**Before LVM:**
```
Physical disks: /dev/sda (100GB), /dev/sdb (100GB)
Partition as: /dev/sda1 (100GB for /)
Result: Only /dev/sda is available, /dev/sdb unused
Problem: If /dev/sda runs out of space, stuck (can't add more)
```

**With LVM:**
```
Physical volumes: /dev/sda, /dev/sdb
Volume group: vg0 (combines both = 200GB total)
Logical volumes:
  - lv_root: 100GB (mounted as /)
  - lv_data: 100GB (mounted as /data)
Flexibility: Can shrink lv_data and grow lv_root if needed
```

---

## Key Concepts

### 1. LVM Architecture

**Three layers:**

```
Physical Volumes (PV) - Physical disks/partitions
    /dev/sda1  /dev/sdb1  /dev/sdc1
         |          |          |
         └──────────┬──────────┘
                    ↓
          Volume Group (VG) - Container
              vg0 (300GB total)
                    |
          ┌─────────┼─────────┐
          ↓         ↓         ↓
    Logical Volumes (LV) - Virtual disks
    lv_root   lv_data  lv_backup
    (100GB)   (100GB)   (100GB)
     |          |        |
    /          /data    /backup
```

**Terminology:**
- **PV (Physical Volume):** /dev/sda1, /dev/sdb1 (raw disk devices)
- **VG (Volume Group):** Pool of storage from multiple PVs
- **LV (Logical Volume):** Virtual disk carved from VG (acts like /dev/sda1)
- **Extent:** Smallest unit of allocation (default 4MB)

### 2. Common LVM Operations

**View LVM structure:**
```bash
# Physical volumes
sudo pvs                    # Summary
sudo pvdisplay              # Detailed

# Volume groups
sudo vgs                    # Summary
sudo vgdisplay              # Detailed

# Logical volumes
sudo lvs                    # Summary
sudo lvdisplay              # Detailed

# All together
sudo lvm dmesg              # Combined view (if available)
```

**Create LVM (typical setup):**
```bash
# 1. Initialize physical volume
sudo pvcreate /dev/sdb1

# 2. Create volume group
sudo vgcreate vg0 /dev/sdb1

# 3. Create logical volume
sudo lvcreate -L 100G -n lv_data vg0

# 4. Format and mount
sudo mkfs.ext4 /dev/vg0/lv_data
sudo mount /dev/vg0/lv_data /data
```

**Resize logical volume (key feature):**
```bash
# Grow a logical volume
sudo lvextend -L +50G /dev/vg0/lv_data

# Resize filesystem to match
sudo resize2fs /dev/vg0/lv_data    # ext4
sudo xfs_growfs /dev/vg0/lv_data   # XFS

# Shrink (dangerous - backup first!)
sudo resize2fs /dev/vg0/lv_data 20G   # First shrink FS
sudo lvreduce -L 20G /dev/vg0/lv_data # Then shrink LV
```

**Add disk to volume group:**
```bash
# Create PV on new disk
sudo pvcreate /dev/sdc

# Add to existing VG
sudo vgextend vg0 /dev/sdc

# Now vg0 has more space to allocate
```

### 3. Snapshots (Powerful Feature)

**What is a snapshot?**
A read-write copy of a logical volume at a point in time. Allows backing up without stopping services.

```bash
# Create snapshot
sudo lvcreate -L 10G -s -n lv_data_backup /dev/vg0/lv_data

# Now you have:
# /dev/vg0/lv_data (original, still in use)
# /dev/vg0/lv_data_backup (snapshot, fixed point in time)

# Mount and backup
sudo mkdir /mnt/backup
sudo mount /dev/vg0/lv_data_backup /mnt/backup
tar czf /backup/data.tar.gz /mnt/backup/

# Remove snapshot
sudo lvremove /dev/vg0/lv_data_backup
```

### 4. Storage Monitoring

**Key metrics:**
- **Capacity:** How full is the disk/filesystem?
- **Performance:** How fast is I/O? How many IOPS?
- **Health:** Any errors, failing disks, RAID degradation?

**Capacity monitoring (the basics):**
```bash
# Used space
df -h                       # Per filesystem
du -sh /var                 # Directory size
du -sh /var/*               # Per-subdirectory

# Inode usage
df -i

# LVM space
sudo lvs                    # LV sizes
sudo vgs                    # VG free space
```

**Thresholds to watch:**
```
Disk usage:
  < 70%: OK
  70-85%: Watch (but not urgent)
  85-95%: Urgent (start cleanup)
  > 95%: Critical (operations may fail)

Inode usage:
  < 80%: OK
  80-95%: Getting tight
  > 95%: Problem
```

---

## Essential Commands

### View LVM Setup

```bash
# Quick status
sudo lvs -o lv_name,lv_size,vg_name

# Detailed info
sudo lvdisplay

# Example output:
$ lvs
  LV         VG   Attr       LSize   Pool Origin Data%
  lv_data    vg0  -wi-ao---- 100.00g
  lv_root    vg0  -wi-ao----  50.00g
  lv_backup  vg0  swi-a-----  10.00g lv_data 0.12

# Interpretation:
# swi = snapshot (s=snapshot, w=writeable, i=inherit)
# -wi = regular (w=writeable, i=inherited)
# Attr[4] = 'a' = active (can be mounted)
```

### Grow Filesystem

```bash
# Check current size
df -h /data
# /dev/mapper/vg0-lv_data   100G   50G  50G  50%

# Extend LV
sudo lvextend -L +50G /dev/vg0/lv_data

# Grow filesystem (online, no unmount needed)
sudo resize2fs /dev/vg0/lv_data     # ext4
# or
sudo xfs_growfs /dev/vg0/lv_data    # XFS

# Verify
df -h /data
# /dev/mapper/vg0-lv_data   150G   50G 100G  33%
```

### Monitor Disk Usage

```bash
# Simple status
df -h                       # Used vs available
df -i                       # Inode usage

# Watch for changes
watch -n 10 'df -h'         # Refresh every 10 seconds

# Historical data (if available)
# Usually via monitoring systems (prometheus, grafana, etc.)
```

### Monitor I/O Performance

```bash
# I/O statistics
iostat -x 1 5               # 5 samples, 1 sec intervals
iotop                       # Real-time I/O by process
vmstat 1 5                  # Virtual memory + I/O

# What to look for in iostat:
# - %util: How busy is the disk (>80% = saturated)
# - await: Average wait time per I/O
# - svctm: Service time per operation
```

### Troubleshoot Full Disk

```bash
# Find what's taking space
du -sh /*                   # Top-level directories
du -sh /var/*               # /var contents
du -sh /home/*              # User home directories

# Find large files
find / -type f -size +100M  # Files larger than 100M
find / -type f -size +1G    # Files larger than 1GB

# Common culprits:
# /var/log/ - Log files (clean up old ones)
# /tmp/ - Temporary files
# /var/cache/ - Cache files
```

---

## Interactive Exercise: Analyze Storage

**Task 1: Check current storage layout**
```bash
# List filesystems
df -h

# Check inode usage
df -i

# Check LVM (if in use)
sudo lvs 2>/dev/null
sudo vgs 2>/dev/null
```

**Task 2: Find space usage**
```bash
# Top directories
du -sh /*

# Look for unexpected large directories
# Compare /var, /home, /usr sizes
```

**Task 3: Identify growth trends**
```bash
# What's in /var?
du -sh /var/*

# Usually large:
# /var/log - if not rotated
# /var/cache - if not cleaned

# What's in /tmp?
ls -lh /tmp | head -20
# Should be relatively empty
```

**Task 4: Check performance**
```bash
# One-time snapshot
iostat -x 1 3

# Look at:
# - %util: Disk utilization
# - await: Average I/O latency
```

**Task 5: Simulate cleanup scenario**
```bash
# Check log file sizes
du -sh /var/log/*

# Find old logs (if not rotated)
find /var/log -type f -mtime +30

# Don't delete without understanding what you're deleting!
```

---

## Challenge Questions

**Answer without looking them up:**

1. **LVM Advantage:** Why would you use LVM instead of traditional partitioning?
   ```
   A) Better performance
   B) Allows resizing filesystems without repartitioning
   C) Uses less disk space
   D) Faster I/O operations
   ```

2. **Snapshot Use:** You create a snapshot of /dev/vg0/lv_data while it's mounted and in use. The snapshot:
   ```
   A) Pauses the original LV until snapshot is done
   B) Makes a copy at that point in time (CoW)
   C) Locks the filesystem
   D) Fails because LV is in use
   ```

3. **Disk Full:** You have 5GB free in VG but LV is full (100GB allocated, all used). Can you grow the LV?
   ```
   A) No, VG doesn't have space
   B) Yes, can extend to 105GB
   C) Need to add more physical volumes
   D) Need to shrink another LV
   ```

4. **Monitoring:** You see:
   ```
   Filesystem           Size Used Avail Use%
   /dev/mapper/vg0-lv  100G  95G  5G  95%
   ```
   What's your next step?
   ```
   (Explain action to take)
   ```

5. **Grow Filesystem:** After `lvextend -L +50G`, you do `resize2fs`. Why two steps?
   ```
   (Explain the purpose of each)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Filesystem is full"

```bash
# Quick check
df -h
# /dev/mapper/vg0-lv_data 100G 100G 0G 100%

# Solution depends on setup:

# Option 1: Has LVM? Extend it
sudo lvextend -L +50G /dev/vg0/lv_data
sudo resize2fs /dev/vg0/lv_data

# Option 2: No LVM? Find and clean
du -sh /*
du -sh /var/*
find /var/log -name "*.log" -mtime +30 -delete

# Option 3: Urgent? Delete large files carefully
find / -type f -size +1G -exec rm {} \;
# (Only if you're sure what you're deleting!)
```

### Scenario 2: "Storage array is running out of space"

```bash
# Check VG free space
sudo vgs
# VG0: 500G total, 400G used, 100G free

# Action plan:
# 1. Identify largest LVs
sudo lvs -o +lv_size | sort -k4 -h

# 2. Check what's in them
sudo mount | grep lvm
# Mount snapshot and inspect if needed

# 3. Clean up inside LVs
du -sh /data/*          # if /data is on LV

# 4. Grow underutilized LVs if others are full
sudo lvextend -L +10G /dev/vg0/lv_full
sudo resize2fs /dev/vg0/lv_full

# 5. Long-term: Add more physical volumes
# (requires downtime or hot addition if supported)
```

### Scenario 3: "I/O is slow, which disk?"

```bash
# Monitor I/O per device
iostat -x 1 5

# If seeing:
# Device: %util=95, await=250ms
# That device is a bottleneck

# Check what's hitting it
iotop
# or
strace -p <pid> (process causing I/O)

# Solution:
# - Add faster disk
# - Rebalance data to different LVs
# - Check if writes can be batched
```

### Scenario 4: "Need to backup active filesystem"

```bash
# Use LVM snapshots (safest method)

# Create snapshot
sudo lvcreate -L 20G -s -n lv_data_snap /dev/vg0/lv_data

# Mount and backup
sudo mount /dev/vg0/lv_data_snap /mnt/snap
tar czf /backup/data_$(date +%Y%m%d).tar.gz /mnt/snap

# Cleanup
sudo umount /mnt/snap
sudo lvremove /dev/vg0/lv_data_snap

# Key advantage: Original LV keeps running, no interruption
```

### Scenario 5: "Add new storage to system"

```bash
# Steps:
# 1. Hot-add disk (if system supports)
# 2. Create physical volume
sudo pvcreate /dev/sdd

# 3. Add to volume group
sudo vgextend vg0 /dev/sdd

# 4. Extend existing LV
sudo lvextend -L +100G /dev/vg0/lv_data

# 5. Grow filesystem
sudo resize2fs /dev/vg0/lv_data

# No downtime needed if hot-addition supported!
```

---

## Key Takeaways

1. **LVM adds flexibility** — can resize without repartitioning
2. **Snapshots are powerful** — backup running systems safely
3. **Monitor capacity closely** — 85%+ is warning sign
4. **Watch I/O performance** — slow disks affect everything
5. **Inode exhaustion is real** — even with space available
6. **Two-step resize** — lvextend then resize2fs/xfs_growfs
7. **Cleanup strategies** — know where space goes (/var/log, /tmp common culprits)

---

## How Ready Are You?

Can you explain these?
- [ ] The three layers of LVM (PV, VG, LV)
- [ ] How to extend an LV and grow the filesystem
- [ ] Why you'd use LVM snapshots
- [ ] How to find what's filling up a disk
- [ ] The difference between capacity and inode usage

If you checked all boxes, you're ready for Week 2 exercises.

---

Powered by UQS v1.8.5-C
