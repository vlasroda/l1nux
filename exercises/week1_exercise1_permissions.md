# Week 1 Exercise 1: Permission Troubleshooting

**Difficulty:** Beginner
**Time:** 20 minutes
**Prerequisites:** Lesson 1.1 (Users & Permissions)

---

## Setup

Run these commands to create the test environment:

```bash
# Create test users and groups
sudo useradd -m -s /bin/bash alice 2>/dev/null || true
sudo useradd -m -s /bin/bash bob 2>/dev/null || true
sudo groupadd -f webdev
sudo usermod -a -G webdev alice
sudo usermod -a -G webdev bob

# Create test directories and files
sudo mkdir -p /srv/web /srv/backup /var/log/apps
sudo touch /srv/web/config.txt /var/log/apps/app.log /etc/secret.conf
```

---

## Problem 1: Web Application Can't Write Logs

**Scenario:** A web app runs as user `www-data` and needs to write to `/srv/web/logs/`. Currently it fails with "Permission denied".

**Current state:**
```bash
ls -ld /srv/web
# drwxr-xr-x 2 root root 4096 Jan 19 /srv/web
```

**Task:** Fix permissions so `www-data` can write to `/srv/web/logs/` directory

**What to do:**
1. Create the logs directory
2. Check current permissions
3. Change ownership or permissions so www-data can write
4. Test by creating a file as www-data

**Expected result:**
```bash
# After your fix:
ls -ld /srv/web
# Should allow www-data to write

# Test
sudo -u www-data touch /srv/web/logs/test.log
# Should succeed without error
```

**Challenge:** Do this without giving `www-data` ownership of the directory (i.e., keep root as owner)

---

## Problem 2: Shared Team Directory

**Scenario:** Team members `alice` and `bob` need to collaborate in `/srv/team`. Files created by one should be readable/writable by the other. Files created by others should not be editable.

**Current state:**
```bash
ls -ld /srv/team
# drwxr-xr-x 2 root root 4096 Jan 19 /srv/team
```

**Task:** Set up `/srv/team` as a proper shared directory

**What to do:**
1. Change group to `webdev` (which contains both alice and bob)
2. Set SGID bit so new files inherit the group
3. Set permissions so alice and bob can read/write, others can't
4. Test by creating files as both users

**Expected result:**
```bash
ls -ld /srv/team
# drwxrws--- 2 root webdev 4096 Jan 19 /srv/team
# (notice 's' for SGID)

# alice creates a file
sudo -u alice touch /srv/team/alice_file.txt
ls -l /srv/team/alice_file.txt
# -rw-r----- 1 alice webdev

# bob can write to it
sudo -u bob bash -c 'echo "hello" >> /srv/team/alice_file.txt'
# Should succeed
```

---

## Problem 3: Script Execution

**Scenario:** You have a backup script at `/opt/backup.sh` that needs to:
- Be executable by the `backup` user
- Run by a cron job as root
- Not be modifiable by regular users

**Task:** Set correct permissions on the script

**What to do:**
1. Create the script file (copy from `/bin/ls` or make a dummy script)
2. Set permissions so:
   - Owner (root) can read/write/execute
   - Group can read/execute but not write
   - Others cannot access it

**Commands:**
```bash
# Create script
sudo tee /opt/backup.sh > /dev/null << 'EOF'
#!/bin/bash
echo "Backup running at $(date)"
EOF

# Your task: set permissions
chmod ??? /opt/backup.sh

# Test
ls -l /opt/backup.sh
```

**Expected result:**
```bash
ls -l /opt/backup.sh
# -rwxr-x--- 1 root root (or similar)

# Verify
ls -l /opt/backup.sh | grep '^-rwxr.----'
```

---

## Problem 4: Log File Security

**Scenario:** A system log file `/var/log/secure_app.log` contains sensitive data. Only `root` and the `admin` group should read it.

**Current state:**
```bash
-rw-r--r-- 1 root root /var/log/secure_app.log
```

**Task:** Restrict access

**What to do:**
1. Change group to `admin`
2. Remove all access from "others"
3. Allow group to read

**Commands:**
```bash
# Create test file
sudo touch /var/log/secure_app.log
# Your commands here

# Verify
ls -l /var/log/secure_app.log
```

**Expected result:**
```bash
ls -l /var/log/secure_app.log
# -rw-r----- 1 root admin
```

---

## Challenge: Complex Scenario

**Scenario:** You need to set up `/srv/projects`:
- Owner: `alice` (can do anything)
- Group: `webdev` (can read and execute)
- New files/dirs created in it should be owned by `webdev` group
- Others cannot access it

**Commands:**
```bash
sudo mkdir -p /srv/projects
sudo chown alice:webdev /srv/projects

# Your commands here to:
# 1. Set SGID bit
# 2. Set proper permissions
# 3. Test with alice creating a file

# Test
sudo -u alice bash -c 'echo "project" > /srv/projects/test.txt'
ls -l /srv/projects/test.txt
# Should show: -rw-r----- 1 alice webdev (or similar)
```

**What permissions should it be?**
```
Owner:  rwx (read, write, execute/enter)
Group:  r-x (read, execute/enter, no write)
Others: --- (no access)
SGID:   yes (files inherit group)
```

---

## Self-Check

After completing all problems:

- [ ] Problem 1: www-data can write to /srv/web/logs
- [ ] Problem 2: alice and bob can share files in /srv/team
- [ ] Problem 3: backup script is executable and restricted
- [ ] Problem 4: secure log file readable only by root and admin group
- [ ] Challenge: /srv/projects works as a shared space

---

## Next Steps

Compare your answers with the solutions in `week1_solutions.md`.

Focus on understanding WHY each permission works, not just the commands.

---

Powered by UQS v1.8.5-C
