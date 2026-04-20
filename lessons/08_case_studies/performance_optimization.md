# Lesson 8.3: Performance Optimization Campaign - Strategic Improvement

**Time:** 25-30 minutes
**Why it matters:** Not all challenges are fires. This lesson teaches proactive performance optimization integrating weeks 4-7 (performance, diagnostics, Lustre/Infinia)
**Skills:** Baseline measurement, identifying bottlenecks systematically, optimization prioritization, measuring impact

---

## The Scenario

**Initial Request (not urgent):**
```
From: Research Director
To: Storage Team
Subject: "Can we make storage faster for our ML pipeline?"

"Our training jobs are slower than our competitor's cluster.
They said their storage is 'well-tuned.'
Can you help optimize our system?"

Scope: Voluntary project (nice-to-have, not crisis)
Timeline: Next quarter (3 months)
Goal: 20% improvement in job throughput
Team: L1 TSE lead + engineering
```

---

## Phase 1: Baseline Measurement (Week 1)

### Step 1: Establish Current State

```bash
# Define a representative workload
# (Need to know what "slow" means)

Standard ML job characteristics:
├─ Data size: 100GB
├─ Runtime: 4 hours
├─ Access pattern: Random (typical for ML)
├─ Files: 50,000 small files (metadata heavy)
└─ I/O: 30% read, 70% write (checkpoint)

Baseline metrics (before optimization):
```

### Measurement 1: I/O Performance

```bash
# Run test ML job, measure everything
# Command: strace, iostat, perf, network all running

1. Disk I/O
   iostat -x 1 240  # 4 hour job
   Peak metrics:
   - r/s: 1000
   - w/s: 5000
   - await: 15ms (read), 20ms (write)
   - %util: 75%

2. Network
   iftop -n -s 2  # During job
   Peak: 3Gbps out, 2Gbps in
   Utilization: 30% of 10Gbps capacity

3. CPU
   top during job
   %user: 45%
   %sys: 12%
   %iowait: 25%
   %idle: 18%

4. Metadata
   strace on sample job
   stat() calls: 500/sec
   open/close: 200/sec
```

### Measurement 2: Job Timeline

```bash
Job execution breakdown:
├─ Startup: 5 min (load data, initialize)
├─ Training: 3:45 (actual ML computation)
│  └─ I/O during training: Random access
├─ Checkpoint: 5 min (write state)
└─ Cleanup: 5 min (close files)

Total: 4:00 hours

Bottleneck analysis:
- Startup: CPU and metadata bound
- Training: I/O bound (iowait 25%)
- Checkpoint: Network bound (3Gbps sustained)
```

### Baseline Summary

```
Current performance:
├─ Job time: 4 hours
├─ I/O: 75% disk util, 15ms latency (reasonable)
├─ Network: 30% utilized (headroom available)
├─ CPU: 57% busy (headroom available)
├─ Bottleneck: I/O during training phase
└─ Improvement opportunity: Reduce metadata overhead

Target:
- Goal: 20% improvement (4:00 → 3:12)
- Strategy: Focus on I/O path optimization
```

---

## Phase 2: Diagnosis and Planning (Weeks 2-3)

### Deep Dive: Where's the Time Going?

```bash
# Use perf to see where CPU time goes during job

sudo perf record -F 99 -g python train.py
sudo perf report

Top CPU consumers:
├─ Python training loops: 40%
├─ Filesystem metadata: 12%  ← Opportunity!
├─ Memory allocation: 8%
├─ Network syscalls: 5%
└─ Other: 35%

Metadata breakdown (that 12%):
├─ stat() calls: 6%
├─ open/close: 4%
├─ permission checks: 2%

This is fixable! Metadata is 12% of CPU.
Optimize by 50% = ~6% overall job speedup.
```

### Opportunities Identified

```
Priority 1: Reduce metadata operations (6% potential)
├─ Problem: Job stats every file before read
├─ Solution: Batch stats, or skip redundant checks
├─ Effort: Low (app-level change)
├─ Risk: Low

Priority 2: Improve I/O caching (5% potential)
├─ Problem: Random I/O causes cache misses
├─ Solution: Larger read buffers, prefetching
├─ Effort: Medium (Lustre tuning)
├─ Risk: Low

Priority 3: Optimize checkpoint I/O (8% potential)
├─ Problem: Checkpoint write is sequential but unbuffered
├─ Solution: Use async I/O, larger writes
├─ Effort: Medium (app change)
├─ Risk: Low

Priority 4: Network optimization (3% potential)
├─ Problem: Network only 30% utilized (headroom)
├─ Solution: Larger transfers, less overhead
├─ Effort: Low (config change)
├─ Risk: Very Low

Total potential: 6% + 5% + 8% + 3% = 22% (exceeds 20% goal!)
```

---

## Phase 3: Implementation (Weeks 4-5)

### Optimization 1: Metadata Reduction

**Current code pattern:**
```python
# Bad: Stat every file before reading
for file in file_list:
    size = os.stat(file).st_size
    if size > 1GB:
        read_file(file)
```

**Optimized code pattern:**
```python
# Better: Bulk stat all files
import os
stat_results = {f: os.stat(f) for f in file_list}
for file, stat in stat_results.items():
    if stat.st_size > 1GB:
        read_file(file)
```

**Impact:**
```bash
Before: 50,000 stat() calls spread over 4 hours = 3.5/sec
After: 50,000 stat() calls in 5 seconds = 10,000/sec (burst)

Result:
- Metadata operations concentrated (cache efficient)
- Reduced metadata CPU from 12% to 6%
- Job time: 4:00 → 3:54 (3% improvement) ✓
```

### Optimization 2: Lustre Striping

**Current configuration:**
```bash
lfs getstripe /mnt/research
stripe_count: 2
stripe_size: 1M
```

**Issue:** Job is reading 50GB sequentially. Two OSSs can't handle all the traffic. Should stripe across 4.

**Optimization:**
```bash
lfs setstripe -c 4 -S 2M /mnt/research/ml-jobs
# 4 OSSs, 2MB stripes (larger for sequential)

Re-run baseline job with new stripe:
Before: await 15ms, throughput 1000 r/s
After: await 10ms, throughput 2500 r/s (2.5x better!)

Result:
- I/O bound section: 3:45 → 2:30 (33% faster!)
- But overall job: 3:54 → 3:20 (11% faster)
- Rest of job not I/O bound, so limited speedup
```

### Optimization 3: Checkpoint Buffering

**Current checkpoint:**
```python
# Inefficient: Write each parameter separately
for param_name, param_tensor in model.items():
    with open(f'checkpoint/{param_name}.pkl', 'wb') as f:
        pickle.dump(param_tensor, f)
```

**Issue:**
- 50,000 files created
- Each file write is separate syscall
- No buffering or batching
- Network overhead per file

**Optimized:**
```python
# Better: Write entire checkpoint as single file
checkpoint_data = {name: tensor for name, tensor in model.items()}
with open('checkpoint/model.pkl', 'wb') as f:
    pickle.dump(checkpoint_data, f)
```

**Impact:**
```bash
Before: 50,000 files, 5 min checkpoint time
After: 1 file, 2 min checkpoint time

Result:
- Checkpoint: 5 min → 2 min (60% faster!)
- Job phases: Training 3:45 + Checkpoint 2:00 = 5:45 (before optimization)
            Training 3:45 + Checkpoint 2:00 = ... wait, still 5:45

Wait, what? Optimization didn't work?
Reason: Training phase is much longer (3:45 vs 2:00)
        Checkpoint optimization helps, but overall job still waiting for training

KEY INSIGHT: Optimize the longest phase first!
Should have focused on training (3:45) not checkpoint (5 min)
```

### Optimization 4: Memory and CPU Tuning

**Issue:** Training is 3:45 and bottleneck. Can we optimize the training loop?

**Analysis:**
```bash
perf shows during training:
- CPU at 50% (could use more)
- Memory at 60% of available (could cache more)
- I/O wait at 25%

Opportunity:
- Increase batch size (uses more CPU, less I/O)
- Add data prefetching (fills cache while training)
- Enable GPU acceleration (if available)
```

**Optimization:**
```python
# Original: Batch size 32
dataloader = DataLoader(dataset, batch_size=32, num_workers=2)

# Optimized: Larger batch size, more workers
dataloader = DataLoader(dataset, batch_size=64, num_workers=8, prefetch_factor=2)

Result:
- CPU utilization: 50% → 75% (better usage)
- I/O wait: 25% → 12% (prefetching helps)
- Training: 3:45 → 3:10 (15% faster!)

This addresses the actual bottleneck!
```

---

## Phase 4: Verification (Weeks 6-7)

### Measure Combined Impact

```bash
Run full test job with all optimizations:

Before optimization:
├─ Metadata CPU: 12%
├─ Metadata time cost: 3% of job
├─ Lustre stripe: 2 OSS
├─ Checkpoint: 50K files, 5 min
├─ ML batch size: 32
└─ Total job time: 4:00

After optimization:
├─ Metadata CPU: 6% (batched)
├─ Metadata time cost: 1.5% of job
├─ Lustre stripe: 4 OSS
├─ Checkpoint: 1 file, 2 min
├─ ML batch size: 64
└─ Total job time: 3:10

Overall improvement: (4:00 - 3:10) / 4:00 = 21%
                                 ↑
                         EXCEEDS 20% GOAL! ✓

Breakdown of improvements:
├─ Metadata optimization: 1.5% (less CPU)
├─ Lustre striping: 11% (I/O reads 2.5x faster)
├─ Checkpoint optimization: 0.5% (checkpoint smaller)
└─ ML batch size: 7% (better CPU/memory usage)
```

### Regression Testing

```bash
Verify we didn't break anything:

1. Job correctness
   - Output files match expected
   - Checkpoint/restore works
   - Training converges correctly

2. Performance consistency
   - Run 5 times, measure variance
   - Before: 4:00 ± 5 min
   - After: 3:10 ± 3 min (better!)
   - Less variance = more predictable

3. Other workloads unaffected
   - Check other jobs still perform OK
   - New Lustre striping affects all files
   - Verify no slowdown for other users
```

---

## Phase 5: Documentation and Rollout (Weeks 8-12)

### Documentation

```
Performance Optimization Report
===============================

Summary:
- Goal: 20% improvement
- Achieved: 21% improvement
- Timeline: 7 weeks (within budget)
- Risk: Low (all changes tested)

Changes made:
1. Metadata batching (app code change)
2. Lustre stripe increase (cluster config change)
3. Checkpoint redesign (app code change)
4. ML batch size increase (model tuning)

Implementation checklist:
□ Deploy to staging
□ Run full regression test (2 weeks)
□ User acceptance testing (1 week)
□ Deploy to production (phased, 10% first)
□ Monitor for issues (1 month)

Expected benefits:
- 5 ML jobs per day → 6.3 per day (26% throughput increase)
- Cost per job: $50 → $40 (20% savings)
- Monthly value: $25K in efficiency gains
```

### Rollout Strategy

```
Phase 1: Code changes (app team)
- Deploy updated ML training code
- Batch metadata operations
- Larger batch size
- New checkpoint format
- Timeline: 1 week deployment, 2 weeks validation

Phase 2: Storage changes (storage team)
- Adjust Lustre striping for ML data directories
- Change affects new files only (safe)
- Old files keep old stripe (can migrate later)
- Timeline: 1 day deployment, ongoing monitoring

Phase 3: Production rollout
- Week 1: 25% of jobs use new code (shadow mode)
- Week 2: 50% of jobs
- Week 3: 75% of jobs
- Week 4: 100% of jobs (full rollout)
- Monitor each week for issues

Rollback plan:
- If issues detected: Revert to old code/config
- Data is safe (checkpoint backward compatible)
- Can rollback at any time
```

---

## Results and Impact

### Metrics After 1 Month in Production

```
Performance:
├─ Average job time: 3:08 (was 4:00)
├─ Throughput: 6.2 jobs/day (was 5.0)
├─ Consistency: ±2 min variance (was ±5 min)
└─ User satisfaction: Excellent (faster turnaround)

Operational:
├─ No regression (other workloads unaffected)
├─ No data corruption (checkpoints all valid)
├─ No downtime (phased rollout worked)
└─ Zero escalations (smooth implementation)

Business:
├─ Cost savings: $25K/month (efficiency gain)
├─ Competitive advantage: Faster training → better results
├─ Team morale: Success story for optimization
└─ Documentation: Reusable process for future optimizations
```

---

## Key Lessons

**Technical:**
1. **Measure first** - Baseline is essential for validation
2. **Focus on longest phase** - Optimize bottleneck, not marginal gains
3. **Batch operations** - Metadata reduction is high-ROI
4. **Striping matters** - Parallel I/O from multiple OSSs
5. **Test thoroughly** - Regression testing prevents surprises

**Process:**
1. **Phased rollout** - Gradual deployment reduces risk
2. **Documentation** - Future work builds on foundation
3. **Communication** - Success stories motivate teams
4. **Measurement** - You can't improve what you don't measure

**For L1 TSE:**
1. Proactive optimization is better than reactive crisis-fighting
2. Systemic improvements compound (21% from four 3-8% gains)
3. Storage tuning requires understanding application access patterns
4. Lustre stripe count/size matter for sequential vs random
5. Checkpoint operations are often overlooked optimization targets

---

## Challenge Questions

**1. Measurement Priority:** What do you measure first?
```
A) CPU utilization
B) Baseline (current job time)
C) Disk I/O latency
D) Network bandwidth

Answer: B (need baseline to measure improvement against)
```

**2. Bottleneck Focus:** Where should optimization focus?
```
A) Everywhere equally
B) The longest phase/bottleneck
C) Easy wins first
D) What users complain about

Answer: B (greatest impact from longest phase)
```

**3. Metadata Optimization:** Why batch stat calls?
```
A) Faster CPU
B) Better cache utilization (burst vs spread)
C) Less network traffic
D) Simpler code

Answer: B (concentrated operations in cache)
```

---

Powered by UQS v1.8.5-C
