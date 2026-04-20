# Week 8 Exercises: Case Studies and Capstone

---

## Exercise 8.1: Crisis Response Simulation

### Scenario: "Storage is down" (Your Turn to Diagnose)

```bash
# You receive this message at 3 PM Friday

URGENT:
Subject: Production storage down!
Content: Users can't save files to /mnt/data

Your goals:
1. Diagnose within 15 minutes
2. Fix within 30 minutes
3. Update stakeholders every 10 minutes
4. Document root cause

Assume environment:
- 5TB storage volume (/mnt/data)
- Lustre filesystem
- 30 active users
- Applications: Research data analysis
```

**Task 1A: First 5 minutes**

```bash
# What are your first 3 checks?
# (Write them out before reading below)

Your checks:
1. _________________________________
2. _________________________________
3. _________________________________

# Expected checks:
# 1. Can you write to /tmp? (basic sanity)
# 2. Is /mnt/data mounted? (mount | grep /mnt/data)
# 3. Is there space? (df -h /mnt/data)
```

**Task 1B: Simulate the diagnosis**

```bash
# Pretend:
df -h /mnt/data
# Output: Filesystem: 4.9GB, Size: 5TB, Use: 99%, Avail: 50MB

# What's your diagnosis?
# A) Disk full
# B) Filesystem corrupted
# C) Users filling with garbage
# D) Hardware failure

# What's your first action?
action: _________________________________

# Expected: Diagnose what's using space
du -sh /mnt/data/* | sort -rh
# Find the culprit (usually backup, cache, or one user)
```

**Task 1C: Communicate the fix**

```bash
# Write a 2-sentence update to users

Your update:
"_________________________________________
_________________________________________"

Expected format:
"Problem identified: Backup directory consuming 4.5TB.
Removing old backups now. Expect resolution in 15 minutes."
```

**Task 1D: Implement the fix**

```bash
# If diagnostics show:
# - /mnt/data/backups/ is 4.5GB (too old)
# - /mnt/data/user_projects/ is 0.4GB

# What's your fix?
Option A: Delete all backups
Option B: Delete backups >30 days old
Option C: Move to archive storage
Option D: Ask users to clean their data

# Select and explain your choice:
choice: ___
reason: _______________________________________________
```

**Task 1E: Verify the fix**

```bash
# After cleanup, how do you verify?

# Test 1: Space is available
df -h /mnt/data
# Should show <80% used

# Test 2: Users can write
touch /mnt/data/test
ls /mnt/data/test
rm /mnt/data/test

# Test 3: Performance normal
# (Run sample job, check it completes)

# All passing? Mark incident RESOLVED
```

---

## Exercise 8.2: Multi-Layer Crisis Analysis

### Scenario: "Everything is slow" (Complex diagnosis)

```
You notice:
- Users complain: "Storage is 2x slower than usual"
- No single obvious failure
- Multiple systems potentially involved

Your challenge:
Diagnose which layers are affected using Week 6-7 tools
```

**Task 2A: Gather data simultaneously**

```bash
# You need to run these in parallel (like real incident)
# Use multiple terminals

Terminal 1: Storage diagnostics
lfs df -h                    # (Lustre)
iostat -x 1 10              # (Disk)
mount | grep lustre         # (Check mount)

Terminal 2: Network diagnostics
ping storage.server         # (Connectivity)
mtr -c 50 storage.server   # (Packet loss)
iftop -n -s 2              # (Bandwidth)

Terminal 3: CPU/Memory diagnostics
top -b -n 1 | head -15     # (System load)
vmstat 1 5                 # (Context switches)

Terminal 4: Application diagnostics
time dd if=/mnt/data/largefile of=/dev/null  # (I/O test)
strace -c command 2>&1      # (Syscall profiling)
```

**Task 2B: Analyze the symptoms**

```bash
# You observe:
# - Disk: await 25ms (normal is 10ms), %util 85%
# - Network: 0% packet loss, 3Gbps / 10Gbps used
# - CPU: iowait 35% (was 5%)
# - Test I/O: 4 hours (was 2 hours) - 2x slower

# What's the bottleneck?
A) Network (bandwidth saturated)
B) Disk (I/O slow)
C) CPU (context switches)
D) Application (inefficient code)

# Evidence for your answer:
evidence: ___________________________________________________
```

**Task 2C: Root cause drill-down**

```bash
# Given: Disk await 25ms (2.5x normal)
# Your investigation questions:

Q1: Is this affecting all disks or one?
How to check: _______________________________________

Q2: Is the high await because of high load or high latency?
How to differentiate: ______________________________

Q3: Did something change recently?
How to investigate: _________________________________
```

---

## Exercise 8.3: Migration Risk Assessment

### Scenario: "Should we migrate from Lustre to Infinia?"

```
Your organization is considering upgrading from Lustre to Infinia.
Management wants your assessment of risks.

You need to present:
1. Why migrate (benefits from Week 7)
2. What could go wrong (risks)
3. How to de-risk (mitigation)
4. Timeline (phased approach)
```

**Task 3A: Assess your current workload**

```bash
# Current Lustre setup:
Cluster size: 50TB
File count: 10 million
Access pattern: 60% read, 40% write
Metadata ops/sec: 500 (peak)
Snapshot strategy: None (no snapshots currently)

# Analyze:
# - Is this metadata-heavy? (Yes/No)
# - Would Infinia help? (Why/Why not)
# - What's the risk level? (Low/Medium/High)

your_assessment:
metadata_heavy: _______
infinia_benefits: ______________________________________
risk_level: _______
reasoning: ____________________________________________
```

**Task 3B: Plan the migration**

```bash
# Phase 1: Assessment
Timeline: ___ weeks
Tasks:
- Hardware specification review
- Data backup to tape
- User acceptance testing plan

# Phase 2: Parallel Running
Timeline: ___ weeks
Tasks:
- Deploy Infinia cluster
- Copy data (50TB takes ___ hours at 200MB/s)
- Run in shadow mode (both systems active)

# Phase 3: Cutover
Timeline: ___ hours
Procedure:
- Final sync
- Switch applications
- Monitor

# Phase 4: Validation
Timeline: ___ weeks
Tasks:
- Run production jobs
- Verify no regressions
- Monitor reliability

Total timeline: ___ weeks
Estimated cost: $_______
Expected benefit: ______% improvement
```

**Task 3C: Risk matrix**

```bash
# Fill in your risk assessment

Risk                Probability  Impact    Mitigation
─────────────────────────────────────────────────────
Data loss          _____        High      Backup to tape
Performance worse  _____        High      Test before cutover
User disruption    _____        Medium    Phased rollout
Vendor issue       _____        Low       Support contract

# Biggest risk: _______________________________
# How to mitigate: ______________________________
```

---

## Exercise 8.4: Performance Analysis Project

### Scenario: "Users say storage is slow for their workload"

```
You're assigned to improve performance for a specific use case.
You have 4 weeks (but this is simplified version).

Steps:
1. Establish baseline (how slow is "slow"?)
2. Identify bottleneck (where's the time going?)
3. Plan improvements (what to optimize?)
4. Implement and test
5. Measure results
```

**Task 4A: Define the workload**

```bash
# Typical user workload:
Operation: Data analysis (read 100GB, compute, write 10GB)
Current time: 4 hours
Users: 5 running simultaneously (peak)
Acceptable time: <3 hours (25% improvement needed)

Your baseline measurements:
# Run actual workload, measure:

Disk I/O:
- r/s: _____
- w/s: _____
- await: _____
- %util: _____

Network:
- Utilization: _____% of 10Gbps

CPU:
- user: _____%
- sys: _____%
- iowait: _____%

Job phases:
- Data load: ___ min
- Processing: ___ min
- Write results: ___ min
```

**Task 4B: Identify bottleneck**

```bash
# Using Week 4 methodology:

Is disk the bottleneck?
Evidence: await _____ (>10ms = possibly yes)
          %util _____ (>80% = yes)
Result: Disk is ___% of problem

Is network the bottleneck?
Evidence: utilization ___% of 10Gbps
Result: Network is ___% of problem

Is CPU the bottleneck?
Evidence: iowait ____%, idle ____%
Result: CPU is ___% of problem

Primary bottleneck: _________________________
```

**Task 4C: Suggest optimizations (Week 7)**

```bash
# Based on bottleneck, suggest 3 improvements

Optimization 1: _________________________________
Why: _________________________________________
Expected impact: ____ % speedup

Optimization 2: _________________________________
Why: _________________________________________
Expected impact: ____ % speedup

Optimization 3: _________________________________
Why: _________________________________________
Expected impact: ____ % speedup

Combined: Would achieve ____% total improvement
          (25% target? Higher?)
```

**Task 4D: Implementation plan**

```bash
Timeline:
Week 1: Baseline measurement ✓
Week 2: _________________________ (opt 1)
Week 3: _________________________ (opt 2)
Week 4: Testing and validation

Risk assessment:
- Can we rollback if needed? (Yes/No)
- Will this affect other users? (Yes/No)
- Do we need downtime? (Yes/No)

Go/No-go decision: ________
Reason: ___________________________________________________
```

---

## Exercise 8.5: Capstone Scenario - Choose Your Own Adventure

```bash
# Pick one real incident type from your organization
# Apply the 8-week curriculum to diagnose/resolve it

Write a case study (500 words):

Title: ___________________________________________________________

Scenario:
- What was the problem? ___________________________________________
- When did it occur? ___________________________________________
- What was the impact? ___________________________________________

Diagnosis:
- What tools did you use? (From weeks 1-7) _______________________
- What did each tool reveal? ______________________________________
- Root cause: ____________________________________________________

Resolution:
- What was the fix? ______________________________________________
- How did you verify? ____________________________________________
- How long did it take? __________________________________________

Prevention:
- How to prevent next time? ______________________________________
- What monitoring to add? ________________________________________
- What documentation needed? _____________________________________

Learning:
- What did you learn? ____________________________________________
- How does this relate to course material? _______________________
```

---

## Self-Assessment: Ready for L1 TSE Role?

After Week 8 / all 8 weeks, you should be able to:

**System Administration (Week 1):**
- [ ] Diagnose permission and user issues
- [ ] Debug process behavior
- [ ] Manage system logging

**Storage Fundamentals (Week 2):**
- [ ] Explain filesystem concepts
- [ ] Understand RAID and LVM
- [ ] Monitor storage health

**Networking (Week 3):**
- [ ] Diagnose network connectivity
- [ ] Understand protocol basics
- [ ] Troubleshoot network issues

**Performance Analysis (Week 4):**
- [ ] Establish performance baselines
- [ ] Identify bottlenecks
- [ ] Optimize systems systematically

**Logs and Troubleshooting (Week 5):**
- [ ] Parse logs effectively
- [ ] Follow troubleshooting methodology
- [ ] Escalate appropriately

**Advanced Diagnostics (Week 6):**
- [ ] Use strace/ltrace for deep debugging
- [ ] Analyze network packets
- [ ] Profile system performance

**Lustre & Infinia (Week 7):**
- [ ] Understand both architectures
- [ ] Troubleshoot each system
- [ ] Plan migrations

**Case Studies (Week 8):**
- [ ] Diagnose multi-layer incidents
- [ ] Communicate during crisis
- [ ] Implement optimizations

---

Powered by UQS v1.8.5-C
