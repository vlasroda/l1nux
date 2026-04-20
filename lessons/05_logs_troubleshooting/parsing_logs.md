# Lesson 5.2: Parsing Logs with grep, awk, sed (Refresher)

**Time:** 25-30 minutes
**Why it matters:** Logs are useless if you can't extract the signal from the noise. Master these tools and troubleshooting becomes 10x faster
**Skills:** Use grep, awk, sed to extract, filter, and transform log data

---

## Quick Review

### The Tool Stack

```
grep  - Filter lines matching pattern
awk   - Extract fields, transform data
sed   - Edit streams (find/replace, delete)
sort  - Order lines
uniq  - Remove duplicates
cut   - Extract columns
```

**Working together:**
```bash
# Example: Find top 10 IPs trying to SSH
grep "sshd" /var/log/auth.log | \
awk '{print $NF}' | \
sort | uniq -c | sort -rn | head -10
```

---

## Key Concepts

### 1. GREP (Filter and Search)

**Basic usage:**
```bash
grep "pattern" file              # Find lines matching pattern
grep -v "pattern" file          # Find lines NOT matching
grep -i "pattern" file          # Case-insensitive
grep -n "pattern" file          # Show line numbers
grep -c "pattern" file          # Count matching lines
grep -B 5 -A 5 "pattern" file   # Context (5 lines before/after)
```

**Patterns (regex):**
```bash
grep "error" file               # Literal word
grep "^error" file              # Start of line
grep "error$" file              # End of line
grep "err.*or" file             # Wildcard (any chars between)
grep -E "error|warn" file       # OR (needs -E for extended)
grep "\[ERROR\]" file           # Literal brackets (need \ to escape)
```

**Examples:**
```bash
# All nginx errors
grep "error" /var/log/nginx/error.log

# Errors in last hour
journalctl -p err --since "1 hour ago"
grep "$(date +%Y-%m-%d\ 10:)" /var/log/syslog  # Hour 10:xx

# Failed SSH attempts
grep "Failed password" /var/log/auth.log

# Out of memory killer
grep "Out of memory" /var/log/syslog

# Connection refused
grep "Connection refused" /var/log/syslog
```

### 2. AWK (Extract Fields)

**The power tool for log parsing:**

```bash
awk '{print $1, $3}'            # Print field 1 and 3
awk -F: '{print $1}'            # Use colon as field separator
awk '{print NF, $NF}'           # Print number of fields and last field
awk '{print $(NF-1)}'           # Print second-to-last field
awk 'NR>10 {print}'             # Print lines after line 10
```

**Operators:**
```bash
awk '$5 > 100 {print}'          # Print if field 5 > 100
awk '$1 == "error" {print}'     # Print if field 1 equals "error"
awk '$2 ~ /timeout/ {print}'    # Print if field 2 matches regex
awk '$3 !~ /success/ {print}'   # Print if field 3 doesn't match
```

**Functions:**
```bash
awk '{print tolower($0)}'       # Convert to lowercase
awk '{print toupper($0)}'       # Convert to uppercase
awk '{print length($0)}'        # Length of line
awk '{sum += $5} END {print sum}'  # Sum a field
awk '{count++} END {print count}'  # Count lines
```

**Examples:**
```bash
# Extract source IPs from log (field 5)
grep "sshd" /var/log/auth.log | awk '{print $5}'

# Count errors by type
grep "error" /var/log/syslog | awk -F: '{print $2}' | sort | uniq -c

# Show processes using >100MB
ps aux | awk '$6 > 100000 {print $0}'

# Extract timestamp and message
grep "nginx" /var/log/syslog | awk '{print $1, $2, $3, $NF}'

# Sum disk usage
du /home/* | awk '{sum += $1} END {print sum / 1024 / 1024 "GB"}'
```

### 3. SED (Stream Editor)

**Find and replace, delete lines:**

```bash
sed 's/old/new/' file           # Replace first "old" with "new" per line
sed 's/old/new/g' file          # Replace all occurrences (g = global)
sed 's/old/new/2' file          # Replace only 2nd occurrence per line
sed -i 's/old/new/g' file       # Edit in-place (modifies file!)

sed '10d' file                  # Delete line 10
sed '1,10d' file                # Delete lines 1-10
sed '/pattern/d' file           # Delete lines matching pattern

sed 's/.*/PREFIX: &/' file      # Prepend "PREFIX: " to each line
sed 's/^/  /' file              # Indent each line
```

**Examples:**
```bash
# Remove timestamps for comparison
sed 's/Jan [0-9]* [0-9:]*//g' /var/log/syslog

# Extract just error message
sed 's/.*: //' /var/log/syslog | grep error

# Hide sensitive data
sed 's/password=[^ ]*/password=***/' config.log

# Count non-comment lines
sed '/^#/d; /^$/d' config.txt | wc -l
```

### 4. Combining Tools (Pipelines)

**The real power comes from chaining:**

```bash
# Find error messages and show unique ones
grep "error" /var/log/syslog | awk -F: '{print $NF}' | sort | uniq -c | sort -rn

# Top 10 processes by memory
ps aux | sort -k 6 -rn | head -10

# Count errors by service
grep -oh "^[^:]*:" /var/log/syslog | sort | uniq -c | sort -rn

# Find and report on failed SSH attempts
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

### 5. Common Log Parsing Patterns

**Extract IPs from access logs:**
```bash
grep "192.168" /var/log/nginx/access.log | awk '{print $1}' | sort -u
```

**Count errors by hour:**
```bash
grep "error" /var/log/syslog | awk '{print $1, $2, $3}' | sort | uniq -c
```

**Find slow queries in logs:**
```bash
grep "query" /var/log/mysql.log | awk '$NF > 1000 {print}'
# Shows queries taking > 1000ms
```

**Extract service restarts:**
```bash
grep "Started\|Stopped\|Restarted" /var/log/syslog | awk '{print $1, $2, $3, $(NF-2), $(NF-1), $NF}'
```

**Show top error messages:**
```bash
journalctl -p err | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head -20
```

---

## Essential Commands

### Quick Filtering

```bash
# Show last N errors
journalctl -p err -n 20

# Show errors since time
journalctl -p err --since "10:00:00" --until "10:30:00"

# Count occurrences
grep -c "pattern" file
journalctl | grep -c "error"

# Show unique values
grep "error" file | awk '{print $5}' | sort -u
```

### Log Analysis

```bash
# Timeline of events
grep "service" /var/log/syslog | awk '{print $1, $2, $3, $(NF-1), $NF}'

# Error summary
grep -i "error\|warn\|fail" /var/log/syslog | awk -F: '{print $2}' | sort | uniq -c | sort -rn

# Find bottleneck (what fails most)
journalctl -p err -o short | awk '{print $NF}' | sort | uniq -c | sort -rn | head -1
```

### Data Extraction

```bash
# Get field
awk '{print $5}' file

# Get fields between delimiters
awk -F: '{print $1, $3}' /etc/passwd

# Get ranges
sed -n '100,200p' file

# Remove duplicates
sort file | uniq
```

---

## Interactive Exercise: Parse a Log File

**Task 1: Find patterns**
```bash
# Count errors
grep -c "error" /var/log/syslog

# Count by type
grep "error" /var/log/syslog | head -20

# See unique errors
grep "error" /var/log/syslog | awk -F: '{print $NF}' | sort -u
```

**Task 2: Extract specific data**
```bash
# Get timestamps of errors
grep "error" /var/log/syslog | awk '{print $1, $2, $3}' | head -10

# Get service names
grep "error" /var/log/syslog | awk -F[ '{print $1}' | sort -u
```

**Task 3: Analyze patterns**
```bash
# Top 5 error messages
grep "error" /var/log/syslog | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head -5

# Timeline
grep "error" /var/log/syslog | awk '{print $1 " " $2 " " $3}' | sort | uniq -c
```

**Task 4: Combine filters**
```bash
# Errors from specific service (e.g., nginx)
grep "nginx" /var/log/syslog | grep -i "error\|warn"

# Complex: Find slow operations
journalctl -u myservice -o json | jq '.duration' | awk '$1 > 5000 {count++} END {print count " slow operations"}'
```

---

## Challenge Questions

**Answer without looking them up:**

1. **grep Command:** What does `grep -v "error"` do?
   ```
   A) Find lines with errors
   B) Find lines WITHOUT errors
   C) Show error count
   D) Delete error lines
   ```

2. **awk Field:** In `awk '{print $NF}'`, what is $NF?
   ```
   A) First field
   B) Last field
   C) Nth field
   D) Non-field
   ```

3. **Pipeline:** What does this do?
   ```bash
   grep "error" /var/log/syslog | awk '{print $5}' | sort | uniq -c | sort -rn
   ```
   (Explain what each step does)

4. **sed Replace:** You want to replace all instances of "old" with "new" in a file. What's wrong?
   ```bash
   sed 's/old/new/' /var/log/syslog
   A) It only replaces the first per line
   B) It will modify the file
   C) It won't work on compressed files
   D) Nothing, it's correct
   ```

5. **Real Scenario:** Logs show "Connection timeout" many times. How would you find which IPs are timing out?
   ```
   (Write a command or pipeline)
   ```

---

## Common L1 TSE Scenarios

### Scenario 1: "Disk full - find what's filling it"

```bash
# Find largest files
find /var/log -type f -exec ls -lh {} \; | sort -k5 -rh | head -10

# Or simpler
du -sh /var/log/* | sort -rh

# Check if logs are rotating
ls -la /var/log/syslog*

# Count log entries
wc -l /var/log/syslog
```

### Scenario 2: "Too many failed logins - find the source"

```bash
# Find IPs attempting login
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn

# Top 5 offenders
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -5

# Block them with firewall
# Or use fail2ban
```

### Scenario 3: "Service keeps crashing - find why"

```bash
# Find crash messages
grep -i "crash\|segfault\|killed" /var/log/syslog | tail -20

# Get timestamps
journalctl -u myservice | grep -i "crash" | awk '{print $1, $2, $3}'

# Check core dumps
ls -la /var/lib/systemd/coredump/

# Check permissions (common cause)
journalctl -u myservice -n 50 | grep -i "permission\|denied"
```

### Scenario 4: "Performance degradation - when did it start?"

```bash
# Find where performance changed
grep -i "slow\|timeout\|error" /var/log/syslog | awk '{print $1, $2}' | uniq -c | sort

# Or look for service restarts
grep -E "Started|Stopped|Restarted" /var/log/syslog | tail -20

# Compare time-based error rates
journalctl --since "2 hours ago" -p err | awk '{print $1 " " $2 " " $3}' | sort | uniq -c
```

---

## Key Takeaways

1. **grep finds lines** — use it first to narrow down
2. **awk extracts fields** — the workhorse for parsing
3. **sed transforms data** — powerful but use carefully
4. **Pipelines combine** — real power comes from chaining
5. **Context matters** — `grep -B 5 -A 5` shows what happened around error
6. **Sort and uniq** — find patterns (top offenders, unique errors)

---

## How Ready Are You?

Can you explain these?
- [ ] How to find all error lines in a log
- [ ] How to extract a specific field from a log
- [ ] The difference between grep and grep -v
- [ ] What `awk '{print $NF}'` means
- [ ] How to find the most common error message

If you checked all boxes, you're ready for Lesson 5.3.

---

Powered by UQS v1.8.5-C
