# Week 3 Solutions: Networking for Storage

---

## Lesson 3.1: TCP/IP Fundamentals

### Challenge Questions Answers

**1. Subnetting**
```
Answer: A) 10.0.1.0

Explanation:
IP: 10.0.1.50
Netmask: /24 (255.255.255.0)

/24 means last 8 bits are host portion
So network = first 24 bits = 10.0.1
Host = 50

Network address = 10.0.1.0 (all host bits = 0)
```

**2. TCP vs UDP for Large Copy**
```
Answer: B) Throughput (bandwidth)

Explanation:
For large bulk transfer:
- Need to send many MB/GB
- Latency (RTT) matters less for throughput
- Throughput formula: Bandwidth = (window size / RTT) + packet size
- But for bulk transfers, throughput dominates

For NFS (small random I/O):
- Latency matters because each request waits for reply
- Can't pipeline requests as much
- High latency = many processes in D state
```

**3. SMB Port Issues**
```
Answer: Multiple possibilities in order of likelihood:

1. Firewall blocking port 445
   - Solution: Allow 445/tcp in firewall

2. Server not running SMB service
   - Check: systemctl status smbd
   - Solution: Start service

3. User permissions
   - Solution: Verify credentials, check shares

4. Network unreachable
   - Solution: Test connectivity (ping, traceroute)

5. SMB service configured on different port
   - Check: ss -tlnp | grep smb
```

**4. DNS Resolution Failure**
```
Answer: B) DNS is broken

Explanation:
- nslookup hangs for 30 seconds then times out
- This is classic DNS timeout
- Means: Nameserver not responding

Solution:
1. Check what nameserver is configured
   cat /etc/resolv.conf

2. Try public DNS
   echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

3. Test again
   nslookup nfs.server.com

If it works: Your nameserver is broken
If it still fails: Network is broken
```

**5. Traceroute Timeouts**
```
Answer: Limits to what you can conclude:

Can conclude:
- Path breaks or becomes unreachable around hop 15
- Hops before timeout are reachable
- Network is working at least partway

Cannot conclude:
- Server is down (could be firewall)
- Path is completely broken (traffic might still flow on TCP)
- Problem is at hop 15 (could be beyond)

Reason:
- Timeouts in traceroute often = ICMP blocked
- But TCP traffic might still flow
- Use TCP-based tools (SSH, HTTP) to verify
```

---

## Lesson 3.2: Network Diagnostics

### Challenge Questions Answers

**1. Traceroute * * * Meaning**
```
Answer: C) Router doesn't respond to ICMP (but traffic flows)

Explanation:
The * * * means router didn't respond to ICMP probes
This is common and usually OK

Why routers might not respond to ICMP:
- Firewall drops ICMP
- Router prioritizes data packets over ICMP
- Security policy (ICMP disabled)

Important: Doesn't mean network is broken
- Data packets might still flow through that router
- ICMP is diagnostic, not data

Workaround:
- Use TCP traceroute instead
- tcptraceroute -p 80 host.com (uses TCP)
```

**2. SSH Connection Delay**
```
Answer: Multiple possible causes (in order of likelihood):

1. SSH server has slow authentication
   - Check: sshd might be doing reverse DNS
   - Fix: Add UseDNS no to /etc/ssh/sshd_config

2. Network path is congested
   - Some routers have congestion
   - Latency spikes during peak hours

3. Client side issue
   - StrictHostKeyChecking
   - Key exchange negotiation taking time

4. Firewall doing stateful inspection
   - Deep packet inspection adds latency

Diagnosis:
$ time ssh -vv host
# Look for "Offering public key authentication" delay
# If there's a pause before auth: server side
```

**3. Bandwidth vs Latency**
```
Answer: A) 1 Gbps (bandwidth matters more)

Explanation:
For large file copy:

1 Gbps, 1ms latency:
- Throughput = 1 Gbps = 125 MB/s
- Latency doesn't matter for bulk transfer
- Copy 1GB file = 8 seconds

100 Mbps, 0.1ms latency:
- Throughput = 100 Mbps = 12.5 MB/s
- Lower latency helps but doesn't overcome bandwidth
- Copy 1GB file = 80 seconds

Winner: 1 Gbps link wins by 10x

Bandwidth is the limiting factor for large transfers
Latency matters more for interactive/small requests (NFS, databases)
```

**4. RX Dropped Interpretation**
```
Answer: B) Driver not handling traffic fast enough

Explanation:
RX dropped means:
- Network interface received packets
- But driver/kernel couldn't process them in time
- Packets were discarded in RX ring buffer

Causes:
- NIC driver too slow
- Kernel not scheduling network processing fast enough
- Old/buggy driver
- Hardware limitation

This is different from:
- RX errors = line errors, CRC errors (cable issue)
- RX overruns = ring buffer overflow (hardware overloaded)

Solutions:
- Update NIC driver
- Check interrupt handling
- Increase ring buffer size: ethtool -G eth0
- Verify hardware compatibility
```

**5. Port Blocked Troubleshooting**
```
Answer: In order of likelihood:

1. Firewall on client is blocking (most common)
   - Check: sudo ufw status
   - Allow: sudo ufw allow out 22/tcp

2. Firewall on server is blocking
   - Check: sudo ss -tlnp | grep :22
   - If listening: SSH is running
   - Then check server firewall: sudo ufw status
   - Allow: sudo ufw allow 22/tcp

3. Intermediate firewall/router blocking
   - Traceroute to identify where path breaks
   - Contact network admin

4. SSH not running on server
   - Check: systemctl status sshd (on server)
   - Start: systemctl start sshd

5. Wrong port
   - SSH might be on different port
   - Check: sudo ss -tlnp | grep ssh
```

---

## Lesson 3.3: Common Network Issues

### Challenge Questions Answers

**1. MTU Fragmentation Cost**
```
Answer: MTU fragmentation is extremely expensive

Why:
Scenario: 9000-byte packets fragmented to 1500
- 1 packet becomes: 9000 / 1500 = 6 packets
- 6× more packets = 6× more processing
- 6× more interrupt handling
- 6× more context switches

CPU impact:
- CPU usage spikes 4-10x
- Latency increases significantly
- Throughput decreases despite same bandwidth

Example:
- Before: 400 MB/s throughput, 10% CPU
- After fragmentation: 50 MB/s throughput, 80% CPU

Solution:
- Match MTU on both sides
- Either all 1500 (standard) or all 9000 (jumbo)
- NOT mixing them
```

**2. NFS Hang Diagnosis**
```
Answer: C) Network packet loss

Explanation:
NFS is sensitive to network conditions:
- High packet loss → retransmissions → hang
- Fragmentation → lost packets
- Congestion → dropped packets

But prioritize by likelihood:
1. First check: Firewall allowing port 2049
2. Second: DNS is slow
3. Third: Packet loss on network

Diagnosis order:
1. nc -zv server 2049      # Port open?
2. time nslookup server    # DNS speed?
3. mtr -c 50 server        # Packet loss?
4. ping -s 1472 server     # MTU OK?

If all pass but still hangs:
- Might be server-side NFS issue
- Or extremely high latency (>200ms)
```

**3. DNS Fix Makes Mount Fail**
```
Answer: The sequence of events:

Original behavior:
1. User tries: mount -t nfs server:/export /mnt
2. mount command hangs for 30 seconds
3. mount command resolves "server" via DNS
4. DNS finally responds (after 30s timeout)
5. mount fails with timeout

After DNS fix:
1. User tries: mount -t nfs server:/export /mnt
2. mount command hangs again
3. mount command tries to resolve "server" via DNS
4. DNS responds immediately
5. Mount connects... and gets "Connection refused"

Reason for "Connection refused":
- NFS port 2049 is BLOCKED by firewall
- Before, the slow DNS hid the firewall issue
- Now DNS is fast, firewall block is exposed

Solution:
1. Allow port 2049 in firewall
   sudo ufw allow 2049/tcp
   sudo ufw allow 111/udp   # RPC portmapper

2. Reload firewall
   sudo ufw reload

3. Retry mount
   mount -t nfs server:/export /mnt
```

**4. Missing RPC Portmapper**
```
Answer: You're missing RPC portmapper (port 111)

Explanation:
NFS uses two components:
1. NFS protocol = port 2049
2. RPC portmapper = port 111

RPC is needed for:
- Initial connection negotiation
- Port discovery
- Service registration

If you allow only port 2049:
- Client can't reach portmapper
- Client can't negotiate
- Mount times out silently

Solution (both required):
sudo ufw allow 2049/tcp
sudo ufw allow 111/udp     # Essential!
sudo ufw allow 111/tcp     # Also often needed

Verify:
nc -zv server 111         # Portmapper
nc -zv server 2049        # NFS

Both must be open
```

**5. MTU vs Latency for NFS**
```
Answer: A) 1 Mbps is better

Explanation:
NFS performance depends on:

Latency dominance (for NFS):
- NFS requests are small (typically < 4KB)
- One request → wait for response → next request
- Can't pipeline without complexity
- High latency = many requests in flight = full network window

10 Mbps, 1ms latency:
- Can sustain ~1.25 MB/s on small files
- Low latency means quick responses
- Good for interactive use

1 Gbps, 100ms latency:
- High latency means slow response
- Even with high bandwidth, limited by RTT
- Formula: Max throughput = Window size / RTT
- Needs large TCP window to compensate
- Not typical for NFS

In practice:
- NFS performance dominated by latency
- Bandwidth matters less for small I/O
- Storage networks are built for low latency + high bandwidth
```

---

## Exercise Solutions

### Exercise 3.1: TCP/IP Fundamentals

**Problem 1 - Expected Output:**
```bash
$ ip addr show
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.100/24
    inet6 fe80::200:5eff:fe00:5301/64

Your recording should show:
- IP: 192.168.1.100
- Netmask: /24
- Broadcast: 192.168.1.255
```

**Problem 2 - Protocol Choice:**
```
TCP: NFS, SSH, HTTP/HTTPS, databases (reliability needed)
UDP: DNS, NTP, DHCP (speed/simplicity matters)

Listening services vary by system:
- Common: SSH (22/tcp), HTTP (80/tcp), DNS (53/udp)
- Business: Database (3306), SMB (445)
- Storage: NFS (2049)
```

**Challenge - Subnetting Math:**
```
IP: 10.20.30.100
Netmask: 255.255.254.0 (/23)

Answers:
1. CIDR: /23
2. Hosts per subnet: /23 = 2^(32-23) = 2^9 = 512 addresses
   Usable: 510 (minus network and broadcast)
3. Network: 10.20.30.0 (last bit of second octet = 0)
4. Broadcast: 10.20.31.255
5. First usable: 10.20.30.1
6. Last usable: 10.20.31.254

/23 covers two /24 networks (30-31)
```

---

### Exercise 3.2: Network Diagnostics

**Problem 1 - Expected Results:**
```bash
$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=45.2ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=119 time=44.8ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=119 time=45.1ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=119 time=45.0ms

rtt min/avg/max/stddev = 44.8/45.0/45.2/0.15 ms

Analysis:
- All 4 packets succeeded (0% loss)
- Latency ~45ms (typical for internet)
- Very consistent (stddev 0.15 = good)
- Network is working well
```

**Problem 2 - SSH Test:**
```bash
$ nc -zv localhost 22
Connection to localhost (127.0.0.1) 22 port [tcp/ssh] succeeded!

Result: SSH is running and listening
```

**Problem 3 - Performance Baseline:**
```bash
$ iperf3 -c iperf.server.com
Connecting to host iperf.server.com, port 5201
[  4] local 192.168.1.100 port 49152 connected to 10.0.0.50 port 5201
[ ID] Interval       Transfer     Throughput
[  4]   0.00-10.00  sec  1.18 GBytes  1.01 Gbps

Interpretation:
- 1.01 Gbps throughput
- This is near theoretical maximum for 1 Gbps link
- Network is working well
```

---

### Exercise 3.3: Common Network Issues

**Problem 4 - NFS Diagnosis:**
```
Systematic approach:

1. ping server
   PING server (10.0.0.50) 56(84) bytes of data.
   64 bytes from 10.0.0.50: icmp_seq=1 time=2.5ms
   → Server is reachable

2. nc -zv server 2049
   Connection to server (10.0.0.50) 2049 port [tcp/nfs] succeeded!
   → NFS port open

3. nc -zv server 111
   Connection to server (10.0.0.50) 111 port [tcp/rpc] succeeded!
   → RPC portmapper open

4. time nslookup server
   real    0m0.052s
   → DNS is fast

5. Try with hostname
   mount -t nfs server:/export /mnt
   → Should work if all above pass

If mount still fails:
- Check if export exists: showmount -e server
- Check if export is accessible to your IP
- Check server firewall: sudo ufw status (on server)
```

---

## Real-World Scenarios

### Scenario: "NFS is Slow"

```bash
# Diagnostic sequence:

# 1. Latency test
ping -c 100 nfs.server | tail -1
# If > 100ms: Latency is issue

# 2. Bandwidth test
iperf3 -c nfs.server
# Compare with expected for your network

# 3. Check for loss
mtr -c 50 nfs.server
# Should be 0% loss on LAN

# 4. Check if saturated
iftop -n -s 1
# Is link 80%+ used?

# 5. Check file size effects
# Copy different sizes and measure
time dd if=/dev/zero of=/data/test bs=1M count=100
# Large sequential should be faster

# 6. If issue confirmed
# - Upgrade network link
# - Reduce latency (network redesign)
# - Add more NFS servers (spread load)
```

### Scenario: "Storage Mount Hangs"

```bash
# Root cause checklist:

# 1. Basic connectivity
ping -c 1 storage.server
# If fails: Network unreachable

# 2. DNS
time nslookup storage.server
# If slow: Add to /etc/hosts

# 3. Port open
nc -zv storage.server 2049
# If times out: Firewall blocking

# 4. RPC portmapper
nc -zv storage.server 111
# If blocked: Add to firewall

# 5. Try with IP
mount -t nfs 10.0.0.50:/export /mnt
# If works: DNS or hostname issue
# If fails: Firewall or export issue

# Fix priority:
# 1. Firewall rules
# 2. DNS
# 3. RPC portmapper
# 4. Network connectivity
```

---

## Key Takeaways

1. **Systematic diagnosis** — start simple, get specific
2. **MTU mismatch = fragmentation** = CPU spike + slowness
3. **DNS failures manifest as hangs** — always test DNS
4. **Firewall blocks silently** — use TCP tests (nc, telnet)
5. **NFS needs both port 2049 AND 111** — don't forget RPC
6. **Latency vs throughput** — NFS cares about latency
7. **Verify before blaming** — check each layer systematically

---

Powered by UQS v1.8.5-C
