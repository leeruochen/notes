# AD Credential Validation & Spraying

Credential reuse is the engine of AD compromise. This note is how to validate credentials domain-wide **without locking accounts**, and how to read the three different results a single test can give.

## Goal
Turn a user list and/or a candidate secret into a validated set of `identity → host → access-level` facts, safely, and feed them into the loop ([[00_AD_METHODOLOGY]]).

## Preconditions (all of them, before you spray)
- **Authorised scope.** Spraying a domain is noisy and can lock users; only in labs or engagements you own.
- **Known domain.** FQDN + DC ([[00_AD_METHODOLOGY]] Step 1).
- **User list with known provenance.** Where each name came from (LDAP, RID cycling, Kerberos enum, emails). Bad lists waste your lockout budget.
- **Password-policy / lockout evidence.** You must know the lockout threshold *first* (Step 1 below).
- **A timebox** and a recorded stop condition.

---

## The three techniques (don't confuse them)
```text
PASSWORD SPRAYING   many users  × ONE password    -> stays under per-account lockout (the AD default)
CREDENTIAL REUSE /  one/few known secret × many    -> "does this found password work elsewhere?"
  STUFFING            users/services                   (highest-yield, lowest-risk)
BRUTE FORCE         one user × many passwords       -> trips lockout fast; almost never right in AD
```
Reuse first (you already earned those secrets), spray second (one password across users), brute force essentially never against domain accounts.

---

## Step 1: Check lockout policy before any spray
This is the gate for everything below: spray without it and you lock accounts. Read the policy first, every time.
```bash
# authenticated (any valid domain cred):
nxc smb  <DC_IP> -u <user> -p '<pass>' --pass-pol
nxc ldap <DC_IP> -u <user> -p '<pass>' --pass-pol
# sometimes available anonymously:
nxc smb  <DC_IP> --pass-pol
enum4linux-ng -P <DC_IP>
```
Read: **lockout threshold** (attempts before lock), **observation window** (resets the counter), **lockout duration**.
- Threshold `0` / "never": you can spray more freely (still be conservative).
- Threshold `N`: keep attempts **per account** below `N` within the window; leave a safety margin (e.g. N-2).
- Can't determine it: assume a strict policy, one attempt per account, then wait a full window.

> [!warning] Never spray before you know the policy
> An unknown lockout threshold plus a spray = locked accounts, a burned engagement, and possibly a failed exam objective. Policy first, always.

---

## Step 2: Reuse known secrets (do this first)
For every secret you already have (from configs, LSASS, GPP, cracking), test it across plausible identities and services. This is the highest-yield, lowest-risk move: one attempt per pair, no lockout pressure.

| Prerequisite | Value |
| --- | --- |
| Have | - a known secret (password or NTLM hash), source [[03_credential_access]] or a config/GPP/crack<br>- one or more plausible target identities/services |
| Need | know local vs domain context (`--local-auth` = a local SAM account, not a domain account) |
| Result | a validated `identity → host → access` fact, tracked in the matrix |

```bash
# one password across the user list, one attempt per account within the approved budget:
nxc smb <dc_ip> -d <domain_fqdn> -u users.txt -p '<known_password>' --continue-on-success
# a specific pair on the services it might reuse to:
nxc smb   <target> -d <domain_fqdn> -u <user> -p '<password>'
nxc winrm <target> -d <domain_fqdn> -u <user> -p '<password>'
nxc mssql <target> -d <domain_fqdn> -u <user> -p '<password>'
# pass-the-hash reuse (NTLM), same idea:
nxc smb <target> -u <local_user> -H <ntlm_hash> --local-auth
nxc smb <target> -d <domain_fqdn> -u <domain_user> -H <ntlm_hash>
```
**Verify:** a `(+)` (auth valid) or `Pwn3d!` (code exec) on a service, read per the table in Step 4.
**Fail:** `(-)` / `LOGON_FAILURE` = wrong pair (tool works, fix the secret); `(-)` everywhere = the secret doesn't reuse here, move on.

---

## Paired users/passwords or users/hashes: one-to-one validation, `--no-bruteforce`, Cartesian warning

### TLDR

Two aligned files can represent exact `user -> password` or `user -> hash` pairs. NetExec's `--no-bruteforce` keeps line `N` paired with line `N`. Without it, two lists can become every user by every secret, causing an accidental Cartesian-product spray.

### When you see this

You have `users.txt` plus `passwords.txt` or `hashes.txt`, where each secret belongs to exactly one user.

### Check first

Confirm the lockout policy and attempt budget, state domain or local context, use one target service or a tightly justified target set, and require equal nonblank line counts.

```bash
wc -l users.txt passwords.txt
rg -n '^\s*$' users.txt passwords.txt

wc -l users.txt hashes.txt
rg -n '^\s*$' users.txt hashes.txt
```

The counts must match and both `rg` checks must return no hits. Remove unintended blank lines before authentication.

### If confirmed

```bash
# Domain user/password pairs
nxc smb <target> -d <domain_fqdn> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# Domain user/hash pairs
nxc smb <target> -d <domain_fqdn> -u users.txt -H hashes.txt --no-bruteforce --continue-on-success
```

For local pairs, replace `-d <domain_fqdn>` with `--local-auth` and target only the host whose SAM context applies. `--continue-on-success` is intentional here so later pairs are tested. Omit it when the first success should stop the run. Stop immediately on lockout evidence.

### Expected result

Each user is tried only with the secret on the same line. A success proves authentication to SMB only. A count mismatch is a hard stop. `STATUS_ACCOUNT_LOCKED_OUT` is a hard stop for the entire run.

### What did I gain?

Record `credential or NTLM hash` plus `service authentication`. Record service authorization, local administrator, or host execution only after separate proof. See [[00_AD_METHODOLOGY#Capability ledger: what did I gain?]].

### Next

For each success, validate one plausible downstream service, refresh the session or ticket if necessary, update the matrix, then follow [[#Step 5: After every new valid credential, re-enumerate]].

---

## Step 3: Password spray (one password, many users)
Only after Step 1. Use a small, evidence-based candidate set, **one password at a time across all users**, with delays.

| Prerequisite | Value                                                                                                      |
| ------------ | ---------------------------------------------------------------------------------------------------------- |
| Have         | - a user list with known provenance<br>- lockout policy read (Step 1)<br>- a small, justified password set |
| Need         | one password per account per observation window; a recorded stop condition                                 |
| Result       | any `(+)` = a valid credential to validate everywhere and feed the loop                                    |

```bash
# NetExec: one password across the list; add spacing between rounds
nxc smb <dc_ip> -d <domain_fqdn> -u users.txt -p '<candidate_password>' --continue-on-success
# Kerberos pre-auth spray (kerbrute): often quieter, no SMB session
kerbrute passwordspray -d <domain_fqdn> --dc <dc_ip> users.txt '<candidate_password>'
```
Scheduling discipline:
- **One password per account per observation window.** Wait out the window before the next password.
- Space rounds; don't hammer. Stop the instant you see lockouts.
- Domain vs local: no flag = **domain** auth; `--local-auth` = **local** SAM account on that host. State which you mean every time.

**Verify:** a `(+)` line for a user = a valid credential. Validate it across services (Step 2 / Step 4) before assuming access.
**Fail (stop immediately):** any `account locked` / `STATUS_ACCOUNT_LOCKED_OUT`, an unexpected policy (shorter window than assumed), or you have used your per-window budget. Stop and wait.

---

## Step 4: Read the result correctly (the marker is protocol-specific)
This is result interpretation, not a new attack: three separate facts hide behind two markers.
```text
(-)  / LOGON_FAILURE          -> wrong credential (tool works; fix the pair)
(+)                            -> AUTHENTICATION valid: the secret is real (no command execution implied)
(+) + Pwn3d!                   -> CODE EXECUTION available on THAT protocol (meaning differs by protocol, below)
account locked / STATUS_LOCKED -> STOP: lockout hit
```
Per NetExec, `(+)` = authenticated and `Pwn3d!` = code execution, but "code execution" does **not** mean the same thing on every protocol:

| Protocol | `Pwn3d!` means | Implies local admin? |
| --- | --- | --- |
| **SMB** | code execution via SMB (admin shares / service control) | **most likely yes**, SMB exec needs admin |
| **WinRM** | command execution over WinRM | **no**: Remote Management Users *or* admin; a non-admin RMU member also gets `Pwn3d!` |
| **MSSQL** | command execution as the SQL service context (`xp_cmdshell` / sysadmin) | no, that is the service account, not OS admin |
| **RDP** | interactive / RDP access | no |

Source: NetExec docs, SMB `Pwn3d!` = "most likely (local) admin"; WinRM `Pwn3d!` = "code execution at least" (<https://www.netexec.wiki/>).

Three separate facts, never collapsed:
1. **Authentication:** is the credential valid at all? (`(+)`)
2. **Service authorization / command execution:** can *this* user use *this* service here (WinRM shell? RDP? share access? MSSQL exec)?
3. **Local administrator:** a *confirmed* admin context on the OS.

> [!warning] Confirm admin in-session before any local-admin-only action
> Fact 2 is not fact 3. A WinRM `Pwn3d!` non-admin can run commands but **cannot** read SAM/LSA/LSASS. Before routing to local-hive / LSASS / DCSync collection ([[03_credential_access]]), obtain the session and verify: `whoami /groups` shows **Administrators** / **High Mandatory Level** (or you hold `SeDebug` / `SeBackup`). Do not send a WinRM-authorized non-admin straight to SAM/LSA/LSASS.

A valid credential that isn't admin anywhere is still gold: it unlocks LDAP/BloodHound as an authenticated principal.

---

## Step 4b: Password must change, valid-but-must-change, or expired
Some valid credentials come back flagged *must-change* or *expired*. The secret is **correct**, it just can't be used until it is changed. Recognise the status:
```text
SMB / NetExec : STATUS_PASSWORD_MUST_CHANGE (0xC0000224)  /  STATUS_PASSWORD_EXPIRED
Kerberos      : KDC_ERR_KEY_EXPIRED
```
Record this as **VALID (must-change)** in the matrix, never as `(-)` / failed.

| Prerequisite | Value |
| --- | --- |
| Have | - a valid credential flagged must-change / expired<br>- the account's CURRENT (old) password |
| Need | a reachable change channel (smb-samr / rpc-samr / kpasswd / ldap) |
| Result | the same identity, usable, with a new password |

**Change it (this is a CHANGE, it needs the OLD password, not a rights-based reset):**
```bash
# impacket changepasswd: change YOUR OWN password with the old one (works in must-change state).
# default protocol is smb-samr; -p allows: smb-samr | rpc-samr | kpasswd | ldap
impacket-changepasswd '<domain_fqdn>/<user>:<old_password>'@<dc_fqdn> -newpass '<new_password>'
# Kerberos path (KDC_ERR_KEY_EXPIRED): force the kpasswd protocol
impacket-changepasswd '<domain_fqdn>/<user>:<old_password>'@<dc_fqdn> -newpass '<new_password>' -p kpasswd
```
This is a **self password change** (needs the account's **current** password). It grants **no** privilege, it only unlocks your own valid credential. It is **not** the same as `changepasswd -reset -altuser <admin> ...`, which *resets another user* and requires directory rights over that user: that path is branch B in [[02_bloodhound_and_acl_paths]], not this one.

**Verify:** `changepasswd` prints a success message, then one controlled retest with the new secret succeeds. Re-enumerate as the **same identity** with the new password ([[00_AD_METHODOLOGY]] Step 6).
**Fail / stop conditions:**
- New password rejected (complexity / history / min-age policy): choose a compliant one, don't loop the same reject.
- No supported change channel reachable (RPC-SAMR / kpasswd filtered): record it and move on.
- Change attempts still count toward policy: if lockout risk is rising, stop.

Source: impacket `changepasswd.py` (protocols `smb-samr` / `rpc-samr` / `kpasswd` / `ldap`; changes an expired/must-change password with the old secret, or `-reset -altuser` for a rights-based reset), <https://github.com/fortra/impacket>.

---

## Step 5: After every new valid credential, re-enumerate
A new identity changes what you can see and do. Re-run:
```bash
nxc smb <targets> -d <domain_fqdn> -u <new_user> -p '<password>' --shares
nxc ldap <dc_ip> -d <domain_fqdn> -u <new_user> -p '<password>' --users --groups
nxc winrm <targets> -d <domain_fqdn> -u <new_user> -p '<password>'
```
Then re-collect BloodHound as the new principal ([[02_bloodhound_and_acl_paths]]) and update the matrix in [[00_AD_METHODOLOGY]].

---

## Recordkeeping (avoids repeat attempts and lockouts)
Log every attempt so you never re-spray a user or re-test a dead pair:

| Credential                | Domain/Local | `<dc_fqdn>` (445)   | `<member_fqdn>` (WinRM) | `<sql_fqdn>` (MSSQL) | Notes                                                                           |
| ------------------------- | ------------ | ------------------- | ----------------------- | -------------------- | ------------------------------------------------------------------------------- |
| `<domain_user>:<secret>`  | domain       | (+) valid-no-access | (-)                     | (+) valid            | record date and attempt budget                                                  |
| `<service_user>:<secret>` | domain       |                     |                         | Pwn3d!               | MSSQL command execution as the service account; confirm OS privilege in-session |

Cross-linked from [[88_kerberos]], SMB [[authenticated_enum]], [[3389_rdp]], [[5985_winrm]].

> [!important] Helpline
> Reuse before spray, spray before (almost-never) brute. **Policy first: one password per account per window.** Auth-valid ≠ service-authorised ≠ local-admin: record all three. Stop on the first lockout. Every new cred → re-enumerate.
