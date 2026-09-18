Before deep enumeration, try the cheapest wins: default, common, and reused passwords. Many boxes hand you root from a guessed or already-found credential — no exploit needed.
## Goal
Get root (or a better user) by guessing common passwords or reusing creds you already found.
## Tips
ALWAYS try every credential you found (web, db, config, history) against EVERY user including root, before reaching for exploits. It's the cheapest path on the box.

Checklist:
- [ ] tried root with: root / toor / password / boxname / appname ?
- [ ] tried each found credential via `su user`?
- [ ] after switching user, re-ran `sudo -l`?
- [ ] pre-seeded users present (vagrant, ubuntu)?
# Commands

### Step 1: list login users
```bash
cat /etc/passwd | grep -vE 'nologin|false'   # users with a real shell
ls -la /home                                  # home dirs = real users
```

### Step 2: try passwords
```bash
su root            # try: root, toor, password, <boxname>, admin
su <user>          # try found creds / username-as-password
```

## Exploit
### Case 1: Common root password
**Enumerate:**
```bash
cat /etc/passwd | grep -v nologin
# if root has /bin/bash and box looks default/dev -> guessable
```
**Exploit:**
```bash
su -               # try: root, toor, password, password123, <boxname>, <appname>
id                 # uid=0 -> root
```

### Case 2: Password reuse
**Enumerate:**
```bash
# you found a password in web config / db / .env / banner
```
**Exploit:**
```bash
su <user>          # reuse the found password on each user
sudo -l            # re-check sudo as the new user
# reuse is the #1 pivot on real boxes
```

### Case 3: Pre-seeded / default accounts
**Enumerate:**
```bash
grep -E 'vagrant|ubuntu|admin|test|guest' /etc/passwd
```
**Exploit:**
```bash
su vagrant         # classic: vagrant:vagrant
```

## Helpline:
1. Common passwords to try
```
root toor password password123 admin administrator
<boxname> <appname> 123456 letmein changeme
+ username as its own password (bob:bob)
```

2. `su -` vs `su`
```
su -   -> full login shell, root's env + PATH (use this)
su     -> keeps your env (can break paths)
```

3. Reuse principle
```
one password = try it EVERYWHERE: every /etc/passwd user, sudo, ssh, db, web.
most user->root and box->box jumps are reuse, not exploits.
```

4. Grab creds first
```
web configs (wp-config.php, .env), db dumps, .bash_history, .my.cnf,
service banners, linpeas "passwords". -> see [[08_interesting_files]]
```
