# bof-scripts

A beginner-friendly toolkit for the **memory-corruption questions in teaching CTFs**:
stack buffer overflows, ret2win, shellcode, format strings and ret2libc.

You give it the service's address and whatever you were handed (a binary, a `.c` file, a
sentence from the question paper). It works out which bug it is, tries the matching
technique, and prints the flag — or tells you plainly why it could not.

- **Pure Python standard library.** No pwntools, no pip installs, works offline.
- **One command, no options to remember:** `python3 go.py`
- **Honest output.** It does not claim a flag or a shell unless it can prove it.

> **Use it only on targets you are allowed to attack**: your own machines, CTF challenges,
> and course labs you are enrolled in. Everything in `testbed/` is a local practice program
> with a dummy flag.

---

## Contents

1. [Requirements](#1-requirements)
2. [Install and check](#2-install-and-check)
3. [Quick start](#3-quick-start)
4. [Examples](#4-examples)
5. [Reading the output](#5-reading-the-output)
6. [What flags it recognises](#6-what-flags-it-recognises)
7. [When it does not solve](#7-when-it-does-not-solve)
8. [What it handles, and what it does not](#8-what-it-handles-and-what-it-does-not)
9. [Practise without a server](#9-practise-without-a-server)
10. [The tools](#10-the-tools)
11. [Repository layout](#11-repository-layout)
12. [Testing](#12-testing)
13. [FAQ](#13-faq)

---

## 1. Requirements

| Need | Why | Debian / Ubuntu / Kali |
|---|---|---|
| Linux, x86-64 | the tools target 32- and 64-bit x86 Linux programs | — |
| Python 3.10 or newer | the tools themselves | `sudo apt install python3` |
| `objdump`, `readelf`, `strings` | reading a binary you were given | `sudo apt install binutils` |
| `gcc`, `make`, 32-bit libraries | **only** to build the practice programs in `testbed/` | `sudo apt install gcc make gcc-multilib` |

Developed and fully tested on Kali Linux with Python 3.13. Earlier versions were also run
on Debian 12 (Python 3.11) and Ubuntu 22.04 (Python 3.10); `bash smoke.sh` tells you whether
the current one works on your machine.

## 2. Install and check

```bash
git clone <this repository>
cd bof-scripts
bash smoke.sh
```

`smoke.sh` takes about 30 seconds. It checks your Python and binutils, builds the practice
programs if it can, and solves a few of them. It ends with **`TOOLKIT OK`** or tells you
what is missing.

## 3. Quick start

```bash
python3 go.py
```

It asks four questions. Press Enter to skip any you do not have.

| It asks | You type |
|---|---|
| **Host** | the first half of `nc HOST PORT` from the question |
| **Port** | the second half |
| **A file you were given** | the path to the program or its `.c` source (drag it into the terminal) |
| **Does the paper tell you anything?** | any sentence containing a number, an address or a password; finish with an empty line |

Then look for a line starting **`[+] FLAG`**.

You can also give everything on one line, in any order:

```bash
python3 go.py 10.0.0.5 5002 ./chall chall.c paper.txt
```

A file is recognised by what it *is*: a compiled program is the binary, a `.c`/`.h` file is
the source, any other text file is treated as notes.

## 4. Examples

**Only an address** — the service itself prints what is needed:

```bash
python3 go.py 10.0.0.5 5002
```

**You were given the program:**

```bash
python3 go.py 10.0.0.5 5003 ./echo32
```

**You were given the source:**

```bash
python3 go.py 10.0.0.5 5001 chall.c
```

**The values are on the question paper.** Run `python3 go.py`, skip the file, and type what
the paper says:

```
  > win() is at 0x401186
  > offset to RIP: 40 bytes
  >
```

**A format-string question with an address on paper:**

```
  > The secret is stored at 0x0804c060
```

```
  > target is at 0x40406c. It must equal 0xDEADBEEF
```

```
  > puts GOT is at 0x0804c018, win() is at 0x80492f0
```

**A login in front of the bug:**

```
  > password: hunter2
```

### What the notes reader understands

| You write | It reads |
|---|---|
| `the buffer is 64 bytes`, `72 A's`, `'A' * 76` | a buffer size (it tries size + 4, + 8, … for the real distance) |
| `offset to RIP: 40 bytes`, `the offset is 76` | the exact distance to the return address |
| `win() is at 0x401186`, `jmp esp is at 0x8049196` | an address |
| `the code is 0x1337`, `key = 0x40 + 12`, `1,024`, `1337h`, `0b1010` | a value |
| `password: hunter2` | a word to type at a login |
| a pasted C snippet | struct layout and the compared constant |

It ignores numbers that follow words like *port*, *question*, *marks*, *page*. It does not
read PDFs or images (type the sentence in), and it does not solve riddles (work the number
out and type the result). It always shows what it understood before running:

```
[*] from your notes I read:
  offset   40 (0x28)        <- "offset to RIP: 40 bytes"
  address  0x401186         <- "win(): 0x401186"
```

If a line there is wrong, fix your wording and run again.

## 5. Reading the output

```
[*]   information: what it found, what it is about to try
[!]   a warning, or a decision you should read
[+]   success

=== [2/9] ret2win (25s cap) - why this was chosen
[+] FLAG (off=40 addr=0x401186): SIT{...}
[+] payload: b'AAAA...\x86\x11@\x00...'
[+] SOLVED via ret2win
```

| You see | It means |
|---|---|
| `[+] FLAG (...)` | the answer, and how it was reached |
| `READ THIS FIRST` | printed last when nothing worked: what was established and what to try |
| `Ctrl-C if that already answers it` | a diagnosis is already on screen; stopping is fine, the summary is kept |
| `ONE OF N in this reply` | a shell printed several flags; check which one is yours |
| `answers a HARMLESS input with flag-shaped text` | the service shows something flag-like without being exploited. Not called solved (it may be an example), but the candidate is printed on its own line |
| `prints bare hex tokens (32 digits) without being exploited` | the service shows something like a session ID, so hex of that length is not counted as a flag |
| `your notes say offset 76; this binary's own layout says 60` | your notes and your copy of the program disagree: wrong question's notes, or the server runs a different build |

Everything sent and received is saved to `solve_transcript.log`.

## 6. What flags it recognises

| The service prints | Found? |
|---|---|
| `SIT{...}` | yes |
| `flag{...}`, `ehCTF{...}`, `picoCTF{...}` — any `name{...}` | yes (a body of only 3–4 characters needs a name of 4 or more) |
| `flag is XYZ`, `flag: XYZ` | yes |
| a wrapped flag that was hex-encoded, base64-encoded, reversed or rot13'd | yes; decoded, with a note saying how it was disguised |
| a bare hex string of 16–64 digits | on overflow questions only |
| a bare string with no wrapper (plain base64, random letters) | no — nothing marks it as a flag |

For the last two, either read the reply the tool prints under *"last response (the flag
may be here but unrecognised)"*, or give it the pattern:

```bash
python3 solve.py HOST PORT --binary ./chall --run --flag-regex '[A-Za-z0-9+/=]{20,}'
```

## 7. When it does not solve

Work down this list. Each step takes under a minute.

1. **Read `READ THIS FIRST`** at the bottom of the output.
2. **Give it more.** Run `go.py` again and type in anything from the paper you left out.
3. **Connect by hand** with `nc HOST PORT`. Is there a menu or a login first? A number that
   changes each time? Add what you see to your notes.
4. **Ask about the program:** `python3 elftool.py ./chall` ends with a plain-language verdict.
5. **Run it once more** if the technique was shellcode or ret2libc: those use addresses that
   change on every connection.
6. **Move on.**

`RUNBOOK.md` lists the manual command for each situation.

## 8. What it handles, and what it does not

**Handles, without being told which:**

| Family | Variants |
|---|---|
| Overflow into a variable | struct fields, `strcmp` targets, run-time values the service announces, a secret you can overwrite along with what it is compared to |
| Overflow into the return address | ret2win (with arguments, 32/64-bit, stack alignment), bad bytes, tight reads, canary (leak or fork brute-force), PIE (leak or partial overwrite) |
| Shellcode | `jmp esp` / `jmp rsp`, a printed buffer address with a NOP sled, no-offset spray, `scanf`-safe shellcode, a server running a shifted build |
| Format string | read the stack, `%s` at an address, `%n` / `%hn` writes, GOT → `win` or `system`, leak-then-overflow, 32- and 64-bit |
| ret2libc | leak `puts`, compute the base, call `system("/bin/sh")` |
| In front of the bug | menus, simple gates, logins (credentials from the binary or your notes), a harmless first prompt |
| No files at all | values read from the banner, offsets found by crash search, values sprayed |

**Does not handle** (it says so when it meets them):

- a login whose password is not in the program, the banner or your notes;
- a format string whose buffer is not on the stack, when a write is needed;
- PIE with a single `printf` and no second input;
- a different libc on the server when you do not have that libc file;
- anything outside these families: heap, kernel, SROP, ret2csu, encoding puzzles.

## 9. Practise without a server

`testbed/` holds about 130 small vulnerable programs, each two lines of explanation and
roughly 40 lines of C.

```bash
make -C testbed            # build them all, with dummy flags
bash drill.sh 1 easy       # start a random one and tell you only the port
bash drill.sh 1 hard       # the awkward ones: gates, menus, logins
```

`drill.sh` waits while you solve it with `go.py`, then reveals which program it was.

To serve one yourself:

```bash
python3 testbed/server.py testbed/paperlab64 --port 4000 --cwd testbed
python3 go.py 127.0.0.1 4000
```

`testbed/lagproxy.py` adds network delay in front of a local program, to check that a
solve is not relying on localhost speed.

## 10. The tools

You normally run only `go.py`. The rest are what it drives, and can be run directly.

| Tool | Does |
|---|---|
| `go.py` | the front door: asks what you have, reads notes, runs the solver |
| `solve.py` | inspects, ranks the techniques and runs them. Without `--run` it only reports |
| `notes.py` | turns sentences into offsets, sizes, addresses, values and words |
| `auto_pwn.py` | overflows: variable overwrite, ret2win, canary, PIE, offset search |
| `fmtpwn.py` | the format-string engine (`--goal dump` shows the stack) |
| `fmtwrite.py` | one targeted `%hn` write |
| `ret2shell.py` | shellcode |
| `ret2libc.py` | ret2libc |
| `elftool.py` | protections, symbols, offsets and gadgets of a binary, with a verdict |
| `verify.py` | is the server running the same build as your copy? |
| `analyze.py` | first look at a `.c` file or a binary |
| `flags.py` | flag recognition and the "did a shell really answer" check |
| `compat.py` | checks Python and binutils on this machine |
| `bof.py`, `fmt.py` | small manual helpers |

Useful direct commands:

```bash
python3 solve.py HOST PORT --binary ./chall          # look only: what does it think this is?
python3 solve.py HOST PORT --binary ./chall --run    # what go.py runs
python3 elftool.py ./chall                           # the binary, explained
python3 fmtpwn.py HOST PORT --goal dump              # stack words through a format string
python3 notes.py paper.txt                           # what would it read from these notes?
```

## 11. Repository layout

```
bof-scripts/
├── README.md        this file
├── HOWTO.md         the three bugs explained, and how to read the output
├── RUNBOOK.md       reference: the command for each situation
├── go.py            start here
├── solve.py  auto_pwn.py  fmtpwn.py  fmtwrite.py  ret2shell.py  ret2libc.py
├── notes.py  flags.py  elftool.py  verify.py  analyze.py  compat.py  bof.py  fmt.py
├── smoke.sh         30-second health check
├── drill.sh         blind practice
├── selftest.sh      full regression suite (see Testing)
├── docs/
│   ├── TOOLS.md     every tool and option in detail
│   └── STATUS.md    what has been tested, round by round
└── testbed/
    ├── Makefile     builds every practice program and its dummy flag
    ├── src/         one small C file per practice program
    ├── server.py    serves a program on a local port
    ├── launch.sh    helpers for scripts
    ├── lagproxy.py  adds network delay
    └── fuzzflags.py checks that junk is never reported as a flag
```

## 12. Testing

| Command | Time | Checks |
|---|---|---|
| `python3 compat.py` | 1 s | Python version and binutils |
| `bash smoke.sh` | 30 s | the tools run and solve a handful of practice programs |
| `bash selftest.sh` | about 55 min | the full regression suite: 325 checks |

`selftest.sh` is the author's development suite. A small number of its checks (roughly a
dozen) use extra held-out binaries that are **not in this repository** and will fail
elsewhere; the rest run from `testbed/`. For "does it work on my machine", `smoke.sh` is the right test.

The suite includes negative tests: programs with no bug at all that must never be reported
as solved (an echo service, a word-splitting parser, an example flag in a reply, a random
session ID, a polite length check).

## 13. FAQ

**Do I need to understand buffer overflows to use it?**
No. Type the host, port, file and what the paper says, and read the `[+] FLAG` line.
`HOWTO.md` explains the bugs if you want to follow what the output means.

**Does it need pwntools or internet access?**
No. Standard library only.

**It says SOLVED but the flag looks wrong.**
Read the line: if it says `ONE OF N`, several flags were printed and yours may be another.

**It solved the practice program but not the real server.**
The server may run a different build from your copy. `python3 verify.py --binary ./chall
--host HOST --port PORT` checks, and prints the shift to use.

**It failed once on something simple.**
Run it again. Shellcode and ret2libc depend on addresses that change every connection.

**Can it attack arbitrary real software?**
No. It is built for small teaching challenges in the families listed above.
