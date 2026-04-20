# Lesson 3.3: Common Network Issues (Storage Context)

**Time:** 25-30 minutes
**Why it matters:** Certain network issues are very common in storage environments. Knowing typical symptoms and fixes saves diagnostic time
**Skills:** Diagnose and fix MTU issues, DNS timeouts, firewall blocks, NFS timeouts

---

## Quick Review

### Why Network Issues Manifest as Storage Problems

```
User experience: "Storage is slow" or "backup hangs"
Root cause: Network issue, not storage

Examples:
- High latency → NFS hangs (processes in D state)
- Low bandwidth → Slow transfers
- DNS failure → Mount hangs
- MTU mismatch → Packet fragmentation
- Firewall → Connection times out
```

---

## Key Concepts

### 1. MTU (Maximum Transmission Unit) Issues

**What is MTU?**
```
MTU = Largest packet size that can be sent on network
Default: 1500 bytes (Ethernet standard)
Jumbo frames: 9000 bytes (datacenter networks)

Packet structure:
[Ethernet header 14B][IP header 20B][TCP header 20B][Data XXB]
[                    1500 bytes total                        ]
                                       [Data only = 1500-54 = 1446B]
```

**MTU mismatch problems:**
```
Scenario: Your device MTU=1500, network MTU=9000
1. You send 1500-byte packets
2. Network passes them through
3. Receiver expects 9000-byte packets
4. Fragmentation/performance issues

Scenario: Your device MTU=9000, network MTU=1500
1. You send 9000-byte packets
2. Network MTU is 1500, must fragment
3. Fragmentation overhead (~6x more packets)
4. Performance tanks, CPU spikes

Symptoms:
- Works at first, then suddenly slow
- Specific file sizes cause problems
- Works over local, slow over WAN
- CPU high despite low bandwidth usage
```

**How to diagnose:**
```bash
# Check your MTU
ip link show
# Look for: mtu 1500 (or 9000)

# Test MTU to destination
# Method 1: ping with DF flag
ping -M do -s 1472 server.com    # 1472 + 28 header = 1500
ping -M do -s 8972 server.com    # 8972 + 28 header = 9000

# If fails at 8972: Network doesn't support jumbo frames

# Method 2: traceroute with large packets
traceroute --mtu server.com      # Test MTU along path

# Fix (if needed):
ip link set dev eth0 mtu 9000    # Change to 9000
```

### 2. DNS Timeouts

**Symptoms:**
```
mount nfs.server:/export /mnt
# Hangs for 30 seconds, then error

Or:
systemctl status service
# Hangs when resolving hostname

Cause: DNS server is slow or unreachable
```

**How to diagnose:**
```bash
# Time the DNS lookup
time nslookup nfs.server
# > 1 second = problem

# Check which server is being used
cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4

# Test directly to that server
dig nfs.server @8.8.8.8

# If slow: DNS is the issue
```

**Solutions:**
```bash
# Use IP instead of hostname (temporary)
mount -t nfs 10.0.0.50:/export /mnt

# Fix DNS (permanent)
# Option 1: Use faster DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# Option 2: Add entry to /etc/hosts (quick fix)
echo "10.0.0.50 nfs.server" | sudo tee -a /etc/hosts

# Option 3: Configure systemd-resolved
sudo systemctl restart systemd-resolved
# Edit /etc/systemd/resolved.conf
# Change DNS=[8.8.8.8 8.8.4.4]

# Verify DNS is working
nslookup nfs.server
# Should resolve quickly
```

### 3. Firewall Blocks

**Symptoms:**
```
nc -zv server.com 2049    # Times out (never connects)
telnet server.com 22      # Hangs

But server is up:
ping server.com           # Works
ssh user@10.0.0.50        # Works (using IP)
```

**Root cause:**
```
Firewall drops connection attempts to port from your IP
Server never sees the attempt
Client waits 30 seconds, then times out
```

**How to diagnose:**
```bash
# Check if port responds locally
ss -tlnp | grep :2049
# If listening: Service is running

# Check firewall status
sudo ufw status                 # UFW (Ubuntu)
sudo iptables -L -n             # iptables (detailed)
sudo firewall-cmd --list-all    # firewalld (RHEL)

# Check if specific port is allowed
sudo ufw show added
# Look for port 2049 or 2049/tcp

# Test from different source
# If works from one IP but not another: Firewall rule

# Check firewall logs
journalctl -u ufw          # UFW logs
journalctl SYSLOG_IDENTIFIER=kernel | grep DROP  # Firewall drops
```

**Solutions:**
```bash
# Add firewall rule for NFS
sudo ufw allow 2049/tcp
sudo ufw allow 111/udp                 # RPC portmapper

# Or specific source
sudo ufw allow from 10.0.0.0/24 to any port 2049

# Apply immediately
sudo ufw reload

# Verify
sudo ufw show added
```

### 4. NFS Timeout Issues

**Common causes:**
```
1. DNS (covered above)
2. Firewall (covered above)
3. Network latency (RTT > 100ms)
4. Network packet loss (> 0.1%)
5. Server overloaded
6. Missing exports
7. RPC portmapper issues
```

**Symptoms:**
```
mount hangs for 30 seconds
Then: mount.nfs: timeout set for ...
Or: mount.nfs: mount(2): Permission denied
Or: processes stuck in state 'D' (uninterruptible)
```

**How to diagnose systematically:**
```bash
# 1. Check connectivity
ping nfs.server
# Response time? (should be <50ms for LAN)

# 2. Check NFS port
nc -zv nfs.server 2049
# Port open?

# 3. Check RPC portmapper
nc -zv nfs.server 111
# RPC should be available

# 4. Check DNS
nslookup nfs.server
# Slow?

# 5. Try with IP address
mount -t nfs 10.0.0.50:/export /mnt
# Works? If so, DNS issue

# 6. Check network path
mtr -c 20 nfs.server
# Packet loss? High latency?

# 7. Check server exports
# (From server):
showmount -e
# Are exports configured?

# 8. Check firewall on client
sudo ufw show added
# Is NFS allowed?

# 9. Check firewall on server
# (From server):
sudo ufw status
# Is NFS port 2049 allowed?
```

### 5. Bandwidth Saturation

**Symptoms:**
```
File transfer slow
- Expected: 100 MB/s
- Actual: 10 MB/s
```

**Root causes:**
```
1. Network link is saturated
2. Multiple transfers competing
3. Client or server CPU maxed
4. Disk I/O bottleneck (not network)
```

**How to diagnose:**
```bash
# Check network utilization
iftop
# See if link is 80%+ used

# Check which processes use bandwidth
nethogs
# Identify culprits

# Check CPU
top
# Is CPU 100%?

# Check disk I/O
iostat -x 1 5
# Is %util 100% on source/dest?

# Measure available bandwidth
iperf3 -c server.com
# Compare with actual transfer speed
```

**Solutions:**
```bash
# Identify what's saturating link
iftop
# Wait for peak usage

# Kill unnecessary traffic
# Or:

# Prioritize storage traffic (QoS)
# (Requires kernel/network admin privileges)

# Use multiple connections
# rsync has --parallel option
# Or run multiple transfers

# Compress if possible
rsync -z                    # Enable compression
scp -C                      # Enable compression for SCP
```

### 6. Interface Configuration Issues

**Common issues:**
```
Wrong IP address
Wrong netmask
Wrong gateway
Interface down
Duplex/speed mismatch
```

**How to check:**
```bash
# Check interface status
ip link show
# Should see: <BROADCAST,MULTICAST,UP,LOWER_UP>

# Check IP assignment
ip addr show
# Should show IP address with netmask

# Check gateway
ip route show
# Should see default route via XXX

# Check actual link speed
ethtool eth0
# Look for: Speed: 1000Mb/s
# Or verify it matches network

# Check duplex
ethtool eth0 | grep Duplex
# Should be: Full
```

**Typical issues:**
```
Problem: Auto-negotiation fails
Result: Stuck at 10 Mbps instead of 1 Gbps
Fix: Force speed/duplex
$ ethtool -s eth0 speed 1000 duplex full autoneg on

Problem: Wrong netmask
Result: Can reach gateway but not network
Fix: Correct netmask
$ ip addr add 10.0.1.100/24 dev eth0

Problem: No default gateway
Result: Can reach local network but not remote
Fix: Add gateway
$ ip route add default via 10.0.1.1
```

---

## Essential Commands

### Quick Diagnostics

```bash
# Is the network working?
ping -c 1 8.8.8.8 && echo "Network OK" || echo "Network down"

# Can you reach storage server?
ping -c 1 storage.server && echo "Server reachable" || echo "Unreachable"

# Is NFS port open?
nc -zv storage.server 2049 && echo "NFS open" || echo "Blocked"

# Is DNS working?
nslookup storage.server >/dev/null && echo "DNS OK" || echo "DNS broken"

# Network at capacity?
iftop -n -s 1              # Show top talkers once and exit
```

### Network Configuration Check

```bash
# Full network config
ip addr show
ip link show
ip route show
ss -tlnp

# Or one-liner:
echo "=== Network Status ===" && \
ip addr | grep "inet " && \
echo && \
echo "=== Routes ===" && \
ip route show && \
echo && \
echo "=== Listening ===" && \
ss -tlnp | head -20
```

### Diagnose Specific Issues

```bash
# MTU problems
ping -M do -s 1472 server.com       # Standard
ping -M do -s 8972 server.com       # Jumbo frames

# DNS slow?
time nslookup nfs.server
time dig nfs.server

# Bandwidth available
iperf3 -c server.com -t 10          # 10 second test

# Latency
ping -c 100 server.com | grep rtt   # Min/avg/max/stddev
mtr -c 50 server.com                # Continuous ping with routing
```

---

## Interactive Exercise: Troubleshoot Network Issue

**Scenario:** Your NFS mount is hanging

**Task 1: Basic connectivity**
```bash
ping nfs.server.local

# What did you see?
# - Times out? Server unreachable
# - Works? Server is up, something else is wrong
```

**Task 2: Check NFS port**
```bash
nc -zv nfs.server.local 2049

# What did you see?
# - Connection succeeded? Port is open
# - Connection refused? Service not running or firewall blocked
# - Times out? Firewall is dropping packets
```

**Task 3: Check DNS**
```bash
time nslookup nfs.server.local

# What did you see?
# - Resolved quickly? DNS is OK
# - Timed out? DNS is broken or slow
# - took 30 seconds? This might be the hang!
```

**Task 4: Try with IP**
```bash
# If you know the IP:
nc -zv 10.0.0.50 2049

# If this works but hostname doesn't: DNS is the issue
```

**Task 5: Create a checklist**
```bash
echo "NFS Troubleshooting Checklist:"
echo "[ ] Can ping server"
echo "[ ] Port 2049 is open"
echo "[ ] DNS resolves server name"
echo "[ ] No firewall blocking"
echo "[ ] Network latency < 50ms"
echo "[ ] No packet loss"

# Add checkmarks as you verify each
```

---

## Challenge Questions

**Answer without looking them up:**

1. **MTU Issue:** Your jumbo frames (9000) are being fragmented to 1500 on the network. Performance tanks. Why?
   ```
   (Explain the fragmentation cost)
   ```

2. **NFS Hang:** Mount succeeds, but file operations hang. What do you check first?
   ```
   A) Server disk space
   B) Server CPU
   C) Network packet loss
   D) NFS protocol version
   ```

3. **Slow DNS:** NFS mount takes 30 seconds to timeout. You fix DNS to be faster. Mount now fails immediately with "Connection refused". What happened?
   ```
   (Explain the sequence of events)
   ```

4. **Firewall Rule:** You allow port 2049 but NFS still times out. What are you missing?
   ```
   (Hint: NFS uses two protocols)
   ```

5. **Bandwidth vs Latency:** You have:
   - 10 Mbps link, 1ms latency
   - vs 1 Gbps link, 100ms latency
   - Which is better for NFS?
   ```
   A) 10 Mbps (latency matters more for NFS)
   B) 1 Gbps (throughput always wins)
   C) Same performance
   D) Depends on file size
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "NFS mount hangs"

```bash
# Systematic approach:

# 1. Test each layer
ping nfs.server              # Network
nc -zv nfs.server 2049      # NFS port
nslookup nfs.server         # DNS

# 2. If DNS works but is slow
time nslookup nfs.server    # Measure
# > 1 second? Use /etc/hosts

# 3. If port times out
sudo iptables -L -n | grep 2049  # Check firewall
sudo ufw status                   # Check ufw

# 4. Add firewall rule
sudo ufw allow 2049/tcp
sudo ufw allow 111/udp

# 5. Verify and retry
mount -t nfs nfs.server:/export /mnt
```

### Scenario 2: "Network is slow"

```bash
# Determine the bottleneck:

# 1. Check available bandwidth
iperf3 -c remote.server
# If < expected: Network is saturated

# 2. Check latency
ping -c 100 remote.server | grep rtt
# If > 100ms: High latency (normal for WAN)

# 3. Check who's using bandwidth
iftop
# Look for culprits

# 4. Check if MTU mismatch
ping -M do -s 1472 remote.server
ping -M do -s 8972 remote.server

# If second fails: MTU issue
# Solution: Match MTU on both sides
ip link set dev eth0 mtu 1500  # Standard
# OR
ip link set dev eth0 mtu 9000  # Jumbo frames
```

### Scenario 3: "Firewall is blocking storage"

```bash
# Identify blocked ports:

# Try to connect
nc -zv storage.server 2049    # Times out = blocked
nc -zv storage.server 111     # RPC portmapper

# Check firewall rules
sudo ufw show added
# Look for NFS-related rules

# Add rules
sudo ufw allow 2049/tcp
sudo ufw allow 111/udp
sudo ufw allow from 10.0.0.0/24 port 2049

# Or use specific source
sudo ufw allow from 192.168.1.100 to any port 2049

# Reload
sudo ufw reload

# Verify
sudo ufw show added
```

---

## Key Takeaways

1. **MTU mismatch causes fragmentation** — 6x more packets and CPU spike
2. **DNS failures cause hangs** — check DNS when mount/connections hang
3. **Firewalls silently drop** — use nc/telnet to detect blocks
4. **NFS uses RPC** — must allow both port 2049 and 111
5. **Latency > 100ms hurts NFS** — processes enter "D" state waiting
6. **Systematic approach** — ping → port check → DNS check → firewall check

---

## How Ready Are You?

Can you explain these?
- [ ] Why MTU mismatch causes performance problems
- [ ] What to check first when NFS mount hangs
- [ ] How to diagnose firewall blocks
- [ ] What port NFS uses (hint: not just 2049)
- [ ] When latency vs throughput matters more

If you checked all boxes, you're ready for Week 3 exercises.

---

Powered by UQS v1.8.5-C
