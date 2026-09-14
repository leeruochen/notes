You are in the **Administrators** group but running at **medium integrity** — UAC handed your admin account a filtered token. This note is the decision, not a vague "elevate": go from medium-integrity admin to a High-integrity (elevated) process.

## Goal
Turn a *medium-integrity account that is already an administrator* into a High-integrity process — using a legitimate elevated logon where possible, a build-matched bypass only when necessary.

> [!important] UAC bypass creates no privilege — it changes token elevation
> A UAC bypass only elevates an account that is **already privileged**. It does **not** turn a standard user into an admin. If `whoami /groups` does not show Administrators, this note does not apply — go back to [[00_master_checklist]] / [[02_token_privileges]].

## Step 1 — Confirm you're actually in this situation
```powershell
whoami /groups        # is "BUILTIN\Administrators" present? AND
                      # "Mandatory Label\Medium Mandatory Level" (not High)?
whoami /priv          # a filtered (medium) admin token shows a SHORT privilege list
```
Decision:
- **Administrators + Medium integrity** → you're in the right place, continue.
- **Administrators + High integrity** already → you're elevated; you don't need a bypass — proceed to SYSTEM via [[02_token_privileges]] or just act as admin.
- **Not in Administrators** → **stop.** UAC is not your path; standard users are never routed here.

## Step 2 — Prefer a legitimate elevated logon path (verify, don't assume)
Some logon/execution paths *can* yield a High-integrity token — but **none is guaranteed**. Whether you get High integrity depends on the account type (local vs domain vs the built-in Administrator), the logon/session type, UAC policy, and remote-token filtering. Treat each as a candidate and **verify integrity in the resulting process** (Step 4). Try the cheapest available:

| Candidate route | May give High integrity when… | Key caveat |
|---|---|---|
| RDP / interactive logon, then "Run as administrator" | interactive consent/elevation is available | needs a desktop + the ability to click through consent |
| WinRM as an admin | the account is a local admin and WinRM grants an elevated token | a **non-admin** Remote Management Users member gets only a filtered/medium token — WinRM ≠ elevated |
| `runas /user:...` | you have the password **and** an elevation path applies | `runas` changes **user context**, it does **not** itself trigger UAC elevation — you may still land at medium integrity |
| PsExec / WMI to the host as an admin | the account is authorised for admin exec (SMB `Pwn3d!`-style) | conditional on admin rights; **local accounts** are throttled by remote UAC token filtering (LocalAccountTokenFilterPolicy) — the **built-in Administrator RID-500** is the usual exception |

> [!warning] Remote token filtering — local vs domain accounts
> By default, a **non-RID-500 local admin** authenticating **remotely** (PsExec/WMI/WinRM) receives a **filtered (medium) token**, so admin actions can still be denied. **Domain** accounts in the host's local Administrators group are **not** filtered this way, and the built-in **Administrator (RID 500)** is exempt. So "I'm a local admin" does not guarantee an elevated remote session — verify.

Over a plain reverse/bind shell you often *can't* get interactive elevation — that's when Step 3 applies.

## Step 3 — Build-matched UAC bypass (only if no legitimate path)
There is **no single bypass that always works** — the technique must match the target. Research it via [[public_exploit_research]], matching:
```text
- OS build number + patch level   (systeminfo -> 01_orientation)
- UAC policy / EnableLUA + ConsentPromptBehaviorAdmin (registry) — some setups auto-elevate, some always prompt
- executable availability          (the auto-elevating binary / LOLBAS the technique needs)
- session type                     (interactive vs non-interactive — many bypasses need a desktop)
- payload architecture             (match host arch — WOW64 trap, 01_orientation)
```
Pick the technique whose prerequisites you can confirm; don't hard-code one method.

## Step 4 — Verify (mandatory after ANY route) & stop conditions
Whether you took a Step-2 legitimate route or a Step-3 bypass, **do not assume elevation — observe it** in the resulting process:
```powershell
# in the new process:
whoami /groups      # "High Mandatory Level" now? -> elevated  (Medium = the route did NOT elevate)
```
- **Success:** a High-integrity process (then chase SYSTEM via token privileges if needed).
- **Fail:** bypass silently no-ops on a patched build, or needs a desktop you don't have → stop, try a legitimate logon path, or reconsider whether you even need SYSTEM.
- **Stop condition:** if you're already admin and the objective is met at high integrity, you may not need SYSTEM at all — don't burn time chasing it.

> [!important] Helpline
> Medium-integrity **admin** only. Bypass ≠ new privilege. A **verified** legitimate elevated logon beats a bypass — but no route (RDP/WinRM/runas/PsExec) is inherently elevated; `runas` changes user context, not UAC state, and remote local-admin logons are token-filtered (RID-500/domain accounts excepted). **Confirm High integrity in the resulting process every time.** Build-specific bypass → [[public_exploit_research]]. Standard users are **not** routed here → [[00_master_checklist]].
