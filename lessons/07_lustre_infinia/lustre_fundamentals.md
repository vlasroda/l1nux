# Lesson 7.1: Lustre Fundamentals and Architecture (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Understanding traditional Lustre is essential context for understanding why Infinia was built differently. Many production DDN systems still run Lustre; L1 TSEs must support both legacy and new architectures
**Skills:** Understand Lustre architecture, identify Lustre components, diagnose Lustre-specific issues, prepare for Infinia comparison

---

## Quick Review

### What is Lustre?

```
Lustre = Linux + Cluster = Distributed POSIX Filesystem

Built on: Linux kernel + XFS filesystem
Purpose: High-performance parallel filesystem for HPC/storage clusters
Scale: Hundreds to thousands of nodes
Bandwidth: Can saturate 10Gbps+ networks
Use case: Supercomputers, research facilities, enterprise storage
```

---

## Key Concepts

### 1. Lustre Architecture

**Three-tier design:**

```
┌─────────────────────────────────────────┐
│         Client Nodes                    │
│    (Mount Lustre filesystem)            │
│  /mnt/lustre (POSIX mountpoint)        │
└──────────────┬──────────────────────────┘
               │
        Network (Ethernet, InfiniBand)
               │
        ┌──────┴────────────────────────────┐
        │                                   │
   ┌────▼────────┐              ┌──────────▼──┐
   │    MGS       │              │     MDS     │
   │  (Mgmt Srvr) │              │ (Meta Srvr) │
   └─────────────┘              └──────────────┘
        (1 per fs)                  (1+ per fs)
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
              ┌─────▼──┐         ┌─────▼──┐         ┌─────▼──┐
              │ OSS #1 │         │ OSS #2 │  ...    │ OSS #N │
              │(Object │         │(Object │         │(Object │
              │ Server)│         │ Server)│         │ Server)│
              └────────┘         └────────┘         └────────┘
              ↓(many OSTs)       ↓(many OSTs)       ↓(many OSTs)
         [Disk arrays]      [Disk arrays]      [Disk arrays]
         (XFS backend)      (XFS backend)      (XFS backend)
```

**Components:**

```
MGS (Management Server):
- Manages filesystem metadata
- Configuration database
- Handles mount tokens
- Usually small (1 per filesystem)

MDS (Metadata Server):
- Handles directory structure
- File metadata (permissions, timestamps)
- Lock management for file access
- Can be single or HA pair

OSS (Object Storage Server):
- Stores actual file data (objects)
- One OSS can have many OSTs
- Each OST is one filesystem partition
- Data split across multiple OSSs (striping)

OT (Object Target):
- Physical storage on OSS
- Each OT is XFS partition or RAID volume
- Client data distributed across OTs
```

**Data flow for write operation:**

```
1. Client calls write()
2. Client contacts MDS for lock
3. MDS grants lock to client
4. Client writes to OSTs (directly, not through MDS)
5. Client confirms write to MDS
6. MDS logs transaction
7. Application returns from write()

Key insight: Data goes directly to OSSs
            Metadata goes through MDS
            This separates bottlenecks
```

### 2. Lustre Striping

**How data is distributed:**

```
File: /mnt/lustre/bigfile.dat (1GB)

With striping across 4 OSSs (stripe size 1MB):

Stripe 0 (1MB):     OST0 on OSS0
Stripe 1 (1MB):     OST1 on OSS1
Stripe 2 (1MB):     OST2 on OSS2
Stripe 3 (1MB):     OST3 on OSS3
Stripe 4 (1MB):     OST0 on OSS0  (cycles)
...

Advantages:
- Data parallelism (4 servers write in parallel)
- 4x throughput vs single server
- Single server failure doesn't lose file
- Automatic load distribution

Disadvantages:
- Metadata overhead (MDS must track all stripes)
- Small files have fragmentation overhead
- Unbalanced cluster underutilizes capacity

Parameters:
stripe_size: How much data before next OST (usually 1MB)
stripe_count: How many OSSs to stripe across
stripe_offset: Which OST gets first stripe
```

### 3. Lustre on XFS

**Backend filesystem choice:**

```
Why XFS?
- Handles large files efficiently
- Good for sequential I/O
- Mature, proven at scale
- Good metadata performance
- Journaling prevents corruption

XFS on Lustre:
- Each OT is a dedicated XFS volume
- Usually single large partition (no divisions)
- Optimized mount options (logbsize, sunit, etc.)
- Inode handling tuned for Lustre

Weaknesses:
- Not optimized for random I/O (Lustre does striping)
- Fragmentation can develop over time
- Defragmentation requires downtime
```

### 4. Metadata Performance

**The MDS bottleneck:**

```
Problem: Single MDS for all metadata operations

Operations going through MDS:
- Directory listing (ls)
- Permission checks
- Lock management
- File creation/deletion
- Attribute access

Performance characteristics:
Single MDS can handle ~1000-5000 ops/sec
Striped filesystem might need 100,000+ ops/sec

Result: MDS becomes bottleneck for metadata-heavy workloads

Solutions:
1. Use multiple MDSs (federation)
   - Partition namespace across MDSs
   - Each MDS handles subset of files
   - Requires application awareness

2. Increase MDS hardware
   - More cores, more RAM
   - SSD for metadata journals
   - Better network connectivity

3. Change application pattern
   - Batch operations
   - Reduce file count
   - Use larger stripe sizes
```

### 5. Common Lustre Issues

**Problem: "OST is full"**

```bash
lfs df -h
# Shows space on each OST

# If one OST much fuller than others:
# Data is unbalanced
# New files stripe poorly
# Fix: Rebalance with lfs migrate

# Prevention:
# Monitor regularly
# Set quotas per user
# Track growth trends
```

**Problem: "Metadata is slow"**

```bash
# Check MDS load
ssh mdsserver top
# High CPU = MDS bottleneck

# Check lock contention
ssh mdsserver "lctl get_param ldlm.namespaces.*.lock_count"
# High lock count = contention

# Solutions:
# 1. Increase MDS hardware
# 2. Move operations to different time
# 3. Use MDSs for different tree branches
```

**Problem: "File deletion is slow"**

```bash
# Lustre has to communicate with all OSSs
rm bigfile
# Might take minutes for large files

# Why: Must update every OT's journal
# Each OST confirms deletion

# Solutions:
# 1. Delete in batches (gives MDS time)
# 2. Use unlink queue
# 3. Schedule deletes during maintenance
```

**Problem: "Stripe imbalance"**

```bash
lfs getstripe /mnt/lustre/dir
# Shows stripe pattern for files

# If all new files go to OST0:
# One OSS getting all traffic
# Other OSSs idle

# Root causes:
# 1. OST weights unbalanced
# 2. Specific OST failing (shows as full)
# 3. File creation timestamp near rebalance

# Fix:
lfs setstripe -c -1 /mnt/lustre/newdir
# -c -1 = stripe across all available OSSs
# Use for new files/directories
```

---

## Essential Commands

### Lustre Status

```bash
# Check filesystem health
lfs df -h
# Shows space on each OST
# Identifies full OSSs

# Check MDS status
lctl list_nids
# List network interfaces Lustre uses

# Check mounted filesystems
mount | grep lustre
# Show Lustre mount points

# Check OST status
lctl pool_list
# Show OST pools (groups)

# Check version
lustre_build_version
# Lustre build info
```

### File Striping

```bash
# See how file is striped
lfs getstripe /mnt/lustre/myfile
# Output:
# stripe_count:  4
# stripe_size:   1048576
# stripe_offset: 2

# Set striping for new files
lfs setstripe -c 4 -S 1M /mnt/lustre/newdir
# 4 stripe count, 1MB stripe size

# Migrate file to different striping
lfs migrate -c 8 /mnt/lustre/bigfile
# Restripe to 8 OSSs
```

### Quotas

```bash
# Check quotas
lfs quota -u username /mnt/lustre
# Show user's space/file usage and limits

# Set quotas
setquota -u username -b 0 100G 0 1M /mnt/lustre
# User limited to 100GB space, 1M files
```

---

## Interactive Exercise: Understand Lustre Behavior

**Task 1: Check cluster health**

```bash
# On Lustre client:
lfs df -h

# Record:
# - How many OSSs?
# - How many OSTTs?
# - Are any nearly full?
```

**Task 2: Check file striping**

```bash
# Create test file
dd if=/dev/zero of=/mnt/lustre/test1 bs=1M count=100

# Check how it's striped
lfs getstripe /mnt/lustre/test1

# Record:
# - Stripe count?
# - Which OSSs has the stripes?
# - Is it balanced?
```

**Task 3: Monitor metadata performance**

```bash
# Metadata-heavy operation
time ls -lR /mnt/lustre > /dev/null
# Time how long directory listing takes

# Repeat several times
time ls -lR /mnt/lustre > /dev/null

# Record:
# - How long does it take?
# - Is it consistent or variable?
# - Variable = MDS contention
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Lustre Component:** What does the MDS do?
   ```
   A) Store actual file data
   B) Manage directory structure and metadata
   C) Route network traffic
   D) Manage user authentication
   ```

2. **Stripe Benefits:** Why does Lustre stripe files across multiple OSSs?
   ```
   A) For redundancy
   B) For fault tolerance
   C) For parallel I/O performance
   D) For compression
   ```

3. **Metadata Bottleneck:** Why is MDS often a bottleneck?
   ```
   A) Not enough bandwidth
   B) Single point for all metadata ops
   C) XFS is slow
   D) Network latency
   ```

4. **OST Full Problem:** If one OST is full but others empty, what's wrong?
   ```
   A) Rebalance needed
   B) OSS hardware failing
   C) Stripe count wrong
   D) Files need migration
   ```

5. **Write Operation:** Where does client send data in Lustre?
   ```
   (Explain the path, not through MDS)
   ```

---

## Common L1 TSE Scenarios for Lustre

### Scenario 1: "Lustre mount is hanging"

```bash
# Step 1: Check if OST is online
lfs df -h
# If an OST shows down/unavailable, that's the cause

# Step 2: Check MDS
ssh mdsserver ping localhost
# MDS responsive?

# Step 3: Check network
ping ost0server
# Can reach OST servers?

# Step 4: Check mount parameters
mount | grep lustre
# Timeout values? (mount_timeout=30 vs mount_timeout=5)

# Quick diagnosis:
# If lfs df hangs: MDS or network problem
# If specific OST missing: That OST server down
# If all OSSs missing: Major network issue

# Solutions:
# Increase mount timeouts
# Restart down servers
# Check network connectivity
```

### Scenario 2: "File deletion is taking forever"

```bash
# Lustre must track deletion on all OSSs

# Check MDS activity
ssh mdsserver "lctl get_param mdt.*.job_cleanup_interval"
# How often does cleanup run?

# Check OSS deletion queues
ssh ossserver "lctl get_param obdfilter.*.last_grant"
# How many pending deletions?

# Solutions:
# 1. Wait (delete operations are queued and happen in background)
# 2. Restart filesystem (drastic)
# 3. Increase OSS resources if persistently slow
# 4. Change backup strategy to avoid large deletes
```

### Scenario 3: "OST is full, storage is unavailable"

```bash
# Check space
lfs df -h

# Find the full OST
# Output shows which OST is 100%

# Identify files on that OST
lfs find /mnt/lustre -obd OST0002

# Options:
1. Clean up old files: rm /mnt/lustre/olddir
2. Migrate to other OSTs: lfs migrate -c -1 /mnt/lustre/file
3. Add more disk to OST (requires downtime)
4. Move users to different filesystem

# Prevention:
# Monitor quota
# Set alerts at 80% full
# Plan capacity growth
```

---

## Key Takeaways

1. **Lustre architecture:** MGS, MDS, OSS with XFS backend
2. **Striping:** Data distributed across multiple OSSs for parallel I/O
3. **MDS bottleneck:** Single point for all metadata operations
4. **XFS choice:** Proven, mature, good for sequential I/O
5. **Common issues:** Full OSSs, metadata contention, slow deletion

---

## How Ready Are You?

Can you explain these?
- [ ] Three main Lustre components and their roles
- [ ] How striping improves performance
- [ ] Why MDS is often the bottleneck
- [ ] What commands to use for Lustre diagnosis
- [ ] How Lustre differs from traditional filesystems

If you checked all boxes, you're ready for Lesson 7.2: **Infinia Revolution**.

---

Powered by UQS v1.8.5-C
