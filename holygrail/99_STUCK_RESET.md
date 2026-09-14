# Stuck Reset

## Goal
Break a rabbit hole and return to one cheap, evidence-based test.

Open this **before**: trying a fourth exploit · repeating the same payload with minor changes · opening a walkthrough · running another scanner without a clear purpose.

## Open this when
- You have spent ~15 minutes on one idea with no new evidence
- You cannot explain why the current idea should work
- You are relying on guesses rather than confirmed prerequisites
- You feel like you have "tried everything"

---

# 60-Second Reset
Write down:
1. What is confirmed?
2. What am I assuming?
3. What prerequisite does my current idea require?
4. What is the cheapest test?
5. What result will make me stop?

If you cannot answer these clearly, continue with the full reset.

---

# Part 1 — Rebuild the Facts
Write the answers down. If you cannot fill a line, that line may be your next move.

- [ ] All TCP ports and services confirmed?
- [ ] UDP checked where appropriate?
- [ ] All discovered hostnames and virtual hosts recorded?
- [ ] Discovered hostnames added to `/etc/hosts`?
- [ ] All usernames, passwords, hashes and tokens stored in one place?
- [ ] Which versions are exact / probable / guesses?
- [ ] Any port left as `unknown` / `tcpwrapped` / unidentified? → [[unknown_service_triage]]
- [ ] What functionality has not been tested?
- [ ] What can currently be read / written / uploaded?
- [ ] What access level and role does the current account have?

Capture it as:
```markdown
## Confirmed
- Fact

## Assumed
- Assumption

## Current hypothesis
- Statement

## Cheapest test
- Test

## Stop condition
- Result that invalidates the path
```

---

# Part 2 — Missed-Surface Sweep

## Network and hostnames → [[00_WEB_METHODOLOGY]] · [[vhost_fuzzing]]
- [ ] Did a redirect reveal a hostname?
- [ ] Did page source reveal a hostname?
- [ ] Did the TLS certificate reveal another hostname?
- [ ] Is there another HTTP or HTTPS port?
- [ ] Is the application different when accessed by hostname?
- [ ] Was virtual-host fuzzing performed?

## Web functionality → [[00_WEB_METHODOLOGY]]
- [ ] Was every visible feature tested manually?
- [ ] What does each feature do server-side (read a DB / read a file / write a file / fetch a URL / execute a utility / render a template)?
- [ ] Is there authenticated functionality that has not been explored?
- [ ] Are hidden parameters or API endpoints present?
- [ ] Were source, comments, JavaScript and requests inspected?
- [ ] Were backup files, robots and sitemap entries checked?

## Credentials and identity → [[default_creds_logins]]
- [ ] Are all discovered usernames recorded?
- [ ] Are application users being confused with system users?
- [ ] Were known credentials tested against plausible services?
- [ ] Were product defaults tested?
- [ ] Was password reuse tested safely?
- [ ] Is the authentication method actually enabled?
- [ ] Could the service lock or delay accounts?
- [ ] Was a small targeted wordlist generated?

```bash
cewl -d 4 -m 5 http://$IP/path -w words.txt
```

Track it with a reuse matrix — test only plausible services where the attempt is safe; avoid broad, unjustified spraying:

| Credential | Web | SSH | FTP | SMB | Database |
|---|---|---|---|---|---|
| user:password | Untested | Failed | Success | N/A | Untested |

## After foothold → [[internal_port_enum]] · [[08_interesting_files]] · [[00_FOOTHOLD]]
- [ ] Was the application configuration inspected?
- [ ] Were web roots searched for credentials?
- [ ] Were backup files and archives checked?
- [ ] Were local-only listening services checked? → [[internal_port_enum]]
- [ ] Were internal ports mapped?
- [ ] Is there a second NIC / another subnet to pivot into? → [[pivoting_and_tunnelling]]
- [ ] Were discovered credentials tested locally?
- [ ] Was the shell stabilised when interactive commands behaved incorrectly?

```bash
ss -tulpen
find /var/www /opt -type f \( -iname '*config*' -o -iname '.env' -o -iname '*.bak' -o -iname '*.zip' -o -iname '*.sql' \) 2>/dev/null
grep -RniE 'password|passwd|secret|token|database|db_user|db_pass' /var/www /opt 2>/dev/null
```

---

# Part 3 — Challenge the Current Hypothesis

## Prerequisites
- What product/component, version range, endpoint, feature, auth level, role, writable path, local service or configuration does the hypothesis require?
- Do you **possess** those prerequisites, or are you assuming them?

## Evidence
- What evidence supports the current path, and what would disprove it?
- What result counts as success, and what result makes you stop?
- Has the last attempt produced any new evidence?

## Experiment quality
- Did you change only one variable?
- Are you comparing response status, size, timing and content?
- Are you testing the payload, or fighting the delivery method?
- Does the raw request contain what you intended to send?
- Did copied characters become smart quotes or Unicode dashes? Is URL-encoding / shell-escaping / the `Host` header / the target path correct?

> [!tip] When copying payloads
> Retype quotes and dashes yourself · inspect the raw request · confirm encoding · confirm the payload reaches the server unchanged.

## Time
- How long since the last piece of new evidence?
- Are you debugging an exploit that does not match the environment?
- Are you continuing because the path is strong, or because time has already been spent? **Past effort is not evidence.**

---

# Part 4 — Pick One Reset Action
Choose one action only. Commit to it before changing direction again.

1. **Re-enumerate a rushed service** — return to the service note and complete the missing checklist.
2. **Map every web feature manually** → [[00_WEB_METHODOLOGY]] — for each feature record `Input → Server-side action → Observable output → Possible primitive → Cheapest test`.
3. **Re-test credentials systematically** — build a small reuse matrix; test only plausible combinations.
4. **Inspect source, requests and responses** — HTML source, JavaScript, API calls, headers, cookies, redirects, hidden fields, error behaviour, response differences.
5. **Search the filesystem after foothold** → [[08_interesting_files]] — configuration, `.env`, DB credentials, API keys, backups, archives, scripts, reuse, local-service credentials.
6. **Return to the attack-surface table** — re-read every discovery and fill the row; look for discoveries recorded but never acted on.

| Surface | Discovery | Meaning | Hypothesis | Cheapest test | Result | Next |
|---|---|---|---|---|---|---|
| | | | | | | |

---

# Lessons From Practice Boxes

> [!tip] Hostname handling
> **Transferable rule:** a hostname discovered through redirects, certificates or source code must be tested directly.

> [!tip] Server-side feature behaviour
> **Transferable rule:** for every interactive feature, identify the server-side action before selecting a vulnerability class.
> **Seen on:** a monitoring feature performed server-side processing that was not obvious from its interface.

> [!tip] Local services and application configuration
> **Transferable rule:** after gaining access, inspect application configuration and local listening services before searching for complicated privilege-escalation exploits.
> **Seen on:** a locally bound database and config files provided the next set of credentials.

> [!tip] Application users vs system users
> **Transferable rule:** keep application identities and system identities separate until credential reuse or configuration proves a relationship.

> [!tip] Payload transport
> **Transferable rule:** validate the raw payload and transport before concluding the vulnerability hypothesis is wrong.
> **Seen on:** a valid payload failed because copied characters changed during delivery.

> [!tip] Interactive shells
> **Transferable rule:** if `sudo`, terminal programs or interactive commands behave strangely, stabilise the shell before rejecting the technique.

---

> [!important] Helpline
> If you cannot write the prerequisite chain, you do not have a hypothesis — you have a guess.
> **Cheapest test first. One variable at a time. New evidence or move on.**
