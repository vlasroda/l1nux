# Lesson 3.2: Network Diagnostics (Refresher)

**Time:** 25-30 minutes
**Why it matters:** When storage is slow or unreachable, network diagnostics tells you if it's a network problem or something else
**Skills:** Use ping, traceroute, netstat/ss, tcpdump, iperf to diagnose network issues

---

## Quick Review

### Diagnostic Workflow

```
Is the server reachable?
  ├─ No → PING (ICMP)
  │        ├─ Works? Server is up but something is blocking me
  │        └─ Times out? Network path is broken or server is down
  │
  └─ Yes → TRACEROUTE (find where path breaks)
           ├─ Hops work until point X
           ├─ Then timeouts or high latency
           └─ Problem is usually near timeout point

Can I access the service?
  ├─ TELNET/NC to port
  ├─ Check if port is open
  └─ If open but service not responding: service problem

Is the network saturated?
  ├─ IFTOP/NETHOGS
  ├─ See what's using bandwidth
  └─ If 80%+: Congestion is issue

Detailed packet inspection?
  └─ TCPDUMP (last resort, complex)
```

---

## Key Concepts

### 1. ICMP (Internet Control Message Protocol)

Used by: PING, TRACEROUTE

```
ICMP Echo Request: "Are you there?"
ICMP Echo Reply: "Yes, I'm here"

Time to receive reply = latency

Issues:
- Firewalls often block ICMP
- If ping fails, doesn't mean server is down
  (could just be firewall)
- Use other methods if ping fails

Example:
$ ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=45.2ms
```

### 2. TTL (Time To Live)

```
Each router decrements TTL by 1
When TTL = 0, packet is discarded

Default: 64 for Linux
If after 64 hops you haven't reached destination: unreachable

Example:
Start: TTL = 64
Hop 1: TTL = 63
Hop 2: TTL = 62
...
Hop 64: TTL = 0 (packet dropped)

Traceroute uses this:
- Sends packets with TTL=1 (gets response from router 1)
- Then TTL=2 (gets response from router 2)
- And so on...
- Until reaching destination
```

### 3. TCP Handshake (for connection checks)

```
To connect to port 22:

Client: SYN (sync, "I want to connect")
  ↓
Server: SYN-ACK ("OK, here I am")
  ↓
Client: ACK ("Great, I've connected")
  ↓
Connection established

Time for this handshake = connection latency

If server doesn't respond:
- Port not open
- Service not listening
- Firewall dropping packets
```

### 4. Network Saturation Indicators

```
CPU usage on network device
- If CPU 100%: Device is overloaded

Interface RX/TX errors
- Errors often indicate cable/hardware issues
- Some errors are normal but >1% is bad

Throughput vs capacity
- If using >80% of link capacity: Congested
- If using >95%: Definitely saturated

Packet loss
- Should be 0% on LAN
- If >0.1%: Network issue
```

### 5. Common Diagnostic Tools

**PING** - ICMP reachability
```
ping -c 4 server.com
- Simplest test
- Tests basic connectivity
- Blocked by firewalls sometimes
```

**TRACEROUTE** - Show path to host
```
traceroute server.com
- Shows each hop
- Identifies where path breaks
- Useful for understanding routing
```

**SS (Socket Statistics)** - Network connections
```
ss -tlnp
- Shows listening ports
- Shows established connections
- Shows which process owns connection
```

**NETSTAT** - Old version of SS (deprecated)
```
netstat -tlnp
- Same as ss -tlnp
- Still works on older systems
- Being phased out
```

**IPERF** - Bandwidth measurement
```
iperf3 -c server.com
- Measures throughput
- Tests TCP/UDP performance
- Requires iperf3 on both ends
```

**IFTOP** - Bandwidth by host
```
iftop
- Shows real-time traffic
- Displays top talkers
- Useful for finding hogs
```

**TCPDUMP** - Packet capture
```
tcpdump -i eth0 host server.com
- Captures raw packets
- Advanced analysis
- Useful for debugging protocol issues
```

---

## Essential Commands

### Basic Connectivity Tests

```bash
# Ping (ICMP)
ping -c 1 server.com                    # One ping
ping -c 4 server.com                    # Four pings
ping -i 0.2 -c 10 server.com            # Fast pings (0.2s interval)

# TCP connection test (better than ping if ICMP blocked)
nc -zv server.com 22                    # Test SSH port
telnet server.com 22                    # Old method
ss -tlnp | grep :22                     # See if listening locally

# DNS resolution
nslookup server.com                     # Query DNS
dig server.com                          # More detailed
host server.com                         # Another option
```

### Trace Path to Host

```bash
# Show route to destination
traceroute server.com                   # Standard
traceroute -m 30 server.com             # 30 hop max
mtr server.com                          # Continuous traceroute (better)
mtr -c 10 server.com                    # 10 probes then exit

# Example output:
#  1  gateway.local (192.168.1.1)  1.2 ms
#  2  isp.router (203.0.113.1)    10.5 ms
#  3  backbone.1 (198.19.1.1)     15.3 ms
# ...
# 15  server.com (93.184.216.34)   45.2 ms
```

### Check Open Ports and Services

```bash
# What's listening?
ss -tlnp                                # TCP listeners
ss -ulnp                                # UDP listeners
ss -tlnp | grep :3306                   # Specific port

# All connections (listening + established)
ss -tanp                                # All TCP
ss -uanp                                # All UDP

# Show connections to specific host
ss -tanp | grep server.com

# Example output:
# LISTEN 0 128 *:22 *:* users:(("sshd",pid=1234))
# ESTAB  0 0 192.168.1.100:49152 10.0.0.50:22 users:(("ssh",pid=5678))
```

### Measure Network Performance

```bash
# Bandwidth test
iperf3 -c server.com                    # Connect to server
# (Requires: iperf3 -s on server end)

# Expected output:
# [ ID] Interval       Transfer     Throughput
# [  4]   0.00-10.00  sec  1.18 GBytes   1.01 Gbps

# Real-time traffic by host
iftop -n                                # Numeric IPs
iftop -i eth0                           # Specific interface

# Traffic by process
nethogs                                 # Show by process
nethogs -d eth0                         # Specific device
```

### Monitor Network Health

```bash
# Interface statistics
ip -s link                              # All interfaces with stats
ethtool -S eth0                         # Detailed stats for eth0

# Look for:
# - RX errors: 0 (should be zero)
# - TX errors: 0 (should be zero)
# - Dropped: 0 (bad sign if high)
# - Overrun: 0 (buffer overflow)

# Watch for changes
watch -n 1 'ip -s link | grep -A 3 eth0'

# Netstat stats
netstat -s                              # Protocol stats
netstat -s | grep -i error             # Focus on errors
```

### Advanced Packet Analysis

```bash
# Capture all traffic
sudo tcpdump                            # Default interface

# Capture to specific host
sudo tcpdump -i eth0 host server.com   # Just this host

# Save to file
sudo tcpdump -w capture.pcap           # Binary format
tcpdump -r capture.pcap                # Read saved capture

# Specific protocol
sudo tcpdump tcp port 22               # SSH only
sudo tcpdump udp port 53               # DNS only

# Analysis
# (Open in wireshark for GUI analysis)
# Or use filters for CLI
```

---

## Interactive Exercise: Network Diagnostics

**Task 1: Test basic connectivity**
```bash
# Ping a reliable server
ping -c 4 8.8.8.8

# Record:
# - All pings succeeded?
# - What's the latency (time=XXms)?
# - Any packet loss?
```

**Task 2: Trace route to a server**
```bash
# Trace to a public server
traceroute google.com

# Record:
# - How many hops?
# - Are all hops responding?
# - Where's the latency increasing?
```

**Task 3: Check your listening services**
```bash
# See what's open
ss -tlnp

# Record:
# - Which ports are listening?
# - Which process owns each port?
# - Any unexpected services?
```

**Task 4: Test connectivity to a specific port**
```bash
# Try SSH port
nc -zv localhost 22

# Expected: "Connection succeeded" or "refused"
# If refused: SSH not running (or not on port 22)
```

**Task 5: Check for network issues**
```bash
# Quick health check
ip -s link show

# Look for:
# - RX errors: Should be 0 or very low
# - TX errors: Should be 0 or very low
# - Dropped: Should be 0
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Traceroute timeout:** You get:
   ```
    1  gateway.local  1.2ms
    2  isp.router     10.5ms
    3  backbone.1     *  *  *
    4  backbone.2     45ms
   ```
   What does the `*  *  *` mean?
   ```
   A) Router 3 is down
   B) Network is broken at that point
   C) Router 3 doesn't respond to ICMP (but traffic flows)
   D) Packet loss is occurring
   ```

2. **Connection latency:** You measure:
   - Ping to storage server: 2ms
   - But SSH takes 5 seconds to connect
   - What's the issue?
   ```
   (Explain possible causes)
   ```

3. **Bandwidth vs Latency:** Copying 1GB file over:
   - 1 Gbps network, 1ms latency
   - 100 Mbps network, 0.1ms latency
   - Which is faster?
   ```
   A) 1 Gbps (bandwidth matters more)
   B) 100 Mbps (latency compensates)
   C) Same speed
   D) Depends on file system
   ```

4. **Interface Errors:** You see:
   ```bash
   RX errors 0  RX dropped 15  RX overruns 0
   ```
   What does this mean?
   ```
   A) Network cable is bad
   B) Driver not handling traffic fast enough
   C) Network perfectly fine (errors = 0)
   D) Buffer overflow
   ```

5. **Port Blocked:** You can ping server but can't SSH. Port 22 should be open:
   ```bash
   ping server.com  # Works
   ssh server.com   # Times out
   nc -zv server.com 22  # Times out
   ```
   What's wrong?
   ```
   (Explain in order of likelihood)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Connection to storage is slow"

```bash
# Step 1: Basic connectivity
ping -c 4 storage.server
# If fails: Network issue, not server
# If works: Latency OK, connection works

# Step 2: Check bandwidth
iperf3 -c storage.server
# Compare with expected (1 Gbps = ~125 MB/s)
# If much lower: Network congestion

# Step 3: Check saturation
iftop
# Is link 80%+ utilized?
# If yes: Someone else is using bandwidth

# Step 4: Detailed path
traceroute storage.server
# Are there latency increases at certain hops?
# Indicates congestion at that point
```

### Scenario 2: "NFS mount times out"

```bash
# Step 1: Can you reach server?
ping nfs.server
# If no: Network broken

# Step 2: Is NFS port open?
nc -zv nfs.server 2049
# If no: Firewall or service not running

# Step 3: Check DNS (often overlooked!)
time nslookup nfs.server
# If slow: DNS might be hanging NFS mount

# Step 4: Try IP directly
mount -t nfs 10.0.0.50:/export /mnt
# If works with IP but not hostname: DNS issue
```

### Scenario 3: "Backup speed inconsistent"

```bash
# Step 1: Check available bandwidth
iftop
# Is link saturated?
# Monitor during backup

# Step 2: Check latency
mtr -c 20 backup.server
# Is latency stable or jittery?

# Step 3: Check for packet loss
mtr -c 100 backup.server
# Look for % loss
# Should be 0% on LAN

# Step 4: Check congestion window
netstat -s | grep -i tcp
# Look for retransmits: should be low
```

### Scenario 4: "Can't resolve storage.company.com"

```bash
# Step 1: Check /etc/resolv.conf
cat /etc/resolv.conf
# Is nameserver configured?

# Step 2: Query directly
nslookup storage.company.com 8.8.8.8
# Does public DNS work?

# Step 3: Query corporate DNS
nslookup storage.company.com 10.0.0.1
# Does corporate DNS work?

# Step 4: Check timing
time dig storage.company.com
# How long does resolution take?
# >1 second = slow DNS
```

---

## Key Takeaways

1. **Ping tests ICMP reachability** — useful but can be blocked
2. **Traceroute shows where path breaks** — identifies problem network segment
3. **ss/netstat shows listening services** — verify port is open
4. **Bandwidth = throughput**, latency = delay — both matter for different reasons
5. **Systematic diagnosis** — start simple (ping), then get more specific
6. **Firewalls can block ICMP but allow TCP** — use nc/telnet to test TCP
7. **DNS failures break everything** — always test DNS when troubleshooting

---

## How Ready Are You?

Can you explain these?
- [ ] The difference between ICMP and TCP connection tests
- [ ] What a traceroute `*  *  *` means
- [ ] How to test if a port is open
- [ ] The difference between bandwidth and latency
- [ ] How to find which process is using bandwidth

If you checked all boxes, you're ready for Lesson 3.3.

---

Powered by UQS v1.8.5-C
