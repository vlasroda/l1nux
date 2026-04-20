# Lesson 5.3: Real Production Scenarios and Troubleshooting (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Theory is nice. Real production problems are messy, urgent, and complex. Master these scenarios and you'll be ready for actual TSE work
**Skills:** Troubleshoot actual problems, escalate appropriately, document findings

---

## Quick Review

### Troubleshooting Methodology

```
1. Gather Information
   ├─ Describe the problem (what's broken?)
   ├─ When did it start?
   ├─ What changed?
   └─ What's the impact?

2. Examine Logs
   ├─ Error messages?
   ├─ Timestamps?
   ├─ Related events?
   └─ Pattern/frequency?

3. Verify Hypothesis
   ├─ Run tests
   ├─ Check metrics
   ├─ Compare before/after
   └─ Rule out alternatives

4. Take Action
   ├─ Implement fix
   ├─ Monitor closely
   ├─ Prepare rollback
   └─ Document everything

5. Verify Resolution
   ├─ Problem is gone?
   ├─ No regressions?
   ├─ Performance normal?
   └─ User confirmed?
```

---

## Key Concepts

### 1. Storage System Problems

**Problem: "Storage is slow"**

```bash
# Step 1: Gather baseline
iostat -x 1 10
vmstat 1 10
journalctl -p warn --since "1 hour ago"

# Step 2: Identify culprit
# Is disk utilization high?
iostat -x | grep %util
# Is network bottleneck?
iperf3 -c storage.server
# Is cache hit rate low?
# Check application metrics

# Step 3: Check logs
journalctl -u lustre -p err
journalctl -u nfs -p err
grep -i "error\|timeout\|slow" /var/log/storage.log

# Step 4: Fix
# If disk: Add RAID, upgrade to SSD, optimize I/O
# If network: Improve bandwidth, reduce latency
# If cache: Clear if necessary, review hit rate

# Example log analysis:
grep "ost_brw_read" /var/log/lustre/lustre.log | \
  awk '{print $NF}' | \
  awk '{sum+=$1; count++} END {print "Avg latency:", sum/count "ms"}'
```

**Problem: "NFS mount times out"**

```bash
# Step 1: Check connectivity
ping nfs.server
nc -zv nfs.server 2049
time nslookup nfs.server

# Step 2: Check logs
tail -50 /var/log/syslog | grep -i "nfs\|timeout"
journalctl -p err --since "1 hour ago"

# Step 3: Specific NFS logs
# Check server-side logs
showmount -e nfs.server  # Are exports visible?
nfsstat -s               # Server stats

# Check client-side logs
nfsstat -c               # Client stats
mount | grep nfs         # What's mounted?

# Step 4: Analysis
# If DNS slow: Use /etc/hosts workaround
# If port blocked: Open firewall (ports 2049, 111)
# If server overloaded: Redirect to different server
# If network latency high: Check mtr, route changes

# Example fix:
echo "10.0.0.50 nfs.server" | sudo tee -a /etc/hosts
mount -t nfs nfs.server:/export /mnt
```

**Problem: "Backup failing randomly"**

```bash
# Step 1: Check backup logs
tail -100 /var/log/backup.log

# Step 2: Find error patterns
grep -i "error\|failed\|timeout" /var/log/backup.log | tail -20

# Step 3: Check when it fails
grep "failed" /var/log/backup.log | awk '{print $1, $2, $3}' | sort | uniq -c

# Step 4: Correlate with system events
# Check if concurrent with other operations
journalctl --since "HH:MM:SS" --until "HH:MM:SS" -p err

# Step 5: Investigate specific error
# "Connection reset" → Network issue
# "No space left" → Disk full
# "Permission denied" → Access issue
# "Timeout" → Performance or service down

# Example: Disk full causing failures
du -sh /backup/*
# Find old backups to clean
find /backup -type f -mtime +90 -delete
```

### 2. Network Problems

**Problem: "Slow data transfer between sites"**

```bash
# Step 1: Baseline
iperf3 -c remote.server
ping -c 100 remote.server | tail -1

# Step 2: Check path
mtr -c 50 remote.server

# Step 3: Check logs
journalctl -p warn --since "1 hour ago" | grep -i "network\|timeout"

# Step 4: Identify bottleneck
# If bandwidth low:
iftop
# Is WAN link saturated?

# If latency high:
# Check route: traceroute remote.server
# Check ISP status
# Consider QoS or traffic shaping

# Step 5: Fix
# Prioritize storage traffic
# Reduce other traffic
# Or accept lower throughput
```

**Problem: "Intermittent connection issues"**

```bash
# Step 1: Reproduce and log
# Keep detailed log of failures
date >> /tmp/failure_log.txt
ping remote.server >> /tmp/failure_log.txt

# Step 2: Check interface status
ip link show
# Look for: UP, DOWN, NO-CARRIER

# Step 3: Check for resets
dmesg | grep -i "reset\|down\|error" | tail -20

# Step 4: Check routing
ip route show
# Are there alternate routes?

# Step 5: Check DHCP/DNS
cat /etc/resolv.conf
ip addr show

# Example: MTU mismatch
# If random packet loss:
ping -M do -s 1472 remote.server
ping -M do -s 8972 remote.server
# If one fails: MTU mismatch

# Fix:
ip link set dev eth0 mtu 1500
```

### 3. Permission and Access Problems

**Problem: "User can't access storage"**

```bash
# Step 1: Check what user is trying to do
journalctl -p err -u storage --since "1 hour ago"

# Step 2: Check auth logs
tail -50 /var/log/auth.log | grep username

# Step 3: Check permissions
ls -ld /storage
# Check owner, group, mode

# Step 4: Check if user in right group
id username
groups username

# Step 5: Fix
# Add to group:
sudo usermod -a -G storage username

# Or fix directory permissions:
sudo chmod g+rx /storage

# Or fix ownership:
sudo chown user:group /storage
```

**Problem: "Permission denied on mount"**

```bash
# Step 1: Check export permissions
showmount -e nfs.server
# Does it show your network?

# Step 2: Check export config on server
# Server-side: /etc/exports (NFS) or /etc/fstab (iSCSI)
# Check if your IP/network is allowed

# Step 3: Check mount options
mount | grep nfs.server
# Right uid/gid/mode?

# Step 4: Fix
# If NFS export missing your network:
# Edit /etc/exports on server, add:
# /export 10.0.0.0/24(rw,no_subtree_check)
# exportfs -ra

# Or remount with correct options:
sudo umount /mnt
sudo mount -t nfs nfs.server:/export /mnt -o rw,hard,intr
```

### 4. Escalation and Documentation

**When to escalate:**
```
Escalate if:
- Problem persists after 30 minutes of diagnosis
- Requires data recovery
- Affects SLA/critical service
- Hardware failure suspected
- Unknown error pattern

How to escalate:
1. Document findings in detail
2. Include exact error messages
3. Show timeline of events
4. Explain attempted fixes
5. Include metrics/logs
6. Be specific in the question
```

**What to document:**
```
□ Problem statement (what's broken)
□ When it started (date/time)
□ Impact (how many users, how much data)
□ Error messages (exact text)
□ Steps taken (what you tried)
□ Findings (what you discovered)
□ Current status (still broken?)
□ Attached logs (relevant excerpts)
```

### 5. Common Patterns

**"Nothing in logs"**
```
Symptoms:
- No error messages
- But service is broken

Causes:
- Logs not being written (disk full)
- Service isn't logging
- Wrong log file
- Time sync issues (timestamps off)

Investigation:
# Check if logs are being updated
tail -f /var/log/syslog
# Should see entries in real-time

# Check timestamp
date
journalctl | tail -1 | awk '{print $1, $2, $3}'
# Should match approximately

# Check service logging
systemctl status myservice
# Does it mention logging?
```

**"Logs full of same error"**
```
Symptoms:
- Thousands of identical errors
- Log file growing rapidly
- Errors repeating every N seconds

Causes:
- Persistent issue (not recovering)
- Service in retry loop
- Misconfiguration

Fix:
# Stop the service
systemctl stop myservice

# Fix the underlying issue
# Then restart
systemctl start myservice

# Monitor to ensure it doesn't repeat
journalctl -u myservice -f
```

**"Error happens intermittently"**
```
Symptoms:
- Sometimes works, sometimes fails
- Timing appears random
- Hard to reproduce

Causes:
- Race condition
- Resource contention
- Timing-sensitive bug
- External dependency (network, disk, etc.)

Investigation:
# Correlate with other events
journalctl --since "time of failure" --until "10 sec after"

# Check concurrent activity
top -b -n 1 | head -15
iostat -x 1 1

# Monitor for pattern
journalctl -u myservice -f | grep -i error
# Run for extended time, note frequency

# Check timestamps of failures
journalctl -u myservice -p err | awk '{print $1 " " $2 " " $3}' | uniq -c | sort -rn
# Do failures cluster at certain times?
```

---

## Interactive Exercise: Troubleshoot a Scenario

**Scenario: "Storage backup window is failing, losing 2 hours per night"**

**Your task:**

```bash
# Step 1: Gather info
echo "Problem: Backup failing in nightly window"
echo "Impact: Missing ~2 hours of backup per night"
echo "Duration: Started 3 days ago"

# Step 2: Check backup logs
tail -100 /var/log/backup/backup_*.log | grep -i "error\|failed"

# Step 3: Find timestamps of failures
grep "FAIL\|ERROR" /var/log/backup/backup_*.log | awk '{print $1, $2, $3}' | head -10

# Step 4: Check system state at failure time
# Correlate with storage metrics
journalctl --since "2024-01-18 22:00:00" --until "2024-01-19 02:00:00" -p err

# Step 5: Identify pattern
# Does it fail at specific time? (2:00 AM every night?)
# Does it fail after N hours of backup?
# Is it related to other scheduled jobs?

# crontab -l
# What else runs at night?

# Step 6: Hypothesis
# Theory: Backup finishes 90% -> disk full -> fails
# Theory: Storage server reboots at 2am -> connection lost
# Theory: Network backup deduplication runs at 2am -> contention

# Step 7: Verify and fix
# If disk: Increase backup storage or compress more
# If reboot: Check server logs, disable auto-reboot
# If contention: Reschedule competing job
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Escalation:** You've spent 45 minutes troubleshooting NFS timeouts. What should you do?
   ```
   A) Keep trying, don't escalate
   B) Escalate with documentation
   C) Restart the NFS server
   D) Blame the network team
   ```

2. **Log Analysis:** Logs show 5000 "Permission denied" errors in 1 minute. What happened?
   ```
   A) User is trying to hack in
   B) Service is in error loop
   C) Permissions were accidentally changed
   D) Disk is full
   ```

3. **Pattern Recognition:** Errors spike at exactly 2:00 AM every night. What's likely happening?
   ```
   A) Time zone issue
   B) Scheduled task/cron job running
   C) User in different timezone
   D) Cannot determine from this info
   ```

4. **Documentation:** What's the most important to document when escalating?
   ```
   A) Every command you ran
   B) Exact error messages and timeline
   C) Your guesses about the cause
   D) How frustrated you are
   ```

5. **Recovery:** Backup failed halfway through. Data already deleted. What do you do?
   ```
   (Explain step-by-step recovery approach)
   ```

---

## Common Escalation Points

**Know when to escalate:**
```
- "I don't know what this error means" → Ask senior/Google
- "Requires hardware replacement" → Escalate to vendor/hardware team
- "SLA at risk" → Alert management immediately
- "Data recovery might be needed" → Escalate to data team
- "Multiple systems affected" → Escalate to incident commander
- "Tried X, Y, Z and still broken" → Escalate after 30 min
```

**Good escalation looks like:**
```
"NFS mount on storage01 timing out since 10:30 AM.
Users in data center affected (~50 people).
Tried: Ping (works), nc port 2049 (times out), remount (no change).
Server logs show no errors.
Network team: can you check 10Gbps connection to storage01?"
```

---

## Key Takeaways

1. **Follow the method** — gather info, examine logs, verify, act, verify resolution
2. **Logs tell the story** — but you must know how to read them
3. **Escalate early** — after 30 min of fruitless troubleshooting
4. **Document everything** — future you will thank present you
5. **Test your fix** — don't assume it worked without verification
6. **Communication matters** — clear escalation saves hours of guessing

---

## How Ready Are You?

Can you explain these?
- [ ] How to troubleshoot "storage is slow" systematically
- [ ] When to escalate and what to document
- [ ] How to find what error is repeating most
- [ ] What to do when logs show nothing
- [ ] How to correlate events with failure times

If you checked all boxes, you're ready for Week 5 exercises.

---

Powered by UQS v1.8.5-C
