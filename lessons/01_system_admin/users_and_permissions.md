# Lesson 1.1: Users and Permissions (Refresher)

**Time:** 20-30 minutes
**Why it matters:** File permissions are the first thing to check in 80% of access issues
**Skills:** Check permissions, understand numeric notation, fix common permission problems

---

## Quick Review

### The Three Permission Classes
```
-rw-r--r--  1 alice staff  4096 Jan 19 10:30 file.txt
```

Breaking this down:
```
- rw- r-- r--
| |  |   |
type u  g  o   (owner, group, others)
     |  |  |
    read, write, execute
```

**Each class gets 3 bits: read (r), write (w), execute (x)**
- **Owner (u):** User who owns the file
- **Group (g):** Users in the file's group
- **Others (o):** Everyone else

**Numeric notation (rwx = 4+2+1 = 7):**
- 7 = rwx (4+2+1)
- 6 = rw- (4+2)
- 5 = r-x (4+1)
- 4 = r-- (4)
- 0 = --- (0)

So `644` = `rw- r-- r--` (owner reads/writes, others read)

---

## Key Concepts

### 1. File Ownership
Every file has an owner (user) and a group

```bash
ls -l /var/log/syslog
-rw-r-----  1 root  adm  123456 Jan 19 10:30 /var/log/syslog
            |  |    |
            |  |    +-- Group is 'adm'
            |  +------- Owner is 'root'
            +---------- Ownership count
```

### 2. Default Umask
When a file is created, permissions are set by `umask`

```bash
umask        # Show current umask (typically 0022)

# 0022 means: remove these bits from new files
# So new files get 666 - 022 = 644 (rw-r--r--)
# And new dirs get 777 - 022 = 755 (rwxr-xr-x)
```

### 3. Special Permissions
Three special permission bits on top of rwx:

- **SUID (4):** `chmod 4755` → file runs as owner, not user
  ```
  -rwsr-xr-x  (notice 's' instead of 'x' in owner position)
  ```
  Example: `/usr/bin/sudo` runs as root even when you call it

- **SGID (2):** `chmod 2755` → new files inherit group from directory
  ```
  -rwxr-sr-x  (notice 's' instead of 'x' in group position)
  ```
  Useful for shared directories

- **Sticky Bit (1):** `chmod 1777` → only owner can delete their own files
  ```
  drwxrwxrwt  (notice 't' instead of 'x' in others position)
  ```
  Example: `/tmp` — you can't delete someone else's files

### 4. Directory Permissions
For directories, the permissions mean:
- **read (r):** Can you list contents (`ls`)?
- **write (w):** Can you create/delete files in it?
- **execute (x):** Can you enter the directory (`cd`)?

So `755` on a directory means:
- Owner: read, write, execute (full access)
- Group: read, execute (can see and enter, not create)
- Others: read, execute (can see and enter, not create)

---

## Essential Commands

### Check Permissions

```bash
# Simple listing
ls -l /path/to/file

# Detailed info
stat /path/to/file

# Find files with specific permissions
find / -perm 644        # Files with exactly 644
find / -perm /u+s       # Files with SUID bit set
find / -perm -g+w       # Files writable by group
```

### Change Permissions

```bash
# Numeric method
chmod 755 myfile        # Owner rwx, group rx, others rx

# Symbolic method
chmod u+x myfile        # Add execute for owner
chmod g-w myfile        # Remove write from group
chmod o=r myfile        # Set others to read-only

# Recursive
chmod -R 755 /dir       # Change /dir and all contents

# Common scenarios
chmod 644 *.txt         # Make files readable but not executable
chmod 755 *.sh          # Make scripts executable
```

### Change Ownership

```bash
# Change owner
chown alice myfile      # alice becomes owner

# Change owner and group
chown alice:staff myfile  # alice becomes owner, staff is group

# Recursive
chown -R alice:staff /home/alice/project

# Be careful!
sudo chown -R root /etc  # Don't accidentally do this
```

### Check Who Can Do What

```bash
# What groups am I in?
groups                  # Current user's groups
groups username         # Specific user's groups

# What can I sudo?
sudo -l                 # What commands can I run with sudo?

# Who owns a file?
ls -l /path
stat /path
```

---

## Interactive Exercise: Permission Troubleshooting

**Scenario:** You're supporting a system where:

1. User `alice` can't read `/var/log/app.log`
2. User `bob` needs to write to `/srv/data/upload`
3. File `/home/shared/secret.txt` should only be readable by `alice`

**Your tasks:**

```bash
# Task 1: Diagnose why alice can't read /var/log/app.log
ls -l /var/log/app.log          # What are the permissions?
id alice                        # What groups does alice have?
# Why can't alice read it? (Answer below)

# Task 2: Set up /srv/data/upload so bob can write
mkdir -p /srv/data/upload       # Create directory
ls -l /srv/data/                # Check current permissions
# Fix it (Answer below)

# Task 3: Restrict secret.txt to only alice
touch /home/shared/secret.txt   # Create file
# Set permissions (Answer below)
```

**Expected output after fixes:**
```bash
# Task 1: Should be readable by alice
ls -l /var/log/app.log
-rw-r--r-- 1 root root 5000 Jan 19 /var/log/app.log
# alice can read because "others" has read permission

# Task 2: Should be writable by bob
ls -ld /srv/data/upload
drwxrwxr-x 2 root staff 4096 Jan 19 /srv/data/upload/
# bob is in staff group, group has write permission

# Task 3: Should only be readable by alice
ls -l /home/shared/secret.txt
-r-------- 1 alice alice 0 Jan 19 /home/shared/secret.txt
# Only alice (owner) can read
```

---

## Challenge Questions

**Answer these without looking them up:**

1. **Numeric Notation:** What does `chmod 700` do?
   ```
   A) Locks everyone out except owner
   B) Makes it readable by everyone
   C) Removes all permissions
   D) Sets owner to read-only
   ```

2. **SUID Bit:** Why does `/usr/bin/sudo` need the SUID bit?
   ```
   Explain in 1-2 sentences.
   ```

3. **Directory Permissions:** Why can't you `cd` into a directory with `744` permissions (rwxr--r--)?
   ```
   What permission bit is missing?
   ```

4. **Group Membership:** You create a file in a directory with SGID bit set. Which group owns the new file?
   ```
   A) Your primary group
   B) The directory's group
   C) root
   D) The group of the user who created it
   ```

5. **Real Scenario:** A user reports they can't delete their own file from `/tmp`. The file is:
   ```
   -rw-r--r-- 1 alice alice 1024 Jan 19 /tmp/myfile
   /tmp permissions are: drwxrwxrwt
   ```
   Why can't they delete it?
   ```
   A) They don't have write permission on the file
   B) They don't have write permission on the /tmp directory
   C) The sticky bit prevents it
   D) They're not in the right group
   ```

---

## Key Takeaways

1. **Three classes:** owner, group, others
2. **Three permissions:** read (4), write (2), execute (1)
3. **Always check:** `ls -l` to see current state
4. **Quick fix:** `chmod 644` for readable files, `chmod 755` for directories/scripts
5. **For directories:** execute permission means "can enter"
6. **Most common issue:** File not readable/writable because of wrong group or "others" bits

---

## Common TSE Scenarios

### Scenario 1: "I can't write to /var/log"
```bash
# Check current permissions
ls -ld /var/log
# Usually: drwxr-xr-x (only root can write)

# Solution: Add yourself to syslog group
sudo usermod -a -G syslog $USER

# Then logout/login for group membership to take effect
```

### Scenario 2: "Cron job can't read my file"
```bash
# Cron runs as the user's UID, needs to read the file
ls -l myfile
# If it's: -rw------- (600), cron can read it

# But if file is in a directory with no execute permission
ls -ld /home/user/scripts
# If it's: drwxr----- and you're not owner, can't access

# Solution: chmod 755 /home/user/scripts
```

### Scenario 3: "Shared directory not working"
```bash
# Goal: alice and bob should both write to /shared
mkdir /shared
sudo chown alice:dev /shared    # Group is 'dev'
sudo usermod -a -G dev bob      # Add bob to dev group
sudo chmod 2770 /shared         # SGID + rwx for owner/group

# Now files created by alice or bob are owned by 'dev' group
```

---

## Next Step

When you're ready, move to **Lesson 1.2: Process Management** or do the exercise and come back.

If you want to test your knowledge first, answer the **Challenge Questions** above.

---

Powered by UQS v1.8.5-C
