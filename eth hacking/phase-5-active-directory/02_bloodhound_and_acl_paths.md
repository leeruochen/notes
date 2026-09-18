# BloodHound & ACL Attack Paths

Collecting a graph is easy; **turning an edge into a verified privilege change** is the skill. This note makes a BloodHound edge actionable: prerequisites, the abuse tool, the directory change to expect, a verification query, and a rollback.

## Goal
Convert a validated graph edge into a prerequisite checklist and a tested action, never blindly "follow the shortest path to DA".

## Open this when
- LDAP enumeration produced collection data ([[389_ldap]] Step 5), or you have a new owned principal to re-collect as.
- The AD loop asks "what can this identity control?" ([[00_AD_METHODOLOGY]] Step 7).

---

## Step 1: Collect (prerequisites + DNS/time)

| Prerequisite | Value |
| --- | --- |
| Have | - a valid domain credential (any user)<br>- the DC reachable, with working DNS + time |
| Need | a collector matching your BloodHound version |
| Result | graph data (users, groups, computers, sessions, ACLs) to import |

```bash
# BloodHound Legacy collector
bloodhound-python -u <user> -p '<password>' -d <domain_fqdn> -ns <dc_ip> -c all --zip
bloodhound-python -u <user> --hashes ':<ntlm_hash>' -d <domain_fqdn> -ns <dc_ip> -c all --zip

# BloodHound Community Edition uses the edition-specific executable with the same auth pattern
bloodhound-ce-python -u <user> --hashes ':<ntlm_hash>' -d <domain_fqdn> -ns <dc_ip> -c all --zip
# Windows, on a domain host:
#   SharpHound.exe -c All        (collector must match your BloodHound version)
```
> [!warning] Collection fails silently on DNS/time
> `bloodhound-python` / SharpHound need name resolution to the DC and low clock skew, exactly like Kerberos. If collection is empty or errors: fix `/etc/hosts` + resolver to the DC and `ntpdate` the DC first ([[53_dns]] · [[88_kerberos]]).
> **Version match:** BloodHound Community Edition needs a CE-compatible collector (the SharpHound bundled with your BloodHound release, or a CE-capable `bloodhound-python` / `bloodhound-ce-python`). Confirm current collectors at the official docs: <https://bloodhound.specterops.io/>.

---

## Step 2: Validate the data, then mark what you own
Before trusting any path:
- **Completeness:** did collection return users, groups, computers, sessions, ACLs? A partial collection hides and invents paths.
- **Mark owned:** set your current principal(s) and any compromised hosts as **Owned** so path queries start from reality.
- **Reachability:** a path is only real if you can actually reach the hosts/services each hop needs (check the matrix + [[pivoting_and_tunnelling]]).

Prioritise shortest paths from an **owned** principal to a high-value target, *after* the above, not before.

---

## Step 3: Universal edge workflow

Use this sequence for every edge. Do not jump from an arrow to an exploit command.

1. **Recognition:** record `source --EdgeName--> target`.
2. **Source control:** identify the source object type, how control was obtained, and prove that control is current.
3. **Target type:** identify whether the target is a User, Group, Computer, GPO, OU, or Domain.
4. **Interpretation:** classify the edge and translate it into target-specific candidate capabilities.
5. **Prerequisite:** confirm the exact effective right, reachability, service, and target condition.
6. **Action or route:** choose the least disruptive useful action, record the baseline and cleanup, then execute or follow the linked note.
7. **Verification:** prove the directory change first, then separately prove the resulting credential, membership, ticket, access, or execution.
8. **What I gained:** record the exact capability in [[00_AD_METHODOLOGY#Capability ledger: what did I gain?|the capability ledger]].
9. **Refresh and re-enumerate:** refresh the token, session, or ticket when required; collect again as the new identity; inspect the next path.

### Prove control of the source before using its edge

- A BloodHound node marked **Owned** is a note, not proof. Authenticate with the source credential or execute as that source identity.
- For a group source, prove your controlled principal is an effective member. Nested membership and a newly added group can require a fresh logon or new Kerberos ticket.
- For a computer source, prove control of the computer account and a usable secret. An administrator/SYSTEM session is useful only after the required machine credential or execution context is actually available.
- Re-read ACL edges from LDAP because graph data can be stale.

A path can be graph-valid but not presently executable because the source is not controlled, a token or session is stale, a host is unreachable, or the required service is unavailable.

```bash
nxc ldap <dc_ip> -d <domain_fqdn> -u <source_user> -p '<source_password>' -M daclread -o TARGET=<target>
impacket-dacledit -action read -principal "<source_principal>" -target "<target>" -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
```

> [!warning] Mutating action gate
> Record the original value, confirm the exact source and target, and prepare rollback before modifying membership, passwords, ownership, DACLs, delegation, or SPNs.

### Classify the edge before acting

| Class                             | Common edges                                                                               | Interpretation and route                                                                                                       |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Structural or inherited context   | MemberOf, Contains, GPLink, HasSession                                                     | Explains inheritance, policy scope, or session evidence. It may enable another edge but is not automatically an action.        |
| Login or access                   | AdminTo, CanPSRemote, CanRDP, SQLAdmin, ExecuteDCOM                                        | Validate authentication, reachability, and the exact service. Access is not necessarily an elevated shell.                     |
| Direct object control             | GenericAll, GenericWrite, AddMember, ForceChangePassword                                   | Dispatch by target type, then use the exact card.                                                                              |
| ACL-enabling                      | WriteDacl, WriteOwner, Owns, AllExtendedRights                                             | Often produces another right. Route that new primitive through the dispatcher again.                                           |
| Credential, ticket, or delegation | ReadLAPSPassword, ReadGMSAPassword, roasting, GetChanges, GetChangesAll, RBCD, CoerceToTGT | Validate the recovered credential or scoped ticket separately. Use [[03_credential_access]] or [[04_kerberos_advanced_paths]]. |

### Arrow direction and target type

Read `A --Edge--> B` as: **A has the named relationship to B**. The source is what you must control. The target tells you what the right can change. A visually adjacent node or reversed path is not equivalent.

| Edge | User target | Group target | Computer target | GPO target | OU target | Domain target |
| --- | --- | --- | --- | --- | --- | --- |
| GenericAll | ForceChangePassword or another verified identity-control option | AddMember, then refresh the token | ReadLAPSPassword when LAPS is present, or scope-gated RBCD and other computer-object options | Prove scope, then use [[#[CORE] Writable GPO and GpoEditDeleteModifySecurity\|Writable GPO]] | [REFERENCE] Descendant and inheritance analysis | Confirm GetChanges plus GetChangesAll, then route to DCSync |
| GenericWrite | Prove `servicePrincipalName`, then use [[#[CORE] Targeted Kerberoasting through GenericWrite or WriteSPN\|targeted Kerberoasting]] | Prove `member` is writable, then AddMember | Prove the exact computer attribute; RBCD is one possible route | Prove scope, then use [[#[CORE] Writable GPO and GpoEditDeleteModifySecurity\|Writable GPO]] | Prove the exact writable property | Prove the exact writable property |
| WriteDacl | Grant the smallest user right, verify it, route again | Grant WriteMembers, verify it, then AddMember | Grant an exact computer-object right, verify it, route again | [REFERENCE] Blast-radius gated | [REFERENCE] Inheritance analysis | Grant and verify both replication rights only when DCSync is the objective |
| WriteOwner or Owns | Take or confirm ownership, then grant a narrow User right and route again | Take or confirm ownership, then grant WriteMembers and route to AddMember | Take or confirm ownership, then grant an exact Computer right and route again | [REFERENCE] Ownership to DACL to policy route, with rollback | [REFERENCE] Ownership to DACL, with inheritance checks | [REFERENCE] Ownership to DACL, with domain-wide blast-radius gate |
| AllExtendedRights | Enumerate effective rights; verified reset right routes to ForceChangePassword | Enumerate exact extended rights; do not assume AddMember | Enumerate exact rights; ReadLAPSPassword may apply when LAPS is present | [REFERENCE] Inspect the exact right | [REFERENCE] Inspect the exact right and inheritance | Inspect exact rights; verified GetChanges plus GetChangesAll routes to DCSync |

> [!note] Dispatcher rule
> GenericAll, GenericWrite, and AllExtendedRights are not single exploits. The same label has different consequences on a User, Group, Computer, Domain, GPO, or OU.

### Common edges that should not become dead ends

| Edge             | Recognition and interpretation                                          | Prerequisite, action/route, and verification                                                                                                                                                                                                 |
| ---------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AddSelf          | The source can add itself, not an arbitrary user, to a Group            | Prove source control, then use [[#[CORE] AddMember\|AddMember]] with the source as `<controlled_user>`. Refresh the token and verify downstream access.                                                                                      |
| MemberOf         | The source inherits the Group's effective rights after a fresh logon    | Verify effective membership and token freshness, then inspect the Group's outgoing edges. No directory mutation is implied.                                                                                                                  |
| Contains         | An OU, Domain, or container contains the target                         | Inspect inheritance and outgoing control edges. Contains alone is structural context, not control of the child.                                                                                                                              |
| GPLink           | A GPO is linked to a container containing affected users/computers      | Verify link order, enforcement, filtering, and the affected target. `[REFERENCE]` unless a separate writable-GPO or WriteGPLink right is proven.                                                                                             |
| HasSession       | Collection observed a user session on a Computer                        | Prove host reachability and local Administrator/SYSTEM first, then use [[03_credential_access#LSASS session credentials: full dump, off-host parse, cleanup\|LSASS]]. Session data can be stale and does not guarantee recoverable material. |
| AdminTo          | The source is expected to have local administrator rights on a Computer | Authenticate to a reachable management service, run `whoami /all`, and enter [[00_AD_METHODOLOGY#Step 6: New Windows shell: low privilege or already local Administrator/SYSTEM?\|the Windows-shell router]].                                |
| CanPSRemote      | The source may open PowerShell Remoting on a Computer                   | Verify WinRM reachability and authentication through [[5985_winrm]]. This can be a non-admin shell, so verify `whoami /all`.                                                                                                                 |
| CanRDP           | The source may log on through RDP                                       | Verify TCP 3389, authentication, and interactive logon through [[3389_rdp]]. RDP access is not local administrator.                                                                                                                          |
| SQLAdmin         | The source has administrative SQL rights on a Computer                  | Verify MSSQL authentication and the exact SQL action through [[1433_mssql]]. Database execution identity and OS privilege are separate facts.                                                                                                |
| ReadLAPSPassword | The source can read a Computer's managed local Administrator secret     | Confirm the Computer uses LAPS, then perform one targeted retrieval/use test: `nxc winrm <target_ip> -d <domain_fqdn> -u <reader_user> -p '<reader_password>' --laps`. Record local service authorization separately.                        |
| ReadGMSAPassword | The source can read a gMSA managed password                             | `[REFERENCE]` Use BloodHound edge help for the current retrieval method, validate only the named service account, then route the resulting key through [[03_credential_access]]. Do not treat the edge as a shell.                           |
| WriteSPN         | The source can change a User's `servicePrincipalName`                   | Record the SPN baseline, then use [[#[CORE] Targeted Kerberoasting through GenericWrite or WriteSPN\|targeted Kerberoasting]]. Verify the TGS hash and restored SPN separately.                                                             |
| GpoEditDeleteModifySecurity | The source can edit, delete, or change security on a GPO       | Prove the GPO's linked scope and affected computers before using [[#[CORE] Writable GPO and GpoEditDeleteModifySecurity\|Writable GPO]]. The edge does not prove policy application or execution.                                      |

---

## Step 4: Common OSCP-relevant edge cards

The lookup tables above provide meaning and target dispatch. Each card below adds the shortest safe route: prerequisite -> action -> verification -> next.

### [CORE] AllExtendedRights

All control-access extended rights on one object. This is not GenericAll.

1. Prove source control, target type, and the live ACE.
2. Identify the exact target-specific right: password reset on a User, LAPS read on a Computer, or both replication rights on a Domain.
3. Route the proven right through the dispatcher. An unexplained extended right stops here.

**Verify:** Prove the downstream reset, LAPS read, replication pair, or other exact right.

**Capability:** Only the verified target-specific action.

**Next:** Refresh identity material if needed, update the capability ledger, and re-enumerate.

### [CORE] GenericAll

Broad control of one typed AD object. It does not automatically provide host execution, local administrator, or domain privilege.

1. Prove source control, target type, live ACE, baseline, and cleanup.
2. Prefer the smallest useful mutation.
3. Route by target: User -> [[#[CORE] ForceChangePassword|ForceChangePassword]]; Group -> [[#[CORE] AddMember|AddMember]]; Computer -> a proven attribute such as [[04_kerberos_advanced_paths#[REFERENCE] RBCD and msDS-AllowedToActOnBehalfOfOtherIdentity|RBCD]]; GPO -> [[#[CORE] Writable GPO and GpoEditDeleteModifySecurity|Writable GPO]]. OU and Domain remain `[REFERENCE]` unless the exact scope and right are understood.

**Verify:** Read back the exact attribute, membership, ACL, credential, or access change.

**Capability:** The verified target-specific action, not automatic Domain Admin.

**Next:** Refresh the token or ticket, record the capability, clean up, and re-collect.

### [CORE] GenericWrite

Property write over one object. It does not prove every attribute is writable.

1. Prove source control and the live ACE.
2. Identify the target type and exact writable attribute. Record its baseline and cleanup.
3. Route by attribute: User `servicePrincipalName` -> [[#[CORE] Targeted Kerberoasting through GenericWrite or WriteSPN|targeted Kerberoasting]]; Group `member` -> [[#[CORE] AddMember|AddMember]]; Computer delegation attribute -> [[04_kerberos_advanced_paths#[REFERENCE] RBCD and msDS-AllowedToActOnBehalfOfOtherIdentity|RBCD]]; GPO -> [[#[CORE] Writable GPO and GpoEditDeleteModifySecurity|Writable GPO]]. Unclear OU or Domain properties remain `[REFERENCE]`.

**Verify:** Read the same attribute before and after. On failure, re-check the property GUID, target DN, protection, and policy.

**Capability:** Only the primitive created by the verified attribute change.

**Next:** Use the primitive, restore the baseline, refresh identity material when required, and re-enumerate.

### [CORE] Targeted Kerberoasting through GenericWrite or WriteSPN

Search aliases: targeted Kerberoast, targetedKerberoast, WriteSPN, `servicePrincipalName`, temporary SPN, SPN-jacking.

Writable `servicePrincipalName` on a User can temporarily make that account Kerberoastable. It yields a crackable TGS candidate, not the plaintext password or a shell.

**Prerequisites:** Control the source, prove the SPN write right, record every existing SPN, and confirm Kerberos DNS/time. If a legitimate SPN exists, do not overwrite it; use ordinary [[88_kerberos|Kerberoasting]].

```bash
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> get object "<target_user>" --attr servicePrincipalName
```

1. Request only the named target. `targetedKerberoast.py` temporarily adds an SPN when needed, requests the TGS, then removes its temporary value.

```bash
targetedKerberoast.py -v -d '<domain_fqdn>' -u '<source_user>' -p '<source_password>' --dc-ip <dc_ip> --request-user '<target_user>' -o targeted_tgs.txt
```

2. Verify the result and the SPN baseline separately.

```bash
rg '^\$krb5tgs\$' targeted_tgs.txt
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> get object "<target_user>" --attr servicePrincipalName
```

**Verify:** The output contains a `$krb5tgs$` record for `<target_user>`, and the SPN list matches the baseline. LDAP access errors mean stale or ineffective write rights. No TGS means stop and check target state, DNS/time, KDC reachability, and tool input.

**Capability:** One offline password-cracking candidate. Validate recovered plaintext through [[01_credentials_and_spraying]] before treating it as a credential.

**Cleanup:** If a temporary SPN remains, restore the exact recorded baseline before continuing.

**Next:** Crack through [[88_kerberos]], validate the exact recovered pair through [[01_credentials_and_spraying]], then re-enumerate as the recovered identity.

### [CORE] AddMember

Adds a controlled principal to a Group. Membership requires verification and a fresh token before downstream rights appear.

1. Prove source control. Confirm `<controlled_user>` is not already a member and record the baseline.

```bash
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> get object "<target_group>" --attr member
```

2. Add the member and repeat the query.

```bash
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> add groupMember "<target_group>" "<controlled_user>"
```

**Verify:** `<controlled_user>` appears in `member`; a fresh logon or Kerberos authentication proves the downstream right.

**Capability:** Group membership only.

**Cleanup:**

```bash
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> remove groupMember "<target_group>" "<controlled_user>"
```

**Next:** Refresh the token, re-collect as `<controlled_user>`, and test only access implied by the group.

### [CORE] ForceChangePassword

Resets a target User's password without the old password. This is disruptive and gives a candidate domain credential, not authorization or a shell.

1. Prove source control and the live right. Confirm the target and password policy.
2. Accept that an unknown old password cannot be restored. Normal password changes route to [[01_credentials_and_spraying]].
3. Reset and test one plausible service.

```bash
bloodyAD -d <domain_fqdn> -u <source_user> -p '<source_password>' --host <dc_ip> set password "<target_user>" '<new_password>'
nxc smb <target_ip> -d <domain_fqdn> -u <target_user> -p '<new_password>'
```

**Verify:** Authentication succeeds as `<target_user>`. Failure can mean stale rights, password policy, protection, reachability, or the wrong target.

**Capability:** A validated domain credential for the target User.

**Cleanup:** No clean rollback exists unless the old password was known and explicitly restored. Record the reset and impact.

**Next:** Start a new session, enumerate plausible services and LDAP, then re-collect BloodHound.

### [CORE] WriteOwner and Owns

WriteOwner permits taking ownership; Owns means it already exists. Ownership enables DACL changes on that object, not GenericAll or execution.

1. Prove source control and the live ownership right. Record the original owner and back up the DACL.

```bash
impacket-owneredit -action read -target "<target>" -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
impacket-dacledit -action backup -target "<target>" -file <dacl_backup.json> -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
```

2. For WriteOwner, take ownership. For Owns, skip this write.

```bash
impacket-owneredit -action write -new-owner "<source_principal>" -target "<target>" -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
```

**Verify:** Repeat `owneredit -action read` and prove `<source_principal>` is the owner.

**Failure:** Re-check the right, source identity, target, LDAP policy, and object protection.

**Capability:** DACL modification on this object only.

**Cleanup:** Owner restoration may require `DS-Set-Owner` or `SeRestorePrivilege`; do not promise rollback. Keep the DACL backup.

**Next:** Grant the narrowest right through [[#[CORE] WriteDacl and WriteDACL|WriteDacl]], verify it, and route it again.

### [CORE] WriteDacl and WriteDACL

Modifies the target object's discretionary ACL. The new ACE is another primitive that must be verified and routed.

1. Prove source control and the live ACE. Back up the DACL; confirm principal, target DN, and the smallest useful right.
2. For a Group, grant membership write. For a User, `ResetPassword` routes to [[#[CORE] ForceChangePassword|ForceChangePassword]]. For a Domain, `DCSync` writes both replication rights and is a loud, high-impact mutation. Computer, GPO, and OU targets return to the dispatcher.

```bash
impacket-dacledit -action backup -target "<target_group>" -file <dacl_backup.json> -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
impacket-dacledit -action write -rights WriteMembers -principal "<controlled_principal>" -target "<target_group>" -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
impacket-dacledit -action read -principal "<controlled_principal>" -target "<target_group>" -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
```

**Verify:** The exact ACE appears. A successful DACL write is not a successful downstream action.

**Failure:** Stop and re-check source control, target DN, object protection, and LDAP policy.

**Capability:** The granted object right, such as membership write or password reset.

**Cleanup:**

```bash
impacket-dacledit -action restore -target "<target>" -file <dacl_backup.json> -dc-ip <dc_ip> '<domain_fqdn>/<source_user>:<source_password>'
```

**Next:** Route the new right, verify its result, refresh identity material, and re-enumerate.

### [CORE] DCSync, GetChanges, and GetChangesAll

Both Domain replication rights can request password material without an interactive DC shell. One right alone is insufficient for the normal DCSync route.

1. Prove source control plus `GetChanges` and `GetChangesAll` on the domain root.
2. Confirm DNS, reachability, and one named target account.
3. Use [[03_credential_access#DCSync (replication rights, no DC shell)|the canonical DCSync workflow]].

**Verify:** Replication output contains the requested account and hash or Kerberos key. Failure can mean incomplete rights, wrong domain context, connectivity, or LDAP/RPC policy.

**Capability:** Credential material for the requested principal, not validated access or a shell.

**Next:** Validate one plausible service, obtain fresh identity material, and re-enumerate.

### [CORE] Writable GPO and GpoEditDeleteModifySecurity

Search aliases: writable GPO, GpoEditDeleteModifySecurity, Group Policy Object, GPLink, SharpGPOAbuse, gpupdate, GPO refresh, local administrators.

Control of a GPO can change policy for computers in its effective linked scope. The edge does not prove that a chosen host receives the GPO, that policy refresh succeeds, or that code executes.

**Prerequisites:** Prove source control and the live GPO right. Identify the exact GPO, linked OU/site/domain, affected computers, link order, enforcement, security filtering, WMI filter, and current policy. Avoid broad or DC-linked GPOs when a narrower route exists.

1. Record the GPO ID, scope, filters, and baseline. `Backup-GPO` requires an existing directory.

```powershell
Import-Module GroupPolicy
Get-GPO -Name "<gpo_name>" | Select-Object DisplayName,Id,GpoStatus,WmiFilter
Get-GPOReport -Name "<gpo_name>" -ReportType Xml -Path "<baseline_report>.xml"   # inspect LinksTo and filters
Get-GPInheritance -Target "<ou_distinguished_name>"
Backup-GPO -Name "<gpo_name>" -Path "<existing_backup_directory>"
```

2. Choose the smallest authorised policy change. One common lab route adds a controlled domain user to the local Administrators group on affected computers.

```powershell
SharpGPOAbuse.exe --AddLocalAdmin --UserAccount <controlled_user> --GPOName "<gpo_name>"
```

3. Wait for normal policy refresh. Use `gpupdate /force` only from a legitimate session on an intended affected host.

```powershell
gpupdate /force
gpresult /scope computer /h "<gpresult_report>.html"
net localgroup Administrators
```

**Verify:** A new GPO report shows only the intended change; `gpresult` proves the target receives the GPO; local group output proves the effect. A successful LDAP/SYSVOL write alone is not policy application or execution.

**Failure:** Access denied means the graph right is stale or ineffective. GPO changed but host unchanged means link, filtering, precedence, scope, or refresh must be re-checked.

**Capability:** Local administrator only on computers that actually apply the modified GPO.

**Cleanup:** Restore the exact original GPO setting, allow or trigger an authorised refresh, and verify the local effect is gone. Keep the before/after reports and backup path.

**Next:** Record GPO edit, policy mutation, host application, and local administrator as separate boundaries. Enter [[00_AD_METHODOLOGY#Step 6: New Windows shell: low privilege or already local Administrator/SYSTEM?|the Windows-shell router]], use [[03_credential_access]] only from verified host access, then re-collect BloodHound.

### [REFERENCE] RBCD and msDS-AllowedToActOnBehalfOfOtherIdentity

RBCD can let a controlled computer principal impersonate a User to one target service context. It yields a scoped service ticket, not a password, TGT, or automatic host execution.

1. Control a source computer account and secret.
2. Prove `msDS-AllowedToActOnBehalfOfOtherIdentity` is writable on the target Computer.
3. Identify a valid target SPN and confirm Kerberos DNS/time.
4. Use [[04_kerberos_advanced_paths#[REFERENCE] RBCD and msDS-AllowedToActOnBehalfOfOtherIdentity|the scope-gated RBCD card]].

**Verify:** Read back the delegation attribute, obtain the expected service ticket, and prove only the matching service accepts it.

**Failure:** Re-check SPN, DNS, time, delegation state, ticket cache, and service authorization.

**Capability:** A Kerberos service ticket and possible authorization to that service.

**Cleanup:** Restore the recorded delegation-attribute baseline and remove temporary ticket material.

**Next:** Verify the SPN and principal with `klist`, test that service, and re-enumerate.

### [REFERENCE] CoerceToTGT

CoerceToTGT coerces a separate target computer to authenticate to a controlled unconstrained-delegation source so its machine TGT can be captured. It is not RBCD, a password, or automatically a human Domain Administrator TGT.

1. Prove Administrator or SYSTEM on the delegation source.
2. Confirm source and target identity, reachability, one applicable coercion primitive, ticket monitoring, and that the machine TGT advances the objective.
3. Use [[04_kerberos_advanced_paths#[REFERENCE] CoerceToTGT|the official-semantics progression]]. Stop when any prerequisite is absent.

**Verify:** Ticket monitoring and `klist` show the intended computer principal and expiry. No ticket, wrong principal, or inapplicable coercion means stop.

**Capability:** A TGT for the captured machine account. Directory and service authorization remain unproven.

**Cleanup:** Remove temporary ticket material.

**Next:** Use only an action authorised to that machine account, link to DCSync only when replication is separately verified, and re-enumerate.

---

## Common false positives / stop conditions
- A path through an edge you **don't actually hold** as the source (re-verify, Step 3 rule).
- A path requiring a host you can't reach (no session, no route).
- Treating "BloodHound drew an arrow" as "it will work", every mutation needs the prerequisite check + verification.
- If two edges fail verification with no new evidence, go to [[99_STUCK_RESET]].

> [!important] Helpline
> Own your principals, validate collection, *then* read paths. Every edge → prerequisite checklist + a verification query + a rollback. **Confirm the ACE a second way before mutating.** Delegation/ADCS edges are reference-only, not core exam actions. Docs: <https://bloodhound.specterops.io/>.
