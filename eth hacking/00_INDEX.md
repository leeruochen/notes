# HolyGrail — Field Manual Index

The exam field manual. Phases run in order, but the loop is cyclic: every new host, credential, or identity sends you back to enumeration.

## Phase order
```text
1 Active Recon      -> scope, host discovery
2 Enumeration       -> per-service notes; the port router lives here
3 Foothold          -> turn a primitive into a stable shell, pivot, research exploits
4 Privilege Escalation -> local Linux / Windows escalation
5 Active Directory  -> the multi-host domain loop
```

## Primary entry points
- **Start every box:** [[00_methodology]] (full port scan → route each port to its service note)
- **Web service:** [[00_WEB_METHODOLOGY]]
- **Unidentified port / odd service:** [[unknown_service_triage]]
- **Got code execution / a write primitive:** [[00_FOOTHOLD]]
- **Found an internal listener / second subnet:** [[internal_port_enum]] → [[pivoting_and_tunnelling]]
- **Named product + version (any target class):** [[public_exploit_research]]
- **Credential-bearing file (archive/db/key/doc):** [[artifact_triage]]
- **Linux privesc:** [[00 master checklist]]
- **Windows privesc:** [[00_master_checklist]]
- **Active Directory:** [[00_AD_METHODOLOGY]]
- **Stuck:** [[99_STUCK_RESET]]

## Active Directory sub-notes
[[00_AD_METHODOLOGY]] → [[01_credentials_and_spraying]] · [[02_bloodhound_and_acl_paths]] · [[03_credential_access]] · [[04_kerberos_advanced_paths]] (silver/PtT core; golden = persistence; delegation = reference-only)

## Reference (non-core)
- [[legacy_stack_buffer_overflow]] — legacy/reference only; writing BOFs is outside current OSCP scope (adapting exploits is inside it → [[public_exploit_research]]).

> [!note] Roles
> This is the live field manual. Teaching notes live in `learning/`, raw machine notes in `walkthroughs/`, source material in `sources/`. Keep those roles separate (see `CLAUDE.md`).

# VMWARE IP DOWN FIX: 

**On Windows (the host) restart the VMware services**
Open an Administrator Command Prompt or PowerShell and run:
```
net stop "VMware NAT Service"
net start "VMware NAT Service"
net stop "VMware DHCP Service"
net start "VMware DHCP Service"
```

Or open `services.msc`, find _VMware NAT Service_ and _VMware DHCP Service_, and restart both. That's usually all it takes.
**On the Kali VM, refresh networking if it still won't connect**
```
sudo systemctl restart NetworkManager
sudo ip link set eth0 down
sudo ip link set eth0 up
```


