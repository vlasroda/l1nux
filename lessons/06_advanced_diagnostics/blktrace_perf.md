# Lesson 6.3: blktrace and Performance Profiling (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Storage I/O is often the bottleneck. Trace exactly what the block layer is doing, where time is spent, and why storage is slow
**Skills:** Capture I/O traces, identify I/O hotspots, analyze disk performance, profile CPU and kernel time

---

## Quick Review

### The Storage Path

```
Your code
    ↓
read/write syscall
    ↓ (strace shows this)
Kernel VFS layer
    ↓
Block device layer
    ↓ (blktrace shows this)
Storage device (disk)
    ↓
Physical hardware
```

**strace:** Shows filesystem layer (open, read, write)
**blktrace:** Shows what actually hits the disk
**perf:** Shows kernel time, CPU sampling, call stacks

---

## Key Concepts

### 1. blktrace - Block Layer Tracing

**What it captures:**
```
- Every I/O operation sent to disk
- Queue time (waiting in kernel queue)
- Dispatch time (sent to device)
- Complete time (device responded)
- Operation type (read, write, flush, etc.)
- Size of I/O
- Device and sector

Example output:
8,0   0   234 108.123456789  1234  Q   R 4096 + 8 [cat]
8,0   0   235 108.123457123  1234  G   R 4096 + 8 [cat]
8,0   0   236 108.123458012  1234  I   R 4096 + 8 [cat]
8,0   0   237 108.123465234  1234  D   R 4096 + 8 [cat]
8,0   0   238 108.124012345  1234  C   R 4096 + 8 [0]

Breaking down:
- 8,0: Device (major,minor)
- 0: CPU
- 234-238: Sequence number
- 108.123...: Timestamp
- 1234: Process ID
- Q/G/I/D/C: Event type
- R/W: Read or Write
- 4096 + 8: Starting block and length
```

**Event types:**
```
Q = Queued (entered I/O queue)
G = Got (assigned to driver)
I = Inserted (into actual device queue)
D = Dispatched (sent to device)
C = Complete (device finished, data ready)

Timeline for one I/O:
Q → G → I → D → C
│   │   │   │   └─ I/O done
│   │   │   └─ Sent to device
│   │   └─ In device queue
│   └─ Kernel processing
└─ Request created

Total latency = Q to C
```

**Common use cases:**

```bash
# Start tracing
sudo blktrace -d /dev/sda

# In another terminal:
# Do I/O operations

# Stop with Ctrl+C

# Analyze
sudo blkparse sda.blktrace.0
# Shows all I/O operations

# Statistics
sudo blkparse -a queue -a dispatch sda.blktrace.0
# Show queue vs dispatch times

# Find slowest I/Os
sudo blkparse sda.blktrace.0 | awk '$8 > 1000 {print}'
# Operations taking > 1000ms
```

### 2. Performance Profiling with perf

**What it shows:**
```
- CPU hotspots (where time is spent)
- Function call stacks
- Cache misses
- Branch prediction failures
- System call frequency
- Interrupt handling

Example output:
# Samples  Self  Command  Shared Object      Symbol
  45.20%  12.4%  myapp   libssl.so.1.1      [.] EVP_EncryptFinal_ex
  23.10%   8.9%  myapp   libcrypto.so.1.1   [.] AES_encrypt
  15.40%   5.3%  myapp   [kernel]           [k] schedule
   8.20%   3.2%  myapp   libc.so.6          [.] malloc
   6.10%   2.3%  myapp   [kernel]           [k] do_syscall_trace
```

**Common use cases:**

```bash
# CPU profiling (sample at 99Hz)
sudo perf record -F 99 program
sudo perf report

# System call profiling
sudo perf trace program

# Specific event (cache misses)
sudo perf record -e cache-misses program
sudo perf report

# Flamegraph generation
sudo perf record -F 99 -g program
sudo perf script > out.perf
# Use flamegraph tools to visualize
```

### 3. iostat Deep Dive

**Beyond basic metrics:**

```bash
# Extended stats
iostat -x 1 10

# Key columns for storage analysis:
r/s       = Reads per second
w/s       = Writes per second
rkB/s     = Read KB per second
wkB/s     = Write KB per second
await     = Average wait time (ms) - VERY IMPORTANT
r_await   = Read wait time
w_await   = Write wait time
%util     = % time device had I/O pending

# Interpretation:
await > 10ms:  Slow disk or queue buildup
await > 50ms:  Very slow disk or severe contention
%util > 80%:   Disk approaching saturation
%util > 95%:   Disk definitely saturated
r/s = w/s = 0: No I/O (if problem is I/O-related, check if app is stuck elsewhere)
```

### 4. vmstat for I/O

**I/O specific columns:**

```bash
vmstat 1 10

# Column meanings:
bi = Blocks in (read from disk per second)
bo = Blocks out (written to disk per second)
in = Interrupts per second
cs = Context switches per second

# Interpretation:
bi/bo both 0: No disk I/O
bi/bo high: Heavy disk activity
in high:     Interrupt handling (especially from disk)
cs very high: Context switching overhead (usually from I/O wait)

# Example output:
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  1  0      500M   10M   200M    0    0    1234  567  234  456  25 15 50  5  0
                                         ^^^   ^^^        High disk read, some wait
```

### 5. Combining Tools for Complete Picture

**Systematic I/O diagnosis:**

```bash
# Step 1: Establish baseline with iostat
iostat -x 1 10

# Step 2: Identify if it's app or hardware
# If iostat shows:
# - High %util and high await: Disk is slow
# - Low %util but app reports slow: App issue

# Step 3: Trace the I/O with blktrace
sudo blktrace -d /dev/sda &
[run your workload]
sudo blkparse sda.blktrace.0

# Step 4: Find kernel bottleneck with perf
sudo perf record -F 99 -g program
sudo perf report
# Look for hot functions in kernel or libraries

# Step 5: Check what process is doing I/O
iotop -b -n 1
# Which process is the culprit?

# Step 6: Trace that process
strace -e read,write process
# Is it doing expected I/O pattern?
```

### 6. Common I/O Patterns and Problems

**Random vs Sequential I/O:**

```bash
# Sequential (good for SSD/HDD, predictable):
# Addresses go: 0, 100, 200, 300...
# Throughput: High
# Latency: Low
# Example: Streaming video, sequential file read

# Random (bad for HDD, slow):
# Addresses jump: 5000, 234, 89000, 12...
# Throughput: Low (HDD seeks)
# Latency: High (each seek takes ~5-10ms)
# Example: Database random access, cache miss patterns

# Detect from blktrace:
blkparse output shows sector numbers
If sectors are sequential: Good pattern
If sectors jump around: Bad pattern
```

**I/O Queue Buildup:**

```bash
Symptom: Applications report slow I/O
iostat shows: High await, %util not maxed

Cause: Too many concurrent requests backing up in queue
Blocked on: Disk can't keep up with request rate

Fix:
1. Reduce request rate (application tuning)
2. Add RAID striping (distribute load)
3. Add SSD cache tier
4. Upgrade storage (more spindles, faster disk)
```

---

## Essential Commands

### blktrace

```bash
# Capture to file
sudo blktrace -d /dev/sda -o sda.blk

# Parse for humans
sudo blkparse sda.blk

# Get statistics
sudo blkparse -a queue sda.blk
sudo blkparse -a complete sda.blk

# Find specific device
lsblk
# Get device name (sda, sdb, nvme0n1, etc.)

# Multiple devices
sudo blktrace -d /dev/sda -d /dev/sdb

# Set buffer size (for high IOPS)
sudo blktrace -b 1024 -d /dev/sda
```

### perf

```bash
# Record CPU profile
sudo perf record -F 99 program

# View report
sudo perf report

# Trace system calls
sudo perf trace program

# Specific event
sudo perf stat -e cycles,instructions,cache-misses program

# Show where time is spent
sudo perf top
# Live updating hotspots

# With call stacks (flamegraph)
sudo perf record -F 99 -g program
sudo perf script
```

### iostat

```bash
# Extended stats every second, 10 times
iostat -x 1 10

# Show all devices
iostat -a 1 5

# Only disk stats (no CPU)
iostat -d 1 10

# Human-readable (kilobytes)
iostat -k 1 10

# One device only
iostat -x /dev/sda 1 10
```

---

## Interactive Exercise: Analyze I/O Performance

**Task 1: Establish baseline**
```bash
# Terminal 1: Monitor I/O
iostat -x /dev/sda 1 5

# Record:
# - Current await
# - Current %util
# - Current throughput (rkB/s, wkB/s)
# This is your baseline
```

**Task 2: Run workload and compare**
```bash
# Terminal 1: Watch iostat
iostat -x /dev/sda 1 10

# Terminal 2: Create I/O workload
dd if=/dev/zero of=/tmp/testfile bs=1M count=100

# Terminal 1: Analysis
# What changed?
# Is await higher?
# Is %util higher?
# Which one grew more?
```

**Task 3: Trace the I/O**
```bash
# Terminal 1: Start trace
sudo blktrace -d /dev/sda

# Terminal 2: Small workload
dd if=/dev/urandom of=/tmp/test2 bs=4K count=100

# Terminal 1: Parse
sudo blkparse sda.blktrace.0 | head -20

# Analyze:
# Are there Q→C patterns?
# Any large latencies?
# Sequential or random sectors?
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Event Timeline:** A blktrace shows: Q at 100ms, D at 101ms, C at 110ms. What does this mean?
   ```
   A) Disk is fast (only 10ms latency)
   B) Kernel queuing (1ms), disk I/O (9ms)
   C) Disk might be slow (9ms)
   D) All of the above
   ```

2. **iostat Interpretation:** iostat shows await=50ms, %util=30%. What's the issue?
   ```
   A) Disk is slow
   B) Kernel queue buildup
   C) Not the disk (only using 30%)
   D) Cannot determine
   ```

3. **perf Hotspot:** perf report shows 45% of time in EVP_EncryptFinal_ex. What does this mean?
   ```
   A) Encryption library is the bottleneck
   B) Something called encryption a lot
   C) CPU is the bottleneck
   D) All of the above
   ```

4. **I/O Pattern:** blktrace shows sectors jumping: 1000, 50000, 500, 100000, 200. What's the problem?
   ```
   A) Sequential is better
   B) Random I/O = slow on spinning disks
   C) Could be normal for database access
   D) All of the above
   ```

5. **Diagnosis:** Storage reports slow, but iostat shows low %util and low await. Where's the problem?
   ```
   (Explain where to look next)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Storage is suddenly slow"

```bash
# Step 1: Get baseline
iostat -x /dev/sda 1 5

# Check: Is await high? Is %util high?
# If both high: Disk can't keep up
# If await high but %util low: Queue buildup

# Step 2: Find the culprit
iotop -b -n 1 | head -20
# Which process is doing I/O?

# Step 3: Trace that process
strace -e read,write -p <pid>
# What's it reading/writing?

# Step 4: Get full picture
blktrace -d /dev/sda &
# Let it run for 10 seconds while slow
sudo blkparse sda.blktrace.0 | grep -i "complete" | awk '{print $NF}' | sort -n | tail -10
# Show 10 slowest I/Os

# Analysis: Are they all from same process? Same file? Pattern?
```

### Scenario 2: "Database queries are hanging"

```bash
# Might be I/O wait, might be lock
# Check first with strace
strace -e read,write,fsync,fdatasync -p <db_pid> 2>&1 | head -50

# If stuck on read/write: I/O is blocking
# If stuck on fsync: Waiting for disk confirm

# Get timing
iostat -x 1 5
# Is %util high? That confirms disk bottleneck

# Dig deeper with blktrace
sudo blktrace -d /dev/sda
# Run query
sudo blkparse sda.blktrace.0
# Look for high latency I/Os (awaited on disk)
```

### Scenario 3: "CPU usage high but throughput low"

```bash
# Might be lock contention, might be inefficient code
# Use perf to find the bottleneck
sudo perf record -F 99 -g program
sudo perf report
# Look for hot functions

# Check if CPU is really saturated
top -b -n 1 | head -3
# Is %user high? %sys high? %wa high?

# If %wa high: Waiting on I/O (not CPU problem)
# If %us high: Application is CPU bound
# If %sy high: Kernel is CPU bound (syscall overhead)

# If kernel is hot, use perf to see why
```

---

## Key Takeaways

1. **blktrace shows block layer** — what actually hits the disk
2. **perf shows CPU hotspots** — where time is spent in code
3. **iostat shows aggregated I/O** — overall disk performance
4. **Q→C timeline** — shows queuing vs disk latency
5. **High await, low %util** — queue buildup, not disk being slow
6. **Random I/O on HDD** — seek time dominates, very slow
7. **Combine tools** — strace + iostat + blktrace + perf = full picture

---

## How Ready Are You?

Can you explain these?
- [ ] What blktrace event types (Q, D, C) mean
- [ ] How to interpret await in iostat
- [ ] When queue buildup vs disk slowness
- [ ] How perf identifies CPU hotspots
- [ ] When to use blktrace vs iostat vs perf

If you checked all boxes, you're ready for Week 6 exercises.

---

Powered by UQS v1.8.5-C
