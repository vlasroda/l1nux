# Lesson 1.2: Process Management (Refresher)

**Time:** 20-30 minutes
**Why it matters:** 50% of L1 tickets are "process won't start" or "process is hung"
**Skills:** View processes, understand signals, kill hung processes, debug process issues

---

## Quick Review

### Process Basics
Every process has:
- **PID** (Process ID) — unique identifier
- **PPID** (Parent PID) — who started this process
- **User/UID** — which user owns it
- **State** — running, sleeping, zombie, etc.
- **Priority/Nice** — CPU scheduling priority

### States You'll See
- **R:** Running (actively using CPU)
- **S:** Sleeping (waiting for something)
- **Z:** Zombie (dead but waiting for parent to collect it)
- **T:** Stopped (paused, usually by signal)
- **D:** Uninterruptible (stuck, usually in I/O)

---

## Key Concepts

### 1. Viewing Processes

```bash
# Quick overview (simple)
ps aux                          # All processes with CPU%, memory%

# Details for one process
ps -ef | grep myprocess        # Find by name
ps -p 1234 -o pid,user,cmd     # Specific format for PID 1234

# Process tree (parent-child relationships)
pstree                         # Tree view of all processes
pstree -p 1234                 # Tree from PID 1234 down

# Real-time monitoring
top                            # Interactive, sorted by CPU
htop                           # Better version of top (if available)
```

### 2. Process Signals

Signals are how you communicate with processes:

| Signal | Name | Effect | Use Case |
|--------|------|--------|----------|
| 0 | SIGCONT | Continue | Resume stopped process |
| 1 | SIGHUP | Hangup | Reload config (if supported) |
| 9 | SIGKILL | Kill | **Force kill** (can't be caught) |
| 15 | SIGTERM | Terminate | Graceful shutdown (default) |
| 19 | SIGSTOP | Stop | Pause process |
| 20 | SIGTSTP | Stop | Pause process (Ctrl+Z) |

**Key difference:**
- **SIGTERM (15):** "Please stop" — process can ignore or cleanup
- **SIGKILL (9):** "Stop now" — instant, can't be caught, no cleanup

**Rule:** Always try SIGTERM first, only use SIGKILL if needed.

### 3. Sending Signals

```bash
# By PID
kill -15 1234               # SIGTERM (polite)
kill -9 1234                # SIGKILL (force)

# By process name
killall nginx                # Kill all 'nginx' processes
pkill -f "java.*app.jar"     # Kill matching pattern

# Send HUP (reload config)
kill -1 1234                 # SIGHUP

# Stop/resume
kill -19 1234                # SIGSTOP (pause)
kill -18 1234                # SIGCONT (resume)
```

### 4. Background & Foreground

```bash
# Start in background
command &                   # Starts, returns prompt

# Suspend current (Ctrl+Z, then)
bg                          # Resume in background
fg                          # Bring to foreground

# List background jobs
jobs                        # Show job numbers
jobs -l                     # With PIDs

# Resume specific job
fg %2                       # Resume job #2
```

### 5. Process Resources

```bash
# What files/sockets is a process using?
lsof -p 1234                # Open files for PID
lsof -c nginx               # Open files for command

# System calls a process is making
strace -p 1234              # System calls (live)
strace -e trace=open command  # Trace specific syscall type

# Environment variables
cat /proc/1234/environ | tr '\0' '\n'   # What env vars does it have?

# Command line that started it
cat /proc/1234/cmdline | tr '\0' ' '    # Original command
ps -p 1234 -o cmd=          # Or simpler command
```

---

## Essential Commands

### View Running Processes

```bash
# All processes (easy to read)
ps aux

# Find specific process
ps aux | grep postgres
ps aux | grep -v grep | grep postgres  # Exclude grep itself

# Show only certain columns
ps -eo pid,user,rss,vsz,cmd --sort -rss  # Top memory users
ps -eo pid,user,%cpu,cmd --sort -%cpu     # Top CPU users

# Process hierarchy
pstree -p
```

### Monitor in Real-Time

```bash
# Basic
top -b -n 1                 # One snapshot
top                         # Interactive

# Better
htop                        # If available
watch 'ps aux | head -20'   # Refresh every 2 sec
```

### Manage Processes

```bash
# Gracefully stop a process
kill -15 <pid>              # SIGTERM

# Force kill
kill -9 <pid>               # SIGKILL

# Kill by name
killall httpd
pkill -f "python.*worker"

# Pause/resume
kill -STOP <pid>            # Pause
kill -CONT <pid>            # Resume

# Restart with HUP
kill -HUP <pid>             # Reload config (if supported)
```

### Understand Process Issues

```bash
# Why isn't a service running?
systemctl status nginx          # Check service
systemctl start nginx           # Start it
journalctl -u nginx -n 50       # Recent logs

# What's this process doing?
strace -p <pid>                 # System calls
ltrace -p <pid>                 # Library calls
lsof -p <pid>                   # Open files/sockets
```

---

## Interactive Exercise: Manage a Runaway Process

**Scenario:** Your app is misbehaving. You need to:

1. Find the process
2. Understand what it's doing
3. Stop it gracefully
4. Force kill if needed

**Task 1: Create a test process**
```bash
# In background, run a long command
sleep 3600 &
sleep 3600 &
sleep 3600 &

# You now have 3 sleeping processes
```

**Task 2: Find them**
```bash
# Show all sleep processes
ps aux | grep sleep

# Get just the PIDs
pgrep sleep

# Show them in a tree
pstree | grep sleep
```

**Task 3: Gracefully stop one**
```bash
# Get one PID
MYPID=$(pgrep sleep | head -1)

# Try graceful shutdown
kill -15 $MYPID

# Did it work?
ps -p $MYPID
```

**Task 4: Force kill if needed**
```bash
# If it's still running, force it
kill -9 $MYPID

# Verify it's gone
ps -p $MYPID        # Should show no output
```

**Expected result:**
```
$ ps aux | grep sleep
[should show fewer sleep processes]

$ pgrep sleep
[fewer PIDs or empty]
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Signals:** When would you use `kill -1` instead of `kill -15`?
   ```
   A) When you want to reload config files
   B) When the process ignores SIGTERM
   C) When the process is a zombie
   D) Never, SIGKILL is always better
   ```

2. **Process State:** What does the 'D' state in `ps` output mean, and why is it important?
   ```
   (Explain in 2-3 sentences)
   ```

3. **Zombies:** A process shows as `<defunct>` in `ps` output. What causes this and how do you clean it up?
   ```
   (Explain the cause and solution)
   ```

4. **Background Jobs:** You're running a command that takes 10 minutes. You realize you need the terminal. What do you do?
   ```
   A) Kill the process and restart
   B) Ctrl+Z then type bg
   C) Wait 10 minutes
   D) Open another terminal
   ```

5. **Real Scenario:** A database process is using 50GB RAM and the system is swapping badly. It won't stop with `kill -15`. What's the right approach?
   ```
   Explain your steps.
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Application won't start"
```bash
# Check if it's already running
pgrep -f myapp

# Look at logs
journalctl -u myapp -n 50

# Try starting manually to see error
/opt/myapp/bin/myapp    # Look at error output

# Check permissions
ls -l /opt/myapp/
ls -l /var/log/myapp/

# Check open file limits
ulimit -n
```

### Scenario 2: "Process takes all the CPU"
```bash
# Find it
top -b -n 1 | head -20      # Or htop

# Get PID and check what it's doing
strace -p <pid> -c          # Summary of system calls

# Is it a busy loop?
strace -p <pid> 2>&1 | head -50  # First 50 calls

# Kill if needed
kill -9 <pid>
```

### Scenario 3: "Process is hung, won't respond"
```bash
# Check state
ps -p <pid> -o pid,stat,cmd

# If state is 'D', it's stuck in I/O
# Check what file it's accessing
lsof -p <pid>

# No choice but to wait or reboot
# (Can't kill D state processes)
```

### Scenario 4: "Service keeps restarting"
```bash
# Check systemd status
systemctl status myservice

# Look at recent logs
journalctl -u myservice -n 100 -f

# See if core dumps exist
ls -la /var/lib/systemd/coredump/

# Disable auto-restart temporarily
systemctl stop myservice
systemctl disable myservice
# Fix the problem
systemctl enable myservice
systemctl start myservice
```

---

## Key Takeaways

1. **Always try SIGTERM first** — gives process time to cleanup
2. **Use SIGKILL only when necessary** — it's forceful and can leave mess
3. **Check the state** — D state means stuck in I/O, can't be killed
4. **Understand parent-child** — killing parent might orphan children
5. **Check permissions** — process might fail due to file access issues
6. **Monitor carefully** — use top/htop to understand system behavior

---

## How Ready Are You?

Can you explain these?
- [ ] The difference between SIGTERM and SIGKILL
- [ ] Why a zombie process exists
- [ ] What the 'D' state means and why it matters
- [ ] How to find what files a process has open
- [ ] When to use kill vs killall vs pkill

If you can answer all of these, you're ready for the next lesson.

---

Powered by UQS v1.8.5-C
