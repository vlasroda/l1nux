# Week 3 Exercises: Networking for Storage

---

## Exercise 3.1: TCP/IP Fundamentals

### Problem 1: Analyze Your Network

```bash
# Task 1A: Get your IP and netmask
ip addr show

# Record: IP, netmask, CIDR notation
# Example: 192.168.1.100/24

# Task 1B: Calculate subnet details
# Network address: 192.168.1.0
# Broadcast: 192.168.1.255
# Usable hosts: 192.168.1.1 - 192.168.1.254

# Task 1C: Check routing
ip route show

# Record: default gateway, local routes
```

### Problem 2: Understand Network Protocols

**Scenario:** You need to choose between TCP and UDP for a real-time monitoring service.

```bash
# Task 2A: When would you use each?
# TCP: Reliable needed, data loss unacceptable (NFS, SSH)
# UDP: Speed matters more, some loss OK (DNS, video)

# Task 2B: Check which services use which
ss -tlnp | head -10   # TCP listeners
ss -ulnp | head -10   # UDP listeners

# Record: What's listening on TCP vs UDP?
```

### Problem 3: Port Analysis

```bash
# Task 3A: What's listening?
ss -tlnp

# Task 3B: Identify each service
# Port 22: SSH
# Port 80: HTTP
# Port 443: HTTPS
# Port 3306: MySQL
# Port 2049: NFS
# Port 445: SMB
```

### Challenge: Subnetting Math

```bash
You have IP 10.20.30.100 with netmask 255.255.254.0

Questions:
1. What's the CIDR notation? (/23)
2. How many hosts per subnet? (512)
3. Network address? (10.20.30.0)
4. Broadcast address? (10.20.31.255)
5. First usable host? (10.20.30.1)
```

---

## Exercise 3.2: Network Diagnostics

### Problem 1: Test Connectivity

```bash
# Task 1A: Ping external server
ping -c 4 8.8.8.8

# Record: response times, packet loss

# Task 1B: Trace path to server
traceroute 8.8.8.8

# Record: number of hops, latency per hop
```

### Problem 2: Check Services

```bash
# Task 2A: What's listening locally?
ss -tlnp

# Task 2B: Test SSH port
nc -zv localhost 22

# Expected: "succeeded" or "refused"
```

### Problem 3: Measure Performance

```bash
# Task 3A: Bandwidth test (if iperf3 available)
iperf3 -c iperf.server.com 2>/dev/null || echo "iperf not available"

# Task 3B: Baseline latency
ping -c 100 8.8.8.8 | tail -1

# Record: min, avg, max, stddev
# What's normal? LAN < 5ms, Internet 10-100ms
```

### Problem 4: Monitor Real-Time Traffic

```bash
# Task 4A: See current network activity
ip -s link  # Show interface stats

# Task 4B: Top talkers (if iftop available)
iftop -n -s 1 2>/dev/null || echo "iftop not available"

# Task 4C: By process (if nethogs available)
nethogs 2>/dev/null || echo "nethogs not available"
```

### Challenge: Diagnose Connection Issue

**Scenario:** Can't SSH to server.example.com

```bash
# Step 1: Can you ping?
ping -c 1 server.example.com

# Step 2: Does DNS work?
nslookup server.example.com

# Step 3: Is SSH port open?
nc -zv server.example.com 22

# Step 4: Try with IP (if you know it)
nc -zv 10.0.0.50 22

# Analysis: Which failed?
# - Ping failed: Network unreachable
# - DNS failed: Can't resolve name
# - Port open but SSH fails: Service issue
# - Works with IP but not hostname: DNS problem
```

---

## Exercise 3.3: Common Network Issues

### Problem 1: MTU Troubleshooting

```bash
# Task 1A: Check current MTU
ip link show eth0
# Look for: mtu 1500 (standard) or 9000 (jumbo)

# Task 1B: Test standard MTU
ping -M do -s 1472 8.8.8.8
# 1472 + 28 header = 1500 bytes

# Task 1C: Test jumbo frames
ping -M do -s 8972 8.8.8.8
# 8972 + 28 header = 9000 bytes

# If second fails: Network doesn't support jumbo

# Task 1D: Fix MTU (if needed)
# ip link set dev eth0 mtu 9000  (requires root)
```

### Problem 2: DNS Issues

```bash
# Task 2A: Check nameservers
cat /etc/resolv.conf

# Task 2B: Test DNS speed
time nslookup google.com

# Record: Did it take > 1 second?

# Task 2C: If slow, try public DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
time nslookup google.com

# Is it faster?
```

### Problem 3: Firewall Blocking

```bash
# Task 3A: Check firewall status
sudo ufw status 2>/dev/null || echo "ufw not enabled"

# Task 3B: Try to connect to port
nc -zv 8.8.8.8 443  # Google HTTPS

# Result:
# - "succeeded": Port is open
# - "refused": Service rejected
# - "timed out": Firewall is dropping
```

### Problem 4: NFS Mount Issues

**Scenario:** mount -t nfs server:/export /mnt hangs

```bash
# Systematic diagnosis:

# 1. Can reach server?
ping -c 1 server

# 2. Is NFS port open?
nc -zv server 2049

# 3. Is RPC portmapper open?
nc -zv server 111

# 4. Is DNS slow?
time nslookup server

# 5. Try with IP?
mount -t nfs 10.0.0.50:/export /mnt

# Analysis:
# - Ping fails: Network issue
# - Port hangs: Firewall blocking
# - DNS slow (> 1s): Add to /etc/hosts
# - IP works but name doesn't: DNS issue
```

### Challenge: Complete Network Audit

```bash
#!/bin/bash

echo "=== Network Audit ==="
echo ""

echo "1. Interface Status:"
ip link show | grep -E "^[0-9]|LOWER_UP"

echo ""
echo "2. IP Configuration:"
ip addr show | grep "inet "

echo ""
echo "3. Default Route:"
ip route show | grep default

echo ""
echo "4. DNS Server:"
cat /etc/resolv.conf | grep nameserver | head -1

echo ""
echo "5. Listening Ports:"
ss -tlnp | head -10

echo ""
echo "6. Connectivity Test:"
ping -c 1 8.8.8.8 && echo "Internet: UP" || echo "Internet: DOWN"

echo ""
echo "7. Network Performance:"
ping -c 10 8.8.8.8 | tail -1

echo ""
echo "8. Firewall Status:"
sudo ufw status 2>/dev/null || echo "UFW not available"
```

---

## Self-Assessment

**After Week 3, you should be able to:**

- [ ] Explain IP addresses, netmasks, and CIDR notation
- [ ] Understand when TCP vs UDP is appropriate
- [ ] Diagnose connectivity issues systematically
- [ ] Identify and fix MTU problems
- [ ] Diagnose DNS failures
- [ ] Detect firewall blocks
- [ ] Troubleshoot NFS mount issues
- [ ] Measure network performance
- [ ] Create network diagnostic scripts

---

Powered by UQS v1.8.5-C
