# Week 1 Exercise 2: Process Management

**Difficulty:** Beginner-Intermediate
**Time:** 20 minutes
**Prerequisites:** Lesson 1.2 (Process Management)

---

## Problem 1: Find and Monitor a Process

**Scenario:** You're told there's a "slow_worker" process running. Users say the system is sluggish. You need to find it and see what it's doing.

**Task 1A: Find the process**
```bash
# Create a test process in the background
sleep 3600 &
sleep 3600 &

# Your command: Find all sleep processes
# Hint: use ps, pgrep, or ps aux | grep
```

**Expected output:**
```bash
# You should see both sleep processes with their PIDs
PID    COMMAND
1234   sleep 3600
5678   sleep 3600
```

**Task 1B: Show them in a tree (parent-child relationship)**
```bash
# Your command: Show process hierarchy
# Hint: Use pstree
```

**Task 1C: Get detailed info for one process**
```bash
# Get PID of one sleep
MYPID=$(pgrep sleep | head -1)

# Your command: Show memory and CPU usage for this PID
# Hint: ps -p $MYPID with specific columns
```

**Expected output:**
```bash
PID    %CPU %MEM VSZ    RSS    COMMAND
1234   0.0  0.1  2308   360    sleep
```

---

## Problem 2: Graceful vs Forceful Shutdown

**Scenario:** You have a stuck process. First try to stop it gracefully, then force kill if needed.

**Setup:**
```bash
# Create a background process
bash -c 'while true; do echo "working"; sleep 1; done' &
PID=$!
```

**Task 2A: Try graceful shutdown**
```bash
# Send SIGTERM to the process
kill -15 $PID

# Wait 2 seconds
sleep 2

# Check if it's still running
ps -p $PID

# What do you see?
```

**Task 2B: Force kill if needed**
```bash
# If ps shows it's still there, force kill
kill -9 $PID

# Verify it's gone
ps -p $PID
# Should return no output
```

**What you learned:**
- SIGTERM (-15) gives the process time to cleanup
- SIGKILL (-9) kills immediately, might leave mess

---

## Problem 3: Monitor CPU and Memory Hogs

**Scenario:** The system is slow. Find what's using all the resources.

**Task 3A: Create a resource-heavy process**
```bash
# This uses a lot of CPU
yes > /dev/null &
PID1=$!

# And another one
dd if=/dev/zero bs=1M count=100 of=/dev/null &
PID2=$!
```

**Task 3B: Find the top CPU consumer**
```bash
# Your command: Show top 5 processes by CPU usage
# Hint: Use top, htop, or ps with --sort
```

**Expected output:**
```bash
PID   %CPU  COMMAND
9999  99.5  yes
8888  50.2  dd
...
```

**Task 3C: Find the top memory consumer**
```bash
# Your command: Show top 5 processes by memory usage
# Hint: ps -eo with --sort -rss
```

**Task 3D: Kill them**
```bash
# Kill both processes
kill -9 $PID1 $PID2

# Verify
ps -p $PID1 $PID2
# Should be gone
```

---

## Problem 4: Service Restart Scenario

**Scenario:** The `systemd-resolved` service (DNS) is having issues. You need to restart it.

**Task 4A: Check service status**
```bash
systemctl status systemd-resolved

# What state is it in? (active, inactive, failed?)
```

**Task 4B: View recent logs**
```bash
# Your command: Show the last 10 logs for this service
# Hint: journalctl -u service
```

**Task 4C: Restart it**
```bash
sudo systemctl restart systemd-resolved

# Verify it restarted
systemctl status systemd-resolved
# Should say "active (running)"
```

**Task 4D: Follow logs in real-time**
```bash
# Open another terminal
journalctl -u systemd-resolved -f

# See new entries as they happen
```

---

## Problem 5: Zombie Cleanup

**Scenario:** A parent process exited without collecting its zombie child. You need to find and clean it up.

**Task 5A: Create a zombie (advanced)**
```bash
# Start a script that exits but leaves a child
bash -c 'sleep 3600 & exit 0' &
sleep 1

# Look for zombies
ps aux | grep defunct
# or
ps aux | grep -E '\<Z\>'

# You should see a sleep process marked as defunct
```

**Task 5B: Find the parent**
```bash
# Get the PID of the zombie
ZPID=$(ps aux | grep defunct | grep -v grep | awk '{print $2}')

# Show its parent
ps -p $ZPID -o ppid=

# What's the parent PID?
```

**Task 5C: Understand the cause**
```bash
# The parent (usually the shell) needs to collect it
# Wait for the parent to exit or:
kill -9 $(ps -p $ZPID -o ppid=)  # Kill the parent
# Zombie should disappear
```

---

## Challenge: Build a Process Monitor Script

**Create a simple monitoring script that:**

1. Finds all nginx processes
2. Shows their memory usage
3. Shows their open file count
4. Reports if any are in state 'D' (uninterruptible)

**Script outline:**
```bash
#!/bin/bash

# Find all nginx processes
PIDS=$(pgrep nginx)

for PID in $PIDS; do
    # Get memory usage
    MEM=$(ps -p $PID -o rss= | awk '{print $1}')

    # Get state
    STATE=$(ps -p $PID -o stat= | cut -c1)

    # Get open files
    OPENFILES=$(lsof -p $PID 2>/dev/null | wc -l)

    # Report
    echo "PID: $PID  MEM: ${MEM}KB  STATE: $STATE  OPENFILES: $OPENFILES"

    # Alert if in D state
    if [[ "$STATE" == "D" ]]; then
        echo "WARNING: PID $PID is in uninterruptible state (stuck I/O)"
    fi
done
```

**Your task:**
1. Write the script
2. Make it executable
3. Run it on your system
4. See what processes it finds

---

## Self-Check

After completing all problems:

- [ ] Problem 1: Found processes and showed tree view
- [ ] Problem 2: Used SIGTERM and SIGKILL correctly
- [ ] Problem 3: Identified resource hogs by CPU and memory
- [ ] Problem 4: Restarted a service and watched logs
- [ ] Problem 5: Found and explained zombie processes
- [ ] Challenge: Created a monitoring script

---

## Key Skills You Practiced

- `ps`, `pgrep`, `pstree` — Finding processes
- `kill` — Sending signals to processes
- `journalctl -u service` — Viewing service logs
- `top`, `ps --sort` — Finding resource hogs
- Process states and zombie cleanup

---

## Next Steps

Compare your answers with `week1_solutions.md`.

If you got stuck:
1. Review the lesson
2. Try the command with `--help`
3. Check the solutions
4. Understand WHY it works

---

Powered by UQS v1.8.5-C
