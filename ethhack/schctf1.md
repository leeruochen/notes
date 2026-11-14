# SCHCTF1 — Learning Notes

Lab box: `3.144.177.121` (also reachable via `lab.local` vhosts). Flags are 32-char strings, no wrapper.

set up 3.144.177.121    lab.local login.lab.local dev.lab.local in /etc/hosts (for kali)

---

## Challenge Walkthroughs

### i love school
**Category:** Steganography
**Tools:** exiftool, base64, steghide

1. Ran `exiftool ilovesit.jpeg` on the provided image. The **Comment** field held `NXVwM3I1M2N1cjNwNDU1dzByZA==` — recognizable as Base64 by its charset and `==` padding.
2. Decoded it (`echo "..." | base64 -d`) to get the plaintext passphrase `5up3r53cur3p455w0rd`.
3. Ran `steghide extract -sf ilovesit.jpeg -p 5up3r53cur3p455w0rd` to pull the hidden payload out of the JPEG using that passphrase.
4. Flag came out along with a note explaining *why* this worked: steghide hides data in a JPEG's **DCT coefficients** (used during lossy compression), not in raw pixel values. That's why naive LSB (least-significant-bit) steg tools find nothing in a JPEG — pixel-level hidden data wouldn't survive JPEG's lossy compression, so JPEG steg tools operate at the compression-coefficient level instead.

### what is 2222
**Category:** Recon
**Tools:** ssh

1. Challenge name hinted at port 2222. Connected directly: `ssh -p 2222 fakeuser@3.144.177.121`.
2. The service banner/login response itself contained the flag `2xkCpdAy4l6tGZ3YUhv1xd6Gf7i9Kzn9` — no actual valid credentials needed, just connecting to the non-standard port revealed it.

### more than just 2222
**Category:** Recon
**Tools:** nmap, curl

1. Since the previous challenge was about port 2222, this one hinted there's more to find — ran a full port scan: `nmap -p- -sV -T4 3.144.177.121`.
   - `-p-` scans all 65,535 ports (not just the default top 1000)
   - `-sV` probes each open port to fingerprint the running service/version
   - `-T4` speeds up scan timing
2. Scan revealed an open port `31337` (a classic "leet" port number often used in CTFs/backdoors).
3. `curl http://3.144.177.121:31337` returned the flag directly from whatever lightweight service was listening there.

### whats 9 + 10
**Category:** Enumeration
**Tools:** curl (FTP mode)

1. The riddle-style title ("9+10=21") hinted at port 21 — FTP.
2. Logged in anonymously: `curl --user anonymous: ftp://3.144.177.121/`. The "anonymous" username is a legacy FTP convention many servers still accept for public/guest access.
3. Listed and grabbed the README: `curl --user anonymous: ftp://3.144.177.121/README.txt`.
4. The file contained a welcome message, an infrastructure note that the file server also exposes an SMB share named "public" (the hint for the next challenge), and the flag `uUyyo0i16y3lIbZfw1Anq63OJJRtTxKv`.

### sharing is caring
**Category:** Enumeration
**Tools:** smbclient

1. Title points at SMB (Server Message Block) file sharing, and the previous challenge's README explicitly named a "public" share.
2. Listed available shares: `smbclient -L //3.144.177.121 -N` (`-N` = no password, anonymous/null session). Confirmed the "public" share.
3. Connected to it: `smbclient //3.144.177.121/public -N`.
4. Inside the SMB session, ran `ls` to see files, then `more IT-NOTES.txt` to read one directly, which contained the flag.

### d and s
**Category:** Enumeration
**Tools:** dig

1. Title = "DNS."
2. Queried TXT records: `dig @3.144.177.121 lab.local TXT`. No TXT record directly on `lab.local`, but the **AUTHORITY SECTION** revealed an SOA record naming `ns1.lab.local` — meaning there's a real nameserver here worth interrogating further, and possibly more subdomains to discover.
3. Attempted a **zone transfer**: `dig @3.144.177.121 lab.local AXFR`. AXFR asks a DNS server to hand over its *entire* zone file at once — a serious misconfiguration if allowed for anyone, but here it's the intended vulnerability.
4. The transfer dumped every record: `dev.lab.local`, `login.lab.local`, `ns1.lab.local`, `www.lab.local`, and critically `flag.lab.local` — a TXT record directly containing the flag `fA5WJ5aD7GqFr7AVO5tX2dm7k9eHpjta`.

### more than just tcp
**Category:** Recon
**Tools:** snmpwalk

1. Title hints that not everything runs over TCP — UDP-based protocols exist too. SNMP (Simple Network Management Protocol) runs on UDP/161 and is commonly used for monitoring network devices, matching the challenge's own hint text about a "monitoring service."
2. Queried it with the classic default/guessable community string: `snmpwalk -v2c -c public 3.144.177.121`.
3. The `sysDescr` OID (`iso.3.6.1.2.1.1.6.0`) returned a string containing the flag: `"Cybersecurity Training Lab - 0uiKwKIDOMK3bcAxWG4eVEQu8z2GKONM"`.

### more than just one (and its pair, "and so it begins")
**Category:** Web / Enumeration
**Tools:** `/etc/hosts` editing, ffuf, curl

Both challenges share one walkthrough — the first five steps solve "and so it begins," and step 6 solves "more than just one."

1. `login.lab.local` doesn't publicly resolve, so it was mapped manually: `sudo nano /etc/hosts` → added `3.144.177.121 login.lab.local`.
2. Downloaded a directory wordlist: `wget .../SecLists/.../common.txt`.
3. Fuzzed the site for hidden paths: `ffuf -w common.txt -u http://login.lab.local/FUZZ -e .bak,.env,.old,.txt,.zip,.swp`. This turned up `robots.txt` and a `/backup/` directory.
4. Read `robots.txt` with curl to see its contents (a disallow entry pointing at/hinting toward the backup path).
5. Requested `/backup/` directly — it showed a `flag.txt` file existed. A plain `curl` to it got kicked out (likely a basic WAF/User-Agent check), but adding a normal browser User-Agent bypassed it: `curl -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" http://login.lab.local/backup/flag.txt`. This flag turned out to belong to the *other* challenge, "and so it begins," not this one.
6. For "more than just one" specifically: guessed that a **virtual host** might exist for a development environment, and tested the obvious name directly via the `Host` header: `curl -H "Host: dev.lab.local" http://3.144.177.121/`. This returned a distinct "Development Portal" page containing this challenge's actual flag.

### a lot of POSTS
**Category:** Web
**Tools:** curl, ffuf

**Challenge:** login portal at `login.lab.local/login`, hint "status codes play a big role here."

1. Manual testing with curl showed every wrong login attempt returned an identical `404 Not Found` / "Invalid credentials" — a deliberately misleading status code (normally you'd expect 401/403 for bad login). Tried SQL injection payloads, method switching (GET/PUT/HEAD/OPTIONS/TRACE), parameter pollution, JSON/multipart bodies, query-string params, header injection, and various other bypass attempts — all uniform 404s. This ruled out shortcuts and confirmed brute force was the intended path.
2. Small brute-force attempts (curl loops, tiny wordlists, Hydra) either found nothing or were far too slow (Hydra estimated ~292 hours) because they weren't parallelized well against this endpoint.
3. Switched to a two-stage **ffuf** approach — find the username first, then the password, rather than trying every combination of both at once:
   - **Stage 1 — find the username:** fixed a guessed password, fuzzed the username field against `common.txt`:
     ```
     ffuf -w common.txt -X POST -d "username=FUZZ&password=password123" \
       -u http://3.144.177.121/login -H "Host: login.lab.local" \
       -H "Content-Type: application/x-www-form-urlencoded" -fc 404
     ```
     Filtering out the standard 404 response surfaced `Admin` as a hit within ~30 seconds.
   - **Stage 2 — find the password:** fixed `username=Admin`, fuzzed the password field against `rockyou.txt`:
     ```
     ffuf -w rockyou.txt -X POST -d "username=Admin&password=FUZZ" \
       -u http://3.144.177.121/login -H "Host: login.lab.local" \
       -H "Content-Type: application/x-www-form-urlencoded" -fc 401,404
     ```
     ~15 minutes later, `harrypotter123` returned an HTTP `200` — the real success status code, confirming the hint.

**Lesson:** don't trust the status code shown on failure alone — the hint was pointing at the success code (200) that only appears once, buried among thousands of uniform "failure" responses. Splitting the attack into two single-field passes (find a valid username, then brute the password for that one user) collapses a huge combinatorial brute force into two much smaller, faster linear searches.

---

## Tools by Category

### Web
- **ffuf** — fast, multi-threaded HTTP fuzzer. Used for directory/file discovery, vhost discovery, and POST-body field brute forcing (username, then password), filtering by status code (`-fc`) or size. Also useful for: parameter fuzzing, subdomain enumeration, header fuzzing, API endpoint discovery.
- **curl** — HTTP client for manual request crafting and inspection. Used for checking raw headers/status codes (`-i`), spoofing the `Host` header for virtual host testing, setting a browser `User-Agent` (`-A`) to get past basic UA-based filtering, and POSTing form data (`-d`). Also useful for: API testing, scripting HTTP interactions, testing auth headers/cookies, FTP transfers.

### Enumeration
- **ffuf (directory/vhost mode)** — same tool as above, but aimed at paths (`/FUZZ`) and Host headers (`Host: FUZZ.lab.local`) to discover hidden directories and virtual hosts not linked from the main page.
- **smbclient** — SMB/CIFS client. `-L` lists available shares on a host; connecting to a share (`//host/share`) lets you `ls`/`get`/`more` files inside. Also useful for: pulling files from Windows file shares, testing anonymous/null-session access, checking share permissions.
- **curl (FTP mode)** — `curl --user anonymous: ftp://host/` logs into an FTP server using the legacy "anonymous" convention many servers still accept for public access. Also useful for: automating uploads/downloads, scripting FTP recon.
- **dig** — DNS lookup tool. A normal query reveals TXT/SOA/A records; an **AXFR** (zone transfer) asks a DNS server to hand over its *entire* zone file at once, revealing every record/subdomain it knows — a major misconfiguration if allowed for anyone. Also useful for: general DNS troubleshooting, reverse lookups, checking record propagation.
- **Editing `/etc/hosts`** — manually mapping a hostname to an IP when there's no real public DNS entry, so tools that key off the `Host` header work naturally without passing `-H` every time.

### Recon
- **nmap** — network/port scanner. `-p-` scans all 65,535 ports rather than just the common top-1000, `-sV` fingerprints the service/version on each open port, `-T4` speeds up scan timing. Used here to find a non-standard open port (`31337`) a default scan would miss. Also useful for: OS fingerprinting (`-O`), NSE script scanning (`-sC`), host discovery across a subnet.
- **snmpwalk** — queries SNMP (UDP/161), commonly used for device monitoring. `-v2c -c public` uses SNMP v2c with the default/guessable "public" community string to pull device info. Also useful for: enumerating full device inventories, network monitoring/troubleshooting, spotting default community strings as a misconfiguration.
- **ssh (non-standard port)** — connecting with `-p <port>` to an SSH service on an unusual port, something a full port scan or banner grab would reveal. Also useful for: remote administration, tunneling/port forwarding, key-based auth testing.

### Steganography
- **exiftool** — reads/writes file metadata (EXIF, comments, etc.). Used to pull a hidden Base64 string out of a JPEG's comment field. Also useful for: forensic analysis of images (GPS data, camera info, timestamps), stripping metadata for privacy, verifying file authenticity.
- **Base64 decoding** — a reversible text-encoding scheme, easily identified by its charset and `=` padding. Common first step whenever a string "looks like gibberish but only uses letters/numbers/+/=".
- **steghide** — extracts/embeds hidden data inside JPEG/BMP/WAV/AU files by manipulating **DCT coefficients** (for JPEGs) rather than raw pixel data, which is why pixel-based LSB steg tools find nothing in a JPEG — lossy compression would destroy pixel-level hidden data. Requires a passphrase to extract. Also useful for: hiding/retrieving arbitrary files inside cover images or audio for basic steganography exercises.

---

## Quick Reference Table

| Challenge | Category | Key tools | Flag mechanism |
|---|---|---|---|
| i love school | Steganography | exiftool, base64, steghide | EXIF comment → base64-decoded passphrase → steghide extraction |
| what is 2222 | Recon | ssh | Non-standard SSH port, banner/login revealed flag |
| more than just 2222 | Recon | nmap, curl | Full port scan found service on port 31337 |
| whats 9 + 10 | Enumeration | curl (FTP) | Anonymous FTP login, README.txt contained flag + SMB hint |
| sharing is caring | Enumeration | smbclient | Anonymous SMB share access, flag in IT-NOTES.txt |
| d and s | Enumeration | dig | DNS zone transfer (AXFR) leaked all records including a flag TXT record |
| more than just tcp | Recon | snmpwalk | Default "public" SNMP community string leaked flag in sysDescr |
| more than just one / and so it begins | Web / Enumeration | ffuf, curl | Directory fuzzing found `/backup/`; vhost fuzzing found `dev.lab.local` |
| a lot of POSTS | Web | curl, ffuf | Two-stage POST field fuzzing (username, then password) found valid login; success signaled by HTTP 200 among a sea of 404s |
