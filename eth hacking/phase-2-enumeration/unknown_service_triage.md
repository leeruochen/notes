# Unknown-Service Triage

An open port with no obvious service — a high random port, a product on a non-standard port, or a protocol Nmap labelled `unknown`/`tcpwrapped`. This note gets you from "port is open" to a defensible product/version hypothesis without assuming the port number tells the truth.

## Goal
Separate **port number** from **protocol identity**, fingerprint what actually answers, and either route it to a real service note or reach a version hypothesis worth researching.

## Open this when
- The port router in [[00_methodology]] has no entry for the port
- A known-port service behaves nothing like it should (e.g. 8080 that isn't HTTP)
- [[misc_ports]] doesn't cover it, or Nmap shows `unknown` / `tcpwrapped` / a bare banner
- After foothold: a local listener ([[internal_port_enum]]) whose product you can't name

> [!warning] The port number is a hint, not an identity
> Anything can listen on any port. 22 can be a web app; 8443 can be a custom binary protocol. Fingerprint the *behaviour*, don't trust the label.

---

# Step 1 — Re-run detection with stronger probes

Your first `-sCV` may have given up. Push harder and read the raw evidence.
```bash
nmap -sV --version-intensity 9 -p <port> $IP -oN svc.txt      # all probes
nmap -sV --version-all -p <port> $IP                           # equivalent aggressive fingerprint
sudo nmap -sU -sV -p <port> $IP                                # if TCP is silent, try UDP
```
Record what comes back: a banner string, an app-proto guess, a `tcpwrapped` (a wrapper/filter, not a service), or nothing.

---

# Step 2 — Poke it with normal clients (cheapest, safest)

Try the common families in order. Watch for framing, prompts, and error strings.
```bash
nc -vn $IP <port>                       # raw connect: does it send a banner? wait for input?
printf 'HELP\r\n' | nc -vn $IP <port>   # does a generic verb draw a text response?
curl -sv http://$IP:<port>/             # HTTP? headers/status/redirect = yes
curl -skv https://$IP:<port>/           # HTTPS on an odd port
openssl s_client -connect $IP:<port>    # is it TLS? read the cert (SAN/CN/issuer = huge clues)
```
Interpretation:
- **HTTP status line / headers / redirect** → it's a web service, even on a weird port → [[00_WEB_METHODOLOGY]].
- **TLS handshake succeeds** → wrap the real protocol in TLS: read the cert for hostnames/product, then re-probe with `openssl s_client` as your "netcat".
- **A text banner or prompt** → a line-based text protocol; try its likely verbs (`HELP`, `VERSION`, `AUTH`).
- **Immediate close / binary garbage** → likely a binary protocol; stop hand-fuzzing (Step 4).

---

# Step 3 — Record a fingerprint

Capture enough to research and to recognise it again. Note your **confidence**.
```text
- port + transport (tcp/udp)
- banner / first bytes (hex if binary)
- response framing (line-based? length-prefixed? silent-until-input?)
- TLS cert data (CN/SAN/issuer/validity) if TLS
- error strings on bad input
- commands/verbs it accepted
- auth behaviour (open? prompts? rejects?)
- confidence: confirmed / probable / guess
```
A distinctive banner or error string is usually enough to name the product → then [[public_exploit_research]].

---

# Step 4 — Capture a packet trace *only if* clients aren't enough

If banners and normal clients don't identify it, take a *small* trace — don't mutate the service blindly.
```bash
sudo tcpdump -ni <iface> host $IP and port <port> -w svc.pcap   # then interact in another shell
# read it back: strings/protocol structure often reveal the product
tshark -r svc.pcap -x | head -50
```
This is a read-only observation step. Do **not** start writing crafted binary payloads at a service you can't identify — that's how you crash it.

---

# Step 5 — After foothold: map the local listener to its product

If you're on the host, identity is easy — the process *is* the answer (full workflow in [[internal_port_enum]]):
```bash
ss -lntup | grep <port>                 # PID
ls -l /proc/<pid>/exe ; cat /proc/<pid>/cmdline | tr '\0' ' '   # binary + args
```
Then `<binary> --version`, its package, and its config files give you an exact product/version — far more reliable than remote fingerprinting.

---

# Routing & stop conditions

| Observation | Route |
|---|---|
| speaks HTTP/HTTPS (any port) | [[00_WEB_METHODOLOGY]] |
| named product + version identified | [[public_exploit_research]] |
| it's a known service on an odd port (ftp/ssh/smb/db/…) | its service note in [[00_methodology]] |
| text protocol, credentials plausible | try defaults/reuse before anything exotic |
| binary/undocumented, no product name | **stop probing** — research the fingerprint, don't fuzz blind |

**Stop conditions:**
- The protocol is binary/fragile and you can't name it → research the fingerprint instead of mutating it.
- Any probe risks changing service state (writes, resets) and you don't understand the object → don't send it.
- You've spent your timebox with only guesses → record the fingerprint, move to another surface, return with new evidence ([[99_STUCK_RESET]]).

> [!important] Helpline
> Protocol identity ≠ port number. `nc` / `curl -v` / `openssl s_client` first; `tcpdump` only when clients fail. HTTP-anywhere → web methodology. Named product → [[public_exploit_research]]. **Don't fuzz a binary protocol you can't identify.**
