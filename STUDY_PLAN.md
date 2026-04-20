# Your Personalized Study Plan

Choose your path and track your progress.

---

## Pre-Test: Where Are You?

**Rate yourself on each topic (1-5):**

- [ ] Users & Permissions: ___  (1=forgot, 5=expert)
- [ ] Process Management: ___
- [ ] Filesystems/Storage: ___
- [ ] Networking: ___
- [ ] Performance Analysis: ___
- [ ] Logs & Troubleshooting: ___
- [ ] Advanced Diagnostics: ___
- [ ] Lustre/Storage Systems: ___

**Total score:** ___ / 40

If 32+: You're good, focus on exercises
If 24-31: Mix of reading and exercises
If <24: Do the lessons thoroughly

---

## Recommended Path (8 weeks, 3 hours/week)

### Week 1: System Admin Refresher ✓

**Status:** [ ] In Progress  [ ] Complete

- [ ] Read: QUICK_START.md (30 min)
- [ ] Lesson 1.1: Users & Permissions (30 min)
  - [ ] Do exercise
  - [ ] Answer challenge questions
  - [ ] Check solutions
- [ ] Lesson 1.2: Process Management (30 min)
  - [ ] Do exercise
  - [ ] Answer challenge questions
  - [ ] Check solutions
- [ ] Lesson 1.3: Logging with journald (30 min)
- [ ] Review & Notes (30 min)

**How you feel:** [ ] Great [ ] OK [ ] Need to re-read

---

### Week 2: Storage & Filesystems

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 2.1: Filesystem Concepts (40 min)
- [ ] Lesson 2.2: Block Devices & RAID (40 min)
- [ ] Lesson 2.3: LVM & Monitoring (40 min)
- [ ] Exercises & Solutions (40 min)
- [ ] Challenge Questions (30 min)

**Key command to master:** `lsblk`, `df`, `du`

---

### Week 3: Networking for Storage

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 3.1: TCP/IP Refresher (40 min)
- [ ] Lesson 3.2: Network Diagnostics (40 min)
- [ ] Lesson 3.3: Common Issues (40 min)
- [ ] Exercises (40 min)
- [ ] Real scenarios (30 min)

**Key command to master:** `ss`, `ip`, `traceroute`

---

### Week 4: Performance Analysis

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 4.1: System Metrics (40 min)
- [ ] Lesson 4.2: Tools (top, iostat, etc) (40 min)
- [ ] Lesson 4.3: Finding Bottlenecks (40 min)
- [ ] Exercises (40 min)
- [ ] Lab: Create bottleneck, diagnose (30 min)

**Key command to master:** `top`, `iostat`, `vmstat`

---

### Week 5: Logs & Troubleshooting

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 5.1: Log Locations & Formats (40 min)
- [ ] Lesson 5.2: Parsing Logs (40 min)
- [ ] Lesson 5.3: Real Production Scenarios (40 min)
- [ ] Exercises (40 min)
- [ ] Case studies (30 min)

**Key command to master:** `journalctl`, `grep`, `awk`

---

### Week 6: Advanced Diagnostics

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 6.1: strace & ltrace (40 min)
- [ ] Lesson 6.2: tcpdump (40 min)
- [ ] Lesson 6.3: blktrace & I/O analysis (40 min)
- [ ] Exercises (40 min)
- [ ] Deep dives (30 min)

**Key command to master:** `strace`, `tcpdump`

---

### Week 7: Storage Systems (DDN/Lustre)

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Lesson 7.1: Lustre Basics (40 min)
- [ ] Lesson 7.2: Lustre Monitoring (40 min)
- [ ] Lesson 7.3: Common Issues (40 min)
- [ ] Exercises (40 min)
- [ ] Health checks (30 min)

**Key command to master:** Lustre-specific tools

---

### Week 8: Real Support Scenarios

**Status:** [ ] Not Started  [ ] In Progress  [ ] Complete

- [ ] Case Study 1: Multi-issue diagnosis (45 min)
- [ ] Case Study 2: Performance problem (45 min)
- [ ] Case Study 3: Storage failover (45 min)
- [ ] Interview prep (30 min)

**What you'll practice:**
- Asking right diagnostic questions
- Documenting findings
- Escalation decisions
- Time management

---

## Daily/Weekly Checklist

### Daily (30-45 minutes)

- [ ] Run quick commands from cheatsheet
- [ ] Read one lesson
- [ ] Do one exercise
- [ ] Note what surprised you

### Weekly

- [ ] Complete all exercises for the week
- [ ] Answer challenge questions
- [ ] Check solutions (learn from mistakes)
- [ ] Practice on your own VM
- [ ] Identify weak areas for next week

---

## Progress Tracking

### Comfort Levels (Rate each topic)

**End of Week 1:**
- Users & Permissions: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5
- Process Management: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 2:**
- Storage & Filesystems: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 3:**
- Networking: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 4:**
- Performance Analysis: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 5:**
- Logs & Troubleshooting: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 6:**
- Advanced Diagnostics: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**End of Week 7:**
- Lustre/Storage Systems: [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

**Overall:** [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5

---

## Faster Path (4 weeks - Skip to Performance/Troubleshooting)

If you're really confident in system admin basics:

- [ ] Quick review: QUICK_START.md (30 min)
- [ ] Skim Week 1 lessons (1 hour)
- [ ] Do Week 1 exercises (1 hour)
- [ ] Jump to Week 4: Performance (start here)
- [ ] Do Weeks 4-5 thoroughly
- [ ] Weeks 6-7 for depth

**Total:** 4 weeks instead of 8

---

## Slow Path (10 weeks - Deep Understanding)

If you want to really rebuild from foundation:

- [ ] Spend 2 weeks on Week 1 (deep dives on each topic)
- [ ] Each subsequent week: 10 hours instead of 3
- [ ] Do all exercises twice
- [ ] Create your own practice scenarios
- [ ] Write documentation for each topic

**Total:** 10 weeks, 80+ hours

---

## Equipment Needed

**Minimum:**
- Linux VM (Ubuntu 22.04 or Rocky 8/9)
- 2+ CPU, 4GB RAM, 30GB disk
- Network access

**For advanced labs:**
- 2-3 VMs (multinode setup)
- For networking: Add network simulation tools
- For storage: Add extra disks to VM

**Setup checklist:**
- [ ] VM created
- [ ] SSH access working
- [ ] Can sudo without password (for labs)
- [ ] Tools installed (see setup section)

---

## When You're Ready for DDN Interviews

By Week 4, you should be able to:
- [ ] Diagnose why a filesystem is slow
- [ ] Find what process is using all the I/O
- [ ] Read and interpret system logs
- [ ] Check network connectivity and bandwidth
- [ ] Monitor system metrics under load

By Week 6, add:
- [ ] Use strace to debug a hung process
- [ ] Explain what's in /proc and /sys
- [ ] Troubleshoot permission issues

By Week 8, you should:
- [ ] Understand Lustre basics
- [ ] Troubleshoot common Lustre issues
- [ ] Ask the right diagnostic questions
- [ ] Write clear technical documentation
- [ ] Escalate appropriately to L2/L3

---

## If You Get Stuck

**On a concept:**
1. Re-read the lesson
2. Do the exercise again
3. Google the topic
4. Try on your own system (break it intentionally)
5. Come back to me for deeper explanation

**On an exercise:**
1. Check the solution
2. Understand WHY (not just WHAT)
3. Modify the exercise (change variables)
4. Try a new scenario

**Low motivation:**
1. Jump to a different week (might be more interesting)
2. Focus on Week 5-6 (real troubleshooting is fun)
3. Create your own scenario
4. Remember: L1 TSE at DDN would use these skills daily

---

## Your Commit to Learning

Write this down:

**My goal:** L1 TSE role at DDN

**My commitment:** ____ hours per week

**Start date:** ___________

**Target completion:** ___________

**My weak areas to focus on:**
1. _____________________
2. _____________________
3. _____________________

**My reward when done:** _____________________

---

Ready? Start with QUICK_START.md, then move to Week 1 Lesson 1.

Good luck! 🚀

---

Powered by UQS v1.8.5-C
