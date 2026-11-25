# ICT2212 CTF kit v10.7: what it covers, and how that was checked (21 Nov 2026)

One script, `ctf-auto.sh`. The VirtualBox and VMware kits differ only in `ctf.conf`.

## 1. Run against the real practice site on 21 Nov, before it closed

| Challenge | Command | Result |
|---|---|---|
| and so it begins, More Than Just One, what's 9+10, sharing is caring, More Than Just TCP, d and s, More than Just 2222, What is 2222 | `ctf --enum 3.144.177.121` | one run, eight flags in the box at the end |
| a lot of POSTS | `ctf --form-brute http://3.144.177.121/login --host-header login.lab.local` | found the username `Admin` by its different answer, then the password, and printed the flag |
| LFI Level 2 | `ctf --lfi "curl -s '.../translations.php?page={P}'" --file /var/www/secrets/review.txt` | flag. Levels 1 and 3 use the same command with their own page and file. |
| SQLi Catalog | `ctf --sqlmap-auto ".../catalog?item=1"` | tables listed, flag dumped |
| SQLi Cart, Account, Search, Orders | `ctf --find-input ".../account"` | found the inputs by itself: the cookie named `uid` (never handed out by the site), the JSON body of the cart button, the signed `id` and `h` link, and listed the two boxes (catalog, search). v10.7 prints a ready `--sqlmap-auto` line for all five when pointed at the site's home page. |
| Access Denied... Or Is It? (5001) | `ctf --overwrite 3.144.177.121:5001 --src chall.c` | read 32 and 0x1337c0de from the source, flag |
| whats a buffer overflow (5002) | `ctf --ret2win 3.144.177.121:5002` | read win() and the offset from the banner, flag |
| Echo Chamber (5004) | `ctf --fmt-remote 3.144.177.121:5004` | flag |
| Echo32 (5003) | `ctf --pwn-check echo32`, then `ctf --ret2shell 3.144.177.121:5003 --binary echo32 --shift=-0x10` | named the type; the run itself is a classmate's script, which the kit launches and reads |
| hidden in plain sight, four messages one secret, same name different AP | `ctf --wifi-hidden`, `--wifi-crack`, `--evil-twin` on the real captures | right SSID, BSSID and ID; passphrase, decryption and flag; twin, victim and the stolen login |
| i love school | `ctf --stegchain ilovesit.jpeg` | read the comment, decoded it, used it as the steghide passphrase, flag |

Faults seen in those runs and fixed in v10.7: the JSON body in the cart hint, the banner repeated in the format-string table, an empty block for port 139, and `--fresh` refused when it was the last word of a command.

## 2. On the practice range

Every "kit command" hint on the scoreboard is run exactly as written and its output is checked against the answer key.

| Set | Hints run | Pass | Not passing, by design |
|---|---|---|---|
| Practice CTF replica, 2024 paper mirror, practice set 3 | 53 | 50 | Echo32 (needs the classmate's script), one whois question (by hand), one doubly filtered file inclusion (by hand) |
| Mock exam 1 and 2 | 20 | 19 | the login behind a filter (by hand, on purpose) |

The replica box now answers with the real site's own pages (ten lab-box responses and twenty web pages are identical byte for byte), so a pass there is close to a pass on the real thing.

Flag formats: the box at the end of every run lists `ehCTF{}` in any letter case, `SIT{}`, `flag{}`, anything ending in `CTF{}`, bare 32-character lab flags, and flags hidden as base64, hex, ROT13 or written backwards. With the range's flags rewritten to an unknown wrapper, 22 of 24 were still listed (under "possible flags"); with bare values, 22 of 24 were visible in the output.

## 3. What the kit does not do

- It does not read the question. It cannot know which answer format a question wants.
- A plain-value answer (a username, a place, a version) has no shape to recognise. It is in the command's output, not in the flag box.
- It has no SQL injection code of its own: `--sqlmap-auto` runs sqlmap and reads what sqlmap prints.
- The Echo32 type (code run from the stack) needs `bof/scripts/ret2shell.py`, a classmate's script. The kit finds it, runs it and explains a failure. Without it, see the walkthrough, challenge 21, for the Lab 10 route.
- `--ret2win` cannot work through a menu before the overflow. `--overwrite` guesses the buffer size and the constant from the source: check both.
- Format-string writes (`--fmt-write`) are for 32-bit programs.
- No password guessing against a live server unless you ask for it (`--form-brute`, `--http-brute`, `--ssh-brute`), and then only with a short list.
- No Burp, no browser. For a page whose button does the work, `--find-input` and `--source` read the script instead.

## 4. What changed since v10

| Version | Change |
|---|---|
| v10.5 | `--sqlmap-auto` (one request to explain a refusal, two attempts, table dump, reasons for a miss), `--overwrite`, `--ret2shell` with a failure breakdown, "what to test next" list after `--enum` |
| v10.6 | Flag detection for any wrapper and encoded flags, `FLAGFMT=`. `--find-input`, `--pwn-check`, `--pcap`, `--crack`, `--ssh-brute`. `--folder` handles programs, sources, plain captures, hashes and zips. `--lfi` shows a named file whatever it holds. `--enum` does the quick services before the web checks. |
| v10.7 | Fixes from the real-site runs (see section 1). `--fmt-remote` now rebuilds a flag that sits on the stack as raw text even when no position prints it, and reads a few more positions when a flag is still open at the end of the sweep. `--find-input` prints a next step for boxes on linked pages. `--sqlmap-auto` has a third attempt that tells sqlmap what a normal page says (`--string`), for pages where true and false differ by only a few words. |
