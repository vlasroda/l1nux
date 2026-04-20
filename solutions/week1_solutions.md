# Week 1 Solutions: System Administration Refresher

---

## Lesson 1.1: Users and Permissions

### Exercise Solutions

#### Task 1: Why can't alice read /var/log/app.log?

**Diagnosis:**
```bash
$ ls -l /var/log/app.log
-rw------- 1 root root 5000 Jan 19 /var/log/app.log

$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice)
```

**Answer:** alice can't read because:
- File owner is `root` (alice is not owner)
- File group is `root` (alice is not in root group)
- "Others" have no permissions (`---`)
- Permission bits `600` means only owner (root) can read/write

**Solution:**
```bash
# Option 1: Make it readable by others
sudo chmod 644 /var/log/app.log

# Option 2: Add alice to root group (not recommended)
# Better: add a specific group
sudo chown root:applog /var/log/app.log
sudo usermod -a -G applog alice
sudo chmod 640 /var/log/app.log
```

**Best practice:** Use a dedicated group (applog) rather than root for shared access.

---

#### Task 2: Set up /srv/data/upload so bob can write

**Current state:**
```bash
$ ls -ld /srv/data/
drwxr-xr-x 2 root root 4096 Jan 19 /srv/data/
```

**Analysis:** bob can't write because:
- Owner is root (bob is not owner)
- Group is root (bob is probably not in root group)
- Others have no write permission
- Permission bits `755` means only owner can write

**Solution:**
```bash
# Find what group bob is in
id bob
# Example: uid=1001(bob) gid=1001(bob) groups=1001(bob)

# Make it writable by bob's group
sudo chown root:bob /srv/data/upload
sudo chmod 775 /srv/data/upload

# Now bob can write
# New files will be owned by bob:bob (regular behavior)
```

**Alternative (using SGID for shared group ownership):**
```bash
# SGID means files inherit directory's group
sudo chmod 2775 /srv/data/upload

# Now files created by bob are owned by bob:bob
# But parent directory group applies to files created in it
```

**Result:**
```bash
$ ls -ld /srv/data/upload
drwxrwxr-x 2 root bob 4096 Jan 19 /srv/data/upload
# bob can now write (group has write permission)
```

---

#### Task 3: Restrict secret.txt to only alice

**Goal:** Only alice can read, no one else

**Solution:**
```bash
# Create the file (assuming running as alice)
touch /home/shared/secret.txt

# Remove all group and other permissions
chmod 600 /home/shared/secret.txt

# Verify
ls -l /home/shared/secret.txt
-rw------- 1 alice alice 0 Jan 19 /home/shared/secret.txt
```

**Explanation:**
- `6` (rw-) for owner: alice can read and write
- `0` (---) for group: no one in alice's group can access
- `0` (---) for others: no one else can access

**Why not 400 (r--):** If alice needs to modify the file, she needs write permission too.

---

### Challenge Questions Answers

**1. What does `chmod 700` do?**
   ```
   Answer: A) Locks everyone out except owner

   Explanation:
   700 = 7 (owner) 0 (group) 0 (others)
   = rwx --- ---
   Only owner has any permission (all three: read, write, execute)
   Group and others have nothing
   ```

**2. Why does `/usr/bin/sudo` need the SUID bit?**
   ```
   Answer: So sudo can run as root even when called by a regular user.

   Explanation: When you run /usr/bin/sudo (which has SUID), it executes
   with the owner's UID (root) rather than your UID. This lets you perform
   actions that require root privileges. Without SUID, it would run as your
   user and couldn't escalate privileges.
   ```

**3. Why can't you `cd` into a directory with `744` permissions?**
   ```
   Answer: The directory is missing execute permission for "others".

   Explanation:
   744 = rwx r-- r--
   Owner: can read, write, execute (enter)
   Group: can only read (can't enter without execute)
   Others: can only read (can't enter without execute)

   To enter a directory, you need execute permission (even if you can read it).
   Fix: chmod 755 (add execute for group/others)
   ```

**4. SGID bit on file creation:**
   ```
   Answer: B) The directory's group

   Explanation: When SGID is set on a directory, any new files created
   in that directory are owned by the directory's group, not the creator's
   primary group. This ensures shared directories maintain consistent
   group ownership.
   ```

**5. Can't delete file from /tmp:**
   ```
   Answer: C) The sticky bit prevents it

   Explanation: /tmp has sticky bit (t), which means only the file owner
   can delete their own files, even if others have write permission on
   /tmp itself. This prevents users from deleting each other's files.

   /tmp permissions: drwxrwxrwt
   The 't' in others position is the sticky bit.
   ```

---

## Lesson 1.2: Process Management

### Exercise Solutions

#### Task 1-4: Manage a Runaway Process

**Setup:**
```bash
# Create background processes
$ sleep 3600 &
[1] 12345
$ sleep 3600 &
[2] 12346
$ sleep 3600 &
[3] 12347
```

**Task 2 Solution (Find processes):**
```bash
# All sleep processes
$ ps aux | grep sleep
alice    12345  0.0  0.1   5688  372 pts/0 S 10:30 0:00 sleep 3600
alice    12346  0.0  0.1   5688  372 pts/0 S 10:31 0:00 sleep 3600
alice    12347  0.0  0.1   5688  372 pts/0 S 10:32 0:00 sleep 3600

# Just the PIDs
$ pgrep sleep
12345
12346
12347

# In a tree
$ pstree -p | grep -A 3 sleep
```

**Task 3 Solution (Graceful stop):**
```bash
# Get one PID
$ MYPID=$(pgrep sleep | head -1)
$ echo $MYPID
12345

# Send SIGTERM (graceful)
$ kill -15 $MYPID

# Check if it stopped (should show no process)
$ ps -p 12345
  PID TTY      STAT   TIME COMMAND
12345 pts/0    S      0:00 sleep 3600
# Still running? SIGTERM was ignored or needs more time

# Give it a moment
$ sleep 1
$ ps -p 12345
# Now it should be gone
```

**Task 4 Solution (Force kill):**
```bash
$ kill -9 12346

# Verify
$ ps -p 12346
# Should show nothing (process terminated)

# Check remaining
$ pgrep sleep
12347
# Only one left
```

**Expected final state:**
```bash
$ jobs
[3]   Running                 sleep 3600 &

# Clean up
$ kill -9 %3
```

---

### Challenge Questions Answers

**1. When would you use `kill -1` instead of `kill -15`?**
   ```
   Answer: A) When you want to reload config files

   Explanation:
   kill -1 = SIGHUP (hangup signal)
   kill -15 = SIGTERM (terminate)

   Many services (nginx, apache, rsyslog) are configured to reload their
   config files when they receive SIGHUP. This is gentler than stopping
   and restarting the service.

   Example: kill -1 <nginx_pid> reloads nginx config without dropping connections
   ```

**2. What does 'D' state mean?**
   ```
   Answer: Uninterruptible sleep (usually I/O)

   Explanation:
   - The process is stuck waiting for I/O (disk read/write, network)
   - It cannot be interrupted by signals (even SIGKILL)
   - Usually means the process is waiting for slow storage or NFS
   - You can't kill it cleanly; only solution is reboot

   Why important: You might see 'D' state as a symptom of:
   - NFS mount hanging
   - Broken disk
   - Network issue
   - Kernel bug

   Solution: Fix the underlying I/O problem, then process will continue
   ```

**3. What causes zombie processes?**
   ```
   Answer: Parent process hasn't read the child's exit status

   Explanation:
   - When a child process exits, it sends SIGCHLD to parent
   - Parent must "reap" it by calling wait()/waitpid()
   - If parent doesn't reap, child becomes zombie
   - Zombie takes up a PID slot but no resources

   Example:
   Parent forks child
   Child finishes
   Parent doesn't call wait()
   Child becomes <defunct> (zombie)

   Solution:
   - Fix the parent code to call wait()
   - Or kill the parent (then init reaps zombies)
   - You can't kill a zombie directly
   ```

**4. If you need the terminal:**
   ```
   Answer: B) Ctrl+Z then type bg

   Explanation:
   Ctrl+Z = SIGTSTP (terminal stop signal) - pauses the job
   bg = Resume in background (you get the terminal back)

   Then:
   $ long_running_command
   <do work> Ctrl+Z
   [1]+ Stopped    long_running_command
   $ bg
   [1]+ long_running_command &
   $ # You're back at the prompt

   Later you can:
   fg %1   # Bring it back to foreground if needed
   ```

**5. Process using 50GB RAM, won't stop with kill -15:**
   ```
   Answer: Follow these steps:

   1. Confirm it's really hung
      $ ps -p <pid> -o pid,stat,rss,vsz,cmd

   2. Try SIGKILL (force kill)
      $ kill -9 <pid>

   3. Verify it's gone
      $ ps -p <pid>

   4. If still not gone (very rare), check state
      $ ps -p <pid> -o stat=
      If state is 'D', stuck in I/O, need system intervention

   5. If nothing works
      $ ps -p <pid>
      If you can't kill it and it's in D state:
      - File/mount might be stuck
      - Might need reboot
      - Check dmesg for kernel warnings

   Prevention:
   - Set ulimit to prevent runaway memory
   - Use cgroups to limit process resources
   - Monitor memory and restart before hitting limits
   ```

---

## Real Scenario Practice

## Week 1 Hands-On Exercises

### Exercise 1: Permission Troubleshooting

#### Problem 1 Solution: Web App Write Access

**Setup creates:**
```bash
drwxr-xr-x 2 root root 4096 /srv/web
```

**Solution:**
```bash
# Create logs directory
sudo mkdir -p /srv/web/logs

# Method 1: Add www-data to root group (not recommended)
# Method 2: Create a dedicated group
sudo groupadd -f www-data  # Usually exists
sudo chown root:www-data /srv/web/logs
sudo chmod 775 /srv/web/logs

# Test
sudo -u www-data touch /srv/web/logs/test.log
# Should succeed
```

**Result:**
```bash
$ ls -ld /srv/web/logs
drwxrwx--- 2 root www-data 4096 Jan 19 /srv/web/logs
# www-data can now read, write, execute
```

---

#### Problem 2 Solution: Shared Team Directory

**Current:**
```bash
drwxr-xr-x 2 root root 4096 /srv/team
```

**Solution:**
```bash
# Change group to webdev (contains alice and bob)
sudo chown root:webdev /srv/team

# Set SGID so new files inherit group
sudo chmod 2770 /srv/team

# Result: drwxrws--- 2 root webdev
#         Notice 's' in group execute position (SGID)

# Test as alice
sudo -u alice touch /srv/team/file1.txt
ls -l /srv/team/file1.txt
# -rw-r----- 1 alice webdev
# Group is webdev (inherited from directory)

# Test bob can write
sudo -u bob bash -c 'echo "hello" >> /srv/team/file1.txt'
# Should succeed because bob is in webdev group
```

**Why this works:**
- `2` = SGID bit (files inherit directory's group)
- `770` = owner/group can read/write/execute, others cannot
- New files created by alice are owned by alice:webdev (not alice:alice)
- bob is in webdev, so he can modify files

---

#### Problem 3 Solution: Script Execution

**Setup:**
```bash
sudo tee /opt/backup.sh > /dev/null << 'EOF'
#!/bin/bash
echo "Backup running at $(date)"
EOF
```

**Solution:**
```bash
# Set permissions
sudo chmod 750 /opt/backup.sh

# Verify
ls -l /opt/backup.sh
# -rwxr-x--- 1 root root
# Owner can execute, group can execute, others cannot

# Alternative acceptable answers:
# chmod 755 /opt/backup.sh  (others can execute, usually OK for scripts)
# chmod 744 /opt/backup.sh  (others can read but not execute)
```

**Explanation:**
```
750 = rwxr-x---
Owner:  rwx (read, write, execute - full control)
Group:  r-x (read, execute - can run it but not modify)
Others: --- (no access)

755 = rwxr-xr-x
Owner:  rwx
Group:  r-x
Others: r-x (anyone can run it - OK for public scripts)
```

---

#### Problem 4 Solution: Log File Security

**Goal:** Only root and admin group can read

**Current:**
```bash
-rw-r--r-- 1 root root /var/log/secure_app.log
```

**Solution:**
```bash
# Create test file
sudo touch /var/log/secure_app.log

# Create admin group if needed
sudo groupadd -f admin

# Change group to admin
sudo chown root:admin /var/log/secure_app.log

# Remove other permissions
sudo chmod 640 /var/log/secure_app.log

# Verify
ls -l /var/log/secure_app.log
# -rw-r----- 1 root admin

# Add users to admin group to let them read
sudo usermod -a -G admin alice
```

**Result:**
```
640 = rw-r-----
Owner:  rw- (root can read/write)
Group:  r-- (admin group can read)
Others: --- (no access)
```

---

#### Challenge Solution: Complex Setup

**Goal:** alice owns, webdev group inherits, others blocked

**Solution:**
```bash
sudo mkdir -p /srv/projects
sudo chown alice:webdev /srv/projects
sudo chmod 2755 /srv/projects

# Verify
ls -ld /srv/projects
# drwxr-sr-x 2 alice webdev
# Notice 's' in group position (SGID)

# Test
sudo -u alice bash -c 'echo "project" > /srv/projects/test.txt'
ls -l /srv/projects/test.txt
# -rw-r----- 1 alice webdev
# File is owned by alice but group is webdev (inherited)
```

**Why `2755`:**
- `2` = SGID bit (new files inherit group)
- `7` (owner) = rwx (alice full control)
- `5` (group) = r-x (webdev can read and enter)
- `5` (others) = r-x (everyone can read and enter, but can't write)

Wait, that's too open. Better:

```bash
sudo chmod 2750 /srv/projects
# Result: drwxr-s--- 2 alice webdev
# Others have no access
```

**Revised:**
- `2750` = SGID + rwxr-x---
- Owner: rwx (alice full)
- Group: r-x (webdev can read/enter)
- Others: --- (blocked)

---

### Exercise 2: Process Management

#### Problem 1 Solution: Find and Monitor

**Task 1A:**
```bash
# Create test processes
$ sleep 3600 &
[1] 12345
$ sleep 3600 &
[2] 12346

# Find them
$ ps aux | grep sleep
alice    12345  0.0  0.1  2308   360 pts/0 S 10:30 0:00 sleep 3600
alice    12346  0.0  0.1  2308   360 pts/0 S 10:31 0:00 sleep 3600

# Or just the PIDs
$ pgrep sleep
12345
12346
```

**Task 1B:**
```bash
# Process tree
$ pstree -p | grep -B2 sleep
bash(1000)─┬─sleep(12345)
           └─sleep(12346)
```

**Task 1C:**
```bash
$ MYPID=$(pgrep sleep | head -1)
$ ps -p $MYPID -o pid,user,%cpu,%mem,rss,vsz,cmd
  PID USER     %CPU %MEM   RSS   VSZ CMD
12345 alice     0.0  0.1   360  2308 sleep 3600
```

---

#### Problem 2 Solution: Graceful vs Forceful

**Setup:**
```bash
$ bash -c 'while true; do echo "working"; sleep 1; done' &
[1] 12350
```

**Task 2A:**
```bash
$ kill -15 12350

# Check status
$ ps -p 12350
# If using sleep in a loop, SIGTERM terminates the bash script
# If a long-running process, it may ignore SIGTERM
```

**Task 2B:**
```bash
$ kill -9 12350

$ ps -p 12350
# No output - process is gone
```

---

#### Problem 3 Solution: Find Resource Hogs

**Task 3A:**
```bash
$ yes > /dev/null &
[1] 12360

$ dd if=/dev/zero bs=1M count=100 of=/dev/null &
[2] 12361
```

**Task 3B:**
```bash
$ ps eo pid,%cpu,cmd --sort -%cpu | head -6
  PID %CPU CMD
12360 99.8 yes
12361 45.2 dd

# Or using top
$ top -b -n 1 | head -12
```

**Task 3C:**
```bash
$ ps eo pid,%mem,rss,cmd --sort -rss | head -6
  PID %MEM   RSS CMD
12361  2.1 102400 dd
12360  0.1   360 yes
```

**Task 3D:**
```bash
$ kill -9 12360 12361

$ ps -p 12360 12361
# No output
```

---

#### Problem 4 Solution: Service Restart

**Task 4A:**
```bash
$ systemctl status systemd-resolved
● systemd-resolved.service - systemd DNS/LLMNR resolver
   Loaded: loaded (/lib/systemd/system/systemd-resolved.service)
   Active: active (running) since Jan 19 10:00:00 UTC
```

**Task 4B:**
```bash
$ journalctl -u systemd-resolved -n 10
Jan 19 10:00:01 host systemd[1]: Starting systemd DNS resolver...
Jan 19 10:00:01 host systemd-resolved[1234]: Listening on 127.0.0.53:53
```

**Task 4C:**
```bash
$ sudo systemctl restart systemd-resolved

$ systemctl status systemd-resolved
● systemd-resolved.service
   Active: active (running) since Jan 19 11:00:00 UTC
```

**Task 4D:**
```bash
# Terminal 1
$ journalctl -u systemd-resolved -f
Jan 19 11:00:00 host systemd[1]: Restarting...
Jan 19 11:00:01 host systemd-resolved[5678]: Listening...

# Watch the restart happen in real-time
```

---

#### Problem 5 Solution: Zombie Cleanup

**Task 5A:**
```bash
$ bash -c 'sleep 3600 & exit 0' &

$ ps aux | grep defunct
alice    12370  0.1  0.0   2308    0 pts/0 Z 11:00 0:00 [sleep] <defunct>
# 'Z' = zombie state
# VSZ=0, RSS=0 = takes no memory, just a PID slot
```

**Task 5B:**
```bash
$ ZPID=12371  # The sleep process

$ ps -p 12371 -o ppid=
12370      # Parent is the bash process

$ ps -p 12370
  PID TTY      STAT   TIME COMMAND
12370 pts/0    S      0:00 bash

# The bash exited but init hasn't reaped yet
```

**Task 5C:**
```bash
# Zombie will disappear when parent exits or init reaps it
# Usually happens after a few seconds
$ sleep 2 && ps -p 12371
# Should be gone

# Or force parent exit
$ kill -9 12370
# Zombie disappears immediately
```

---

#### Challenge: Process Monitor Script

**Solution:**
```bash
#!/bin/bash

PIDS=$(pgrep nginx)

if [ -z "$PIDS" ]; then
    echo "No nginx processes found"
    exit 1
fi

echo "=== Nginx Process Monitor ==="
echo "PID      MEM(KB)  STATE  OPENFILES"
echo "---      -------  -----  ---------"

for PID in $PIDS; do
    MEM=$(ps -p $PID -o rss= 2>/dev/null)
    STATE=$(ps -p $PID -o stat= 2>/dev/null | cut -c1)
    OPENFILES=$(lsof -p $PID 2>/dev/null | wc -l)

    echo "$PID     $MEM     $STATE    $OPENFILES"

    if [[ "$STATE" == "D" ]]; then
        echo "  ⚠️  WARNING: PID $PID stuck in I/O (state D)"
    fi
done
```

**Usage:**
```bash
$ chmod +x monitor_nginx.sh
$ ./monitor_nginx.sh
=== Nginx Process Monitor ===
PID      MEM(KB)  STATE  OPENFILES
---      -------  -----  ---------
1234     2048     S      45
5678     1800     S      40
```

---

### Exercise 3: Logging and Troubleshooting

#### Problem 1 Solution: Find Recent Errors

**Task 1A:**
```bash
$ journalctl -p err --since "1 hour ago"
Jan 19 11:05:23 server kernel: [12345] out of memory
Jan 19 11:10:15 server sudo[5678]: alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/usr/bin/vim /etc/config
```

**Task 1B:**
```bash
$ journalctl -u nginx -p err
# Might be empty if nginx not running or no errors
# If running:
Jan 19 10:30:25 server nginx[1234]: [emerg] open() "/etc/nginx/nginx.conf" failed
```

**Task 1C:**
```bash
$ journalctl -p err --since "1 hour ago" -o short-full
Fri 2024-01-19 11:05:23.456789 UTC kernel[1]: [12345] Out of memory...
```

---

#### Problem 2 Solution: Monitor Service Real-Time

**Terminal 1:**
```bash
$ journalctl -u sshd -f
(waits for log entries)
```

**Terminal 2:**
```bash
$ ssh localhost
# Connection attempt appears in Terminal 1:
Jan 19 11:20:15 server sshd[9999]: Authentication attempt for user...
Jan 19 11:20:15 server sshd[9999]: Failed password for invalid user...
```

---

#### Problem 3 Solution: Time Range Filtering

**Task 3A:**
```bash
$ journalctl --since "10:00:00" --until "11:00:00"
# (Shows all logs between 10am and 11am today)

$ journalctl --since "10:00:00" --until "11:00:00" | wc -l
243    # 243 lines of output
```

**Task 3B:**
```bash
$ journalctl -b
# Shows logs from current boot only
# Much faster than entire journal if system has been up a long time
```

**Task 3C:**
```bash
$ journalctl -b -1
# Previous boot's logs (if available)
# Might be empty on systems that rotate frequently
```

---

#### Problem 4 Solution: Parse and Analyze

**Task 4A:**
```bash
$ journalctl | grep -i "error" | wc -l
42

$ journalctl -u nginx | grep -i "error" | wc -l
5
```

**Task 4B:**
```bash
$ journalctl -p err --since today -o short | awk '{$1=$2=$3=""; print}' | sort | uniq -c | sort -rn
    5 out of memory: Kill process (pid 1234)
    3 [error] could not open file
    2 permission denied
```

**Task 4C:**
```bash
$ journalctl --since today > /tmp/today_logs.txt

$ journalctl --since today -o json > /tmp/today_logs.json

$ ls -lh /tmp/today_logs.*
-rw-r--r-- 1 user user 2.4M Jan 19 /tmp/today_logs.json
-rw-r--r-- 1 user user 1.2M Jan 19 /tmp/today_logs.txt
```

---

#### Problem 5 Solution: Check Journal Settings

**Task 5A:**
```bash
$ ls -la /var/log/journal/
total 512
drwxr-s--- 2 root systemd-journal 4096 Jan 19 /var/log/journal/

$ ls -la /run/log/journal/
total 0
drwxr-s--- 2 root systemd-journal 4096 Jan 19 /run/log/journal/

# /var/log/journal/ has data = persistent
# /run/log/journal/ is empty = not using volatile
```

**Task 5B:**
```bash
$ journalctl --disk-usage
Journals take up 234.3M on disk.
```

**Task 5C:**
```bash
$ journalctl --list-boots
-7 feb3d4a5c6d7e8f9g h i j k l m n o Friday 2024-01-12 10:30:45 CET
-6 a1b2c3d4e5f6g7h8i j k l m n o p Friday 2024-01-13 15:22:10 CET
-5 xyz9876543210abcd e f g h i j k l Friday 2024-01-14 08:15:30 CET
...
 0 feb1a2b3c4d5e6f7g h i j k l m n Friday 2024-01-19 09:00:00 CET

$ journalctl -n 0 | tail -1
# Shows total entries in current journal
```

---

#### Problem 6 Solution: Troubleshoot Service

**Task 6A:**
```bash
$ systemctl status cups
● cups.service - CUPS Printing Service
   Loaded: loaded (/lib/systemd/system/cups.service)
   Active: active (running) since Jan 19 10:15:00 UTC
```

**Task 6B:**
```bash
$ journalctl -u cups -n 20
Jan 19 10:15:02 server cupsd[1234]: [notice] cupsd started
Jan 19 10:15:03 server cupsd[1234]: [warn] listening on socket /var/run/cups/cups.sock
```

**Task 6C:**
```bash
$ journalctl -u cups -p err -o verbose
-- Logs begin at Fri 2024-01-19 09:00:00 UTC, end at Fri 2024-01-19 11:30:00 UTC. --
PRIORITY=3
MESSAGE=Error opening config file /etc/cups/cupsd.conf
_SYSTEMD_UNIT=cups.service
_PID=1234
```

**Task 6D:**
```bash
# Terminal 1
$ journalctl -u cups -f
(watching)

# Terminal 2
$ sudo systemctl restart cups
(watch Terminal 1)

Jan 19 11:30:00 server systemd[1]: Stopping CUPS...
Jan 19 11:30:01 server cupsd[1234]: Shutdown notification received
Jan 19 11:30:02 server systemd[1]: Stopped CUPS
Jan 19 11:30:02 server systemd[1]: Starting CUPS...
Jan 19 11:30:03 server cupsd[5678]: cupsd started
Jan 19 11:30:03 server systemd[1]: Started CUPS
```

---

#### Challenge: Health Check Script

**Solution:**
```bash
#!/bin/bash

echo "=== System Health Check ==="
echo ""

echo "1. Recent Errors (last hour):"
journalctl -p err --since "1 hour ago" -n 5 -o short
echo ""

echo "2. Critical Issues (last 24h):"
journalctl -p crit --since "24 hours ago" -o short
echo ""

echo "3. Failed Services:"
systemctl list-units --failed --no-pager
echo ""

echo "4. Journal Size:"
journalctl --disk-usage
echo ""

echo "5. Most Active Services (last hour):"
journalctl --since "1 hour ago" -o json 2>/dev/null | \
  jq -r '._SYSTEMD_UNIT' | sort | uniq -c | sort -rn | head -5
echo ""

echo "6. System Load:"
uptime
```

**Usage:**
```bash
$ chmod +x health_check.sh
$ ./health_check.sh
=== System Health Check ===

1. Recent Errors (last hour):
Jan 19 11:05:23 kernel: out of memory
Jan 19 11:20:15 sshd: failed password

2. Critical Issues (last 24h):
(none)

3. Failed Services:
(none)

4. Journal Size:
Journals take up 234.3M on disk.

5. Most Active Services (last hour):
      8 systemd-resolved.service
      5 sshd.service
      4 kernel

6. System Load:
 11:35:00 up 5 days, 3:15, 2 users, load average: 0.45, 0.52, 0.48
```

---

### Practice Scenario 1: User loses access to shared directory

**Problem:** User alice suddenly can't access `/shared/project`, though she could yesterday.

**Investigation steps:**
```bash
# 1. Check directory permissions
$ ls -ld /shared/project
drwxr-x--- 2 bob dev 4096 Jan 19 /shared/project

# 2. Check alice's groups
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice)

# 3. Check file ownership
$ ls -l /shared/project/
-rw-r----- 1 bob dev 1024 Jan 19 important_file.txt
```

**Diagnosis:**
- Directory is owned by bob:dev
- Alice is not in 'dev' group
- Directory permissions are `750` (rwxr-x---)
- Alice would be in "others" category
- "Others" has no permission (---)

**Solution:**
```bash
# Add alice to dev group
$ sudo usermod -a -G dev alice

# alice must logout/login for group membership to take effect
# Or use:
$ su - alice   # Switch user to pick up new groups
```

**Root cause:** Permissions were probably tightened recently, or alice was removed from the group.

---

### Practice Scenario 2: Service keeps crashing

**Problem:** The `app` service keeps restarting every 5 minutes.

**Investigation:**
```bash
# 1. Check service status
$ systemctl status app
● app.service - My App
   Loaded: loaded (/etc/systemd/system/app.service)
   Active: active (running) since Jan 19 11:00:00 UTC
   Process: 5432 ExecStart=/opt/app/bin/app (code=exited, status=1)

# 2. Check logs
$ journalctl -u app -n 50
Jan 19 11:05:00 host app[5432]: Permission denied opening /var/app/data
Jan 19 11:04:55 host app[5432]: Permission denied opening /var/app/data
Jan 19 11:04:50 host app[5432]: Permission denied opening /var/app/data

# 3. Check permissions
$ ls -ld /var/app/
drwxr-xr-x 2 root root 4096 Jan 19 /var/app/

$ ps -u | grep app
root     12345  0.0  0.5 123456 5120 ?  S    11:05 /opt/app/bin/app

# 4. Who runs the service?
$ systemctl cat app
[Unit]
Description=My App
[Service]
ExecStart=/opt/app/bin/app
User=appuser
```

**Diagnosis:**
- Service runs as `appuser`
- Directory /var/app is owned by root:root
- appuser can't write to it (permissions are `755`, only root can write)

**Solution:**
```bash
# Fix the directory ownership
$ sudo chown appuser:appuser /var/app
$ sudo chmod 755 /var/app

# Or if using a shared group:
$ sudo chown appuser:appdata /var/app
$ sudo chmod 755 /var/app

# Restart the service
$ sudo systemctl restart app

# Verify it stays running
$ sleep 10 && systemctl status app
```

**Prevention:** When deploying services, ensure the service user owns or has access to all directories it needs.

---

## Key Differences from 18 Months Ago

**Things that might have changed:**
1. **Permissions model still the same** — you got it right
2. **systemd more standard** — most distros moved to journalctl fully
3. **cgroups for resource limiting** — becoming more common
4. **SELinux/AppArmor** — probably in use, adds permission layer on top
5. **Process isolation** — containers/systemd-nspawn common, affects how you see processes

---

Powered by UQS v1.8.5-C
