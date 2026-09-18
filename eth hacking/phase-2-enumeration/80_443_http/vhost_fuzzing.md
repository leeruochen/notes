## Goal
Find virtual hosts / subdomains serving different apps on the same IP. Many boxes hide the real app behind a Host header.

**Checklist**
- [ ] base domain known? (cert, redirect, DNS)
- [ ] base domain in /etc/hosts?
- [ ] fuzzed Host header, filtered by response size?
- [ ] every hit added to /etc/hosts + enumerated?

# Commands

### Fuzz vhosts (Host header)
```bash
ffuf -u http://$IP -H "Host: FUZZ.<domain>" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs <default_size>
```

### gobuster vhost
```bash
gobuster vhost -u http://<domain> --append-domain \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### Helpline
```bash
# -fs / --exclude-length: filter out the default page size, keep only different responses
# every hit -> echo "$IP sub.domain" | sudo tee -a /etc/hosts  then browse it
# a vhost is a different SITE keyed on Host header — not a subdirectory
```
