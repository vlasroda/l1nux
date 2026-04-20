# Week 7 Exercises: Lustre & Infinia

---

## Exercise 7.1: Understanding Lustre Architecture

### Problem 1: Identify Lustre Components

```bash
# On a system with Lustre running:

# Task 1A: Identify MDS
ssh mdsserver
systemctl status mds
# Should show active MDS service
# Check: How much RAM allocated? (Should be substantial for metadata)

# Task 1B: Identify OSS servers
for i in 1 2 3; do
  ssh oss$i systemctl status oss
done
# Should show all OSSs active

# Task 1C: Check OST capacity
lfs df -h
# Record:
# - How many OSSs?
# - How many OSTTs total?
# - Which OSSs are fullest?

# Task 1D: Check replication/RAID
ssh oss1 "lsblk | grep raid"
# Check: Is each OST on RAID? Which level?

# Record findings:
# MDS location:
# Number of OSSs:
# Total capacity:
# Raid level:
```

### Problem 2: Understand File Striping

```bash
# Task 2A: Check default stripe
lfs getstripe -R /mnt/lustre | head -20
# Shows default stripe pattern for directory

# Task 2B: Create files and check striping
dd if=/dev/zero of=/mnt/lustre/stripe-test bs=1M count=100
lfs getstripe /mnt/lustre/stripe-test

# Record:
# - Stripe count (how many OSSs)?
# - Stripe size (bytes)?
# - Which OSSs have stripes?

# Task 2C: Create with custom striping
mkdir -p /mnt/lustre/custom-stripe
lfs setstripe -c 8 -S 2M /mnt/lustre/custom-stripe
dd if=/dev/zero of=/mnt/lustre/custom-stripe/wide-stripe bs=1M count=100
lfs getstripe /mnt/lustre/custom-stripe/wide-stripe

# Record difference:
# - How does wider stripe affect layout?
```

### Problem 3: Diagnose Lustre Issues

```bash
# Task 3A: Check for full OSSs
lfs df -h

# If an OST is 90%+ full:
# Find files on that OST
lfs find /mnt/lustre -obd OST0002
# (replace OST0002 with full one)

# Task 3B: Check MDS load
ssh mdsserver top -b -n 1 | head -3
# Is CPU high? Memory usage?

# If MDS CPU > 50%:
# Check lock contention
ssh mdsserver "lctl get_param ldlm.namespaces.*.lock_count" | tail -5
# If very high: Metadata contention happening

# Task 3C: Check metadata vs data I/O
iostat -x 1 5 | grep -E "sda|sdb"
# MDS disk (usually sda): Should have higher latency
# OST disks (sdb+): Should have high throughput but tolerable latency

# Record findings:
# MDS status:
# OST fullness:
# I/O pattern:
```

### Challenge: Capacity Planning

```bash
# Given: 4 OSSs, each with 10TB
# Current: 30TB used, 10TB free
# Growth: 1TB/week

# Questions:
# 1. How many weeks until full?
# 2. If add one more OSS (10TB), new timeline?
# 3. What rebalancing would happen?
# 4. Impact on performance during rebalance?

# Answers:
# 1. 10 weeks until full
# 2. 50 weeks until full (now have 20TB free)
# 3. New files stripe to 5 OSSs, old files stay on 4
# 4. No impact (reads unaffected, new writes use all 5)
```

---

## Exercise 7.2: Infinia Deep Dive (★ TOP PRIORITY ★)

### Problem 1: CoW Snapshot Comparison

```bash
# Task 1A: Understand CoW concept
# Create a 1GB file
infinia object put mybucket large-file $(mktemp -u --tmpdir largefile.XXXXXX)
dd if=/dev/zero of=/tmp/test-1gb bs=1M count=1000
infinia object put mybucket large-file /tmp/test-1gb

# Task 1B: Create snapshots (should be instant)
time infinia snapshot create mybucket snap-1
time infinia snapshot create mybucket snap-2
time infinia snapshot create mybucket snap-3

# Record:
# - Time for each snapshot (should be <1 second)
# - Space used (should be minimal, just metadata)

# Task 1C: Check space efficiency
infinia bucket stats mybucket | grep -E "logical|physical"
# Logical: Sum of all object sizes (1GB)
# Physical: Actual space used (1GB + 3 snapshots, but ~1.1GB due to CoW)

# Compare to Lustre:
# With Lustre snapshots: 1GB + 1GB (snapshot) = 2GB
# With Infinia CoW: 1GB + ~0.1GB overhead = 1.1GB
# Space savings: 92%
```

### Problem 2: Dedup Effectiveness

```bash
# Task 2A: Create duplicate objects
infinia object put mybucket object-1 /var/log/syslog
infinia object put mybucket object-2 /var/log/syslog
infinia object put mybucket object-3 /var/log/syslog
# Same file, 3 times

# Task 2B: Check space used
infinia bucket stats mybucket | grep -E "logical|physical|dedup"

# Record:
# Logical size: File size × 3
# Physical size: File size × 1.x (dedup sharing)
# Dedup ratio: Should be 2.5x+

# Task 2C: With snapshots (dedup amplified)
infinia snapshot create mybucket dup-snap-1
# Modify object-1
infinia object delete mybucket object-1
infinia object put mybucket object-1 /etc/hosts
# Now snapshot has 2 copies of original file, live has 1 copy

# Check space:
infinia bucket stats mybucket | grep dedup
# Dedup ratio should be even higher (3x+ possible)
```

### Problem 3: Metadata Performance

```bash
# Task 3A: Create many small objects
echo "Testing metadata performance"
for i in {1..10000}; do
  echo "object $i" | infinia object put mybucket test-obj-$i -
  [ $((i % 1000)) -eq 0 ] && echo "Created $i objects"
done

# Task 3B: List them (should be fast)
time infinia object list mybucket | wc -l
# Should complete in <1 second for 10K objects
# Lustre would take 5-10 seconds

# Task 3C: Query with tags
# (If using object metadata/tags)
time infinia object find mybucket tag:type=test

# Record:
# - List time
# - Find time
# - These should be much faster than Lustre directory operations
```

### Problem 4: Replication Tolerance

```bash
# Task 4A: Check current replication
infinia bucket info mybucket | grep replica
# Should be: replica=3

# Task 4B: Simulate node failure
# On production: Would actually power down a node
# For learning: Check status
infinia cluster status

# Task 4C: If node is down:
# Check data availability
infinia bucket health check mybucket
# Should report: All objects available (even with 1 node down)

# Task 4D: Rebuild status
infinia cluster rebuild progress
# Shows: How long until replica on down node rebalanced to another node

# Learn: With 3x replication, can lose up to 2 nodes
```

### Challenge: Migration Scenario

```bash
# Scenario: Company running 50TB Lustre, wants to migrate to Infinia

# Task: Create migration plan

# Step 1: Data assessment
# Questions to answer:
# - How many files? (metadata impact)
# - File size distribution? (snapshot efficiency)
# - Backup requirement? (snapshot frequency)
# - Growth rate? (capacity planning)

# Step 2: Hardware planning
# Current: 4 OSSs + 1 MDS = 5 nodes
# Proposed Infinia: 5 nodes (all equal)
# Benefit: Simpler, better scaling

# Step 3: Timeline
# Week 1: Hardware acquisition
# Week 2: Infinia deployment + testing
# Week 3: Data migration (50TB takes ~5 days at 200MB/s)
# Week 4: Parallel running (both active)
# Week 5: Switchover (applications move to Infinia)
# Week 6: Validation (monitor both)
# Week 7: Decommission Lustre

# Step 4: Risk mitigation
# - Backup all Lustre data to tape (in case)
# - Verify checksum on sample of files
# - Plan for 48-hour cutback window
# - Have rollback procedure documented
```

---

## Exercise 7.3: Operations and Monitoring

### Problem 1: Deploy and Monitor

```bash
# Task 1A: Check cluster status
infinia cluster status
# Record:
# - Node count
# - Health status
# - Free space
# - Rebuild progress (if any)

# Task 1B: Monitor performance
infinia cluster perf
# Record:
# - Operations/sec
# - Average latency
# - Throughput
# - This is your baseline

# Task 1C: Bucket monitoring
infinia bucket perf mybucket
# Record:
# - Read ops/sec
# - Write ops/sec
# - Popular objects (if available)

# Task 1D: Set up alerts
# (Conceptual, may not have full system)
# Critical: Node down, replication degraded
# Warning: Free space < 20%, CPU > 80%
# Info: Snapshots created
```

### Problem 2: Capacity Management

```bash
# Task 2A: Check current usage
infinia cluster stats
# Record:
# - Total capacity
# - Used space
# - Available space
# - Percentage full

# Task 2B: Project growth
# Assume 2TB/week growth
# Current free space: X TB
# Weeks until full: X / 2

# Task 2C: What's consuming space?
infinia bucket list
for bucket in $(infinia bucket list | awk '{print $1}'); do
  size=$(infinia bucket info $bucket | grep "used" | awk '{print $2}')
  echo "$bucket: $size"
done | sort -k2 -rh

# Identify: Which bucket is largest?

# Task 2D: Reduce space (if needed)
# Option 1: Reduce replication
infinia bucket config set mybucket replica=2
# Frees 33% space (but reduces fault tolerance)

# Option 2: Enable compression
infinia bucket config set mybucket compression=aggressive

# Option 3: Add capacity
# New hardware deployment (out of scope here)
```

### Problem 3: Troubleshoot Degraded Cluster

```bash
# Scenario: One node offline, cluster degraded

# Task 3A: Assess damage
infinia cluster status
# Look for: Nodes offline, Rebuild status

# Task 3B: Data safety check
infinia cluster health
# Should report: All objects have sufficient replicas
# Even with 1 node down, should be OK

# Task 3C: Estimate recovery time
infinia cluster rebuild progress
# Shows: Data being redistributed, ETA to completion

# Task 3D: Immediate actions
if_node_unresponsive_after_5min:
  infinia cluster remove node-name
  # Triggers rebuild immediately

if_node_responsive:
  wait_5_min
  infinia cluster health
  if_degraded:
    infinia cluster remove node-name
    investigate_and_fix
    infinia cluster join node-name

# Task 3E: After recovery
infinia cluster status
# Should show: All nodes online, no rebuild in progress
```

### Challenge: Optimize Infinia for Workload

```bash
# Scenario: AI research lab storing model checkpoints
# 100 training runs, each saves checkpoint every hour
# = 100 × 24 hours × 100 models = 240,000 checkpoints
# Each checkpoint: ~5GB
# Total: 1.2PB logical data

# Task: Design Infinia configuration

# Step 1: Evaluate workload
# - Is dedup effective? YES (many identical models)
# - Snapshot frequency? High (hourly)
# - Metadata intensity? Medium (directories per model)
# - Retention? 1 year

# Step 2: Recommend settings
# Replication factor: 2 (not critical, can rerun)
# Compression: aggressive (models compress well)
# Dedup: enabled (many identical blocks)
# Snapshot retention: 365 days of hourly = 8760 snapshots

# Step 3: Estimate space
# Logical: 1.2PB
# With 2x compression: 600TB
# With 2x dedup (many identical models): 300TB
# With 2x replication: 600TB total storage needed

# Compare to Lustre:
# Logical: 1.2PB
# With 3x replication: 3.6PB minimum
# Lustre has no snapshots: Must keep 365 full backups = 438PB needed!
# Or: Keep only weekly backups (52 weeks) = 62PB

# Infinia savings: 438PB → 600TB = 730x reduction!
```

---

## Exercise 7.4: Migrate Lustre to Infinia

### Scenario: "Company has 100TB Lustre, wants Infinia"

```bash
# Phase 1: Planning (Week 1)

# Task 1A: Data assessment
# - Count total files
find /mnt/lustre -type f | wc -l
# - Size distribution
find /mnt/lustre -type f -exec du {} \; | awk '{print $1}' | sort -rn | head -20
# - Identify hotspots (frequently accessed data)

# Task 1B: Backup preparation
# - Full backup to tape
tar czf /mnt/backup/lustre-backup-$(date +%Y%m%d).tar.gz /mnt/lustre
# (Simplified, would use actual backup tool)

# Task 1C: Application inventory
# - List all apps using Lustre
ps aux | grep lustre | grep -v grep
# - Document access patterns
# - Identify upgrade windows

# Phase 2: Parallel Running (Weeks 2-4)

# Task 2A: Deploy Infinia cluster
# (Conceptual)
infinia cluster init node1
infinia cluster join node2 node1
infinia cluster join node3 node1

# Task 2B: Create Infinia buckets
infinia bucket create prod-data
infinia bucket create test-data

# Task 2C: Copy data from Lustre to Infinia
# For each dataset:
for dataset in /mnt/lustre/data/*; do
  rsync -avR "$dataset" infinia://prod-data/
  # Verify
  src_count=$(find "$dataset" -type f | wc -l)
  dst_count=$(infinia object list prod-data | wc -l)
  [ "$src_count" == "$dst_count" ] && echo "OK" || echo "MISMATCH"
done

# Task 2D: Run in parallel
# Apps read from both Lustre and Infinia
# New writes go to Infinia
# Read from Infinia if available, fall back to Lustre

# Task 2E: Validate on Infinia
# - Run nightly jobs on Infinia
# - Compare results to Lustre
# - Check performance
infinia bucket perf prod-data

# Phase 3: Switchover (Week 5)

# Task 3A: Final sync
# Copy any final Lustre writes to Infinia
rsync -avR /mnt/lustre/ infinia://prod-data/

# Task 3B: Point apps to Infinia
# Reconfigure all apps to use Infinia
# Update mount paths, URLs, etc.

# Task 3C: Monitor closely
for i in {1..60}; do
  infinia bucket health check prod-data
  sleep 60  # Check every minute for first hour
done

# Task 3D: Fallback plan ready
# If problem detected:
# - Revert apps to Lustre
# - Investigate issue
# - Reschedule switchover

# Phase 4: Validation (Week 6)

# Task 4A: Check data integrity
# Sample verification
for obj in $(infinia object list prod-data | shuf | head -100); do
  infinia object get prod-data "$obj" | md5sum
done

# Task 4B: Performance baseline
infinia cluster perf > perf-after-switchover.txt

# Task 4C: Backup verification
# Can we restore from Infinia snapshots?
infinia snapshot list prod-data
# Test restore to dev environment
infinia bucket restore snap-2024-week5 prod-data-dev

# Phase 5: Decommission Lustre (Week 7)

# Task 5A: Final backup
tar czf /mnt/backup/lustre-final-backup.tar.gz /mnt/lustre

# Task 5B: Shutdown
systemctl stop mds
systemctl stop oss
for i in 1 2 3 4; do
  ssh oss$i systemctl stop oss
done

# Task 5C: Archive media
# Archive Lustre disks to long-term storage
# Document for compliance/audits

# Task 5D: Reclaim hardware
# Repurpose or sell Lustre hardware
```

---

## Exercise 7.5: Comparison and Decision Matrix

### Compare Lustre vs Infinia for Different Workloads

```bash
# Create a decision matrix

# Workload 1: HPC Simulation
# - 100,000 small files per job
# - High sequential I/O (read input, write output)
# - Frequent 100GB file writes
#
# Analysis:
# Metadata intensity: Medium (directory structure)
# I/O pattern: Sequential (good for both)
# Snapshot needs: Low (job checkpoints, not continuous)
#
# Recommendation: Either works
# - Lustre: Simpler setup, proven HPC
# - Infinia: Better metadata performance (better choice)

# Workload 2: VDI Environment
# - 1000 virtual machine templates
# - Each 10GB
# - Mostly read (users boot from templates)
# - Occasional clone (create new VM from template)
#
# Analysis:
# Metadata: High (1000 files to manage)
# I/O: Random access (typical VM access)
# Snapshots: High (clone = instant snapshot)
# Dedup: Extreme (clones share all blocks)
#
# Recommendation: Infinia (clear winner)
# - Dedup: 1000 VMs = 10TB logical, ~1TB physical
# - Snapshots: Clone instant
# - Metadata: 100,000x better than Lustre
# - Lustre would need 10TB × 1000 = 10PB

# Workload 3: Streaming Video Server
# - 50,000 movies (5GB each)
# - Sequential access (streaming)
# - High throughput requirement
# - Minimal metadata operations
#
# Analysis:
# Metadata: Low (files rarely change)
# I/O: Sequential (predictable pattern)
# Snapshots: Low (content is published, not changing)
# Dedup: None (each movie unique)
#
# Recommendation: Lustre (equally good, simpler)
# - Sequential I/O is Lustre strength
# - Simpler MDS tuning
# - Proven for media workloads
# - Infinia: Unnecessary complexity

# Workload 4: Backup Storage
# - Store 1000 daily backups
# - Each backup 500GB
# - Mostly writes (backups incoming)
# - Frequent restores (point-in-time recovery)
#
# Analysis:
# Metadata: Medium (many incremental backup versions)
# I/O: Mostly write, some read
# Snapshots: Very high (point-in-time recovery = snapshot!)
# Dedup: Very high (incremental backups = many duplicates)
#
# Recommendation: Infinia (only reasonable choice)
# - 1000 backups × 500GB = 500TB logical
# - With Infinia dedup: ~50TB physical (90% savings)
# - With Infinia snapshots: Can restore from any backup instantly
# - Lustre: Would need 500TB × 3 = 1.5PB (no dedup, expensive snapshots)
# - Infinia cost: 10% of Lustre for same data

# Summary:
# Lustre: Pure sequential I/O, simple deployment
# Infinia: Metadata-heavy, snapshots, dedup, cloud-native
```

---

## Self-Assessment

After Week 7, you should be able to:

- [ ] Explain Lustre architecture (MGS, MDS, OSS, OSTTs)
- [ ] Understand why Infinia was built (limitations of POSIX)
- [ ] Explain Copy-on-Write and instant snapshots
- [ ] Describe key-value distributed metadata advantages
- [ ] Compare Lustre vs Infinia for different workloads
- [ ] Plan migration from Lustre to Infinia
- [ ] Troubleshoot Infinia cluster issues
- [ ] Monitor Infinia health and performance
- [ ] Optimize Infinia settings for workload

---

Powered by UQS v1.8.5-C
