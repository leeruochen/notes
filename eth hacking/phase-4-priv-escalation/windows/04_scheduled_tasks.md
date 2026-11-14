A scheduled task is cron for Windows: if a higher-privileged identity repeatedly runs an action you can influence, that is your escalation.

## Goal
Find a task run by a more-privileged user, whose action you control and whose trigger will fire again.

## The gate (all five)
```text
1. run-as identity is more privileged than you (SYSTEM / admin)
2. you know the action + arguments (what runs)
3. the trigger can fire again (schedule / logon / startup / event) or you can invoke it
4. you control the action file (or a script/config/DLL it consumes, or another proven link)
5. the result is verifiable
```

## Tips
As a low-privileged user you often cannot see other users' tasks - so rely on **clue files**: a stray `.exe` / `.ps1` / `.bat`, or a log showing a job runs every few minutes. But a stray writable script is **not** a task escalation until you prove a privileged task actually runs it. Control must be **shown**, not inferred from a directory name.

Checklist:
- [ ] `Run As User` more privileged than me?
- [ ] `Task To Run` (action + args) known?
- [ ] `Next Run Time` in the future / repeats (not one-time already past)?
- [ ] the action file (or a consumed file) writable by me? (`icacls`)
- [ ] backed up the original before replacing?

# Commands

### Step 1: enumerate (cmd.exe)
```batch
schtasks /query /fo LIST /v
:: read: TaskName · Author · Task To Run · Run As User · Next Run Time · Schedule Type
```
```powershell
Get-ScheduledTask | ? { $_.TaskPath -notlike "\Microsoft*" } | ft TaskName,TaskPath,State  # hide built-ins
```

### Step 2: confirm control of the action
```batch
icacls "C:\path\to\TaskAction.exe"   :: F/M/W for me or Users = replaceable
```

## Exploit

### Case 1: Writable action binary/script, privileged run-as
**Check:** `Run As User` = admin / SYSTEM, `Next Run Time` repeats, and `icacls` on the `Task To Run` file shows you `(F)` / `(M)` / `(W)`.
**Abuse:** back up, replace, wait for the trigger.
```batch
copy "C:\path\TaskAction.exe" "%TEMP%\TaskAction.bak"
copy /Y "C:\Users\Public\payload.exe" "C:\path\TaskAction.exe"
```
(payload / replacement → [[01_orientation#File transfer - Kali to Windows|file transfer]])
**Verify (after the schedule elapses):**
```batch
net localgroup Administrators     :: your new user present?
```
Restore the original file afterwards.
**Dead end / pivot:** a one-time task that already ran, or `Run As User` = you → no gain; pivot.

### Case 2: Writable script/config/DLL consumed by the task
**Check:** the action runs a script that sources another file, or loads a DLL, that *you* can write.
**Abuse:** drop your payload into that consumed file (same rules as the DLL case in [[03_services]]). Prove the task actually consumes it before trusting it.

## vs services
No `binPath` or config to reconfigure - the lever is the **action file** (or a file it consumes). Fewer links than a service, but the same backup / trigger / verify discipline.

## Correcting common misconceptions
- A scheduled task ≈ cron, but Windows triggers also include **logon, startup, events, and one-time** schedules - not only periodic ones.
- A task running as **your** user gives no privilege gain.
- An **expired / one-time** task with no repeatable trigger is a pivot condition, not a vector.
- Control of the action must be **shown** (`icacls` write), never inferred from the folder it sits in.
- Low-privileged visibility may be incomplete - missing output is not proof that no candidate task exists (see [[99_windows_stuck_reset]]).

## Helpline:
1. Fields that matter (from `schtasks /query /fo LIST /v`)
```
Run As User   -> more privileged than me?
Task To Run   -> the action file + args (the thing to hijack)
Next Run Time -> future & repeating = usable; past & one-time = dead
```

2. Trigger you can invoke
```
Get-ScheduledTask -TaskName <name> | Start-ScheduledTask   # only if you are allowed to run it
otherwise wait for the schedule / logon / startup event
```
