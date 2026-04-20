# Week 5 Solutions: Logs and Troubleshooting

---

## Lesson 5.1: Log Locations and Formats

### Challenge Questions Answers

**1. SSH Log Location**
```
Answer: B) /var/log/auth.log (Debian/Ubuntu)
         D) /var/log/secure (RHEL/CentOS)

Explanation:
Different distributions use different log files:
- Debian/Ubuntu: /var/log/auth.log
- RHEL/CentOS: /var/log/secure
- Or use: journalctl -u sshd (works on all)

To find SSH logs:
grep "sshd" /var/log/syslog     (if no dedicated file)
journalctl _SYSTEMD_UNIT=sshd.service  (modern way)
```

**2. Hostname Extraction**
```
Answer: C) webserver

Log line: Jan 19 10:30:45 webserver sshd[1234]: Connection closed
          │                │
          │                └─ hostname
          └─ timestamp

Standard syslog format:
[Timestamp] [Hostname] [Service][PID]: [Message]
```

**3. Log Rotation Meaning**
```
Answer: /var/log/syslog.3.gz indicates:
- syslog.3 = 3rd most recent (4 days old, if daily rotation)
- .gz = compressed with gzip

Timeline:
Day 1: /var/log/syslog (current, growing)
Day 2: /var/log/syslog.1 (yesterday), syslog (new)
Day 3: /var/log/syslog.2, syslog.1, syslog (new)
Day 4: /var/log/syslog.3, syslog.2, syslog.1, syslog (new)
Day 5: syslog.3.gz (4+ days old, now compressed), syslog.2, syslog.1, syslog (new)

Eventually old logs deleted based on retention policy
```

**4. JSON Field _PID**
```
Answer: A) Process ID of the service that logged it

_PID = 1234 means the process with PID 1234 generated this log entry
This is part of journalctl's rich metadata:
- _PID: Process ID
- _UID: User ID
- _GID: Group ID
- _HOSTNAME: Hostname
- MESSAGE: The actual log message
```

**5. Out of Memory Event**
```
Answer: Linux kernel's OOM killer is terminating processes

Explanation:
"Out of memory: Kill process" means:
1. System RAM is completely full
2. All swap is used
3. Kernel can't allocate more memory
4. Kernel activates OOM killer
5. OOM killer selects a process to terminate
6. Selected process is killed (possibly data loss)

Implications:
- This system is in severe memory pressure
- Immediate action needed
- Either:
  a) Add more RAM
  b) Reduce memory use
  c) Identify memory leak

Prevention:
- Monitor memory usage proactively
- Set up memory limits per process
- Configure swap appropriately
```

---

## Lesson 5.2: Parsing Logs

### Challenge Questions Answers

**1. grep -v Command**
```
Answer: B) Find lines WITHOUT errors

Explanation:
grep: Find lines matching pattern
grep -v: Find lines NOT matching pattern (v = invert)

Examples:
grep "error" file        # Lines WITH error
grep -v "error" file     # Lines WITHOUT error
grep -v "^#" file        # Remove comments (lines starting with #)
grep -v "^$" file        # Remove blank lines
```

**2. awk $NF Field**
```
Answer: B) Last field

Explanation:
In awk:
- $1 = first field
- $2 = second field
- $NF = N(umber of) F(ields), the last field
- $(NF-1) = second-to-last field

Examples:
echo "a b c d" | awk '{print $NF}'      # Output: d
echo "one two three" | awk '{print $(NF-1)}'  # Output: two

Useful for:
- Getting last column (often the message in logs)
- Getting second-to-last IP in auth logs
```

**3. Pipeline Explanation**
```
Answer: This pipeline finds the top error messages

Breaking it down:
grep "error" /var/log/syslog
│ Find all lines containing "error"

| awk '{print $5}'
│ Extract field 5 from each line

| sort
│ Sort the extracted fields

| uniq -c
│ Count occurrences of each unique value

| sort -rn
│ Sort numerically in reverse (highest first)

Result:
   10 "Permission denied"
    5 "Timeout occurred"
    3 "File not found"

Usage: Identify most common error types
```

**4. sed Replace Issue**
```
Answer: A) It only replaces the first per line

Explanation:
sed 's/old/new/' file
│ s = substitute
│ /old/new/ = find "old", replace with "new"
│ (missing the 'g' flag for global)

This only replaces FIRST occurrence per line
To replace ALL:
sed 's/old/new/g' file      # g = global (all per line)

The 'i' flag (if used):
sed -i 's/old/new/g' file   # i = in-place (modifies file!)

Caution: Always test sed without -i first!
```

**5. Finding Timeout Sources**
```
Answer: Pipeline to find IPs causing timeouts

Step 1: Find timeout lines
grep "Connection timeout" /var/log/syslog

Step 2: Extract IP (field varies by service)
# Example for SSH:
grep "Connection timeout" /var/log/auth.log | awk '{print $(NF-3)}'

Step 3: Count and sort
grep "Connection timeout" /var/log/auth.log | \
  awk '{print $(NF-3)}' | \
  sort | uniq -c | sort -rn

Output example:
  15 192.168.1.100
   8 10.0.0.50
   2 203.0.113.200

These IPs are timing out most frequently
```

---

## Lesson 5.3: Production Scenarios

### Challenge Questions Answers

**1. When to Escalate**
```
Answer: B) Escalate with documentation

Explanation:
- 30 minutes of troubleshooting = good faith effort
- 45 minutes with no progress = time to escalate
- Escalation is not failure, it's efficiency

What to include in escalation:
□ Problem statement
□ Timeline (when it started)
□ Error messages (exact text)
□ What you tried
□ Current logs
□ Specific question for next level
```

**2. Thousands of Same Error**
```
Answer: B) Service is in error/retry loop

Explanation:
Thousands of identical errors in 1 minute = not one-off
Service is continuously hitting the same error

Causes:
1. Startup error that keeps retrying (most common)
2. Persistent misconfiguration
3. External dependency unavailable (database, etc.)

Symptoms:
- Error occurs every N milliseconds
- Service keeps restarting
- Load average increasing
- System resources consumed by retries

Fix:
1. Stop the service: systemctl stop myservice
2. Fix the underlying issue
3. Restart: systemctl start myservice
4. Monitor: journalctl -u myservice -f
```

**3. Errors at Exact Times**
```
Answer: B) Scheduled task/cron job running

Explanation:
Errors at exactly 2:00 AM every night indicates:
- cron job scheduled for that time
- Automated task (backup, sync, maintenance)
- Potentially another service starting

Investigation:
crontab -l              # User cron jobs
cat /etc/crontab        # System cron
ls /etc/cron.d/         # Cron directory
systemctl list-timers   # systemd timers

Example:
0 2 * * * /opt/backup/backup.sh
└─ Runs at 2:00 AM (minute 0, hour 2)

If errors spike at 2:00 AM:
- Backup is starting
- Might be competing with another task
- Or the backup itself has an issue
```

**4. Best Escalation Documentation**
```
Answer: B) Exact error messages and timeline

Explanation:
Good escalation:
"Service started failing at 10:30 AM today.
Error message: 'Connection refused to database at 10.0.0.50:5432'
Tried: Restarting service (no change), checking network (ping works).
Logs show error recurring every 10 seconds.
Database team: Is 10.0.0.50 up and accepting connections?"

Bad escalation:
"Storage is broken. Fix it."

Why timeline matters:
- Helps identify what changed
- Correlates with other events
- Narrows down root cause

Why error messages matter:
- Different errors = different fixes
- Exact text helps with searching
```

**5. Backup Failed Halfway**
```
Answer: Recovery steps in order

Step 1: Assess what was backed up
# Check backup log
tail -100 /var/log/backup.log | grep -i "complete\|fail"
# See if partial data was written

Step 2: Don't re-run backup yet!
# First understand why it failed
# Fix must come before retry

Step 3: Diagnose failure reason
journalctl --since "backup start time" -p err
# What error caused it to stop?

Step 4: Fix the cause
If "Disk full": Free up space
If "Network timeout": Check connectivity
If "Permission denied": Fix access

Step 5: Run backup again
# Now safe to retry
/opt/backup/backup.sh

Step 6: Verify completion
# Check that backup finished
tail -10 /var/log/backup.log | grep -i "success\|complete"

Step 7: Document incident
# For future reference and learning
```

---

## Exercise Solutions

### Exercise 5.1: Log Navigation

**Expected findings:**

```bash
$ ls -la /var/log/
# Key files:
# syslog or messages = general system log
# auth.log or secure = authentication attempts
# kernel.log = kernel messages
# Various service logs: apache2/, nginx/, mysql/, etc.

$ du -sh /var/log/*
# Find largest log files
# If one is very large: May be runaway logging

$ tail -1 /var/log/syslog
# Example: Jan 19 14:30:25 server sshd[1234]: Accepted publickey for user from 192.168.1.100 port 54321 ssh2
# Parsed:
# When: Jan 19 14:30:25
# Where: server
# What: sshd (SSH service)
# Why: User successfully authenticated
```

---

### Exercise 5.2: Log Parsing

**Example pipeline results:**

```bash
# Task 1B: Failed logins
$ grep "Failed password" /var/log/auth.log | wc -l
247
# 247 failed login attempts

# Task 2B: Top services
$ journalctl -n 100 | awk '{print $3}' | sort | uniq -c | sort -rn | head -10
 45 sshd[1234]:
 30 kernel:
 15 systemd:
 10 sudo[5678]:
# sshd is the most active service in recent logs

# Task 3A: Most common errors
$ grep "error" /var/log/syslog | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head -5
  23 Permission denied
  15 Connection timeout
   8 No space left on device
   5 File not found
   3 Out of memory

# Analysis: Permission errors are most common
# Likely cause: File permissions wrong somewhere
```

---

## Troubleshooting Flowchart

```
Problem Reported
    ↓
1. GATHER INFO
   - What's broken?
   - When did it start?
   - What changed?
   - Who's affected?
    ↓
2. CHECK LOGS
   - Grep for error messages
   - Find timestamps
   - Look for patterns
    ↓
3. IDENTIFY ROOT CAUSE
   ├─ Permission? → Fix file permissions
   ├─ Disk full? → Clean up or expand
   ├─ Network? → Check connectivity
   ├─ Service down? → Restart and check logs
   ├─ Performance? → Check metrics
   └─ Unknown? → Escalate
    ↓
4. VERIFY FIX
   - Error gone?
   - No regressions?
   - Performance normal?
    ↓
5. DOCUMENT
   - What was wrong
   - How it was fixed
   - How to prevent
```

---

## Common Log Patterns Quick Reference

```
Permission denied        → File permissions wrong
Connection refused       → Service not listening / port blocked
Connection timeout       → Network latency / firewall
Disk full               → Need to clean up or expand
Out of memory           → Memory leak or need more RAM
Segmentation fault      → Application crash / bug
Address already in use  → Port in use, restart needed
Cannot find file        → Path wrong or file deleted
Timeout                 → Service slow or unresponsive
Authentication failed   → Wrong password / permissions
```

---

Powered by UQS v1.8.5-C
