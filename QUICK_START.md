# Quick Start: 30-Minute Linux Refresher

**Goal:** Brush up on the most essential commands and concepts
**Time:** 30 minutes
**Format:** Read + run commands

---

## 1. System Status (5 minutes)

**What's running right now?**

```bash
# See overall system health
uptime                    # Load average, how long system running

# CPU and memory
free -h                   # Memory usage
top -b -n 1              # One snapshot of top (CPU, memory, processes)

# Check recent boot/failures
last -10                 # Last logins
journalctl --boot -n 20  # Last 20 messages this boot
```

**What this tells you:**
- `load average 2.5 2.0 1.8` = how busy is the system
- Memory: free vs used (used can be misleading, check available)
- Top processes eating resources

---

## 2. Disk & Storage (5 minutes)

**What's using disk space?**

```bash
# Disk usage
df -h                    # Disk usage by mountpoint
du -sh /var/*            # What's in /var directory

# Block devices
lsblk                    # Show all block devices (disks, partitions)
sudo fdisk -l            # Detailed disk info (needs sudo)

# Filesystem status
mount                    # What's currently mounted
```

**What this tells you:**
- Which disks are full
- What's mounted where
- RAID status (check `lsblk` for md devices)

---

## 3. Network (5 minutes)

**Is the network working?**

```bash
# Interface status
ip link                  # All network interfaces (up/down status)
ip addr                  # IP addresses assigned

# Can we reach things?
ping -c 1 8.8.8.8       # Can we reach the internet
traceroute 8.8.8.8      # Route to a destination

# Listening services
ss -tlnp                 # TCP listeners (services listening on ports)
netstat -an              # All network connections

# DNS
nslookup google.com      # DNS resolution
```

**What this tells you:**
- Is the NIC up and has IP?
- Can we reach remote hosts?
- What services are listening?
- Is DNS working?

---

## 4. Processes & Performance (5 minutes)

**What's the system doing?**

```bash
# Process info
ps aux                   # All processes (memory, CPU %)
ps aux | grep <process>  # Find specific process

# I/O activity
iostat -x 1 3           # I/O stats, 1-second interval, 3 samples
iotop -b -n 1           # Top I/O processes (needs root)

# Running processes details
pstree                   # Process tree (parent/child relationships)
```

**What this tells you:**
- Which process is consuming CPU/memory
- Is I/O the bottleneck?
- What processes are running under which user?

---

## 5. Logs & Diagnostics (5 minutes)

**What happened recently?**

```bash
# System journal (systemd)
journalctl -xe           # Recent errors with explanations
journalctl --since "1 hour ago"  # Last hour of logs
journalctl -u nginx      # Logs for specific service

# Traditional logs
tail -f /var/log/messages      # Real-time system log
tail -50 /var/log/syslog       # Last 50 lines of syslog
dmesg                          # Kernel messages

# Check specific issues
sudo dmesg | tail -20    # Recent kernel messages (might show hardware issues)
```

**What this tells you:**
- What error messages occurred?
- When did a service start/stop?
- Did something fail and why?

---

## 6. Users & Permissions (5 minutes)

**Who has access to what?**

```bash
# Current user
whoami                   # Who am I logged in as
id                       # UID, GID, groups

# File permissions
ls -la /var/log          # Show permissions and ownership
stat /path/to/file       # Detailed file info

# Check sudo access
sudo -l                  # What can I sudo?

# Who else is logged in?
who                      # Who's currently logged in
w                        # More detail (what are they doing)
```

**What this tells you:**
- Who owns a file and can access it?
- Can a user run something with sudo?
- Is someone else logged in?

---

## Quick Exercise: Diagnose a Slow System

**Scenario:** User reports system is slow. You have 2 minutes to figure out why.

**Run these in order:**

```bash
# 1. Check load
uptime

# 2. See what's using resources
top -b -n 1 | head -20

# 3. Check disk
df -h

# 4. Check I/O
iostat -x 1 1

# 5. Look for errors
journalctl -xe --since "10 minutes ago"
```

**What you're looking for:**
- High load + high CPU% = CPU bottleneck
- High I/O wait% + high iostat = Disk I/O bottleneck
- Low free memory + swapping = Memory pressure
- Disk full = No space issues

---

## Common L1 TSE Scenarios

### "Network is slow"
```bash
ping <destination>              # Packet loss?
traceroute <destination>        # Which hop is slow?
iperf3 (with remote server)     # Bandwidth test
ss -s                           # TCP connection stats
```

### "Filesystem keeps filling up"
```bash
df -h                           # Which filesystem full?
du -sh /*                       # What's taking space at root?
du -sh /var/*                   # What's in /var? (logs?)
find /var/log -type f -mtime +30 -delete  # Clean old logs
```

### "Process keeps crashing"
```bash
journalctl -u <service> -n 50   # Service logs
ps aux | grep <process>         # Is it running now?
strace -p <pid>                 # What's the process doing?
coredump files?                 # Check /var/lib/systemd/coredump
```

### "High CPU usage"
```bash
top -b -n 1 | head -12          # Top 10 processes by CPU
perf top                        # Real-time CPU profiling
```

---

## Key Things to Remember

**These commands are your toolkit — get comfortable with them:**
1. `top` / `htop` — Process status
2. `iostat` — I/O analysis
3. `journalctl` — System logs
4. `ss` / `netstat` — Network connections
5. `df` / `du` — Disk usage
6. `strace` — Debug process behavior

**Always ask yourself:**
- What's the baseline (what's normal for THIS system)?
- When did this start?
- What changed?
- Is it CPU, disk I/O, memory, or network?

---

## Next Steps

**If this felt familiar:**
- Jump to the exercises in Week 2+ (storage and performance)
- Skip the fundamentals, focus on advanced diagnostics

**If this felt rusty:**
- Start with Week 1: System Administration Refresher
- Do the interactive exercises
- Build confidence before moving forward

**If you want to focus on DDN/Lustre:**
- Skip ahead to Week 7: Storage-Specific Training

---

## How Ready Are You?

**Quick Self-Assessment:**

Can you explain these without looking them up?
- [ ] What `load average` means and how to interpret it
- [ ] The difference between `free` and `available` memory
- [ ] What `iowait` means and why it matters
- [ ] How to find which process is listening on port 8080
- [ ] Where system logs are stored and how to read them
- [ ] What a file permission like `644` or `755` means
- [ ] How to tail a log file and follow new entries in real-time

**All checked?** → Jump to Week 2
**Some unchecked?** → Start with Week 1
**Most unchecked?** → Do Week 1 slowly, no rush

---

That's your 30-minute refresher. You now have the mental model to support production systems.

**Ready to dig deeper?** Open the Week 1 lessons.

Powered by UQS v1.8.5-C
