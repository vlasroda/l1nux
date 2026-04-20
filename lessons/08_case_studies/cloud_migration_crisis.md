# Lesson 8.2: Cloud Migration Gone Wrong - Multi-Layer Crisis

**Time:** 25-30 minutes
**Why it matters:** Complex incidents involve multiple failing systems simultaneously. This lesson teaches systematic root cause analysis when everything appears broken
**Skills:** Multi-system debugging, correlation analysis, vendor coordination, rollback decisions

---

## The Crisis Scenario

**Event Timeline:**

```
Monday 11 AM: Cloud migration from on-prem Lustre to AWS S3 starts
Monday 2 PM: Initial 10TB copied, looks good
Monday 3 PM: Performance degrades (jobs taking 2x longer)
Monday 4 PM: Errors start appearing in application logs
Monday 5 PM: 25% of jobs failing with "connection timeout"
Monday 6 PM: VP Production calls, asks: "Should we roll back?"
```

**What's at stake:**
```
- 500 jobs running (batch processing)
- Deadline: Tuesday 9 AM (some jobs won't finish)
- Partially migrated (10TB on S3, 40TB still on-prem)
- Users confused (can't tell which system is slow)
- Multiple teams investigating (app, network, storage)
```

---

## The Complexity

Unlike the disk-full crisis (one cause), this has **overlapping issues:**

```
┌─────────────────────────────────────────┐
│ Observed symptoms:                      │
│ ├─ Performance degraded (2x slower)    │
│ ├─ Some jobs timing out (connection)   │
│ ├─ Network utilization high             │
│ ├─ AWS egress cost spiking             │
│ └─ On-prem storage CPU at 95%          │
└─────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│ Possible causes (all contributing):     │
│ ├─ S3 latency higher than on-prem      │
│ ├─ Network bandwidth limited           │
│ ├─ MTU mismatch (fragmentation)        │
│ ├─ DNS resolution slow                 │
│ ├─ On-prem storage overloaded         │
│ └─ AWS API throttling                  │
└─────────────────────────────────────────┘
```

---

## Initial Response (First 30 Minutes)

### Multi-Team Triage

```
Team                    Responsibility
─────────────────────────────────────
Network Team           Bandwidth, MTU, routing
Storage Team (On-prem) Lustre health, capacity
AWS Team               S3 configuration, API limits
App Team               Job errors, timeout settings
L1 Support (YOU)       Coordinate, synthesize findings
```

**Coordination meeting (5 minutes):**
```
You: "Describe what each team is seeing"

Network team: "Bandwidth 9.5Gbps out of 10Gbps available
              Packets from AWS to on-prem: 50% loss!
              MTU is 1500 bytes (standard)"

Storage team: "Lustre CPU at 95%, clients at 100 connections
              Disk I/O normal (not saturated)
              No errors in lustre logs"

AWS team: "S3 latency normal (20ms average)
          Request count high (50K req/sec)
          No throttling detected in API"

App team: "Jobs on S3 taking 2x longer than on-prem
          Timeout: 30 seconds (some requests take 45sec)
          Failures only for large object access (>100MB)"

You: "Hmm... let me check something"
```

---

## Root Cause Analysis

### Issue 1: Packet Loss (Network Layer)

**Discovery:**
```bash
# Network team reported 50% packet loss
# This is the killer

mtr -c 100 to-aws
# Shows packet loss on one hop

ping -M do -s 1472 aws-endpoint
# 1472 = 1500 MTU - 28 byte IP/ICMP header
# If this fails: MTU mismatch

ping -M do -s 8972 aws-endpoint
# Bigger packet
# If this fails too: Different problem (real loss)
```

**Root Cause:**
```
MTU Mismatch discovered:
- On-prem network: 1500 bytes (Ethernet standard)
- AWS VPC: 1500 bytes (also standard)
- BUT: Company added firewall/proxy at office
  Firewall MTU: 1480 bytes (non-standard!)

When job tries to send 1500-byte packet:
- Firewall: "Can't fit through! Fragment it"
- Fragmentation: Causes 50% loss with this firewall
- Result: Retransmissions (slow) or timeouts (failed)
```

**Fix (Immediate):**
```bash
# Reduce MTU on AWS side to match company network
aws ec2 modify-network-interface-attribute \
    --network-interface-id eni-xxxxx \
    --mtu 1480

# Or: Fix firewall MTU (proper solution)
ssh firewall
ip link set eth0 mtu 1500
# OR access web console and adjust MTU
```

**Verification:**
```bash
ping -M do -s 1452 aws-endpoint
# 1452 = 1480 - 28
# Should succeed

# Jobs should run faster
# Packet loss: 50% → 0%
```

### Issue 2: On-Prem Storage Overload

**Discovery:**
```bash
# Storage team reported high CPU (95%)
# But disk I/O normal?
# This is weird - usually they correlate

ssh lustre-server
top -b -n 1 | head -20
# Shows: CPU threads all busy

vmstat 1 5
# cs (context switches): Very high!
# This means too many threads contending

lctl list_nids
# Check how many client connections
# Count: 150 simultaneous connections (too many!)
```

**Root Cause:**
```
Application behavior changed with S3:

OLD (on-prem):
- Each job reads file once → closes connection
- 50 jobs running = 50 connections (peak)
- Predictable, low overhead

NEW (S3):
- Job reads from S3 (slow), retries on timeout
- Then fallback reads from on-prem
- Result: DOUBLED requests (retries + fallback)
- 50 jobs × 2 (retry+fallback) = 100+ connections
- 150 connections total (spikes)

Lustre hit connection limit:
- Default max connections: 128
- Was exceeded
- New connections queued
- Context switching overhead explodes
```

**Fix (Immediate):**
```bash
# Increase connection limit
ssh lustre-server
lctl set_param -P \
    ldlm.namespaces.*.lock_limit=1000

# OR: Tune application retry logic
# Edit job config: Reduce retry count
# Old: Retry 10 times
# New: Retry 3 times (fail faster if S3 is slow)
```

### Issue 3: DNS Resolution Slow

**Discovery:**
```bash
# Some jobs report "connection timeout"
# But ping is fine?
# DNS might be the issue

time nslookup s3-endpoint.amazonaws.com
# Result: 2.5 seconds (normal is <100ms)

# DNS query count
dig s3-endpoint.amazonaws.com
# Compare with on-prem
dig lustre-server
# On-prem DNS: 0.05 seconds
```

**Root Cause:**
```
AWS migration added DNS resolution layer:

OLD (on-prem):
- Direct IP: 10.0.0.50
- No DNS lookup
- Instant

NEW (AWS):
- Domain name: s3-us-east-1.amazonaws.com
- AWS DNS: 8.8.8.8 (external)
- Company corporate DNS: 8.8.4.4 (resolver)
- Resolution path: client → corporate DNS → AWS DNS
- Latency: 2.5 seconds (!)

Why slow?
- DNS queries cross company WAN (100ms)
- Corporate DNS is in datacenter A
- Jobs run from datacenter B
- Each job startup does fresh DNS query
- 500 jobs × 2.5 sec = 1250 seconds of DNS queries!
```

**Fix (Immediate):**
```bash
# Cache DNS locally
ssh job-runner-server
echo "nameserver 8.8.8.8" > /etc/resolv.conf
# Direct external DNS (lower latency)

# OR: Pin IP in /etc/hosts (faster)
echo "52.88.25.75 s3-us-east-1.amazonaws.com" >> /etc/hosts
# AWS S3 endpoint IP
```

**Better Fix (Long-term):**
```bash
# Local DNS cache server
apt-get install dnsmasq
# Configure to cache queries
# Result: First query 2.5s, subsequent <1ms
```

---

## Cascading Failures Understanding

**What happened:**

```
Step 1: MTU mismatch causes 50% packet loss
        ↓
Step 2: Jobs retry requests (exponential backoff)
        ↓
Step 3: Retry + fallback to on-prem = 2x connections
        ↓
Step 4: Lustre connection limit exceeded
        ↓
Step 5: Context switching overhead spikes (CPU 95%)
        ↓
Step 6: Slower responses cause timeout in application
        ↓
Step 7: Application adds more retries (makes it worse)
        ↓
Result: Death spiral (more retries = more load = more failures)
```

**Why diagnosis was hard:**
```
- Symptoms (slow, timeout) didn't directly point to MTU
- Network team focused on bandwidth (OK), missed MTU
- Storage team saw CPU high, but didn't see it was context-switching (not disk)
- Each layer was individually working (S3 OK, Lustre OK, network OK)
- But **interaction** between layers caused failure
```

---

## Communication During Crisis

### Update 1 (30 min in)

```
Status: ACTIVE DIAGNOSIS - Multiple issues identified

Findings so far:
1. MTU mismatch on network path to AWS (FOUND)
2. High context switches on Lustre (FOUND)
3. Slow DNS resolution to S3 (FOUND)

Impact:
- These issues compound each other
- Jobs retrying → more load → context switch spikes
- Cascading effect makes situation worse

Actions in progress:
- Adjusting MTU (AWS side, 10 min)
- Tuning Lustre connection limit (5 min)
- Local DNS caching (15 min)

ETA: Performance should improve within 30 minutes
     Full resolution: 1 hour
```

### Update 2 (60 min in)

```
Status: PARTIALLY RESOLVED

Fixed:
✓ MTU adjusted from 1500 to 1480 (packet loss: 50% → 5%)
✓ Lustre connection limit increased (context switch spike reduced)
✓ DNS caching configured locally (resolution: 2.5s → 0.1s)

Current performance:
- Jobs on S3: 1.5x slower than on-prem (was 2x)
- Timeouts: 5% (was 25%)
- Network: 9.5Gbps utilization (was saturated)

Remaining issue:
- 5% timeouts still happening (investigating)
- Likely: Large file reads still slow on S3

Root cause of large file slowness:
- S3 default buffer: 8MB chunks
- Should be 128MB chunks for large files
- Requires app-side fix (S3 library config)

Next: App team will tune S3 client configuration
ETA: Full resolution by tomorrow morning
```

---

## Decision Point: Roll Back?

**VP asks at 6 PM:** "Should we roll back to all on-prem?"

**Analysis:**
```
Option A: Roll back (revert to on-prem)
Pro: ✓ Immediate stability (known working state)
Pro: ✓ Jobs complete by deadline
Con: ✗ Migration effort wasted
Con: ✗ Would need to re-do later
Con: ✗ Demoralizes team

Option B: Continue fixing (hybrid approach)
Pro: ✓ Learn what's wrong
Pro: ✓ Build robustness for future
Pro: ✓ Still might meet deadline
Con: ✗ Risk of continued issues
Con: ✗ Jobs might not finish by deadline

Option C: Staged approach
Pro: ✓ Run old jobs on on-prem
Pro: ✓ Run new jobs on S3 (smaller, safer)
Pro: ✓ Both strategies running
Con: ✗ Operational complexity
Con: ✗ Still need to stabilize both

RECOMMENDATION: Option B
Reasoning:
- Issues are understood and fixable
- Not structural problems (like data corruption)
- Each fix brings us closer to working solution
- Deadline risk is acceptable (if needed, partial rollback)
- Learning value high for future migrations
```

**VP approval:** "Continue fixing. Let me know hourly."

---

## Post-Crisis Technical Review

```
Time breakdown:
30 min: Initial diagnosis (identify 3 issues)
20 min: Fix MTU (immediate impact)
15 min: Fix Lustre connection limit
15 min: Fix DNS caching
Total: 80 minutes to major improvement

Root causes (actual):
1. Pre-migration checklist missing MTU step
2. Application assumptions (1 connection per job) violated by retries
3. Architecture didn't consider DNS as critical path

Prevention for next migration:
- Pre-migration network testing (MTU, latency, loss)
- Load testing (simulate actual retry behavior)
- Architecture review (DNS, connection pooling, caching)
- Staged migration (small → large)
- Fallback plan (keep on-prem, not decommission immediately)
```

---

## Key Lessons

**Technical:**
1. MTU mismatch causes invisible packet loss
2. Cascading failures require systems thinking
3. Each layer might be "working" but interaction fails
4. Retry logic can create death spiral
5. DNS is critical infrastructure (needs caching)

**Operational:**
1. Staged migration safer than big bang
2. Keep old system running (fast rollback)
3. Parallel running helps find issues
4. Communication is life support during crisis
5. Post-mortems prevent repeat incidents

**For L1 TSE:**
1. Multi-layer diagnosis requires multi-team coordination
2. Symptoms don't always point to root cause
3. Context switching, packet loss, DNS are often hidden causes
4. Fix the worst bottleneck first (usually network or storage limits)
5. Verify each fix had intended impact

---

## Challenge Questions

**1. Diagnosis Order:** What should you check first?
```
A) AWS S3 configuration
B) Lustre logs
C) Network packet loss
D) Application code

Answer: C (network is foundation, most fundamental)
```

**2. Context Switching:** Why did CPU spike from 10% to 95%?
```
A) More load
B) Too many threads competing (context switching)
C) Encryption overhead
D) Memory leak

Answer: B (each connection = context switch)
```

**3. Cascading Failure:** What breaks the cycle?
```
A) Roll back (stop retries)
B) Fix network (stop packet loss)
C) More Lustre CPUs
D) AWS caching

Answer: B (network is root cause of cascade)
```

---

Powered by UQS v1.8.5-C
