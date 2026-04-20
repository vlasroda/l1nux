# Week 6 Exercises: Advanced Diagnostics

---

## Exercise 6.1: System Call and Library Tracing

### Problem 1: Debug a Failing Program

```bash
# Given: A program fails silently
# Task: Use strace to find what's wrong

# Create a test scenario
cat > /tmp/test_fail.sh << 'EOF'
#!/bin/bash
# Program looking for config in wrong place
cat /etc/myapp/config.conf
EOF
chmod +x /tmp/test_fail.sh

# Task 1A: Trace the program
strace /tmp/test_fail.sh 2>&1 | grep -E "open|ENOENT"

# Record:
# - What file is it trying to open?
# - What error code?
# - Is the path correct?

# Task 1B: Use strace to find what libraries it uses
strace -e openat /tmp/test_fail.sh 2>&1 | grep -E "\.so|lib"

# Record:
# - Which libraries does it load?
```

### Problem 2: Find Performance Bottleneck

```bash
# Task 2A: Profile a command with strace -c
strace -c /bin/ls -la /etc | head -20

# Record from output:
# - Which syscall takes most time (% time column)?
# - How many times is it called (calls column)?

# Task 2B: Look deeper at the hot syscall
strace -c -e trace=file /bin/ls /etc > /dev/null 2>&1

# Focus on: stat, open, read syscalls
# Which operation is most expensive?
```

### Problem 3: Identify Permission Issues

```bash
# Given: User can't access a file
# Task: Use strace to find the exact failure point

# Create test case
sudo mkdir -p /root/restricted
sudo touch /root/restricted/secret.txt
sudo chmod 600 /root/restricted/secret.txt

# Task 3A: Trace access attempt
strace -e open,openat,stat,statx cat /root/restricted/secret.txt 2>&1 | grep -E "secret|EACCES"

# Record:
# - Which operation failed?
# - What was the error?
# - What are the actual permissions?

# Task 3B: Find the fix
ls -la /root/restricted/
chmod g+r /root/restricted/secret.txt  # Fix
strace -e open,openat cat /root/restricted/secret.txt 2>&1 | tail -5
# Did it work now?
```

### Problem 4: Memory Leak Detection with ltrace

```bash
# Task 4A: Trace memory allocations
ltrace -e malloc,free /bin/ls /etc 2>&1 | head -30

# Record:
# - Is malloc called more than free?
# - Any patterns?

# Task 4B: Find unmatched allocations
ltrace -e malloc,free program 2>&1 | grep -c malloc
ltrace -e malloc,free program 2>&1 | grep -c free

# If malloc > free: Possible memory leak
```

### Challenge: Real Crash Debug

```bash
# Create a program that crashes
cat > /tmp/crash.c << 'EOF'
#include <stdio.h>
#include <string.h>
int main() {
    char buffer[10];
    strcpy(buffer, "This is a very long string that overflows");
    return 0;
}
EOF
gcc -o /tmp/crash /tmp/crash.c

# Task: Use ltrace to see what causes the crash
ltrace /tmp/crash 2>&1 | tail -20

# Record:
# - Which function call causes crash?
# - What was the last successful call?
```

---

## Exercise 6.2: Network Packet Analysis

### Problem 1: Capture DNS Traffic

```bash
# Task 1A: Start packet capture for DNS
sudo tcpdump -i any udp port 53 -n | head -10
# (Run in one terminal)

# Task 1B: Trigger DNS query
nslookup google.com
# (Run in another terminal)

# Record from tcpdump output:
# - Source and destination IP
# - Query name
# - Response received?
```

### Problem 2: SSH Connection Handshake

```bash
# Task 2A: Capture SSH connection attempt
# Terminal 1:
sudo tcpdump -i any tcp port 22 -n | head -20

# Task 2B: Initiate SSH
# Terminal 2:
ssh -o ConnectTimeout=5 127.0.0.1
# (If SSH not running, this will timeout)

# Record from tcpdump:
# - Do you see SYN packet?
# - Do you see SYN,ACK response?
# - Is connection established (ACK back)?
# - Or does connection timeout (RST or nothing)?
```

### Problem 3: Identify Connection Refused

```bash
# Task 3A: Try to connect to closed port
sudo tcpdump -i any tcp port 9999 -n &
TCPDUMP_PID=$!

# Task 3B: Attempt connection
nc -zv 127.0.0.1 9999
# or
timeout 2 bash -c 'echo > /dev/tcp/127.0.0.1/9999' 2>&1

kill $TCPDUMP_PID

# Record:
# - Do you see SYN?
# - Do you see RST (reset) response?
# - What's the difference from "timeout" vs "refused"?
```

### Problem 4: Filter Complex Traffic

```bash
# Task 4A: Capture NFS traffic only
sudo tcpdump port 2049 -c 10

# Task 4B: Capture SSH from specific host
# First, change 192.168.1.100 to actual IP or use 127.0.0.1
sudo tcpdump src 127.0.0.1 and tcp port 22 -c 5

# Task 4C: Capture all traffic except SSH
sudo tcpdump 'not port 22' -c 10

# Task 4D: Capture TCP with specific flags
sudo tcpdump 'tcp[tcpflags] & tcp-syn != 0' -c 10
# All SYN packets

# Record:
# - How many packets matched each filter?
```

### Challenge: Diagnose Connection Problem

```bash
# Scenario: Service appears to accept connection but doesn't respond

# Task: Capture the full interaction
sudo tcpdump -i any -A host 127.0.0.1 and port 9999 > /tmp/capture.txt 2>&1 &
TCPDUMP_PID=$!

# Attempt connection (change port if needed)
(echo "HELLO"; sleep 1) | nc 127.0.0.1 9999 2>&1 &

# Let it run for 5 seconds
sleep 5
kill $TCPDUMP_PID 2>/dev/null

# Analyze:
cat /tmp/capture.txt | grep -E "SYN|ACK|PSH|FIN|RST"

# Questions:
# - What flags do you see?
# - Is there a complete 3-way handshake?
# - Is the FIN graceful or RST abrupt?
```

---

## Exercise 6.3: I/O and Performance Analysis

### Problem 1: Establish I/O Baseline

```bash
# Task 1A: Get current I/O stats
iostat -x 1 5

# Record (average from 5 samples):
# - r/s (reads/sec)
# - w/s (writes/sec)
# - await (avg wait time)
# - %util (percent utilized)

# Task 1B: Identify busiest device
iostat -d 1 3
# Which device has highest activity?
```

### Problem 2: Create I/O Workload and Measure Impact

```bash
# Task 2A: Baseline (no workload)
iostat -x /dev/sda 1 3
# Record await and %util

# Task 2B: Create heavy I/O
dd if=/dev/zero of=/tmp/testfile bs=1M count=500 &
DD_PID=$!

# Task 2C: Monitor while running
iostat -x /dev/sda 1 10

# Task 2D: Stop workload
wait $DD_PID

# Analyze:
# - How much did await increase?
# - How much did %util increase?
# - Did throughput match expected (based on device type)?
```

### Problem 3: Identify Random vs Sequential I/O

```bash
# Task 3A: Sequential read
dd if=/var/log/syslog of=/dev/null bs=1M 2>&1 | head -5

# Task 3B: Monitor I/O pattern
iostat -x /dev/sda 1 10

# Record:
# - r/s value
# - rkB/s value
# - Calculate: avg_size = rkB/s / r/s
# - If avg_size large (>100KB): Sequential
# - If avg_size small (<10KB): Random

# Task 3C: Now do random reads
# Create random access pattern
dd if=/dev/urandom bs=4K count=1000 of=/tmp/random_data
dd if=/tmp/random_data of=/dev/null bs=4K 2>&1 | head -5

# Compare I/O stats
# Sequential should have higher throughput
```

### Problem 4: Using vmstat for I/O Analysis

```bash
# Task 4A: Baseline vmstat
vmstat 1 5

# Record:
# - bi (blocks in per sec)
# - bo (blocks out per sec)
# - wa (wait on I/O %)

# Task 4B: During I/O workload
dd if=/dev/zero of=/tmp/test2 bs=4K count=10000 &
vmstat 1 10
kill %1 2>/dev/null

# Analyze:
# - Did bo (writes) spike?
# - Did wa (I/O wait %) increase?
# - Does this match iostat findings?
```

### Challenge: Complete I/O Diagnosis

```bash
# Scenario: Application reports storage is slow

# Task 1: Establish baseline
iostat -x 1 5
vmstat 1 5
# Record await, %util, bi, bo

# Task 2: Simulate slow storage
# (This is your workload that might be normal)
dd if=/dev/zero of=/tmp/slow_test bs=1M count=200 &
WORKLOAD_PID=$!

# Task 3: Diagnose
# 3A: Is disk the bottleneck?
iostat -x /dev/sda 1 10
# If await > 20ms or %util > 80: Yes

# 3B: What process is doing I/O?
iotop -b -n 1 2>/dev/null | head -10
# (Or use ps aux if iotop not available)

# 3C: Find the process PID from above
# And trace its I/O
strace -e read,write -c dd if=/dev/zero of=/tmp/test3 bs=1M count=10 2>&1

# Task 4: Conclusions
# - Is disk saturated?
# - Is I/O queue backing up?
# - Is application using right I/O pattern?
# - What's the bottleneck?
```

---

## Exercise 6.4: Multi-Tool Diagnostic Scenario

### Scenario: "Application is hung, storage might be involved"

```bash
# Given: A long-running application appears to be hung

# Step 1: Use strace to see if it's blocked on I/O
# Find PID first
ps aux | grep myapp
# Get PID (let's say 1234)

# Attach strace
sudo strace -p 1234 -e read,write,fsync 2>&1 | head -20

# Questions to answer:
# - Is it stuck on a syscall?
# - Which one?
# - Is it an I/O operation?

# Step 2: If stuck on I/O, check disk status
iostat -x /dev/sda 1 5

# Questions:
# - Is await high (>10ms)?
# - Is %util high (>80%)?
# - Is the disk actually doing I/O?

# Step 3: Find what's causing the I/O
iotop -b -n 1 2>/dev/null | head -15
# or use: lsof -p 1234 | grep /var (check what file is open)

# Step 4: Check the state of the process
cat /proc/1234/status
# Look for State: (S=sleeping, R=running, D=disk sleep, Z=zombie)

# Step 5: If in D state (disk wait), that's your answer
# If in other state but strace shows read, it's waiting

# Step 6: Find if it's queue buildup or disk slowness
# Get timeline: strace -T shows time in each syscall
sudo strace -T -p 1234 -e read,write 2>&1 | head -30
# If calls take >10ms each: Disk is slow
# If calls are quick but queued up: Queue buildup
```

---

## Exercise 6.5: Self-Assessment Challenge

```bash
# Create a diagnostic report for your system

# Part 1: System Call Analysis
echo "=== SYSTEM CALL PROFILE ===" > /tmp/diagnosis.txt
strace -c /bin/ls /etc 2>&1 | head -20 >> /tmp/diagnosis.txt

# Part 2: Network Baseline
echo "" >> /tmp/diagnosis.txt
echo "=== NETWORK BASELINE ===" >> /tmp/diagnosis.txt
sudo timeout 5 tcpdump -i any -c 20 2>&1 | tail -15 >> /tmp/diagnosis.txt

# Part 3: I/O Baseline
echo "" >> /tmp/diagnosis.txt
echo "=== I/O BASELINE ===" >> /tmp/diagnosis.txt
iostat -x 1 3 >> /tmp/diagnosis.txt

# Part 4: CPU Hotspots (if perf available)
echo "" >> /tmp/diagnosis.txt
echo "=== CPU BASELINE ===" >> /tmp/diagnosis.txt
top -b -n 1 | head -15 >> /tmp/diagnosis.txt

# Review your report
cat /tmp/diagnosis.txt

# Questions to answer in report:
# 1. What's your baseline I/O performance (await, %util)?
# 2. What syscalls does your system make most?
# 3. Are there any obvious bottlenecks in baseline?
# 4. What tool would you use first if storage was reported slow?
```

---

## Self-Assessment

After Week 6, you should be able to:

- [ ] Use strace to trace system calls and find failures
- [ ] Use ltrace to identify library function issues
- [ ] Interpret strace -c output to find hotspots
- [ ] Use tcpdump to filter and capture specific traffic
- [ ] Diagnose network problems from packet analysis
- [ ] Use iostat to identify I/O bottlenecks
- [ ] Understand blktrace event sequences (Q, D, C)
- [ ] Combine multiple tools for comprehensive diagnosis
- [ ] Differentiate between disk slowness and queue buildup
- [ ] Find the culprit process in performance issues

---

Powered by UQS v1.8.5-C
