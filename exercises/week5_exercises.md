# Week 5 Exercises: Logs and Troubleshooting

---

## Exercise 5.1: Navigate Log Files

### Problem 1: Find the Right Log

```bash
# Task 1A: Locate system logs
ls -la /var/log/

# Record: Which files look relevant to:
# - SSH access attempts
# - System errors
# - Disk issues
# - Network problems
```

### Problem 2: Understand Log Format

```bash
# Task 2A: Read a log entry
tail -1 /var/log/syslog

# Example: Jan 19 10:30:45 server kernel: [12345.678] Out of memory

# Parse it:
# When? Jan 19 10:30:45
# Where? server (hostname)
# What? kernel (service)
# Why? Out of memory (message)
```

### Problem 3: Search Logs

```bash
# Task 3A: Find specific patterns
grep -i "error" /var/log/syslog | wc -l
# How many errors in all of syslog?

# Task 3B: Find recent errors
tail -100 /var/log/syslog | grep -i "error"

# Task 3C: Use journalctl
journalctl -p err -n 20
# Show last 20 errors
```

### Challenge: Log Exploration

```bash
# Create a report of your system's log status

echo "=== Log File Inventory ==="
du -sh /var/log/* | sort -rh

echo ""
echo "=== Largest Log Files ==="
find /var/log -type f -exec ls -lh {} \; | sort -k5 -rh | head -10

echo ""
echo "=== Recent Errors ==="
journalctl -p err -n 10 -o short

echo ""
echo "=== Log Rotation Status ==="
ls -la /var/log/syslog* | head -10
```

---

## Exercise 5.2: Parse Logs with Tools

### Problem 1: grep - Find Patterns

```bash
# Task 1A: Find all SSH activity
grep "sshd" /var/log/syslog | wc -l
# How many SSH entries?

# Task 1B: Find failed logins
grep "Failed password" /var/log/auth.log 2>/dev/null | wc -l
# Or if on system without auth.log:
journalctl | grep "Failed password" | wc -l

# Task 1C: Find with context
grep -B 2 -A 2 "error" /var/log/syslog | head -20
# What happened before/after each error?
```

### Problem 2: awk - Extract Fields

```bash
# Task 2A: Extract IPs from logs
grep "Failed password" /var/log/auth.log 2>/dev/null | awk '{print $(NF-3)}' | head -10
# Show IPs of failed login attempts

# Task 2B: Count by service
journalctl -n 100 | awk '{print $3}' | sort | uniq -c | sort -rn | head -10
# Which service appears most in logs?

# Task 2C: Extract timestamps
grep "error" /var/log/syslog | awk '{print $1, $2, $3}' | tail -10
# Show when errors occurred
```

### Problem 3: Combine Tools

```bash
# Task 3A: Find top error types
grep "error" /var/log/syslog | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head -5
# What are the most common errors?

# Task 3B: Find suspicious activity
grep "sshd" /var/log/syslog | grep -i "fail\|refused" | awk '{print $(NF-3), $(NF-2), $(NF-1), $NF}' | sort | uniq -c | sort -rn | head -10
# Which IPs are having SSH issues?

# Task 3C: Timeline analysis
journalctl --since "2 hours ago" | grep -i "error\|warn" | awk '{print $1 " " $2 " " $3}' | sort | uniq -c
# When did errors occur?
```

### Challenge: Real Log Analysis

```bash
# Analyze your system's actual logs

# 1. Find most common error message
journalctl -p err | awk -F'MESSAGE=' '{print $NF}' | cut -d' ' -f1-5 | sort | uniq -c | sort -rn | head -1

# 2. Find top services with errors
journalctl -p err | awk -F'_SYSTEMD_UNIT=' '{print $NF}' | cut -d' ' -f1 | sort | uniq -c | sort -rn | head -5

# 3. Find errors in last hour
journalctl -p err --since "1 hour ago" | wc -l

# 4. Find busiest hour
journalctl --since "24 hours ago" | awk '{print $1 " " $2}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -5
```

---

## Exercise 5.3: Troubleshoot Real Scenarios

### Scenario 1: "Service Won't Start"

```bash
# Given: myservice fails to start
# Steps:

# 1. Check service status
systemctl status myservice
# Look for error hint

# 2. Check logs
journalctl -u myservice -n 50
# What error message?

# 3. Parse the error
# Common causes:
# - "Permission denied" → Check file permissions
# - "Port in use" → Another service on that port
# - "Out of memory" → Not enough RAM
# - "File not found" → Missing config or binary

# 4. Fix based on error
# If "Permission denied":
sudo ls -la /var/run/myservice/
sudo chown myservice:myservice /var/run/myservice/

# 5. Try again
systemctl restart myservice
systemctl status myservice
```

### Scenario 2: "Backup Failing"

```bash
# Given: Backup times out every night

# 1. Check backup logs
tail -100 /var/log/backup.log | grep -i "error\|timeout\|failed"

# 2. Find timestamps of failures
grep "TIMEOUT\|FAIL" /var/log/backup.log | awk '{print $1, $2, $3}' | uniq -c

# 3. Correlate with system events
journalctl --since "22:00:00" --until "23:00:00" -p warn

# 4. Check for competing jobs
crontab -l
# What else runs at backup time?

# 5. Hypothesis and fix
# If disk full:
du -sh /backup
# Clean old backups:
find /backup -type f -mtime +90 -delete

# If competing job:
# Reschedule backup or competing job

# 6. Test
# Run backup manually:
/opt/backup/backup.sh --verbose
```

### Scenario 3: "Slow Storage"

```bash
# Given: Storage performance degraded today

# 1. Establish baseline
# (Skip if you have from before)
iostat -x 1 10

# 2. Compare with current
iostat -x 1 10 > current.txt

# 3. Analyze difference
# If %util > 80%: Disk bottleneck
# If await > 10ms: Slow disk or contention
# If throughput low: Limited bandwidth

# 4. Check logs for changes
journalctl --since "today" -p warn | head -20

# 5. Identify culprit
iotop -b -n 1
# What process is using I/O?

# 6. Fix
# If one process: Kill or reschedule
# If disk: Add RAID, upgrade to SSD
# If network: Check bandwidth usage
```

### Challenge: Create Troubleshooting Runbook

```bash
# Document the steps for troubleshooting "storage is slow"

cat > /tmp/storage_slow_runbook.txt << 'EOF'
TROUBLESHOOTING: Storage is Slow

Step 1: Establish impact
- How many users affected?
- How slow? (actual vs expected throughput)
- When did it start?

Step 2: Get metrics
iostat -x 1 10        (Disk status)
iftop -n -s 2         (Network usage)
top -b -n 1           (Process status)
df -h                 (Disk space)

Step 3: Analyze logs
journalctl -p err --since "1 hour ago"   (Errors)
journalctl -u lustre --since "1 hour ago" (Lustre logs)

Step 4: Identify bottleneck
CPU? → Use perf top
I/O? → Check iostat %util and await
Network? → Use iperf3 to test bandwidth
Memory? → Check for swapping

Step 5: Check for concurrent activity
ps aux                            (All processes)
crontab -l                        (Scheduled jobs)
systemctl list-units --running    (Services)

Step 6: Fix based on root cause
[See specific cause above]

Step 7: Verify resolution
iostat -x 1 10        (Metrics back to normal?)
Run test               (Performance acceptable?)
Check logs            (Any new errors?)
EOF

cat /tmp/storage_slow_runbook.txt
```

---

## Self-Assessment

After Week 5, you should be able to:

- [ ] Navigate log files and find relevant entries
- [ ] Use grep to filter log lines
- [ ] Use awk to extract fields from logs
- [ ] Combine tools in pipelines to analyze logs
- [ ] Identify error patterns and trends
- [ ] Troubleshoot common storage issues
- [ ] Escalate appropriately with good documentation
- [ ] Read journalctl output and understand metadata

---

Powered by UQS v1.8.5-C
