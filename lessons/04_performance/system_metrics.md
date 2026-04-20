# Lesson 4.1: System Metrics and Baselines (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Understanding CPU, memory, and I/O metrics is essential for diagnosing why storage is slow
**Skills:** Understand load average, memory states, I/O wait, context switching, and baseline metrics

---

## Quick Review

### The Four System Resources

```
1. CPU - Compute power
2. Memory - RAM and caching
3. I/O (Disk) - Read/write performance
4. Network - Data transport

When storage is slow, problem is usually:
- CPU: Process can't process fast enough
- Memory: Not enough cache, swapping occurring
- I/O: Disk saturated or slow seeks
- Network: Bandwidth or latency limited
```

---

## Key Concepts

### 1. Load Average (CPU)

**What is it?**
```
Load average = number of processes ready to run
Sampled over 1, 5, and 15 minutes

Example: load average: 2.5, 2.0, 1.8
  - Last 1 min: 2.5 processes waiting for CPU
  - Last 5 min: 2.0 processes (average)
  - Last 15 min: 1.8 processes (trend down)
```

**How to interpret:**
```
On single-CPU system:
  - Load 1.0 = CPU fully utilized
  - Load 2.0 = CPU overloaded (processes waiting)
  - Load 0.5 = CPU has spare capacity

On 4-CPU system:
  - Load 4.0 = All CPUs fully utilized
  - Load 8.0 = CPUs overloaded
  - Load 2.0 = 50% utilized

Rule of thumb:
  Safe: Load < (number of CPUs)
  Warning: Load = number of CPUs to 2x
  Problem: Load > 2x number of CPUs
```

**What NOT to confuse with:**
```
Load average ≠ CPU %
Load average = processes waiting for CPU
CPU % = percentage of time CPU is busy

High load + low CPU% = I/O wait (not CPU issue)
High load + high CPU% = CPU issue
```

### 2. Memory States

**Different types of memory:**
```
Total = Mem + Buffers + Cached + Swap (total addressable)
Used = Actually in use by processes
Free = Truly unused
Available = Can be freed quickly (cached + buffers)
Cached = Filesystem cache (reclaimed if needed)
Swap = Disk-based virtual memory (SLOW)
```

**Example output:**
```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi       8.2Gi       1.5Gi      234Mi       5.2Gi       6.5Gi
Swap:          4.0Gi       0Bi       4.0Gi
```

**Interpretation:**
```
Used: 8.2Gi (what's actually in use)
Free: 1.5Gi (truly unused)
Available: 6.5Gi (can be freed, includes cache)

Health check:
- If Available is high: OK
- If Available drops below 10%: Getting tight
- If swap is being used: Memory pressure (bad)
```

**Memory pressure symptoms:**
```
High swap usage = major performance impact
Process stuck in D state = waiting for I/O (memory paging)
Slow response time = swapping is happening

Example: 15GB RAM, using 14GB, swap active
- 1GB available space
- System starts paging to disk
- Disk I/O 1000x slower than RAM
- Everything slows down
```

### 3. I/O Wait (iowait)

**What is it?**
```
iowait = % of CPU time spent waiting for I/O
         (CPU idle, but process is blocked on disk/network)

CPU states sum to 100%:
- user: Time running user code
- system: Time in kernel calls
- iowait: Time waiting for I/O
- idle: Time doing nothing
- steal: Time stolen by hypervisor (VMs)
```

**High iowait indicators:**
```
Symptoms:
- System appears busy (high load) but CPU% is low
- Disk light constantly on
- Processes stuck in 'D' state (uninterruptible)

Causes:
- Disk is slow or saturated
- Network is slow (NFS, iSCSI)
- Processes doing heavy I/O

Example:
$ top
load average: 8.0, 7.5, 7.2
%Cpu(s): 20.0 us, 15.0 sy, 50.0 wa, 15.0 id
         ↑ iowait is 50% = disk bottleneck
```

### 4. Context Switching

**What is it?**
```
Context switch = CPU switching from one process to another
Overhead: Save/restore CPU state, flush cache, invalidate TLB

Excessive switching symptoms:
- High load but low single-process CPU%
- Many processes competing for CPU
- Latency increases

Example:
- 1 process running: 0 context switches/sec
- 100 processes: 10,000+ context switches/sec
- System thrashing: Context switch overhead > useful work
```

**How to check:**
```bash
# Context switches
vmstat 1 5 | grep -E "cs|in"
# cs = context switches
# in = interrupts
# High cs: Many processes competing

# Per-process
strace -c command    # Count syscalls and context switches
```

### 5. Cache Effectiveness

**Buffer cache:**
```
Caches filesystem data
More cache = fewer disk reads
Full cache = processes don't have memory

Cache hit rate = (Total reads - Disk reads) / Total reads
80%+ = Good
<50% = Poor (disk is bottleneck)
```

**How to check:**
```bash
# Cache stats
vmstat 1 | head -10
# bi/bo = blocks in/out (disk activity)
# If high: Cache not helping

# Monitor buffer usage
free -h
# buff/cache should be high
```

### 6. Baseline Metrics for Storage Systems

**What is "normal"?**
```
Baseline = typical performance when system is healthy

Storage system baseline (varies by hardware):
- Load: < number of CPUs
- Memory available: > 10%
- Disk utilization: < 70%
- iowait: < 10% (varies by workload)
- Context switches: < 10,000/sec (depends on load)
```

**Why baselines matter:**
```
Without baseline:
- "System is slow" = too vague
- Hard to compare before/after changes

With baseline:
- "Normally 100 MB/s, now 20 MB/s" = clear problem
- Can identify anomalies
- Can predict when to upgrade

Establishes baseline:
1. Run system under normal load
2. Record key metrics (iostat, vmstat, top)
3. Review periodically
4. Alert when 20%+ above baseline
```

---

## Essential Commands

### Quick System Status

```bash
# One command overview
uptime                          # Load average
free -h                         # Memory usage
df -h                           # Disk usage
top -b -n 1 | head -15         # CPU, memory, processes

# Or one comprehensive check:
echo "=== Load ===" && uptime && \
echo "=== Memory ===" && free -h && \
echo "=== CPU ===" && top -b -n 1 | head -3
```

### Detailed Metrics

```bash
# CPU and load
top                            # Interactive, real-time
uptime                         # Load average only
ps aux | wc -l                 # Process count

# Memory details
free -h                        # Quick summary
vmstat 1 5                      # Virtual memory stats over time
cat /proc/meminfo             # Detailed memory breakdown

# I/O details
iostat -x 1 5                  # I/O stats with extended info
vmstat 1 5                      # Disk I/O in iostat-like format
iotop                          # Top I/O processes

# Process-level metrics
ps aux --sort -%cpu            # Top CPU users
ps aux --sort -%mem            # Top memory users
ps aux | grep <process>        # Specific process
```

### Monitor Over Time

```bash
# Watch metrics continuously
watch -n 1 'free -h && echo && uptime'

# Record to file for analysis
top -b -d 1 -n 300 > top_log.txt    # 300 seconds
iostat -x 1 300 > iostat_log.txt    # 300 seconds

# Later analysis
grep "Cpu" top_log.txt | awk '{print $NF}'  # Get iowait%
```

---

## Interactive Exercise: Establish Baseline

**Task 1: Current system status**
```bash
uptime
free -h
df -h
top -b -n 1 | head -15

# Record all values
```

**Task 2: Identify resource constraints**
```bash
# Which resource is most constrained?
echo "Load (target < CPUs):"
uptime | awk -F'load average:' '{print $2}'

echo ""
echo "Memory available (target > 10%):"
free -h | awk '/^Mem/ {printf "%.1f%%\n", $7/$2 * 100}'

echo ""
echo "Disk usage (target < 70%):"
df -h / | tail -1 | awk '{print $5}'

echo ""
echo "CPU type:"
nproc
echo "CPUs available"
```

**Task 3: Simulate load and observe**
```bash
# Create CPU load
stress-ng --cpu 2 --timeout 10s 2>/dev/null &

# In another terminal, watch metrics
watch -n 1 'uptime && echo && top -b -n 1 | head -15'

# Observe:
# - Load average increases
# - top shows processes taking CPU%
# - iowait stays low (CPU-bound workload)
```

**Task 4: Document baseline**
```bash
# Create baseline file
cat > /tmp/system_baseline.txt << 'EOF'
System Baseline - $(date)

CPUs: $(nproc)
Memory: $(free -h | grep Mem | awk '{print $2}')
Load (normal): $(uptime | awk -F'load average:' '{print $2}')
Disk: $(df -h / | tail -1 | awk '{print $2, "(" $5 ")")')

Max safe load: $(nproc)
Warning at: $(( $(nproc) * 2 ))
EOF

cat /tmp/system_baseline.txt
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Load Average:** System has 4 CPUs and load average of 8.0. What does this mean?
   ```
   A) System is fine, 8.0 is normal
   B) CPUs are overloaded, processes waiting
   C) Half the CPUs are being used
   D) Disk is the bottleneck
   ```

2. **Memory:** You see `available = 500MB` on a 16GB system. Should you be concerned?
   ```
   A) No, plenty of memory available
   B) Yes, swap will be used
   C) No, buffers/cache can be freed
   D) Only if swap is being used
   ```

3. **iowait:** Load is 6.0 but CPU% is only 20%. What's happening?
   ```
   A) CPUs are mostly idle
   B) Processes are blocked on I/O
   C) System is fine
   D) Memory is overcommitted
   ```

4. **Baseline:** You establish baseline at 3.0 load average. When should you alert?
   ```
   A) When load hits 4.0
   B) When load hits 6.0 (2x baseline)
   C) When load hits 3.5 (20% above)
   D) Only when load > number of CPUs
   ```

5. **Context Switching:** vmstat shows 50,000 cs/sec (context switches). What does this indicate?
   ```
   (Explain what's happening and if it's concerning)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "System is slow but I/O looks OK"

```bash
# Check each metric:
uptime                     # High load?
free -h                    # Memory pressure?
top -b -n 1 | head -12    # CPU usage?
iostat -x 1 3             # Disk utilization?

If:
- High load
- Low CPU%
- Normal I/O
- High iowait%
→ Disk I/O is bottleneck (but not showing in iostat)

Possible causes:
- NFS latency (shows as iowait, not local disk)
- Network congestion
- Remote storage slow
```

### Scenario 2: "Memory keeps growing"

```bash
# Check memory trend
free -h
# Is "used" constantly increasing?

# Find culprit
ps aux --sort -%mem | head -10
# Which process is using memory?

# Check if it's cache or actual use
# If buffers/cache high: Normal, can be freed
# If actual process memory: Memory leak

# Monitor swap
free -h
# If swap > 0: Memory pressure (bad)

# Action:
# - Kill the process consuming memory
# - Or increase RAM
# - Or reduce cache (rare)
```

### Scenario 3: "CPU maxed but storage isn't slow"

```bash
# Check what's using CPU
top
# Is it a legitimate process?
# Or runaway job?

# Check I/O
iostat -x 1 3
# If I/O low: CPU just busy with computation

# This might be OK if:
- Process is compressing data
- Process is computing (not I/O bound)
- Encryption being applied

# Only a problem if:
- Should be I/O bound but isn't
- Process is stuck in loop
```

---

## Key Takeaways

1. **Load average = processes ready for CPU** — not CPU%
2. **iowait = CPU waiting for I/O** — high iowait means I/O is bottleneck
3. **Available memory matters more than free** — cache can be freed
4. **Baseline is essential** — anomalies are visible only compared to baseline
5. **Four resources: CPU, memory, I/O, network** — identify which is bottleneck
6. **Context switching overhead matters** — excessive switching = thrashing
7. **Swap = danger zone** — indicates memory pressure

---

## How Ready Are You?

Can you explain these?
- [ ] What load average means and when it's too high
- [ ] The difference between "free" and "available" memory
- [ ] What iowait is and what causes high iowait
- [ ] Why context switching overhead matters
- [ ] How to establish a performance baseline

If you checked all boxes, you're ready for Lesson 4.2.

---

Powered by UQS v1.8.5-C
