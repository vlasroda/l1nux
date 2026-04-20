# Week 2 Exercise 1: Filesystem Analysis

**Difficulty:** Beginner-Intermediate
**Time:** 25 minutes
**Prerequisites:** Lesson 2.1 (Filesystem Concepts)

---

## Problem 1: Analyze Current Filesystems

**Task 1A: View all mounted filesystems**
```bash
# Command to run:
df -h

# Questions to answer:
# 1. How many filesystems are mounted?
# 2. Which is the largest?
# 3. Which is closest to full (highest Use%)?
# 4. Are any over 80% full?
```

**Task 1B: Get detailed filesystem information**
```bash
# For your root filesystem:
sudo lsblk -f

# Or for a specific filesystem:
sudo file -s /dev/sda1

# Questions to answer:
# 1. What filesystem types are used?
# 2. Any ext4? XFS? Other?
# 3. What are their UUIDs?
```

**Task 1C: Check inode usage**
```bash
# Command:
df -i

# Questions to answer:
# 1. Which filesystem has highest inode usage (IUse%)?
# 2. Any above 80%?
# 3. Biggest difference between disk full and inode full?
```

---

## Problem 2: Diagnose "Filesystem Full" Scenario

**Scenario:** User reports they can't create files in /var. They have plenty of disk space. Diagnose the problem.

**Task 2A: Check capacity**
```bash
# Check space
df -h /var

# Note:
# - Used vs Available
# - Use percentage
```

**Task 2B: Check inode usage**
```bash
# Check inodes
df -i /var

# If IUse% is 100%:
# Problem is inode exhaustion, not space!
```

**Task 2C: Find what's consuming inodes**
```bash
# Count files in /var
find /var -type f | wc -l

# Find directories with most files
find /var -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 -type f | wc -l) $1"' _ {} \; | sort -rn | head -10

# Common culprits:
# /var/log/ - If not rotated
# /var/spool/ - Mail queue
# /var/cache/ - Package cache
```

**Task 2D: Clean up inodes**
```bash
# Find and list old files
find /var/log -type f -mtime +30 | head -10

# Count how many
find /var/log -type f -mtime +30 | wc -l

# (Don't delete yet - understand what's there first)
```

---

## Problem 3: Monitor Filesystem Performance

**Scenario:** Your storage is slower than expected. Investigate if filesystem fragmentation is the cause.

**Task 3A: Check fragmentation (ext4 only)**
```bash
# If using ext4:
sudo e4defrag -c /dev/sda1 2>/dev/null || echo "Not ext4 or command unavailable"

# Expected output:
# /dev/sda1: 5.2% non-contiguous

# If < 10%: OK
# If 10-20%: Watch
# If > 20%: Consider defrag
```

**Task 3B: Check I/O performance**
```bash
# Get one snapshot
iostat -x 1 3

# Focus on:
# %util: % of time disk is busy
# await: Average wait time in ms
# r/s, w/s: Read/write operations per second

# Record baseline:
# Disk 1: %util=30, await=5ms
# (You'll compare with future measurements)
```

**Task 3C: Check filesystem block allocation**
```bash
# Get filesystem details
stat /

# Look for:
# - Block size (usually 4096 bytes)
# - Fragment size
# - Total blocks and used blocks
```

---

## Problem 4: Manage Filesystem Space

**Scenario:** The root filesystem is at 85% capacity. You need to free up space carefully.

**Task 4A: Find large directories**
```bash
# Show top 10 directories by size
du -sh /* | sort -rh | head -10

# Expected output:
# 20G    /var
# 15G    /usr
# 5G     /home
# 3G     /opt
```

**Task 4B: Drill down into largest directory**
```bash
# If /var is largest
du -sh /var/* | sort -rh | head -10

# If /var/log is large
du -sh /var/log/* | sort -rh | head -10

# Check log file ages
ls -lh /var/log/*.log | awk '{print $6, $7, $8, $9}' | tail -20
```

**Task 4C: Identify safe cleanup targets**
```bash
# Find files not accessed in 60+ days
find /var/log -type f -atime +60 -ls

# Find temporary files
find /tmp -type f -mtime +7

# DON'T delete without verifying first!
```

**Task 4D: Estimate space to free**
```bash
# Count and size old files
find /var/log -type f -mtime +30 -exec ls -lh {} \; | awk '{sum += $5} END {print "Total: " sum}'

# (This shows how much you could free safely)
```

---

## Problem 5: Handle Filesystem Limits

**Scenario:** You notice a filesystem has both low space AND high inode usage. What takes priority?

**Current state:**
```
Filesystem /data:
- Disk: 100G total, 95G used (95%)
- Inodes: 1000000 total, 980000 used (98%)
```

**Task 5A: Determine the bottleneck**
```bash
# Command to check both:
df -h /data && df -i /data

# Which is closer to limit?
# Which will run out first?
```

**Task 5B: Prioritize cleanup**
```bash
# If space is issue:
# Find large files
find /data -type f -size +100M | sort -k

# If inodes issue:
# Find many small files
find /data -type f -size -1k | wc -l

# If both:
# Find large directories with many files
find /data -type d -exec sh -c 'COUNT=$(find "$1" -type f | wc -l); SIZE=$(du -sh "$1" | cut -f1); echo "$SIZE $COUNT $1"' _ {} \; | awk '$3 ~ /^[0-9]/ && $2 > 1000' | sort -rn | head -10
```

**Task 5C: Create a cleanup plan**
```bash
# Document what you'd clean:
echo "Cleanup plan for /data:"
echo "- Find and compress old files"
echo "- Delete temporary files"
echo "- Archive logs > 90 days old"
echo "- Estimated space to free: XGB"
echo "- Estimated inodes to free: Y"
```

---

## Challenge: Filesystem Health Report

**Create a script that reports filesystem health:**

```bash
#!/bin/bash

echo "=== Filesystem Health Report ==="
echo ""

echo "1. Space Usage:"
df -h | awk 'NR==1 || NR>1 {if (NF) printf "%-20s %10s %10s %10s\n", $1, $2, $3, $5}'

echo ""
echo "2. Inode Usage:"
df -i | awk 'NR==1 || NR>1 {if (NF) printf "%-20s %10s %10s %10s\n", $1, $2, $3, $5}'

echo ""
echo "3. Filesystems Over 80%:"
df -h | awk '$5 ~ /^[89][0-9]|100/ {print $1, $5, "(" $2 " total, " $3 " used)"}'

echo ""
echo "4. Top Directories (by size):"
du -sh /* 2>/dev/null | sort -rh | head -5

echo ""
echo "5. Inode Exhaustion Risk:"
df -i | awk '$5 ~ /^[89][0-9]|100/ {print $1, $5 " inodes used"}'
```

**Your task:**
1. Create this script
2. Make it executable
3. Run it weekly to monitor trends
4. Set up alerts when thresholds are crossed

---

## Self-Check

After completing all problems:

- [ ] Problem 1: Analyzed current filesystems
- [ ] Problem 2: Identified inode vs space issues
- [ ] Problem 3: Checked fragmentation and I/O perf
- [ ] Problem 4: Located and planned cleanup
- [ ] Problem 5: Balanced multiple constraints
- [ ] Challenge: Created health report script

---

## Key Skills You Practiced

- `df -h` and `df -i` — Capacity and inode monitoring
- `du -sh` — Finding space usage
- `stat` — Detailed filesystem information
- `e4defrag -c` — Checking fragmentation
- `iostat` — Performance monitoring

---

## Next Steps

Compare your answers with `week2_solutions.md`.

Focus on understanding:
1. How to quickly identify space vs inode issues
2. Safe ways to locate and clean up files
3. How to monitor trends over time

---

Powered by UQS v1.8.5-C
