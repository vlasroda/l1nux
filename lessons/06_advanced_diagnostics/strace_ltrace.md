# Lesson 6.1: strace and ltrace (Refresher)

**Time:** 25-30 minutes
**Why it matters:** When everything else fails, trace the actual system calls. See exactly what a process is doing, why it's hanging, or why it's failing
**Skills:** Use strace to debug process behavior, ltrace for library calls, find blocking operations

---

## Quick Review

### The Call Stack

```
Your code
    ↓
Library calls (libc, libssl, etc.)
    ↓ (ltrace shows this)
System calls (kernel)
    ↓ (strace shows this)
Kernel
    ↓
Hardware
```

**strace:** Shows system calls and signals
**ltrace:** Shows library calls and signals

---

## Key Concepts

### 1. strace - System Call Tracing

**What it shows:**
```
- Every system call the process makes
- Parameters passed to each call
- Return value and errno (if error)
- Time spent in each call
- Signal handling

Example output:
open("/etc/passwd", O_RDONLY)           = 3
read(3, "root:x:0:0:root:/root:/bin/bash...", 4096) = 2841
close(3)                                 = 0

Shows:
- What file was opened
- File descriptor returned (3)
- What data was read
- Successfully closed
```

**Common system calls:**
```
Process:
  fork, execve, exit, wait, clone

Files:
  open, close, read, write, stat, chmod, chown

Directories:
  mkdir, rmdir, rename, opendir, readdir

Network:
  socket, bind, listen, accept, connect, send, recv

Memory:
  mmap, munmap, brk, sbrk

Signals:
  signal, sigaction, kill

Permissions:
  getuid, setuid, getgid, setgid
```

**Common use cases:**

```bash
# Find what file a process is looking for (and failing)
strace -e openat program

# Debug permission denied error
strace -e open,stat program 2>&1 | grep -E "open|stat|EACCES"

# Find where it's hanging
strace -p <pid>
# Will show the system call where it's blocked

# Performance profile
strace -c program
# Shows count and time for each syscall

# Follow child processes
strace -f program
# Traces all child processes too
```

### 2. ltrace - Library Call Tracing

**What it shows:**
```
- Calls to library functions
- Parameters and return values
- Time spent in each call

Example output:
strlen("hello")                          = 5
printf("Hello, %s!\n", "world")         = 14
malloc(1024)                            = 0x7ffff7dd5000
free(0x7ffff7dd5000)                    = <void>

Shows:
- Which library function called
- What parameters passed
- What was returned
- Timing
```

**Library calls vs system calls:**

```
Library call (ltrace shows):
strlen("hello")  → strlen in libc

System call (strace shows):
write(1, "hello", 5)  → write syscall to kernel

Most programs:
- Few direct syscalls
- Many library calls
- Libraries make the syscalls
```

**Use cases:**

```bash
# Debug memory issues
ltrace -e malloc,free,realloc program
# See memory allocation patterns

# Find which library function fails
ltrace program 2>&1 | grep -i "error\|fail"

# Debug segfaults
ltrace -e '*' program
# See all calls up to crash

# Performance of library calls
ltrace -c program
# Which library functions consume most time
```

### 3. Combining Traces

**When to use each:**

```
Question: Why does my program crash?
  → Use ltrace + strace (ltrace shows library, strace shows kernel)

Question: Why is it slow?
  → strace -c or ltrace -c (count time in calls)

Question: Where is it hanging?
  → strace -p <pid> (shows syscall where blocked)

Question: Why can't it find a file?
  → strace -e open,stat (shows file access attempts)

Question: Why does it crash on network call?
  → ltrace -e 'socket*' + strace -e network (see both levels)
```

### 4. Interpreting Common Errors

**ENOENT (No such file or directory)**
```
open("/etc/myconfig.conf", O_RDONLY) = -1 ENOENT (No such file or directory)

Means:
- File doesn't exist at that path
- Program is looking in wrong location
- Path needs to be created
```

**EACCES (Permission denied)**
```
open("/root/secret.txt", O_RDONLY) = -1 EACCES (Permission denied)

Means:
- File exists but process can't read it
- Check: ls -l /root/secret.txt
- Fix: chmod or chown to allow access
```

**EAGAIN (Resource temporarily unavailable)**
```
read(5, <buffer>, 1024) = -1 EAGAIN (Resource temporarily unavailable)

Means:
- Non-blocking I/O attempted but data not ready
- Normal for non-blocking sockets
- Should retry later
```

**EPIPE (Broken pipe)**
```
write(4, "data...", 100) = -1 EPIPE (Broken pipe)

Means:
- Writing to closed pipe/socket
- Other end disconnected
- Program should handle gracefully

Common in: Pipeline commands, network connections
```

### 5. Performance Analysis with strace

**Timing syscalls:**

```bash
strace -c program
# Output shows:
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 40.15    0.102345        2000        51           read
 35.20    0.089764        1234        73           write
 15.30    0.039012        1000        39     10    open
  5.20    0.013245         500        26           close
  4.15    0.010567         100       107           mmap
------ ----------- ----------- --------- --------- ----------------
100.00    0.255000                    296        10 total

Analysis:
- 40% of time in read (slow disk I/O)
- 35% of time in write
- 10 errors in open (permission issues?)
```

**Finding hot spots:**

```bash
strace -c -e trace=file program
# Only file-related syscalls

strace -c -e trace=network program
# Only network syscalls

strace -c -e trace=memory program
# Only memory syscalls
```

---

## Essential Commands

### Basic strace

```bash
# Trace a command
strace ls /tmp                      # Trace entire ls command

# Trace a running process
strace -p <pid>                     # Attach to running process

# Filter syscalls
strace -e open,read,write command   # Only these syscalls
strace -e trace=file command        # Only file-related

# See timing
strace -t command                   # Print timestamps
strace -T command                   # Print time spent in call
strace -tt command                  # Precise timestamps

# Performance profile
strace -c command                   # Count calls and time

# Output to file
strace -o output.txt command        # Save trace to file
strace -ff -o output.txt -p <pid>   # Per-thread output
```

### Basic ltrace

```bash
# Trace library calls
ltrace command                      # Trace library calls

# Filter by function
ltrace -e malloc command            # Only malloc
ltrace -e 'socket*' command         # Wildcard patterns

# Trace running process
ltrace -p <pid>                     # Attach to process

# Performance profile
ltrace -c command                   # Count and time

# Follow children
ltrace -f command                   # Trace child processes

# Combine with strace
strace -c command 2>&1 | head -20   # See syscalls
ltrace -c command 2>&1 | head -20   # See library calls
```

### Attach to Running Process

```bash
# Find the process
pgrep myprocess
# Example output: 1234

# Trace it
strace -p 1234
# Will show syscalls in real-time

# Trace with output to file
strace -p 1234 -o trace.txt

# Detach when done
# Ctrl+C to stop tracing (doesn't kill process)
```

---

## Interactive Exercise: Debug with strace

**Task 1: Trace a simple command**
```bash
# Trace ls and see file access
strace ls /tmp 2>&1 | head -30

# You should see:
# - open() calls for files
# - stat() for file info
# - read() for directory contents
# - write() to output
```

**Task 2: Find missing file**
```bash
# Try to run missing program
strace nonexistent 2>&1 | grep "open\|ENOENT"

# Should show:
# open("/usr/bin/nonexistent", O_RDONLY) = -1 ENOENT

# This tells you exactly what it was looking for
```

**Task 3: Profile a command**
```bash
# See where time is spent
strace -c sleep 1

# Should show mostly clock_nanosleep (the sleep syscall)
```

**Task 4: Debug a hung process**
```bash
# Start a background process that might hang
# sleep 3600 &

# In another terminal:
strace -p <pid>

# If it's truly hung, you'll see it stuck on a syscall
# like select(), poll(), read(), etc.

# Ctrl+C to stop tracing
```

---

## Challenge Questions

**Answer without looking them up:**

1. **Tool Choice:** You want to see why a program is slow. Should you use strace or ltrace?
   ```
   A) strace (system calls)
   B) ltrace (library calls)
   C) Both (get full picture)
   D) Depends on where time is spent
   ```

2. **Error Code:** strace shows `open(...) = -1 ENOENT`. What's the problem?
   ```
   A) File is write-protected
   B) File doesn't exist
   C) No permission to access
   D) File is locked
   ```

3. **Hanging Process:** strace -p <pid> shows stuck on `futex()`. What does this mean?
   ```
   A) Process is waiting for disk I/O
   B) Process is in a deadlock/waiting for lock
   C) Process crashed
   D) Process is in a network call
   ```

4. **Performance:** strace -c shows 80% of time in `read()`. What's the issue?
   ```
   A) Slow disk I/O
   B) Slow network
   C) CPU limited
   D) Memory leak
   ```

5. **Permission Debug:** Permission denied error. How would you trace it?
   ```
   (Explain the command and what to look for)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Program fails with cryptic error"

```bash
# Run with strace to see exact failure
strace -e trace=file,network program 2>&1 | tail -20

# Look for:
# - ENOENT (file not found)
# - EACCES (permission denied)
# - ECONNREFUSED (connection refused)
# - ETIMEDOUT (timeout)

# Example output might show:
open("/etc/myconfig.conf", O_RDONLY) = -1 ENOENT

# Now you know exactly what file is missing
```

### Scenario 2: "Process hangs when writing"

```bash
# Attach strace to running process
strace -p <pid> 2>&1 | tail -20

# Might show stuck on:
write(3, "data...", 1024)
# Stuck = other end of pipe closed
# Or: waiting for socket to be writable

# Solution: Check the other end of connection
netstat -an | grep the-connection
```

### Scenario 3: "Segmentation fault - where?"

```bash
# Run with both strace and ltrace for full picture
ltrace -e '*' program 2>&1 | tail -30

# Look for last syscall or library call before crash
# That's where the segfault occurs

# Example:
strcpy(buffer, data)              # Last library call
# Segfault after this = strcpy overflowed buffer

# Fix: Use safer function like strncpy
```

---

## Key Takeaways

1. **strace shows system calls** — what the kernel is doing
2. **ltrace shows library calls** — what libraries are doing
3. **Use both** — get the full picture
4. **ENOENT = file not found**, EACCES = permission denied
5. **Stuck on syscall = blocked waiting** for that operation
6. **strace -c profiles** — see where time is spent
7. **Attach with -p** — trace running process without restart

---

## How Ready Are You?

Can you explain these?
- [ ] The difference between strace and ltrace
- [ ] What ENOENT, EACCES mean
- [ ] How to find where a process is hanging
- [ ] How to profile with strace -c
- [ ] When to use each tool

If you checked all boxes, you're ready for Lesson 6.2.

---

Powered by UQS v1.8.5-C
