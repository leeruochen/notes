# Linux PrivEsc — Master Checklist

Run top to bottom: lowest-hanging fruit first, kernel exploits last. Drop linpeas + pspy first, then work the list manually while they run.

## Run first (auto-enum)

```bash
./linpeas.sh -a | tee lp.txt          # red/yellow = likely vector
./pspy64 -f -i 1000                   # hidden cron / root processes
id ; sudo -l ; uname -a               # 3 freebies before anything else
```
## Fallback linpeas alternative
```bash
./lse.sh -l 1      # linux smart enum, level 1 (interesting only); -l 2 = everything
./LinEnum.sh -t    # LinEnum, thorough, good if have creds 
```

## Orientation: processes, environment & local listeners (do this early, not as a last resort)
Cheap, high-signal, and often missed. Do it *before* kernel exploits.
```bash
ps auxww                                   # full args — creds leak in command lines (mysql -p<pass>)
cat /proc/<pid>/cmdline | tr '\0' ' '; echo   # a running process's exact args
cat /proc/<pid>/environ | tr '\0' '\n'     # its environment (DB_PASS, tokens) — only for PIDs you can read
env ; cat /etc/environment                  # your env + machine-wide env files
ss -lntup                                   # local/loopback listeners = internal services
ip -brief addr ; ip route                   # extra NIC / second subnet -> pivot
```
Then map each interesting listener/process: **listener → PID → executable → owner → config/version**.
- Creds in a service's args or env file → reuse everywhere → [[08_interesting_files]].
- Loopback-only service or a second subnet → [[internal_port_enum]] (a loopback listener is a *clue*, not an escalation).
- A **root-owned unfamiliar** binary/service → confirm version + prerequisites, then [[public_exploit_research]] — *not every root process is vulnerable.*

> [!warning] Permissions & noise
> `/proc/<pid>/environ` and `cmdline` are only readable for your own processes unless you're root; don't assume access. Reading another user's process env usually needs higher privilege — absence of output ≠ absence of secrets.
## Checklist (1 = lowest-hanging fruit)

- [ ] 1. su common / reused passwords (root:root) → [[01_low_hanging_fruit]]
- [ ] 2. /etc/passwd & shadow — read hashes / writable → UID-0 → [[02_etc_passwd_shadow]]
- [ ] 3. sudo -l misconfigs → [[03_sudo_misconfig]]
- [ ] 4. SUID binaries — `find / -perm -u=s -type f 2>/dev/null` → [[04_suid_binaries]]
- [ ] 5. capabilities — `getcap -r / 2>/dev/null` → [[05_capabilities]]
- [ ] 6. writable files — sudoers.d / root scripts / PATH → [[06_writable_files]]
- [ ] 7. cron jobs (+ pspy) → [[07_Cron_Jobs]]
- [ ] 8. interesting files / creds — history, .ssh, .my.cnf, .bak → [[08_interesting_files]]
- [ ] 9. systemd services / timers → [[09_systemd_services]]
- [ ] 10. NFS no_root_squash → [[10_nfs_no_root_squash]]
- [ ] 11. kernel exploits (LAST — can crash) → [[11_kernel_exploits]]
- [ ] 12. docker / lxd / disk groups (`id`) → [[12_docker_lxd_groups]]

## Section → look for → route

| LinPEAS section                                                | look for                                        | go to                    |
| -------------------------------------------------------------- | ----------------------------------------------- | ------------------------ |
| `Sudo version`                                                 | version vs CVE (1.8.27 = Baron Samedit)         | [[03_sudo_misconfig]]    |
| `Checking 'sudo -l'`                                           | NOPASSWD, LD_PRELOAD, runnable bin              | [[03_sudo_misconfig]]    |
| `/etc/passwd` writable / readable `/etc/shadow`                | inline hash, write perms                        | [[02_etc_passwd_shadow]] |
| `SUID`                                                         | non-standard bin, `(Unknown SUID binary!)`      | [[04_suid_binaries]]     |
| `Capabilities` → `Files with capabilities`                     | `cap_setuid+ep` on non-root bin (ignore CapBnd) | [[05_capabilities]]      |
| `Interesting writable files` / `Writable root-owned exec`      | sudoers.d, root script, PATH dir                | [[06_writable_files]]    |
| `Cron jobs`                                                    | root job you can modify                         | [[07_Cron_Jobs]]         |
| `Searching passwords` / `Analyzing SSH Files` / `Backup files` | creds, id_rsa, .bak                             | [[08_interesting_files]] |
| `Container` / `id` groups                                      | docker/lxd, disk group                          | [[12_docker_lxd_groups]] |
| `Kernel Exploit Registry`                                      | matched CVEs (mind caveats)                     | [[11_kernel_exploits]]   |
## Traps

- SUID `--->` OS hints (SuSE / HP-UX / Solaris / Mac) = ancient noise; the real vector is often **unannotated**.
- Capabilities section is mostly `CapBnd` bitmask noise — only **Files with capabilities** counts.
- Kernel Registry may say _"reachable but public PoC blocked by disabled user namespaces"_ (`unpriv_userns_clone=0`) → that CVE is a likely dead end, don't burn time on it.
## If everything is dry

- re-read linpeas red/yellow line by line
- re-try every credential found → su / sudo / ssh