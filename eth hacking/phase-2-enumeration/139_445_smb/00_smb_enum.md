## Goal
Fingerprint SMB, list shares, and check for the classic vulns. SMB(139/445) is a top OSCP vector: anonymous shares, user lists, and RCE all live here.

> In a domain, SMB yields a user list (RID cycling), GPP passwords, credential shells, and SAM/LSA/NTDS. Drive it from [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].

**Checklist**
- [ ] version + OS from nxc/nmap?
- [ ] signing disabled? (relay potential)
- [ ] anonymous / guest access? (-> anonymous_enum)
- [ ] MS17-010 / other SMB vuln? (-> smb_attacks)
- [ ] have creds? (-> authenticated_enum)

# Commands

### Step 1: Version + overview
```bash
nxc smb $IP                              # OS, hostname, domain, signing
enum4linux-ng -A $IP                     # broad one-shot enum
nmap -p139,445 -sV --script "smb-os-discovery,smb-security-mode,smb-enum-shares,smb-enum-users" $IP
```

### Step 2: Vuln scan
```bash
nmap -p445 --script "smb-vuln-ms17-010,smb-vuln-ms08-067,smb-vuln-*" $IP
nxc smb $IP -u '' -p '' -M zerologon      # ZeroLogon check (DCs)
```

### Helpline
```bash
# WHAT nxc TELLS YOU (the top [*] line):
#   OS build + hostname + domain  -> add the domain to /etc/hosts.
#   signing:False  -> possible relay target; use [[smb_attacks#SMB NTLM relay: signing disabled|SMB NTLM relay]].
#   (name):(domain) shows standalone vs domain-joined -> domain = go down the AD path.
# ROUTE: no creds -> anonymous_enum | have creds -> authenticated_enum | vuln flagged -> smb_attacks.
# nxc = netexec (crackmapexec successor), same syntax. install: pipx install netexec.
```
