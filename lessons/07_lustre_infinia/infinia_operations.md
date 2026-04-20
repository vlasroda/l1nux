# Lesson 7.3: Infinia Operations and Troubleshooting

**Time:** 25-30 minutes
**Why it matters:** Understanding how to deploy, monitor, and troubleshoot Infinia is essential for L1 TSE support. Infinia requires different operational approaches than Lustre due to its architecture
**Skills:** Deploy Infinia, monitor health, troubleshoot issues, migrate from Lustre, optimize performance

---

## Quick Review

### Infinia vs Lustre Operational Model

```
Lustre:
├─ Mount as POSIX filesystem
├─ One-to-one MDS/OSS relationship
├─ Striping configuration per file
├─ Snapshots are complex
└─ Metadata server management critical

Infinia:
├─ Bucket-based (like S3)
├─ Distributed metadata (no single point of failure)
├─ Automatic striping/distribution
├─ Native snapshots (simple)
└─ Cluster node management (any node = equal)
```

---

## Key Concepts

### 1. Deployment Models

**On-Premise Infinia:**

```
Deployment options:

1. Dedicated Infinia Cluster
   ├─ 3 node minimum (for HA)
   ├─ All nodes equal (no MGS/MDS/OSS roles)
   ├─ Automatic replication
   ├─ Recommended: 5-10 nodes for production
   └─ Storage: Direct-attached or SAN

2. Converged Storage
   ├─ Infinia shares cluster with compute
   ├─ Storage consumed on each node
   ├─ Some nodes compute-focused, some storage-focused
   └─ Useful for edge, remote sites

3. Hybrid (Cloud + On-Prem)
   ├─ Primary on-prem Infinia
   ├─ Failover replica in cloud
   ├─ Snapshot replication to cloud for DR
   └─ Cost: Pay for cloud only on failover

Example topology:
3 x DDN SB2030 nodes (10 drives each)
2 x DDN SB2015 nodes (5 drives each)
Result: 40 drives total, ~500TB effective (with 3x replication)
```

**Cloud-Native Infinia:**

```
Kubernetes deployment:
├─ Infinia as container workload
├─ StatefulSets for persistence
├─ S3 API (AWS SDK compatible)
├─ Auto-scaling (add nodes on-demand)
└─ Perfect for:
   - Multi-tenant environments
   - Burst capacity
   - Dev/test environments
```

### 2. Cluster Architecture

**Distributed architecture:**

```
Node topology (5-node cluster):

┌─Node1─────────────────────────┐
│ ├─ Key-Value Store            │
│ ├─ Data (1/5 of objects)       │
│ ├─ Replica (1/5 of others)     │
│ └─ Metadata Index              │
└─Node1─────────────────────────┘
┌─Node2─────────────────────────┐
│ ├─ Key-Value Store            │
│ ├─ Data (1/5 of objects)       │
│ ├─ Replica (1/5 of others)     │
│ └─ Metadata Index              │
└─Node2─────────────────────────┘
... (Node3, Node4, Node5)

Replication:
Object A stored on Node1
Replica 1 on Node2
Replica 2 on Node3
If Node1 dies, Node2 and Node3 can reconstruct

Read: Fastest replica (usually closest)
Write: Quorum (usually 2/3 for consistency)
```

**Failure tolerance:**

```
Replication factor 3 (standard):
- Can lose 2 nodes and still operate
- Can lose 1 node with no performance impact
- Rebuild time: Minutes to hours (depends on data size)

Replication factor 2 (economy):
- Can lose 1 node
- Rebuild time: Faster
- Risk: Single failure = data vulnerable

Replication factor 4 (paranoid):
- Can lose 3 nodes
- Space overhead: 4x
- Typical only for critical data
```

### 3. Monitoring and Health

**Key metrics to watch:**

```bash
Health check:
infinia cluster health

# Output includes:
# Node status (online/degraded/offline)
# Replication status (healthy/recovering/critical)
# Disk health (S.M.A.R.T. status)
# Network latency between nodes
# CPU/Memory utilization per node

Performance metrics:
infinia cluster perf

# Shows:
# - Operations per second (get/put/delete)
# - Average latency (ms)
# - Throughput (MB/s read/write)
# - Concurrent connections
# - Cache hit rate

Bucket-specific metrics:
infinia bucket perf mybucket

# Shows:
# - Objects count
# - Total capacity used
# - Compression ratio
# - Dedup efficiency
# - Popular keys (hotspots)
```

**Alerting rules:**

```
Critical (page on-call):
├─ Node down (replication at risk)
├─ Replication degraded (data vulnerable)
├─ Disk error rate high
└─ Free space < 10%

Warning (investigate next business day):
├─ Node CPU > 80%
├─ Node memory > 90%
├─ Rebuild in progress (slow)
├─ Free space < 20%
└─ Latency spike > 2x normal

Info (monitor, no action):
├─ Snapshot created
├─ Node added to cluster
├─ Node removed from cluster
└─ Dedup ratio improved
```

### 4. Migration from Lustre to Infinia

**Why migrate:**

```
Reasons to move from Lustre to Infinia:

Performance:
├─ Metadata-heavy workloads (file count >> throughput)
├─ Frequent snapshots (backup strategy)
├─ Need deduplication savings
└─ Cloud-native apps (S3 API)

Operational:
├─ Reduce complexity (distributed vs centralized MDS)
├─ Better snapshot support
├─ Easier scaling (add nodes, no rebalancing)
└─ No MDS bottleneck tuning needed

Cost:
├─ Dedup savings (10-100x for favorable workloads)
├─ Snapshot efficiency (can keep more history)
├─ Simpler operations (fewer tools, less expertise)
└─ Hardware: Similar cost, better utilization
```

**Migration strategy:**

```
Phase 1: Parallel running (1-2 months)
├─ Deploy Infinia alongside Lustre
├─ Copy data from Lustre → Infinia
├─ Test application on Infinia
├─ Run in parallel for validation
└─ Fallback ready (original Lustre still running)

Phase 2: Switchover (1 day)
├─ Final sync from Lustre → Infinia
├─ Point applications to Infinia
├─ Monitor for issues
└─ Keep Lustre as cold backup for 1 week

Phase 3: Decommission Lustre (after 1 month)
├─ Confirm Infinia stable
├─ Archive Lustre data
├─ Remove Lustre hardware
└─ Reclaim space

Data transfer approaches:
1. rsync (for small datasets)
   rsync -avR /mnt/lustre/* s3://infinia-bucket/

2. Data migration tool (for large)
   infinia migrate lustre-nfs-gateway infinia-bucket

3. Application-driven (for continuous)
   App writes to both during transition
```

### 5. Common Infinia Issues and Fixes

**Problem: "Node went offline, performance degraded"**

```bash
# Check cluster status
infinia cluster status
# Shows: Node2 OFFLINE

# What's happening:
# - Replication: Objects on Node2 being rebuilt to other nodes
# - Performance: Reduced capacity, higher latency during rebuild
# - Rebuild time: Depends on data size and network (hours typical)

# Investigation:
ssh node2 "infinia logs" | grep -i error | tail -20
# Look for: disk error, network error, OOM

# Solutions:
1. If hardware failure:
   - Remove node from cluster
   - Replace disk/hardware
   - Rejoin cluster (data rebalances)

2. If network issue:
   - Check connectivity: ping node2
   - Check firewall: sudo iptables -L
   - Restart node2

3. If software crash:
   - Restart Infinia service
   - Check logs for bugs
   - Upgrade if patch available
```

**Problem: "Snapshots taking more space than expected"**

```bash
# Check dedup effectiveness
infinia bucket stats mybucket | grep -E "dedup|compression"

# Output might show:
# dedup_ratio: 1.5x (not much dedup)
# compression_ratio: 1.2x (minimal compression)

# Causes:
1. Data already compressed
   - Videos, images, archives already use compression
   - Further compression doesn't help

2. Data patterns don't match dedup
   - Each object unique
   - Dedup works best with duplicates

3. Many small unique objects
   - Metadata overhead may exceed space savings

# Solutions:
1. Verify dedup is enabled
   infinia bucket config get mybucket | grep dedup
   # Should be: dedup=enabled

2. Check replication count
   infinia bucket info mybucket | grep replica
   # Each replica counts as "used" space
   # Reduce from 3 to 2 if cost-sensitive

3. Check actual data patterns
   infinia bucket stats --detailed mybucket
   # See distribution of object sizes and types
```

**Problem: "Metadata operations are slow"**

```bash
# Remember: Infinia shouldn't have metadata bottleneck
# If slow, it's a different issue

# Check node health
infinia cluster health
# Any nodes degraded? High CPU?

# Check cluster load
infinia cluster perf | grep operations
# Operations/sec too low?

# Check specific operation
time infinia object list mybucket | wc -l
# Listing 1M objects should be <1 second

# If slow, investigate:
1. Network latency between nodes
   infinia cluster latency
   # Should be <5ms between nodes
   # If >50ms: Network problem

2. Disk latency on metadata store
   ssh node1 "iostat -x 1 5" | grep await
   # Metadata storage await should be <5ms
   # If >20ms: Disk performance issue

3. CPU saturation
   ssh node1 "top -b -n 1" | head -5
   # Should not be consistently >80%

# Solutions:
1. Upgrade network (if latency)
2. Add metadata cache (if disk bottleneck)
3. Add CPU resources (unlikely)
```

**Problem: "Data inconsistency or corruption detected"**

```bash
# Infinia has built-in checksums
# If corruption detected, automatic recovery happens

# Check repair status
infinia cluster repair status

# If repair needed:
1. Don't panic (replicas have good copies)
2. Monitor repair progress
   infinia cluster repair progress
   # Shows: nodes repaired, time remaining

3. Check what caused it
   infinia logs | grep -i "checksum\|repair"

4. Fix root cause
   - Bad RAM? (run memtest)
   - Faulty disk? (replace)
   - Network bit flips? (less likely, check cabling)
```

### 6. Performance Optimization

**Tuning for Infinia:**

```bash
# Cache tuning
infinia bucket config set mybucket cache_size=100GB
# Increase cache if metadata queries slow

# Compression tuning
infinia bucket config set mybucket compression=aggressive
# Trade: CPU for space savings

# Replication factor
infinia bucket config set mybucket replica=2
# Reduce from 3 to 2 if cost-sensitive
# Increase to 4 for critical data

# Dedup granularity
infinia bucket config set mybucket dedup_block=8KB
# Smaller block = more dedup but more metadata overhead

# Access pattern hints
infinia bucket config set mybucket access_pattern=random
# Helps optimize caching
# Options: sequential, random, mixed
```

**Monitoring during optimization:**

```bash
# Baseline before change
infinia bucket perf mybucket > before.txt

# Make change
infinia bucket config set mybucket cache_size=100GB

# Wait 5 minutes for stabilization
sleep 300

# Check after
infinia bucket perf mybucket > after.txt

# Compare
diff before.txt after.txt
# Look for improvements in latency/throughput
```

---

## Essential Commands

### Cluster Management

```bash
# Check overall health
infinia cluster status

# Add node to cluster
infinia cluster join new-node-ip

# Remove node (gracefully)
infinia cluster remove node-name
# Data redistributes automatically

# Expand capacity
infinia cluster expand /dev/sdb
# Add new disk to node

# Monitor rebuild progress
infinia cluster rebuild progress

# Cluster statistics
infinia cluster stats
```

### Bucket Operations

```bash
# Create bucket
infinia bucket create mybucket

# Delete bucket (careful!)
infinia bucket delete mybucket --force

# Get bucket info
infinia bucket info mybucket

# Set bucket options
infinia bucket config set mybucket replica=3

# Monitor bucket performance
infinia bucket perf mybucket

# Check replication status
infinia bucket replication status mybucket
```

### Troubleshooting

```bash
# View logs
infinia logs --node all --since "1 hour ago"

# Check specific error
infinia logs | grep -i "error\|fail"

# Diagnose network
infinia cluster latency
infinia cluster bandwidth

# Health check
infinia cluster diagnose
# Generates detailed report

# Performance profile
infinia perf profile 60
# 60-second profile
```

---

## Interactive Exercise: Understand Infinia Operations

**Task 1: Deploy and join cluster**

```bash
# Initial node (if fresh cluster)
infinia cluster init node1

# Additional nodes
infinia cluster join node2 node1-ip
infinia cluster join node3 node1-ip

# Verify
infinia cluster status
# Should show all 3 online
```

**Task 2: Create bucket with snapshots**

```bash
# Create bucket
infinia bucket create mybucket

# Create snapshot
infinia snapshot create mybucket daily-$(date +%Y%m%d)

# Create 3 more
infinia snapshot create mybucket daily-$(date -d yesterday +%Y%m%d)

# List
infinia snapshot list mybucket
# Should show 4 snapshots, all instant

# Check space
infinia bucket stats mybucket
# Physical space used (with CoW/dedup)
```

**Task 3: Monitor performance**

```bash
# Baseline
infinia cluster perf > baseline.txt

# Create some load
for i in {1..1000}; do
  echo "test data" | infinia object put mybucket obj-$i -
done

# Monitor during
infinia cluster perf
# Watch operations/sec spike

# Check impact
infinia bucket perf mybucket
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Node Failure:** If one node in 3-node cluster fails, what happens?
   ```
   A) Data is lost
   B) System offline (no quorum)
   C) Still operational (2 replicas sufficient)
   D) Need to rebuild immediately
   ```

2. **Metadata Performance:** Why is Infinia metadata faster than Lustre?
   ```
   A) Better hardware
   B) Distributed (no single point)
   C) Caching
   D) Compression
   ```

3. **Migration Strategy:** Best approach for migrating 50TB Lustre → Infinia?
   ```
   (Explain phases and checkpoints)
   ```

4. **Space Overhead:** 3 replicas + CoW snapshots. How much total space?
   ```
   A) 3x data size
   B) 3x data + metadata (small)
   C) Depends on dedup/compression
   D) Cannot determine without workload info
   ```

5. **Optimize For:** Metadata workload (10M files). What settings?
   ```
   (Recommend: cache, dedup block size, replica factor)
   ```

---

## Common L1 TSE Scenarios for Infinia Operations

### Scenario 1: "Cluster degraded, need to bring node offline safely"

```bash
# Step 1: Check status
infinia cluster status
# Shows: Node3 degraded

# Step 2: Remove gracefully
infinia cluster remove node3
# Triggers data rebalancing

# Step 3: Monitor
infinia cluster rebuild progress
# Shows: 45% complete, ETA 2 hours

# Step 4: Physical work
# While rebuild running:
# - Replace disk on node3
# - Upgrade hardware
# - Network maintenance

# Step 5: Rejoin cluster
infinia cluster join node3 node1-ip

# Step 6: Verify
infinia cluster status
# All nodes online, rebuild complete
```

### Scenario 2: "Running out of space, need capacity expansion"

```bash
# Current space
infinia cluster stats | grep "total\|available"

# Option 1: Add disks to existing nodes
ssh node1 "lsblk"  # Find available slot
infinia cluster expand node1 /dev/sdc
# Repeat for node2, node3

# Option 2: Add new nodes
infinia cluster join node4 node1-ip
# New node brings 10TB capacity

# Option 3: Reduce replication (emergency only)
infinia bucket config set mybucket replica=2
# From 3x to 2x = 33% more usable space
# Risk: Can only lose 1 node now

# Monitor expansion
infinia cluster rebalance progress
# Data redistributes across new capacity
```

### Scenario 3: "Need to verify data integrity before decommissioning Lustre"

```bash
# Step 1: Verify object count matches
Lustre side:
  find /mnt/lustre -type f | wc -l
  # Example: 5,234,567 files

Infinia side:
  infinia object list mybucket | wc -l
  # Should match: 5,234,567 objects

# Step 2: Verify data completeness
Sample verification:
  for file in $(find /mnt/lustre -type f | shuf | head -100); do
    sum=$(md5sum "$file" | cut -d' ' -f1)
    oid=$(infinia object find mybucket name:"$(basename $file)")
    isum=$(infinia object get mybucket "$oid" | md5sum | cut -d' ' -f1)
    [ "$sum" == "$isum" ] && echo "OK" || echo "MISMATCH: $file"
  done

# Step 3: Check Infinia performance
infinia bucket perf mybucket
# Operations should be normal, no errors

# Step 4: Final sign-off
# Document:
# - Total objects: 5,234,567
# - Data verified (sample of 100)
# - Performance: Normal
# - No corruption detected
# - Safe to decommission Lustre
```

### Scenario 4: "Backup strategy with CoW snapshots"

```bash
# Old Lustre approach:
# Monthly full backup (24 hours)
# Weekly incremental (2 hours)
# = 30 days * 0.5TB = 15TB archive storage

# Infinia approach:
infinia snapshot schedule create daily mybucket daily 02:00
infinia snapshot schedule create weekly mybucket weekly 03:00
infinia snapshot schedule create monthly mybucket monthly 04:00

# Keep all:
# Daily for 30 days: 30 snapshots
# Weekly for 12 months: 52 snapshots
# Monthly for 7 years: 84 snapshots
# Total: 166 snapshots, ~100GB overhead (with CoW dedup)

# Recovery is instant:
# "Data corruption detected on Friday"
# → Restore Thursday snapshot (instant, no data loss)
# → Previous day's work might be lost, but data intact

# Cost: 100GB vs 15TB = 150x better
```

---

## Key Takeaways

1. **Distributed architecture** — no single point of failure, easy scaling
2. **Replication automatic** — specify factor, cluster handles distribution
3. **Snapshots are native** — CoW makes them instant and cheap
4. **Migration from Lustre is straightforward** — parallel run then switchover
5. **Operations simpler than Lustre** — less tuning, more automation
6. **Monitoring focuses on node health** — not MDS bottleneck
7. **Dedup + snapshots = massive space savings** — 10-100x common

---

## How Ready Are You?

Can you explain these?
- [ ] Infinia distributed architecture and replication
- [ ] How to add/remove nodes gracefully
- [ ] Snapshot snapshots for backup strategy
- [ ] Migration strategy from Lustre
- [ ] Troubleshooting degraded clusters

If you checked all boxes, you're ready for Week 7 exercises.

---

Powered by UQS v1.8.5-C
