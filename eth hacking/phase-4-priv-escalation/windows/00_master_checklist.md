Local Windows privilege escalation: you already have a shell - turn a low-privileged Windows account into `NT AUTHORITY\SYSTEM` (or a local admin) by abusing what the host already trusts. This page is the map; each row routes to a focused note. Active Directory attacks are out of scope here.

## Scope & stop condition

- Run these steps **locally**, after you already have an authorised shell on the Windows host.
- **Stop condition:** run `whoami`. If it prints `NT AUTHORITY\SYSTEM`, local privilege escalation is already complete - collect proof and stop. A service-borne RCE (IIS / SQL / mail server) often lands you as SYSTEM already. [BT]
- Confirm identity from a command **you** run - never trust an exploit banner or another person claiming "you are SYSTEM". [BT]

## Validation legend

- `[BT]` - battle-tested / grounded by observed practical evidence in this vault.
- `[T]` - taught and checked for understanding, **not yet** proven end-to-end on a real Windows privesc path. Treat `[T]` as sound technique, but verify each result yourself.

Most vector routes below are `[T]`: correct course technique, not yet exercised on a live box.

## How to work this list

1. **Manual baseline first.** Read native evidence (`whoami /priv`, `whoami /groups`, `systeminfo`, service / task / port lists) and interpret it yourself - that is what decides your route.
2. **Automated gap-check second.** Then run winPEAS to catch what you missed, and keep doing targeted manual work while it runs. Seatbelt, PowerUp / SharpUp, and AccessChk are targeted cross-checks, **not** proof - every finding needs manual confirmation. Tool roles and usage live in [[01_orientation]].

## The sweep (top to bottom)

- [ ] 1. **Identity + stop check** - `whoami` · `hostname`. SYSTEM = done. [BT] → [[01_orientation]]
- [ ] 2. **Groups, integrity & privileges** - `whoami /groups` · `whoami /priv`; notice Administrators, service identities, dangerous privileges. [T] → [[01_orientation]] then [[02_token_privileges]]
- [ ] 3. **OS build / arch / patches** - `systeminfo` (record the **build number** and architecture). [T] → [[01_orientation]] (kernel evidence used later in [[08_kernel_exploits]])
- [ ] 4. **Processes, services, software, ports** - `tasklist /v` · `Get-CimInstance win32_service` · `netstat -ano`. [T] → [[01_orientation]]
- [ ] 5. **Token-privilege paths** - SeImpersonate / SeAssignPrimaryToken / SeBackup / SeRestore … (highest-signal, low-risk - chase first). [T] → [[02_token_privileges]]
- [ ] 6. **Stored credentials** - config / history / registry / `cmdkey`; reuse them, then re-sweep as the new user. [T] → [[05_stored_credentials]]
- [ ] 7. **Services** - privileged service + controllable link + usable trigger. [T] → [[03_services]]
- [ ] 8. **Scheduled tasks** - higher-priv run-as + repeatable trigger + controllable action. [T] → [[04_scheduled_tasks]]
- [ ] 9. **Writable paths & registry** - PATH / search-order, autoruns / startup, weak registry keys, AlwaysInstallElevated. [T] → [[06_writable_paths_and_registry]]
- [ ] 10. **Internal-only services** - loopback listeners with an escalation hypothesis; tunnel only if needed. [T] → [[07_internal_services]]
- [ ] 11. **Kernel / userland exploits** - LAST resort; candidates only, validate build / patch / arch / risk. [T] → [[08_kernel_exploits]]
- [ ] 12. **Stuck?** - change the hypothesis, re-enumerate after any identity change. → [[99_windows_stuck_reset]]

## Finding → interpretation → route

| You observe                                                                           | It means                                           | Go to                              |
| ------------------------------------------------------------------------------------- | -------------------------------------------------- | ---------------------------------- |
| `whoami` = `NT AUTHORITY\SYSTEM`                                                      | already SYSTEM - done                              | stop, collect proof                |
| `whoami` = `IIS APPPOOL\…`, `NT AUTHORITY\LOCAL SERVICE` / `NETWORK SERVICE`, `svc_*` | service identity - usually holds SeImpersonate     | [[02_token_privileges]]            |
| `whoami /priv` lists `SeImpersonate` / `SeAssignPrimaryToken`                         | impersonation → Potato                             | [[02_token_privileges]]            |
| `whoami /priv` lists `SeBackup` / `SeRestore` / `SeTakeOwnership` / `SeDebug`         | read / write / own any object; dump hashes         | [[02_token_privileges]]            |
| `whoami /groups` = Administrators (but medium integrity)                              | admin token behind UAC - elevate to high integrity | [[09_uac]]                         |
| `whoami /groups` = Backup Operators                                                   | read/write any file → dump SAM + SYSTEM            | [[02_token_privileges]]            |
| password in a file / history / registry                                               | reuse it on every user and service                 | [[05_stored_credentials]]          |
| writable service binary / config / DLL, runs as SYSTEM, you can (re)start it          | service hijack                                     | [[03_services]]                    |
| task runs as a higher-priv user, repeats, action file writable                        | scheduled-task hijack                              | [[04_scheduled_tasks]]             |
| writable dir in PATH / autorun / `AlwaysInstallElevated`=1 (both hives)               | consumer-trust write                               | [[06_writable_paths_and_registry]] |
| `netstat -ano` shows a `127.0.0.1:<port>` listener only                               | internal service - investigate / tunnel            | [[07_internal_services]]           |
| old build, missing security KBs                                                       | kernel candidate (last resort)                     | [[08_kernel_exploits]]             |

## Traps

- `NT AUTHORITY\…` is an authority namespace, **not** a rank. Only `NT AUTHORITY\SYSTEM` is SYSTEM; `LOCAL SERVICE` / `NETWORK SERVICE` are limited local identities. → [[01_orientation]]
- `Disabled` beside a privilege in `whoami /priv` does **not** mean useless - if it is **listed**, it is assigned. But "assigned" does not guarantee a given technique's other prerequisites are met. → [[02_token_privileges]]
- Administrators-group membership ≠ high integrity. UAC hands admins a filtered (medium) token; you may still need to elevate.
- "Writable" is only a vector if a **more-privileged** process reads, loads, or executes the object.
- A service running as SYSTEM is **necessary but not sufficient** - you also need a controllable link and a usable trigger.
- winPEAS / PowerUp findings are **leads**, not proof; they miss real vectors and raise false positives - confirm manually.
- Kernel exploits can crash the box; keep them last.

Still stuck after a full pass? → [[99_windows_stuck_reset]]
