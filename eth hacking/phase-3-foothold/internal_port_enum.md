# Internal Port & Local-Listener Enumeration

You have a shell. Some services only listen on `127.0.0.1` (or on a second NIC), invisible to your external Nmap. Each is a **clue**, not a win — you still have to identify it and decide whether it's worth reaching.

## Goal
Enumerate every local/internal listener, map it to a process → owner → product → version, and decide the cheapest way to interact with it: locally, single-port forward, SOCKS, or a routed tunnel.

## Open this when
- Post-shell checks in [[00_FOOTHOLD]] showed loopback-only listeners or a second interface
- A stuck-reset ([[99_STUCK_RESET]]) sweep asks "were internal ports mapped?"
- On Windows privesc, a loopback service looks escalation-relevant → this is the platform-neutral companion to [[07_internal_services]]

> [!warning] A loopback listener is a clue, not an escalation
> `127.0.0.1:<port>` means *not exposed*, it does not mean *vulnerable* and it does not mean *privilege escalation*. You still need an exploitable/admin function, credential reuse, or unsafe local trust to turn it into anything.

---

# Step 1 — List every listener

```bash
# Linux
ss -lntup            # all TCP/UDP listeners + PID/process (preferred)
netstat -lntup       # fallback if ss is absent
```
```batch
:: Windows
netstat -ano | findstr LISTENING       :: note 127.0.0.1:<port> + PID
```
```powershell
Get-NetTCPConnection -State Listen | ? LocalAddress -in '127.0.0.1','::1' |
  Select LocalPort,OwningProcess | Sort LocalPort
```
Record each as: **address:port · PID · bound to loopback or a real NIC**. A listener on `0.0.0.0`/a routable NIC that your external scan *missed* usually means a host firewall — still worth a forward.

---

# Step 2 — Map listener → PID → owner → product → version

The port number is a hint; the **owning process** is the identity.
```bash
# Linux — PID from ss, then:
ls -l /proc/<pid>/exe            # the actual binary
cat /proc/<pid>/cmdline | tr '\0' ' '; echo   # args (creds leak here) — see interesting_files
ps -o user= -p <pid>             # owning account
<binary> --version 2>/dev/null   # version, if safe to run
```
```batch
:: Windows — PID from netstat, then:
tasklist /fi "PID eq <PID>"                                   :: PID -> process
```
```powershell
Get-CimInstance Win32_Service | ? ProcessId -eq <PID> | Select Name,StartName,PathName  # service + account
```
Now you can answer the only two questions that matter:
1. **Does it run with more privilege than me?** (owner = root/SYSTEM/another user)
2. **What product/version/auth is it?** (fingerprint locally in Step 3)

---

# Step 3 — Fingerprint locally first (cheapest test)

Before building any tunnel, interact with the service *from the shell you already have* — it's faster and needs no forwarding.
```bash
curl -sv http://127.0.0.1:<port>/          # HTTP-like? -> route to web methodology
nc -vn 127.0.0.1 <port>                     # grab a banner
# HTTP on an odd port is still HTTP -> [[00_WEB_METHODOLOGY]]
# unrecognised banner/protocol -> [[unknown_service_triage]]
```
If it's a DB, local admin panel, or RPC endpoint with default/known creds and it runs as a higher-privileged account, **attack it in place** — reuse any credential you already found. Verify with `whoami`/`id` in whatever the code path produces.

---

# Step 4 — Decision gate: how to reach it

Only tunnel when running the exploit locally is genuinely harder. Pick the *smallest* mechanism that fits.

| You need… | Use | Why |
|---|---|---|
| just to *interact* with one internal port from the shell | nothing — use local `curl`/`nc` | zero setup |
| one internal port reachable *from Kali* | **single reverse/local forward** | one port, one tool (e.g. run a GUI/complex exploit on Kali) |
| a Kali *tool* to reach many internal ports/hosts | **SOCKS proxy** (`-D` / chisel socks) | proxy-aware tools via `proxychains` |
| a *whole subnet* reachable transparently by any Kali tool | **routed tunnel** (ligolo-ng) | Nmap/exploit run natively against the subnet |

Full command patterns for each live in [[pivoting_and_tunnelling]]. This note stops at *which* to choose.

---

# Step 5 — Verify in layers (do not skip)

A pivot fails silently. Test one layer at a time — a working tunnel to a service that rejects you is still a dead end.
```text
1. TUNNEL   is the agent/session actually connected?           (proxy console / ssh established)
2. ROUTE    does traffic for the target subnet enter the tunnel?  (ip route / proxychains resolves)
3. TRANSPORT can you open the TCP port at all?                  (nc -vn <ip> <port> through the tunnel)
4. APP      does the service accept your request/creds?          (curl / client behaves normally)
```
If layer 3 works but layer 4 fails, the tunnel is fine — the *service* is rejecting you (wrong creds, wrong vhost, wrong protocol). If layer 3 fails, go back to route/tunnel; do not touch the payload.

---

# Common false positives
- A loopback listener that's just a local cache/agent (some updater, a metrics endpoint) with no privileged function — record it and move on.
- A "filtered" external port that is actually open behind a host firewall — a forward reveals the real service.
- Treating the forward working as the exploit working — always re-verify at the application layer.

> [!important] Helpline
> Listener → PID → owner → product → version, *then* decide to tunnel. Local test beats a tunnel. **A loopback service is a clue; privilege still has to come from what the service lets you do.** Broader multi-host/subnet pivoting → [[pivoting_and_tunnelling]].
