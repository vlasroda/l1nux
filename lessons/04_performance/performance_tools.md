# Lesson 4.2: Performance Tools (Refresher)

**Time:** 25-30 minutes
**Why it matters:** The right tool gives you the answer quickly. Knowing which tool to use for each problem saves hours of debugging
**Skills:** Master top/htop, iostat, vmstat, perf, and when to use each

---

## Quick Review

### Tool Selection Matrix

```
Question                           Tool
─────────────────────────────────────────────────────────
What's using CPU?                  top, htop, perf
What's using memory?               free, top, ps
What's doing I/O?                  iostat, iotop, vmstat
Is CPU bottleneck?                 top (high CPU%, high load)
Is I/O bottleneck?                 iostat (high util%, wait)
Is memory bottleneck?              vmstat (swap in/out)
Which process uses bandwidth?      iftop, nethogs
What's the system doing?           vmstat (overall view)
Detailed system profile?           perf (advanced)
Real-time monitoring?              top, iotop, iftop
Historical analysis?               iostat/vmstat logs
```

---

## Key Concepts

### 1. TOP (CPU and Memory View)

**What it shows:**
```
- Process list sorted by CPU or memory
- System load and uptime
- Total CPU%, memory usage
- Per-process: PID, USER, CPU%, MEM%, VIRT, RES, COMMAND

Best for: Quick "what's using resources?"
```

**Key columns:**
```
PID    = Process ID
USER   = Process owner
PR     = Priority (higher # = lower priority)
VIRT   = Virtual memory (allocated, includes swapped)
RES    = Resident memory (actually in RAM)
%CPU   = Percent of total CPU
%MEM   = Percent of total memory
TIME+  = Total CPU time used
COMMAND = Process command
```

**Common uses:**
```bash
top                         # Interactive
top -b -n 1                # Batch mode, one snapshot
top -b -n 1 -p 1234       # Specific process
top -o %CPU               # Sort by CPU% (default)
top -o %MEM               # Sort by memory%
top -u username           # Specific user's processes
```

**Interpretation:**
```
High %CPU, high load, low iowait: CPU bottleneck
High load, low %CPU, high iowait: I/O bottleneck
RES much less than VIRT: Process has swapped memory
Many processes with low CPU%: Context switching overhead
```

### 2. IOSTAT (I/O Performance)

**What it shows:**
```
- Throughput: r/s (reads/sec), w/s (writes/sec), MB/s
- Latency: await (average wait time)
- Utilization: %util (percent disk busy)
- Service time: svctm (time per operation)

Best for: "Is disk the bottleneck?"
```

**Key metrics:**
```
r/s, w/s      = Operations per second
rMB/s, wMB/s  = Megabytes per second
%util         = % of time disk is busy (0-100)
await         = Average wait time in milliseconds
svctm         = Average service time per operation

High %util (>80%) + high await = Disk bottleneck
Low %util + high await = Not all capacity being used
```

**Common uses:**
```bash
iostat                      # Cumulative since boot
iostat -x 1 5              # Extended, every 1s, 5 samples
iostat -x 1 5 /dev/sda    # Specific disk
iostat -xm 1 5             # Extended, in MB/s
iostat -t                  # Include timestamp
```

**Interpretation:**
```bash
$ iostat -x 1 3
avg-cpu:  %user %nice %system %iowait %idle
           20    0     10      50      20
                                ↑ High iowait = I/O intensive

Device: rrqm/s wrqm/s r/s w/s rMB/s wMB/s avgrq-sz avgqu-sz await svctm %util
sda       0     0    150  50  75    10    400      5       50    2    100
                                                                   ↑ Disk fully used
```

### 3. VMSTAT (Virtual Memory / Overall System)

**What it shows:**
```
- Memory: free, used, buffer, cache
- Swap: in/out (page faults)
- I/O: blocks in/out (disk activity)
- CPU: user, system, idle, iowait
- Processes: running, blocked

Best for: "Overall system health snapshot"
```

**Key columns:**
```
r, b      = Processes running, blocked
swpd      = Virtual memory used
free      = Free memory
buff      = Buffers
cache     = Page cache
si, so    = Swap in/out (IMPORTANT: >0 means memory pressure)
bi, bo    = Blocks in/out (disk I/O)
us, sy, id, wa = CPU percentages
```

**Common uses:**
```bash
vmstat                      # One-time snapshot
vmstat 1 5                  # Every 1 second, 5 samples
vmstat -s                   # Detailed memory breakdown
vmstat -d                   # Disk I/O summary
```

**Interpretation:**
```bash
$ vmstat 1 5
r b   swpd   free  buff  cache   si   so    bi    bo in  cs us sy id wa
2 0      0  5000  1000  8000     0    0    100   200 200 500 20 10 50 20
            ↑ >0 = swap active (bad)
                      ↑ >0 = paging to disk (memory pressure)
                                          ↑ High context switches
                                                      ↑ High iowait
```

### 4. IOTOP (I/O by Process)

**What it shows:**
```
- Which process is doing I/O
- Threads reading/writing
- Total bandwidth per process
- Disk reads and writes

Best for: "Which process is killing I/O?"
```

**Similar to top, but for I/O:**
```
TID     = Thread ID
PRIO    = Process priority
USER    = Process owner
DISK READ = Read bandwidth
DISK WRITE = Write bandwidth
SWAPIN  = Memory page-ins
IO> = Percentage of CPU time used by I/O (I/O-bound indicator)
COMMAND = Process command
```

**Common uses:**
```bash
iotop                       # Interactive, requires sudo
iotop -b -n 1              # Batch mode, one snapshot
iotop -o                    # Only show I/O-active processes
iotop -d 1                  # Update every 1 second
```

### 5. PERF (Advanced CPU Profiling)

**What it shows:**
```
- Which functions consume CPU
- Cache misses
- Branch mispredictions
- Memory bandwidth issues

Best for: "Where exactly is CPU time going?"
```

**Common uses:**
```bash
perf record command         # Record CPU profile
perf report                 # Show results
perf top                    # Real-time CPU usage
perf stat command           # Count events

perf record -g command      # With call graph
perf report --stdio        # Text output
```

**Example output:**
```
   10.23%  my_app  my_app [.] expensive_function
    8.50%  my_app  libc.so [.] malloc
    7.20%  my_app  my_app [.] process_data
    6.15%  my_app  kernel [.] copy_to_user
```

### 6. Memory Tools

**free -h**
```
Shows: Total, used, free, shared, buffers, cached, available
Simple overview of memory state
```

**ps aux --sort**
```
Shows: All processes sorted by CPU or memory
Good for finding outliers
```

**valgrind (memory debugging)**
```
Detects memory leaks in applications
Advanced tool, slower execution
```

---

## Essential Commands

### One-Liner System Check

```bash
echo "=== CPU ===" && \
uptime && \
echo "=== Memory ===" && \
free -h && \
echo "=== Top CPU ===" && \
top -b -n 1 | head -15 && \
echo "=== I/O ===" && \
iostat -x 1 1 | tail -5
```

### Performance Monitoring Script

```bash
#!/bin/bash

echo "System Performance Report - $(date)"

echo ""
echo "=== CPU & Load ==="
uptime
nproc
echo "CPUs"

echo ""
echo "=== Memory ==="
free -h

echo ""
echo "=== I/O (last 5 seconds) ==="
iostat -x 1 5 | tail -10

echo ""
echo "=== Top CPU Processes ==="
ps aux --sort -%cpu | head -6

echo ""
echo "=== Top Memory Processes ==="
ps aux --sort -%mem | head -6

echo ""
echo "=== I/O by Process ==="
iotop -b -n 1 2>/dev/null | head -10 || echo "iotop not available"
```

### Continuous Monitoring

```bash
# Watch CPU in real-time
watch -n 1 'top -b -n 1 | head -20'

# Watch I/O
watch -n 1 'iostat -x 1 1'

# Watch memory
watch -n 1 'free -h'

# Watch all three
watch -n 1 'echo "=== CPU ===" && uptime && echo && echo "=== Memory ===" && free -h && echo && echo "=== I/O ===" && iostat -x 1 1 | tail -3'
```

---

## Interactive Exercise: Use Each Tool

**Task 1: top - Find CPU hog**
```bash
# Run top
top

# Press 'P' to sort by CPU
# Record: What process uses most CPU?

# Press 'q' to quit
```

**Task 2: iostat - Check disk I/O**
```bash
# Five samples, 1 second apart
iostat -x 1 5

# Record:
# - Which disk?
# - What's the %util?
# - What's the await time?
```

**Task 3: vmstat - Overall health**
```bash
# Five samples
vmstat 1 5

# Record:
# - Is swap active (si/so columns)?
# - What's the iowait%?
# - Context switches high?
```

**Task 4: free - Memory check**
```bash
free -h

# Record:
# - Total memory
# - Available memory (% of total)
# - Any swap usage?
```

**Task 5: ps - Process list**
```bash
# Top 10 by CPU
ps aux --sort -%cpu | head -11

# Top 10 by memory
ps aux --sort -%mem | head -11

# Record what's running
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Tool Selection:** You want to know which process is reading from disk. Which tool?
   ```
   A) iostat
   B) iotop
   C) top
   D) vmstat
   ```

2. **Interpretation:** iostat shows %util=100, await=5ms. What's happening?
   ```
   A) Disk is fine, low latency
   B) Disk is saturated but responsive
   C) Disk is slow
   D) I/O is not the bottleneck
   ```

3. **vmstat:** You see si=100, so=50. What does this mean?
   ```
   A) System is healthy
   B) Memory pages being swapped (memory pressure)
   C) Normal I/O activity
   D) Disk is slow
   ```

4. **top:** Process shows VIRT=5GB, RES=100MB. What does this indicate?
   ```
   A) Process is using 5GB of RAM
   B) Process allocated 5GB but only 100MB in RAM
   C) Process is memory leaking
   D) Cannot determine from this info
   ```

5. **Performance Analysis:** Load is 8, but `top` shows only one process at 50% CPU. Why?
   ```
   (Explain what's likely happening)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Server is slow"

```bash
# Use this sequence:

# 1. Check CPU and load
top -b -n 1 | head -12

# 2. Check memory
free -h

# 3. Check I/O
iostat -x 1 3

# 4. Correlate findings:
# If high load + high CPU% + low iowait: CPU bottleneck
# If high load + low CPU% + high iowait: I/O bottleneck
# If memory available < 10%: Memory pressure
```

### Scenario 2: "Backup is slow"

```bash
# During backup, monitor:
iotop -d 1
# Is backup process getting I/O?

vmstat 1 5
# Is I/O at capacity (bi/bo columns)?

iostat -x 1
# What's the disk throughput (rMB/s, wMB/s)?

# Compare with:
# Expected: 100 MB/s
# Actual: 20 MB/s
# = I/O issue (network, disk, or CPU bottleneck)
```

### Scenario 3: "Memory keeps growing"

```bash
# Monitor memory trend
watch -n 5 'free -h && ps aux --sort -%mem | head -5'

# Wait 5-10 minutes, observe:
# - Does 'available' keep shrinking?
# - Which process is growing?

# Actions:
# 1. Identify the process
# 2. Check if it's normal (caching = OK)
# 3. If memory leak: Restart process or increase RAM
```

---

## Key Takeaways

1. **top = CPU and memory** — your first go-to tool
2. **iostat = disk I/O** — answer "is disk bottleneck?"
3. **vmstat = overall view** — memory pressure, swap usage
4. **iotop = process I/O** — which process is doing I/O?
5. **perf = CPU profiling** — where is CPU time really going?
6. **Know your baseline** — anomalies are visible compared to normal

---

## How Ready Are You?

Can you explain these?
- [ ] What iostat columns mean (%util, await, r/s, w/s)
- [ ] When to use top vs iostat vs vmstat
- [ ] What high iowait means
- [ ] What swap in/out (si/so) means
- [ ] How to find which process is using most I/O

If you checked all boxes, you're ready for Lesson 4.3.

---

Powered by UQS v1.8.5-C
