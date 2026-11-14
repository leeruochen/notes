# Active Directory - Master Methodology

The protocol notes (DNS/Kerberos/LDAP/SMB/RDP/WinRM/MSSQL) tell you how to enumerate one service (indexed in [[00_AD_SERVICES]]). This note is the **ordered multi-host loop** that ties them together: given one credential or one foothold, what do you enumerate, validate, and escalate next, and how do you track it across a domain.

## Goal
Move from *one* valid domain credential or *one* domain-joined foothold to domain-wide access by always taking the **lowest-prerequisite** next step, re-enumerating after every new identity or host.

## The loop
```text
Establish domain / DNS / time context
 → inventory hosts, users, groups, reachable management services
 → inspect password / lockout policy
 → validate supplied or recovered credentials safely
 → obtain a host session
 → re-enumerate local AND domain context as the new identity
 → collect credential + relationship evidence (BloodHound, hives, tickets)
 → pivot when the next targets are only reachable through this host
 → choose the lowest-prerequisite path
 → verify access, update the matrix, repeat
```

## Required variables (fill these before attacking)
```text
DOMAIN_FQDN   = <domain_fqdn>        NETBIOS  = <domain_netbios>
DC_HOST/IP    = <dc_fqdn> / <dc_ip>  DNS      = usually the DC
TIME_SOURCE   = the DC (Kerberos needs <5 min skew)
CURRENT CRED  = user + secret + context (domain? local? which host?)
SUBNET(S)     = reachable networks (a second subnet => pivot, see below)
```
> [!warning] DNS and time are prerequisites, not optionals
> In AD, **DNS must point at the domain (usually the DC)** or names won't resolve, and **Kerberos fails if clock skew > 5 min**. Fix these *first*: most "tool is broken" moments are actually DNS or skew. → [[53_dns]] · [[88_kerberos]]

---

## Step 1: Domain, DNS, time context
```bash
echo "<dc_ip> <domain_fqdn> <dc_fqdn>" | sudo tee -a /etc/hosts
sudo ntpdate -s <dc_ip>            # or faketime for one command, see 88_kerberos
nxc smb <dc_ip>                    # banner: domain, hostname, OS, signing, SMBv1
```
Get the domain FQDN + NetBIOS name from: SMB banner, LDAP rootDSE, Kerberos, an HTTP redirect, or a TLS cert SAN. Identify the DC (it's the host with 88/389/445/53 together).

---

## Step 2: Inventory hosts, users, groups, reachable services
Do as much as the current access allows; redo it after every new credential.
```bash
# users/groups/computers via LDAP (anon or authed) -> feed the user list + BloodHound
#   -> [[389_ldap]]
# RID cycling when LDAP anon bind is denied -> [[00_smb_enum]] / anonymous_enum
# which management services answer, per host: 445 / 5985 / 3389 / 1433 / 88
nxc smb  <subnet_or_ip>            # host discovery + SMB
nxc winrm <subnet_or_ip>          # who exposes WinRM
nxc ldap <dc_ip> -u <user> -p '<password>' --users --groups
```
Record every host, its role hint (DC / SQL / web / workstation), and which services it exposes → build the **matrix** below.

---

## Step 3: Password / lockout policy (before any spray)
```bash
nxc smb  <dc_ip> -u <user> -p '<password>' --pass-pol      # authenticated
nxc ldap <dc_ip> -u <user> -p '<password>' --pass-pol
# anonymous, sometimes:
nxc smb  <dc_ip> --pass-pol
```
Lockout threshold and observation window decide whether, and how slowly, you may spray. **Never spray before checking this.** Full discipline → [[01_credentials_and_spraying]].

---

## Step 4: Validate credentials safely
A credential is **valid / valid-no-access / local-admin**: three different results. Test on plausible services only, tracking every attempt in the matrix.
```bash
nxc smb   <target> -u <user> -p '<password>'   # valid? (+) local admin? (Pwn3d!)
nxc winrm <target> -u <user> -p '<password>'   # can this user get a shell here?
nxc mssql <target> -u <user> -p '<password>'   # DB auth, if 1433
```
`(+)` = authentication works. `Pwn3d!` = **code execution on that protocol**, and its meaning is protocol-specific: on **SMB** it most likely means local admin; on **WinRM** it only means command execution (Remote Management Users *or* admin), so it is **not** proof of local admin, confirm with `whoami /groups` in-session before treating it as such. `(+)` without `Pwn3d!` = the cred is real but not privileged there, try it elsewhere. Protocol-specific reading + spraying → [[01_credentials_and_spraying]].

---

## Step 5: Obtain a host session
Pick the service the identity is actually allowed to use (from the matrix):
- Remote Management Users → WinRM ([[5985_winrm]], `evil-winrm`)
- Local admin (`Pwn3d!`) → SMB exec / psexec / wmiexec ([[smb_attacks]])
- Remote Desktop Users → RDP ([[3389_rdp]])
- MSSQL sysadmin → `xp_cmdshell` ([[1433_mssql]]) → [[00_FOOTHOLD]]

---

## Step 6: New Windows shell: low privilege or already local Administrator/SYSTEM?

### TLDR

Classify the shell before choosing the next workflow. A low-privilege shell enters the normal manual Windows sweep. `NT AUTHORITY\SYSTEM` or a verified high-integrity local Administrator context is already the end of local privilege escalation. From that point, collect credentials, sessions, artifacts, and internal reachability, then return to the AD loop.

### When you see this

Any new WinRM, RDP, SMB-exec, MSSQL, web-service, or reverse-shell session on Windows.

### Check first

Run the native baseline yourself. Do not trust a tool banner or `Pwn3d!` alone.

```batch
whoami /all
whoami /priv
whoami /groups
hostname
tasklist /v
netstat -ano
route print
ipconfig /all
```

Interpret integrity and group output in [[01_orientation]]. A medium-integrity member of Administrators is not yet an elevated process, see [[09_uac]]. A WinRM-capable user may be non-admin.

### If confirmed

```text
standard or low privilege
-> [[00_master_checklist]] manual sweep
-> [[03_services]] + [[04_scheduled_tasks]] + [[05_stored_credentials]] + [[07_internal_services]]

high-integrity local Administrator or NT AUTHORITY\SYSTEM
-> STOP local privilege escalation
-> [[03_credential_access]] for SAM, LSA, LSASS, or DC-specific sources
-> [[05_stored_credentials]] and [[artifact_triage]] for operational artifacts
-> [[07_internal_services]] for listeners and services
-> [[pivoting_and_tunnelling]] when routes reveal another subnet
-> validate new material narrowly, then re-enumerate AD and BloodHound
```

Automated enumeration is an optional gap-check only after this manual baseline.

### Expected result

You can state the current identity, integrity level, local group membership, execution identity, host/domain context, active sessions, and reachable networks. Access denied from a non-admin WinRM session is not proof that a source does not exist. It means the session lacks the required local privilege.

### What did I gain?

Record either `interactive login path`, `host execution and execution identity`, or `local administrator`. Do not promote any of them to `domain privilege` without separate evidence.

### Next

After any new credential, hash, ticket, membership, host session, or directory right:

1. record its exact class in the [[#Capability ledger: what did I gain?|capability ledger]];
2. validate only plausible services or objects;
3. create a fresh logon session or refresh Kerberos material when needed;
4. rerun identity, service, LDAP, share, and BloodHound checks as the new principal;
5. update the host, service, and credential matrix.

> [!important] Local admin here ≠ domain privilege
> Being local admin (or even SYSTEM) on one member host does **not** make you a domain admin. It gives you that host's secrets (local hashes, LSASS sessions, cached creds) which you then *reuse* to reach the next identity. Keep the two escalations separate.

---

## Capability ledger: what did I gain?

Use one exact class after every observation or action. Scope it to the identity, object, host, service, SPN, and expiry that you actually verified.

| Capability class | Record only after you verify |
| --- | --- |
| **directory visibility** | which LDAP or BloodHound data the principal can read |
| **structural relationship** | membership, containment, GPO link, trust, or session context exists |
| **credential or NTLM hash** | usable secret material for one named identity |
| **group membership** | directory membership and, after a fresh logon, effective membership |
| **AD object control** | the exact object, attribute, or ACL right you can modify |
| **Kerberos TGT** | principal, realm, validity, and ticket cache |
| **Kerberos service ticket and exact SPN scope** | principal, exact SPN, host naming, and expiry |
| **service authentication** | a service accepted the identity or credential |
| **service authorization** | the accepted identity can perform a named service action |
| **interactive login path** | RDP, WinRM, or another session route works |
| **local administrator** | high-integrity local Administrator on one host |
| **host execution and execution identity** | a command ran and `whoami` proved its OS identity |
| **domain replication rights** | the effective `GetChanges` plus `GetChangesAll` pair is present |
| **domain privilege** | a domain-level privileged action actually succeeded |

Rules that prevent false promotion:

- Authentication is not authorization.
- A credential is not a shell.
- A ticket is scoped and is not automatically a shell.
- AD object control is not automatically local administrator or host execution.
- Local administrator or SYSTEM on one host is not domain privilege.
- A directory mutation is not a completed attack until the resulting capability is verified separately.

---

## Step 7: Collect credential & relationship evidence
From a host session, harvest what lets you become someone new:
- **Relationships / attack paths** → [[02_bloodhound_and_acl_paths]] (works as any authenticated user)
- **Host-local credential material** (SAM, LSA, LSASS, **offline NTDS**) → [[03_credential_access]]. **Host/DC-privilege-gated:** needs appropriate local privilege on the machine (Administrators / High integrity, or `SeDebug`/`SeBackup`; offline NTDS needs admin/`SeBackup` access **on a DC**). Confirm `whoami /groups` first: a WinRM `Pwn3d!` that is only a Remote Management Users member cannot read these.
- **DCSync** → [[03_credential_access]]. Gated on **directory replication rights** (`DS-Replication-Get-Changes` **and** `-Get-Changes-All`), **not** on local administrator access and **not** on an interactive shell on the DC. A non-admin principal that holds those rights can DCSync remotely, over the network.
- **Roastable accounts** (AS-REP / Kerberoast) → [[88_kerberos]] (works with any valid domain cred)

---

## Step 8: Pivot when required
If the next targets sit on a subnet only the current host can reach, tunnel through it before continuing the loop → [[pivoting_and_tunnelling]] (route the internal subnet; mind DNS to the DC).

---

## The host / service / credential matrix
Keep this live: it's how you avoid repeat attempts and see the next-lowest-prerequisite move.

| Host | Role | 445 | 5985 | 3389 | 1433 | Credential | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `<dc_fqdn>` | DC | ✓ | ✓ | | | `<domain_user>:<secret>` | valid-no-access |
| `<member_fqdn>` | member | ✓ | ✓ | ✓ | | `<domain_user>:<secret>` | local admin (Pwn3d!) |
| `<sql_fqdn>` | member | ✓ | | | ✓ | `<service_user>:<secret>` | untested |

Status vocabulary: **untested · valid-no-access · interactive access · local admin · domain privilege**.

---

## Decision routing (observation → note)
| You have / observe                                                          | Go to                                                                               |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| DNS/time failing, names won't resolve                                       | [[53_dns]] · [[88_kerberos]]                                                        |
| a user list, no creds                                                       | AS-REP roast → [[88_kerberos]]; then spray → [[01_credentials_and_spraying]]        |
| one valid cred                                                              | Kerberoast → [[88_kerberos]]; validate everywhere → [[01_credentials_and_spraying]] |
| anon/authed LDAP                                                            | dump users/desc, feed BloodHound → [[389_ldap]]                                     |
| SMB shares / RID cycling                                                    | [[00_smb_enum]] · authenticated_enum                                                |
| a graph edge (ACL/group/reset)                                              | [[02_bloodhound_and_acl_paths]]                                                     |
| local admin / SYSTEM on a host, or admin/`SeBackup` on a DC (NTDS)          | harvest host-local creds → [[03_credential_access]]                                 |
| replication rights (`GetChanges`+`GetChangesAll`), no admin/DC shell needed | DCSync → [[03_credential_access]]                                                   |
| second subnet only via this host                                            | [[pivoting_and_tunnelling]]                                                         |
| stuck                                                                       | [[99_STUCK_RESET]]                                                                  |

> [!important] Helpline
> Fix DNS + time first. One cred is **valid**, not **admin**: prove access per host. Re-enumerate after every new identity. **Local-admin-on-a-host and domain-privilege are different ladders.** Lowest-prerequisite path next; update the matrix every time.
