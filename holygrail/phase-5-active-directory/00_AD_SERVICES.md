# AD Services Index

When you already know you are in an Active Directory environment, this is the protocol-by-protocol jump-off: which service answers which question. The ordered whole-domain loop lives in [[00_AD_METHODOLOGY]]; the port router for *any* box is [[00_methodology]].

> The scan-to-port workflow is unchanged: an open port still routes to its own service note in `phase-2-enumeration/`. This index just groups the AD-relevant ones in one place.

## Core AD protocols

| Port    | Service  | What it gives you in AD                                                                          | Note                                             |
| ------- | -------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| 53      | DNS      | resolve the domain/DC (a Kerberos/LDAP/BloodHound prerequisite); SRV records reveal the DC       | [[53_dns]]                                       |
| 88      | Kerberos | user enumeration, AS-REP roasting, Kerberoasting; ticket forgery/reuse                           | [[88_kerberos]] · [[04_kerberos_advanced_paths]] |
| 389     | LDAP     | dump users/groups/computers/ACLs (anon or authed), hunt passwords in attributes, feed BloodHound | [[389_ldap]]                                     |
| 139/445 | SMB      | shares, RID cycling for a user list, GPP passwords, credential shells, SAM/LSA/NTDS              | [[00_smb_enum]]                                  |
| 1433    | MSSQL    | domain-auth login, `xp_cmdshell` RCE, NetNTLM capture, a Silver-ticket target                    | [[1433_mssql]]                                   |
| 5985    | WinRM    | validate who can get a shell; evil-winrm; pass-the-hash                                          | [[5985_winrm]]                                   |
| 3389    | RDP      | GUI/shell with creds or PtH; `rdp-ntlm-info` leaks domain/host/OS                                | [[3389_rdp]]                                     |

## Attack-phase notes (not port-bound)
- [[00_AD_METHODOLOGY]] - the ordered multi-host loop (start here).
- [[01_credentials_and_spraying]] - validate creds domain-wide without lockouts; spray.
- [[02_bloodhound_and_acl_paths]] - turn a BloodHound edge into a verified privilege change.
- [[03_credential_access]] - SAM/LSA/LSASS/NTDS/DCSync, matched to your access.
- [[04_kerberos_advanced_paths]] - overpass / pass-the-ticket / silver / golden / delegation.

> [!important] Helpline
> This is a directory, not a method. Fix DNS + time first ([[53_dns]] · [[88_kerberos]]), then drive the engagement from [[00_AD_METHODOLOGY]]; open a service note when a port answers or when the loop sends you to it.
