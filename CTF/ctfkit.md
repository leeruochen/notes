# CTF Kit

## Install

```bash
cd ctf-kit && chmod +x install.sh ctf-auto.sh && ./install.sh
```

This adds the `ctf` command and asks before installing anything.

- First line of `ctf --help` must say **v10.7**.
- Without installing: `./ctf-kit/ctf-auto.sh --enum 127.0.0.1`
- If scans miss ports or end in "No route to host" (VMware NAT does this), add `--threads 2`.
- Do **not** run `apt upgrade` / `full-upgrade` on your exam VM. Take a snapshot before you change anything.

## The Commands You Will Use Most

| Command                                                      | What it does                                                                                                                                                                           |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ctf --help`                                                 | Every command with an example (first line shows the version, last line the profile)                                                                                                    |
| `ctf --enum <ip>`                                            | Whole box: port scan, then web / virtual hosts / FTP / SMB / SSH banner / SNMP / DNS                                                                                                   |
| `ctf --analyze <file>`                                       | Any file: what it is, flags inside, which solver to run next                                                                                                                           |
| `ctf --web <url>`                                            | Server, robots.txt, hidden directories and their files                                                                                                                                 |
| `ctf --source <url>`                                         | Comments, hidden fields, passwords in HTML/JS, encoded strings                                                                                                                         |
| `ctf --form-brute <login url>`                               | POST login: usernames first, then the password                                                                                                                                         |
| `ctf --lfi "curl -s '<url>?page={P}'" --file /path/to/file`  | LFI testing                                                                                                                                                                            |
| `ctf --sqlmap-auto "<url>"`                                  | Runs sqlmap on a page (`--data '<body>'` `--cookie 'name=value'`); on a miss it says what to try                                                                                       |
| `ctf --sqlmap-levels <url> --run`                            | The same on the five practice-site pages (without `--run` it only prints the commands)                                                                                                 |
| `ctf --overwrite <host:port> --src chall.c`                  | Overflow into the variable after a buffer (`--offset N --value 0x..`)                                                                                                                  |
| `ctf --ret2shell <host:port> --binary ./file`                | Runs `bof/ret2shell.py` (a classmate's script) if it is on this machine                                                                                                                |
| `ctf --folder <dir>`                                         | Every downloaded file in a folder: captures, images, programs, sources, hashes. Asks for what only the challenge text holds (Wi-Fi name, MAC, password, the address:port of a program) |
| `ctf --pwn-check <binary \| file.c>`                         | Which attack fits a program or its source, and the command for it                                                                                                                      |
| `ctf --find-input "<page url>"`                              | Where does a page take its input: address value, cookie name, form, script. Prints the next command                                                                                    |
| `ctf --pcap <capture>`                                       | Plain capture: logins sent in clear text (FTP, Telnet, POP3, HTTP), forms, files, flags                                                                                                |
| `ctf --crack <hash \| file>`                                 | Password hashes: names the type, runs john, shows the password (`--wordlist LIST`)                                                                                                     |
| `ctf --ssh-brute <host:port>`                                | SSH password guessing with hydra, short list only (`--user NAME --wordlist LIST`)                                                                                                      |
| `ctf --ret2win <host:port>`                                  | Buffer overflow to a `win()` function, 32- or 64-bit                                                                                                                                   |
| `ctf --fmt-remote <host:port>`                               | Format string leak: every position as number, letters and text; rebuilds a flag stored on the stack even when no position prints it; `--fmt-write` for `%n` (32-bit)                   |
| `ctf --stegchain <image>`                                    | steghide / appended data / morse / base32; `--lsb` for PNG bit planes                                                                                                                  |
| `ctf --wifi-hidden \| --wifi-crack \| --evil-twin <capture>` | Wireless challenges                                                                                                                                                                    |
| `ctf --decode "<string>"`                                    | base64 / hex / rot / url and more                                                                                                                                                      |
| `ctf --show`                                                 | Every flag and finding so far, with where it came from                                                                                                                                 |

## Flag Detection

Every run ends with **"FLAGS FOUND IN THIS RUN"**, matching:

- `ehCTF{...}` in any letter case
- `SIT{...}`
- `flag{...}`
- anything ending in `CTF{...}`
- bare 32-character lab flags
- flags hidden as base64 / hex / ROT13 / backwards

A wrapper it was never told about (`WORD{...}`) is listed under **POSSIBLE FLAGS**; make it a normal flag with:

```bash
export FLAGFMT=WORD
```

A plain-value answer (a name, a place, a version) cannot be recognised by shape — read the command's own output for it.

> **Practice-set wireless answers:** put `FLAGFMT=SIT` in front of the command.

## What the Kit Does Not Do

See `COVERAGE.md` for the full check.

- It has no SQL injection code of its own: `--sqlmap-auto` runs sqlmap and reads its output.
- Echo32-style challenges (code on an executable stack) need `bof/ret2shell.py`; the kit only launches it.
- `--ret2win` cannot work through a menu first.
- `--overwrite` needs you to check the source it read.
- 64-bit `%n` writes; Burp; whois/country lookups (run whois yourself) are not handled.

## Scope

**Only point it at targets the challenge names, and never at the CTFd scoreboard site.**
