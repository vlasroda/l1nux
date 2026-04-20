# Lesson 4.3: Identifying Bottlenecks and Optimization (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Fixing the wrong bottleneck wastes time and money. Finding the real problem fast is what separates good TSEs from great ones
**Skills:** Systematically identify bottlenecks, know when to optimize, understand optimization trade-offs

---

## Quick Review

### The Bottleneck Principle

```
System performance limited by bottleneck (slowest component)

Example:
CPU: 80% utilized
Memory: 30% utilized
Disk: 100% utilized (bottleneck)
Network: 50% utilized

Fixing CPU won't help. Fix disk.
```

---

## Key Concepts

### 1. Identifying the Bottleneck

**Systematic approach:**

```
Step 1: Measure all resources
├─ CPU load and %
├─ Memory used/available
├─ Disk I/O %util and latency
└─ Network bandwidth

Step 2: Find the constraint
├─ Which is at >80% capacity?
├─ Which has longest latency?
└─ Which has highest queue depth?

Step 3: Verify with application-level metrics
├─ Is cache hit rate low? (memory issue)
├─ Are I/O latencies high? (disk issue)
├─ Are packets being dropped? (network issue)
└─ Is CPU time spent waiting? (CPU issue)

Step 4: Fix the bottleneck
└─ Not something else
```

**Decision tree:**

```
Is load high and CPU% high?
├─ YES: CPU bottleneck
│       Fix: Add CPU cores, optimize code, reduce workload
│
└─ NO: Load high but CPU% low?
       Is iowait% high?
       ├─ YES: I/O bottleneck
       │       Fix: Faster disk, RAID, caching, parallelism
       │
       └─ NO: Memory issue?
               Is swap active (si/so > 0)?
               ├─ YES: Memory pressure
               │       Fix: Add RAM, reduce process memory
               │
               └─ NO: Network bottleneck?
                       Throughput low despite low disk/CPU?
                       └─ YES: Network issue
                               Fix: Faster network, protocol optimization
```

### 2. CPU Bottleneck

**Symptoms:**
```
- High load average (> number of CPUs)
- High %CPU in top (>80%)
- Low iowait% (not waiting for I/O)
- Processes waiting for CPU (long run queue)
```

**Diagnosis:**
```bash
# Confirm CPU bound
uptime                  # Load > CPUs?
top -b -n 1            # %CPU high?
vmstat 1 5             # us+sy high, wa low?
perf top               # What function uses CPU?
```

**Solutions (in order):**
```
1. Optimize code (fastest, costs time)
   - Find hot spots with perf
   - Optimize algorithm
   - Cache results

2. Parallelize (medium effort)
   - Use multiple cores/processes
   - Requires code changes

3. Add more CPUs (easiest, costs money)
   - Scale horizontally
   - Upgrade to faster CPU

4. Reduce workload (often best)
   - Do less processing
   - Offload to other system
```

### 3. I/O Bottleneck

**Symptoms:**
```
- High iowait% (>30%)
- High load but low CPU%
- Disk %util >80%
- High await time (>10ms for SSD, >100ms for HDD)
- Processes in 'D' state
```

**Diagnosis:**
```bash
# Confirm I/O bound
iostat -x 1 5          # %util and await high?
iotop                  # Which process is doing I/O?
vmstat 1 5             # bi/bo columns high?
vmstat -d              # Disk read/write rates?
```

**Solutions (in order):**
```
1. Reduce I/O (no cost, requires analysis)
   - Add caching
   - Batch operations
   - Reduce frequency

2. Parallelize I/O (medium effort)
   - Multiple disks/RAID
   - Parallel reads
   - Async I/O

3. Faster storage (costs money)
   - SSD vs HDD
   - RAID configuration
   - NVMe
   - Network storage optimization

4. Distribute load (architectural change)
   - Sharding
   - Multiple servers
   - Caching tiers
```

### 4. Memory Bottleneck

**Symptoms:**
```
- Available memory < 10% of total
- Swap active (si/so > 0)
- High page fault rate
- Processes slow down over time
- OOM killer activating
```

**Diagnosis:**
```bash
# Confirm memory pressure
free -h                # Available < 10%?
vmstat 1 10           # si/so > 0?
ps aux --sort -%mem   # Which process consumes most?
dmesg | grep OOM      # Has OOM killer activated?
```

**Solutions (in order):**
```
1. Reduce memory use (no cost)
   - Identify memory leak (valgrind)
   - Reduce data size
   - Tune cache sizes

2. Optimize memory access (medium effort)
   - Improve cache hit rate
   - Reduce memory fragmentation
   - Use NUMA-aware allocation

3. Add more RAM (costs money)
   - Increase available memory
   - Reduce swap pressure

4. Virtual memory tuning (can help or hurt)
   - Increase swap (slower but more headroom)
   - Tune swappiness (rarely helpful)
```

### 5. Network Bottleneck

**Symptoms:**
```
- Throughput lower than expected
- High latency (RTT > 100ms)
- Packet loss > 0.1%
- TCP window scaling issues
- Connection timeouts
```

**Diagnosis:**
```bash
# Confirm network bound
iperf3 -c server      # Test bandwidth
ping -c 100 server    # Check latency
netstat -s            # Check retransmits/errors
ss -i                 # TCP window size
mtr -c 50 server      # Path analysis
```

**Solutions (in order):**
```
1. Optimize network use (no cost)
   - Reduce round trips (batching)
   - Compression
   - Caching

2. Tune network stack (medium effort)
   - TCP window tuning
   - MTU optimization
   - Buffer size tuning

3. Upgrade network (costs money)
   - Faster links (1 Gbps to 10 Gbps)
   - Better routing
   - Dedicated circuits
```

### 6. Combined Bottlenecks

**Real systems often have multiple bottlenecks:**

```
Scenario: Backup is slow
- CPU: 60% (not bottleneck)
- Memory: Available 50% (OK)
- Disk source: 100% util, 50ms latency (bottleneck #1)
- Network: 500 Mbps sustained on 1 Gbps link (bottleneck #2)
- Disk destination: 80% util, 20ms latency (bottleneck #3)

Fix priority:
1. Network: Upgrade to 10 Gbps
2. Source disk: Add RAID or SSD
3. Dest disk: Improve stripe configuration

But maybe simpler fix:
- Run parallel backups to different disks
- Uses multiple disks, bypasses source disk bottleneck
```

### 7. Trade-offs in Optimization

**No perfect solution, always trade-offs:**

```
Faster CPU
→ Pro: Shorter latency
→ Con: Higher power consumption, heat

Larger cache
→ Pro: Better hit rate
→ Con: Higher latency if miss, memory overhead

More disks in RAID
→ Pro: Better throughput
→ Con: Worse rebuild time, higher failure rate

Network optimization
→ Pro: Lower latency
→ Con: Higher CPU usage (checksums, etc.)

Optimize code
→ Pro: Permanent improvement
→ Con: Time-consuming, risk of bugs
```

---

## Essential Decision Framework

### When to Optimize

```
DON'T optimize if:
- System is fast enough ("fast enough" defined by requirements)
- Bottleneck is already solved (e.g., storage faster than network)
- Cost > benefit (spend $10k to save 1 second/month?)

DO optimize if:
- System doesn't meet performance SLAs
- Bottleneck is real and fixable
- ROI justifies cost (time, money, complexity)
```

### How to Verify Fix Worked

```bash
# 1. Measure before
iostat -x 1 30 > before.txt
vmstat 1 30 > before_vm.txt

# 2. Apply fix

# 3. Measure after
iostat -x 1 30 > after.txt
vmstat 1 30 > after_vm.txt

# 4. Compare
diff before.txt after.txt
# Should see improvement in bottleneck metric

# 5. Verify no regression
# Check other metrics haven't gotten worse
```

---

## Interactive Exercise: Identify a Bottleneck

**Scenario:** Your storage system is slow. Customers report backup takes 2x longer than last month.

**Task 1: Measure current state**
```bash
# Collect baseline metrics
echo "=== Load and CPU ===" && uptime && top -b -n 1 | head -3 && \
echo "=== Memory ===" && free -h && \
echo "=== I/O ===" && iostat -x 1 3 && \
echo "=== Top CPU ===" && ps aux --sort -%cpu | head -5 && \
echo "=== Top Memory ===" && ps aux --sort -%mem | head -5
```

**Task 2: Run test workload**
```bash
# During backup/copy test, collect live data
# Terminal 1: Run copy operation
cp -r /large/dataset /backup/

# Terminal 2: Monitor
watch -n 1 'iostat -x 1 1 | tail -5 && echo && vmstat 1 1 | tail -2'

# Terminal 3: Monitor processes
watch -n 1 'top -b -n 1 | head -15'
```

**Task 3: Analyze findings**
```bash
# After test, answer:
# 1. Which metric is highest?
#    - CPU%? Load > CPUs?
#    - Disk %util > 80%?
#    - Memory swap active?
#    - Network saturated?

# 2. What's the bottleneck?
#    CPU? I/O? Memory? Network?

# 3. What's the fix?
#    Optimize code? Faster disk? Add RAM? Better network?
```

**Task 4: Create optimization plan**
```bash
# Document findings
cat > /tmp/bottleneck_report.txt << 'EOF'
System Performance Analysis

Bottleneck Identified: [CPU / I/O / Memory / Network]

Symptoms:
- [measurement 1]
- [measurement 2]

Root Cause:
[What's actually limiting performance]

Recommended Fix:
[Solution and estimated impact]

Cost-Benefit:
[Time/money to implement vs performance gain]
EOF

cat /tmp/bottleneck_report.txt
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Bottleneck Priority:** You find three issues:
   - Disk 100% utilized, 80ms latency
   - Memory 85% utilized, swap active
   - CPU 40% utilized

   Which to fix first?
   ```
   (Explain your reasoning)
   ```

2. **False Optimization:** High load average and CPU usage at 100%. You optimize the code for 20% CPU reduction. What happens?
   ```
   A) Load average drops 20%
   B) Load average stays same (still at CPU capacity)
   C) Load average increases (less efficient)
   D) Depends on what else is running
   ```

3. **Combined Bottleneck:** Network is 1 Gbps, disk can do 500 MB/s. Which limits throughput?
   ```
   A) Network (1 Gbps = 125 MB/s limit)
   B) Disk (500 MB/s limit)
   C) Network (Gbit not Byte)
   D) Depends on RAID
   ```

4. **Swap Activity:** vmstat shows si=100, so=50, and disk I/O is high. What's the issue?
   ```
   (Explain what's happening and the cascade effect)
   ```

5. **Optimization Trade-off:** Adding more RAID disks increases throughput but increases rebuild time. When is this worth it?
   ```
   (Explain the decision factors)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Backup Slow After System Upgrade"

```bash
# Compare before/after:

# Then-
# - 4 SATA disks, RAID 5
# - 1 Gbps network
# - Backup: 100 MB/s

# Now-
# - Same 4 SATA disks, RAID 5
# - 10 Gbps network
# - Backup: Still 100 MB/s (didn't improve!)

# Problem: Disk is still bottleneck
# Network was never the bottleneck
# Upgrading network wasted money

# Real fix: Upgrade to faster disks (SSD) or more disks
```

### Scenario 2: "Memory Leaking Slowly"

```bash
# Monitor over time
watch -n 300 'free -h >> /tmp/memory_log.txt && echo "---" >> /tmp/memory_log.txt'

# After 1 hour, analyze
tail -100 /tmp/memory_log.txt
# Does "available" keep decreasing?

# If yes, find culprit
ps aux --sort -%mem | head -5
# Which process is growing?

# Action: Restart process or investigate code
```

### Scenario 3: "Performance Regression"

```bash
# Before code change:
# - Throughput: 500 MB/s
# - CPU: 50%
# - I/O wait: 10%

# After code change:
# - Throughput: 300 MB/s
# - CPU: 80%
# - I/O wait: 5%

# Analysis: Code change made CPU usage higher
# CPU is now bottleneck (was I/O before)

# Decision: Revert or optimize code
# Don't upgrade disk (won't help if CPU is bottleneck)
```

---

## Key Takeaways

1. **Identify the bottleneck first** — fixing the wrong thing wastes effort
2. **Use metrics to verify** — don't guess
3. **Trade-offs exist** — no perfect solution
4. **Verify improvement** — measure before and after
5. **Cost-benefit matters** — sometimes "fast enough" is best answer
6. **Bottlenecks shift** — fixing one reveals the next

---

## How Ready Are You?

Can you explain these?
- [ ] How to systematically find a bottleneck
- [ ] The difference between CPU, I/O, and memory bottlenecks
- [ ] What trade-offs exist in each solution
- [ ] Why upgrading the wrong resource doesn't help
- [ ] How to verify an optimization actually worked

If you checked all boxes, you're ready for Week 4 exercises.

---

Powered by UQS v1.8.5-C
