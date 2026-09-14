## Goal
Find every open port, fingerprint each service, then open the matching note. Enumeration is the whole game: most OSCP boxes fall to something you find, not an 0day.

**Checklist**
- [ ] full TCP scan done? (`-p-`)
- [ ] version + default scripts on the open ports? (`-sCV`)
- [ ] top UDP ports scanned? (SNMP/DNS/TFTP hide there)
- [ ] every open port has its note opened + checked?
- [ ] box redirects to a domain? added it to /etc/hosts?

# Commands

### Step 1: fast full port scan, then version-scan the open ones
```bash
nmap -p- --min-rate 1000 -T4 $IP -oN ports.txt
ports=$(grep ^[0-9] ports.txt | cut -d/ -f1 | paste -sd,)
nmap -p$ports -sCV -oN services.txt $IP
```

### Step 2: UDP (top ports)
```bash
sudo nmap -sU --top-ports 20 $IP -oN udp.txt   # 161 snmp, 53 dns, 69 tftp, 123 ntp
```

### Step 3: add vhost to /etc/hosts
```bash
echo "$IP box.local" | sudo tee -a /etc/hosts   # if a port redirects to a domain
```

### Route by port
```
21 -> 21_ftp        22 -> 22_ssh         25 -> 25_smtp        53 -> 53_dns
80/443/8080 -> 80_443_http/   (CMS -> cms | Tomcat/Jenkins/phpMyAdmin/panels -> web_apps_panels)
88 -> 88_kerberos   110/143 -> 110_143_pop3_imap              111/2049 -> 111_2049_nfs
139/445 -> 139_445_smb/       161 -> 161_snmp                 389 -> 389_ldap
1433 -> 1433_mssql  3306 -> 3306_mysql   3389 -> 3389_rdp     5432 -> 5432_postgresql
5900 -> 5900_vnc    5985 -> 5985_winrm   6379 -> 6379_redis   873 -> 873_rsync    27017 -> 27017_mongodb
less common (69 tftp | 79 finger | 512-514 r-svc | 1521 oracle | 9200 elastic | 2375 docker | 623 ipmi) -> misc_ports
unknown high port / product on an odd port / nmap says unknown|tcpwrapped -> unknown_service_triage
```

> In an AD engagement, the AD-relevant services (53 / 88 / 389 / 139-445 / 1433 / 5985 / 3389) are indexed in [[00_AD_SERVICES]]; drive the domain from [[00_AD_METHODOLOGY]].

### Helpline
```bash
# nmap: -p- all ports | -sC default scripts | -sV version | -sU UDP | -T4 faster | -Pn skip ping
# faster discovery: rustscan -a $IP -- -sCV
# re-scan every discovered vhost/subdomain separately, they often host different apps
```
