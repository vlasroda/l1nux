# Lesson 6.2: tcpdump and Packet Analysis (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Network-level debugging shows exactly what data flows between systems. Essential for storage network troubleshooting
**Skills:** Capture packets, filter by protocol/host, analyze captured data, diagnose network issues

---

## Quick Review

### Network Packet Anatomy

```
Packet structure:
┌─────────────────────────────────────┐
│ Ethernet (L2) - MAC addresses       │
├─────────────────────────────────────┤
│ IP (L3) - IP addresses              │
├─────────────────────────────────────┤
│ TCP/UDP (L4) - Ports                │
├─────────────────────────────────────┤
│ Data (Application)                  │
└─────────────────────────────────────┘

tcpdump captures all layers
```

---

## Key Concepts

### 1. tcpdump Basics

**What it captures:**
```
- Every packet on network interface
- Source and destination
- Protocol type
- Packet flags
- Size
- Data payload (optionally)

Output format:
timestamp source > destination proto flags seqnum ack window size
14:30:45.123456 192.168.1.100.50123 > 10.0.0.50.22 \
  S [SYN] 1234567:1234567(0) win 65535

Breaking down:
- 14:30:45.123456: Timestamp
- 192.168.1.100.50123: Source IP and port
- 10.0.0.50.22: Destination IP and port
- S [SYN]: Packet flag (SYN = connection request)
- 1234567: Sequence number
- 0: Bytes of data
- win 65535: TCP window size
```

**Packet flags:**
```
S = SYN (start connection)
F = FIN (finish/close connection)
R = RST (reset connection)
P = PUSH (send data now)
A = ACK (acknowledge receipt)
.= No flags set (data only)

Common sequences:
SYN → SYN,ACK → ACK = Three-way handshake (connection start)
FIN → ACK → FIN → ACK = Graceful close
RST = Abrupt close/error
```

### 2. Capture Filters

**By host:**
```bash
tcpdump host 192.168.1.100          # Any traffic to/from this IP
tcpdump src 192.168.1.100           # Source only
tcpdump dst 192.168.1.100           # Destination only
tcpdump src net 192.168.0.0/16      # Entire subnet
```

**By port:**
```bash
tcpdump port 22                      # SSH traffic
tcpdump tcp port 22                  # TCP only (exclude UDP)
tcpdump src port 5432               # Traffic from port 5432
tcpdump dst port 2049               # Traffic to NFS port
```

**By protocol:**
```bash
tcpdump tcp                          # TCP only
tcpdump udp                          # UDP only
tcpdump icmp                         # ICMP (ping)
tcpdump arp                          # ARP (address resolution)
```

**Complex filters:**
```bash
tcpdump host 10.0.0.50 and port 2049      # NFS to specific host
tcpdump src 192.168.1.0/24 and dst port 22  # SSH from subnet
tcpdump tcp[tcpflags] & tcp-syn != 0       # Only SYN packets
```

### 3. Analyzing Packet Flow

**Connection handshake:**
```
Client → Server: SYN (connection request)
Server → Client: SYN,ACK (accepted)
Client → Server: ACK (confirmed)
[Connection established, data flows]
Client → Server: FIN (closing)
Server → Client: ACK (acknowledged)
Server → Client: FIN (closing from server)
Client → Server: ACK (confirmed)
```

**Identifying problems:**

```
Problem: Connection refused
Packets show:
SYN → [no response or RST]
Means: Server not listening on that port

Problem: Slow connection
Packets show:
ACK with 0 bytes for long time
Means: Network ready but application not sending data

Problem: Packet loss
Packets show:
Duplicate ACKs (same sequence number)
Means: Retransmitting due to loss

Problem: Timeout
Packets show:
Long gap between packets, then connection reset
Means: Idle timeout triggered
```

### 4. tcpdump Output Formats

**Default (tcpdump)**
```bash
tcpdump -i eth0
# Human-readable format
```

**Verbose**
```bash
tcpdump -v              # More detail
tcpdump -vv             # Even more detail
tcpdump -vvv            # Everything
```

**Save to file**
```bash
tcpdump -w capture.pcap -i eth0
# Can analyze later with wireshark
```

**ASCII output**
```bash
tcpdump -A              # Show payload in ASCII
tcpdump -X              # Hex and ASCII (useful for debugging)
```

### 5. Common Diagnostic Scenarios

**Check if packets reach server:**
```bash
# On server
tcpdump -i eth0 host client.ip and port 2049
# Run this, then try to connect from client
# If nothing shows: packets not reaching (network issue)
# If SYN but no SYN,ACK: server side issue
# If full handshake: network OK, app issue
```

**Check for packet loss:**
```bash
tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0'
# Look for duplicate ACKs (sign of packet loss)
```

**Analyze NFS traffic:**
```bash
tcpdump -i eth0 port 2049
# Monitor NFS requests/responses
# Check if requests/responses are coming through
```

**Check DNS:**
```bash
tcpdump -i eth0 udp port 53
# DNS queries and responses
# If queries go out but no responses: DNS server issue
# If no queries go out: Client not trying DNS
```

---

## Essential Commands

### Quick Captures

```bash
# Capture on specific interface
sudo tcpdump -i eth0

# Specific host
sudo tcpdump host 10.0.0.50

# Specific port
sudo tcpdump port 2049

# NFS traffic
sudo tcpdump port 2049 or port 111

# SSH connections only
sudo tcpdump tcp port 22

# HTTP (port 80)
sudo tcpdump port 80
```

### Save and Analyze

```bash
# Save to file
sudo tcpdump -w capture.pcap host 10.0.0.50

# Read saved capture
sudo tcpdump -r capture.pcap

# Filter saved capture
sudo tcpdump -r capture.pcap 'tcp.flags.syn == 1'

# Count packets
sudo tcpdump -r capture.pcap | wc -l

# See payloads
sudo tcpdump -r capture.pcap -A | less
```

### Verbose Analysis

```bash
# Very detailed output
sudo tcpdump -vvv host 10.0.0.50

# With ASCII payload
sudo tcpdump -vvv -A host 10.0.0.50

# TCP flags analysis
sudo tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'
```

---

## Interactive Exercise: Capture and Analyze

**Task 1: Capture DNS query**
```bash
# Terminal 1: Start capture
sudo tcpdump -i eth0 udp port 53

# Terminal 2: Make DNS query
nslookup google.com

# Terminal 1: Should show:
# DNS query for google.com going out
# DNS response coming back
```

**Task 2: Capture connection to storage**
```bash
# Terminal 1: Start capture
sudo tcpdump -i eth0 host storage.server and port 2049

# Terminal 2: Try to access storage
mount -t nfs storage.server:/export /mnt

# Terminal 1: Should show:
# SYN to server port 2049
# Either SYN,ACK back, or RST if server not listening
```

**Task 3: Analyze connection problem**
```bash
# If connection times out, capture shows:
# SYN → [nothing back]
# Then after timeout:
# SYN → [nothing]
# SYN → [nothing]
# (retries)

# Diagnosis: Server not responding (firewall or service down)
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Packet Flag:** You see `[SYN]` in tcpdump. What does it mean?
   ```
   A) Connection is established
   B) Connection is being closed
   C) Connection is being requested
   D) Data is being sent
   ```

2. **Filter:** How would you capture NFS traffic?
   ```bash
   tcpdump [what goes here?]
   A) port nfs
   B) port 2049
   C) service nfs
   D) protocol nfs
   ```

3. **Problem Analysis:** tcpdump shows SYN but no SYN,ACK coming back. What's wrong?
   ```
   A) Client firewall blocking
   B) Server not listening or firewall blocking
   C) Network unreachable
   D) DNS problem
   ```

4. **Duplicate ACKs:** What do repeated packets with the same ACK number mean?
   ```
   A) Normal traffic
   B) Packet loss / retransmission happening
   C) Connection closing
   D) Flow control
   ```

5. **Filtering:** Write a tcpdump filter for SSH traffic from a specific IP.
   ```
   (Write the command)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "NFS mount times out"

```bash
# Capture traffic to NFS server
sudo tcpdump -i eth0 host storage.server and port 2049

# Try to mount
mount -t nfs storage.server:/export /mnt

# Analyze:
# If you see SYN but no response: Server not listening or firewalled
# If you see handshake but then nothing: Server issue or performance
# If you see no packets at all: Network unreachable or wrong IP
```

### Scenario 2: "Network is slow"

```bash
# Capture all traffic
sudo tcpdump -i eth0 host storage.server -w slow.pcap

# Analyze later
tcpdump -r slow.pcap | grep -i "retrans\|duplicate"

# If many retransmissions: Packet loss on network
# If normal but throughput low: Application or server issue
```

### Scenario 3: "Connection refused"

```bash
# Capture attempt
sudo tcpdump -i eth0 host problem.server port 22

# Try SSH
ssh problem.server

# Packets should show:
# SYN from client to server
# RST (reset) from server back

# RST = "I'm listening but refusing"
# (vs no response = "I'm not listening")
```

---

## Key Takeaways

1. **tcpdump shows packets** — what crosses the network
2. **Flags tell the story** — SYN/ACK/FIN/RST each mean something
3. **Filters narrow down** — focus on relevant traffic
4. **No packets = network issue** — packets in, none out = firewall
5. **RST = connection refused** — server rejected it
6. **Duplicate ACKs = packet loss** — network issue
7. **Save to file** — analyze later with tcpdump or wireshark

---

## How Ready Are You?

Can you explain these?
- [ ] What packet flags (SYN, ACK, FIN, RST) mean
- [ ] How to filter tcpdump for specific traffic
- [ ] What it means when you see SYN but no SYN,ACK
- [ ] How to identify packet loss in tcpdump
- [ ] When to use tcpdump vs other network tools

If you checked all boxes, you're ready for Lesson 6.3.

---

Powered by UQS v1.8.5-C
