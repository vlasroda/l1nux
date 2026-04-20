# Linux L1 Technical Support Engineer Training
## DDN Storage Systems - Practical Refresher

**Goal:** Refresh advanced Linux knowledge for production support on DDN storage systems
**Level:** Intermediate → Advanced (assumes prior Linux knowledge)
**Format:** Interactive lessons + hands-on exercises with real scenarios
**Time Commitment:** ~2-3 hours per week for 6-8 weeks (flexible)

---

## What This Training Covers

### Why DDN-Focused?
DDN (Data Direct Networks) manufactures high-performance storage systems (EXAScaler, Lustre, ObjectStor). L1 TSEs need to:
- Diagnose storage performance issues
- Troubleshoot networking problems
- Read system logs and metrics
- Run diagnostics and collect data
- Understand filesystem behavior
- Monitor system health

### Core Skills (In Order)

1. **System Administration Refresher** (Week 1)
   - User/permission management
   - Process management and signals
   - System logging and journald
   - Package management

2. **Storage & Filesystems** (Week 2)
   - Filesystem types and performance
   - Block devices and partitions
   - RAID and LVM basics
   - Storage monitoring

3. **Networking for Storage** (Week 3)
   - TCP/IP fundamentals
   - Network diagnostics (ping, traceroute, netstat, ss)
   - Interface configuration
   - Common network issues

4. **Performance Analysis** (Week 4)
   - System metrics (CPU, memory, I/O)
   - Tools: top, iostat, iotop, perf
   - Identifying bottlenecks
   - Baseline vs anomaly

5. **Log Analysis & Troubleshooting** (Week 5)
   - Log locations and formats
   - grep, awk, sed for log parsing
   - Systemd journal queries
   - Real production scenarios

6. **Advanced Diagnostics** (Week 6)
   - strace and ltrace
   - tcpdump for packet analysis
   - blktrace for I/O analysis
   - Performance profiling

7. **Storage-Specific (DDN/Lustre)** (Week 7)
   - Lustre filesystem basics
   - Lustre monitoring
   - Common Lustre issues
   - Storage appliance diagnostics

8. **Real Support Scenarios** (Week 8)
   - Multi-issue case studies
   - Triage and escalation
   - Documentation & handoff

---

## How to Use This Training

### Format
Each lesson includes:
1. **Quick Review** (refresher of key concepts)
2. **Key Commands** (most useful tools for this topic)
3. **Interactive Exercise** (hands-on scenario)
4. **Challenge Questions** (test understanding)
5. **Solutions** (with explanations)

### Suggested Workflow

**Option A: Full Immersion (2-3 hours/week)**
- Monday: Read lesson + run quick commands
- Wednesday: Do interactive exercise
- Friday: Challenge questions + review

**Option B: Daily Micro-Learning (30-45 min/day)**
- Day 1: Lesson + key commands
- Day 2: Interactive exercise
- Day 3: Challenge questions
- Day 4-5: Review or move ahead

**Option C: Skill-Based (Jump to what you need)**
- Use the reference index to find topics
- Start with weak areas first
- Use exercises to validate knowledge

### Prerequisites

You should be comfortable with:
- SSH access and basic shell commands
- File permissions and ownership
- Package installation and updates
- Running commands with sudo
- Basic text editing (vi/vim or nano)

If any of these feel rusty, start with **Week 1: System Administration Refresher**.

---

## Progress Tracking

**Recommended approach:** Track your answers to challenge questions

```
[ ] Week 1: System Admin Refresher
    [ ] Lesson 1.1: Users & Permissions
    [ ] Lesson 1.2: Process Management
    [ ] Lesson 1.3: Logging with journald
    [ ] Challenge Questions Week 1

[ ] Week 2: Storage & Filesystems
    [ ] Lesson 2.1: Filesystem Concepts
    [ ] Lesson 2.2: Block Devices & RAID
    [ ] Lesson 2.3: LVM & Storage Monitoring
    [ ] Challenge Questions Week 2
```

---

## Quick Reference Index

**Need to quickly brush up on:**
- [Permissions?](#permissions) → Lesson 1.1
- [Network troubleshooting?](#networking) → Lesson 3.1-3.3
- [Performance analysis?](#performance) → Lesson 4.1-4.4
- [Log analysis?](#logs) → Lesson 5.1-5.3
- [Lustre?](#lustre) → Lesson 7.1-7.3

---

## Getting Started

### Step 1: Assessment
Before jumping in, run this quick check:

```bash
# Can you explain what these show?
top              # System load
iostat -x 1      # I/O statistics
ss -tlnp         # Network listeners
journalctl -xe   # System journal
lsblk            # Block devices
df -h            # Disk usage
```

If these feel familiar, you're in the right place. If not, start with Week 1.

### Step 2: Choose Your Path

**Path A: Refresher Mode** (you remember most, need hands-on)
- Skim lessons quickly
- Do every exercise
- Focus on challenge questions

**Path B: Deep Dive** (rebuild from foundation)
- Read lessons thoroughly
- Do exercises with explanation
- Test yourself on challenge questions

**Path C: Spot Check** (know your weak spots)
- Jump to specific lessons
- Do relevant exercises
- Build understanding from gaps

### Step 3: Set Up a Practice Lab

You'll need:
- **Option 1:** Linux VM on your machine (VirtualBox, KVM, Hyper-V)
- **Option 2:** Cloud VM (AWS, Azure, GCP free tier)
- **Option 3:** Your current Linux machine (if you're comfortable testing there)

Recommended: **Ubuntu 22.04 LTS** or **Rocky Linux 8/9** (similar to production)

VM specs:
- 2+ CPU cores
- 4+ GB RAM
- 30+ GB disk space
- Network access

---

## Files in This Project

```
linux_tse_training/
├── README.md (you are here)
├── QUICK_START.md (30-minute refresher)
├── lessons/
│   ├── 01_system_admin/
│   │   ├── users_and_permissions.md
│   │   ├── process_management.md
│   │   └── logging_journald.md
│   ├── 02_storage/
│   ├── 03_networking/
│   ├── 04_performance/
│   ├── 05_logs_and_troubleshooting/
│   ├── 06_advanced_diagnostics/
│   └── 07_storage_specific/
├── exercises/
│   ├── week1_*.md
│   ├── week2_*.md
│   └── ...
├── solutions/
│   ├── week1_solutions.md
│   └── ...
└── reference/
    ├── command_cheatsheet.md
    ├── file_locations.md
    └── troubleshooting_flowchart.md
```

---

## Daily Format (What to Expect)

**Typical lesson structure (15-20 min read):**

```
# Lesson 1.1: Users and Permissions

## Quick Review
- What you need to remember about this topic
- Why it matters for TSE work

## Key Concepts
- 5-10 essential ideas with examples

## Essential Commands
# See who's logged in
who
whoami

# Check user/group info
id
getent passwd username
```

Then:
- **Interactive Exercise:** "User alice can't access /var/log/app.log. Diagnose and fix."
- **Challenge Questions:** 3-5 questions to test understanding
- **Solution:** Step-by-step walkthrough

---

## Tips for Success

1. **Actually run the commands** — Don't just read them. Type them out.
2. **Modify the exercises** — Change parameters, see what breaks, learn why.
3. **Keep notes** — Write down what surprised you or what you learned.
4. **Ask questions** — If something doesn't make sense, we'll work through it.
5. **Review before interviews** — Do the challenge questions daily.

---

## For DDN Interviews

After completing this training, you should be able to:

✅ Diagnose why a filesystem is slow
✅ Find what process is using all the I/O
✅ Read and interpret system logs
✅ Check network connectivity and bandwidth
✅ Monitor system metrics under load
✅ Use strace to debug a hung process
✅ Explain what's in /proc and /sys
✅ Troubleshoot permission issues
✅ Understand Lustre basics
✅ Ask the right diagnostic questions

---

## Where to Start Right Now

**Next Step:** Open [`QUICK_START.md`](QUICK_START.md) for a 30-minute refresher of the most essential tools and commands.

Then choose:
- **Week 1** if system admin feels rusty
- **Week 2** if you're more storage-focused
- **Week 3** if networking is your weak point
- **Week 4** if performance analysis is new
- **Week 7** if you want to jump to Lustre/storage systems

---

## Questions or Feedback?

As we work through the lessons, let me know:
- Which topics are hardest to refresh?
- Which exercises are too easy/hard?
- What scenarios matter most for DDN roles?

We can adjust the difficulty, add more exercises, or focus on specific areas.

---

**Estimated Total Time:** 40-60 hours of learning + 20-30 hours of hands-on practice = 60-90 hours total (spreadable over 6-8 weeks)

**Time to Proficiency:** You should feel ready for L1 TSE interviews after Week 4. Weeks 5-8 are for confidence and deep knowledge.

Let's get started! 🚀

---

Powered by UQS v1.8.5-C
