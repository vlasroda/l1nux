# Week 4 Solutions: Performance Analysis

---

## Lesson 4.1: System Metrics

### Challenge Questions Answers

**1. Load Average Interpretation**
```
Answer: B) CPUs are overloaded, processes waiting

Explanation:
System has 4 CPUs
Load average: 8.0 means 8 processes waiting for CPU
On 4 CPUs, normal is load = 4.0
Load 8.0 = double the CPU capacity
Processes are waiting in queue for CPU time
```

**2. Memory Available Concern**
```
Answer: D) Only if swap is being used

Explanation:
500MB available on 16GB system = 3%
This is concerning IF swap is active
If swap = 0: Buffers/cache can be freed, not critical
If swap > 0: System is in memory pressure (bad)

Check with: free -h | grep Swap
If Swap used > 0: PROBLEM
If Swap used = 0: Just tight, not critical
```

**3. High Load, Low CPU**
```
Answer: B) Processes are blocked on I/O

Explanation:
Load 6.0 but CPU% 20% = processes aren't using CPU
Where are they? Waiting for I/O

Check with: vmstat 1 5
Look at wa (iowait%) column
If wa > 50%: Confirms I/O bottleneck
```

**4. Baseline Alert Threshold**
```
Answer: C) When load hits 3.5 (20% above)

Explanation:
Baseline 3.0 = normal
20% above baseline = 3.6 (round to 3.5)

This alerts before critical (which is 2x baseline = 6.0)
Allows time to investigate before crisis

Progressive alerts:
- 0-3.0: Normal
- 3.0-3.6: Monitor closely
- 3.6-6.0: Warning, take action
- >6.0: Critical, immediate action
```

**5. Context Switch Rate**
```
Answer: 50,000 cs/sec indicates high process contention

What it means:
- CPU switching between processes 50,000 times/second
- Each switch = overhead (save/restore state, flush cache)
- Suggests many processes competing for CPU

Is it concerning?
- Depends on CPU count and workload
- 50,000 cs/sec on 1 CPU = very bad (thrashing)
- 50,000 cs/sec on 64 CPUs = might be OK
- Exceeds ~10,000 cs/sec = investigate

Likely causes:
1. Too many processes
2. Thread pool with many workers
3. No process affinity (processes migrate between CPUs)

Fix:
- Reduce number of processes
- Use process affinity (pin to CPUs)
- Batch operations
```

---

## Lesson 4.2: Performance Tools

### Challenge Questions Answers

**1. Tool Selection for Process I/O**
```
Answer: B) iotop

Explanation:
Tools and their purposes:
- iostat: Shows disk-level I/O (not per-process)
- iotop: Shows per-process I/O (what you need)
- top: Shows CPU/memory (not I/O detail)
- vmstat: Shows overall system (not per-process)

iotop is designed specifically to show:
- Which process is reading/writing
- Bandwidth per process
- Total disk I/O

Run with: iotop (requires sudo)
```

**2. Disk Utilization Interpretation**
```
Answer: B) Disk is saturated but responsive

Explanation:
%util=100: Disk is at 100% capacity (saturated)
await=5ms: Average wait time is 5 milliseconds

5ms latency is actually good for HDD:
- HDD seek time ~10ms
- So 5ms is fast

Interpretation:
- Disk is fully utilized (working at max)
- But responsiveness is good
- Not a slow disk issue, capacity issue

If %util=100 and await=50ms:
- Disk saturated AND slow response
- Problem disk or overloaded controller
```

**3. vmstat Swap In/Out**
```
Answer: B) Memory pages being swapped (memory pressure)

Explanation:
si (swap in) = pages read from swap to RAM
so (swap out) = pages written from RAM to swap

si=100, so=50 means:
- 100 pages/sec being read from disk
- 50 pages/sec being written to disk
- System is in memory pressure
- RAM is full, using disk for virtual memory

This is BAD because:
- Disk 10,000x slower than RAM
- Causes huge performance hit
- System thrashing if high sustained

Fix: Add more RAM or reduce memory use
```

**4. Memory Usage Interpretation**
```
Answer: B) Process allocated 5GB but only 100MB in RAM

Explanation:
VIRT = Virtual memory (allocated by process)
RES = Resident memory (actually in RAM)

VIRT 5GB, RES 100MB means:
- Process allocated 5GB
- But only 100MB is in RAM
- 4.9GB is probably unallocated/mapped but not used

This is normal and OK:
- Process reserves space for potential use
- Actual usage is 100MB
- Not a memory leak or problem

RED FLAG would be:
- VIRT 5GB, RES 4.9GB: All in memory (using lots of RAM)
```

**5. Load vs CPU Mismatch Explanation**
```
Load 8 but one process at 50% indicates:

What's really happening:
1. One process using 50% of CPU
2. But load shows 8 processes waiting
3. Where are the other 7.5 process-equivalents?

Likely explanation:
- Process is doing I/O wait (blocked, not running)
- Shows in load but not in %CPU
- Other processes may be asleep/blocked

Example breakdown:
- 1 process running at 50% CPU (contributes 0.5 to load)
- 7 processes blocked on I/O (contribute 7 to load)
- Total load = 7.5

Check with:
vmstat 1 5
Look at 'b' column (blocked processes)
And 'wa' column (iowait%)
```

---

## Lesson 4.3: Bottleneck Identification

### Challenge Questions Answers

**1. Bottleneck Fix Priority**
```
Answer: Fix disk first, then memory

Reasoning:
Disk: 100% utilized, 80ms latency (BOTTLENECK #1)
→ Causing immediate performance impact
→ Everything is waiting on disk

Memory: 85% utilized, swap active (BOTTLENECK #2)
→ Causing paging to disk
→ Makes disk problem worse

CPU: 40% (not a bottleneck)
→ Has capacity

Fix order:
1. Fix memory first actually! (counterintuitive)
   - Reduces paging to disk
   - This reduces disk load
   - Improves disk performance

OR fix disk first if memory is just tight
→ Depends on root cause

Moral: In cascading bottlenecks, fix upstream first
```

**2. Code Optimization Impact**
```
Answer: B) Load average stays same

Explanation:
Scenario:
- Before: CPU 100%, load = 4 (on 4-CPU system)
- Optimize code 20%
- After: CPU reduced to 80%, load still 4

Why?
- Load average = processes waiting for CPU
- If you reduce CPU use by 20%, you use 80% capacity
- Still waiting processes fill the remaining 20%
- Load stays same

What WOULD improve:
- Throughput (more work done per second)
- Response time (less waiting)
- Power consumption (lower CPU%)

But load stays near CPU capacity until workload changes
```

**3. Bandwidth Bottleneck**
```
Answer: A) Network (1 Gbps = 125 MB/s limit)

Explanation:
Network: 1 Gbps = 1 billion bits/second
  = 1,000,000,000 bits / 8 = 125 MB/sec

Disk: 500 MB/s

Effective throughput: min(125, 500) = 125 MB/s
Limited by network

This is why network upgrades matter:
- 10 Gbps = 1,250 MB/s
- Now disk at 500 MB/s is bottleneck
- Network no longer limits
```

**4. Swap Activity Cascade**
```
Answer: Memory pressure causing I/O explosion

Explanation:
si=100 (pages/sec from disk to RAM)
so=50 (pages/sec from RAM to disk)
High disk I/O in iostat

This creates cascading problems:

Step 1: RAM full (85% utilized)
Step 2: System starts paging to disk (si/so > 0)
Step 3: Disk gets hammered with page I/O
Step 4: Disk I/O causes latency/congestion
Step 5: More processes block on I/O (load increases)
Step 6: Load causes more memory allocation
Step 7: Memory pressure increases → more paging
Step 8: Vicious cycle (system thrashing)

Visible symptoms:
- High load
- High I/O wait%
- Slow everything
- But actual workload hasn't changed

Fix: Add RAM to break the cycle
```

**5. RAID Upgrade Decision**
```
Answer: Trade-off decision based on:

Worth upgrading (add more RAID disks) if:
✓ Throughput gain > rebuild time cost
✓ Service SLA requires the bandwidth
✓ Current disks are bottleneck
✓ Rebuild time acceptable (<24hrs)

NOT worth upgrading if:
✗ Network is bottleneck (disk upgrade won't help)
✗ Current throughput is sufficient
✗ Rebuild time would violate SLA (too long)
✗ Disk cost exceeds benefit

Example calculation:
Current: 4-disk RAID 5, 200 MB/s throughput, 20hr rebuild
Add 2 more disks: 400 MB/s (2x throughput), 40hr rebuild

Worth it if:
- 200 MB/s throughput insufficient
- 40hr rebuild acceptable
- Not network limited

Not worth it if:
- 200 MB/s already fast enough
- Can't tolerate 40hr rebuild
```

---

## Exercise Solutions

### Exercise 4.1: Baseline

**Expected Output:**
```bash
$ nproc
4
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi       6.5Gi       2.0Gi      100Mi       6.5Gi       8.2Gi

$ uptime
 14:30:25 up 5 days,  3:15,  2 users,  load average: 0.45, 0.52, 0.48

Baseline document should record:
CPUs: 4
Memory: 15GB
Normal load: 0.45-0.52
Available memory: 8.2GB
Safe load: < 4.0
Warning at: > 8.0
Alert when: Load > 0.52 × 1.2 = 0.62 (20% above)
```

---

### Exercise 4.2: Tools Practice

**top Output Example:**
```
top - 14:30:25 up 5 days,  3:15,  2 users,  load average: 0.45, 0.52, 0.48
Tasks: 245 total,   2 running, 243 sleeping
%Cpu(s): 15.0 us,  5.0 sy,  0.0 ni, 78.0 id,  2.0 wa,  0.0 hi,  0.0 si
MiB Mem : 15340.3 total,  2000.5 free,  6500.2 used,  6839.6 buff/cache
MiB Swap:  4096.0 total,  4096.0 free,     0.0 used.  8200.0 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 user      20   0  500000 100000  50000 R  25.5  0.6   1:23.45 myapp
 5678 root      20   0  200000  80000  30000 S  10.2  0.5   0:45.12 sshd
 9012 user      20   0  100000  50000  20000 S   5.1  0.3   0:23.01 bash

Analysis:
- Load 0.45 (safe, < 4 CPUs)
- CPU: 15% user, 5% system, 78% idle (healthy)
- Memory: 8.2GB available (good)
- Top process (myapp) uses 25.5% CPU, 0.6% memory (reasonable)
- No swap usage
- No concerning metrics
```

**iostat Output Example:**
```
$ iostat -x 1 3
Device     r/s    w/s    rMB/s wMB/s avgrq-sz avgqu-sz await svctm %util
sda      150     50     75     10     400       2       20    2    100
sdb       50     20     25      5     400       0.5     10    1     50

Analysis:
- sda: 100% utilized, 20ms latency (busy disk)
- sdb: 50% utilized, 10ms latency (capacity available)

If backup slow:
- sda is bottleneck
- Could move some workload to sdb
- Or upgrade sda to faster disk
- Or add more disks in RAID
```

---

### Exercise 4.3: Bottleneck Identification

**CPU Bottleneck Scenario:**
```
Symptoms observed:
- uptime: load average: 12.5, 11.0, 9.8 (> 4 CPUs)
- top: %CPU total 95%, %mem 30% (CPU is limiting)
- iostat: %util 20%, await 2ms (disk is idle)
- vmstat: us 80, sy 15, wa 5 (CPU time, not I/O)

Diagnosis: CPU BOTTLENECK
Fix: Reduce load, optimize code, or add CPUs
```

**I/O Bottleneck Scenario:**
```
Symptoms observed:
- uptime: load average: 8.5, 7.2, 6.1
- top: %CPU 30%, %mem 40%
- iostat: %util 95%, await 50ms (disk saturated)
- vmstat: us 10, sy 15, wa 60, id 15 (60% iowait!)

Diagnosis: I/O BOTTLENECK
Fix: Faster disk, RAID, or reduce I/O frequency
```

**Memory Bottleneck Scenario:**
```
Symptoms observed:
- free: available only 500MB (out of 16GB)
- vmstat: si=500, so=200 (heavy paging)
- iostat: increased I/O (disk handling page faults)
- Load increases over time as memory pressure builds

Diagnosis: MEMORY BOTTLENECK
Fix: Add RAM or reduce memory use
```

---

## Real-World Case Studies

### Case 1: Backup Performance Regression

**Before:**
```
- Throughput: 200 MB/s
- Duration: 10 hours for 7.2 TB
- CPU: 30%
- Disk: 70% utilized
- Memory: 50% utilized
```

**After (slow):**
```
- Throughput: 50 MB/s
- Duration: 40 hours (4x slower!)
- CPU: 10%
- Disk: 100% utilized, 80ms latency
- Memory: 60% utilized
```

**Analysis:**
- CPU dropped 30% → 10% (not CPU bottleneck)
- Disk jumped 70% → 100% with 4x latency (DISK BOTTLENECK)
- Likely cause: New workload added to storage (contention)
- Or: New disk is slower than expected
- Or: RAID configuration changed

**Investigation:**
```bash
# Before vs after metrics
iostat -x 1 10 > before.txt
[new workload added]
iostat -x 1 10 > after.txt

# Compare
- Before: sda %util 70, await 5ms
- After: sda %util 100, await 80ms

# Find culprit
iotop  # What's causing I/O?
# Might see competing backup, new service, etc.
```

**Fix:**
- If competing workload: Reschedule backups
- If new disk slower: Upgrade disk or add to RAID
- If RAID misconfigured: Optimize stripe size/alignment

---

## Performance Optimization Checklist

**Before optimizing:**
- [ ] Establish baseline (measure current state)
- [ ] Identify actual bottleneck (not guessing)
- [ ] Measure specific metric that's limiting
- [ ] Estimate improvement from fix
- [ ] Calculate ROI (is it worth doing?)

**During optimization:**
- [ ] Change one thing at a time
- [ ] Measure impact immediately
- [ ] Document what changed
- [ ] Check for regressions in other metrics

**After optimization:**
- [ ] Verify improvement (measured, not felt)
- [ ] Update baseline with new metrics
- [ ] Create alerts for when regresses
- [ ] Share findings with team

---

Powered by UQS v1.8.5-C
