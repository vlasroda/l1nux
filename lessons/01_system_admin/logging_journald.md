# Lesson 1.3: Logging with journald (Refresher)

**Time:** 20-30 minutes
**Why it matters:** 60% of troubleshooting starts with "check the logs" — need to know where they are and how to read them
**Skills:** Query journalctl, filter by service/time/priority, find errors, tail logs in real-time

---

## Quick Review

### Two Logging Systems on Modern Linux

**Traditional (still exists):**
- Files in `/var/log/` (syslog, messages, auth, etc.)
- Text-based, simple to parse with grep/awk
- Slower, limited retention

**Modern (systemd/journald):**
- Binary journal in `/run/log/journal/` and `/var/log/journal/`
- Indexed, fast queries, automatic rotation
- Rich metadata (timestamps, PIDs, priorities, etc.)

**For DDN/TSE work:** You need both. Some legacy systems use syslog. Modern systems use journald. Storage systems log to both.

---

## Key Concepts

### 1. The Journal (journald)

Every systemd system has a journal that captures:
- System messages (kernel)
- Service logs (nginx, mysql, custom apps)
- User login/logout events
- Hardware events

```bash
journalctl      # View entire journal (newest at bottom)
journalctl -e   # Jump to end of journal
journalctl -f   # Follow in real-time (like tail -f)
```

### 2. Journal Storage

```
/run/log/journal/       # Volatile (lost on reboot)
/var/log/journal/       # Persistent (survives reboot)
```

By default, most systems use persistent storage if `/var/log/journal/` exists.

Check:
```bash
ls -la /var/log/journal/
# or
journalctl --disk-usage     # How much space is journal using?
```

### 3. Log Levels (Priority)

From most severe to least:
```
0 = emerg   (system is unusable)
1 = alert   (action must be taken immediately)
2 = crit    (critical condition)
3 = err     (error condition)
4 = warn    (warning condition)
5 = notice  (normal but significant)
6 = info    (informational)
7 = debug   (debug-level messages)
```

Filter by level:
```bash
journalctl -p err           # Errors and above
journalctl -p warn..err     # Warnings through errors
journalctl -p info          # Info and above
```

### 4. Units (Services)

A "unit" is any systemd-managed thing (service, socket, mount, etc.).

```bash
journalctl -u nginx         # All nginx logs
journalctl -u nginx -f      # Follow nginx in real-time
journalctl -u systemd-user-sessions  # User session logs
journalctl -u ssh           # SSH logs
```

### 5. Time-Based Filtering

```bash
journalctl --since "1 hour ago"     # Last hour
journalctl --until "2023-01-19 10:00:00"  # Before this time
journalctl --since today            # Since midnight today
journalctl --since "2023-01-19" --until "2023-01-20"  # Full day

# Combined
journalctl -u nginx --since "1 hour ago" -p err
```

### 6. Output Formats

```bash
journalctl                  # Default (readable)
journalctl -o json          # JSON (machine-friendly)
journalctl -o short         # Compact
journalctl -o verbose       # Detailed metadata
journalctl -o cat           # Just the message (no timestamps)

# Pipe to jq for JSON parsing
journalctl -u myapp -o json | jq '.[].MESSAGE'
```

### 7. Traditional Log Files

Still useful on systems with persistent syslog:

```
/var/log/syslog         # General system log (Debian/Ubuntu)
/var/log/messages       # General system log (RHEL/CentOS)
/var/log/secure         # Authentication events
/var/log/auth.log       # Authentication (Debian)
/var/log/kernel.log     # Kernel messages
```

View:
```bash
tail -f /var/log/syslog         # Real-time
grep "error" /var/log/syslog    # Search
```

---

## Essential Commands

### View Logs Basics

```bash
# Entire journal (recent first)
journalctl -e

# Jump to time period
journalctl --since "2 hours ago"
journalctl --since "2023-01-19 09:30:00" --until "2023-01-19 10:30:00"

# Today's logs
journalctl --since today

# Last boot
journalctl -b              # Current boot
journalctl -b -1           # Previous boot
```

### Filter by Service

```bash
# All logs from a service
journalctl -u nginx
journalctl -u mysql
journalctl -u ssh

# Multiple services
journalctl -u nginx -u mysql

# All user services for a user
journalctl --user-unit=myservice
```

### Filter by Severity

```bash
# Only errors and above
journalctl -p err

# Warning through error
journalctl -p "warn..err"

# Debug messages (verbose)
journalctl -p debug

# Info only (not warnings)
journalctl -p 6
```

### Follow in Real-Time

```bash
# All logs
journalctl -f

# Single service
journalctl -u nginx -f

# With timestamps
journalctl -u nginx -f --no-pager

# Color output
journalctl -u nginx -f | less -R
```

### Search and Filter

```bash
# Grep-like search in logs
journalctl | grep "error"

# By process ID
journalctl _PID=1234

# By user ID
journalctl _UID=1000

# Combining filters
journalctl -u nginx -p err --since "1 hour ago"
```

### Get Detailed Information

```bash
# All metadata for recent entries
journalctl -o verbose -n 10

# Show available boots
journalctl --list-boots

# Disk usage
journalctl --disk-usage

# Journal statistics
journalctl -n 0 --no-pager

# Follow specific request
journalctl -xe     # With explanations
```

### Manage Journal Size

```bash
# Rotate journal
journalctl --rotate

# Vacuum to max 100M
journalctl --vacuum-size=100M

# Keep only 7 days
journalctl --vacuum-time=7d

# Limit to max 1 month per boot
journalctl --vacuum-files=12
```

---

## Interactive Exercise: Troubleshoot a Service Issue

**Scenario:** User reports nginx isn't serving requests. You have 5 minutes to diagnose.

**Step 1: Check if service is running**
```bash
systemctl status nginx
# If stopped, try starting
sudo systemctl start nginx
# Check status again
systemctl status nginx
```

**Step 2: Look at recent nginx logs**
```bash
journalctl -u nginx -n 50
# Or if nginx logs to file:
tail -50 /var/log/nginx/error.log
```

**Step 3: Find errors in last hour**
```bash
journalctl -u nginx --since "1 hour ago" -p err
# Or errors since last restart:
journalctl -u nginx -b -p err
```

**Step 4: Check what nginx is listening on**
```bash
ss -tlnp | grep nginx
# Should see something like: LISTEN 0 128 *:80 *:* users:(("nginx",pid=xxx))
```

**Step 5: If service keeps restarting, check logs**
```bash
journalctl -u nginx -n 20 -o verbose
# Look for the repeated error pattern
```

**Expected findings:**
```bash
$ journalctl -u nginx -b
Jan 19 10:30:25 server nginx[1234]: bind() to 0.0.0.0:80 failed
Jan 19 10:30:25 server nginx[1234]: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
# This means something else is on port 80
```

**Solution for this scenario:**
```bash
# Find what's using port 80
ss -tlnp | grep :80
# Kill that process or change nginx config
sudo ss -tlnp | grep :80
sudo kill -9 <pid of other process>
sudo systemctl restart nginx
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Log Levels:** What's the difference between `-p warn` and `-p "warn..err"`?
   ```
   A) One shows warnings, the other shows errors
   B) One shows warnings, the other shows warnings through errors
   C) They're the same
   D) The second one doesn't exist
   ```

2. **Filtering:** You want to see all errors from nginx in the last 2 hours. Write the command.
   ```
   journalctl ...
   ```

3. **Real-Time Logs:** A user's service keeps crashing. How would you watch it live?
   ```
   A) journalctl -u service -f
   B) tail -f /var/log/service.log
   C) journalctl -u service --since "1 hour ago"
   D) systemctl status service
   ```

4. **Disk Space:** The journal is taking up 5GB. How do you limit it to 1GB?
   ```
   A) journalctl --vacuum-size=1G
   B) rm -rf /var/log/journal/
   C) journalctl --vacuum-time=1d
   D) Clear /var/log/messages
   ```

5. **Real Scenario:** You see this error:
   ```
   Jan 19 10:30:45 server kernel: [12345.678] Out of memory: Kill process nginx (pid 5678) score 150
   ```
   What does this mean and what should you do?
   ```
   (Explain in 2-3 sentences)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "What did the system do 2 hours ago?"
```bash
# Query time range
journalctl --since "2 hours ago" --until "1 hour ago"

# For specific service
journalctl -u nginx --since "2 hours ago"

# Save to file for analysis
journalctl --since "2 hours ago" > /tmp/logs.txt
```

### Scenario 2: "How many errors occurred?"
```bash
# Count by service
journalctl -p err --since today | grep -c "nginx"

# Or
journalctl -p err -u nginx --since today | wc -l

# Get unique error messages
journalctl -u nginx -p err --since today -o short | sort | uniq -c | sort -rn
```

### Scenario 3: "Service is spamming logs"
```bash
# See what it's logging
journalctl -u service -n 20

# Excessive INFO messages?
journalctl -u service --since today -p info | wc -l

# Set log level in service config or:
systemctl set-environment SERVICE_LOGLEVEL=warn
systemctl restart service
```

### Scenario 4: "Storage system collected logs, where are they?"
```bash
# Check if journald is configured for persistent storage
ls -la /var/log/journal/

# If not, check /run (lost on reboot)
ls -la /run/log/journal/

# Export for analysis
journalctl -u lustre --since "1 week ago" -o json > lustre_logs.json
```

### Scenario 5: "Need logs from before last reboot"
```bash
# List available boots
journalctl --list-boots

# View previous boot
journalctl -b -1

# Logs from specific boot
journalctl -b ae3f2a4c6f8d9e1b
```

---

## Key Takeaways

1. **journalctl is your primary tool** — modern systems log to systemd journal
2. **Always check time range** — `-p err --since "1 hour ago"` is a good start
3. **Filter by service** — `-u service_name` narrows down the noise
4. **Follow in real-time** — `-f` is great for watching live issues
5. **Traditional logs still exist** — Some systems log to `/var/log/syslog`
6. **Export for analysis** — Use `-o json` for programmatic access

---

## How Ready Are You?

Can you explain these?
- [ ] What `journalctl -u nginx -p err -f` does
- [ ] How to see logs from a time window
- [ ] What log levels mean
- [ ] The difference between `/run/log/journal/` and `/var/log/journal/`
- [ ] How to find a message from a specific process

If you checked all boxes, you're ready for Week 1 exercises.

---

**Next Step:** Do the Week 1 exercises to practice all three lessons.

---

Powered by UQS v1.8.5-C
