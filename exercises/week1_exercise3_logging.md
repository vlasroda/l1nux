# Week 1 Exercise 3: Logging and Troubleshooting

**Difficulty:** Beginner-Intermediate
**Time:** 20 minutes
**Prerequisites:** Lesson 1.3 (Logging with journald)

---

## Problem 1: Find Recent Errors

**Scenario:** Users report errors on the system in the last hour. You need to find them quickly.

**Task 1A: Get all errors from the last hour**
```bash
# Your command: Show all error-level and above messages
# Hint: journalctl -p err --since "1 hour ago"

# What errors do you see?
```

**Task 1B: Get errors from a specific service**
```bash
# Your command: Show only nginx errors
# Hint: journalctl -u nginx -p err

# Note: nginx might not be running, that's okay
```

**Task 1C: Get errors with timestamps**
```bash
# Your command: Show errors with full timestamps and context
# Hint: journalctl -p err --since "1 hour ago" -o short-full

# Notice the detailed timestamp
```

---

## Problem 2: Monitor a Service in Real-Time

**Scenario:** You've just restarted a service and want to watch for any issues.

**Task 2A: Start following a service**
```bash
# Open a terminal and run:
journalctl -u sshd -f

# Leave it running
```

**Task 2B: In another terminal, trigger some activity**
```bash
# Try connecting (or attempt to, will fail if not configured)
ssh localhost

# Watch the first terminal - you should see log entries
```

**Task 2C: Stop following (in first terminal)**
```bash
# Ctrl+C to stop the -f (follow)
```

---

## Problem 3: Filter by Time Range

**Scenario:** Something broke between 10am and 11am. You need to review all logs from that period.

**Task 3A: Get logs from today between 10:00 and 11:00**
```bash
# Your command:
# journalctl --since "2024-01-19 10:00:00" --until "2024-01-19 11:00:00"
# (Replace the date with today's date)

# Count how many entries
journalctl --since "10:00:00" --until "11:00:00" | wc -l
```

**Task 3B: Get logs from this boot only**
```bash
# Your command: Show logs since current boot
# Hint: journalctl -b

# This is useful because reboots add a lot of noise
```

**Task 3C: Get logs from before the last reboot**
```bash
# Show previous boot logs
journalctl -b -1

# Note: May be empty if only one boot is in journal
```

---

## Problem 4: Parse and Analyze Logs

**Scenario:** You need to find out how many times a specific error occurred.

**Task 4A: Count occurrences of a message**
```bash
# Find how many times "error" appears in recent logs
journalctl | grep -i "error" | wc -l

# Or combine with service filter
journalctl -u nginx | grep -i "error" | wc -l
```

**Task 4B: Get unique error messages**
```bash
# Find what different errors occurred
journalctl -p err --since today | grep -oE '\[.*\]' | sort | uniq -c | sort -rn

# Or simpler:
journalctl -p err --since today -o short | awk '{$1=$2=$3=""; print}' | sort | uniq -c | sort -rn
```

**Task 4C: Export logs for analysis**
```bash
# Save to a file
journalctl --since today > /tmp/today_logs.txt

# Save in JSON format
journalctl --since today -o json > /tmp/today_logs.json

# View file size
ls -lh /tmp/today_logs.*
```

---

## Problem 5: Configure and Check Journal Settings

**Scenario:** You need to understand how much space the journal is using and make sure it's being persisted.

**Task 5A: Check if journal is persistent**
```bash
# Check both locations
ls -la /var/log/journal/
ls -la /run/log/journal/

# /var/log/journal = persistent (survives reboot)
# /run/log/journal = volatile (lost on reboot)

# If both are empty, journal might be disabled
```

**Task 5B: Check journal size**
```bash
# How much disk space?
journalctl --disk-usage

# Example output: "Journals take up 245.4M"
```

**Task 5C: View journal statistics**
```bash
# How many entries per boot?
journalctl --list-boots

# How many entries in current journal?
journalctl -n 0 | tail -1
```

---

## Problem 6: Troubleshoot a Stuck Service

**Scenario:** A service keeps failing to start. Use logs to diagnose.

**Task 6A: Check service status**
```bash
# Try a service that might have issues (or a made-up one)
systemctl status cups  # Or another service

# Is it active? Failed? Inactive?
```

**Task 6B: Get recent logs for that service**
```bash
# Your command: Show last 20 entries for the service
# journalctl -u service_name -n 20

# Look for error messages
```

**Task 6C: See the full error**
```bash
# Your command: Show errors in verbose format
# journalctl -u service_name -p err -o verbose

# Check the MESSAGE field for full details
```

**Task 6D: Follow the service during restart**
```bash
# Terminal 1: Follow logs
journalctl -u service_name -f

# Terminal 2: Restart service
sudo systemctl restart service_name

# Watch what happens in Terminal 1
```

---

## Challenge: Create a Monitoring Dashboard

**Create a script that reports system health:**

```bash
#!/bin/bash

echo "=== System Health Check ==="
echo ""

echo "Recent Errors (last hour):"
journalctl -p err --since "1 hour ago" -n 5 -o short
echo ""

echo "Critical Issues:"
journalctl -p crit --since "24 hours ago" -o short
echo ""

echo "Failed Services:"
systemctl list-units --failed
echo ""

echo "Journal Size:"
journalctl --disk-usage
echo ""

echo "Most Active Service (last hour):"
journalctl --since "1 hour ago" -o json | jq -r '._SYSTEMD_UNIT' | sort | uniq -c | sort -rn | head -5
```

**Your task:**
1. Save this as `health_check.sh`
2. Make it executable
3. Run it
4. Understand each section
5. Modify to show what YOU want to monitor

---

## Self-Check

After completing all problems:

- [ ] Problem 1: Found errors from specific time ranges
- [ ] Problem 2: Monitored a service live with -f
- [ ] Problem 3: Filtered logs by time period
- [ ] Problem 4: Parsed and counted error messages
- [ ] Problem 5: Checked journal size and persistence
- [ ] Problem 6: Diagnosed a service startup issue
- [ ] Challenge: Created a health check script

---

## Key Skills You Practiced

- `journalctl -p err` — Find errors
- `journalctl -u service -f` — Follow service in real-time
- `journalctl --since/--until` — Time-based filtering
- `journalctl -o json` — Export for analysis
- `--disk-usage`, `--list-boots` — Journal management
- Combining journalctl with grep/awk for analysis

---

## Common Real Scenarios

**"System crashed at 3am"**
```bash
journalctl --since "2024-01-19 03:00:00" --until "2024-01-19 04:00:00" -p err
```

**"nginx keeps restarting"**
```bash
journalctl -u nginx -f
# Watch it restart and see errors
```

**"What changed yesterday?"**
```bash
journalctl --since "1 day ago" --until "2 days ago" -o verbose
```

---

## Next Steps

Compare your answers with `week1_solutions.md`.

Make sure you understand:
1. When to use journalctl vs grep
2. How to combine filters (-u, -p, --since)
3. How to export logs for external analysis

---

Powered by UQS v1.8.5-C
