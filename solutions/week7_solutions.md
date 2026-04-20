# Week 7 Solutions: Lustre & Infinia

---

## Lesson 7.1: Lustre Fundamentals

### Challenge Questions Answers

**1. MDS Function**
```
Answer: B) Manage directory structure and metadata

Explanation:
MDS = Metadata Server

Responsibilities:
✓ Directory structure (/path/to/file hierarchy)
✓ File attributes (permissions, owner, size, timestamps)
✓ Lock management (file access serialization)
✓ Inode allocation
✓ Layout decisions (which OSTs to stripe to)

Not responsible for:
✗ Actual data storage (that's OSS)
✗ Network routing (not in data path)
✗ User authentication (that's OS/kernel)

Why this matters for L1 TSE:
- MDS is bottleneck for metadata-heavy workloads
- Performance of `ls`, `stat`, file creation depends on MDS
- Tuning MDS is complex (ram, CPU, network)
- Infinia solves this by distributing metadata
```

**2. Stripe Benefits**
```
Answer: C) For parallel I/O performance

Explanation:
Striping across multiple OSSs:
- Data split into chunks
- Each chunk on different OSS
- Multiple OSSs read/write in parallel
- 4 OSSs = 4x throughput potential

Example:
File: 1GB, 4 OSSs, 1MB stripe size
├─ Stripe 0 (1MB) → OSS0: read 20MB/s
├─ Stripe 1 (1MB) → OSS1: read 20MB/s
├─ Stripe 2 (1MB) → OSS2: read 20MB/s
├─ Stripe 3 (1MB) → OSS3: read 20MB/s
└─ Total throughput: 4 × 20MB/s = 80MB/s

vs. No striping (single OSS):
- Throughput: 20MB/s (4x slower)

Not primarily for:
A) Redundancy (RAID is separate from striping)
B) Fault tolerance (striping doesn't help if one copy lost)
D) Compression (not related)
```

**3. Why MDS is Bottleneck**
```
Answer: B) Single point for all metadata operations

Explanation:
All metadata must go through one MDS:
- Every file create → MDS
- Every permission check → MDS
- Every directory list → MDS
- Every attribute get/set → MDS

Throughput limit:
- Single MDS: 1000-5000 ops/sec
- Application needs: 100,000+ ops/sec (with many clients)
- Result: Clients wait in queue (slower response)

Lustre tries to mitigate:
- Caching on clients (helps repeated access)
- Batching operations
- Multiple MDSs (federation, complex)

Infinia solves:
- Distributed metadata (no single MDS)
- Operations go to nearest/fastest node
- Scales with cluster size
```

**4. Full OST Problem**
```
Answer: A) Rebalance needed

Explanation:
When one OST is full but others empty:

Data distribution is imbalanced:
Old files striped across 4 OSSs (when all had space)
New files try to stripe across 4, but one is full
Result: New files use only 3 OSSs effectively
And: Load is unbalanced

Issues this causes:
- 3 OSSs get all new traffic (bottleneck)
- 1 full OSS is unavailable (risk)
- Some data is inaccessible (if full OST fails)

Solutions (in order):
1. Migration: lfs migrate -c -1 /mnt/lustre/dir
   Move files from full OST to empty ones

2. Cleanup: rm old files on full OST
   Free space for new data

3. Expansion: Add new OSTs to cluster
   Automatically used for new files

4. Preventive:
   Monitor space regularly
   Set quotas
   Alert when >80% full
```

**5. Write Operation Path**
```
Answer: Data goes directly to OSTs, not through MDS

Explanation:
Lustre write architecture:

Step 1: Client → MDS
- Get write lock
- Check permissions
- MDS grants lock

Step 2: Client → OSTs (direct!)
- Write data to multiple OSTs (striped)
- Bypasses MDS entirely
- Parallel writes to multiple OSSs

Step 3: Client → MDS
- Acknowledge write
- Update metadata
- Release lock

Why this design?
- MDS is bottleneck for data (avoids it)
- Data goes directly to storage
- Multiple OSSs handle I/O in parallel
- MDS handles only control plane

If MDS was in data path:
- Would be 4x slower (single MDS can't keep up)
- Network link to MDS would saturate
- Storage network underutilized

This is key insight: Separate data and metadata paths
```

---

## Lesson 7.2: DDN Infinia - Object Storage Revolution

### Challenge Questions Answers

**1. CoW Snapshot Speed**
```
Answer: B) No data copying, just metadata pointers (CoW)

Explanation:
Copy-on-Write mechanism:

Traditional snapshot (full copy):
Original file: [AAABBBCCC] (on disk)
Create snapshot: Copy [AAABBBCCC] → [AAABBBCCC]' (doubles space, takes time)

CoW snapshot (pointers):
Original file: Points to blocks {A1, A2, A3, B1, B2, B3, C1, C2, C3}
Create snapshot: Points to same blocks {A1, A2, A3, B1, B2, B3, C1, C2, C3}
Space used: 0 bytes (pointers only, <1KB metadata)
Time: 1 millisecond

If file modified:
Original: Changes block A1 → A1'
         Writes A1' (one block)
         Snapshot still sees original A1
Snapshot: Still sees old A1
         No change needed

Result:
- Snapshot instant (no copy needed)
- Snapshot cheap (metadata only)
- Can have thousands per day
- Original and snapshot independent
```

**2. Key-Value Improvements**
```
Answer: D) All of the above

Explanation:
Key-value store advantages:

A) No directory tree overhead ✓
   Traditional: /users/alice/documents/reports/q1/sales.pdf
   └─ Path depth = 5 levels
   └─ Each level = metadata lookup
   └─ Tree must be created (mkdir chain)

   Key-value: Object ID 0x7a3f8e9c + tags {user:alice, type:report}
   └─ Flat namespace
   └─ No tree creation
   └─ Direct lookup by ID

B) Lookups are O(1) ✓
   Directory tree: O(path_depth)
   └─ /a/b/c/d/e/file.txt = 5 lookups minimum

   Hash table: O(1)
   └─ ID → data always same lookup time
   └─ Billions of objects same lookup

C) Distributed across nodes ✓
   Lustre: All metadata → single MDS
   Infinia: Metadata distributed
   └─ Node1 has metadata for objects 0-1billion
   └─ Node2 has metadata for objects 1-2billion
   └─ Load spread across cluster

Combined advantage:
- Billions of objects without performance cliff
- Linear scaling (2 nodes = 2x metadata capacity)
- No single point bottleneck
```

**3. No MDS Bottleneck**
```
Answer: B) Metadata distributed across cluster

Explanation:
Lustre architecture:
All clients → Single MDS → Bottleneck
- 100 clients all asking MDS for metadata
- MDS can handle maybe 5000 ops/sec
- Each client getting ~50 ops/sec (way too slow)

Infinia architecture:
Client1 → Node1 (manages objects 0-1B)
Client2 → Node2 (manages objects 1-2B)
Client3 → Node3 (manages objects 2-3B)

Results:
- Each node independent
- 3 nodes = 3x capacity
- No queue (requests spread)
- No bottleneck (load distributed)
- Linear scaling (add nodes = add capacity)

Analogy:
Lustre: One teller at bank, 100 customers waiting
Infinia: 5 tellers, 100 customers (each teller has 20)
Result: 5x faster service
```

**4. Dedup Effectiveness**
```
Answer: D) All of the above

Explanation:
When dedup works best:

A) With many snapshots (identical blocks) ✓
   Snapshot 1: data blocks {1, 2, 3, 4, 5}
   Snapshot 2: Same blocks {1, 2, 3, 4, 5} (no copy, same pointers)
   Modify original: Block 1 → 1'
   Snapshots: Still see original 1
   Dedup: Blocks 2-5 shared 10x (10 snapshots)
   Storage: 1 copy of blocks 2-5, 10 copies of block 1
   Result: 10x dedup ratio

B) With VMs (cloned systems) ✓
   Master VM: 10GB (OS, common apps)
   Clone VM1: Pointer to master + 100MB changes
   Clone VM2: Pointer to master + 150MB changes
   ...
   Clone VM100: Pointer to master + 200MB changes
   Storage: 10GB + (100+150+...200)MB = ~10.5GB
   vs Lustre: 10GB × 100 = 1TB
   Savings: 99x

C) With backup data (duplicates) ✓
   Daily backup: Incremental (only changes)
   Day 1: 1GB new data
   Day 2: 50MB new + 950MB from day 1 (incremental)
   Day 3: 50MB new + 950MB from day 2 (which is day 1 data)
   ...
   Result: Lots of duplicated blocks across backups
   Dedup: Shares common blocks
   Storage: 1GB + 50MB × 30 days ≈ 2.5GB
   vs full copy: 1GB × 30 = 30GB
   Savings: 12x

Conclusion: Dedup is most effective with snapshots/clones/backups
```

**5. Workload Comparison**
```
Answer: HPC vs Metadata-Intensive

HPC Simulation Workload:
- Files: 100 huge files (1GB each)
- Access: Sequential (read input, compute, write output)
- Metadata ops: Few (file count is low)
- Snapshots: Never
- Recommendation: EITHER (maybe slight edge to Lustre)

AI Model Versioning Workload:
- Files: 1000s of models + checkpoints
- Access: Random (model reads different sections)
- Metadata ops: Many (directory structure, find by tag)
- Snapshots: Frequent (hourly)
- Dedup: Extreme (many similar models)
- Recommendation: INFINIA (clear winner)

Why:
Lustre: Metadata bottleneck (1000s of files slow)
Infinia: Scales effortlessly (distributed metadata)

Lustre: Expensive snapshots (not practical hourly)
Infinia: Instant snapshots (use freely)

Lustre: No dedup (1000 40GB models = 40TB)
Infinia: Dedup (1000 40GB models = 4TB with 10x dedup)
```

---

## Lesson 7.3: Infinia Operations

### Challenge Questions Answers

**1. Node Failure Impact**
```
Answer: C) Still operational (2 replicas sufficient)

Explanation:
Replication factor = 3 (standard):
Object A:
- Primary copy: Node1
- Replica 1: Node2
- Replica 2: Node3

If Node1 fails:
- Clients read from Node2 or Node3 (still have copy)
- Data not lost
- Rebuild automatically copies to another node
- No downtime

If Node2 fails too:
- Still have Node3 (can still operate)
- Higher risk (only 1 copy left)

If Node3 fails:
- System offline (no quorum for new writes)
- OR system goes read-only (can read but not write)

Lustre comparison:
Lustre striped across 4 OSSs:
- Lose 1 OSS = partial data loss (1/4 of file)
- Must recover from backup or RAID

Infinia replication:
- Lose 1 node = no data loss (2 replicas left)
- Lose 2 nodes = still OK (1 replica left, but rebuild)
```

**2. Metadata Speed Reason**
```
Answer: B) Distributed (no single point)

Explanation:
Lustre: Centralized MDS
- All metadata → single server
- If 1000 clients each do 10 ops/sec = 10,000 ops/sec needed
- Single MDS can handle ~5000 ops/sec
- Half the requests queue (slow)

Infinia: Distributed metadata
- Metadata sharded across 5 nodes
- Each node handles 2000 ops/sec
- Total capacity: 10,000 ops/sec
- All requests handled without queue (fast)

Parallel example:
Lustre 1 MDS handling 10,000 clients:
Time to respond: 10,000 / 5000 = 2 seconds (client waits)

Infinia 5 nodes, distributed metadata:
Time to respond: 10,000 / 10,000 = 1 second (client waits)

Actually better:
Infinia keeps metadata local to data node:
- Data on Node1, metadata on Node1
- Client reading from Node1 = local lookup (microseconds)
- No network latency
```

**3. Migration Strategy**
```
Answer: Phased approach with parallel running

Detailed strategy:

Phase 1: Assessment (Week 1)
✓ Inventory all data: file count, sizes, access patterns
✓ Backup to tape (safety net)
✓ Document applications using storage
✓ Identify cutover window (when outage acceptable)

Phase 2: Parallel Operation (Weeks 2-4)
✓ Deploy Infinia cluster
✓ Copy 50TB from Lustre → Infinia (5 days at 200MB/s)
✓ Run test jobs on Infinia
✓ Compare results (should be identical)
✓ Both systems active (read from either, write to both)

Phase 3: Cutover (Day 1)
✓ Final sync: Lustre → Infinia
✓ Point applications to Infinia
✓ Monitor heavily (1st hour = watch for issues)
✓ Fallback ready: Can revert to Lustre if needed

Phase 4: Validation (Week 5)
✓ Run week of jobs normally
✓ No performance regressions
✓ All users happy
✓ Backup strategy working

Phase 5: Decommission (Week 6)
✓ Archive Lustre data (long-term backup)
✓ Shutdown Lustre (no turning back)
✓ Reclaim hardware

Risk mitigation:
- Keep Lustre running 1 week after switchover (fallback)
- Have rollback procedure tested
- Monitor both systems during parallel run
- Verify checksum on sample files after copy
```

**4. Space Overhead**
```
Answer: C) Depends on dedup/compression

Explanation:
Naïve answer:
3x replication = 3x space
With snapshots: 3x + snapshot copies

More accurate answer:
With CoW snapshots:
- Original: 100GB
- Snapshot 1: <1GB overhead (pointers only)
- Snapshot 2: <1GB overhead
- Snapshot 100: <1GB overhead
- Total: 100GB + 100MB overhead (not 300GB per snapshot)

With dedup:
- Original: 100GB (5 identical objects, each 20GB)
- Physical stored: 20GB (dedup sees identical blocks)
- Logical: 100GB (application sees 5 objects)
- Overhead: 0% (already deduped)

With compression:
- Original: 100GB
- Compressed: 50GB (2:1 ratio for typical data)
- With 3x replication: 150GB total
- But adds snapshots: +0.1GB per snapshot (negligible)

Realistic:
100GB data:
- Best case (compressible, dedup-able): 25GB
- Average case: 50GB (just replication)
- Worst case (incompressible): 300GB (3x replication)

Formula:
Total space = Logical size × Compression ratio × Replication factor + Snapshot overhead

Example: 1TB files
= 1TB × 0.5 (compression) × 3 (replication) + 10MB (snapshots)
= 1.5TB + 0.01TB
≈ 1.5TB
```

**5. Metadata Workload Optimization**
```
Answer: Increase cache, smaller dedup block, lower replica

Detailed recommendations for 10M files workload:

1. Increase cache:
   infinia bucket config set mybucket cache_size=500GB
   Why: Metadata heavy = keep metadata in RAM
   Trade: More memory needed

2. Dedup block size:
   infinia bucket config set mybucket dedup_block=4KB
   Why: 10M files = many small objects
   Trade: More metadata (smaller blocks = more pointers)

3. Replica factor:
   infinia bucket config set mybucket replica=2
   Why: Metadata can be regenerated if lost
   Trade: Less fault tolerance (only 1 node failure allowed)

4. Enable compression:
   infinia bucket config set mybucket compression=aggressive
   Why: Metadata tends to compress well
   Trade: Slight CPU overhead

5. Distribute metadata:
   infinia bucket config set mybucket metadata_distribution=wide
   Why: Spread metadata across all nodes
   Trade: Slightly higher latency (but parallelism wins)

6. Batch operations:
   (At application level)
   Instead of: Create 1 file, wait
   Better: Batch create 1000 files at once
   Why: Amortize MDS overhead

Combined effect:
Single MDS (Lustre): 5000 ops/sec, bottleneck
Infinia optimized: 50,000+ ops/sec, no bottleneck
Result: 10x faster metadata operations
```

---

## Exercise Solutions

### Exercise 7.1: Lustre Diagnostics

**Problem 1: Component Identification**

Expected findings:
```
MDS location: mds.example.com
Number of OSSs: 4
Total capacity: 40TB (4 OSSs × 10TB)
RAID level: RAID6 (common for protection)

Signs of healthy Lustre:
✓ MDS has 16+ GB RAM
✓ All OSSs online and healthy
✓ Space distribution relatively balanced
✓ No degraded OSTTs
```

**Problem 3: Lustre Issues Diagnosis**

Common findings:
```
If one OST at 95% full:
- Identify files on that OST
- Migrate to others: lfs migrate -c 4 /path/file
- Clean up old data

If MDS CPU > 50%:
- Check metadata ops: lctl get_param mdt.*.operations
- Might be slow clients doing repeated metadata ops
- Could require MDS tuning or client fixes

I/O patterns:
- MDS disk: Metadata journal (usually lower throughput, important)
- OST disks: Data (usually high throughput)
```

### Exercise 7.2: Infinia CoW and Snapshots

**Problem 1: Snapshot Speed**

Expected results:
```
Snapshot create time: <1 second (vs Lustre's 30 minutes)
Space per snapshot: <1MB (vs Lustre's full copy)
Total 3 snapshots: ~3MB overhead (vs Lustre's 3GB)
Space savings: 1000x

Logical vs Physical:
- Logical: 1GB (data size)
- Physical: 1.1GB (data + 3 snapshot metadata)
```

**Problem 2: Dedup Effectiveness**

Expected results:
```
Three identical objects:
Logical: 3 × file size
Physical: 1 × file size + metadata
Dedup ratio: 3:1

With snapshots:
Snapshot shares blocks with original
Dedup ratio can reach 4:1 or higher

Real-world impact:
VDI with 100 identical VMs:
Logical: 100 × 10GB = 1TB
Physical: 10GB + 0.5GB = 10.5GB
Savings: 99%
```

**Problem 3: Metadata Performance**

Expected results:
```
Creating 10,000 objects: <10 seconds (Infinia)
Listing 10,000 objects: <1 second (Infinia)

Lustre equivalent: 5-10 seconds for listing

Performance advantage: 5-10x for metadata operations
```

### Exercise 7.4: Migration Plan

**Phased Migration Timeline:**

```
Week 1: Planning
- File count: 5,234,567
- Space: 50TB
- Growth: 500GB/month
- Risk: High (production system)

Week 2-3: Deployment
- Infinia cluster: 5 nodes
- Buckets: prod, staging, archive
- Network: 10Gbps connectivity

Week 4: Data Copy
- Method: rsync + verification
- Time: 250 hours at 200MB/s = ~10 days
- Parallel: Multiple threads = 5 days
- Verification: Checksum every 100 files

Weeks 5-6: Parallel Running
- App reads from both systems
- Writes cached (replicate to both)
- Performance baseline on Infinia: OK?
- User acceptance testing: Pass?

Week 7: Switchover
- Final sync
- Apps point to Infinia
- Monitor: Every 10 minutes first 4 hours, then hourly
- Fallback prepared if needed

Week 8: Validation
- Production jobs running on Infinia
- No performance regressions
- Backup strategy working
- Users satisfied

Week 9: Decommission
- Archive Lustre
- Shutdown services
- Reclaim hardware
```

---

## Week 7 Key Comparison Summary

```
Aspect              Lustre                Infinia
─────────────────────────────────────────────────────
Metadata            Centralized MDS       Distributed
Bottleneck          MDS (5K ops/sec)      None (100K+ ops/sec)
Snapshots           Minutes (expensive)   Instant (free)
Snapshot space      Full copy per snap    CoW (negligible)
Dedup               No built-in           Yes (10-100x)
Compression         Optional              Included
Scaling             Complex (rebalance)   Linear (add nodes)
Typical files       100K-1M               1B+
Ideal workload      HPC sequential I/O    Metadata-heavy
API                 POSIX (mount)         S3, native
Migration           Full copy/resync      Continuous or batch
Cost/TB (typical)   $100                  $80 (with dedup savings)
```

---

## L1 TSE Key Takeaways

**When to recommend Lustre:**
- Pure sequential I/O workloads (HPC)
- Existing integrations with POSIX
- Simple deployment needed
- Cost-sensitive all-throughput

**When to recommend Infinia:**
- Metadata-heavy workloads (file servers)
- Frequent snapshots needed (backup, versioning)
- Dedup-friendly workloads (VMs, backups)
- Cloud-native applications (S3 API)
- Multi-tenant environments

**Migration path:**
- Large Lustre clusters → Infinia
- New deployments → Infinia
- Legacy systems → Lustre until EOL

**Support focus:**
- Lustre: Troubleshoot MDS bottleneck, balance OSSs
- Infinia: Monitor distributed health, manage snapshots

---

Powered by UQS v1.8.5-C
