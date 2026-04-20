# Week 8 Solutions: Case Studies and Capstone

---

## Exercise 8.1: Crisis Response Simulation Solutions

### Task 1A: First 5 Minute Checks

```
Correct answer: (in order of importance)

1. Can you write to /tmp?
   touch /tmp/test_write
   Why: Sanity check (is system even working?)

2. Is /mnt/data mounted?
   mount | grep /mnt/data
   Why: If not mounted, remount it (easy fix)

3. Is there space?
   df -h /mnt/data
   Why: Disk full is 50% of outages

Other valid checks:
- Is NFS service running? (systemctl status nfs-server)
- Can I reach storage server? (ping storage.server)
- Are permissions correct? (ls -ld /mnt/data)
```

### Task 1B: Diagnosis

```
Correct answer: A) Disk full

Evidence:
- 99% used (almost exactly)
- 50MB available (not enough for significant files)
- Write failures would happen when allocation needed

This is classic disk full symptoms.
```

### Task 1C: User Communication

```
Good update example:
"Problem identified: /mnt/data is full (99% capacity).
We're moving old backups to archive now.
Service should resume in 15 minutes."

Why this works:
✓ States the problem (full)
✓ States what you're doing (moving backups)
✓ Gives ETA (15 minutes)
✓ Is honest and professional

Bad examples:
✗ "Storage is broken" (vague)
✗ "Users filled it with garbage" (blaming)
✗ "Not sure how long" (no commitment)
```

### Task 1D: Implement Fix

```
Best answer: B or C (not A)

B) Delete backups >30 days old
- Pros: Frees space (4.5TB), keeps recent backups
- Cons: Still need to move really old backups

C) Move to archive storage
- Pros: Preserves all backups, frees space
- Cons: Takes longer, need archive target

Why not A:
- Deleting all backups is dangerous
- What if recent backup was corrupted?
- Would not keep compliance backups

Best sequence:
1. Move oldest backups (60+ days) to archive: Frees 3TB
2. Delete backups 30-60 days old: Frees 1.5TB
3. Verify space freed
4. Test that recent backups still work
```

### Task 1E: Verify the Fix

```
Verification checklist:

✓ Space available: df -h shows <80% full
✓ User can write: touch /mnt/data/test (succeeds)
✓ Performance normal: No errors in dmesg
✓ Recent backups intact: Check latest backup valid
✓ Users confirmed: Have them test their applications

Expected results after fix:
- df: 50% full (4TB free out of 8TB maybe, after deleting some)
- write: Success (no ENOSPC errors)
- ls: Normal output
- No new errors in journalctl
```

---

## Exercise 8.2: Multi-Layer Crisis Analysis

### Task 2A: Data Collection

```
Parallel terminals simultaneously:

Terminal 1 (Storage):
- Check if mount is healthy
- Check disk I/O (await, util)
- Check space

Terminal 2 (Network):
- Check connectivity
- Check packet loss (mtr)
- Check bandwidth utilization

Terminal 3 (CPU):
- Check load average
- Check context switches
- Check I/O wait percentage

Terminal 4 (Application):
- Test actual I/O performance
- Trace to see syscalls
- Measure job time

Key: Parallel collection means you get true picture
(Not sequential - doesn't capture peak/interactions)
```

### Task 2B: Bottleneck Analysis

```
Given observations:
- Disk await: 25ms (was 10ms) = 2.5x slower
- %util: 85% (was maybe 40%)
- Network: 0% loss, plenty of headroom
- CPU: iowait 35% (was 5%)
- Test I/O: 2x slower

Correct answer: B) Disk is the bottleneck

Evidence:
- Disk await increased 2.5x (direct correlation to slowness)
- %util at 85% (near saturation)
- CPU iowait high (waiting on disk)
- Network not saturated (plenty of headroom)

Why not others:
A) Network: Only 30% utilized, 0% packet loss (not a factor)
C) CPU: 35% iowait means CPU is waiting on disk (not bottle)
D) App: Observe 2x slowness matches disk slowness (not app bug)

Conclusion: Focus optimization efforts on disk I/O
```

### Task 2C: Root Cause Investigation

```
Q1: Is this affecting all disks or one?

Check: iostat -x 1 5 | grep await
Shows each disk's latency separately

What you're looking for:
- One disk much slower than others: That disk failing
- All disks equally slow: Shared resource (network, controller)
- Some disks normal, some slow: Unbalanced load

Q2: Is high await because of load or latency?

Check: Compare r/s, w/s with await
- High r/s and high await: Queue backlog (overloaded)
- Low r/s but high await: Slow disk (hardware)

If load:
- Solution: Distribute load (striping, load balancing)

If disk is slow:
- Solution: Replace disk, repair filesystem, check SMART

Q3: Did something change recently?

Check: journalctl --since "24 hours ago" -p warn
      dmesg | grep -i "error\|disk\|ata"
      lsblk (check for new disks, changed config)

Common changes:
- New backup job started (competing I/O)
- Someone re-mounted filesystem with different options
- Disk developed bad sectors (SMART errors)
- Load balancer redistributed traffic
```

---

## Exercise 8.3: Migration Risk Assessment

### Task 3A: Workload Assessment

```
Current Lustre setup analysis:
- 50TB / 10M files = 5KB average file size (SMALL)
- Small files = metadata heavy
- 500 metadata ops/sec = moderate (not extreme)
- 60% read / 40% write = balanced
- No snapshots = not using that feature

Is metadata heavy? YES
- 10 million files is significant
- Infinia scales better with file count

Would Infinia help? YES
- Distributed metadata (no MDS bottleneck)
- Could handle 10x more files with same performance
- If doing snapshots, would be 1000x improvement

Risk level: LOW-MEDIUM
- Data migration is straightforward (no Infinia-specific features)
- Rollback is possible (keep Lustre running during test)
- No urgent need (current system working)

Recommendation: Good candidate for migration
- Benefits outweigh risks
- Phased approach (parallel running) is feasible
- Should plan for next fiscal year
```

### Task 3B: Migration Timeline

```
Phase 1: Assessment
Timeline: 2-3 weeks
Tasks:
- Hardware spec review: 1 week
- Data backup: 1 week (50TB to tape)
- UAT plan: 1 week

Phase 2: Parallel Running
Timeline: 4-5 weeks
Tasks:
- Deploy Infinia: 1 week
- Copy data: 50TB at 200MB/s = 250K seconds = 70 hours (3 days)
  (With parallel copy, can do overnight)
- Test shadow mode: 2-3 weeks (run jobs on both)
- Validation: 1 week

Phase 3: Cutover
Timeline: 1 day (can abort if needed)
Procedure:
- Final sync: 2-3 hours
- Switch applications: 1 hour
- Monitor: next 4 hours

Phase 4: Validation
Timeline: 4 weeks
- Run production jobs: 2 weeks
- Verify regressions: 1 week
- Decommission Lustre: 1 week

Total timeline: 12-14 weeks (3 months)

Estimated cost: $500K (hardware) + 500 engineering hours ($100K)
= $600K

Expected benefit:
- Metadata performance: 5x better (500 → 2500 ops/sec)
- Snapshot efficiency: 1000x (can do hourly snapshots)
- Overall: 15-20% throughput improvement likely
```

### Task 3C: Risk Matrix

```
Risk                    Probability  Impact    Mitigation
────────────────────────────────────────────────────────
Data loss               5%           High      Backup to tape
Performance worse       10%          High      Test before cutover
User disruption         15%          Medium    Phased rollout
Vendor issue            5%           Medium    Support contract
Incompatible app        10%          Medium    UAT testing
Network issues          10%          Medium    Parallel network test
Hardware failure        5%           Low       RAID, redundancy
Cost overrun            20%          Medium    Budget buffer

Biggest risk: Performance regression
- Mitigation: Extensive testing in Phase 2
- Parallel run for 2 weeks minimum
- Compare actual numbers before cutover

Second risk: User adoption
- Mitigation: Good communication, training, gradual rollout

Key success factor: Never shut down Lustre until confident
- Keep running as fallback for 2-4 weeks
- Can always go back if major issue found
```

---

## Exercise 8.4: Performance Analysis Solutions

### Task 4A: Baseline Establishment

```
Typical workload baseline for 100GB analysis job:

Disk I/O:
- r/s: 800-1200 (depending on file size, cache hit)
- w/s: 400-600 (checkpoint writes)
- await: 12-15ms (normal for good disk)
- %util: 60-75% (healthy range)

Network:
- Utilization: 20-30% of 10Gbps (typical for storage I/O)

CPU:
- user: 40-45% (actual computation)
- sys: 8-12% (syscall overhead)
- iowait: 20-25% (waiting on I/O)

Job phases (4-hour total):
- Data load: 20-30 min (read from storage)
- Processing: 200-220 min (CPU-bound)
- Write results: 5-10 min (checkpoint)
- Overhead: 10-15 min (startup, cleanup)

Bottleneck phases:
- Data load: Disk I/O bound
- Processing: CPU bound (waiting on I/O between batches)
- Write results: Network bound (checkpoint data flush)
```

### Task 4B: Bottleneck Identification

```
Is disk the bottleneck?
Evidence: await 12ms (not extreme), %util 70% (moderate)
Result: Disk contributes ~30% of problem

Is network the bottleneck?
Evidence: utilization 25% of 10Gbps (lots of headroom)
Result: Network contributes ~5% of problem

Is CPU the bottleneck?
Evidence: iowait 22% (significant), idle 10% (not much)
Result: CPU contributes ~60% of problem
(CPU waiting on I/O is actually I/O problem)

Primary bottleneck: I/O (disk + CPU waiting on I/O) = 65%
Secondary: CPU efficiency = 25%
Tertiary: Network = 10%

Observation: Even though job is "CPU-bound" (training),
the actual limitation is memory/I/O interactions
(loading batches is slower than CPU can process)
```

### Task 4C: Suggested Optimizations

```
Based on 4-hour time and 65% I/O bottleneck:

Optimization 1: Larger batch size (ML optimization)
Why: Bigger batches = fewer memory/I/O ops
Expected impact: 8-12% speedup
(Reduces I/O operations by batching)

Optimization 2: Prefetch data (application tuning)
Why: Load next batch while computing current batch
Expected impact: 10-15% speedup
(Hide I/O latency behind computation)

Optimization 3: Lustre striping increase (storage tuning)
Why: Distribute reads across more OSSs
Expected impact: 5-8% speedup
(Parallel reads from multiple servers)

Combined: 8-12% + 10-15% + 5-8% = 23-35% speedup
(Exceeds 25% target easily!)

Note: These are cumulative, not additive
(1.08 × 1.12 × 1.06 = 1.28 = 28% speedup)
```

### Task 4D: Implementation Plan

```
Timeline:

Week 1: Baseline measurement ✓
- Run job 3 times, measure consistency
- Document current performance

Week 2: Optimization 1 (Batch size)
- Change from batch_size=32 to batch_size=64
- Retest, measure impact
- If no regression, keep change

Week 3: Optimization 2 (Prefetching)
- Enable data prefetch in training loop
- Retest, measure incremental impact
- Verify no memory issues (larger batch + prefetch = more memory)

Week 4: Optimization 3 (Storage tuning) + Validation
- Increase Lustre stripe count from 2 to 4 OSSs
- Retest on existing data (stripe only affects new files)
- Run full test suite for regressions

Risk assessment:
- Can we rollback? YES (just revert code changes)
- Will affect others? Minimal (storage change affects all, but benign)
- Need downtime? NO (all changes online)

Go decision: YES (low risk, high benefit)
```

---

## Key Takeaways from Week 8 Case Studies

### Crisis Response (Lesson 8.1)

```
The disk-full incident teaches:
1. Most incidents are simple (diagnosis in <15 min)
2. Quick checks find root cause (not complex debugging)
3. Communication matters as much as technical fix
4. Prevention beats firefighting (retention policies)
5. Stakeholder updates build confidence
```

### Multi-Layer Crisis (Lesson 8.2)

```
The cloud migration incident teaches:
1. Symptoms don't point directly to root cause
2. Multiple failures can compound (cascading)
3. Multi-team coordination is essential
4. MTU, context switching, DNS are often hidden issues
5. Root cause is rarely obvious on first look
```

### Performance Optimization (Lesson 8.3)

```
The optimization campaign teaches:
1. Measure before optimizing (baseline is essential)
2. Focus on bottleneck (biggest impact per effort)
3. Optimize longest phase first (not easy wins)
4. Verify each change has intended effect
5. Batch operations are high-ROI (metadata reduction)
```

---

## L1 TSE Readiness Checklist

### Can you handle a crisis?

```
☑ Gather information quickly (first 15 min)
☑ Perform triage efficiently (quick wins first)
☑ Diagnose root cause systematically
☑ Communicate with stakeholders
☑ Implement fix with minimal risk
☑ Verify resolution
☑ Document for next time

If checked: Ready for production support
```

### Can you debug complex issues?

```
☑ Use multiple tools simultaneously (network, storage, perf)
☑ Correlate findings from different layers
☑ Identify root cause among multiple symptoms
☑ Escalate appropriately when stuck
☑ Work effectively with other teams
☑ Prioritize troubleshooting efforts

If checked: Ready for senior incident response
```

### Can you optimize systems?

```
☑ Establish baselines (before and after measurement)
☑ Identify real bottlenecks (not assumptions)
☑ Suggest targeted improvements (not random changes)
☑ Implement safely with validation
☑ Measure impact accurately
☑ Document changes for future reference

If checked: Ready for performance engineering
```

---

## Common L1 TSE Real-World Incidents

**Incident Type 1: Disk Full** (40% of incidents)
- Root cause: Backup, old logs, cache not cleaned
- Time to fix: <30 minutes
- Tools: df, du, find
- Prevention: Quota, retention, alerts

**Incident Type 2: Performance Degradation** (30% of incidents)
- Root cause: High load, configuration change, hardware issue
- Time to fix: 30 min - 4 hours
- Tools: iostat, top, perf, tcpdump
- Prevention: Monitoring, capacity planning, testing

**Incident Type 3: Connectivity/Access** (20% of incidents)
- Root cause: Network, firewall, permissions, service down
- Time to fix: 15 min - 2 hours
- Tools: ping, netstat, selinux/iptables, systemctl
- Prevention: Network testing, firewall rules, service monitoring

**Incident Type 4: Data/Corruption** (5% of incidents)
- Root cause: Bug, hardware failure, incomplete write
- Time to fix: Hours to days
- Tools: fsck, recovery tools, backup restore
- Prevention: RAID, regular backups, checksums

**Incident Type 5: Cloud/Integration Issues** (5% of incidents)
- Root cause: Latency, API limits, credentials, config
- Time to fix: 1-4 hours
- Tools: tcpdump, AWS CLI, time measurement
- Prevention: Testing, metrics, pre-flight checks

---

Powered by UQS v1.8.5-C
