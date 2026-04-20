# Week 4 Exercises: Performance Analysis

---

## Exercise 4.1: Establish Baseline Metrics

### Problem 1: Document System Capacity

```bash
# Task 1A: Hardware specs
nproc                   # Number of CPUs
lscpu | head -10        # CPU details
free -h                 # Memory info
df -h                   # Disk info

# Record in baseline file:
# CPUs: X
# Memory: XGB
# Max safe load: X
# Warning threshold: 2X
```

### Problem 2: Measure Normal State

```bash
# Task 2A: Collect metrics during normal operation
uptime
free -h
iostat -x 1 10
vmstat 1 10

# Record: Normal values for each metric
# Load: __
# Memory available: __
# Disk %util: __
# iowait%: __
```

### Problem 3: Simulate and Measure Load

```bash
# Task 3A: Create CPU load
stress-ng --cpu $(nproc) --timeout 30s &

# In another terminal:
# Watch metrics change
watch -n 1 'uptime && echo && top -b -n 1 | head -10'

# Record:
# - How load increases
# - How CPU% changes
# - How other metrics respond
```

### Challenge: Baseline Documentation

```bash
#!/bin/bash
# Create comprehensive baseline

cat > system_baseline.txt << 'EOF'
System Baseline - $(date)

HARDWARE:
CPUs: $(nproc)
Memory: $(free -h | grep Mem | awk '{print $2}')
Disks: $(lsblk | grep disk | wc -l)

NORMAL STATE METRICS:
Load (1min): $(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}')
Memory available: $(free -h | awk '/^Mem/ {print $7}')
Disk usage: $(df -h / | tail -1 | awk '{print $5}')

ALERT THRESHOLDS:
Load warning: $(echo "$(nproc) * 1.5" | bc)
Load critical: $(echo "$(nproc) * 2" | bc)
Memory warning: 20% available
Swap usage: > 0 MB
EOF

cat system_baseline.txt
```

---

## Exercise 4.2: Use Performance Tools

### Problem 1: Master top

```bash
# Task 1A: Run top and interpret
top

# Observe and record:
# - Load average trend
# - Top 3 CPU consumers
# - Top 3 memory consumers
# - Overall CPU% and memory%

# Task 1B: Sort by different columns
# Press 'P' = sort by CPU
# Press 'M' = sort by memory
# Press 'T' = sort by time
# Press 'q' = quit
```

### Problem 2: Analyze iostat

```bash
# Task 2A: Run iostat
iostat -x 1 10

# Record for each disk:
# r/s (reads/sec): ___
# w/s (writes/sec): ___
# %util (disk busy): ___
# await (latency): ___

# Interpretation:
# If %util > 80%: Disk bottleneck
# If await > 10ms: Slow disk
```

### Problem 3: vmstat Summary

```bash
# Task 3A: Overall system view
vmstat 1 10

# Record:
# - Any swap in/out (si/so)?
# - CPU user vs system vs iowait?
# - Processes waiting (b column)?

# Analysis:
# If si/so > 0: Memory pressure
# If wa > 30%: I/O bound
# If us+sy > 80%: CPU bound
```

### Problem 4: Identify Process I/O

```bash
# Task 4A: See which process uses I/O
# Create some disk activity
dd if=/dev/zero of=/tmp/large_file bs=1M count=1000 &

# In another terminal:
iotop -b -n 5 -d 1

# Record: Which process shows highest DISK READ/WRITE?
```

### Challenge: Full System Profile

```bash
#!/bin/bash
# Comprehensive performance profile

echo "=== System Performance Profile ==="
echo "Timestamp: $(date)"

echo ""
echo "CPU and Load:"
uptime
nproc
echo "CPUs"

echo ""
echo "Memory:"
free -h

echo ""
echo "CPU Time Distribution:"
top -b -n 1 | grep "Cpu(s)"

echo ""
echo "Top CPU Processes:"
ps aux --sort -%cpu | head -6

echo ""
echo "Top Memory Processes:"
ps aux --sort -%mem | head -6

echo ""
echo "Disk I/O:"
iostat -x 1 1

echo ""
echo "I/O by Process:"
iotop -b -n 1 2>/dev/null | head -10
```

---

## Exercise 4.3: Identify Bottlenecks

### Problem 1: CPU Bottleneck Scenario

```bash
# Simulate: High CPU demand

# Terminal 1: Create load
yes > /dev/null &
yes > /dev/null &
yes > /dev/null &
PID_LIST=$(pgrep -f "yes > /dev/null")

# Terminal 2: Analyze
uptime
top -b -n 1 | head -20
iostat -x 1 1

# Questions:
# 1. Is load > number of CPUs? (YES = CPU issue)
# 2. Is %CPU high? (YES = confirms CPU)
# 3. Is iowait% high? (NO = confirms CPU, not I/O)
# 4. Is disk activity high? (NO = confirms CPU)

# Conclusion: CPU bottleneck
# Fix: Reduce load, optimize code, or add CPUs

# Cleanup:
kill $PID_LIST 2>/dev/null
```

### Problem 2: I/O Bottleneck Scenario

```bash
# Simulate: Heavy disk I/O

# Terminal 1: Create I/O load
dd if=/dev/zero of=/tmp/io_test bs=1M count=5000 2>/dev/null &

# Terminal 2: Analyze
uptime
top -b -n 1 | head -20
iostat -x 1 5

# Questions:
# 1. Is load high but CPU% low? (YES = not CPU)
# 2. Is iowait% high? (YES = I/O issue)
# 3. Is disk %util high (>80%)? (YES = confirms disk)
# 4. Is await latency high? (YES = confirms slow disk)

# Conclusion: I/O bottleneck
# Fix: Faster disk, RAID, or reduce I/O frequency

# Cleanup:
killall dd 2>/dev/null
rm /tmp/io_test 2>/dev/null
```

### Problem 3: Memory Bottleneck Scenario

```bash
# Simulate: Memory pressure
# (Be careful - don't actually run this on production)

# Create many large processes
# stress-ng --vm 2 --vm-bytes 90% --timeout 10s

# Monitor
free -h
vmstat 1 10

# Look for:
# - Available memory shrinking
# - Swap activity (si/so > 0)
# - Page faults increasing

# This causes slowdown due to paging
```

### Challenge: Real Bottleneck Analysis

```bash
# During normal/peak workload:

# 1. Measure everything
echo "=== CPU ===" && uptime && top -b -n 1 | grep Cpu
echo "=== Memory ===" && free -h
echo "=== I/O ===" && iostat -x 1 1
echo "=== Processes ===" && top -b -n 1 | head -20

# 2. Identify bottleneck
# Which metric is at highest utilization?

# 3. Verify with specific tools
# CPU high? → Use perf
# I/O high? → Use iotop
# Memory high? → Use ps aux

# 4. Create optimization plan
# What's the fix?
# What's the estimated gain?
# What's the cost?
```

---

## Self-Assessment

After Week 4, you should be able to:

- [ ] Establish performance baseline for a system
- [ ] Use top, iostat, vmstat correctly
- [ ] Identify CPU bottleneck (high load + high CPU%)
- [ ] Identify I/O bottleneck (high load + high iowait%)
- [ ] Identify memory bottleneck (swap active)
- [ ] Find which process is causing the problem
- [ ] Create optimization plan with cost-benefit analysis
- [ ] Verify improvements with measurements

---

Powered by UQS v1.8.5-C
