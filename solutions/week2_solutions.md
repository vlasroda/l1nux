# Week 2 Solutions: Storage and Filesystems

---

## Lesson 2.1: Filesystem Concepts

### Challenge Questions Answers

**1. Inode Depletion**
```
Answer: B) Ran out of inodes

Explanation:
You have 50GB free space but can't create new files.
This happens when:
- Filesystem has limited inode count (set at mkfs time)
- Millions of small files were created, each using one inode
- No more inodes available, even though blocks are free

Solution:
- Find and delete small files
- Archive old data
- On next filesystem: create with larger inode count
```

**2. Journaling Protection**
```
Answer: Journaling protects against data loss by:
1. Writing all changes to a journal first (safe zone)
2. Only after journaling: writing actual data to disk
3. If crash occurs: replay journal entries after reboot

Without journaling:
- Crash during write: file metadata and data mismatch
- Filesystem inconsistent, needs long fsck
- Possible data loss

With journaling:
- Crash during write: journal replayed automatically
- Filesystem consistent
- No fsck needed
```

**3. Filesystem Choice**
```
Answer: XFS is better for this scenario

Reasons:
- 10,000 large video files = large sequential I/O
- XFS optimized for large files
- Better performance for big files than ext4
- Less fragmentation with large files

ext4 would work too, but:
- Slower for large sequential writes
- More prone to fragmentation with large files

Note: For a storage array like DDN uses, Lustre (built on XFS) is ideal
```

**4. Hard Links**
```
Answer: A) 1 copy (3 links point to same inode)

Explanation:
Hard link count = 3 means:
- One inode (one actual file on disk)
- Three filenames point to that inode
- Deleting one filename doesn't delete file (inode count becomes 2)
- Only when all links are deleted is inode freed

Example:
$ ln myfile myfile_copy1      # Hard link
$ ln myfile myfile_copy2      # Another hard link
$ ls -l
-rw-r--r-- 3 alice staff 1024 myfile        (link count = 3)
-rw-r--r-- 3 alice staff 1024 myfile_copy1 (same inode, count = 3)
-rw-r--r-- 3 alice staff 1024 myfile_copy2 (same inode, count = 3)

$ rm myfile       # Link count becomes 2
$ rm myfile_copy1 # Link count becomes 1
$ rm myfile_copy2 # Link count becomes 0, inode freed
```

**5. NFS Performance Issues**
```
Answer: Two main reasons NFS is slow:

1. Network latency:
   - Latency = round-trip time (typically 1-10ms per operation)
   - Each file read/write = network call + wait
   - SSD IOPS = 10,000+, network = 1,000s
   - Network is the bottleneck

2. Stateless protocol overhead (NFSv3):
   - Every operation must include all context
   - No state saved on server
   - More network traffic per operation
   - NFSv4 improved this with state, but still not local speed

For DDN/Lustre:
- Lustre is designed for high-latency networks with parallel striping
- Direct-attached storage = microsecond latency
- NFSv4 over Lustre improves performance but adds complexity
```

---

## Exercise 2.1: Filesystem Analysis

### Problem 1 Solution: Analyze Filesystems

**Task 1A: Expected findings**
```bash
$ df -h
Filesystem      Size Used Avail Use%
/dev/sda1       100G  50G  50G   50%
/dev/sdb1       500G 250G 250G   50%
tmpfs           8.0G   0   8.0G   0%

Analysis:
- 2 main filesystems (sda1, sdb1)
- tmpfs is RAM-based, always small
- Both at 50% (balanced)
- No urgency yet
```

**Task 1B: Filesystem Types**
```bash
$ sudo lsblk -f
NAME      FSTYPE
sda1      ext4       (traditional Linux filesystem)
sdb1      ext4       (or XFS, or other)
nvme0n1p1 ext4       (NVMe SSD)

Typical modern system uses ext4 or XFS
Some may still have older ext3 or btrfs
```

**Task 1C: Inode Usage**
```bash
$ df -i
Filesystem     Inodes IUsed IFree IUse%
/dev/sda1      1000000 500000 500000 50%
/dev/sdb1      4000000 1000000 3000000 25%

Analysis:
- sda1: 50% inode usage (OK)
- sdb1: 25% inode usage (very safe)
- No inode exhaustion risk

If any were 80%+:
- Would need investigation
- Likely millions of small files
```

---

### Problem 2 Solution: "Filesystem Full" Diagnosis

**Task 2A: Check Capacity**
```bash
$ df -h /var
Filesystem      Size Used Avail Use%
/dev/mapper/vg0-lv /var  100G 100G 0G 100%
```

**Task 2B: Check Inodes**
```bash
$ df -i /var
Filesystem     Inodes IUsed IFree IUse%
/dev/mapper/vg0-lv 1000000 500000 500000 50%

Analysis:
- Disk is 100% full (capacity issue)
- Inodes only 50% used (not inode exhaustion)
- PROBLEM: Out of space, not inodes
```

**Task 2C: Find Large Files**
```bash
$ du -sh /var/*
50G    /var/log      (logs!)
20G    /var/cache    (package cache)
15G    /var/spool    (mail queue)
10G    /var/lib      (application data)
5G     other
---
100G   total

Clear culprit: /var/log taking 50% of /var
```

**Task 2D: Find Old Files**
```bash
$ find /var/log -type f -mtime +30 | wc -l
2150    (files older than 30 days)

$ find /var/log -type f -mtime +30 -exec du -ch {} \; | tail -1
25G     (total size of old files)

ACTION: Archive or delete old logs
```

---

### Problem 3 Solution: Filesystem Performance

**Task 3A: Fragmentation Check**
```bash
$ sudo e4defrag -c /dev/sda1
/dev/sda1: 5.2% non-contiguous

Analysis:
- 5.2% fragmentation is good (< 10%)
- Performance not impacted
- No defragmentation needed
- This is typical for ext4 with modern systems
```

**Task 3B: I/O Performance Baseline**
```bash
$ iostat -x 1 3
Device: sda
     r/s    w/s   rMB/s  wMB/s %util await svctm r_await w_await
    150    50     75     10    45    5.2   2.1   4.5     7.2

Baseline interpretation:
- %util = 45 (healthy, not saturated)
- await = 5.2ms (reasonable)
- r/s = 150 (150 reads/sec)
- w/s = 50 (50 writes/sec)

If await were 100ms, disk would be slow
If %util were 95%, disk would be saturated
```

**Task 3C: Filesystem Block Info**
```bash
$ stat /
  File: /
  Size: 4096      Blocks: 8       IO Block: 4096
  Block size: 4096 bytes
  Fragment size: 4096 bytes
  Total blocks: 25000000
  Free blocks: 12500000

Block size = 4096 (standard)
Free blocks = 50% (good)
```

---

### Problem 4 Solution: Free Up Space

**Task 4A: Top Directories**
```bash
$ du -sh /* | sort -rh | head -10
30G    /var      (need cleanup)
20G    /usr      (usually don't touch)
15G    /home     (user data)
8G     /opt      (applications)
5G     /boot     (kernel images)

Focus on /var first
```

**Task 4B: Drill Down**
```bash
$ du -sh /var/* | sort -rh | head -10
50G    /var/log    (biggest culprit!)
20G    /var/cache
10G    /var/spool
...

$ du -sh /var/log/* | sort -rh
30G    syslog.log
15G    auth.log
5G     kernel.log

Plan: Archive or delete old logs
```

**Task 4C: Identify Safe Cleanup**
```bash
$ find /var/log -type f -mtime +60
/var/log/syslog.1.gz (rotated, old)
/var/log/auth.log.1.gz (rotated, old)
...

Safe to delete:
- Rotated and compressed logs (*.gz)
- Files not accessed in 60+ days
```

**Task 4D: Estimate Cleanup**
```bash
$ find /var/log -type f -mtime +30 -exec ls -lh {} \; | awk '{sum += $5} END {print "Total: " sum}'
Total: 25G

If you delete files 30+ days old, free 25GB
New usage: 100G - 25G = 75G (75% full, safe)
```

---

### Problem 5 Solution: Filesystem Limits

**Task 5A: Both Metrics Critical**
```bash
Filesystem /data:
- Disk: 100G total, 95G used (95%) = 5GB free
- Inodes: 1000000 total, 980000 used (98%) = 20000 free

Which runs out first?
- Space: At 5MB/day growth = 1000 days
- Inodes: At 50 inodes/day growth = 400 days

Inode exhaustion happens first!
```

**Task 5B: Prioritize Cleanup**
```bash
Since inodes are the bottleneck:

Check file count:
$ find /data -type f | wc -l
980000 files in /data

Find files to delete:
$ find /data -type f -size -1k | wc -l
500000 files under 1KB (likely logs or temp files)

These small files use inodes but take little space
Delete small files first to free inodes
```

**Task 5C: Cleanup Plan**
```bash
Cleanup plan for /data:
1. Find and compress small files
   find /data -type f -size -10k -mtime +30 -exec gzip {} \;

2. Find and delete old temp files
   find /data/tmp -type f -mtime +30 -delete

3. Archive to external storage
   tar czf /backup/archive_$(date +%Y%m%d).tar.gz /data/oldfiles/

4. Remove archives
   rm -rf /data/oldfiles/

Estimated cleanup:
- Delete 50000 small files (frees 50000 inodes)
- New inode usage: 980000 - 50000 = 930000 (93%)
- Still tight, consider extending

For long-term:
- Increase inodes (requires mkfs with -i option)
- Or add more filesystems
```

---

## Exercise 2.2: RAID and Disk Management

### Problem 1 Solution: Block Device Analysis

**Task 1A: Device Layout**
```bash
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0  500G  0 disk
├─sda1    8:1    0    1G  0 part /boot
├─sda2    8:2    0  499G  0 part
sdb       8:16   0  500G  0 disk
├─sdb1    8:17   0  500G  0 part
nvme0n1 259:0    0    1T  0 disk
└─nvme0n1p1 259:1 0 1000G 0 part /data

Analysis:
- 2 SATA drives (500GB each)
- 1 NVMe drive (1TB)
- Total: 2TB

Typical setup for storage server
```

**Task 1B: Filesystem Types**
```bash
$ sudo lsblk -f
NAME         FSTYPE
sda1         ext4       (/boot)
sda2         ext4       (root)
sdb1         ext4       (/data)
nvme0n1p1    xfs        (/data)

Mix of ext4 and xfs common
XFS often for large storage volumes
```

**Task 1C: Partition Table**
```bash
$ sudo fdisk -l /dev/sda | head -5
Disklabel type: gpt
Device     Boot   Start      End  Sectors Size
/dev/sda1          2048  2099199  2097152 1G
/dev/sda2      2099200 976773119 974673920 465G

Findings:
- Using GPT (modern, supports large partitions)
- 2 partitions on sda1, lots of unallocated space
- Could create more partitions if needed
```

---

### Problem 2 Solution: RAID Status

**Task 2A: RAID Detection**
```bash
$ sudo cat /proc/mdstat
Personalities : [raid1]
md0 : active raid1 sdb1[0] sdc1[1]
      524288 blocks super 1.2 [2/2] [UU]
      bitmap: 0/10 pages [0 KB], 0KB in-use

Analysis:
- RAID 1 array (mirrored)
- 2 disks, both up [UU]
- 512MB array size
- Clean state

Or if no RAID:
$ sudo cat /proc/mdstat
No md devices present.
(No software RAID configured)
```

**Task 2B: Detailed RAID Info**
```bash
$ sudo mdadm --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : Jan 1 2024
     Raid Level : raid1
     Array UUID : xxxxx
       Raid Devices : 2
      Total Devices : 2
     Active Devices : 2
    Failed Devices : 0
     Spare Devices : 0
           State : clean

    Number   Major   Minor   RaidDevice State
       0       8       17        0      active sync   /dev/sdb1
       1       8       33        1      active sync   /dev/sdc1

Status: Healthy, all devices active
```

**Task 2C: RAID Status Interpretation**
```
Status codes:
[UUU]  = 3 devices, all Up
[U_U]  = 3 devices, one Failed
[U__]  = 3 devices, two Failed
state: clean = no rebuild
state: recovering = rebuilding
state: degraded = missing disk

In this example: [UU] clean = Perfect health
```

---

### Problem 3 Solution: Disk Health

**Task 3A: SMART Status**
```bash
$ sudo smartctl -H /dev/sda
=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

Result: Disk is healthy
```

**Task 3B: SMART Details**
```bash
$ sudo smartctl -a /dev/sda | grep -A 20 "ID# ATTRIBUTE"
  1 Raw_Read_Error_Rate        0x000f   100   100   016    Pre-fail
  5 Reallocated_Sector_Ct      0x0033   100   100   010    Pre-fail
  9 Power_On_Hours             0x0032    99    99   000    Old_age
189 High_Fly_Writes           0x003a   100   100   001    Old_age
191 G-Sense_Error_Rate         0x001a   100   100   001    Old_age
192 Power-Off_Retract_Count    0x0032   100   100   000    Old_age
193 Load_Cycle_Count           0x0032    98    98   000    Old_age
194 Temperature_Celsius        0x0022    55    55   000    Old_age
196 Reallocated_Event_Count    0x0032   100   100   000    Old_age
197 Current_Pending_Sector     0x0012   100   100   000    Old_age
198 Offline_Uncorrectable      0x0010   100   100   000    Offline

Analysis:
- Reallocated_Sector_Ct = 0 (no bad sectors yet)
- Current_Pending_Sector = 0 (no pending)
- Offline_Uncorrectable = 0 (no uncorrectable reads)

All green = disk is healthy
```

**Task 3C: Kernel Error Logs**
```bash
$ journalctl -p err --since "24 hours ago" | grep -i "disk\|ata"
(empty)

Result: No disk errors in kernel logs
If showing:
[ata2.00]: status: { DRDY }
[ata2.00]: error: { UNC NCQ }

This would mean disk I/O errors
```

---

### Problem 4 Solution: Failing Disk Replacement

**Task 4A: Verify Problem Disk**
```bash
$ lsblk | grep sdb
sdb           8:16   0  500G  0 disk
├─sdb1        8:17   0  500G  0 part

$ sudo cat /proc/mdstat | grep sdb
md0 : active raid5 sda1[0] sdb1[1] sdc1[2]
(sdb1 is part of RAID5)

$ mount | grep sdb
/dev/mapper/vg0-lv_data on /data type ext4
(LVM on top of sdb1)

Documentation:
- sdb has 1 partition (sdb1)
- sdb1 is part of md0 RAID5 array
- RAID provides redundancy
- Data is critical (mounted as /data)
- Replacement needed ASAP but not emergency yet
```

**Task 4B: Replacement Plan**
```bash
Documented plan for sdb replacement:

Step 1: Prepare
- Verify backup of /data exists
- Check RAID status
- Alert team

Step 2: Failover
- Mark sdb1 as failed: sudo mdadm --fail /dev/md0 /dev/sdb1
- Remove from array: sudo mdadm --remove /dev/md0 /dev/sdb1
- Verify data still accessible: mount /data

Step 3: Physical replacement
- Power down server
- Replace sdb with identical disk
- Power on

Step 4: Rebuild
- Add disk to array: sudo mdadm --add /dev/md0 /dev/sdb1
- Watch rebuild: watch cat /proc/mdstat
- Rebuild time: ~4 hours for 500GB

Step 5: Verify
- Check RAID status: [UUU] clean
- Verify data: ls /data
```

**Task 4C: Backup Status**
```bash
Documentation:
- Critical data: /data (on sdb1)
- Backup location: /backup/data/
- Last backup: Today 10:00 AM
- Backup status: GOOD

If sdb completely failed:
- Data still accessible via RAID (2/3 disks working)
- Backup available as safety net
- Recovery possible, no data loss
```

---

### Problem 5 Solution: Capacity Planning

**Task 5A: Space Calculation**
```
Given:
- 10× 1TB drives in RAID 5
- RAID 5 usable capacity = (n-1) × disk_size = 9TB

Current:
- Used: 5TB
- Free: 9TB - 5TB = 4TB

At 100GB/month growth:
- Months until full = 4TB / 100GB = 40 months
- That's about 3.3 years

Timeline:
- Month 0: 5TB used (55%)
- Month 12: 6.2TB used (68%)
- Month 24: 7.4TB used (82%) - Getting urgent
- Month 28: 7.8TB used (87%) - Critical
- Month 32: 8.2TB used (91%) - Emergency
- Month 40: 9TB used (100%) - Out of space

Recommendation: Plan upgrade at month 24 (before 85% threshold)
```

**Task 5B: Rebuild Timeline**
```
With RAID 5:
- Array: 9TB total
- Rebuild time: ~30-50 hours (slow due to size)
- Rebuild stress: Very high I/O for 1-2 days

Risk during rebuild:
- If another disk fails during rebuild: TOTAL DATA LOSS
- Unrecoverable read error (URE) risk: ~3% for 1TB+ disks
- Solution: RAID 6 for large arrays (dual parity)

Planning:
- Keep array healthy with RAID 6 for large disks
- Don't operate degraded for extended periods
- Replace failing disks within 1-2 weeks
```

**Task 5C: Upgrade Timeline**
```
Action plan:

Month 20: Check trend
- Verify 100GB/month growth assumption
- Adjust if trend changed

Month 22-24: Plan upgrade
- Order new disks
- Verify capacity increase formula
- Test procedures

Month 24: Execute upgrade
Before: 10×1TB drives (9TB usable)
After: 10×2TB drives (18TB usable) OR add 5 new drives

Method 1: Replace all drives
- Time: ~1 week (5 drives/day replacement)
- Risk: Lower (replaces entire array)

Method 2: Add drives to array
- Time: 1-2 hours per drive
- Risk: Rebuild risk

Recommended: Replace all for safety
```

---

### Problem 6 Solution: Partition vs LVM

**Task 6A: Partitioning Evaluation**
```bash
Option 1: Traditional partitions

$ sudo fdisk /dev/sdc
(Create /dev/sdc1 and /dev/sdc2)

Pros:
- Simple, legacy systems understand it
- MBR: traditional, works everywhere
- GPT: modern, supports large disks

Cons:
- Fixed sizes (hard to change after creation)
- Limited partitions (4 primary or 3+extended)
- Not flexible for dynamic allocation
```

**Task 6B: LVM Evaluation**
```bash
Option 2: LVM (raw device)

$ sudo pvcreate /dev/sdc
$ sudo vgextend vg0 /dev/sdc

Pros:
- Flexible allocation (create/resize LVs dynamically)
- Easy to expand (add disk to VG)
- Snapshots for backup
- Online resizing

Cons:
- Requires LVM infrastructure
- Adds complexity (PV → VG → LV)
- Performance overhead (usually minimal)
```

**Task 6C: Recommendation**
```
Scenario analysis:

If system already uses LVM:
→ Recommend: Raw disk to LVM
   Reasons:
   - Consistent with existing infrastructure
   - Simplifies management (single VG for all storage)
   - Enables snapshot backups
   - Easy future resizing

If system uses traditional partitions:
→ Recommend: Partition disk (GPT)
   Reasons:
   - Consistent with existing setup
   - Simpler for admins familiar with fdisk
   - Works with existing scripts

If new system being designed:
→ Recommend: LVM everywhere
   Reasons:
   - More flexible long-term
   - Better for growth
   - Easier to manage multiple disks
   - Allows snapshots
```

---

## Exercise 2.3: LVM and Storage Monitoring

### Problem 1 Solution: LVM Analysis

**Task 1A: LVM Detection**
```bash
$ sudo lvs 2>/dev/null
  LV         VG   Attr       LSize   Pool Origin Data%
  lv_root    vg0  -wi-ao----  50.00g
  lv_home    vg0  -wi-ao---- 100.00g
  lv_data    vg1  -wi-ao---- 200.00g

Or if no LVM:
$ sudo lvs 2>/dev/null
bash: lvs: command not found
(Or no volumes listed)

LVM present: Yes
Configuration:
- 2 volume groups (vg0, vg1)
- 3 logical volumes
- Total: 350GB
```

**Task 1B: VG Space Status**
```bash
$ sudo vgs
  VG   #PV #LV #SN VSize  VFree
  vg0    2   2   0 300.00g 150.00g
  vg1    1   1   0 300.00g 100.00g

Documentation:
- vg0: 300GB total, 150GB free (50% full)
- vg1: 300GB total, 100GB free (66% full)
- vg1 has more pressure (less free space)
```

**Task 1C: Snapshot Check**
```bash
$ sudo lvs -o lv_name,lv_type,lv_size
  LV              Type     LSize
  lv_root         linear   50.00g
  lv_home         linear  100.00g
  lv_data         linear  200.00g
  lv_home_snap    snapshot 10.00g    <- This is snapshot

Or:
$ sudo lvs -o +origin
  LV              Origin
  lv_home_snap    lv_home

Documentation:
- One snapshot found: lv_home_snap
- Snapshot of lv_home
- Should be removed if backup complete
```

---

### Problem 2 Solution: Expand Logical Volume

**Task 2A: Current Size**
```bash
$ df -h /home
Filesystem                Size Used Avail Use%
/dev/mapper/vg0-lv_home  50G  43G   7G  86%

Interpretation:
- 50GB allocated to lv_home
- 43GB in use (86%)
- 7GB free
- Action needed: Expand
```

**Task 2B: VG Free Space**
```bash
$ sudo vgs vg0
  VG   VSize  VFree
  vg0 300.00g 100.00g

Interpretation:
- VG0 has 100GB free
- Can safely extend lv_home
```

**Task 2C: Expansion Plan**
```
Step-by-step plan:

1. Check free VG space:
   sudo vgs vg0
   → 100GB available

2. Extend LV by 30GB:
   sudo lvextend -L +30G /dev/vg0/lv_home

3. Grow filesystem to match:
   sudo resize2fs /dev/vg0/lv_home

4. Verify:
   df -h /home
   → Should show ~80GB total

Expected result:
- /home grows from 50GB to 80GB
- Users can store more files
- VG free space reduces from 100GB to 70GB
```

**Task 2D: Pre-Check**
```bash
$ sudo lsblk -f | grep lv_home
lv_home ext4   /home

Findings:
- ext4 filesystem ✓ (supports online resize)
- Mounted on /home ✓

$ mount | grep lv_home
/dev/mapper/vg0-lv_home on /home type ext4

Can resize while mounted: YES
No unmount needed: SAFE
```

---

### Problem 3 Solution: Storage Monitoring

**Task 3A: Baseline**
```bash
$ date > /tmp/storage_baseline.txt
$ df -h >> /tmp/storage_baseline.txt
$ cat /tmp/storage_baseline.txt

Results:
2024-01-19 11:30:00 UTC
Filesystem      Size Used Avail Use%
/dev/sda1       100G  50G  50G   50%
/dev/sdb1       500G 250G 250G   50%

Baseline established - save for monthly comparison
```

**Task 3B: Growth Analysis**
```bash
$ du -sh /var/* | sort -rh
30G    /var/log
20G    /var/cache
10G    /var/spool
...

Findings:
- /var/log is growing (probably not rotated)
- /var/cache could be cleaned
- Watch /var/log closely
```

**Task 3C: Growth Rate Calculation**
```
Hypothetical scenario:
- 30 days ago: /var/log = 10GB
- Today: /var/log = 15GB
- Growth: 5GB in 30 days = 166MB/day = ~5GB/month

Projection:
- /var filesystem: 100GB total, 85GB used
- Free: 15GB
- At 5GB/month growth: 15GB / 5GB = 3 months until full
- ALERT: Schedule cleanup within 1 month

Action:
- Set reminder for 1 month (before threshold)
- Plan log cleanup or compression
- Consider moving logs to different filesystem
```

**Task 3D: Cleanup Opportunities**
```bash
$ find /var/log -type f -mtime +30 | head -20
/var/log/syslog.2
/var/log/auth.log.1.gz
/var/log/kernel.log.5.gz
...

Safe to clean:
- All .gz files (already compressed)
- Files older than 60 days (keep ~2 months)
- Estimated cleanup: 10-15GB

$ ls -lh /tmp/ | tail -20
(Usually mostly empty)

$ du -sh /var/cache/
15G

Safe cleanup:
- apt cache: sudo apt-get clean (2-5GB)
- Build cache: Can usually delete
```

---

### Problem 4 Solution: Emergency Cleanup

**Task 4A: Identify Culprit**
```bash
$ du -sh /* | sort -rh
30G    /var

Zoom in:
$ du -sh /var/* | sort -rh
25G    /var/log
5G     /var/cache

Find large files:
$ find /var -type f -size +100M -exec ls -lh {} \; | sort -rh | head -10
1.5G   /var/log/app.log
1.2G   /var/log/syslog
800M   /var/log/auth.log
...

Clear culprit: Unrotated log files
```

**Task 4B: Prioritize Cleanup**
```
Rank by safety (safest first):

1. SAFEST - Definitely delete:
   - Compressed logs (*.gz) > 60 days old: 5GB
   - Old apt cache: 2GB
   - Old temp files: 1GB
   Total safe: 8GB

2. SAFE - Usually OK:
   - Uncompressed logs > 30 days: 10GB
   - Build artifacts: 2GB
   Total: 12GB

3. RISKY - Need to verify:
   - Application data: Unknown
   - User files: Unknown
   - System files: Dangerous

Action: Delete Level 1 (8GB) to bring below 90% full
```

**Task 4C: Execute Cleanup (CAREFUL!)**
```bash
# 1. Compress and archive old logs (safest)
cd /var/log
find . -name "*.log" -mtime +7 ! -name "*.gz" -exec gzip {} \;
# This gzip all non-compressed logs older than 7 days
# Reduces size significantly (usually 80% compression)
# Estimated freed: 5-10GB

# 2. Remove apt cache (safe)
sudo apt-get clean
# Removes downloaded packages
# Estimated freed: 1-2GB

# 3. Clean /tmp (safe)
find /tmp -atime +7 -type f -delete
# Remove files not accessed in 7 days
# Estimated freed: 0.5-1GB

# Total cleaned: ~6-13GB
# New usage: 100G - 8G = 92G (92% full, urgent but not critical)
```

**Task 4D: Document Results**
```
Cleanup Summary:
- Log compression: 8GB freed
- Apt cache clean: 1.5GB freed
- /tmp cleanup: 0.5GB freed
- Total freed: 10GB

Before: 100% full (out of space)
After: 90% full (still urgent)

Next action:
- Schedule aggressive log rotation
- Increase /var size or add disk
- Monitor growth going forward
```

---

### Problem 5 Solution: Snapshot Backup

**Task 5A: LVM Availability**
```bash
$ df /data | grep mapper
/dev/mapper/vg0-lv_data on /data type ext4

LVM is available: YES
LV name: lv_data
VG name: vg0
Current size: 50GB
```

**Task 5B: Snapshot Plan**
```
Snapshot considerations:

Current LV size: 50GB
Used space: 40GB
Free in LV: 10GB

Snapshot size:
- Rule: 10-20% of LV size
- For 50GB LV: 5-10GB snapshot
- Choose: 10GB (to be safe)

Metadata:
- Snapshots are copy-on-write
- Don't need entire LV size
- Only changes after snapshot are written to snapshot
- As long as 10GB snapshot space > changes during backup
```

**Task 5C: Backup Command**
```bash
#!/bin/bash
SNAP_NAME="lv_data_snap_$(date +%Y%m%d_%H%M%S)"
SNAP_SIZE="10G"
BACKUP_DIR="/backups"
LV="/dev/vg0/lv_data"

echo "Step 1: Create snapshot"
sudo lvcreate -L $SNAP_SIZE -s -n $SNAP_NAME $LV

echo "Step 2: Mount snapshot"
sudo mkdir -p /mnt/snap
sudo mount /dev/vg0/$SNAP_NAME /mnt/snap

echo "Step 3: Backup"
sudo tar czf $BACKUP_DIR/data_$(date +%Y%m%d).tar.gz /mnt/snap/

echo "Step 4: Cleanup"
sudo umount /mnt/snap
sudo lvremove -f /dev/vg0/$SNAP_NAME

echo "Backup complete"

Benefits of this approach:
- Original /data keeps running (no downtime)
- Snapshot is point-in-time (consistent backup)
- After unmount, snapshot auto-cleaned
- No performance impact on /data during backup
```

---

### Problem 6 Solution: Storage Dashboard

**Delivered earlier in Exercise 3**

The dashboard script provides:
1. Filesystem capacity with alerts
2. Inode usage tracking
3. LVM space status
4. Top space consumers
5. Critical alerts for >90% full
6. RAID health if present

**Monthly review checklist:**
```
□ Are any filesystems trending toward 90%?
□ Has inode usage changed significantly?
□ Have new LVs been created?
□ Any RAID issues detected?
□ Total storage consumption trending?
□ Time until next upgrade needed?
```

---

## Real-World Scenarios Summary

### Scenario 1: Inode Exhaustion
```
Problem: "Can't create files, disk has space"
Solution:
1. Identify: df -i shows 100% IUsed
2. Find culprit: find / -type f | wc -l
3. Clean: Archive or delete millions of small files
4. Prevention: Monitor df -i monthly
```

### Scenario 2: RAID Degradation
```
Problem: /proc/mdstat shows [U_U] (failed disk)
Solution:
1. Verify: mdadm --detail /dev/md0
2. Check: smartctl -H /dev/failed_disk
3. Replace: Mark failed, add replacement
4. Rebuild: Wait 24-48 hours for rebuild
5. Monitor: watch cat /proc/mdstat
```

### Scenario 3: Storage Full
```
Problem: Filesystem at 100%, applications failing
Solution:
1. Identify: du -sh /* | sort -rh
2. Cleanup: Archive logs, clean cache
3. Expand: lvextend + resize2fs (if LVM)
4. Prevention: Monitor with dashboard
```

### Scenario 4: Slow I/O
```
Problem: Disk I/O slow, affecting performance
Solution:
1. Baseline: iostat -x 1 5
2. Identify: iotop (what process?)
3. Check: Fragmentation, hot spots, RAID issues
4. Action: Defrag, rebalance, or upgrade hardware
```

---

Powered by UQS v1.8.5-C
