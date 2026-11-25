ICT2212 CTF KIT v10.7
=====================
A command-line helper for the ICT2212 practice CTF and this practice range. One script (ctf-auto.sh) plus a dashboard page.
It runs the usual Kali tools for you (nmap, curl, ffuf, sqlmap, tshark, aircrack-ng, steghide, john ...) and ends every
run with a box of the flags it saw. Next to most results it prints the by-hand command, so you can learn the manual way from it.

INSTALL
  cd ctf-kit && chmod +x install.sh ctf-auto.sh && ./install.sh        (adds the "ctf" command; asks before installing anything)
  First line of  ctf --help  must say v10.7.
  Without installing:  ./ctf-kit/ctf-auto.sh --enum 127.0.0.1
  If scans miss ports or end in "No route to host" (VMware NAT does this), add  --threads 2.
  Do not run apt upgrade / full-upgrade on your exam VM. Take a snapshot before you change anything.

THE COMMANDS YOU WILL USE MOST
  ctf --help                         every command with an example (first line shows the version, last line the profile)
  ctf --enum <ip>                    whole box: port scan, then web / virtual hosts / FTP / SMB / SSH banner / SNMP / DNS
  ctf --analyze <file>               any file: what it is, flags inside, which solver to run next
  ctf --web <url>                    server, robots.txt, hidden directories and their files
  ctf --source <url>                 comments, hidden fields, passwords in HTML/JS, encoded strings
  ctf --form-brute <login url>       POST login: usernames first, then the password
  ctf --lfi "curl -s '<url>?page={P}'" --file /path/to/file
  ctf --sqlmap-auto "<url>"          runs sqlmap on a page ( --data '<body>'  --cookie 'name=value' ); on a miss it says what to try
  ctf --sqlmap-levels <url> --run    the same on the five practice-site pages (without --run it only prints the commands)
  ctf --overwrite <host:port> --src chall.c     overflow into the variable after a buffer ( --offset N --value 0x.. )
  ctf --ret2shell <host:port> --binary ./file   runs bof/ret2shell.py (a classmate's script) if it is on this machine
  ctf --folder <dir>                 every downloaded file in a folder: captures, images, programs, sources, hashes. It asks for what only
                                     the challenge text holds (Wi-Fi name, MAC, password, the address:port of a program)
  ctf --pwn-check <binary | file.c>  which attack fits a program or its source, and the command for it
  ctf --find-input "<page url>"      where does a page take its input: address value, cookie name, form, script. Prints the next command
  ctf --pcap <capture>               plain capture: logins sent in clear text (FTP, Telnet, POP3, HTTP), forms, files, flags
  ctf --crack <hash | file>          password hashes: names the type, runs john, shows the password   ( --wordlist LIST )
  ctf --ssh-brute <host:port>        SSH password guessing with hydra, short list only   ( --user NAME --wordlist LIST )
  ctf --ret2win <host:port>          buffer overflow to a win() function, 32- or 64-bit
  ctf --fmt-remote <host:port>       format string leak: every position as number, letters and text; rebuilds a flag stored
                                     on the stack even when no position prints it;  --fmt-write for %n (32-bit)
  ctf --stegchain <image>            steghide / appended data / morse / base32;  --lsb for PNG bit planes
  ctf --wifi-hidden | --wifi-crack | --evil-twin <capture>
  ctf --decode "<string>"            base64 / hex / rot / url and more
  ctf --show                         every flag and finding so far, with where it came from

EVERY RUN ENDS WITH "FLAGS FOUND IN THIS RUN": ehCTF{...} in any letter case, SIT{...}, flag{...}, anything ending in CTF{...},
bare 32-character lab flags, and flags hidden as base64 / hex / ROT13 / backwards. A wrapper it was never told about (WORD{...})
is listed under POSSIBLE FLAGS; make it a normal flag with  export FLAGFMT=WORD . A plain-value answer (a name, a place, a
version) cannot be recognised by shape: read the command's own output for it.
Practice-set wireless answers: put FLAGFMT=SIT in front of the command.

WHAT THE KIT DOES NOT DO (see COVERAGE.md for the full check)
  - It has no SQL injection code of its own: --sqlmap-auto runs sqlmap and reads its output.
  - Echo32-style challenges (code on an executable stack) need bof/ret2shell.py; the kit only launches it.
  - --ret2win cannot work through a menu first. --overwrite needs you to check the source it read.
  - 64-bit %n writes; Burp; whois/country lookups (run whois yourself).

Only point it at targets the challenge names, and never at the CTFd scoreboard site.
