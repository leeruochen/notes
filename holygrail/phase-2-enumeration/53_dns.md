## Goal
Try a zone transfer (dumps every record), brute subdomains, and learn all hostnames for /etc/hosts + vhost fuzzing. DNS reveals the domain and subdomains.

> If records point at a DC (dc/ad/`_ldap._tcp`), you're in an AD engagement: DNS to the DC is a prerequisite for Kerberos/LDAP/BloodHound. Drive it from [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].

**Checklist**
- [ ] domain known? (from cert SAN / SMB / LDAP / HTTP redirect)
- [ ] zone transfer (AXFR) works?
- [ ] subdomains brute-forced (if no AXFR)?
- [ ] every discovered hostname added to /etc/hosts?

# Commands

### Step 1: Banner + find the domain
```bash
nmap -p53 -sC -sV $IP
# domain often comes from: LDAP rootDSE, SMB, Kerberos banner, HTTP redirect, TLS cert SAN
```

### Step 2: Zone transfer (AXFR)
```bash
dig axfr @$IP target.htb           # full zone dump if allowed
dig axfr @$IP                      # try without domain too
host -l target.htb $IP             # alternative AXFR
```

### Step 3: Query records
```bash
dig any @$IP target.htb
dig ns  @$IP target.htb            # nameservers
dig mx  @$IP target.htb            # mail servers
dig txt @$IP target.htb            # SPF/DKIM/misc
# nslookup interactive:
nslookup
> server $IP
> set type=any
> target.htb
```

### Step 4: Brute subdomains (no AXFR)
```bash
dnsenum --dnsserver $IP --enum -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt target.htb
gobuster dns -d target.htb -r $IP -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
# manual dig brute:
for s in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do
  r=$(dig +short @$IP "$s.target.htb"); [ -n "$r" ] && echo "$s.target.htb -> $r"; done
```

### Step 5: Reverse lookups + add hosts
```bash
for i in $(seq 1 254); do r=$(dig -x 10.10.11.$i @$IP +short); [ -n "$r" ] && echo "10.10.11.$i -> $r"; done
echo "$IP dc01.corp.local corp.local web01.corp.local" | sudo tee -a /etc/hosts
```

### Helpline
```bash
# READING RESULTS:
#   AXFR returns A/CNAME records = FULL zone dump -> add ALL hostnames to /etc/hosts, enumerate each
#     (HTB Cronos: AXFR -> admin.cronos.htb -> SQLi). "Transfer failed/refused" = blocked -> brute subs.
#   records show dc01/dc/ad = this is a Domain Controller -> pivot to LDAP(389)/Kerberos(88)/SMB(445).
#   MX -> mail.target -> check 25/110/143.   CNAME -> internal-app.target = hidden vhost -> browse it.
# WHY /etc/hosts MATTERS: many boxes redirect to a name; without the host entry the site won't load and
#   vhost fuzzing can't work -> add every discovered name.
# HIDDEN admin/dev subdomains usually have weaker security -> enumerate them separately (Cronos admin sub).
# no AXFR + brute finds nothing -> port 53 just confirms a DC; work LDAP/Kerberos/SMB and revisit if you
#   learn a new domain name elsewhere (cert SAN, SMB, LDAP rootDSE).
```
