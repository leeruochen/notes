# BUG HUNT #1 — Study Notes (bit manipulation, INF2004 Lab 1)

Five defects were planted and fixed in `bughunt1.c`. Below is each one broken down: what was wrong, why it broke, and the underlying C/embedded concept it's testing.

---

## Bug 1 — Missing `#include <stdint.h>`

**Where:** top of file, before `count_bits`.

**What was wrong:** the file uses `uint8_t` and `uint32_t` everywhere but never included the header that defines them.

**Why it matters:** `uint8_t`, `uint32_t`, etc. are not built-in C keywords. They're typedefs defined in `<stdint.h>` (part of the C99 standard library). Without the include, the compiler has no idea what `uint32_t` means and you get an "unknown type name" error at compile time — this is a build-time failure, not a runtime bug.

**Concept:** fixed-width integer types. `int` and `long` have sizes that vary by platform and compiler. On embedded targets like the Pico, you almost always want to be explicit about width (`uint32_t` = exactly 32 bits, unsigned) rather than relying on `int`, especially when you're doing bitwise work where the exact bit width determines correctness.

**Fix:** add `#include <stdint.h>`.

---

## Bug 2 — Missing semicolon

**Where:** `uint8_t count = 0` inside `count_bits`.

**What was wrong:** no terminating `;` after the declaration.

**Why it matters:** in C, every statement needs a semicolon. This is a pure syntax error — the compiler will refuse to build and point you at the line (though sometimes the error appears on the *next* line, since the parser doesn't know the statement ended until it sees something unexpected).

**Concept:** not a logic bug at all — just a reminder that C is unforgiving about syntax, and semicolon-related errors are usually reported one token or one line later than where the missing semicolon actually is.

**Fix:** `uint8_t count = 0;`

---

## Bug 3 — Missing cast to `uint32_t` before shifting (infinite loop on negative input)

**Where:** `count_bits(int value)` — the line that turns `value` into `v` before the counting loop.

**What was wrong:** the loop does `v >>= 1` until `v` becomes `0`. If `v` is a **signed** negative `int`, right-shifting it is *implementation-defined* behaviour in C, but on virtually every real compiler/CPU (including the Pico's ARM core) it performs an **arithmetic shift**: the sign bit (1 for negative numbers) gets copied back into the new leftmost bit instead of being replaced with a `0`.

**Why it matters — walking through `count_bits(-1)`:**
- `-1` in 32-bit two's complement is `0xFFFFFFFF` — every bit is already 1.
- With a *signed* right shift, each `v >>= 1` copies the sign bit (1) into bit 31 again.
- So `v` stays `0xFFFFFFFF` forever. The `while (v)` loop never sees `v == 0` and **spins forever** — an infinite loop, not a crash. On the Pico this would hang the board; on a laptop it just pegs a CPU core until you kill the process.

**The fix:** `uint32_t v = (uint32_t)value;` converts the *bit pattern* of `value` into an unsigned type before any shifting happens. Right-shifting an **unsigned** integer is always a *logical* shift — the vacated top bit is always filled with `0`, guaranteeing that after enough shifts `v` reaches `0` and the loop terminates.

**Concept:** signed vs. unsigned right shift.
| Shift type | Applies to | What fills the vacated top bit |
|---|---|---|
| Logical shift | unsigned types | always `0` |
| Arithmetic shift | signed types | copy of the sign bit (0 or 1) |

Rule of thumb: **never right-shift a signed value if you're relying on the result to eventually hit zero, or if you're treating the bits as a raw pattern rather than a number.** Cast to unsigned first.

---

## Bug 4 — Missing parentheses around `&` (operator precedence)

**Where:** `pin_is_clear`, originally `return (mask & 1u << pin == 0);`

**What was wrong:** it looks like it checks "is `(mask & (1u << pin))` equal to zero", but C's operator precedence doesn't group it that way.

**Why it matters — precedence order here (highest to lowest):**
1. `<<` (shift)
2. `==` (equality)
3. `&` (bitwise AND)

So `mask & 1u << pin == 0` is actually parsed as:
```
mask & ( (1u << pin) == 0 )
```
`1u << pin` is never `0` for any valid pin (it's always a power of two), so `(1u << pin) == 0` is always `false` (i.e. `0`). That collapses the whole expression to `mask & 0`, which is always `0` — meaning the function would (almost) always return "clear" regardless of the actual bit, which is silently wrong rather than a crash.

**Concept:** in C, **bitwise operators (`&`, `|`, `^`) bind more loosely than comparison operators (`==`, `!=`, `<`, `>`)** — the opposite of what most people intuitively expect coming from math notation. This is one of the most common real-world C bugs (famously flagged by `-Wall`/`-Wparentheses`). The safe habit is: **always parenthesize bitwise-AND/OR expressions when they're mixed with comparisons**, even when you're sure of the precedence — it documents intent and prevents this exact class of bug.

**Fix:** `return (mask & (1u << pin)) == 0;`

**Worked example (from the file's own comments):**
`LED_MASK = 0x24 = 0b0010_0100`
- `pin_is_clear(0x24, 2)`: `1u << 2 = 0b0100`. `0x24 & 0b0100 = 0b0100` (non-zero) → bit 2 is *set* → function returns `false`.
- `pin_is_clear(0x24, 3)`: `1u << 3 = 0b1000`. `0x24 & 0b1000 = 0b0000` → bit 3 is *clear* → function returns `true`.

---

## Bug 5 — Off-by-one loop bound (`i <= 32` instead of `i < 32`)

**Where:** `reverse_bits`, the `for` loop.

**What was wrong:** the loop ran `for (int i = 0; i <= 32; i++)`, so `i` took the value `32` on its last iteration.

**Why it matters:** a 32-bit word only has valid bit positions `0` through `31`. When `i == 32`:
- `v >> 32` — shifting a 32-bit value by an amount **equal to or greater than its own bit width is undefined behaviour** in C. It is *not* guaranteed to produce `0`; on some compilers/architectures the shift amount gets silently taken mod 32, so `v >> 32` could actually behave like `v >> 0` (i.e. return `v` unchanged) instead of `0`.
- `1 << (31 - i)` becomes `1 << -1` — shifting by a **negative** amount is *also* undefined behaviour.

**Why it's dangerous rather than obviously broken:** this is a textbook case of undefined behaviour that can appear to "work" on one compiler/platform (e.g. pass silently on a laptop with GCC/x86) and fail or produce garbage on another (e.g. the Pico's ARM Cortex-M0+). That's exactly why the comment calls it out: code that "runs fine on my laptop" isn't proof of correctness in embedded C, because UB is allowed to do *anything*, including "appear to work by accident."

**Fix:** `for (int i = 0; i < 32; i++)` — bit positions `0..31` inclusive, 32 iterations total.

**Concept:** always double-check loop bounds against the actual valid index range (`0` to `width-1`, not `0` to `width`), and treat "shift amount ≥ operand width" and "negative shift amount" as always-undefined in C — don't rely on what one compiler happens to do with them.

---

## Bonus — `swap_nibbles` (flagged for review, but not actually broken)

The comment says "this one may or may not be broken — check it," which is really an exercise in **verifying code instead of assuming a bug exists just because it's under suspicion.**

```c
return ((v & 0x0F0F0F0Fu) << 4) | ((v & 0xF0F0F0F0u) >> 4);
```

Walking it through with `v = 0x12345678`:
- `v & 0x0F0F0F0F` keeps only the **low nibble** of each byte → `0x02040608`. Shifted left 4 → `0x20406080`.
- `v & 0xF0F0F0F0` keeps only the **high nibble** of each byte → `0x10305070`. Shifted right 4 → `0x01030507`.
- OR'd together → `0x21436587`, matching the expected output exactly.

**Takeaway:** part of debugging is confirming correct code *is* correct, not just hunting for problems everywhere. Bit-mask-and-shift pairs like this are a very common, reusable pattern for swapping/reordering fixed-size groups of bits (nibbles, bytes, etc.) — worth remembering as a template.

---

## Recurring themes across all 5 bugs

1. **Signed vs. unsigned matters a lot in bit manipulation.** Right shifts, comparisons, and loop bounds all behave differently depending on signedness — always be deliberate about which one you're using.
2. **C's operator precedence is not "math order."** Bitwise operators sit below comparisons and arithmetic in precedence — parenthesize explicitly whenever you mix them.
3. **Undefined behaviour is not "guaranteed to crash."** It can look like it works — until you change compiler, optimization level, or hardware. Off-by-one loop bounds around a type's bit width are a classic source of this.
4. **Embedded targets (like the Pico) will expose bugs your laptop won't.** Different compiler (arm-none-eabi-gcc vs. gcc/clang), different architecture, different handling of undefined behaviour — never treat "passes on my laptop" as proof of correctness for firmware.
5. **`-Wall -Wextra` is your friend.** Several of these (missing semicolon, `&`/`==` precedence) are things GCC will flag directly if warnings are enabled — get in the habit of building with them on and reading every warning.
