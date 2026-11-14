The cheapest win on Windows is a password someone left behind. Find it, use it through a real logon path, then start the whole sweep again as the new identity.

## Goal
Find secrets or saved authentication material, use them via an available logon / execution path, then re-enumerate as the new user.

## Tips
Two ideas run this note: **reuse** (try a found password on every user and service) and **cyclical** (a new identity means re-running identity / groups / priv / services / tasks / files / creds from scratch - do not keep enumerating as the old user). RDP and WinRM membership are *access enablers*, not privilege escalation by themselves.

Checklist:
- [ ] PSReadLine history + transcripts read? (`Clear-History` does NOT clear the PSReadLine file)
- [ ] config / unattend / web / db files searched?
- [ ] registry + Winlogon autologon checked?
- [ ] `cmdkey /list` for saved creds?
- [ ] found password tried on EVERY user + service?

# Commands

### Step 1: PowerShell artifacts (highest yield)
```powershell
Get-History                                                     # session buffer (often empty)
(Get-PSReadlineOption).HistorySavePath                          # the real, persistent history file
type (Get-PSReadlineOption).HistorySavePath                     # survives Clear-History
Get-ChildItem C:\Users\ -Recurse -Filter *transcript*.txt -EA SilentlyContinue   # transcripts
```

### Step 2: files, environment, registry
```batch
findstr /si password *.txt *.ini *.config *.xml
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"   :: DefaultUserName / DefaultPassword (autologon)
cmdkey /list                                                              :: saved credentials
```
```powershell
$roots = @(
    $env:USERPROFILE,
    'C:\Users\Public',
    'C:\ProgramData',
    'C:\Scripts',
    'C:\inetpub',
    'C:\xampp'
) | Where-Object { Test-Path $_ }
$extensions = @('.zip','.7z','.rar','.rdg','.vbs','.ps1','.bat','.cmd','.config','.xml','.json','.ini','.db','.sqlite','.sqlite3','.kdbx')

Get-ChildItem -Path $roots -File -Recurse -Force -ErrorAction SilentlyContinue |
    Where-Object { $extensions -contains $_.Extension.ToLowerInvariant() } |
    Select-Object FullName,Length,LastWriteTime

 Get-ChildItem env:                                             # env vars may hold secrets
Get-CimInstance Win32_Service | Select Name,PathName          # service command lines (creds in args?)
```
These roots are examples, not universal truth. Start with observed paths that exist, note access-denied areas, include other readable profiles or administrator-readable backups only when your current privilege justifies it, and widen deliberately. Hotspots include `web.config`, `*.config`, `unattend.xml` under `C:\Windows\Panther\`, `my.ini`, and KeePass `.kdbx`. SAM/SYSTEM backups may exist in `C:\Windows\Repair\` or `C:\Windows\System32\config\RegBack\`.

Route discoveries once:

- archives (`ZIP`, 7z, RAR): copy, hash, list, identify, and crack only if appropriate in [[artifact_triage]];
- VBS, PowerShell script, batch, and configuration files: inspect read-only for credentials, command lines, shares, connection strings, and target names;
- `.rdg`: follow [[#RDCMan, Remote Desktop Connection Manager, and .rdg]];
- `confCons.xml`: follow [[#mRemoteNG and confCons.xml]];
- `.db`, `.sqlite`, and `.sqlite3`: [[artifact_triage#SQLite: `.db`, `.sqlite`, `.sqlite3`|generic SQLite triage]];
- `.kdbx`: use the existing conversion/cracking flow in [[artifact_triage]], then [[artifact_triage#KeePass/KeePassXC: after cracking|KeePass/KeePassXC post-crack handling]];
- unknown formats: [[artifact_triage]];
- every identity or secret: [[01_credentials_and_spraying]], then re-enumerate.

## Exploit

### Case 1: Plaintext password found → use it via a real logon path
**Check:** which path fits the target user?

| Target user can… | Use |
|---|---|
| interactive / GUI session | `runas /user:<HOST>\<user> cmd` (GUI / RDP session only) |
| Remote Desktop Users | RDP to 3389 |
| Remote Management Users | `evil-winrm -i <TARGET> -u <user> -p <pass>` |
| saved cred present (`cmdkey`) | `runas /savecred /user:<HOST>\<user> cmd` |

**Abuse & verify:**
```batch
runas /user:<HOST>\<user> cmd      :: password is PROMPTED, never on the command line
```
Then in the new context: `whoami` → and **re-run the full sweep** ([[00_master_checklist]]).
**Dead end / pivot:** in a bind shell / WinRM the `runas` password prompt fails or hangs (no interactive session) → use RDP or evil-winrm instead. RDP / WinRM membership grants *access* only; it is not privilege escalation on its own.

### Case 2: Autologon / saved credentials
**Check:** Winlogon `DefaultUserName` + `DefaultPassword`, or `cmdkey /list` shows a stored credential.
**Abuse:** `runas /savecred /user:<user> cmd` (uses the stored secret without a prompt), or reuse the autologon password directly.
**Verify / pivot:** as Case 1.

### Case 3: SAM + SYSTEM hive access
If you can read the SAM and SYSTEM hives - via `SeBackupPrivilege` / Backup Operators, or a leftover backup in `C:\Windows\Repair\` - extract local hashes and pass-the-hash. **Detailed route: [[02_token_privileges]] Case 2.** SAM + SYSTEM provide local password-hash and decryption material analogous in *purpose* to `/etc/shadow` and its key, but the storage / extraction model is different.

### Case 4: Third-party application credential stores (category workflow)
The built-in checks above cover Windows' own stores; installed apps keep secrets too. Work it by *category*, not by memorising products.

**Where to look:** enumerate likely vendor/app data directories:
```powershell
Get-ChildItem "$env:APPDATA","$env:LOCALAPPDATA" -Directory      # per-user app data
Get-ChildItem "C:\ProgramData" -Directory                         # machine-wide app data
Get-ChildItem "C:\Program Files","C:\Program Files (x86)" -Directory
# then read structured files inside, read-only:
Get-ChildItem <appdir> -Recurse -Include *.config,*.xml,*.json,*.ini,*.db,*.sqlite,*.kdbx -EA SilentlyContinue
```

**Classify the protection before trying to "decrypt" anything** (found ≠ recovered):

| Protection | Sign | What it takes to recover |
|---|---|---|
| plaintext / encoded | value in a config/history/db | read / decode (base64 etc.) |
| per-user **DPAPI** | blob tied to the logged-on user | that user's context (or their masterkey) |
| **machine** DPAPI | blob usable by any admin on the host | admin/SYSTEM on this host |
| **master-password** vault | app prompts for one password | crack that password (a hash) |
| **documented default key** | app "encrypts" with a published/static key | decrypt with the known key |
| hash | `$…`/hex user table | crack the hash |

**Then:**
- Prefer a **source-reviewable** parser/decryptor and understand its assumptions before trusting output. Verify the recovered value by actually authenticating with it.
- The identify → classify → convert → crack → **verify** mechanics live in [[artifact_triage]]; don't duplicate them here.
- **Connect the app to the host:** which service account runs it, its binary path, any loopback listener ([[07_internal_services]]), and its version (→ [[public_exploit_research]] if the app itself is exploitable).
- Route every recovered credential into the reuse loop: try it on every user + service, then re-sweep ([[00_master_checklist]]).

### RDCMan, Remote Desktop Connection Manager, and .rdg

#### TLDR

An RDG file is XML that can reveal server names, usernames, domains, gateways, credential profiles, and inherited settings. Passwords can be absent, inherited, DPAPI-protected, or certificate-protected. Finding `.rdg` does not guarantee a recoverable password.

#### When you see this

The scoped search returns an `.rdg` file or the host has evidence of Remote Desktop Connection Manager use.

#### Check first

Copy and hash the file with [[artifact_triage]], record its owner and path, then inspect the XML read-only. Identify referenced profiles, inheritance, encryption method, and useful hosts before attempting recovery.

#### If confirmed

- User-context DPAPI requires the matching user's context or master-key material.
- Certificate encryption requires the relevant private key.
- Feed hostnames and domains into the host/service matrix even if no password is recoverable.
- Use a maintained, source-reviewable parser only after its supported format matches the file.

#### Expected result

You recover connection metadata and, only when the required protection context is available, a candidate credential. Parser failure commonly means wrong user context, missing key material, unsupported format, or no stored password.

#### What did I gain?

Record `directory visibility` for infrastructure metadata or `credential or NTLM hash` only after recovery and verification. Finding encrypted data alone is neither.

#### Next

Validate one recovered identity/service pair through [[01_credentials_and_spraying]], update the matrix, then re-enumerate. Route unknown protection or master-key artifacts to [[artifact_triage]].

### mRemoteNG and confCons.xml

#### TLDR

`confCons.xml` is an mRemoteNG connection file. It may contain hosts, users, domains, and encrypted passwords. A Base64-looking `Password` field is not plaintext, and default-key behavior is version and configuration dependent.

#### When you see this

The scoped search finds `confCons.xml` or one of its backup copies in readable user or application-data locations.

#### Check first

Copy and hash the file, preserve its path and owner, then inspect root XML metadata for format/version, `BlockCipherMode`, KDF settings, full-file encryption, and evidence of a custom master password.

#### If confirmed

Use the source-reviewable `mremotedec` parser only when its documented CBC/GCM support matches the observed metadata:

```bash
mremotedec <confCons.xml>
mremotedec -p '<known_or_recovered_master_password>' <confCons.xml>
```

Use the no-password invocation only when the file metadata and parser documentation justify mRemoteNG's documented default-key behavior. If a custom master password is required, treat it as a master-password artifact in [[artifact_triage]]. Do not guess online.

#### Expected result

Successful parsing returns connection records whose host, username, domain, and secret can be checked separately. Authentication-tag, padding, or decryption failures usually mean the wrong master password, wrong encryption mode, full-file encryption, corruption, or an unsupported version.

#### What did I gain?

Record connection metadata as `directory visibility`. Record a plaintext secret as `credential or NTLM hash` only after one controlled authentication check.

#### Next

Send validated pairs to [[01_credentials_and_spraying]], update the matrix, and re-enumerate services, shares, LDAP, and BloodHound as the new principal.

## Correcting common misconceptions
- A found encrypted-looking blob is **not** a credential until you recover and *verify* it. Classify the protection first (DPAPI vs default-key vs hash), and do not assume it is crackable.
- `runas` does **not** accept a password on its command line; it prompts, and the prompt fails / hangs in a non-interactive shell (bind shell, WinRM).
- **Remote Desktop Users** and **Remote Management Users** are access paths, not privilege escalation.
- After switching user, **re-enumerate everything** - new permissions expose new files, services, and tasks. Do not keep hunting as the old user.

## Helpline:
1. PSReadLine truth
```
Clear-History clears only Get-History (the session buffer).
The persistent file survives:
  (Get-PSReadlineOption).HistorySavePath
  %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

2. Reuse principle
```
one password -> try it on EVERY local user + service (RDP, WinRM, DB, service accounts).
most user->user and user->admin jumps on real boxes are reuse, not exploits.
```

[HackTricks - Windows Local Privilege Escalation](https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html) and [PayloadsAllTheThings / InternalAllTheThings - Windows Privilege Escalation](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/) - deeper credential-artifact locations and troubleshooting. Local host only: this note stays on the standalone host. In a **domain**, continue with the source-selection table in [[03_credential_access]] (local SAM vs LSA vs LSASS vs NTDS vs DCSync) and the loop in [[00_AD_METHODOLOGY]].
