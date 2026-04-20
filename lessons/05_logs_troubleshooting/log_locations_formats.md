# Lesson 5.1: Log Locations and Formats (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Troubleshooting without logs is guessing. Logs tell the story of what happened and why
**Skills:** Know where logs are, understand formats, extract useful information

---

## Quick Review

### Two Logging Systems

```
Traditional (syslog):
├─ Text files in /var/log/
├─ Format: Timestamp, hostname, service, message
├─ Simple to parse with grep/awk
└─ Good for: Simple services, legacy systems

Modern (systemd/journald):
├─ Binary journal in /run/log/journal/ and /var/log/journal/
├─ Rich metadata (PID, UID, priority, etc.)
├─ Fast indexed searches
└─ Good for: Most modern Linux systems
```

**Both coexist on modern systems** — learn both.

---

## Key Concepts

### 1. Traditional Log Locations

**System logs:**
```
/var/log/syslog          (Debian/Ubuntu - general)
/var/log/messages        (RHEL/CentOS - general)
/var/log/auth.log        (Debian - authentication)
/var/log/secure          (RHEL - authentication)
/var/log/kernel.log      (Kernel messages)
/var/log/dmesg           (Early boot messages)
```

**Service logs:**
```
/var/log/apache2/        (Apache web server)
/var/log/nginx/          (Nginx web server)
/var/log/mysql/          (MySQL database)
/var/log/postgresql/     (PostgreSQL database)
/var/log/sshd.log        (SSH daemon)
/var/log/cron            (Cron jobs)
/var/log/mail.log        (Mail system)
```

**Storage-specific:**
```
/var/log/lustre/         (Lustre filesystem)
/var/log/nfs/            (NFS server)
/var/log/iscsi/          (iSCSI target)
```

**Application logs:**
```
Wherever the application puts them (varies)
Common: /var/log/myapp/, /opt/myapp/logs/, ~/.local/share/myapp/

Check application documentation
```

### 2. Standard Syslog Format

**Example:**
```
Jan 19 10:30:45 server kernel: [12345.678] Out of memory: Kill process nginx
│     │ │      │      │       │                           │
│     │ │      │      │       │                           └─ Message
│     │ │      │      │       └─ Facility (kernel, daemon, etc.)
│     │ │      │      └─ Hostname
│     │ │      └─ Time
│     └─ Date
└─ Month

Standard fields:
- Timestamp: Month Day HH:MM:SS
- Hostname: System where it happened
- Service: What generated it (kernel, sshd, nginx)
- Message: The actual log content
```

**Syslog priorities (levels):**
```
0 emerg   - System unusable (kernel panic)
1 alert   - Immediate action needed
2 crit    - Critical condition
3 err     - Error
4 warn    - Warning
5 notice  - Normal but significant
6 info    - Informational
7 debug   - Debug-level messages
```

### 3. JSON Format (Systemd)

**Modern logs often output JSON:**
```json
{
  "MESSAGE": "Connection refused",
  "PRIORITY": "3",
  "_SYSTEMD_UNIT": "sshd.service",
  "_PID": "1234",
  "_UID": "0",
  "_GID": "0",
  "_HOSTNAME": "server",
  "_COMM": "sshd",
  "_CMDLINE": "sshd -D",
  "_SYSTEMD_CGROUP": "/system.slice/sshd.service",
  "__REALTIME_TIMESTAMP": "1705678245123456",
  "_SOURCE_REALTIME_TIMESTAMP": "1705678245456789"
}
```

**Rich metadata:**
```
MESSAGE    = Actual log message
PRIORITY   = Severity level (0-7)
_PID       = Process ID
_UID       = User ID
_GID       = Group ID
_HOSTNAME  = Hostname
_COMM      = Command name
_CMDLINE   = Full command line
__MONOTONIC_TIMESTAMP = Time since boot
__REALTIME_TIMESTAMP  = Real time
```

### 4. Common Log Patterns

**Error patterns to look for:**

```bash
# Permission denied
Permission denied
Access denied
EACCES
Operation not permitted

# No space
No space left on device
ENOSPC
Disk full

# Connection issues
Connection refused
Connection timeout
Connection reset
Network unreachable
Host unreachable

# Service issues
service failed
service crashed
seg fault
segmentation fault

# Resource issues
Out of memory
OOM killer
Memory limit exceeded
Too many open files
```

**Example log analysis:**
```
Jan 19 10:00:01 server sshd[1234]: error: Could not load host key: /etc/ssh/ssh_host_rsa_key
├─ Time: 10:00:01
├─ Service: sshd (process 1234)
├─ Level: error
└─ Problem: Can't load SSH host key

Likely causes:
1. File permissions wrong on /etc/ssh/ssh_host_rsa_key
2. File deleted
3. Disk where /etc is mounted is full
4. File corruption
```

### 5. Log Rotation

**Why logs rotate:**
```
Logs keep growing indefinitely
Eventually fill disk
Need to: Archive old logs, start fresh
```

**Rotation configuration:**
```bash
/etc/logrotate.d/           # Rotation config
/var/log/syslog             # Current log
/var/log/syslog.1           # Yesterday's log
/var/log/syslog.2.gz        # Older, compressed
/var/log/syslog.3.gz        # Even older

Common rotation schedules:
- Daily (most common)
- Weekly
- Monthly
- When file reaches size (10MB, 100MB, etc.)
```

**Check rotation status:**
```bash
ls -la /var/log/syslog*      # See rotation files
head -1 /var/log/syslog.1    # Check if actually rotated
logrotate -d /etc/logrotate.conf  # Dry run
```

### 6. Log Retention

**Important for compliance:**
```
Keep logs for X days/months/years
Older logs deleted or archived

Check current retention:
grep /var/log /etc/logrotate.d/* | grep rotate
# Shows: rotate 7 (keep 7 weeks)

Also check journald retention:
journalctl --vacuum-time=7d  # Keep 7 days
journalctl --vacuum-size=100M  # Limit to 100MB
```

---

## Essential Commands

### Find Relevant Logs

```bash
# Search for filename
find /var/log -name "*storage*" -o -name "*lustre*"

# Search inside logs
grep -r "error" /var/log/*.log

# Most recent logs
ls -lt /var/log/*.log | head -10

# Check log size (find large ones)
du -sh /var/log/* | sort -rh | head -10
```

### View Logs

```bash
# Simple view
cat /var/log/syslog
tail -20 /var/log/syslog          # Last 20 lines
head -20 /var/log/syslog          # First 20 lines

# Follow in real-time
tail -f /var/log/syslog           # Like "less" mode
less /var/log/syslog              # Interactive
grep pattern /var/log/syslog      # Search

# Decompress archived logs
zcat /var/log/syslog.1.gz | grep error

# Count occurrences
grep -c "error" /var/log/syslog
```

### Check Service Logs via Journalctl

```bash
# Service-specific
journalctl -u nginx -n 50         # Last 50 entries
journalctl -u nginx -f            # Follow live

# By priority
journalctl -p err                 # Errors only
journalctl -p "warn..err"         # Warnings through errors

# Time range
journalctl --since "2 hours ago"
journalctl --since today

# JSON format
journalctl -u nginx -o json | jq '.[]'
```

### Understand What Happened

```bash
# Find error pattern
grep -E "error|warn|fail" /var/log/syslog | tail -20

# Show context around error
grep -B 5 -A 5 "error message" /var/log/syslog

# Timeline
grep "Jan 19 10:" /var/log/syslog | head -20

# By service
grep "nginx" /var/log/syslog | tail -10
```

---

## Interactive Exercise: Analyze a Log File

**Task 1: Explore log locations**
```bash
# What logs exist?
ls -la /var/log/

# Which are largest?
du -sh /var/log/* | sort -rh | head -10

# Are any rotated?
ls -la /var/log/syslog*
```

**Task 2: View a log file**
```bash
# Recent entries
tail -20 /var/log/syslog

# Scroll through
less /var/log/syslog

# Search for patterns
grep -i "error" /var/log/syslog | head -10
```

**Task 3: Check service logs**
```bash
# SSH logs
grep sshd /var/log/syslog | tail -20

# Or via journalctl
journalctl -u sshd -n 20

# Authentication attempts
grep "sshd" /var/log/auth.log | tail -10
```

**Task 4: Find recent issues**
```bash
# Errors in last hour
grep -i "error" /var/log/syslog | tail -5

# Or via journalctl
journalctl -p err --since "1 hour ago"

# Time range
journalctl --since "10:00:00" --until "10:30:00"
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Log Location:** Where are SSH authentication attempts logged?
   ```
   A) /var/log/syslog
   B) /var/log/auth.log
   C) /var/log/ssh
   D) /var/log/secure (RHEL)
   ```

2. **Syslog Format:** What's the hostname in this log line?
   ```
   Jan 19 10:30:45 webserver sshd[1234]: Connection closed
   A) Jan
   B) 10:30:45
   C) webserver
   D) sshd
   ```

3. **Log Rotation:** You see `/var/log/syslog.3.gz`. What does this mean?
   ```
   (Explain what happened and when)
   ```

4. **JSON Fields:** You see `"_PID": "1234"` in journalctl output. What is this?
   ```
   A) Process ID of the service that logged it
   B) Priority ID
   C) Port ID
   D) Process Index
   ```

5. **Error Pattern:** You see "Out of memory: Kill process" in logs. What's happening?
   ```
   (Explain what triggered this and implications)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Service won't start"

```bash
# Step 1: Check logs
tail -50 /var/log/syslog
journalctl -u myservice -n 50

# Look for error like:
# myservice[1234]: error: Could not bind to port 8080
# myservice[1234]: error: Permission denied
# myservice[1234]: error: Out of memory

# Step 2: Investigate specific error
# If "Permission denied":
ls -l /var/run/myservice/
# Check ownership

# If "Port in use":
ss -tlnp | grep 8080
# See what's using the port

# If "Out of memory":
free -h
# Check available memory
```

### Scenario 2: "Sudden performance drop"

```bash
# Check logs for changes
journalctl --since "1 hour ago" -p warn

# Look for:
# - OOM killer activating
# - Disk full
# - Network errors
# - Service restarts

# Find timestamp of change
grep "restart\|crash\|error" /var/log/syslog | grep "10:00"
# Then look at metrics at that time
```

### Scenario 3: "User can't access file"

```bash
# Check authentication logs
tail -20 /var/log/auth.log

# Look for permission denied:
grep "Permission denied" /var/log/auth.log | tail -5

# Or for specific user:
grep "username" /var/log/auth.log | tail -20
```

---

## Key Takeaways

1. **Logs are your detective work** — tells the story of what happened
2. **Know both systems** — traditional /var/log and journald
3. **Error patterns matter** — learn to recognize "permission denied", "no space", "connection refused"
4. **Context is crucial** — look before/after the error for context
5. **Check rotation** — old logs might be compressed or deleted
6. **Timestamps matter** — correlate with other events

---

## How Ready Are You?

Can you explain these?
- [ ] Where system logs are stored
- [ ] What syslog format looks like
- [ ] The difference between /var/log/syslog and /var/log/auth.log
- [ ] How to search logs efficiently
- [ ] What happens when logs rotate

If you checked all boxes, you're ready for Lesson 5.2.

---

Powered by UQS v1.8.5-C
