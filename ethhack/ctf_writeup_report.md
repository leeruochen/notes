# CTF Writeup Report: CyberBlitz / CYBLTZ 2026

A comprehensive technical writeup of solved reverse engineering, forensics, cryptography, protocol analysis, and VCS data-carving challenges, including tools and extraction techniques used.

## Master Tooling Index

| Category                          | Primary Utilities / Libraries                                    | Practical Use Case in Solves                                                              |
| --------------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Static Inspection & Recon**     | `file`, `strings`, `xxd`, `hexdump -C`                           | Inspecting magic headers, embedded strings, and RIFF chunk headers                        |
| **Binary Reversing & Debugging**  | `objdump`, `gdb`, `strace`, `gcc` (`LD_PRELOAD`)                 | Disassembling x86-64 PEs/ELFs, tracing syscalls, intercepting `memfd_create`              |
| **Cryptanalysis & Math**          | Python (`hashlib`, `pow()`, modular inverse)                     | Computing discrete modular arithmetic for ECDSA nonce recovery                            |
| **Forensics & Packet Analysis**   | Python (`struct`, `wave`, `base64`)                              | Parsing raw RIFF WAV subchunks, PCAP linked-list stream reassembly                        |
| **Git / VCS Forensics**           | `tail`, `grep`, `git unpack-objects`, `git cat-file`, `git show` | Carving raw `PACK` streams from bundles and recovering orphaned/unreferenced commit trees |
| **Custom Architecture Emulation** | Python 3                                                         | Constructing register VMs to execute custom bytecode                                      |

## Table of Contents

1. [Singlish as a Service (Crypto / Esoteric)](#1-singlish-as-a-service)
2. [OFF BY ONE (Crypto / ECDSA Nonce Attack)](#2-off-by-one)
3. [Scytale (Crypto / Transposition)](#3-scytale)
4. [Infinite GPA Hack (Reverse Engineering / Python Obfuscation)](#4-infinite-gpa-hack)
5. [Eight Registers, No Compiler (Reverse Engineering / Custom VM)](#5-eight-registers-no-compiler)
6. [The Lift Only Goes Sideways (Reverse Engineering / Embedded CAN Replay)](#6-the-lift-only-goes-sideways)
7. [Soundcheck (Audio Forensics / RIFF WAV Parsing)](#7-soundcheck)
8. [Packet Kueh (Network Forensics / Protocol Analysis)](#8-packet-kueh)
9. [One More Commit Before Kopitiam (Git Forensics / Data Carving)](#9-one-more-commit-before-kopitiam)

---

## 1. Singlish as a Service

* **Category:** Cryptography / Esoteric Encoding
* **Files Provided:** `phrase_cards.txt`, `midnight.sas`
* **Flag:** `CYBLTZ2026{d0n7_pL4y_pL4y}`

### Tools & Commands Used

* **`cat` / Terminal Pager:** Inspecting raw ASCII card files and token strings:
  ```bash
  cat phrase_cards.txt
  cat midnight.sas
  ```

### Challenge Overview & Analysis

The challenge simulates a transit kiosk encoding phrases into compact tokens. Inspecting `phrase_cards.txt` revealed a direct pipe-delimited token-to-fragment substitution scheme:

```text
CARD 1 PLAIN: d0n7|pL4y          CODE: K2|P7
CARD 2 PLAIN: CYBLTZ2026{|pL4y|} CODE: L4|P7|R9
CARD 3 PLAIN: d0n7|_|pL4y        CODE: K2|SP|P7
```

From the active (non-retired) cards, we build the symbol dictionary:
* `L4` $\to$ `CYBLTZ2026{`
* `K2` $\to$ `d0n7`
* `SP` $\to$ `_`
* `P7` $\to$ `pL4y`
* `R9` $\to$ `}`

The encoded announcement in `midnight.sas` was:
```text
L4|K2|SP|P7|SP|P7|R9
```

### Solution

Mapping each token sequentially:
$$
\text{L4} + \text{K2} + \text{SP} + \text{P7} + \text{SP} + \text{P7} + \text{R9} \implies \text{CYBLTZ2026\{d0n7\_pL4y\_pL4y\}}
$$

---

## 2. OFF BY ONE

* **Category:** Public Key Cryptography (ECDSA)
* **Files Provided:** `distrib.txt` (Parameters, 2 signatures, and a hex ciphertext)
* **Curve:** NIST P-256 (`secp256r1`) / SHA-256

### Tools & Commands Used

* **Python 3:** Standalone script implementing modular arithmetic, modular multiplicative inverses (`pow(x, -1, n)`), and SHA-256 hashing via `hashlib`.

### Mathematical Vulnerability

Standard ECDSA signatures satisfy:
$$
s \equiv k^{-1}(h + r \cdot d) \pmod n \iff k \cdot s \equiv h + r \cdot d \pmod n
$$

Where $h = \text{SHA256}(m)$, $n$ is the curve order, and $d$ is the private signing key.

The description stated that nonces are never reused, but highlighted an "off-by-one" condition between the signatures:
$$
k_2 = k_1 + \delta \pmod n \quad \text{where } \delta \in \{1, -1\}
$$

Substituting $k_2$ into the second signature equation:
$$
(k_1 + \delta) s_2 \equiv h_2 + r_2 \cdot d \pmod n
$$
$$
k_1 s_2 + \delta s_2 \equiv h_2 + r_2 \cdot d \pmod n
$$

Multiplying the first signature equation by $r_2$ and the second by $r_1$ eliminates the private key $d$:
$$
r_2 k_1 s_1 \equiv r_2 h_1 + r_1 r_2 d \pmod n
$$
$$
r_1 k_1 s_2 + r_1 \delta s_2 \equiv r_1 h_2 + r_1 r_2 d \pmod n
$$

Subtracting the two equations:
$$
k_1 (r_2 s_1 - r_1 s_2) \equiv r_2 h_1 - r_1 h_2 + \delta r_1 s_2 \pmod n
$$
$$
k_1 \equiv \frac{r_2 h_1 - r_1 h_2 + \delta r_1 s_2}{r_2 s_1 - r_1 s_2} \pmod n
$$

### Solution Script

Once $k_1$ was determined, the private key $d$ was recovered:
$$
d \equiv r_1^{-1} (k_1 s_1 - h_1) \pmod n
$$

XORing the 32-byte big-endian representation of $d$ with the provided ciphertext decoded the flag:

```python
import hashlib

n = 0xFFFFFFFF00000000FFFFFFFFFFFFFFFFBCE6FAADA7179E84F3B9CAC2FC632551
m1 = b"This is message 1"
r1 = 84918391022880580917404599665324292434691745381102244941556045001372735351319
s1 = 87741392621874592924723018144529307098849418024912183482695319500250268744009

m2 = b"This is message 2"
r2 = 34231198634407520436945515392215999690091853063969785984537880200814831292997
s2 = 53371404715937007600535556326097711593280131994602709883282028166572336213092
ciphertext = bytes.fromhex("2b6612d7406f3a21d64291885994eaec3502845602262323d33cfb46bf1ad0b1")

h1 = int.from_bytes(hashlib.sha256(m1).digest(), "big")
h2 = int.from_bytes(hashlib.sha256(m2).digest(), "big")

for delta in [1, -1]:
    denom = (r2 * s1 - r1 * s2) % n
    numer = (r2 * h1 - r1 * h2 + delta * r1 * s2) % n
    k1 = (numer * pow(denom, -1, n)) % n
    d = (pow(r1, -1, n) * (k1 * s1 - h1)) % n
    key_bytes = d.to_bytes(32, "big")
    decrypted = bytes(c ^ k for c, k in zip(ciphertext, key_bytes))
    print(f"Decrypted (delta={delta}): {decrypted}")
```

---

## 3. Scytale

* **Category:** Classical Cryptography / Web Puzzle
* **Platform:** Interactive Web App (`http://45.77.247.35:22000`)
* **Intel Recovered:**
  * **Location:** `THE OLD CLOCK TOWER`
  * **Time:** `NINE PM`
  * **Password:** `SILVER FALCON`

### Tools & Commands Used

* **Browser DevTools / In-Game Workbench:** Navigating the rooms and collecting raw cipher ribbons and rods.
* **Custom Python Grid Transposition Solver:** Testing transposition factors / rod circumferences.

### Analysis & Decryption

In a Scytale cipher, a strip is wrapped around a cylinder of circumference $R$ (rows). Text written along columns is read across rows once wrapped.

By inspecting the ribbons on the three rods:

1. **Rod B + Strip 1:**
   ```text
   THEMEE
   TINGPL
   ACEIST
   HEOLDC
   LOCKTO
   WERXXX
   ```
   $\to$ `THE MEETING PLACE IS THE OLD CLOCK TOWER`

2. **Rod C + Strip 2:**
   ```text
   ARRIVEATNINEPMAN
   DWAITFORTHECOURI
   ERAFTERDARKXXXX
   ```
   $\to$ `ARRIVE AT NINE PM AND WAIT FOR THE COURIER AFTER DARK`

3. **Rod A + Strip 3:**
   ```text
   THESECRETPASSWO
   RDISSILVERFALCO
   NSAYITTOTHECOUR
   IERATTHEMEETING
   ```
   $\to$ `THE SECRET PASSWORD IS SILVER FALCON SAY IT TO THE COURIER AT THE MEETING`

Submitting the location, time, and courier password at the workbench completed the challenge.

---

## 4. Infinite GPA Hack

* **Category:** Reverse Engineering / Python Deobfuscation
* **Files Provided:** Python script with terminal styling
* **Flag:** `CYBLTZ2026{inf1n1t3_gp4_gl1tch}`

### Tools & Commands Used

* **`cat` / Terminal Window Expansion:** Revealing whitespace-padded text beyond standard column boundaries.
* **`base64` CLI / Python `base64` module:** Decoding the embedded execution payload:
  ```bash
  echo "dHJ5OgogICAgaWYgb3BlbigiLi9mbGFnLnR4dCIpLnJlYWQoKSA9PSAiQ1lCTFRaMjAyNntpbmYxbjF0M19ncDRfZ2wxdGNofSI6IHByaW50KCJUaGF0J3MgdGhlIGNvcnJlY3QgZmxhZyEiKTsgc3lzLmV4aXQoMCkKZXhjZXB0OgogICAgcGFzcwo=" | base64 -d
  ```

### Analysis & Solution

Inspecting `main()` showed an intentional horizontal whitespace gap designed to hide code off-screen:

```python
loading_bar();                                                                                                              exec(base64.b64decode(b"dHJ5OgogICAgaWYgb3BlbigiLi9mbGFnLnR4dCIpLnJlYWQoKSA9PSAiQ1lCTFRaMjAyNntpbmYxbjF0M19ncDRfZ2wxdGNofSI6IHByaW50KCJUaGF0J3MgdGhlIGNvcnJlY3QgZmxhZyEiKTsgc3lzLmV4aXQoMCkKZXhjZXB0OgogICAgcGFzcwo=").decode())
```

Decoding the base64 string exposed the exact flag string comparison:

```python
try:
    if open("./flag.txt").read() == "CYBLTZ2026{inf1n1t3_gp4_gl1tch}": 
        print("That's the correct flag!")
        sys.exit(0)
except:
    pass
```

---

## 5. Eight Registers, No Compiler

* **Category:** Reverse Engineering / Custom VM Architecture
* **Files Provided:** `hello.vm`, `math.vm`, `courier.vm`

### Tools & Commands Used

* **`file`:** Identifying raw binary data format.
* **`xxd`:** Dumping raw hex bytes to reverse opcode structures:
  ```bash
  xxd hello.vm
  xxd math.vm
  xxd courier.vm | head -n 30
  ```
* **Python 3:** Developing a standalone bytecode emulator.

### Instruction Set Reverse Engineering

Comparing `hello.vm` and `math.vm` against known output strings (`'H'`, `'I'`) and mathematical additions identified fixed-width 4-byte little-endian instructions:

```text
Header: 45 56 4d 43 ("EVMC")
Format: [OPCODE: 1 byte] [ARG1: 1 byte] [ARG2: 1 byte] [ARG3: 1 byte]
```

| Opcode | Mnemonic | Operation | Description | 
| ----- | ----- | ----- | ----- | 
| `0x17` | `LOAD_IMM` | `regs[R] = imm16` | Loads 16-bit value (`b2 | (b3 << 8)`) into register `R` | 
| `0x39` | `ADD` | `regs[dst] = (regs[s1] + regs[s2]) & 0xFF` | 8-bit modular addition | 
| `0x52` | `XOR` | `regs[dst] = regs[s1] ^ regs[s2]` | Bitwise XOR | 
| `0x8A` | `PUTC` | `print(chr(regs[R]))` | Emits character from register `R` | 
| `0xFF` | `HALT` | `exit()` | Terminates VM execution | 

### Execution

The custom Python emulator decoded and executed `courier.vm`:

```python
def run_evm(filename):
    with open(filename, "rb") as f:
        data = f.read()
    regs = [0] * 8
    pc = 4
    out = []
    while pc < len(data):
        op, r1, r2, r3 = data[pc : pc + 4]
        pc += 4
        if op == 0x17:
            regs[r1] = r2 | (r3 << 8)
        elif op == 0x39:
            regs[r1] = (regs[r2] + regs[r3]) & 0xFF
        elif op == 0x52:
            regs[r1] = regs[r2] ^ regs[r3]
        elif op == 0x8A:
            out.append(chr(regs[r1] & 0xFF))
        elif op == 0xFF:
            break
    return "".join(out)

print(run_evm("courier.vm"))
```

---

## 6. The Lift Only Goes Sideways

* **Category:** Reverse Engineering / Automotive CAN Protocol
* **Files Provided:** `controller_firmware.bin` (PE32+), `lift_can.log`, `maintenance_card.txt`

### Tools & Commands Used

* **`file` & `hexdump` / `strings`:** Identifying 64-bit Windows PE format and discovering the `.lift` section and `LIFTST8` signature:
  ```bash
  file controller_firmware.bin
  strings controller_firmware.bin | head -n 20
  ```
* **`objdump`:** Disassembling code and extracting raw sections:
  ```bash
  objdump -M intel -d controller_firmware.bin
  objdump -s -j .lift controller_firmware.bin
  ```
* **Python 3 (`struct`):** Emulating the state machine and parsing `lift_can.log`.

### Firmware Reverse Engineering

1. **CRC32 & Table Decryption:**
   * Section `.lift` held 132 four-byte transition records XOR-encrypted with `0xA5`, verified by CRC32 `0x71c52b35`.
2. **Transition Schema:**
   $$
   \text{Entry} = [\text{current\_state}, \text{next\_state}, \text{can\_input\_byte}, \text{emitted\_label}]
   $$
3. **Dispatch Logic:**
   * Filtered exclusively for CAN ID `0x321`.
   * Matched `(current_state, payload_byte)` against `(entry[0], entry[2])`.
   * On match, updated `current_state = entry[1]` and emitted `entry[3]`.

### Python Replay & Decoding

```python
import struct

def solve():
    label_map = {}
    with open("maintenance_card.txt", "r") as f:
        for line in f:
            if "=" in line and not line.startswith("#"):
                k, v = line.strip().split("=", 1)
                label_map[int(k, 16)] = v

    with open("controller_firmware.bin", "rb") as f:
        data = f.read()

    idx = data.find(b"LIFTST8")
    count = struct.unpack("<H", data[idx + 7 : idx + 9])[0]
    raw = data[idx + 9 : idx + 9 + count * 4]
    dec = bytes(b ^ 0xA5 for b in raw)

    transitions = [
        (dec[i*4], dec[i*4+1], dec[i*4+2], dec[i*4+3])
        for i in range(count)
    ]

    state = 0
    emitted = []
    with open("lift_can.log", "r") as f:
        for line in f:
            parts = line.strip().split()
            if len(parts) >= 3 and parts[1] == "321":
                inp = int(parts[2][:2], 16)
                for cs, ns, ib, lbl in transitions:
                    if cs == state and ib == inp:
                        state = ns
                        emitted.append(lbl)
                        break

    print("Decoded:", "".join(label_map.get(l, "?") for l in emitted))

solve()
```

---

## 7. Soundcheck

* **Category:** Audio Forensics / Container Parsing (RIFF WAV)
* **Files Provided:** `soundcheck.wav`
* **Flag:** `CYBLTZ2026{cu3_7h3_r34L_ch0ru5}`

### Tools & Commands Used

* **Python 3 (`struct`, `wave`, `base64`):** Parsing RIFF chunk headers, extracting cue point sample offsets, and dumping associated data list (`LIST` / `adtl`) label sub-chunks.

### Container Reverse Engineering & Analysis

The challenge points directly to non-audio metadata embedded inside the RIFF container:
1. **`cue ` Chunk:** Defines sample offsets where cue points are placed in the waveform.
2. **`LIST` (`adtl`) Chunk:** Contains `labl` records associating human-readable text labels with specific Cue IDs.

Running a custom RIFF chunk dumper revealed 10 cue markers scattered chronologically along the sample timeline:

| Cue ID | Sample Offset | Label | 
| ----- | ----- | ----- | 
| 5 | 340,000 | `Q1lCT` | 
| 9 | 712,800 | `FRaMj` | 
| 1 | 121,600 | `AyNnt` | 
| 7 | 130,400 | `jdTNf` | 
| 2 | 939,200 | `N2gz` | 
| 8 | 948,000 | `X3Iz` | 
| 4 | 156,800 | `NExf` | 
| 0 | 765,600 | `Y2gw` | 
| 6 | 174,400 | `cnU1` | 
| 11 | 983,200 | `fQ==` | 

### Solution & Decoding

Concatenating the labels in chronological order of their waveform sample offsets:

```text
Q1lCT + FRaMj + AyNnt + jdTNf + N2gz + X3Iz + NExf + Y2gw + cnU1 + fQ==
```

Yields the full Base64 string:

```text
Q1lCTFRaMjAyNntjdTNfN2gzX3IzNExfY2gwcnU1fQ==
```

Decoding via Base64 exposes the exact flag:

```python
import base64

b64_str = "Q1lCTFRaMjAyNntjdTNfN2gzX3IzNExfY2gwcnU1fQ=="
print(base64.b64decode(b64_str).decode("utf-8"))
# Output: CYBLTZ2026{cu3_7h3_r34L_ch0ru5}
```

---

## 8. Packet Kueh

* **Category:** Network Forensics / Protocol Analysis
* **Files Provided:** `packet_kueh.pcap`
* **Flag:** `CYBLTZ2026{l4y3r5_b3f0r3_f1ll1nG}`

### Tools & Commands Used

* **Python 3 (`struct`, `base64`, `itertools`):** Parsing PCAP global/packet records, extracting raw application layer payloads, and filtering conflicting frames.

### Analysis & Protocol Mechanics

Inspecting the PCAP payload reveals custom application tokens:

```text
PKUEH|<current_idx>|<next_idx>|<b64_data>|<crc32_hex>
```

The capture contains 7 distinct stages (`current_idx` from 0 to 6). For each stage, two competing packets were transmitted across the wire: one authentic courier packet and one decoy packet (e.g., payloads decoding to `cold-`, `-steamer-`, `kueh-`, `wrong`).

Examining the second integer field reveals a linked list where `field2` points to the next packet's index:
$$
0 \to 4 \to 1 \to 6 \to 2 \to 5 \to 3 \to 255 \text{ (End-of-Chain)}
$$

| Node Index | Choice A | Choice B | Authentic Fragment Selected | 
| ----- | ----- | ----- | ----- | 
| **0** | `Q1lCTFR` (`CYBLT`) | `Y29sZC1` (`cold-`) | `Q1lCTFR` | 
| **4** | `aMjAyNn` (`Z2026{`) | `rdWVoLX` (`kueh-`) | `aMjAyNn` | 
| **1** | `tsNHkz` (`l4y3`) | `dyb25n` (`wrong`) | `tsNHkz` | 
| **6** | `cjVfYj` (`r5_b`) | `LXN0ZW` (`-ste`) | `cjVfYj` | 
| **2** | `NmMHIz` (`3f0r3`) | `FtZXIt` (`amer-`) | `NmMHIz` | 
| **5** | `X2YxbG` (`_f1l`) | `Y29sZC` (`cold`) | `X2YxbG` | 
| **3** | `1rdWVo` (`l1nG}`) | `wxbkd9` (`decoy`) | `1rdWVo` | 

### Solution Script

Following the linked list hops ($0 \to 4 \to 1 \to 6 \to 2 \to 5 \to 3$) and eliminating the decoys reconstructs the base64 flag payload:

```python
import base64

# Traversal order: 0 -> 4 -> 1 -> 6 -> 2 -> 5 -> 3
authentic_chain = [
    "Q1lCTFR",  # Node 0 (points to 4)
    "aMjAyNn",  # Node 4 (points to 1)
    "tsNHkz",   # Node 1 (points to 6)
    "cjVfYj",   # Node 6 (points to 2)
    "NmMHIz",   # Node 2 (points to 5)
    "X2YxbG",   # Node 5 (points to 3)
    "1rdWVo"    # Node 3 (points to 255/terminal)
]

b64_payload = "".join(authentic_chain)

# Pad to valid Base64 boundary
pad = (4 - (len(b64_payload) % 4)) % 4
flag = base64.b64decode(b64_payload + "=" * pad).decode("ascii")
print("Flag:", flag)
# Output: Flag: CYBLTZ2026{l4y3r5_b3f0r3_f1ll1nG}
```

---

## 9. One More Commit Before Kopitiam

* **Category:** Forensics / Git Internals & Data Carving
* **Files Provided:** `kopitiam.bundle`, `built_Receipt.txt`
* **Receipt Reference:** `d3143b8dfed6`
* **Flag / Recovered Data:** `incident_note.txt`

### Tools & Commands Used

* **`grep` & `tail`:** Scanning for raw magic bytes (`PACK`) and carving the binary packfile out of the bundle container.
* **`git init` & `git unpack-objects`:** Initializing a clean workspace and unpacking loose Git objects directly into `.git/objects`.
* **`git cat-file` & `git show`:** Inspecting object headers, parsing tree structures, and extracting file contents by commit/blob hash.

### Challenge Overview & Analysis

The prompt states:
> *"An incident note vanished with its Git branch. A build receipt still points to something the visible history cannot reach. The branch label was removed after the bundle was sealed."*

A standard `git bundle verify` or `git clone` only imports references explicitly registered in the bundle's header. If a branch label was deleted or omitted from the bundle index, Git's porcelain commands will ignore unreferenced objects during cloning or traversal.

However, a Git bundle is simply an ASCII header containing reference declarations, followed by a raw, standard binary Git Packfile starting with the 4-byte signature `PACK` (`0x50 0x41 0x43 0x4B`). Because Git creates delta-compressed packfiles encompassing all commits made prior to bundle sealing, orphaned and unreachable objects remain physically stored within the trailing packfile data.

### Technical Recovery Process

#### 1. Binary Packfile Carving
To extract the unpruned packfile without relying on Git bundle transport logic:

```bash
# Locate the byte offset of the 'PACK' magic header
OFFSET=$(grep -a -b -o "PACK" kopitiam.bundle | head -n 1 | cut -d: -f1)

# Carve out the packfile starting from the detected offset (offset 120 in this bundle)
tail -c +$((OFFSET + 1)) kopitiam.bundle > bundle.pack
```

#### 2. Populating the Object Store
Creating a clean Git repository and feeding the carved packfile directly into `git unpack-objects` inflates all objects—including the orphaned commit—into loose objects under `.git/objects`:

```bash
mkdir kopitiam_recovery && cd kopitiam_recovery
git init -q
git unpack-objects -q < ../bundle.pack
```

#### 3. Inspecting the Receipt Object
Querying the hash provided in `built_Receipt.txt` (`d3143b8dfed6`):

```bash
git cat-file -t d3143b8dfed6
# Output: commit
```

Inspecting the commit details via `git cat-file -p d3143b8dfed6` exposes:

```text
tree 389387a4259b2de904f70430c37d1c3555070746
parent 4b4b94f5adfdefa073b50e846520eb79dd2bf5e9
author Afterlight Builder <builder@afterlight.local> 1788108688 +0800
committer Afterlight Builder <builder@afterlight.local> 1788108688 +0800

record incident note before kopi
```

Checking the root tree (`389387a4259b2de904f70430c37d1c3555070746`) reveals two files:
* `README.md`
* `incident_note.txt`

#### 4. Extracting the Evidence
To read the incident note directly from the unreachable commit:

```bash
git show d3143b8dfed6:incident_note.txt
```

Alternatively, checking out the commit to a new local branch exposes the entire historical state in the working directory:

```bash
git checkout -b recovered-incident d3143b8dfed6
cat incident_note.txt
```