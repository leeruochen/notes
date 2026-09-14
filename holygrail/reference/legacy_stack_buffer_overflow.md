# Classic 32-bit Stack Buffer Overflow — LEGACY / REFERENCE ONLY

> [!important] Scope — this is NOT current core OSCP
> Current PEN-200 states plainly: *"writing buffer overflow exploits is beyond the scope of the course"* (Fixing Exploits module). The exam-relevant skill today is **understanding and *adapting/fixing* an existing memory-corruption exploit** (see [Fixing Exploits] in `sources/official/pen200/14-fixing-exploits.md`), not developing a BOF from scratch. This note is kept for **understanding and legacy practice only** and is deliberately **omitted from the primary phase checklists** ([[00_methodology]] does not route here).
>
> Use this only if a legacy exercise or clone explicitly requires from-scratch BOF development. Otherwise, when you meet a memory-corruption PoC, go to [[public_exploit_research]] + the course's Fixing-Exploits method (adjust offsets/return address/badchars/payload to *your* target).

## Goal
On an **authorised local clone** of the target service, reproduce the crash, gain reliable EIP control, and land a working payload — one verified stage at a time.

> [!warning] No exercise-specific constants belong here
> This note contains **no** fixed offsets, bad-character lists, return addresses, final shellcode, target protocol names, or copied exploit scripts — those are per-target and per-exercise. It is the *method* only.

---

# Stage 0 — Set up a safe repro (prerequisite)
- Run the vulnerable service on a **local clone / lab VM** you own — never fuzz a live exam service you can't restart.
- Attach a debugger; keep a way to restart the service between crashes.
- **Success signal:** you can crash the service and restart it at will.

# Stage 1 — Reproduce & control the crash
- Send progressively larger input; confirm the service crashes and the debugger shows the fault.
- Note the offending field/command and the **protocol prefix/suffix** the input must keep (many services need a specific command wrapper for the input to reach the vulnerable buffer).
- **Success signal:** a repeatable crash tied to one input field.

# Stage 2 — Fuzz safely & find the offset
- Increase length in steps to bracket where the crash occurs (don't jump straight to huge buffers).
- Send a **cyclic (De Bruijn) pattern** of that length; read the value that lands in the instruction pointer; compute the exact offset with the pattern tool.
- **Success signal:** a single offset value that predicts where EIP is overwritten.

# Stage 3 — Confirm EIP control
- Send `offset` bytes of filler + 4 marker bytes (e.g. a recognizable non-address like `BBBB`) + trailing filler.
- The debugger must show your 4 marker bytes **exactly** in EIP.
- **Success signal:** EIP == your marker bytes. If not, recheck the offset before continuing.

# Stage 4 — Map space & find bad characters
- Determine how much space is available for shellcode (before/after the return address) and whether the stack pointer points at attacker-controlled data.
- Send a full byte array (0x01–0xFF) into the buffer; in the debugger, compare the in-memory bytes to what you sent; any byte that is **mangled/truncated/missing** is a bad character. Remove it and repeat until the array is clean.
- **Success signal:** a byte array that appears in memory intact (minus already-known delimiters like 0x00).

# Stage 5 — Choose a control-transfer instruction
- Inspect loaded modules for one **without protections** (no ASLR/DEP/rebase) that ships with the target — prefer a module belonging to the app itself.
- Within it, find a suitable instruction that redirects execution to your buffer (e.g. a `JMP <reg>` / `CALL <reg>` for whichever register points at your payload), whose address contains **none of your bad characters**.
- **Success signal:** a static, badchar-free address that reliably lands on a valid control-transfer instruction.

# Stage 6 — Generate architecture-appropriate shellcode
- Generate shellcode for the correct architecture, **excluding every verified bad character**, with your callback details (LHOST/LPORT) as the only variables.
- Add a short NOP-style landing pad if the technique needs alignment.
- **Success signal:** shellcode produced with no bad chars reported by the generator.

# Stage 7 — Build the final buffer in stages & retest
- Assemble: `protocol-prefix + [offset filler] + [return address] + [pad] + [shellcode] + protocol-suffix`.
- Start a listener; verify the **callback interface/routing** first ([[00_FOOTHOLD]] — a shell that never arrives is usually a wrong `LHOST`/interface, not a broken exploit).
- Fire against the clone; on success, retest for reliability.
- **Success signal:** a shell/command execution that reproduces across restarts.

---

# Troubleshooting & stop conditions
| Symptom | Likely cause | Action |
|---|---|---|
| service won't crash | wrong field / needs protocol wrapper | fix prefix/suffix; confirm input reaches the buffer |
| EIP not your bytes | wrong offset | recompute the cyclic-pattern offset (Stage 2/3) |
| lands then crashes/garbage | a bad character corrupted the payload | rerun the badchar comparison (Stage 4) |
| return address won't take | module rebased/ASLR/DEP, or address has a badchar | pick another module/instruction (Stage 5) |
| unstable across runs | non-static return address / alignment | choose a static module address; adjust padding |
| no shell despite "hit" | callback routing/interface/egress | verify listener + interface ([[00_FOOTHOLD]]) |

**Stop conditions:** protections (DEP/ASLR) make the classic path infeasible → this is beyond the classic-BOF method and beyond core OSCP; if you're on the exam and hit a memory-corruption PoC, **adapt an existing exploit** ([[public_exploit_research]] + Fixing-Exploits method) rather than developing one from scratch.

> [!important] Helpline
> Legacy/reference only — writing BOFs is outside current OSCP scope; adapting exploits is inside it. Work on a **clone**. One verified stage before the next: crash → offset → EIP → badchars → return address → shellcode → buffer. No fixed constants live in this note.
