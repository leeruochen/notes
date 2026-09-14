# BUG HUNT #2 — Study Notes (frame codec + UART link, INF2004 Lab 2)

Seven defects were planted and fixed across `frame.c` (5 bugs) and `bughunt2_pico.c` (2 bugs). The spec in `frame.h` is correct — the job was always to fix the code to match the spec, never the other way round.

**Frame layout (per spec):** `0xAA | LEN | 9 bytes of big-endian payload | checksum`, where payload = `sensor_id(u16) + status(u8) + temp_c_x10(i16) + timestamp_ms(u32)` = 2+1+2+4 = 9 bytes.

---

## `frame.c` — the encoder/decoder (5 bugs)

### Bug #1 — `len` was `sizeof(reading_t)` instead of `FRAME_PAYLOAD`

**What was wrong:** the length byte written into the frame was computed from `sizeof(reading_t)`, which is **12**, not the spec's **9**.

**Why:** the compiler silently inserts padding bytes into structs so each member starts at an address aligned to its own size — a `uint16_t` must start on a 2-byte boundary, a `uint32_t` on a 4-byte boundary.

Walking through `reading_t`:
```
uint16_t sensor_id;     // offset 0-1   (needs 2-byte alignment - fine)
uint8_t  status;        // offset 2     (needs 1-byte alignment - fine)
int16_t  temp_c_x10;    // wants offset 4 (2-byte aligned) -> 1 byte padding at offset 3
uint32_t timestamp_ms;  // wants offset 8 (4-byte aligned) -> 2 bytes padding at offsets 6-7
```
Total: 2 + 1 + 1(pad) + 2 + 2(pad) + 4 = **12 bytes**, matching the `offsetof()` results logged in `bughunt2_host.c` (`sensor_id=0, status=2, temp_c_x10=4, timestamp_ms=8`).

**Concept: struct padding / alignment.** A struct's `sizeof` is *not* just the sum of its members' sizes — it includes compiler-inserted padding to satisfy each member's alignment requirement, and often trailing padding so arrays of the struct stay aligned too. **You must never assume `sizeof(struct) == sum of field sizes`,** especially when the struct is about to be serialized onto a wire, written to a file, or sent over a network — none of those care about your compiler's alignment rules.

**Fix:** use the spec's own constant, `FRAME_PAYLOAD` (9), not `sizeof(reading_t)`.

---

### Bug #2 — raw pointer-cast copy instead of explicit byte-by-byte packing

**What was wrong (the removed line):** `const uint8_t *p = (const uint8_t *)r;` — reinterpreting the struct's raw memory as a byte array and copying it directly onto the wire.

**Why it's broken, two separate ways at once:**
1. **Padding leaks onto the wire.** Because of Bug #1's alignment gaps, a raw memory copy includes the compiler's padding bytes (`00 00 00`) sitting in the middle of the struct — bytes the specification never asked for and the receiver has no idea how to interpret.
2. **Endianness is wrong.** The Pico and a typical laptop are both **little-endian**: multi-byte values are stored least-significant-byte first in memory. The spec requires **big-endian** (most-significant-byte first) on the wire. A raw memory copy preserves whatever order the *machine* uses internally, not the order the *protocol* demands. That's why `sensor_id = 0x1234` came out on the wire as `34 12` instead of `12 34`.

**Concept: endianness.**
| | Byte order for `0x1234` |
|---|---|
| Little-endian (x86, RP2040 memory layout) | `34 12` |
| Big-endian ("network byte order", this spec) | `12 34` |

**The fix, and the general lesson: never memcpy/cast a struct straight onto a wire, file, or network socket.** Instead, write each field out **explicitly, one byte at a time, most-significant byte first**, using shifts and masks:
```c
r->sensor_id >> 8,   // high byte first (big-endian)
r->sensor_id,        // low byte (implicit truncation to uint8_t keeps only the low 8 bits)
```
This is more typing, but it is correct on *every* compiler and *every* architecture, regardless of that platform's native endianness or padding rules — which is exactly why real-world protocol code almost never serializes structs directly.

---

### Bug #3 — loop bound written as `len` when it should track the payload array, not the frame's `len` byte blindly

**What was wrong:** originally (back when `len` was `sizeof(reading_t)` = 12, Bug #1), the payload-writing loop needed to stop at `len - 1` to avoid running past the 9 real payload bytes into the 3 padding bytes. Once Bug #1 was fixed so `len` correctly equals `FRAME_PAYLOAD` (9), the loop bound needed to change too — to plain `len`, since `p[]` is now sized to exactly the 9 real bytes with no padding to skip.

**Concept: bugs are not always independent — fixing one can require fixing how a *different* line depends on it.** This is the encoder equivalent of what the README calls out in Step 2 for the decoder's symptoms: several observed problems can share one root cause, and fixing the root cause changes what the "correct" value elsewhere needs to be. Always re-check downstream logic (loop bounds, sizes, indices) after changing an upstream constant it was implicitly built around.

**Fix:** `for (uint8_t i = 0; i < len; i++)` with `p[]` sized `FRAME_PAYLOAD` and `len == FRAME_PAYLOAD` — the loop, the array, and the length byte are all now consistent with the 9-byte spec.

---

### Bug #4 — left-shifting a small unsigned type triggers *integer promotion* and undefined behaviour

**Where:** `frame_decode`, reconstructing `timestamp_ms` from four `payload[]` bytes: `payload[5] << 24`.

**What was wrong:** `payload[i]` is `uint8_t`. Before any arithmetic or shift operator runs on an operand smaller than `int`, C **promotes** it to `int` first ("integer promotion" / "the usual arithmetic conversions"). On a typical platform, `int` is 32-bit and **signed**. So `payload[5] << 24` doesn't shift a `uint8_t` — it shifts a *promoted, signed* `int`. If `payload[5]` has its top bit set (e.g. `0x80`), after promotion to `int` it's still the small positive value `128`, but shifting it left by 24 places produces `0x80000000`, which as a **signed 32-bit `int` is out of representable range** (the sign bit gets set as a side effect of the shift, not because you intended a negative number) — this is exactly the undefined behaviour the sanitiser flags:
```
frame.c:48:36: runtime error: left shift of 128 by 24 places
               cannot be represented in type 'int'
```

**Why this is nasty:** the code can produce the *correct-looking* answer today, on this compiler, at this optimisation level — undefined behaviour is licence for the compiler to do anything, including "happen to work," which is precisely why `-fsanitize=undefined` exists: to catch operations that are UB even when they don't visibly misbehave yet.

**Concept: integer promotion.** Any operand narrower than `int` (e.g. `uint8_t`, `int8_t`, `uint16_t`, `int16_t`) is promoted to `int` (or `unsigned int` if `int` can't hold all its values) before most operators run — including `<<`. If you then shift it far enough that the result can't fit in a *signed* `int`, you have undefined behaviour, even though every individual value involved "fits" in the types you wrote in your source.

**Fix:** cast to the *destination* unsigned type **before** shifting, not after:
```c
((uint32_t)payload[5] << 24) | ((uint32_t)payload[6] << 16)
| ((uint32_t)payload[7] <<  8) |  (uint32_t)payload[8];
```
Casting first makes the shift operate on an *unsigned 32-bit* value from the start, so there's no signed-overflow UB regardless of which bit ends up set. The comment in the code notes that `payload[6] << 16` and `payload[7] << 8` happen not to hit the sign bit on a 32-bit `int`, but casting *all four* bytes to `uint32_t` up front is the portable habit — it's also safe if this code is ever ported to an architecture where `int` is narrower than 32 bits.

---

### Bug #5 — decoder trusted an attacker-controlled length byte (buffer overflow)

**Where:** `frame_decode`, `len = in[1];` followed by a loop copying `len` bytes into a fixed-size local array.

**What was wrong:** `len` comes directly from the incoming byte stream — it could be anything from `0` to `255`, including values from corruption, noise, or a malicious sender. Nothing checked it against the actual size of `payload[]` before using it as a loop bound:
```c
len = in[1];                              /* attacker-controlled */
for (uint8_t i = 0; i < len; i++)
    payload[i] = in[2 + i];               /* payload[] is fixed-size! */
```
If `len` were, say, `200`, the loop would write 200 bytes into a local array sized for far fewer — overrunning it into adjacent stack memory, which on a real system could include saved registers or the **return address**, a classic stack buffer overflow.

**Concept: never trust a length (or any size-like value) that came from outside your program**, whether that's a network packet, a file, or user input. Validate it *before* using it as a loop bound, array index, or allocation size — check it against both the buffer's actual capacity and, where relevant, how many bytes you actually received.

**Fix:** validate `len` against the spec before touching the payload buffer:
```c
if (len != FRAME_PAYLOAD || in_len != len + 3u) return false;
```
This rejects any frame whose claimed length doesn't match the fixed spec size, and cross-checks it against how many bytes actually arrived — closing off the overflow entirely rather than just making the local buffer bigger (which only raises the bar for the same class of bug).

---

## `bughunt2_pico.c` — the physical link (2 bugs, hardware-only)

These can't be found by the laptop harness — they only show up once two real Picos talk over UART, because they depend on how bytes travel across the wire, not just on what the encode/decode functions compute from a byte array already in memory.

### Bug #6 — UART's CR/LF translation convenience corrupting binary data

**Where:** `link_init()`, `uart_set_translate_crlf(...)`.

**What was wrong:** the Pico SDK's UART driver has an optional convenience feature that watches every outgoing byte for a line-feed (`\n`, `0x0A`) and automatically inserts a carriage-return (`\r`, `0x0D`) in front of it — a nicety for **human-readable text output** on a serial terminal, where "line ending" usually means CRLF. That feature was left enabled while sending a binary protocol frame.

The frame's payload happens to contain the byte `0x0A` inside `timestamp_ms` (`0x0A0B0C0D`). The translator sees that byte, treats it as a text line-feed, and silently inserts an extra `0x0D` right before it:
```
tx  AA 09 12 34 01 00 FD 0A 0B 0C 0D 72
rx  AA 09 12 34 01 00 FD 0D 0A 0B 0C 0D     <- extra 0x0D, everything after is shifted
```
The receiver now sees a 12-byte frame that's actually the first 11 real bytes plus one injected byte, with the true final bytes cut off/shifted — the checksum and header no longer line up, and the frame is rejected.

**Concept: a helper that's correct for the use case it was designed for (human-readable text) can silently corrupt a different use case it's applied to (raw binary data).** This is an extremely common real-world firmware bug class: transport-layer conveniences (text translation, buffering assumptions, encoding conversions) that assume "this stream is text" quietly break the moment you push arbitrary bytes through the same path. The general defence is: **know what convenience features your driver/library enables by default, and turn them off for anything that isn't actually text.** (The SDK even ships a `_raw` variant of the relevant UART function for exactly this reason.)

**Fix:** `uart_set_translate_crlf(LINK_UART, false);` — transmit the exact bytes given, with no interception.

---

### Bug #7 — receiver never verifies the start-of-frame byte, so one dropped/extra byte desyncs it forever

**Where:** `rx_poll()`, the byte-accumulation loop.

**What was wrong (before the fix shown in the file):** the receiver blindly appended every incoming byte into `buf[]` and declared a frame "complete" purely by *counting* — once it had accumulated 12 bytes, it treated them as a frame, without ever checking that `buf[0]` was actually `0xAA` (the real start-of-frame marker).

**Why that's catastrophic:** if a single byte is ever dropped, duplicated, or corrupted on the wire — which happens on any real serial link eventually — the receiver's "window" of 12 bytes silently shifts by one relative to the real frame boundaries. From that point on, **every subsequent 12-byte group is a mixture of the tail of one real frame and the head of the next**, forever, even though nothing else is wrong with the link. There is no mechanism to notice the misalignment and recover.

**Concept: a length-only (or count-only) framing scheme has no way to detect or recover from desynchronisation.** A robust receiver needs to actively *hunt* for the frame's start marker rather than assume "the next N bytes I see must be a frame." The fix applied here is the first step toward that:
```c
if (n == 1 && buf[0] != FRAME_SOF) {
    n = 0;   /* not a real SOF - drop it and keep hunting */
}
```
This at least discards a bad first byte and starts looking again, rather than accepting anything as a valid frame prefix.

**The fully correct design (sketched in the file's commented-out "ideal answer"):** a small state machine —
> *hunt for `0xAA`* → *read the length byte and confirm it matches the spec* → *read exactly that many payload bytes* → *verify the checksum* → back to hunting.

At *any* step, if something doesn't match, the machine goes back to hunting for the next `0xAA` instead of blindly continuing to accumulate bytes. That design also adds an inter-byte timeout so a stalled, incomplete frame doesn't linger in the buffer forever waiting for bytes that will never come. This exact state machine is what gets built properly in Bug Hunt #3 — this bug hunt only requires recognising *why* a naive byte-counter can't recover from a single bad byte.

---

## Recurring themes across all 7 bugs

1. **Never serialize a struct directly onto a wire, file, or network socket.** Padding and native endianness are both invisible in your source code but very visible the moment raw memory crosses a machine boundary. Always pack/unpack fields explicitly, one byte at a time, in the order the *protocol* (not the *compiler*) demands.
2. **`sizeof(struct)` is not "sum of the fields."** Alignment padding can silently inflate it — verify with `offsetof()` whenever a struct's exact byte layout matters.
3. **Integer promotion is a frequent, underappreciated source of undefined behaviour.** Any shift or arithmetic on a type narrower than `int` gets promoted first; cast explicitly to the *unsigned* destination type **before** shifting when you're building up a wider value from narrow pieces.
4. **Never trust a size/length value that came from outside your program** (network, file, user input) — validate it against your actual buffer capacity before using it as a loop bound or index. This is a real, exploitable class of vulnerability (stack buffer overflow), not just a correctness nitpick.
5. **Convenience features in drivers/libraries often assume "this is text."** Know what your UART/serial driver does by default (line-ending translation, buffering, encoding) and disable anything that isn't appropriate for raw binary data.
6. **A framing protocol needs a way to detect and recover from desynchronisation**, not just a way to detect a single corrupted frame. Counting bytes without validating the start marker means one bad byte corrupts every frame after it, forever.
7. **Tools beat guessing.** `-Wall -Wextra`, hex-dumping both sides of a wire and diffing by eye, `offsetof()`, and `-fsanitize=undefined` each caught a different class of these bugs — some (like Bug #4's UB) are close to undiscoverable by reading the source alone.
8. **`char` signedness is platform-dependent** (unsigned on `arm-none-eabi-gcc` for the Pico, signed on typical x86 GCC). Comparing a received byte stored in `char` against a fixed value like `0xAA` can differ between your laptop and the target hardware from *identical source* — always use `uint8_t` for raw bytes, never bare `char`.
