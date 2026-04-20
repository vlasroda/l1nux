# Lesson 2.1: Filesystem Concepts (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Storage system engineers need to understand filesystem behavior, performance characteristics, and how filesystems interact with hardware
**Skills:** Understand filesystem types, inode vs data blocks, filesystem performance factors, when to use which filesystem

---

## Quick Review

### What is a Filesystem?

A filesystem is the layer between files/directories (what users see) and blocks on disk (what hardware understands).

```
User: "Write hello.txt"
     ↓
Filesystem: "Let me find a free inode (for metadata), allocate data blocks (for content)"
     ↓
Disk: 32 blocks written at sectors 2048-2303
```

### Two Main Components

**1. Metadata (Inode):** Information about a file
- Permissions (755), owner (alice), group (staff)
- Size, modification time, access time
- Which data blocks contain the actual content
- Hard link count

**2. Data Blocks:** The actual file content
- Where the bytes of your file live
- Allocated independently from inodes

---

## Key Concepts

### 1. Filesystem Types and Their Characteristics

**ext4** (Extended 4)
```
Common on: Linux systems (default for many distros)
Strengths:
  - Stable, well-tested
  - Good performance for most workloads
  - Reliable metadata journaling
  - Supports up to 16TB filesystems

Weaknesses:
  - Not optimized for very high-concurrency
  - Max file size: 16TB

Use when: You need stability, not extreme performance
```

**XFS**
```
Common on: Enterprise Linux (RHEL, CentOS), especially with large storage
Strengths:
  - Excellent for large files
  - Very good concurrent I/O performance
  - Better scalability than ext4
  - Supports up to 8EB filesystems

Weaknesses:
  - Slower small-file operations
  - Recovery can take longer

Use when: Large sequential workloads, high concurrency
Note: Lustre storage systems often use XFS under the hood
```

**Btrfs** (B-tree filesystem)
```
Common on: Modern systems (Ubuntu, Fedora)
Strengths:
  - Copy-on-write (CoW) for snapshots
  - Subvolumes for hierarchical management
  - Built-in RAID support
  - Self-healing checksums

Weaknesses:
  - Still considered "maturing"
  - Recovery can be complex
  - Not as battle-tested as ext4/XFS

Use when: You need snapshots, subvolumes, or modern features
```

**NFS** (Network File System)
```
Not a real filesystem, but a network protocol
Characteristics:
  - Client mounts server's filesystem over network
  - Stateless (v3) or stateful (v4)
  - Performance depends on network and server
  - Good for shared access, poor for performance-critical

Use when: You need shared filesystem access (home dirs, projects)
Note: Slow I/O can cause processes to hang in 'D' state
```

**Lustre**
```
Special parallel filesystem for DDN storage
Characteristics:
  - Stripe data across multiple storage targets
  - Designed for high-throughput computing (HPC)
  - Can handle 1000s of clients
  - Built on MDTs (metadata targets) and OSTs (object storage targets)

Use when: You need massive parallel I/O (HPC cluster storage)
Note: This is the focus of Week 7
```

### 2. Inodes and Data Blocks

**Inode structure:**
```bash
$ stat /etc/passwd
  File: /etc/passwd
  Size: 2048      <- Data block size
  Blocks: 8       <- Number of 512B blocks used
  Inode: 654321   <- Inode number
  Links: 1        <- Hard link count
  Access: (0644/-rw-r--r--)
  Uid: (    0/  root)
  Gid: (    0/  root)
  Access: 2024-01-19 10:00:00.000000000
  Modify: 2024-01-19 10:00:00.000000000
  Change: 2024-01-19 10:00:00.000000000
```

**Key insight:** Each file has ONE inode, but can have multiple hard links. The inode is the real file; the filename is just a link to the inode.

**Inode depletion scenario:**
```bash
# If filesystem runs out of inodes before disk space
$ df -i
Filesystem     Inodes IUsed IFree IUse%
/dev/sda1     1000000 999999   1    100%

# Lots of free space, but can't create new files!
$ df -h
Filesystem Size Used Avail Use%
/dev/sda1  100G  50G  50G   50%

# This happens if you create millions of small files
```

### 3. Filesystem Performance Factors

**Throughput (MB/s):** How much data per second
- Sequential read: 200 MB/s typical for spinning disk
- Sequential write: depends on controller and disk
- Network filesystem: limited by network (1 Gbps ≈ 125 MB/s)

**Latency (ms):** Time to first byte
- Spinning disk seek: ~10ms
- SSD: ~0.1ms
- Network: ~1-10ms depending on network

**IOPS (I/O operations per second):**
- Spinning disk: ~100 IOPS (limited by seeks)
- SSD: ~10,000+ IOPS
- NFS: ~1,000-5,000 IOPS (network limited)

**Concurrency:** How well does it handle many simultaneous operations
- ext4: Good
- XFS: Better
- Lustre: Excellent (designed for it)

### 4. Journaling

Journaling protects against corruption during crashes.

```
Write to disk:
1. Write change to journal log (safe)
2. Write actual data to filesystem
3. Mark journal entry as complete
4. On reboot: replay any incomplete transactions

Overhead:
- Metadata journaling (ext4 default): ~5-10% performance cost
- Full journaling: ~15-30% performance cost
- No journaling: Faster but risk of corruption
```

### 5. Mounting and Unmounting

```bash
# See what's mounted
mount
# or with more detail
lsblk

# Inode reuse after delete:
# When you delete a file, inode is marked free
# Next file can reuse the inode number
# But data blocks are only cleared when overwritten

# This is a security concern: deleted files can be recovered!
```

### 6. Filesystem Aging and Fragmentation

**ext4 fragmentation:**
- Less common than older ext3
- Preallocation helps prevent it
- Still possible with heavy random writes

**How to check fragmentation:**
```bash
$ e4defrag -c /dev/sda1  # ext4 only
/dev/sda1: 5.2% non-contiguous

# If over 10%, might want to defragment (care: I/O intensive)
```

---

## Essential Commands

### Check Filesystem Type and Capacity

```bash
# What filesystems are mounted?
df -h                      # Human-readable disk usage
df -i                      # Inode usage
lsblk                      # Block device layout
blkid                      # Identify filesystem types
sudo file -s /dev/sda1     # Detailed filesystem info

# Example output:
$ df -h
Filesystem      Size Used Avail Use%
/dev/sda1       100G  50G  50G   50%
/dev/sdb1       500G 250G 250G   50%
tmpfs           8.0G   0   8.0G   0%
```

### Detailed Filesystem Information

```bash
# Get inode info
stat /path/to/file          # Single file details
stat /                      # Filesystem stats

# Get filesystem stats
$ stat /
  File: /
  Size: 4096      Blocks: 8      IO Block: 4096   directory
  Inode: 2        Links: 19
  Access: (0755/drwxr-xr-x)

# Check inode usage
df -i /                     # See inode limit
```

### Filesystem Maintenance

```bash
# Check filesystem for errors (must unmount first!)
sudo fsck -n /dev/sda1     # Dry run (read-only check)
sudo fsck /dev/sda1         # Actual repair (dangerous!)

# Check fragmentation (ext4 only)
sudo e4defrag -c /dev/sda1

# List open files in filesystem
lsof +D /var               # All files open in /var

# Get size of directory
du -sh /var                # Summary
du -sh /var/*              # Per-subdirectory
```

### Monitor Filesystem Activity

```bash
# Real-time I/O
iostat -x 1                # I/O statistics (will cover more in Week 4)

# What process is writing?
iotop                      # Top I/O processes

# Watch filesystem calls
strace -e open,read,write command    # System calls (Week 6)
```

---

## Interactive Exercise: Analyze a Filesystem

**Task 1: Check your current filesystem setup**
```bash
# What filesystems are mounted?
df -h

# Expected output:
Filesystem      Size Used Avail Use%
/dev/sda1       100G  50G  50G   50%
/dev/sdb1       500G 250G 250G   50%
tmpfs           8.0G   0   8.0G   0%

# What are their types?
lsblk
# or
blkid
```

**Task 2: Check inode usage**
```bash
# Any filesystem near inode limit?
df -i

# If you see > 80% in IUse%, that's concerning

# What's taking up inodes?
find /var -type f | wc -l      # Count files in /var
du -sh /var                    # Space used
# Mismatch = lots of small files
```

**Task 3: Get detailed filesystem info**
```bash
# Pick one filesystem
stat /

# Note:
# - Total inodes available
# - Free inodes
# - Block size
# - Fragment size
```

**Task 4: Simulate a scenario**
```bash
# Create many small files to see inode/space tradeoff
mkdir /tmp/test_files
cd /tmp/test_files

# Create 1000 files
for i in {1..1000}; do touch file_$i; done

# Check space and inodes
du -sh .
ls -la | wc -l

# See the difference:
# 1000 files × 4KB average block = 4MB on disk
# But many more inodes used

# Clean up
rm -rf /tmp/test_files
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Inode Depletion:** You have 50GB free space but can't create new files. What's the likely cause?
   ```
   A) Filesystem is corrupted
   B) Ran out of inodes
   C) Disk is full
   D) Someone revoked your permissions
   ```

2. **Journaling:** Why does journaling protect against data loss in a crash?
   ```
   (Explain in 2-3 sentences)
   ```

3. **Filesystem Choice:** You're storing 10,000 large video files on a storage server. Would you use ext4 or XFS? Why?
   ```
   (Explain the trade-offs)
   ```

4. **Hard Links:** A file has hard link count of 3. How many copies exist on disk?
   ```
   A) 1 copy (3 links point to same inode)
   B) 3 copies
   C) Depends on filesystem
   D) 3 × size of file
   ```

5. **NFS Performance:** When mounting an NFS filesystem, performance is sometimes bad even on fast networks. Why?
   ```
   (Explain 2 reasons)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Why is the filesystem so slow?"

```bash
# Check fragmentation
sudo e4defrag -c /dev/sda1

# If high:
# 1. Is it ext4? (XFS doesn't fragment as badly)
# 2. Are there lots of random writes?
# 3. Is the filesystem full? (slows allocation)

# Check if many small I/O operations
iotop -b -n 1
# If seeing many seeks, fragmentation likely
```

### Scenario 2: "Can't create new files, but disk has space"

```bash
# Check inode usage
df -i

# If 100% in IUse%:
# Root cause: millions of small files
# Solution:
# 1. Find what's taking space
find / -type f | wc -l
# 2. Clean up old/temporary files
# 3. Consider increasing inode count (requires mkfs)

# More investigation:
find / -size -1k -type f | wc -l  # Count tiny files
```

### Scenario 3: "Storage cluster losing performance"

```bash
# Check for hot spots
iostat -x 1
# If one disk shows high utilization:
# - Maybe unbalanced RAID stripe
# - Or specific filesystem hotspot

# Check if multiple filesystems contending
df -h
# If many active mounts on same disk

# For Lustre:
# Check OST balance
# (More in Week 7)
```

### Scenario 4: "NFS mount is hanging"

```bash
# Check mount status
mount | grep nfs
# If showing "hung" or "noresponse"

# Check if server is reachable
ping nfs_server

# Check NFS stats
nfsstat -c   # Client stats

# Might need to:
# 1. Remount with timeout
# 2. Check server logs
# 3. Look for network issues
```

---

## Key Takeaways

1. **Filesystem is a mapping** from files/directories (logical) to blocks (physical)
2. **Inodes store metadata**, data blocks store content — they're separate
3. **Know your filesystem type**: ext4 for stability, XFS for performance, Btrfs for features
4. **Inode exhaustion is real**: Can happen even with free space
5. **Journaling protects against crashes**: But costs performance
6. **NFS is convenient but slow**: Avoid for performance-critical workloads
7. **Fragmentation matters**: Especially for large sequential I/O

---

## How Ready Are You?

Can you explain these without looking them up?
- [ ] The difference between an inode and a data block
- [ ] How hard links work (why 3 links = 1 file, not 3)
- [ ] Why you might have free space but can't create files
- [ ] The trade-off between ext4 and XFS
- [ ] Why NFS performance is limited

If you checked all boxes, you're ready for Lesson 2.2.

---

Powered by UQS v1.8.5-C
