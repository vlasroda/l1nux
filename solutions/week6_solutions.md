# Week 6 Solutions: Advanced Diagnostics

---

## Lesson 6.1: strace and ltrace

### Challenge Questions Answers

**1. Event Timeline in blktrace**
```
Answer: Q at 100ms, D at 101ms, C at 110ms
        → B) Kernel queuing (1ms), disk I/O (9ms)

Explanation:
Q → D = 1ms: Time in kernel queue (fast)
D → C = 9ms: Time at disk (actual I/O latency)
Total = 10ms latency

This is actually reasonable for a spinning disk (typically 5-20ms).

If this was consistently 9ms, the disk is working normally.
If this was consistently 100ms+, disk is slow or heavily loaded.
```

**2. Tool Choice for Slow Program**
```
Answer: D) Depends on where time is spent

Explanation:
- If 80% of time is in malloc/free (library): ltrace
- If 80% of time is in read/write syscalls: strace
- If split between both: Use both

Real answer: Run strace -c first to see which syscalls consume time.
Then look deeper with ltrace if library calls are hot.

Combination is most informative:
strace -c program > calls.txt 2>&1
ltrace -c program > libs.txt 2>&1
Compare both to get full picture
```

**3. ENOENT Error**
```
Answer: B) File doesn't exist

Explanation:
ENOENT = Error NO ENTry (Linux error codes)
open("/etc/myconfig.conf", O_RDONLY) = -1 ENOENT

This means:
- /etc/myconfig.conf does not exist at that path
- Program is looking in wrong location
- Solution: Either create the file or fix the path

Common fixes:
1. Check if file exists elsewhere: find / -name myconfig.conf
2. Create the file with correct config
3. Update program path to correct location
4. Create symlink to correct location
```

**4. Hanging on futex()**
```
Answer: B) Process is in deadlock/waiting for lock

Explanation:
futex() = fast userspace mutex - synchronization primitive

When strace shows stuck on futex():
- Process is waiting for a lock
- Another thread/process holds the lock
- Often indicates deadlock or contention

Causes:
1. Deadlock (thread A waits for B, B waits for A)
2. Lock held by crashed thread
3. Excessive contention (too many threads)
4. Lock never released due to bug

Debug approach:
strace -f shows all threads
Look for which thread holds lock and which wait
If all blocked on futex: Likely deadlock
```

**5. 80% Time in read()**
```
Answer: A) Slow disk I/O

Explanation:
strace -c shows:
% time     seconds  usecs/call     calls
 80.00    0.800000       1000      800   read

Breaking this down:
- 800 read() calls
- Each averaging 1000 microseconds (1ms)
- Total 0.8 seconds spent in read

This means:
- Application is waiting on disk 80% of the time
- Not CPU-bound (CPU would show in user/system time)
- Disk is the bottleneck

Solutions:
1. Check if disk is actually slow (iostat)
2. Check if read pattern is inefficient (random vs sequential)
3. Add SSD cache, increase RAM for caching
4. Profile the actual I/O: use blktrace to see disk performance
```

### Exercise Solutions: strace Tracing

**Problem 1: Debug Failing Program**

```bash
# Creating test that tries to open /etc/myapp/config.conf
strace /tmp/test_fail.sh 2>&1 | grep -E "open|ENOENT"

# Expected output:
open("/etc/myapp", O_RDONLY|O_DIRECTORY) = -1 ENOENT (No such file or directory)

# Analysis:
# - Program is looking for /etc/myapp/config.conf
# - Directory /etc/myapp doesn't exist
# - The file cannot be found

# Fix approach:
mkdir -p /etc/myapp
echo "config_here" > /etc/myapp/config.conf
/tmp/test_fail.sh
# Now works
```

**Problem 2: Performance Bottleneck**

```bash
strace -c /bin/ls -la /etc | head -20

# Output example:
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 35.20    0.035620        9.90      3600           stat
 25.10    0.025456        8.45      3012           read
 18.50    0.018765        6.23      3012           open
 15.20    0.015430        5.12      3012           close
  4.00    0.004050        1.35      3000           mmap
  1.00    0.001015        0.34      3000           mmap2
------ ----------- ----------- --------- --------- ----------------
100.00    0.101346                 15648        100 total

Analysis:
- 35% of time in stat() - checking file info
- 25% in read() - reading directory data
- 18% in open() - opening files

Bottleneck: stat() calls (getting file metadata)
Why: ls -la must stat every file to get permissions/ownership

This is normal, not a problem.
```

**Problem 3: Permission Issues**

```bash
strace -e open,openat,stat,statx cat /root/restricted/secret.txt 2>&1 | grep -E "secret|EACCES"

# Output example:
openat(AT_FDCWD, "/root/restricted/secret.txt", O_RDONLY) = -1 EACCES (Permission denied)

# This shows:
- Attempted to open /root/restricted/secret.txt
- Got EACCES (permission denied)
- File exists (not ENOENT) but can't read

# Root cause from permissions:
ls -la /root/restricted/secret.txt
# Output: -rw------- 1 root root

# Diagnosis:
- File owned by root
- Only owner (root) can read (rw for owner, --- for group and others)
- Current user cannot read

# Fix options:
Option 1: Change to root
sudo cat /root/restricted/secret.txt

Option 2: Make readable
sudo chmod o+r /root/restricted/secret.txt
sudo chmod g+r /root/restricted/secret.txt

Option 3: Change ownership
sudo chown $USER:$USER /root/restricted/secret.txt
```

**Problem 4: Memory Leak with ltrace**

```bash
ltrace -e malloc,free /bin/ls /etc 2>&1 | head -30

# Output example shows:
malloc(1024)                                = 0x55c1b8a01260
malloc(2048)                                = 0x55c1b8a02340
free(0x55c1b8a01260)                        = <void>
free(0x55c1b8a02340)                        = <void>
malloc(512)                                 = 0x55c1b8a03400
free(0x55c1b8a03400)                        = <void>

Analysis:
Count allocations vs deallocations:
malloc count = free count: Good (no leak)
malloc count > free count: Possible leak

For /bin/ls specifically:
- ls allocates temp buffers
- ls frees them all at exit
- Should see balanced malloc/free

If imbalanced:
1. Program has memory leak
2. Program exits without cleanup (some programs do this)
3. Large objects held to program end
```

**Challenge: Crash Debug**

```bash
# Buffer overflow program crashes

ltrace /tmp/crash 2>&1 | tail -20

# Output example:
strcpy(0x7ffd5e5e5d30, "This is a very long string...")
--- SIGSEGV (Segmentation fault) ---

# Analysis:
- strcpy() called with long string
- Overwrites buffer of size 10
- Segmentation fault occurs during/after strcpy
- Root cause: Buffer overflow

# Fix:
Use strncpy or safer alternatives:
- strncpy(buffer, string, sizeof(buffer)-1)
- snprintf(buffer, sizeof(buffer), "%s", string)

Real-world lesson:
- Always use bounds-checking functions
- strcpy, sprintf, strcat are dangerous
- Use -Wall -Wextra compiler flags to catch warnings
```

---

## Lesson 6.2: tcpdump and Packet Analysis

### Challenge Questions Answers

**1. SYN Packet Meaning**
```
Answer: C) Connection is being requested

Explanation:
[SYN] flag indicates the start of a TCP three-way handshake.

In the handshake:
1. Client → Server: SYN (request connection)
2. Server → Client: SYN,ACK (accept connection)
3. Client → Server: ACK (confirm)

If you see [SYN] alone, it means:
- Connection is being initiated
- Not yet established
- Waiting for server response

Common scenarios:
[SYN] → (nothing back): Server not responding (firewall, down, etc.)
[SYN] → [SYN,ACK]: Server is listening, connection opening
[SYN] → [RST]: Server refusing (active reject)
```

**2. Capture NFS Traffic**
```
Answer: B) port 2049

Explanation:
NFS uses port 2049 for data transfer.

Also needed:
- Port 111 for RPC portmapper
- Dynamic high ports for NFSv4

To capture all NFS:
tcpdump 'port 2049 or port 111' -i eth0

Some contexts might accept:
- port 2049 (most precise)
- (port nfs) if system has service defined
```

**3. No SYN,ACK Received**
```
Answer: B) Server not listening or firewall blocking

Explanation:
If tcpdump shows SYN but no SYN,ACK:
- Client sent the request
- Server didn't respond
- Either:
  1. Service not running on that port
  2. Firewall blocks the traffic
  3. Network unreachable
  4. Server crashed/rebooted

To diagnose which:
1. Check server is running:
   ssh server 'netstat -tln | grep 2049'
   If no listening socket, start service

2. Check firewall:
   Check server firewall: sudo ufw status
   Check intermediate firewalls: firewalls between client and server

3. Check connectivity:
   ping server (basic connectivity)
   traceroute server (path to server)
   mtr server (path with packet loss)
```

**4. Duplicate ACKs Meaning**
```
Answer: B) Packet loss / retransmission happening

Explanation:
Duplicate ACKs indicate:
- Sender received out-of-order data
- Sends ACK for last good byte received
- When this happens 3+ times = triggering fast retransmit

Example:
Sender: sends bytes 1-1000
Receiver: gets 1-500
Receiver: missing 501-1000, ACK says "got up to 500"
Sender: sends 501-1000 (but it's lost again)
Receiver: still missing, ACK says "got up to 500" (duplicate)
Receiver: another ACK "got up to 500" (3 times = retransmit)

Causes:
1. Packet loss on the network (most common)
2. Out-of-order delivery (less common)
3. Congestion on link

Impact:
- Higher latency (retransmissions)
- Lower throughput
- Affects NFS performance severely

Diagnosis:
tcpdump 'tcp.flags.ack' host storage | grep -E "ack [0-9]+ win" | sort | uniq -c | sort -rn
# If same ack number appears many times: duplicate ACKs detected
```

**5. SSH Filter Command**
```
Answer: Filter for SSH from specific IP
tcpdump 'src 192.168.1.100 and tcp port 22'

Or with -i for interface:
tcpdump -i eth0 'src 192.168.1.100 and tcp port 22'

Variations:
tcpdump -i eth0 'src 192.168.1.100 and dst port 22'  # SSH TO server
tcpdump -i eth0 'dst 192.168.1.100 and src port 22'  # SSH FROM server
tcpdump -i eth0 'host 192.168.1.100 and port 22'     # Either direction
```

### Exercise Solutions: Packet Analysis

**Problem 1: Capture DNS**

```bash
# Terminal 1:
sudo tcpdump -i any udp port 53 -n | head -20

# Expected output (while nslookup running in terminal 2):
23:45:12.123456 192.168.1.50.45678 > 8.8.8.8.53: 12345+ A? google.com. (28)
23:45:12.234567 8.8.8.8.53 > 192.168.1.50.45678: 12345 1/0/0 CNAME [5ms]

# Analysis:
Query: 192.168.1.50 asks 8.8.8.8 "What is google.com?"
Response: 8.8.8.8 replies with CNAME

If no response:
- DNS server down
- Firewall blocking port 53
- Network unreachable

Fix:
Check reachability: ping 8.8.8.8
Check firewall: sudo iptables -L | grep 53
Verify service: nc -zv 8.8.8.8 53
```

**Problem 2: SSH Handshake**

```bash
# If SSH listening on 127.0.0.1:22

tcpdump -i any tcp port 22 -n | head -30

# Expected sequence:
Client.54321 > Server.22: Flags [S]        # SYN from client
Server.22 > Client.54321: Flags [S.]       # SYN,ACK from server
Client.54321 > Server.22: Flags [.]        # ACK from client
Client.54321 > Server.22: Flags [P.]  SSH-2.0-OpenSSH_7.4
                                       # SSH banner exchange
# ... rest of SSH protocol

If SSH not running:
Client.54321 > Server.22: Flags [S]        # SYN
Server.22 > Client.54321: Flags [R.]       # RST (refused)
No further packets

If firewall blocks:
Client.54321 > Server.22: Flags [S]        # SYN
(timeout, no response)
```

**Problem 3: Connection Refused**

```bash
# Attempting to connect to closed port 9999

tcpdump -i any tcp port 9999 -n

# Output when port closed:
127.0.0.1.12345 > 127.0.0.1.9999: Flags [S]      # SYN attempt
127.0.0.1.9999 > 127.0.0.1.12345: Flags [R.]     # RST (refused)

# Interpretation:
- RST immediately after SYN = Connection Refused
- This means: Host is reachable, but service not listening

vs. Timeout (no response):
127.0.0.1.12345 > 127.0.0.1.9999: Flags [S]      # SYN attempt
127.0.0.1.12345 > 127.0.0.1.9999: Flags [S]      # Retry (no response)
127.0.0.1.12345 > 127.0.0.1.9999: Flags [S]      # Retry again
(eventually timeout)

# Difference:
RST = Active refuse (quickly)
Timeout = Passive ignore (firewall drops packets, no response)

Troubleshooting:
RST: Service not running, needs to be started
Timeout: Firewall blocking, or network unreachable
```

---

## Lesson 6.3: blktrace and Performance Profiling

### Challenge Questions Answers

**1. blktrace Event Timeline**
```
Answer: D) All of the above

Explanation:
Q at 100ms = Request queued at 100ms
D at 101ms = Sent to device at 101ms (1ms kernel overhead)
C at 110ms = Completed at 110ms (9ms device latency)

Q→D: 1ms = Kernel processing time
D→C: 9ms = Actual disk I/O time
Q→C: 10ms = Total latency

This is GOOD performance for a spinning disk.

Context:
- HDD seek time: ~5-10ms typical
- SSD latency: 0.1-1ms typical
- Network storage: 5-50ms typical

So 10ms total indicates:
- Normal HDD performance
- Or high latency network storage
- Not a problem
```

**2. iostat: High await, Low %util**
```
Answer: B) Kernel queue buildup

Explanation:
When await=50ms but %util=30%:

await = Average time I/O request waits (in queue + on device)
%util = Percent time device is active

High await + Low %util means:
- Requests are waiting in queue
- But disk isn't actively serving them
- This is queue buildup, not disk slowness

Scenario:
- 10 requests queued
- Only 3 being processed
- Each queued request adds ~5ms average wait
- But disk is only 30% used

Causes:
1. Application submits many I/O at once (burst)
2. Requests back up faster than device processes
3. Queue depth growing

Fix:
1. Reduce request rate (application tuning)
2. Process requests in smaller batches
3. Check if filesystem cache is helping
4. Consider I/O scheduling (CFQ vs deadline)

vs. High await + High %util:
- Disk is heavily used AND responses are slow
- This means disk itself is slow (upgrade hardware)
```

**3. perf Hotspot: 45% in EVP_EncryptFinal_ex**
```
Answer: D) All of the above

Explanation:
perf showing 45% in EVP_EncryptFinal_ex means:
- Function spent 45% of CPU time
- It's called frequently (not one expensive call)
- This library function is the bottleneck

What this tells us:
A) Encryption library is the bottleneck ✓
B) Something (app) called encryption a lot ✓
C) CPU is the bottleneck ✓
All are true - it's a CPU-intensive encryption operation

Diagnosis path:
1. Encrypt operations are the hotspot
2. Encryption is consuming 45% of CPU
3. Either: (a) App is doing too much encryption, OR
           (b) Wrong encryption algorithm (expensive), OR
           (c) Encryption config inefficient

Solutions:
1. Profile the caller: perf report --annotate shows where EVP called
2. Reduce encryption: Use faster algorithm (AES-NI vs software)
3. Batch operations: Encrypt in batches instead of per-item
4. Offload: Use hardware accelerators if available
```

**4. Random I/O Pattern**
```
Answer: D) All of the above

Explanation:
Sectors jumping: 1000, 50000, 500, 100000, 200
This is a random I/O pattern (blocks accessed non-sequentially)

A) Sequential is better ✓
   - Sequential avoids seeks
   - Much higher throughput
   - Lower latency

B) Random I/O = slow on spinning disks ✓
   - Each jump requires disk head seek (5-10ms)
   - 100 random seeks = 500-1000ms overhead
   - Sequential: same 100 blocks read in 10-20ms

C) Could be normal for database access ✓
   - Databases often access random rows
   - B-tree structure causes this pattern
   - DBAs accept lower performance vs. sequential

Context matters:
HDD + Random I/O = Bad performance (slow)
SSD + Random I/O = OK (no seek time, good performance)
DB workload = Expected (inherent to data model)

Diagnosis:
Pattern from blktrace = understanding workload characteristics
```

**5. Diagnosis When No I/O Shown but Slow**
```
Answer: Problem is elsewhere, not disk

Explanation:
If storage is reported slow but:
- iostat shows low %util
- iostat shows low await
- blktrace shows no activity or low latency

Then the problem is NOT the disk. Look elsewhere:
1. CPU bound? Top shows %us > 80%
   → Profile with perf

2. Memory swapping? top shows swap in use
   → Add RAM or reduce memory usage

3. Network? Data going to remote system
   → Check iftop, tcpdump, iperf for bandwidth

4. Lock contention? Multiple threads fighting
   → strace shows futex() calls waiting
   → Look for database locks

5. Application code? Inefficient algorithm
   → Profile with perf to find hot functions

6. Cache effectiveness? Cache misses
   → perf stat -e cache-misses shows high misses

Diagnostic path:
top → Check CPU, Memory
iostat → Check disk
tcpdump/iftop → Check network
perf → Check CPU hotspots and cache
strace → Check if waiting on locks
```

### Exercise Solutions: I/O Analysis

**Problem 1: Establish Baseline**

```bash
iostat -x 1 5

# Example output (5 snapshots):
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           5.50    0.00   3.20    1.10    0.00   90.20

Device     r/s    w/s   rkB/s   wkB/s    await   %util
sda        12.4   23.1  456.2   890.3    8.5    18.2
sdb         2.1   5.3   100.4   234.1    12.3   6.5

# Baseline interpretation:
- sda is busier than sdb
- Average await for sda: 8.5ms (reasonable)
- %util: 18.2% (not saturated, has room)
- CPU: 5.5% user, 1.1% iowait (balanced)

Your baseline becomes the reference for:
- "Is this normal?"
- "Did wait time increase?"
- "Is this device busier than usual?"
```

**Problem 2: Create Workload and Measure**

```bash
# Baseline
iostat -x /dev/sda 1 3
# Example: await=5ms, %util=15%

# During dd if=/dev/zero of=/tmp/testfile bs=1M count=500
iostat -x /dev/sda 1 10
# Example: await=25ms, %util=85%

# Analysis:
await increased from 5ms to 25ms (5x increase)
%util increased from 15% to 85% (writing 500MB)

Interpretation:
- Write workload is saturating disk
- Queue building up (higher await)
- This is expected behavior for sequential writes
- If dd reports: 500MB written in 10 seconds = 50MB/s throughput
```

**Problem 3: Sequential vs Random I/O**

```bash
# Sequential read (from syslog):
dd if=/var/log/syslog of=/dev/null bs=1M
# iostat shows:
rkB/s=123456, r/s=120
# Average size = 123456/120 = 1024KB per request
# Large size = Sequential access

# Random read (from /dev/urandom):
dd if=/dev/urandom of=/tmp/random bs=4K count=1000
dd if=/tmp/random of=/dev/null bs=4K
# iostat shows:
rkB/s=2048, r/s=512
# Average size = 2048/512 = 4KB per request
# Small size = Random access (matches block size)

Comparison:
Sequential: rkB/s=123MB/s (fast, high throughput)
Random:     rkB/s=2MB/s (slow, low throughput)

On HDD: 50-100x difference for random vs sequential
On SSD: 2-5x difference (much better at random)

This demonstrates why:
- Databases are slow on HDD (random access)
- RAID prefetching helps sequential
- SSDs changed game for database performance
```

**Problem 4: vmstat During I/O**

```bash
# Baseline vmstat 1 5
vmstat
# Example:
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0  0      1000M  100M  2000M   0    0     10   15   234  456  5  3  90  2  0

# During dd if=/dev/zero of=/tmp/test bs=4K count=10000
vmstat 1 10
# Example:
 0  2  0      800M   200M  1800M   0    0    100   5000  100  200  1  15 60 24  0

# Analysis:
bo (blocks out): increased from 15 to 5000 (writing 5000 blocks/sec)
wa (iowait): increased from 2% to 24% (CPU waiting for I/O)
id (idle): decreased from 90% to 60% (less idle, more active)
b (processes blocked): 2 processes waiting

Interpretation:
- Disk writing heavily (bo=5000)
- CPU spending 24% waiting for I/O completion
- This is expected during heavy write workload

Verification:
Compare with iostat: bo from vmstat should match w/s from iostat
5000 blocks = ~5000 blocks/sec at 4K = 20MB/s writes
```

**Challenge: Complete I/O Diagnosis**

```bash
# Scenario: Storage is slow

# Step 1: Baseline
iostat -x 1 5
vmstat 1 5

# Step 2: During simulated workload
dd if=/dev/zero of=/tmp/test bs=1M count=200 &
iostat -x /dev/sda 1 10

# Results analysis:
If await > 20ms and %util > 80%:
  → Disk is saturated
  → More I/O requests than disk can handle
  → Queue is building up
  → Fix: Reduce concurrent I/O, upgrade disk, add RAID

If await normal but reported slow:
  → Disk not the bottleneck
  → Look at CPU, network, locks, memory

# Step 3: Find the process
iotop -b -n 1 | head -10
# Shows dd is the process doing I/O

# Step 4: Trace that process
strace -e read,write -c dd if=/dev/zero of=/tmp/test3 bs=1M count=10

# Step 5: Conclusions
# Disk shows: 100% util, 50ms await
# Conclusion: Disk IS the bottleneck
# Next: Identify if this is expected or abnormal
# Expected if: Sequential write workload, new baseline
# Abnormal if: Was 5ms await yesterday, now 50ms
#              → Something changed (backup running? corruption?)
```

---

## Common L1 TSE Scenarios from Week 6

### Scenario 1: "NFS mount times out"

```bash
# Investigation path using Week 6 tools

# Tool 1: strace (what's the application doing?)
strace mount -t nfs server:/export /mnt 2>&1 | tail -30
# Look for: socket creation, connect syscall, timeout

# Tool 2: tcpdump (does the packet reach server?)
sudo tcpdump host storage.server and port 2049 -n | head -20
# If SYN appears but no response: Network or firewall issue
# If no packets at all: DNS problem or client blocked

# Tool 3: iostat (is local disk slow, adding delay?)
iostat -x 1 3
# If disk looks normal: Problem is network or server

# Root cause scenarios:
Case 1: tcpdump shows SYN with no response
  → Server not listening or firewall blocking
  → Check: is NFS running on server?
  → Check: Are firewall rules allowing traffic?

Case 2: strace stuck on connect() syscall
  → Network not responding
  → Check: nc -zv server 2049
  → Check: route to server exists?

Case 3: strace stuck after connection
  → Connected but server slow to respond
  → Check: iostat on server (disk slow there?)
  → Check: Load average on server
```

### Scenario 2: "Performance degraded since this morning"

```bash
# Before-and-after comparison using Week 6 tools

# Get current state:
iostat -x 1 5
vmstat 1 5
top -b -n 1 | head -15
sudo tcpdump -c 100 2>/dev/null | tail -20

# Compare to known baseline (need to have been collecting):
# "Yesterday this system showed await=5ms, now it's 20ms"

# Investigation:

# 1. Is it constant or intermittent?
iostat -x /dev/sda 1 60 | grep -E "await|%util"
# Look for pattern (always high, or spikes?)

# 2. What's different?
# Check: New process? Higher load?
ps aux | head -20
uptime

# 3. Is it one device or all?
iostat -x 1 5 | head -20
# Multiple devices with high await? System-wide problem
# One device only? That device specific

# 4. What changed since yesterday?
# Check: Cron jobs
crontab -l
# Check: New backups running
ps aux | grep backup
# Check: Software updates
apt list --upgradable 2>/dev/null
journalctl --since "24 hours ago" -p err | head -20

# 5. tcpdump to see if network pattern changed:
# More traffic? Different type?
sudo timeout 10 tcpdump -i any -c 1000 2>/dev/null | tail -50

# Root cause examples:
"New backup started running at 3am, competing with normal I/O"
→ Reschedule backup or increase resources

"Disk developed bad sectors, causing slowdown"
→ Check dmesg for errors, replace disk

"Cache disabled by system update"
→ Check I/O cache settings, re-enable

"Network became congested"
→ Check iftop for bandwidth usage
```

### Scenario 3: "Application using way too much I/O"

```bash
# Use strace to see I/O pattern

# Start capture:
strace -e read,write,open,openat -f program 2>&1 > /tmp/trace.txt

# Analyze:
grep "read\|write" /tmp/trace.txt | head -50
# Count read/write calls:
grep -c "read(" /tmp/trace.txt
grep -c "write(" /tmp/trace.txt

# Look for patterns:
# Many small reads? Should buffer
# Repeated opens of same file? Should cache handle
# Seeking back and forth? Bad algorithm

# Example bad pattern:
for line in file:
    open(file)  # Each line reads entire file!
    close(file)

# Fix:
open(file) once
read all lines
close(file)

# Use iotop to confirm which process:
iotop -b -n 1 | head -15
# Shows exact process consuming I/O

# Use iostat during execution:
iostat -x 1 10
# Correlate high I/O rates with process behavior
```

---

## Week 6 Integration: When to Use Each Tool

```
Problem: Application is slow

Decision tree:

Is it responsive?
├─ No, hung
│   ├─ Use strace to see where it's blocked
│   ├─ If blocked on read/write:
│   │   └─ Use iostat to check disk
│   └─ If blocked on network:
│       └─ Use tcpdump to check traffic
│
└─ Yes, but throughput low
    ├─ Use iostat baseline first
    ├─ If disk is bottleneck (high await, %util):
    │   ├─ Use blktrace for I/O details
    │   └─ Use strace to see what process
    ├─ If CPU is bottleneck:
    │   └─ Use perf to find hotspot
    └─ If network is bottleneck:
        └─ Use tcpdump to analyze packets

```

---

## Key Takeaways for Week 6

1. **strace shows system calls** — where code interacts with kernel
2. **ltrace shows library calls** — where code uses libraries
3. **tcpdump shows network packets** — what flows over wire
4. **iostat shows I/O performance** — disk latency and throughput
5. **blktrace shows block layer** — kernel I/O queue behavior
6. **perf shows CPU hotspots** — where CPU time is spent
7. **Combine tools** — single tool shows one dimension, need multiple

---

Powered by UQS v1.8.5-C
