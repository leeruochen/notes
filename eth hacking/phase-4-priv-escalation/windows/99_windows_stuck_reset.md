Stuck is a signal to change the hypothesis, not to try the same payload harder. Reset the evidence, re-read it deliberately, and pick the cheapest test that would change your mind.

## Goal
Stop thrashing. Convert "I'm stuck" into a new observation.

## Reset checklist (run top to bottom)
- [ ] **Re-confirm identity.** `whoami` / `whoami /priv` / `whoami /groups` again. Pivoted to a new user since the last sweep? Re-run the ENTIRE sweep as that identity ([[00_master_checklist]]) - new permissions expose new vectors.
- [ ] **Re-read winPEAS - don't re-run it.** Read the saved output line by line; highlight anything unfamiliar (a process, service, path, or username you don't recognise).
- [ ] **Cross-check with a second tool.** Seatbelt / PowerUp / SharpUp / AccessChk on the specific category you suspect - different tools surface different findings, and all need manual confirmation.
- [ ] **Revisit the cheap checks you skipped:** token privileges ([[02_token_privileges]]) · saved creds + PS history / transcripts + autologon + unattended files ([[05_stored_credentials]]) · **both** AlwaysInstallElevated values ([[06_writable_paths_and_registry]]) · service + task triggers ([[03_services]] / [[04_scheduled_tasks]]) · internal listeners ([[07_internal_services]]).
- [ ] **Investigate the unknowns.** Any third-party process / software you can't identify → get its version → search for an exploit.
- [ ] **Kernel last.** Only after the safer categories are genuinely exhausted ([[08_kernel_exploits]]).

## Write it down (one line each)
```text
Observation    -> what did I actually see (evidence, not assumption)?
Interpretation -> what does it mean, given what I know?
Hypothesis     -> the vector I now suspect
Cheapest test  -> the smallest action that would confirm or kill it
Result         -> what happened
Next action    -> route, or a NEW hypothesis
```

## Discipline
- Separate a **confirmed dead end** (a specific tested negative) from an **untested assumption** ("probably nothing there"). Only the first closes a path.
- Change the **hypothesis**, not merely the payload. Re-running a variant of a failed idea is thrashing.
- **Timebox** risky or low-evidence paths, then return to breadth.

Deeper troubleshooting / coverage: [HackTricks - Windows Local Privilege Escalation](https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html) (expanded technique reference) and [PayloadsAllTheThings / InternalAllTheThings - Windows Privilege Escalation](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/) (payloads / edge cases). Use them to widen coverage, not as the primary checklist.
