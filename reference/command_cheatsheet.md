# Linux L1 TSE Command Cheatsheet

Quick reference for the most useful commands. Keep this open while practicing.

---

## System Information

```bash
# General status
uptime                          # Load average and uptime
hostnamectl                     # System hostname and OS
uname -a                        # Kernel and system info
lsb_release -a                  # Distribution info

# Current user/groups
whoami                          # Who am I?
id                              # My UID, GID, groups
groups                          # My groups
id alice                        # Someone else's info
```

---

## Users & Groups

```bash
# User info
getent passwd alice             # Full user entry
getent group dev                # Full group entry

# Permissions
ls -l file                      # Show permissions
stat file                       # Detailed file info
chmod 755 file                  # Change permissions (numeric)
chmod u+x file                  # Change permissions (symbolic)
chown alice:staff file          # Change owner:group
chown -R alice:staff /dir       # Recursive

# Groups
sudo usermod -a -G groupname user    # Add user to group
sudo usermod -G groupname user       # Replace groups
sudo groupadd newgroup               # Create group
```

---

## Files & Directories

```bash
# Navigation & listing
pwd                             # Current directory
ls -la                          # List all, detailed
ls -lh                          # Human-readable sizes
cd /path                        # Change directory
cd ~                            # Home directory
cd -                            # Previous directory

# File operations
cat file                        # Show contents
less file                       # Paginated view
head -20 file                   # First 20 lines
tail -20 file                   # Last 20 lines
tail -f file                    # Follow new lines
wc -l file                      # Count lines
grep pattern file               # Search for pattern

# Searching
find /path -name "*.txt"        # Find by name
find /path -type f -size +10M   # Large files
locate filename                 # Fast search (uses index)
which command                   # Find command in PATH
```

---

## Processes

```bash
# View
ps aux                          # All processes
ps aux | grep app               # Find process by name
pgrep app                       # Just PIDs
pstree                          # Process tree
pstree -p                       # With PIDs
top                             # Interactive, real-time
htop                            # Better version of top
watch 'ps aux | head -10'       # Refresh view

# Signal/control
kill -15 <pid>                  # SIGTERM (graceful stop)
kill -9 <pid>                   # SIGKILL (force kill)
kill -1 <pid>                   # SIGHUP (reload config)
killall httpd                   # Kill by name
pkill -f "java.*app"            # Kill by pattern
kill -STOP <pid>                # Pause process
kill -CONT <pid>                # Resume process

# Details
lsof -p <pid>                   # Open files for process
strace -p <pid>                 # System calls
ps -p <pid> -o cmd=             # Process command line
cat /proc/<pid>/environ         # Environment variables
```

---

## Storage & Filesystems

```bash
# Usage
df -h                           # Disk usage by mount
du -sh /path                    # Directory size
du -sh /path/*                  # Size of each item
lsblk                           # Block devices tree
lsblk -a                        # All devices

# Filesystem info
mount                           # Currently mounted
mount | grep /dev               # By device
mount /dev/sdb1 /mnt            # Mount a device
umount /mnt                     # Unmount
fsck /dev/sdb1                  # Check filesystem

# RAID/LVM
cat /proc/mdstat                # RAID status
mdadm --detail /dev/md0         # RAID details
lvs                             # Logical volumes
vgs                             # Volume groups
```

---

## Network

```bash
# Interfaces
ip link                         # Interface status
ip addr                         # IP addresses
ip route                        # Routing table
ifconfig                        # Older tool (deprecated)

# Connectivity
ping -c 3 8.8.8.8              # Test reachability
traceroute 8.8.8.8             # Route to destination
host 8.8.8.8                   # DNS reverse lookup
nslookup google.com            # DNS forward lookup
mtr google.com                 # Traceroute + ping

# Active connections
ss -tlnp                        # TCP listeners
ss -an                          # All connections
netstat -an                     # Network stats
netstat -tulnp                  # Services listening

# Performance
iperf3 -s                       # Start server (in one terminal)
iperf3 -c <server>              # Test from client
iftop                           # Bandwidth by connection
nethogs                         # Bandwidth by process
```

---

## System Logs

```bash
# Systemd journal
journalctl                      # All logs
journalctl -f                   # Follow logs (real-time)
journalctl -n 50                # Last 50 lines
journalctl -u service           # Specific service
journalctl --since "1 hour ago" # Time range
journalctl -p err               # Only errors

# Traditional logs
tail -f /var/log/syslog         # System log (real-time)
tail -f /var/log/auth.log       # Authentication
dmesg                           # Kernel messages
dmesg | tail -20                # Recent kernel messages

# Filtering
grep ERROR /var/log/app.log     # Search logs
grep -c pattern file            # Count matches
tail -f file | grep ERROR       # Real-time filter
```

---

## Performance Analysis

```bash
# CPU
top -b -n 1                     # One snapshot of top
ps aux --sort=-%cpu             # Top CPU users
perf top                        # Detailed CPU profiling
mpstat -P ALL 1 3              # CPU stats per core

# Memory
free -h                         # Total/used/free
top -b -n 1 | head -20          # Memory-using processes
ps aux --sort=-%mem             # Top memory users
vmstat 1 3                       # Memory stats over time

# I/O
iostat -x 1 3                   # Disk I/O stats
iotop -b -n 1                   # Top I/O processes
blktrace /dev/sda               # Low-level disk tracing
dstat -d                        # Disk activity

# Load
uptime                          # Load average
w                               # Load and user activity
```

---

## Package Management

```bash
# Debian/Ubuntu
apt update                      # Update package list
apt install package             # Install
apt remove package              # Uninstall
apt search term                 # Search
dpkg -l | grep term             # Installed packages

# RedHat/CentOS
yum install package             # Install
yum remove package              # Uninstall
yum search term                 # Search
rpm -qa | grep term             # Installed packages

# Universal
systemctl list-unit-files       # Services installed
systemctl enable service        # Auto-start on boot
systemctl start service         # Start now
systemctl stop service          # Stop
systemctl status service        # Check status
```

---

## SSH & Remote Access

```bash
# SSH
ssh user@host                   # Connect
ssh -i keyfile user@host        # With key file
ssh -p 2222 user@host           # Custom port
scp file user@host:/path        # Copy file to remote
scp user@host:/path file        # Copy from remote

# SSH keys
ssh-keygen -t rsa -b 4096       # Generate key
ssh-copy-id user@host           # Copy public key to remote
chmod 700 ~/.ssh                # SSH dir permissions
chmod 600 ~/.ssh/id_rsa         # Private key permissions
```

---

## Text Processing

```bash
# Search/Filter
grep pattern file               # Find lines
grep -v pattern file            # Exclude lines
grep -i pattern file            # Case insensitive
grep -c pattern file            # Count matches

# Text transformation
sed 's/old/new/g' file         # Replace all
awk '{print $1}' file          # Extract column
awk -F: '{print $1}' file      # With delimiter
cut -d: -f1 file               # Get field

# Sorting/Counting
sort file                       # Sort lines
uniq file                       # Remove duplicates
uniq -c file                    # Count duplicates
wc -l file                      # Line count

# Combine operations
cat file | grep pattern | awk '{print $1}'  # Pipe commands
```

---

## System Administration

```bash
# Users
sudo useradd username           # Create user
sudo usermod -s /bin/bash user  # Change shell
sudo passwd username            # Set password
sudo userdel username           # Delete user

# System control
sudo systemctl reboot           # Reboot
sudo systemctl shutdown         # Shutdown
sudo systemctl rescue           # Rescue mode
sudo systemctl emergency        # Emergency mode

# Cron jobs
crontab -l                      # List my cron jobs
crontab -e                      # Edit my cron jobs
sudo crontab -l -u user         # User's cron jobs

# Sudo access
sudo -l                         # What can I sudo?
sudo visudo                     # Edit sudoers (safely)
```

---

## Useful Combinations

```bash
# Find largest files
du -ah /path | sort -rh | head -20

# Show top 10 memory-using processes
ps aux --sort=-%mem | head -11

# Real-time network stats
watch -n1 'netstat -an | grep ESTABLISHED | wc -l'

# Find and remove old files
find /path -type f -mtime +30 -delete

# Monitor specific service
watch -n1 'systemctl status service'

# Log analysis - show errors last hour
journalctl --since "1 hour ago" -p err

# Check who's using most CPU right now
top -b -n 1 -o %CPU | head -12

# Disk space by directory (depth 1)
du -sh /* | sort -rh
```

---

## Pro Tips

**Redirections:**
```bash
command > file          # Redirect output to file (overwrites)
command >> file         # Append to file
command 2> errors       # Redirect errors
command 2>&1            # Errors to same place as output
command < file          # Use file as input
```

**Pipes:**
```bash
cat file | grep pattern | wc -l  # Count matching lines
ps aux | grep app | grep -v grep  # Better than ps aux | grep
```

**Background/Foreground:**
```bash
command &               # Start in background
Ctrl+Z, then bg         # Move current to background
fg                      # Bring to foreground
jobs                    # List jobs
```

**Command history:**
```bash
history                 # Show history
!n                      # Run command #n
!!                      # Run last command
!string                 # Run last command starting with string
Ctrl+R                  # Search history
```

---

## Emergency Commands

When things are on fire:

```bash
# System is out of disk space
du -ah /* | sort -rh | head  # Find culprit
find /var/log -type f -delete # Clear old logs

# Process is stuck
ps aux | grep process          # Find it
kill -9 <pid>                  # Force kill

# Can't log in
sudo systemctl rescue          # Rescue mode

# Network down
ip link                        # Check interfaces
ip addr add 192.168.1.1/24 dev eth0  # Manual IP

# Filesystem corrupted
fsck /dev/sda1                 # (run from recovery mode)
```

---

Powered by UQS v1.8.5-C
