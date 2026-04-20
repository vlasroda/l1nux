# Lesson 3.1: TCP/IP Fundamentals (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Storage systems are networked. Understanding TCP/IP is essential for diagnosing slow storage, timeouts, and connectivity issues
**Skills:** Understand network layers, TCP vs UDP, IP addressing, subnetting, ports and services

---

## Quick Review

### The Network Stack (OSI Model Simplified)

```
Application Layer (HTTP, SSH, NFS, SMB)
    ↓
Transport Layer (TCP, UDP)
    ↓
Internet Layer (IP, ICMP)
    ↓
Link Layer (Ethernet, WiFi)
    ↓
Physical (cables, wireless signals)
```

Each layer adds its own header to data as it goes down.

---

## Key Concepts

### 1. IP Addressing

**IPv4:** Standard dotted-decimal notation
```
192.168.1.100
   |    |   |    |
   |    |   |    └─ Host (0-255)
   |    |   └────── Subnet (0-255)
   |    └───────── Class B private range
   └──────────────Class C for networks


Ranges:
10.0.0.0 - 10.255.255.255 (Class A private)
172.16.0.0 - 172.31.255.255 (Class B private)
192.168.0.0 - 192.168.255.255 (Class C private)
127.0.0.1 (Loopback - localhost)
```

**IPv6:** 128-bit addresses (getting more common)
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334

Simplified:
2001:db8:85a3::8a2e:370:7334

::1 (Loopback - localhost)
```

### 2. Subnetting

**Netmask:** Defines which part of IP is network, which is host

```
IP: 192.168.1.100
Netmask: 255.255.255.0  (or /24 in CIDR notation)

Network portion: 192.168.1.0 (where hosts live)
Host portion: .100 (this specific host)
Broadcast: 192.168.1.255 (all hosts on this network)

Usable hosts: 192.168.1.1 to 192.168.1.254 (2-254)
Total: 254 addresses per /24 network
```

**CIDR notation:**
```
/32 = Single host (255.255.255.255)
/24 = 256 addresses (255.255.255.0) — common for LANs
/16 = 65,536 addresses (255.255.0.0) — large network
/8  = 16 million addresses (255.0.0.0)
/30 = 4 addresses (255.255.255.252) — point-to-point links
```

### 3. TCP vs UDP

**TCP (Transmission Control Protocol):**
```
Characteristics:
  - Connection-oriented (handshake first)
  - Reliable (guaranteed delivery, order)
  - Slower (error checking overhead)
  - Ordered (data arrives in sequence)

Use cases:
  - SSH (secure shell)
  - HTTP/HTTPS (web)
  - SMB/CIFS (Windows file sharing)
  - Most storage protocols

Overhead:
  - 3-way handshake before data
  - ACK for every packet
  - Retransmission if lost
```

**UDP (User Datagram Protocol):**
```
Characteristics:
  - Connectionless (no handshake)
  - Unreliable (no guarantee)
  - Fast (minimal overhead)
  - May arrive out of order

Use cases:
  - DNS queries (fast, simple)
  - NTP (time sync)
  - Video streaming (some loss OK)
  - DHCP (bootstrap)

Overhead:
  - Send immediately, no setup
  - No ACK needed
  - Sender doesn't know if received
```

### 4. Ports and Services

**Well-known ports:**
```
22   SSH (secure shell)
80   HTTP (web)
443  HTTPS (secure web)
53   DNS (domain name service)
123  NTP (network time protocol)
3306 MySQL database
5432 PostgreSQL database
6379 Redis
111  RPC (Remote Procedure Call - used by NFS)
2049 NFS (Network File System)
445  SMB (Windows file sharing)
```

**Port ranges:**
```
1-1023       Well-known (requires root to open)
1024-49151   Registered ports (applications use)
49152-65535  Dynamic/private (temporary connections)
```

### 5. Common Protocols for Storage

**NFS (Network File System):**
```
Stateless (v3) or stateful (v4)
Uses: RPC portmapper (port 111)
Performance: Sensitive to latency
Common issue: Network timeouts cause "D" state processes

Example mount:
mount -t nfs server:/export/path /local/mount
```

**SMB/CIFS (Windows sharing):**
```
Stateful, Windows-native
Port: 445 (modern) or 139 (legacy)
Performance: Better with Windows servers
Useful for: Cross-platform sharing

Example mount:
mount -t cifs //server/share /mount -o username=user,password=pass
```

**iSCSI (IP Storage):**
```
Storage over network
Port: 3260
Performance: Near local storage
Common in: Enterprise storage systems

Appears as local block device (/dev/sdX)
```

### 6. Network Latency vs Throughput

**Latency (milliseconds):**
- How long for first byte to arrive
- Critical for: NFS, databases, interactive protocols
- Typical WAN: 10-100ms
- Typical LAN: 0.1-1ms
- SSD: 0.1ms, HDD: 10ms

**Throughput (megabytes/second):**
- How much data per second
- Critical for: Bulk transfers, backups
- 1 Gbps network = ~125 MB/s (theoretically)
- 10 Gbps network = ~1,250 MB/s

**Example bottleneck scenario:**
```
Storage is fast: 1GB/s local
Network is slow: 1 Gbps = 125 MB/s
Result: Network limits you to 125 MB/s
Latency doesn't matter much for bulk transfer
But latency matters for small NFS requests
```

### 7. DNS (Domain Name System)

**How it works:**
```
Host: "Please resolve example.com"
  ↓
Recursive resolver (your ISP or 8.8.8.8)
  ↓
Root nameserver: "Ask TLD server about .com"
  ↓
TLD server: "Ask authoritative server for example.com"
  ↓
Authoritative server: "example.com is 93.184.216.34"
  ↓
Result cached and returned

Time: Usually 10-100ms per lookup
Issue: Slow DNS breaks everything (NFS mounts, backups, etc.)
```

---

## Essential Commands

### Check IP Configuration

```bash
# Modern method (iproute2)
ip addr                         # Show all IP addresses
ip link                         # Show network interfaces
ip route                        # Show routing table

# Legacy method (still works)
ifconfig                        # Show interfaces (deprecated)
netstat -r                      # Show routing table (deprecated)

# Example output:
$ ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.100/24 brd 192.168.1.255
    inet6 fe80::1/64
```

### Test Network Connectivity

```bash
# Ping (ICMP)
ping -c 1 8.8.8.8              # Send one ping
ping -c 4 example.com          # Four pings
# Look for: response time, packet loss

# Traceroute (show path to host)
traceroute example.com         # Show hops
traceroute -m 30 example.com   # Custom max hops

# Check specific port
telnet server.com 22           # Test SSH port
nc -zv server.com 22           # Test with netcat (better)

# Measure bandwidth
iperf3 -c server.com           # Client mode
# (Requires iperf3 server running on remote)
```

### Check Open Ports and Services

```bash
# Modern method
ss -tlnp                        # TCP listening, numeric, processes
ss -ulnp                        # UDP listening

# Legacy method
netstat -tlnp                   # Same as ss -tlnp
netstat -tulnp                  # Both TCP and UDP

# Example output:
$ ss -tlnp
State  Recv-Q Send-Q Local:Port Foreign:Port
LISTEN 0      128    *:22      *:*        processes=(("sshd",pid=1234))
LISTEN 0      128    *:80      *:*        processes=(("nginx",pid=5678))
LISTEN 0      128    127.0.0.1:3306 *:*   processes=(("mysqld",pid=9012))
```

### Test DNS

```bash
# Query DNS
nslookup example.com            # Simple query
dig example.com                 # Detailed query (better)
host example.com                # Another option

# Example output:
$ dig example.com
example.com. 300 IN A 93.184.216.34

# Check which nameserver you're using
cat /etc/resolv.conf

# Change nameserver temporarily
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

### Monitor Network Performance

```bash
# Real-time traffic
iftop                          # Top by bandwidth
nethogs                        # Top by process

# Interface stats
ethtool -S eth0                # Interface statistics
ip -s link                     # Interface stats (all)

# Packet analysis (advanced)
tcpdump -i eth0 host server.com  # Capture packets
wireshark                      # GUI packet analyzer
```

---

## Interactive Exercise: Network Diagnostics

**Task 1: Check your network configuration**
```bash
# View your IP
ip addr show

# Record:
# - Your IP address
# - Your netmask/CIDR
# - Your default gateway
```

**Task 2: Test basic connectivity**
```bash
# Ping a public DNS
ping -c 1 8.8.8.8

# Expected:
# "bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=45.2ms"

# If fails: Check network cable, interface status
```

**Task 3: Check local network services**
```bash
# See what's listening
ss -tlnp

# Record:
# - SSH on port 22
# - Any web servers
# - Any storage services (NFS on 2049, SMB on 445)
```

**Task 4: Test DNS**
```bash
# Resolve a domain
dig google.com

# Record:
# - Does it resolve?
# - How long did it take?
# - What nameserver was used?
```

**Task 5: Check routing**
```bash
# Show your routing table
ip route

# Look for:
# - Default gateway (0.0.0.0)
# - Local network routes
# - Any custom routes
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Subnetting:** You have IP 10.0.1.50/24. What's the network address?
   ```
   A) 10.0.1.0
   B) 10.0.1.50
   C) 10.0.1.255
   D) 10.0.0.0
   ```

2. **TCP vs UDP:** You're copying a large backup over NFS (TCP-based). Which matters more?
   ```
   A) Latency (RTT time)
   B) Throughput (bandwidth)
   C) Packet loss rate
   D) MTU size
   ```

3. **Ports:** You can't access SMB shares. Server says port 445 is open. What's likely wrong?
   ```
   (Explain 2-3 reasons)
   ```

4. **DNS:** Suddenly all NFS mounts hang. You check:
   ```bash
   nslookup nfs.server.com
   # Hangs for 30 seconds, then times out
   ```
   What's the problem?
   ```
   A) Network is down
   B) DNS is broken
   C) NFS server is down
   D) Firewall blocking port 53
   ```

5. **Network Path:** Traceroute to storage server shows 15 hops, last three time out. What does this mean?
   ```
   (Explain what you can and can't conclude)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "NFS mount hanging"

```bash
# Check connectivity to server
ping nfs.server.com

# If ping hangs or times out:
# - Network issue, not NFS

# If ping works, check NFS port
nc -zv nfs.server.com 2049
# If times out: Firewall or service not running

# Check DNS resolution time
time nslookup nfs.server.com
# If slow: DNS might be issue

# Check network to server
traceroute nfs.server.com
# Look for timeouts in path

# Check local NFS mounts
mount | grep nfs
# See which mounts are responsive
```

### Scenario 2: "Slow backup to remote storage"

```bash
# Check network bandwidth
iperf3 -c remote.server

# Check if throughput is below expected
# 1 Gbps line = max 125 MB/s

# Check network latency
ping -c 10 remote.server | grep rtt
# High latency doesn't matter for bulk transfer
# But indicates network congestion

# Check if network is bottleneck
# Monitor local storage speed
dd if=/dev/zero of=/tmp/test bs=1M count=1000

# Compare:
# If local: 500 MB/s
# Remote: 50 MB/s
# Network is the bottleneck
```

### Scenario 3: "Can't reach storage server"

```bash
# Systematic diagnosis:

# 1. Check local network
ip addr
ip route

# 2. Check if server is reachable
ping storage.server

# 3. If unreachable, try gateway
ping (your default gateway)

# 4. Try traceroute to see where path breaks
traceroute storage.server

# 5. Check DNS
nslookup storage.server

# 6. If DNS works but ping fails
# - Server might be down
# - Firewall might block ICMP
# - Try SSH instead: ssh user@storage.server
```

### Scenario 4: "Database queries very slow"

```bash
# Check network latency to DB server
ping database.server | grep time=
# Look for: typical LAN = 1-5ms
# If 50ms+: Network latency is issue

# Check network saturation
iftop
# See if network is maxed out

# Check database connection port
ss -tlnp | grep 3306
# Is database listening on expected port?

# Test connection
mysql -h database.server -u user -p
# If slow to connect: Network issue
# If connected but queries slow: Database issue
```

---

## Key Takeaways

1. **IP addresses and subnets** determine which networks can reach each other
2. **TCP is reliable but slower**, UDP is fast but unreliable
3. **Ports identify services** (SSH=22, HTTP=80, NFS=2049)
4. **Latency matters for interactive** (NFS, databases), throughput matters for bulk transfers
5. **DNS failures break everything** because all hostnames depend on it
6. **Network diagnostics follow a logical path**: ping → traceroute → port check → service check

---

## How Ready Are You?

Can you explain these?
- [ ] What /24 subnet means and which hosts it includes
- [ ] Why TCP is slower than UDP but more reliable
- [ ] What port NFS uses and why it matters
- [ ] The difference between latency and throughput
- [ ] How to diagnose network problems systematically

If you checked all boxes, you're ready for Lesson 3.2.

---

Powered by UQS v1.8.5-C
