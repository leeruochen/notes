# BUG HUNT #3 — Study Notes (IR wheel-encoder ISR driver, INF2004 Lab 3)

This hunt is different from #1 and #2: most of these bugs don't show up as wrong output — they show up as **timing-dependent behaviour that changes depending on build flags, instrumentation, or luck**. Eight defects are planted: 4 logical, 3 "Heisenbugs" (bugs that change or vanish when you try to observe them), and 1 undefined-behaviour-adjacent issue. The source comments explicitly number seven of them (`bug 1` through `bug 7`, with `bug 7` touching two lines); the eighth is called out in the README's hint table ("the final report in `main`") rather than with an inline `bug N` comment — it's flagged below where it belongs.

---

## Task A — Debug works, Release hangs forever (the `volatile` bug)

**Where:** every variable written inside `encoder_isr` and read inside `main` — `pulse_count`, `last_edge_us`, `last_debounce_us`, `slot_start_us`, `slot_width_us`, and `state` — is declared `volatile`. This is **Bug #1**, and it's the one you're told to find with a disassembler before touching source.

**What was wrong (before `volatile` was added):** the wait loop is
```c
while (pulse_count < TARGET_SLOTS)
    tight_loop_contents();
```
Without `volatile`, the compiler is allowed to assume that **nothing inside this loop's visible code path can change `pulse_count`** — because, as far as the compiler can see, nothing does. `encoder_isr` is a separate function that the compiler has no reason to believe runs concurrently with `main`; the standard C abstract machine has no concept of "an interrupt can fire between any two instructions."

At `-O0` (Debug), the compiler doesn't optimise aggressively enough to notice this, so it reloads `pulse_count` from memory every pass — the loop happens to work by accident. At `-O3` (Release), the compiler performs the optimisation the standard allows it to: it loads `pulse_count` into a register **once**, and since the loop body doesn't change it, concludes the condition's truth value can never change during the loop. Having "proven" that, if the condition is false on entry it reasoned it will be false forever, and collapsed the whole loop into:
```
100003e4:  b.n   100003e4     <- branch to itself, forever
```
Two bytes of machine code that can never terminate under any circumstances — not a bug in the CPU or the wiring, a **correct optimisation based on a false premise**, because nothing told the compiler an interrupt exists.

**Fix:** mark every ISR-shared variable `volatile`. This is a contract with the compiler: *"this value can change at any time, for reasons outside this code; reload it from memory on every access, and don't reorder or eliminate accesses to it."* With `volatile` in place, `-O3` is forced to re-read `pulse_count` from memory each pass, exactly like `-O0` did by accident.

**Concept — what `volatile` actually promises, and what it doesn't:** `volatile` disables a specific class of compiler optimisation (caching a value in a register, reordering accesses, eliminating "dead" reads/writes). **It says nothing about atomicity or ordering with respect to other variables.** This distinction matters for two of the other bugs below — see Bug #1b and Bug #8.

---

## Bug #1b — the second "shared state" defect: a 64-bit variable is not atomic on a 32-bit core

The README's hint table lists **two** defects under "shared state between ISR and `main`," both Heisenbugs. Bug #1 (missing `volatile`) is one. The second is `last_edge_us`, declared `volatile uint64_t`.

**Why this matters:** the Cortex-M0+ inside the RP2040 is a **32-bit machine** — every load/store instruction moves at most 32 bits. Reading or writing a `uint64_t` therefore takes **two separate 32-bit bus transactions** (low word, then high word), not one atomic operation. If the ISR fires and updates `last_edge_us` in between those two transactions on the `main`-side read, you get a **torn read**: half of the old value stitched to half of the new one, producing a number that was never actually the timer's value at any instant.

**`volatile` does not fix this.** `volatile` only forces the compiler to actually perform the reads/writes and not optimise them away — it says nothing about whether an interrupt can land in the middle of the two 32-bit halves. That's exactly the distinction the README pushes you to articulate: *what `volatile` guarantees* (visibility/no-elision) versus *what atomicity guarantees* (an operation completes as one indivisible step, or not at all).

> **Direct connection to Lecture 3, §31 ("16-bit reads on 8-bit buses: Torn Reads"):** the lecture's worked example is a 16-bit timer read split across two 8-bit bus transactions, where a rollover mid-read stitches together a stale high byte with a fresh low byte (`0x00FF` read as `0x01FF` instead of the true `0x0100`). This is the **same mechanism, one level up**: a 64-bit value split across two 32-bit transactions on a 32-bit core. The lecture's fix generalises directly: **briefly disable interrupts around the read, or use the hardware's atomic-read support if available**, rather than trusting a plain multi-word `volatile` read to be safe.

In this specific file, the code gets away with it in practice only because `finished_at = last_edge_us;` happens **after** the interrupt has already been disabled (see Bug #2 below) — but that's a fragile accident of ordering, not a real fix. If you ever read a multi-word ISR-shared variable while the ISR is still live, protect it explicitly (disable interrupts around the read, or keep a hardware-atomic 32-bit "sequence" pattern instead of a bare 64-bit value).

---

## Bug #2 — the ISR was never disabled once the target was reached

**Where:** right after the wait loop, before the results are printed:
```c
gpio_set_irq_enabled_with_callback(ENCODER_PIN, GPIO_IRQ_EDGE_FALL|GPIO_IRQ_EDGE_RISE, false, &encoder_isr);
```

**What was wrong without it:** the physical wheel doesn't know or care that `main` has finished waiting. If the interrupt is left enabled, `encoder_isr` keeps firing, keeps incrementing `pulse_count`, and keeps overwriting `slot_width_us`, `state`, `slot_start_us`, and `last_edge_us` — **while `main` is trying to read those same variables to print the final report.** This is the second "shared state" Heisenbug: whether the printed report reflects "slot 20" or some later slot depends entirely on how much the wheel has turned in the tiny window between the loop exiting and the printf calls running — a race whose outcome varies run to run.

**Concept:** an interrupt source that's still enabled is still a writer. "I've read what I needed from the shared variables" is not the same as "nothing can write to them anymore" — if you need a stable snapshot, you must either disable the interrupt source, or copy the values out atomically before doing anything with them. This is the same principle as the lecture's timer configuration traps (§32): failing to properly stop/acknowledge a hardware event source has knock-on effects on code that assumes it has finished.

---

## Bug #3 — debounce comparison used addition instead of subtraction (unsigned wraparound)

**Where:** `encoder_isr`, the debounce gate: `if (now - last_debounce_us > DEBOUNCE_US)` (originally written as `if (now > last_debounce_us + DEBOUNCE_US)`).

**Why they look equivalent but aren't:** `time_us_32()` is a 32-bit counter that **wraps around to 0** after `2^32` microseconds (about 71 minutes) of uptime. Try the README's worked numbers by hand in 32-bit unsigned arithmetic: `last = 0xFFFFF000`, `DEBOUNCE = 50000`, `now = 0x00001000` (i.e. the timer just wrapped past zero, only a few thousand microseconds after `last`, well inside the debounce window and should be rejected).

- **Addition form:** `last + DEBOUNCE = 0xFFFFF000 + 50000`, which **overflows 32 bits** and wraps to a small number. Now `now > (last+DEBOUNCE)` can spuriously evaluate `true` right when it should be `false` — the debounce gate opens exactly when it shouldn't, near every wraparound.
- **Subtraction form:** `now - last_debounce_us = 0x00001000 - 0xFFFFF000`, which **also wraps**, but unsigned wraparound subtraction correctly computes the true elapsed time modulo `2^32` — as long as the real elapsed interval is less than half the counter's range, `now - last` always gives the right answer even across a wraparound boundary, and `> DEBOUNCE_US` correctly stays `false` here.

**Concept: unsigned subtraction is wraparound-safe for computing elapsed time; addition-then-comparison is not.** This is a very common embedded pattern (`if ((uint32_t)(now - last) > threshold)`), and the "add the threshold and compare" version is a classic trap that only breaks near the counter's rollover point — which is why the README's technique of *seeding* `last_debounce_us` near `0xFFFFF000` at startup is so useful: it turns a bug that would otherwise take 71 minutes to reproduce into one you can trigger in seconds.

---

## Bug #4 — `DEBOUNCE_US` was tuned for a button, not a wheel encoder

**Where:** `#define DEBOUNCE_US 500u` (originally `50000u`).

**What was wrong:** 50,000 µs = 50 ms is a perfectly reasonable debounce time for a **mechanical button press**, where a human finger and a springy switch bounce for a few milliseconds. It is wildly too long for an **optical wheel encoder** spinning at any real speed — the debounce window would swallow legitimate, fast, back-to-back slot edges as if they were noise, undercounting slots or stalling entirely. Dropping it to 500 µs (0.5 ms) is appropriate for the electrical noise characteristics of the HC020K's phototransistor output while still passing genuine edges through.

**Concept:** a debounce constant isn't a universal safety margin — it has to be sized to the **physical source of noise you're actually filtering** and checked against the fastest legitimate signal you expect. The README's follow-up question makes this concrete: *"`DEBOUNCE_US` is 50000. What wheel speed makes a real slot shorter than that? Above that speed, what does the driver report — an error, or a plausible-looking wrong number?"* A debounce window that's too long doesn't fail loudly; it silently reports a slower, wrong speed that still looks physically plausible, which is far more dangerous than an obvious crash.

---

## Bug #5 — missing `break` in the `switch`, causing fallthrough

**Where:** `encoder_isr`, the `ENC_IDLE` case:
```c
case ENC_IDLE:
    slot_start_us = now;
    state = ENC_SLOT;
    break;      // <- was missing
```

**What was wrong:** without the `break`, execution falls straight through into `case ENC_SLOT:` on the same call — so a single edge would set `slot_start_us = now`, immediately fall into computing `slot_width_us = now - slot_start_us` (which is `0`, since `slot_start_us` was just set to `now` one line above), increment `pulse_count`, and reset back to `ENC_IDLE` — all in one ISR invocation, from a single edge.

**Why it matters:** the whole point of the two-state design (`ENC_IDLE` ↔ `ENC_SLOT`) is that a **full slot measurement requires two separate edges**: one to mark the start, a later one to mark the end and compute the width. Fallthrough collapses that into "every single edge completes a full, zero-width slot," corrupting both the pulse count (counting at twice the real rate, since it no longer waits for the second edge) and the width measurement (always reporting `0`).

**Concept:** `switch`/`case` in C falls through by default unless you explicitly `break` — this is one of the most common real-world C defects, and it's silent: the code compiles cleanly and *runs*, it just executes logic you didn't intend. Always treat a `case` without a `break`, `return`, or an explicit `/* fallthrough */` comment as suspicious.

---

## Bug #6 — `printf()` inside the ISR

**Where:** the (now commented-out) trailing `printf` in `encoder_isr`.

**Why it was there and why it's wrong:** printing gives you visibility into every edge as it happens — tempting when you're debugging. But `printf` over UART at 115200 baud takes on the order of **milliseconds** to complete, and an ISR is expected to run in **microseconds**. While the ISR is blocked doing I/O, it cannot respond to the *next* real edge — at any realistic wheel speed, edges can easily arrive faster than a `printf` call completes, meaning genuine slot transitions get missed entirely while the CPU is busy formatting text.

**Concept — the Heisenbug, and why the README makes you remove it as an experiment rather than just delete it and move on:** *"Remove the `printf` — do not fix anything else — and rerun. Does a different symptom appear or get worse?"* If it does, that's not a coincidence: the `printf`'s own delay was **incidentally throttling how often the ISR could re-fire**, which was masking a *different* timing-sensitive bug (e.g. Bug #3's overflow window, or how fast `pulse_count` was allowed to increment). Adding an instrument (a print, a breakpoint, a debugger) can change the very timing that produces a bug — this is why "I added a print statement and it went away" is not the same as "I fixed it." The lecture material makes the same point from the ADC side (§50): *"Keep the sampling path deterministic and move expensive work elsewhere"* — an ISR (or any tight interrupt/sampling path) should never do slow, variable-latency work like formatted I/O; log raw values into a buffer and drain/print them from `main` instead, exactly as the README's ring-buffer suggestion shows.

---

## Bug #7 — interrupt registered for only one edge type

**Where:** `gpio_set_irq_enabled_with_callback(ENCODER_PIN, GPIO_IRQ_EDGE_FALL|GPIO_IRQ_EDGE_RISE, true, &encoder_isr);` (originally `GPIO_IRQ_EDGE_FALL` alone) — and the same fix has to be mirrored at the **disable** call at the end of `main`, since disabling only the falling-edge interrupt while the rising-edge one is still separately enabled would leave the ISR partially armed (this is why the comment marks it "similar to above" at the second occurrence).

**Why it matters:** the two-state machine (`ENC_IDLE` ↔ `ENC_SLOT`) is designed around seeing **both** transitions of the encoder's optical signal — one edge to mark "slot started," the next (opposite) edge to mark "slot ended" and compute its width. If the ISR is only ever told about falling edges, it only ever sees half of the physical transitions the sensor actually produces, so the state machine's assumption ("the next edge I see is the *other* one") stops matching reality. The fix has to enable **both** `GPIO_IRQ_EDGE_FALL` and `GPIO_IRQ_EDGE_RISE`, and — just as importantly — turn **both** back off symmetrically when the interrupt is disabled at the end.

**Concept:** when you configure a hardware event source, check that the *set of events you're told about* actually matches the *set of events your logic assumes it will see*. A state machine with two states implicitly assumes something about how many, and which, event types drive its transitions — mismatches here are a common, easy-to-miss class of interrupt configuration bug, and the lecture's §26 (Input Capture) is the more "hands-off" hardware alternative: dedicated input-capture peripherals timestamp edges directly in hardware "before the referee has a chance to react," sidestepping this whole class of software edge-configuration mistake — though the RP2040's GPIO IRQ + `time_us_32()` approach used here is the software equivalent, and inherits exactly this kind of configuration risk that a true hardware input-capture peripheral wouldn't have.

---

## Bug #8 — "the final report in `main`" (undefined behaviour, not marked with an inline comment)

The README's hint table lists an eighth defect, described only as living in *"the final report in `main`"* and classed as **undefined behaviour** — distinct from the two "shared state" Heisenbugs already covered above (Bugs #1 and #1b/#2). Unlike bugs #1–#7, this file does not carry an explicit `// bug 8:` comment marking a fix for it, so treat this section as a pointer to keep hunting rather than a fully-worked answer — per the README, it's "not visible at all until you ask what happens on the *second* revolution of the wheel at speed."

**What to actually investigate:** the report reads `slot_width_us`, `pulse_count`, and the 64-bit `finished_at` (copied from `last_edge_us`) only *after* the interrupt has been disabled (Bug #2's fix) — which closes off the most obvious race. The remaining risk sits with things the disable-then-read ordering doesn't fully protect against: whether the very last ISR invocation is guaranteed to have *completely* finished (including its 64-bit write to `last_edge_us`) before `gpio_set_irq_enabled_with_callback(..., false, ...)` returns, and whether reading a `volatile uint64_t` that was written by two separate 32-bit stores is safe to treat as a single value even once the interrupt source is masked. This is the same non-atomic multi-word access problem as Bug #1b, but resurfacing at the point of *consumption* rather than during ordinary concurrent operation — worth tracing through with the same "what if the ISR fires between these two instructions" question the README asks throughout, and worth pairing with Lecture 3 §31's torn-read fix (briefly disable interrupts around the read, or rely on a documented hardware-atomic access path) if your investigation confirms it.

---

## Recurring themes across all eight bugs

1. **The compiler is allowed to assume single-threaded, non-interrupted execution unless you tell it otherwise.** `volatile` is that "otherwise" for a single scalar access — without it, a correct-looking loop can be proven-optimised into an infinite one, and the proof is only wrong because the compiler wasn't told about the interrupt.
2. **`volatile` guarantees visibility, not atomicity.** A multi-word (e.g. 64-bit) value shared with an ISR on a narrower (32-bit) core can still be torn, even when every access is `volatile`. This is Lecture 3 §31's "torn read" problem, scaled from an 8-bit-bus/16-bit-value example up to a 32-bit-core/64-bit-value one.
3. **A hardware event source that's still enabled is still a writer.** "I've read the value I need" doesn't stop an ISR from overwriting it moments later — disable the source (or otherwise guarantee no further writes) before treating shared state as a stable snapshot for a final report.
4. **Unsigned subtraction is the safe way to measure elapsed time against a wrapping counter; addition-then-compare is not**, and this only manifests near the wraparound point — which is exactly why deliberately *seeding* a counter near its wrap value to force early reproduction (rather than waiting 71 minutes) is such a generally useful debugging technique.
5. **A constant tuned for one physical situation (a bouncy button) can silently misbehave in a different one (a fast optical sensor)** — always check a timing constant against the fastest legitimate signal it will actually see.
6. **`switch` falls through unless you `break`.** Silent, common, and it changes *logic*, not syntax — the code still compiles and runs.
7. **ISRs must be fast and must not do I/O.** Adding an instrument (a `printf`, a debugger, a breakpoint) can change the very timing that produces a bug, which is why "it went away after I added a print statement" is not evidence of a fix — log raw values to a buffer and process them outside the ISR instead (this is the same golden rule the lecture states for ADC FIFO draining: keep the time-critical path deterministic, move expensive work elsewhere).
8. **The set of hardware events you register for must match what your logic assumes it will see** — and remember to keep enable and disable calls symmetric, or you can leave part of an interrupt still armed.
9. **Two builds of *identical* source can behave completely differently** once undefined/unspecified behaviour (races, unsynchronised shared state, aggressive optimisation reasoning) is involved. "It works on my build" was never proof of correctness in Bug Hunts #1 and #2 either — this hunt just makes the point unavoidable by having Debug and Release visibly diverge.

## Where this connects most directly to Lecture 3

| Bughunt #3 concept | Lecture 3 section |
|---|---|
| Torn read of `last_edge_us` (64-bit value on a 32-bit core) | §31, "16-bit reads on 8-bit buses: Torn Reads" — same mechanism, different bit widths |
| ISR must be fast; no `printf`/blocking I/O inside it | §21 (blocking delays freeze the whole system) and §50 (keep the sampling path deterministic, move expensive work elsewhere) |
| `encoder_isr` timestamping edges in software via `time_us_32()` | §26, Input Capture — the hardware alternative that timestamps edges immune to ISR latency, which this software-only driver doesn't have access to |
| The Pico's `time_us_32()`/`time_us_64()` ticking in clean microseconds | §22, the Prescaler — the SDK has already divided the 125 MHz system clock down to a 1 MHz (1 µs) tick for you, which is why debounce math in this file can be done directly in microseconds |
| Leaving the encoder interrupt enabled after `main` stops waiting for it | §32.2, "The Infinite ISR" — a related family of bug: forgetting to properly stop/acknowledge a hardware interrupt source has knock-on effects on code that assumes it's done |
