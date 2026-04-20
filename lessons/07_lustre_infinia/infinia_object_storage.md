# Lesson 7.2: DDN Infinia - Object Storage Revolution (★ TOP PRIORITY ★)

**Time:** 25-30 minutes
**Why it matters:** Infinia represents DDN's next-generation architecture. Unlike POSIX-based Lustre, Infinia is built ground-up as object storage with copy-on-write, snapshots, and key-value optimization. This is the future of DDN storage and critical for L1 TSEs
**Skills:** Understand Infinia architecture, identify Infinia advantages, recognize Infinia use cases, diagnose Infinia-specific issues

---

## Quick Review: Why Infinia Exists

```
Traditional Storage Problems (Lustre/POSIX):
├─ MDS is central bottleneck (metadata)
├─ Snapshots are complex (require quiescing)
├─ Large file counts tank performance
├─ Deduplication must be bolted on
├─ Backup/recovery is time-consuming
├─ Scaling metadata is hard
└─ Not optimized for modern workloads

Infinia Solution:
├─ Distributed metadata (no bottleneck)
├─ Native snapshots with CoW (instant, space-efficient)
├─ Object-based (scalable to billions)
├─ Built-in dedup/compression
├─ Efficient recovery from snapshots
├─ Cloud-native architecture
└─ Optimized for modern use cases (AI, analytics, cloud)
```

---

## Key Concepts

### 1. What is Infinia? (Paradigm Shift)

**Not a filesystem, but a storage system:**

```
Traditional View (POSIX Filesystem):
Client → Open file → Read/Write blocks → Close
         (Thinks of files in directory tree)

Infinia Model (Object Storage):
Client → Write object with metadata → Retrieve by ID
         (Objects are first-class, not files in trees)

Key Difference:
File = "Document at path /users/alice/report.pdf"
Object = "This data blob with ID 0x123abc and tags {owner:alice, type:pdf}"
```

**Infinia Architecture:**

```
┌──────────────────────────────────────────────┐
│        Application Layer                     │
│  (S3 API, native protocol, NFS gateway)     │
└──────────────┬───────────────────────────────┘
               │
        ┌──────▼──────────────────────────┐
        │  Distributed Metadata Service   │
        │  (Key-Value Store)              │
        │  No single point of failure     │
        └──────┬───────────────────────────┘
               │
    ┌──────────┼──────────────────────┐
    │          │                      │
┌───▼──┐  ┌───▼──┐              ┌───▼──┐
│Node1 │  │Node2 │    ...       │NodeN │
│ KV   │  │ KV   │              │ KV   │
│Store │  │Store │              │Store │
│ CoW  │  │ CoW  │              │ CoW  │
│ Data │  │ Data │              │ Data │
└──────┘  └──────┘              └──────┘
   ↓         ↓                     ↓
[Disk]   [Disk]   ...         [Disk]
```

**Why "Infinia"?**
```
Infinia = Infinity + (Storage) + (Architecture)
= Infinitely scalable, infinitely flexible storage
```

### 2. Core Technologies: Copy-on-Write (CoW)

**How CoW works in Infinia:**

```
Traditional snapshot (expensive):
File original:     [AAABBBCCC] (on disk)
Snapshot created:  [AAABBBCCC] (copied!)   ← Full copy needed
                                           ← Takes time, space

Result: Large files = slow snapshots

CoW snapshot (efficient):
File original:     Points to [AAABBBCCC] blocks
Snapshot:          Points to same [AAABBBCCC] blocks (no copy!)

When file modified:
Original:          Modified A = [A'AABBBCCC]  ← Copy only changed block
Snapshot:          Still points to original [AAABBBCCC]

Result: Instant snapshots, space-efficient
```

**Implications for Infinia:**

```
CoW Snapshots:
- Create thousands in seconds (not hours)
- Each snapshot costs only metadata
- Modify original without affecting snapshot
- Perfect for backup strategies
- Recovery is simple: restore from snapshot

Example:
Monday 2AM: Snapshot (instant, 0.1MB)
Monday 3AM: Snapshot (instant, 0.1MB)
Monday 4AM: Snapshot (instant, 0.1MB)
... hourly for 30 days ...
Total snapshots: 720
Total space overhead: ~72MB (just metadata)

Try this with traditional snapshot: Impossible
```

### 3. Key-Value Store Architecture

**Object storage model:**

```
Traditional File:
Path: /data/user/documents/report.pdf
Data: [binary content]
Metadata: owner=alice, size=2.5MB, created=2024-01-15

Infinia Object:
ID: 0x7a3f8e9c (hash of content + metadata)
Data: [binary content]
Metadata: owner=alice, size=2.5MB, created=2024-01-15
Tags: {department:sales, project:q1review}
Attributes: {retention:7years, encryption:AES256}
```

**Benefits of Key-Value:**

```
Scalability:
- No directory tree overhead
- Can store billions of objects
- Each object independent
- Metadata lookup is O(1) not O(path_depth)

Performance:
- Metadata stored alongside data
- No separate metadata server bottleneck
- Parallel metadata operations
- Load distributed across all nodes

Flexibility:
- Tag-based organization (not just hierarchy)
- Custom metadata (beyond standard attributes)
- Advanced queries (find by tag, not path)
- Object versioning is native
```

**Traditional vs Infinia:**

```
Lustre (POSIX):
/mnt/lustre/projects/sales/Q1/report.pdf
└─ Requires: projects/ exists, sales/ exists, Q1/ exists
└─ Each directory level = metadata lookup
└─ Tree depth limits performance
└─ Reorganizing = move operations

Infinia (Object):
Object ID: 0x7a3f8e9c
Tags: {project:sales, quarter:Q1, type:report}
└─ No directory requirement
└─ Flat namespace (billions of objects)
└─ Find by tag, not path
└─ Reorganizing = update tags
```

### 4. Distributed Metadata

**The Infinia advantage:**

```
Lustre (Centralized MDS):
All clients → Single MDS → Bottleneck
              1 server must handle all

Infinia (Distributed):
Client A → Node1  (metadata stored here)
Client B → Node2  (different metadata)
Client C → Node3  (different metadata)
Client D → Node1  (same as A, but load-balanced)

No single point of failure:
- Node down? Others handle requests
- Scale metadata with cluster size
- 10 nodes = 10x metadata capacity (vs Lustre's 1x)
```

**Consistency model:**

```
Infinia uses eventual consistency + strong guarantees:
- Write to one replica → Acknowledged
- Replicated to other nodes asynchronously
- Read always sees latest (versioning)
- Conflicts resolved by timestamp

Real-world: Indistinguishable from strong consistency for typical workloads
```

### 5. Snapshots and Versioning

**Native snapshot support:**

```
Infinia Snapshots (CoW):
Create Instant snapshot:
  $ infinia snapshot create volume-1 snap-2024-01-15

Result:
- snapshot-2024-01-15 created in 0.1 seconds
- Takes 0.1MB space (just metadata pointers)
- Can create 1000 per day if needed

Restore from snapshot:
  $ infinia volume restore snap-2024-01-15

Result:
- Original volume reverted to snapshot state
- Or create new volume from snapshot (clone)
- Both instant due to CoW

vs. Lustre:
- Snapshots require quiescing (pause I/O)
- Takes hours for large files
- Expensive (full copy)
- Complex to manage
```

### 6. Data Protection & Dedup

**Built-in capabilities:**

```
Compression:
- Automatic (transparent to app)
- Varies by object type
- Typical: 2-4x compression for typical files
- 0.5x for already-compressed (video, photos)

Deduplication:
- Content-based (hash of data)
- Detects identical blocks across entire system
- 4KB block granularity
- Especially effective for:
  * Snapshots (CoW creates many identical blocks)
  * VDI environments (identical VM copies)
  * Backup storage (duplicate data)
- Space savings: 10-100x in favorable workloads

Encryption:
- End-to-end (data encrypted before sending)
- Hardware accelerated (AES-NI)
- Key management (internal or external)
- At-rest encryption (data encrypted on disk)

Example:
Store 1TB of data:
- With compression: 250-500GB
- With dedup (20 snapshots): 50-100GB
- Result: 95% less space than original
```

### 7. Performance Characteristics

**Where Infinia excels:**

```
Metadata Operations:
Operation          Lustre              Infinia
Directory list     1000-5000 ops/sec   100,000+ ops/sec
File create        500-1000 ops/sec    50,000+ ops/sec
Attribute update   1000-2000 ops/sec   100,000+ ops/sec
Snapshot create    Minutes             Milliseconds
Snapshot delete    Minutes             Instant (with CoW)

Scalability:
Cluster size       Metadata capacity
1 node (Lustre)    ~100M files/dir    (Single MDS limit)
100 nodes (Infinia) ~1B files         (Distributed, scales)

I/O Performance:
- Sequential I/O: Comparable to Lustre (both can saturate network)
- Random I/O: Better on Infinia (distributed metadata = less contention)
- Concurrent clients: Infinia wins (no metadata bottleneck)

Ideal Workloads:
Infinia excels at:
✓ High metadata operations (many small files)
✓ Frequent snapshots (backup, versioning)
✓ Deduplication candidates (VMs, backup data)
✓ Cloud-native apps (S3 API)
✓ AI/ML with versioning (experiments, checkpoints)

Lustre still better for:
✓ Pure sequential I/O (HPC computation)
✓ Existing integrations (some legacy apps expect POSIX)
✓ Cost-sensitive all-sequential (simpler, cheaper)
```

### 8. APIs and Access

**Multiple access methods:**

```
Infinia provides:

1. Native Protocol (Best performance)
   - Optimized for object access
   - Direct to storage nodes
   - Lowest latency

2. S3 API (Cloud-compatible)
   - aws s3 cp file s3://bucket/key
   - Standard S3 tools (boto3, AWS CLI)
   - Perfect for cloud integration

3. NFS Gateway (POSIX Compatibility)
   - Legacy app support
   - Mount as /mnt/infinia
   - Acts like filesystem
   - Proxies to object storage underneath
   - Performance: Not as good as native (gateway overhead)

4. REST API (Web-friendly)
   - HTTP GET/PUT/DELETE
   - Easy integration with web apps
   - Standard REST conventions
```

### 9. Operational Differences from Lustre

**What L1 TSEs need to know:**

```
Lustre Concept          Infinia Equivalent
─────────────────────────────────────────────
Directory              Bucket (container)
File                   Object (data + metadata)
File path              Object key/ID + tags
Striping               Automatic (no user config needed)
Snapshots              Native CoW snapshots
MDS                    Distributed key-value store
XFS backend            Proprietary optimized backend
mount -t lustre        infinia connect bucket

Diagnosis:
Tool                   Lustre              Infinia
─────────────────────────────────────────────
Check health           lfs df              infinia status
Monitor operations     lctl                infinia metrics
Check snapshots        lfs snapshot_list   infinia snapshots ls
Optimize config        lfs setstripe       (automatic)
Troubleshoot           lustre logs         infinia logs
```

---

## Essential Commands

### Infinia Status and Management

```bash
# Check cluster status
infinia cluster status
# Shows all nodes, capacity, health

# Check bucket (filesystem equivalent)
infinia bucket list
# List all buckets

# Create bucket
infinia bucket create mybucket

# Check space
infinia bucket info mybucket
# Shows used/available space

# Monitor performance
infinia metrics get --bucket mybucket
# Operations, latency, throughput
```

### Snapshots

```bash
# Create snapshot (instant)
infinia snapshot create mybucket snap-backup-jan15
# Result: instant, no downtime

# List snapshots
infinia snapshot list mybucket

# Restore from snapshot
infinia bucket restore snap-backup-jan15 mybucket-restored
# Creates new bucket from snapshot (no data loss from original)

# Delete snapshot
infinia snapshot delete snap-backup-jan15

# Schedule automated snapshots
infinia snapshot schedule create daily-backup mybucket daily 02:00

# View dedup savings
infinia snapshot info snap-backup-jan15
# Shows space used vs. space saved by CoW
```

### Access and Performance

```bash
# Check who's accessing
infinia access list mybucket
# Shows clients, tokens, permissions

# Monitor I/O performance
infinia perf top --bucket mybucket
# Live view of operations

# Check compression ratio
infinia bucket stats mybucket | grep compression
# Real compression achieved

# Check dedup effectiveness
infinia bucket stats mybucket | grep dedup
# Space saved by deduplication
```

---

## Interactive Exercise: Understand Infinia Concepts

**Task 1: Compare CoW vs Traditional Snapshot**

```bash
# Infinia: Create snapshot (instant)
infinia snapshot create mybucket snap1
time infinia snapshot create mybucket snap2
time infinia snapshot create mybucket snap3
# Record: Should be sub-second

# Lustre: Create snapshot (full copy)
# Would take: 30min - 2 hours for large filesystem

# Result: Infinia wins for snapshot frequency
```

**Task 2: Metadata Performance**

```bash
# Create many objects
for i in {1..10000}; do
  echo "data" > /tmp/file_$i
  infinia object put mybucket key_$i /tmp/file_$i
done

# Time directory listing
time infinia object list mybucket | wc -l
# Infinia should list 10,000 objects in <1 second
# Lustre would take seconds or longer
```

**Task 3: Explore Dedup**

```bash
# Create duplicate object
infinia object put mybucket key_1 /tmp/largefile
infinia object put mybucket key_2 /tmp/largefile
# Same file, different key

# Check space
infinia bucket stats mybucket
# Space used < 2x file size (dedup working)

# vs Lustre: Would use 2x space (no dedup)
```

---

## Challenge Questions

**Answer without looking them up:**

1. **CoW Advantage:** Why are Infinia snapshots fast?
   ```
   A) Compression
   B) No data copying, just metadata pointers (CoW)
   C) More hardware
   D) Better network
   ```

2. **Key-Value Benefits:** How does object storage improve metadata?
   ```
   A) No directory tree overhead
   B) Lookups are O(1)
   C) Distributed across nodes
   D) All of the above
   ```

3. **Distributed Metadata:** Why no MDS bottleneck in Infinia?
   ```
   A) Faster hardware
   B) Metadata distributed across cluster
   C) Caching
   D) Compression
   ```

4. **Dedup Effectiveness:** Where does dedup help most?
   ```
   A) With many snapshots (identical blocks)
   B) With VMs (cloned systems)
   C) With backup data (duplicates)
   D) All of the above
   ```

5. **Use Case:** Which workload favors Infinia over Lustre?
   ```
   (Explain reasoning for metadata-heavy vs pure I/O workloads)
   ```

---

## Common L1 TSE Scenarios for Infinia

### Scenario 1: "Need backup strategy, snapshot takes hours with Lustre"

```bash
# Lustre problem: Snapshot too expensive
# Solution: Infinia CoW snapshots

infinia snapshot schedule create hourly-backup mybucket hourly
# Result: Automatic snapshots every hour, instant, cheap

# Can also do:
infinia snapshot schedule create daily-backup mybucket daily 02:00
# Daily at 2am
# Can keep 30 days of daily snapshots for 30x100MB = 3GB overhead

# For recovery:
infinia bucket restore snap-2024-01-15-02 restored-bucket
# Data available immediately, no recovery time
```

### Scenario 2: "Storage is full, need to clean up"

```bash
# First: Check dedup savings (Infinia has this, Lustre doesn't)
infinia bucket stats mybucket
# Shows: 2TB used, but 18TB logical (with dedup: 2TB actual is 18TB data)

# If still full:
# 1. Delete old snapshots
infinia snapshot list mybucket | head -10  # Show oldest
infinia snapshot delete snap-old-2023-q1

# 2. Check compression effectiveness
infinia bucket stats | grep compression
# If low: Already compressed, can't improve

# 3. Expand capacity
infinia cluster expand new-node-ip
# Add disk space
```

### Scenario 3: "VDI environment with 100 identical VMs"

```bash
# Store 100 VM images (1GB each)
# Lustre: 100GB used
# Infinia: ~2GB used (98% dedup savings)

# How:
# - Store one master VM image
# - Clone 99 times
# - Infinia dedup sees identical blocks
# - Only stores once, references many times
# - Modify VM A: Only changed blocks stored (CoW)

# Space savings:
# 100 VMs × 1GB = 100GB logical
# Actual usage: ~2-5GB (98% saved)

# Cost impact: 50x cheaper storage
```

### Scenario 4: "Need versioning for AI model checkpoints"

```bash
# Training AI model, saving checkpoints hourly
infinia snapshot create models snap-epoch-00
# After 1 hour: 5GB checkpoint saved (instant, free-ish)

infinia snapshot create models snap-epoch-01
# After 2 hours: Another 5GB (but only new/changed blocks stored)

infinia snapshot create models snap-epoch-10
# After 11 hours: Still ~5GB logical per checkpoint
# Total: 55GB logical, but only ~15GB actual (due to CoW dedup)

# If bad epoch detected:
infinia bucket restore snap-epoch-05 models-restored
# Revert to epoch 5 (instant, no data loss)

# Try with Lustre: Would need ~550GB storage
```

---

## Key Takeaways

1. **Infinia revolutionizes storage** — object-based, not POSIX filesystem
2. **CoW snapshots are instant** — no copying, just metadata pointers
3. **Key-value eliminates metadata bottleneck** — distributed, scalable
4. **Dedup + compression built-in** — space-efficient by default
5. **Multiple APIs** — S3, native, NFS gateway for compatibility
6. **Metadata-heavy workloads love Infinia** — millions of objects, snapshots
7. **Replace Lustre for new projects** — Infinia is the future

---

## How Ready Are You?

Can you explain these?
- [ ] Why CoW snapshots are fast (no copying)
- [ ] How key-value store eliminates MDS bottleneck
- [ ] Dedup works across snapshots
- [ ] When to recommend Infinia vs Lustre
- [ ] Infinia APIs and access methods

If you checked all boxes, you're ready for Lesson 7.3: **Infinia Operations & Troubleshooting**.

---

Powered by UQS v1.8.5-C
