# Lesson 8.1: Enterprise Storage Crisis - Diagnosis and Response

**Time:** 25-30 minutes
**Why it matters:** Real production crises are never single-issue. This lesson integrates weeks 1-5 knowledge into actual incident response under time pressure
**Skills:** Multi-layered diagnosis, prioritization under stress, escalation, documentation, post-mortems

---

## The Crisis Scenario

**Initial Report (8:47 AM Monday):**
```
From: Marcy (VP Research Operations)
To: L1 Storage Team
Subject: URGENT - Research cluster storage down

"Our researchers can't write to /mnt/research. Started this morning.
We have 50+ users blocked. This is affecting our grant deadline work.
Critical issue!"

Time pressure: Grant deadline is 72 hours
Impact: 50 users, 2 research projects, $500K grant at risk
Escalation: Already at VP level
```

---

## Initial Response (First 15 Minutes)

### Step 1: Gather Information Quickly

```bash
# Get basic facts (do this FIRST, before diving deep)
1. Can you write to /mnt/research at all?
   touch /tmp/test_write  # Can write to /tmp?
   touch /mnt/research/test_write  # Can write to storage?

2. When did it stop?
   ls -la /mnt/research/ | tail -1
   # Shows timestamp of last file

3. Is it all users or specific ones?
   As different user: touch /mnt/research/test2
   # Permission issue would fail for that user

4. Is it the storage or network?
   ping storage.server  # Reachable?
   nc -zv storage.server 2049  # NFS port responding?

5. Check Lustre/Infinia status
   lfs df -h  # (if Lustre)
   infinia cluster status  # (if Infinia)
```

### Step 2: Quick Win Checks

```bash
# These often solve 50% of incidents (5 minutes)

Check 1: Disk full?
df -h /mnt/research
# If 100% full: BINGO (solve below)

Check 2: Permissions changed?
ls -ld /mnt/research
# Owner/group/permissions correct?

Check 3: Service crashed?
systemctl status nfs-server  # (if NFS)
systemctl status lustre  # (if Lustre)
systemctl status infinia  # (if Infinia)

Check 4: Network interface down?
ip link show | grep -E "UP|DOWN"
# Any interfaces DOWN?

Check 5: Recent changes?
journalctl -p err --since "2 hours ago"
# Any error messages?
```

### Step 3: Determine Severity Level

```
Severity 1 (Immediate): Data accessible but degraded
├─ Users CAN read, CAN'T write
├─ Some users work, others blocked
└─ Temporary workaround possible

Severity 2 (Urgent): Data mostly inaccessible
├─ All users blocked
├─ Storage appears offline
├─ No workaround available
└─ Escalate immediately

Severity 3 (Critical): Data at risk
├─ Corruption detected
├─ Replication failing
├─ Storage nodes offline (multiple)
└─ Potential data loss

In this scenario: Severity 2 (all write blocked)
→ Escalate, but first try quick fixes
```

---

## Diagnosis Phase (First 30 Minutes)

### Root Cause Analysis Flow

```
Can't write to /mnt/research

├─ Check 1: Is filesystem mounted?
│  └─ mount | grep research
│     ├─ YES: Continue
│     └─ NO: Mount it, done
│
├─ Check 2: Is there space?
│  └─ df -h /mnt/research
│     ├─ 100% full: FREE SPACE (see Solution 1)
│     └─ Space available: Continue
│
├─ Check 3: Can reach storage server?
│  └─ ping storage.server
│     ├─ NO: Network problem (see Solution 2)
│     └─ YES: Continue
│
├─ Check 4: Is NFS/Lustre listening?
│  └─ nc -zv storage.server 2049 (NFS)
│     ├─ TIMEOUT: Service down (see Solution 3)
│     └─ SUCCESS: Continue
│
├─ Check 5: Permissions on mount?
│  └─ ls -ld /mnt/research
│     ├─ drwx------: Only root can write (see Solution 4)
│     └─ drwxrwxrwx: Continue
│
├─ Check 6: User's home directory issue?
│  └─ id username
│     ├─ No group membership: Add to group (see Solution 4)
│     └─ Groups OK: Continue
│
└─ Check 7: Quota issue?
   └─ lfs quota -u username  (if Lustre)
      ├─ At quota: Delete files or increase (see Solution 5)
      └─ Below quota: Continue
```

### Likely Root Causes (in order of probability)

```
Probability    Root Cause              Symptoms
────────────────────────────────────────────────
50%            Disk Full               df shows 100%, write fails
30%            Permission Changed      ls -ld shows wrong perms
10%            Service Crash           Storage server down
5%             Network Issue           Can't ping server
5%             Quota Exceeded          User hit limit (Lustre)
```

---

## Solution Execution (Phases)

### Solution 1: Disk Full (Most Common)

**Diagnosis:**
```bash
df -h /mnt/research
# Output: Filesystem 100% full

lfs df -h  # (if Lustre)
# Shows which OST is full

infinia bucket stats  # (if Infinia)
# Shows which bucket full
```

**Immediate Fix (minutes):**
```bash
# Find largest files
du -sh /mnt/research/* | sort -rh | head -10

# Identify candidates for cleanup
find /mnt/research -type f -mtime +90 -size +1G
# Files older than 90 days, >1GB

# Option 1: Archive old data
tar czf /archive/research-archive-$(date +%Y%m%d).tar.gz /mnt/research/old_project/
rm -rf /mnt/research/old_project

# Option 2: Compress existing files
gzip /mnt/research/backup/*.sql

# Option 3: Move to different storage
mv /mnt/research/large_data /mnt/secondary/

# Verify space freed
df -h /mnt/research
# Should no longer be 100%
```

**Permanent Solution (hours):**
```bash
# Understand growth
du -sh /mnt/research | head -1
# Get baseline

# Growth rate
du -sh /mnt/research > /tmp/size_today
# Compare to yesterday's measurement

# Expansion needed
# If growing 1TB/week, need to add capacity

# Capacity planning
# Growth = 1TB/week
# Current free after cleanup = 5TB
# Will fill in: 5 weeks
# Must add storage before then

# Set quota per user
setquota -u researcher1 -b 0 500G 0 1M /mnt/research
# Max 500GB per user
```

**Prevention:**
```bash
# Monitor space
crontab -l | grep "df -h"
# Add if not present
0 9 * * * df -h /mnt/research | mail -s "Storage report" admin@example.com

# Set alerts
# Alert at 80% full
# Alert at 90% full
# Alert at 95% full (emergency)

# Implement retention policy
find /mnt/research -type f -mtime +180 -delete
# Auto-delete files older than 6 months
```

### Solution 2: Network Problem

**Diagnosis:**
```bash
ping storage.server
# TIMEOUT = network issue

nc -zv storage.server 2049
# TIMEOUT = can't reach port

traceroute storage.server
# Shows where packet is lost
```

**Fix:**
```bash
# Check local interface
ip link show eth0
# Should be UP

# Check route
ip route show
# Default gateway present?

# Check firewall
sudo iptables -L | grep 2049
# Port 2049 should not be blocked

# If interface is DOWN
ip link set eth0 up
# Bring interface up

# If no route
ip route add default via 10.0.0.1
# Add default gateway

# If firewall blocking
sudo iptables -A INPUT -p tcp --dport 2049 -j ACCEPT
# Allow NFS traffic
```

### Solution 3: Storage Service Down

**Diagnosis:**
```bash
# Check if service is running
systemctl status nfs-server  # NFS
systemctl status lustre      # Lustre
systemctl status infinia     # Infinia

# Check logs
journalctl -u nfs-server -n 50  # Last 50 lines
journalctl -u lustre -p err     # Errors only

# Check port listening
ss -tlnp | grep 2049  # Should see listening
netstat -an | grep 2049
```

**Fix:**
```bash
# If service is down, restart it
systemctl restart nfs-server

# Check for obvious errors
journalctl -u nfs-server -p err | head -20

# Common errors:
# "Port already in use" → Kill old process
# "Permission denied" → Check file permissions
# "Out of memory" → Server low on RAM

# If restart fails
systemctl start nfs-server  # Try start (not restart)
systemctl status nfs-server  # Check status

# If still fails, escalate to storage team
# This is now a storage-side issue
```

### Solution 4: Permission Problem

**Diagnosis:**
```bash
ls -ld /mnt/research
# Shows: drwxr-xr-x root root
# Users in group can read, but not write

id researcher1
# Shows groups user is in
# If not in "research" group: That's the problem
```

**Fix:**
```bash
# Option 1: Add user to group
usermod -a -G research researcher1
# User must log out and back in (or newgrp)

# Option 2: Fix directory permissions
chmod g+w /mnt/research
# Give group write permission

# Option 3: Create group directory
mkdir -p /mnt/research/researcher1
chmod 700 /mnt/research/researcher1
chown researcher1:research /mnt/research/researcher1
# Private directory for that user
```

**Verify:**
```bash
# As the user, test write
sudo -u researcher1 touch /mnt/research/test
# Should succeed

rm /mnt/research/test  # Clean up
```

### Solution 5: Quota Exceeded

**Diagnosis:**
```bash
lfs quota -u researcher1 /mnt/research
# Output: researcher1 has used 500GB (limit)
# Can't create more files

# Check who is over quota
lfs quota -u $(whoami) /mnt/research
```

**Fix:**
```bash
# Option 1: Delete old files
find /mnt/research/researcher1 -type f -mtime +90 -delete
# Delete files older than 90 days

# Option 2: Increase quota
setquota -u researcher1 -b 0 750G 0 1M /mnt/research
# Increase from 500G to 750G

# Option 3: Archive to different location
tar czf /archive/researcher1-$(date +%Y%m%d).tar.gz \
    /mnt/research/researcher1/old_project
rm -rf /mnt/research/researcher1/old_project

# Verify
lfs quota -u researcher1 /mnt/research
# Should show reduced usage
```

---

## Communication and Escalation

### First Update (30 minutes after report)

```
To: Marcy, Research Team
From: L1 Storage Support
Subject: RE: Storage issue - UPDATE

Status: ACTIVE INVESTIGATION
Root cause identified: Storage disk full (99% capacity)

What we're doing:
- Cleaning old backup files (2TB freed so far)
- Identifying additional candidates for archival
- ETA: Normal service in next 30 minutes

Temporary workaround:
- Can use /mnt/secondary for urgent work (10TB available)
- Once /mnt/research is cleaned, data can move back

Next update: 30 minutes
```

### Second Update (60 minutes after report)

```
To: Marcy, Research Team
From: L1 Storage Support
Subject: RE: Storage issue - RESOLVED

Status: RESOLVED
Root cause: Backup scripts were not cleaning old data
Fixed: Manually removed 5TB of backup files (>180 days old)

Current status:
✓ Write access restored
✓ All 50 users can access /mnt/research
✓ Storage now at 65% capacity (healthy)
✓ Performance normal

What caused this:
- Backup process has no retention policy
- Old backups accumulate indefinitely
- Reached capacity after 8 months

Permanent solution (being implemented):
- Added retention policy: Delete backups >180 days old
- Automated daily cleanup at 11 PM
- Storage alerts at 80%, 90%, 95% full
- Expected: Won't happen again

Thank you for your patience. Impact: Grant work can resume.
```

### Post-Incident Review (24 hours later)

```
Incident Summary:
- Duration: 1 hour 15 minutes (80 minutes user impact)
- Severity: Critical (50 users, deadline at risk)
- Root cause: No backup retention policy
- Resolution: Manual cleanup + automated policy

Lessons learned:
1. Monitoring gaps: Had no space alerts (now added)
2. Process gaps: Backup process was fire-and-forget
3. Procedure gaps: Didn't have documented triage steps

Preventive measures:
1. Space monitoring: Automated alert at 80% full
2. Backup retention: Delete >180 day old backups
3. Capacity planning: Monthly growth report
4. Triage documentation: Posted in Wiki

Timeline for implementation:
- Alerts: Deployed today ✓
- Retention policy: Deployed today ✓
- Capacity reporting: Start next week
- Documentation: Complete by end of week
```

---

## Key Insights from This Crisis

```
What worked:
✓ Followed triage order (quick checks first)
✓ Communicated regularly (stakeholders informed)
✓ Fixed symptom immediately (freed space)
✓ Found root cause quickly (backup process)
✓ Implemented prevention (automated cleanup)

What to avoid:
✗ Jumping to complex solutions first
✗ Radio silence (keep stakeholders updated)
✗ Fixing once and forgetting (need prevention)
✗ Blaming users ("they filled it with garbage")
✗ Not analyzing root cause (was a process failure)

Time breakdown:
5 min: Initial info gathering
10 min: Quick win checks
5 min: Root cause identification
10 min: Fix execution
10 min: Verification and communication
```

---

## Challenge Questions

**Answer without looking them up:**

1. **First Action:** Storage can't write. What's your first test?
   ```
   A) Restart storage server
   B) Check df -h (disk free)
   C) Blame the network
   D) Escalate to storage team
   ```

2. **Communication:** When should you update the VP?
   ```
   A) Every 5 minutes (too much)
   B) Only when resolved
   C) Every 30 minutes (or milestone)
   D) Never (just fix it)
   ```

3. **Root Cause:** Backup files causing full disk. What's the FIX?
   ```
   A) Just delete them once
   B) Add retention policy (automated)
   C) Blame the backup admin
   D) Increase disk size
   ```

4. **Prevention:** How to avoid next time?
   ```
   (List 3 preventive measures)
   ```

---

## Real-World Extrapolation

This scenario represents:
- **Actual frequency:** Happens 2-3 times per year in production
- **Actual cost:** 1 hour impact × 50 users = 50 user-hours = $2500
- **Grant deadline context:** Very realistic for research orgs
- **VP involvement:** Common when deadline/money at stake

**L1 TSE lesson:**
- Most incidents are simple (80% are disk full or permission)
- Quick systematic checks find root cause in 10 minutes
- Communication matters as much as technical fix
- Prevention beats fire-fighting

---

Powered by UQS v1.8.5-C
