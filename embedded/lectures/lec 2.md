# INF2004 Embedded Systems Programming — Lecture 2 Study Notes
## GPIO Fundamentals & Interrupt Fundamentals

---

## Learning Objectives

By the end of this lecture, you should be able to:

1. Explain how a microcontroller (MCU) bridges the analogue world and the digital world.
2. Define GPIO and describe what a pin's voltage level means to a program.
3. Explain input vs output direction, and why floating pins are dangerous.
4. Explain pull-up and pull-down resistor configurations and active-high vs active-low logic.
5. Explain push-pull outputs and calculate a current-limiting resistor for an LED.
6. Read and manipulate hardware registers using bitmasks (set, clear, toggle, replace).
7. Configure a GPIO pin in code using the Pico SDK (input and output).
8. Explain switch bounce and how debouncing works.
9. Describe Programmable I/O (PIO) and why it exists.
10. Define an interrupt and contrast polling vs interrupt-driven design, including the maths behind utilisation and deadlines.
11. Trace the full interrupt lifecycle: event → IRQ → ISR → context save/restore → resume.
12. Understand the Interrupt Vector Table (IVT), the stack, and context save/restore.
13. Write good ISR code (short, fast, defers heavy work to the main loop).
14. Understand shared-state hazards between an ISR and main code (race conditions).
15. Recognise the range of interrupt sources on a typical MCU (GPIO, timers, UART, DMA, etc.) and system-level events (software IRQs, faults, resets).

---

## PART 1: GPIO FUNDAMENTALS (W2.1)

### 1. The Analogue and Digital World

- The real world is **analogue and continuous** (sound, temperature, light are smooth, continuously varying signals).
- Computers work in a **digital and discrete** world (just 0s and 1s).
- To bridge the two:
  - **ADC (Analog-to-Digital Converter):** turns a real-world analogue signal into digital numbers the MCU can process.
  - **DAC (Digital-to-Analog Converter):** turns digital data back into an analogue signal (e.g., driving a speaker).
- Embedded systems constantly mix analogue and digital electronics to represent the same real-world signal in a form the CPU can compute with.

### 2. How a Microcontroller Talks to the World

An MCU has several ways to communicate with external hardware:

| Interface | Purpose |
|---|---|
| GPIO | Simple digital in/out — one pin, one bit |
| PWM | Simulates an analogue-like output using fast digital pulses (e.g., motor speed, LED brightness) |
| UART / I2C / SPI | Structured digital serial communication protocols (used for sensors, displays, chips) |
| ADC / DAC | Analogue in/out conversion |

### 3. What is GPIO?

**GPIO = General Purpose Input Output.**

- Each GPIO pin carries **one digital state**: 0 or 1 (roughly 0V or 3.3V on most modern MCUs, including the Pico).
- The flow of meaning:

  **External Signal (Voltage)** → **GPIO Pin (Logical level: 0 or 1)** → **Application Meaning (e.g., "is the button pressed?", "is the system ready?")**

- Key idea: **"The circuit sets the voltage. The program gives it meaning."** The pin itself has no idea what 0 or 1 "means" — that interpretation is entirely up to your code.

### 4. Direction: Input Senses, Output Drives

- **MCU as Input:** it *senses* an external circuit (e.g., reading a sensor or button).
- **MCU as Output:** it *drives* an external load (e.g., turning on an LED, controlling a motor).
- **Rule:** always choose the direction (input or output) before you try to read or drive a signal on that pin.

### 5. Fundamental Electronics: Voltage Ranges

Digital logic isn't a perfect 0V/3.3V split in practice — real chips define voltage *ranges*:

- **HIGH = 1:** voltage above a defined minimum (VIH min)
- **LOW = 0:** voltage below a defined maximum (VIL max)
- **Unknown / undefined zone:** anything between VIL(max) and VIH(min) — the chip cannot reliably interpret this as a clean 0 or 1.
- Always check the actual datasheet for real threshold values — they vary by chip and by voltage rail (e.g., 3.3V logic vs 5V logic).

### 6. Logic Levels Need a Shared Reference (Common Ground)

- When connecting two circuits or boards together, they **must share a common ground (GND)**.
- Why this matters: voltage is always relative. If your board's ground is at 0V and its max signal is 5V, but another board's ground sits at -1V, the *effective* voltage difference across the two systems could add up to 6V instead of 5V — this can damage components or corrupt readings.
- **Always match voltage compatibility, direction, and ground before connecting two circuits.**

### 7. Inputs: The Danger of an Open (Floating) Switch

- If a switch is open and nothing else is connected, the pin has **no defined voltage** — this is called a **floating pin**.
- **"Nothing connected" is NOT the same as logic 0.** A floating/undriven pin has no defined default state.
- This is dangerous because a floating pin can pick up stray static electricity or electrical noise from the environment and randomly register as HIGH or LOW — potentially triggering unintended behaviour.
- **Solution:** always give an input pin a defined resting state using a pull-up or pull-down resistor.

### 8. Pull-Up Resistors

- A **pull-up resistor** connects the pin to HIGH through a resistor by default.
- Behaviour:
  - Switch **OPEN** → pin reads **1 (HIGH)**
  - Switch **PRESSED** (connects pin to GND) → pin reads **0 (LOW)**
- This is called **active-low**: "pressed" = LOW.
- Design logic: a *weak* pull-up resistor gently holds the line HIGH, but a *strong*, direct short to GND (via the switch) easily overpowers it when pressed.

### 9. Pull-Down Resistors

- A **pull-down resistor** connects the pin to GND (LOW) through a resistor by default.
- Behaviour:
  - Switch **OPEN** → pin reads **0 (LOW)**
  - Switch **PRESSED** (connects pin to HIGH) → pin reads **1 (HIGH)**
- This is called **active-high**: "pressed" = HIGH.
- **Key takeaway: the circuit design (pull-up vs pull-down) defines the convention** — there's no universal rule that pressed = 1 or pressed = 0. You must check how the circuit is wired.

### 10. Outputs: Push-Pull Drive

- A **push-pull output** can actively drive the pin toward either supply rail:
  - **Write 1** → pin actively drives toward the supply voltage (e.g., 3.3V)
  - **Write 0** → pin actively drives toward GND
- **Important limitation:** a driven GPIO output is a **limited current source or sink**, NOT a power supply. It can only safely supply a small amount of current (often just a few tens of mA) — never try to power a motor or high-current device directly off a GPIO pin.

### 11. LEDs Need a Current-Limiting Resistor

- An LED connected directly to a GPIO pin without a resistor will likely draw too much current and burn out (or damage the pin).
- Standard circuit: **GPIO HIGH → resistor (e.g., 1kΩ) → LED (anode to cathode) → GND**
- Example current calculation:

  $$I \approx \frac{V_{supply} - V_{LED forward drop}}{R} = \frac{3.3V - 2.0V}{1k\Omega} \approx 1.3\text{ mA}$$

- This is an *illustrative* estimate — always check the actual LED's forward voltage and your MCU's specifications before choosing a resistor value.

### 12. Pin Multiplexing ("Pins with Many Hats")

- A single physical pin often has multiple possible functions: GPIO (software-controlled), UART/SPI, PWM/PIO, etc.
- A **function select** mechanism inside the chip decides which function "owns" that physical pin at any given time.
- **Rule:** select exactly one output owner per pin, and confirm the specific pin you're using actually supports the function you want (not all GPIO pins support every peripheral function).

### 13. Registers Connect Software to Pin Hardware

Internally, the MCU exposes hardware pins to software through **registers** — special memory addresses that control or reflect pin state:

- **OUT latch** → feeds the **Output Driver** → which drives the physical **PIN**.
- **IN sample** → reads the current voltage on the **PIN** back into software.
- **OE/DIR** register controls whether the pin acts as output or input.
- Note: register names, exact bit meanings, and memory addresses vary between different MCU families — this is a *generic* model.

### 14. Bitmasks: Selecting the Bits You Mean

- A **mask** is a binary pattern used to select specific bit(s) within a register, leaving the rest alone.
- Example: to work with bit 3 only:

  ```
  bit:    7 6 5 4 3 2 1 0
  MASK:   0 0 0 0 1 0 0 0

  1u << 3 = 0b00001000 = 0x08
  ```

- To **read** bit 3: `(IN & mask) != 0`

**Worked mask example:**
Starting value: `0xA5` = `1 0 1 0 0 1 0 1`
To **set bit 3** using mask `0x08` (`0 0 0 0 1 0 0 0`):

`0xA5 | 0x08 = 0xAD`

You can practice similarly: try clearing bit 2, or toggling bits 0 and 7 — always starting fresh from `0xA5`.

### 15. Changing Bits Without Replacing the Whole Port

| Intent | Expression | Effect on other bits |
|---|---|---|
| Set bit 3 | `OUT |= mask` | Preserved |
| Clear bit 3 | `OUT &= ~mask` | Preserved |
| Toggle bit 3 | `OUT ^= mask` | Preserved |
| Replace port | `OUT = mask` | Overwritten (destroys all other bits!) |

- **Memory aid:** flip a bit from 0 → 1, use **OR** (`|`). Flip a bit from 1 → 0, use **AND** with the inverted mask (`&= ~mask`).
- **Caution:** this only applies to registers designed for this kind of access. If a register can be modified by both your main code and an interrupt at the same time, you need an **atomic strategy** to avoid race conditions (see ISR Design section below).

### 16. Configuring GPIO — The General Sequence

1. **IDENTIFY** — determine the pin number and which function you need.
2. **PRELOAD** — set the pin to a safe output level *before* enabling the output driver (avoids a brief undefined glitch).
3. **ENABLE** — turn on the output driver (or for input: select function → set pull resistor → enable input).
4. **VERIFY** — read the pin back or measure it to confirm it behaves as expected.

**Golden rule:** follow the device's documented configuration sequence. Never assume a guessed reset/default state.

### 17. A Concrete Example — Pico SDK

**Configuring an output (GP15):**
```c
gpio_init(15);
gpio_put(15, 0);
gpio_set_dir(15, GPIO_OUT);
```

**Configuring an input with pull-up (GP2):**
```c
gpio_init(2);
gpio_set_dir(2, GPIO_IN);
gpio_pull_up(2);
```

- General pattern: **configure once** (at startup), then **read and write during the application's main loop**.
- **Important warning:** always check the *pinout diagram* for your exact physical board. The GPIO number in software ("GP2") is not always the same as the physical header pin number printed on the board silkscreen.

### 18. Polling — Converting Electrical Level Into Meaning

Example (active-low button controlling an LED):

| Button | Raw Input | LED Output |
|---|---|---|
| Released | 1 | 0 → off |
| Pressed | 0 | 1 → on |

```python
pressed = (button.value() == 0)
led.value(1 if pressed else 0)
```

### 19. A Held Level Differs From a New Press

- If you just keep reading "LOW" every loop iteration while a button is held down, you'll count the *same* press many times if you're not careful.
- What actually matters is the **transition**, not the level:
  - **Falling edge:** HIGH → LOW transition (e.g., button released → pressed, active-low)
  - **Rising edge:** LOW → HIGH transition (button released again)
- **Rule: count an accepted transition/edge — not every loop iteration that merely observes the held LOW level.**

### 20. Switch Bounce

- Mechanical switches don't cleanly transition from open to closed. The physical contacts bounce and vibrate for a few milliseconds, producing **many rapid electrical edges** from a single physical press.
- The application logically only wants to register **one** press event, not a burst of noisy transitions.

### 21. Debouncing

- **Debouncing** = only accepting a new state once it has remained stable for a minimum period of time.
- Basic algorithm:
  1. **SAMPLE** — read the raw pin.
  2. **CHANGED?** — if the raw reading changed since last sample, restart a stability timer.
  3. **STABLE?** — once the reading has held steady for a set duration (commonly ~30ms), accept it as a real, debounced state change.
- **Rule of thumb:**
  - Raw changes → restart the timer.
  - Stable pressed state → fire exactly one event.

### 22. Responsive Software — Keep the Loop Moving

- General polling-loop pattern: **SAMPLE → UPDATE STATE → WORK IF DUE**, then return to the top of the loop quickly.
- A **shorter loop** reduces "blind time" (the gap between checks), but it does **not guarantee** that every short pulse will be seen — a pulse shorter than your polling interval can still be missed entirely by pure polling.

### 23. GPIO as an Interface — Handshaking

For more reliable sender/receiver communication over GPIO pins:

- **SENDER** puts data on the **DATA** line.
- Sender asserts **REQ** ("data ready").
- **RECEIVER** captures the data, then asserts **ACK** ("captured").
- Rule: **hold the data steady until it has been accepted**, and clearly define signal timing, polarity (active-high/low), and a timeout in case the other side never responds.

### 24. The GPIO Model Survives a Change of CPU

| Device | CPU Core | The Same GPIO Questions Still Apply |
|---|---|---|
| RP2040 | Arm Cortex-M0+ | Input or output? |
| RP2350 | Arm Cortex-M33 | Which pull resistor and function? |
| RP2350 | Hazard3 (RISC-V) | What level, and when? |

- **Takeaway:** the exact register names/addresses change between chip families, but the underlying electrical reasoning (direction, pull resistors, voltage levels, timing) carries over.

### 25. Programmable I/O (PIO)

- **PIO** is a proprietary feature on the RP2040/RP2350 that lets you build **custom hardware interfaces from scratch**, including entirely novel protocols.
- Each PIO instance is a tiny, independent "processor" that runs **completely separately** from the main Cortex-M CPU core.
- The PIO instruction set has only **nine instructions**: `jmp`, `wait`, `in`, `out`, `push`, `pull`, `mov`, `irq`, `set`.
- Purpose: eliminates CPU-intensive "bit-banging" (manually toggling pins in a tight software loop) by offloading timing-critical, repetitive protocol work to dedicated hardware — freeing up the main processor.

**How to use PIO (RP2040 has 2 PIO blocks, 4 state machines each, access to every GPIO):**

1. **SELECT** — choose PIO0 or PIO1 and claim an unused state machine.
2. **CONFIGURE** — load the PIO program, map it to specific GPIO pins, and choose a divided clock speed.
3. **RUN** — enable the state machine. It then autonomously controls the pins while your main `main()` code keeps running independently.

**Example — blinking an LED via PIO (`blink.pio`):**
- Host-side setup: claim a state machine, load `blink.pio`, map it to GPIO 25 at 2 kHz, enable the state machine.
- The PIO program alternates 100 cycles HIGH and 100 cycles LOW, producing a 10 Hz blink — entirely without any CPU intervention once it's running, so `main()` is free to sleep or do other work.

### 26. Reference Documentation — Putting It Together

| Question | Where to Look |
|---|---|
| Where do I connect it? | Exact board pinout and schematic |
| What can this pin safely do? | MCU electrical tables and GPIO chapter (datasheet) |
| Which code call configures it? | Pico SDK `hardware_gpio` documentation |
| How do events request attention? | `pico-examples` → `gpio` → `hello_gpio_irq` |

---

## PART 2: INTERRUPT FUNDAMENTALS (W2.2)

### 27. Why Interrupts Exist

- Problem scenario: the CPU is busy doing useful work (e.g., a 200ms display update) when a button event happens (a brief 20ms pulse). If the CPU is only checking for input at fixed points, it might completely miss short events that occur while it's busy.
- Core question: **how can the MCU "remember" that an event happened while the CPU was busy doing something else?**

### 28. Polling vs Interrupts — Two Answers

**Polling:** the CPU actively and repeatedly asks "did anything happen?" at a fixed interval.
- A **latched** event (one that stays flagged until read) will wait for you to check it.
- But a very short, **unlatched** pulse can disappear before you happen to check — you might poll right before and right after it, and miss it entirely.

**Interrupts:** the hardware itself signals the CPU the moment an event occurs, without the CPU needing to ask.

### 29. The Formal Polling vs Interrupt Model

Definitions:
- **T** = the fastest time between events occurring (worst-case event rate)
- **D** = a deadline (how quickly the event must be handled)
- **p** = polling period (how often you check, if polling)
- **c** = CPU time taken to handle one event
- **h** = hardware response time (interrupt latency)

**Polling constraints:**
- Polling period `p` must be **greater than c** and **less than T** (you must poll often enough to catch events, but each poll+handle cycle must fit before the next event could arrive).
- Worst case, polling takes `p + c` time to fully process an event, so the deadline constraint is:

  $$2c \leq c + p \leq D \leq T$$

- **Utilisation:** $u_{polling} = c / p$ (fraction of CPU time spent servicing events via polling)

**Interrupt constraints:**
- For interrupts, the constraint is different: $h + c \leq D \leq T$ (hardware latency plus handling time must fit within the deadline).
- **Utilisation:** $u_{int} = (h + c) / T$

*(Note: the annotated slide worked through a numeric example computing utilisation percentages — e.g., polling utilisation around 20%, interrupt-based utilisation compared against it — illustrating that interrupts are often far more CPU-efficient for sparse or unpredictable events, since the CPU isn't burning cycles repeatedly checking for nothing.)*

### 30. Choosing the Right Mechanism for the Workload

| | POLL | INTERRUPT | DMA + INTERRUPT |
|---|---|---|---|
| **Behaviour** | Ask repeatedly | React to an event | React to a completed block of data |
| **Best for** | Short, predictable wait | Sporadic / time-sensitive events | Sustained data streams |
| **Complexity** | Simple control flow | Useful work possible between events (or sleep) | Lower per-item CPU overhead |

- **Rule:** choose your mechanism based on event rate, response deadline, CPU cost, and implementation complexity.

### 31. The Five Questions to Trace Any Interrupt

| WHAT | WHY | WHEN | WHERE | HOW |
|---|---|---|---|---|
| A request | Timely response | Event occurs | Source → CPU | ISR → resume |

By the end of this topic you should be able to: **trace** an interrupt end-to-end, **configure** it, and **choose** when it's the right tool.

### 32. What Is an Interrupt? A Temporary Detour

- **IRQ** (Interrupt Request) arrives → the CPU temporarily suspends **MAIN WORK** → runs the **ISR** (Interrupt Service Routine) → then **MAIN RESUMES** where it left off.
- Three phases: **entry** (save context) + **service** (run the ISR) + **return** (restore context and resume).
- **Key principle: "The source requests attention; the CPU decides when it can respond."** (Interrupt handling isn't always instantaneous — priority and masking affect exactly when it runs.)

### 33. Source, Request, Handler — Three Different Things

It's important not to conflate these three separate concepts:

| Concept | Example |
|---|---|
| **SOURCE** | The physical/logical cause — e.g., a timer reaching a match value |
| **IRQ** | The request for attention that gets raised as a result |
| **ISR** | The actual code that runs to handle the cause |

**"A peripheral raises an IRQ. An ISR services it."**

### 34. Types of Interrupts — Why They're Good

- An interrupt is something that interrupts normal program flow because a signal indicates **immediate action is needed.**
- Real-world examples: a computer's reset button, an over-pressure sensor in a nuclear reactor, a deep-sleep wake-up event on an edge node, a memory error, or a floating-point error.
- **"Interrupts are good!"** — they let a system respond immediately to critical or time-sensitive events rather than waiting for the next poll.
- General cycle: **RUN → RESPOND → RESUME**.

### 35. Where Interrupts Connect Into the System

```
GPIO/sensors   ─┐
Timer/PWM      ─┼──► INTERRUPT CONTROLLER ──► CPU executes ISR
UART/ADC/DMA   ─┘
```

- The peripherals themselves are **on-chip**.
- The interrupt controller manages **enable, pending, and priority** state for each interrupt source.
- The CPU/RAM hold the **execution context and data buffers** needed while servicing the interrupt.

### 36. Three Gates Decide Whether an ISR Can Start

An interrupt only actually fires and runs its ISR if **all** of these gates are open:

1. **SOURCE** — the cause itself must be enabled at the peripheral level.
2. **CONTROLLER** — the specific IRQ line must be enabled at the interrupt controller.
3. **CPU** — the CPU's global interrupt mask and priority settings must allow it through.
4. Only then does the **ISR** actually start servicing.

- **Important distinction:** "pending" does NOT mean "running" — an event can be flagged as pending while waiting for a gate to open. And masking an interrupt does not necessarily erase the pending event; it may still be waiting when you unmask it.

### 37. The RESET Button — A Special Case

- Pressing RESET stops the processor immediately and restarts execution from the beginning.
- Reminder of the basic CPU execution cycle: **1) fetch instruction from Program Counter (PC), 2) execute instruction, 3) increment PC.**
- RESET works by forcing the PC to a new, fixed value — specifically, the **reset vector** stored in the **Interrupt Vector Table (IVT)**.

### 38. The Interrupt Vector Table (IVT)

- When any interrupt occurs, code execution "jumps" to a **fixed location** in the IVT associated with that specific interrupt type.
- The programmer (or the operating system/SDK) places instructions at each fixed vector location to say "branch to this interrupt service routine."
- There is normally **one ISR per distinct interrupt/reset type** you want to handle. The ISR code itself, however, can physically live anywhere in memory — the IVT just holds a pointer/branch to it.
- Example table layout includes entries like: boot loader, initial stack pointer, Reset Handler, NMI Handler, HardFault Handler, SVC Handler, PendSV, SysTick Handler, and a series of numbered peripheral ISR slots.

### 39. Interrupt Process — Context Save and the Stack

**Interrupt Response:**
- Before jumping to the IVT, the CPU automatically **saves its context** — this typically includes the Program Counter (PC) and Status Register (SR), and possibly more, depending on the architecture.
- This is essential so that after the interrupt is serviced, the CPU can **restore that context** and continue exactly where it left off.

**The Stack:**
- A region of RAM used to temporarily store data, retrievable later.
- The CPU can **push** things onto the stack, and later **must pop** them back off (in the reverse order) — this matters because if you don't correctly pop everything you pushed, the stack becomes corrupted and the CPU can crash or return to the wrong place.
- The **Stack Pointer (SP)** is a register that always points to the address of the most recently pushed item.

### 40. Context Save — What Actually Gets Stored (ARM Cortex-M example)

On an ARM Cortex-M interrupt entry, a specific stack frame is automatically pushed (in decreasing memory address order), typically including:
- **xPSR** (Program Status Register)
- **PC** (Program Counter — the return address)
- **LR** (Link Register)
- **R12, R3, R2, R1, R0** (general-purpose working registers)

The full register set available includes low registers (R0–R7), high registers (R8–R12), plus special registers: Stack Pointer (R13/SP), Link Register (R14/LR), Program Counter (R15/PC), and status/control registers (PSR, PRIMASK, CONTROL).

### 41. Save → Service → Restore

The overall interrupt handling model, summarised:

**SAVE** (resume address + required registers) → **SERVICE** (handle the actual cause) → **RESTORE** (resume execution correctly)

- This is a shared responsibility between the **hardware** (automatic register stacking) and the **startup/runtime code** (compiler-generated ISR prologue/epilogue).

### 42. Full Recap of the Interrupt Sequence

1. Something **interrupts** the normal flow of program execution.
2. The CPU responds by **pushing** the current context onto the **stack**.
3. It then **branches** to the correct entry in the **Interrupt Vector Table (IVT)**.
4. From there it **branches** to the actual **Interrupt Service Routine (ISR)**, which services the interrupt.
5. The ISR **restores context** (pops it back off the stack).
6. Execution **returns** and **resumes** exactly where the main program left off.

### 43. Edges vs Levels

- **Rising edge:** a transition from LOW to HIGH (a moment in time).
- **Falling edge:** a transition from HIGH to LOW (a moment in time).
- **Level (e.g., "HIGH level remains true"):** a *condition* that persists over a stretch of time, not just a single instant.
- **"An edge is a transition. A level is a condition that persists."** — this distinction matters because you can configure an interrupt to trigger on an edge, on a level, or both, depending on the hardware and what behaviour you want.

### 44. One Pending Bit Is Not an Event Queue

- If three separate rising edges occur in quick succession but there's only a single pending bit/flag for that interrupt source, the hardware can only tell you "at least one event happened" — not "three events happened."
- Three events → **PENDING = 1** → only **ONE service** call results.
- If you truly need to count or capture *every* individual event, you need a hardware counter, a FIFO buffer, or a bounded software queue. Even careful software-based telemetry can still miss events if not designed carefully.

### 45. Service the Cause Before Returning

Different interrupt sources have different ways of signalling "I'm done, stop bothering the CPU":

| Source | What Happened? | What Quiets It? |
|---|---|---|
| Timer | An alarm/match flag fires | Acknowledge the documented status flag |
| UART receive | The FIFO buffer reaches a threshold | Drain enough of the received data |
| GPIO level | The input pin is held in an active state | Physically remove the triggering condition, or mask the source |

- **Rule:** always read the specific peripheral's documented status and acknowledgment rules — this varies by peripheral and chip family.

### 46. Configure First, Enable Last

The recommended safe sequence when setting up any new interrupt source:

1. Keep the source **disabled** initially.
2. Set the pin, timer, or trigger mode you want.
3. Install your handler function, and initialise any shared state variables it depends on.
4. Clear any stale/leftover status flags, and set the desired priority level.
5. **Only then** enable the intended source and its routing to the CPU.

- Doing this in the wrong order can cause a spurious interrupt to fire before your handler or state is actually ready.

### 47. Writing a Good ISR

Key rules for ISR code:

- ISR code should be **tiny, fast, and efficient** — it should disturb the rest of the system as little as possible.
- Interrupts are **asynchronous and unpredictable** — they can happen at literally any time (raising re-entrancy concerns).
- You can **mask (disable)** interrupts you don't currently need, but some interrupts are **Non-Maskable (NMI)** — meaning they cannot be disabled, by design, because they're too critical to ever ignore.
- On exit, leave the CPU in exactly the same context/state it had at entry (though CPUs handle some — but not necessarily all — of this automatically).
- You usually **must explicitly clear the interrupt flag** before exiting the ISR — otherwise, the same interrupt will fire again immediately upon return.
- ISRs typically rely on **global variables** for communication, since you generally can't pass normal function parameters to an interrupt handler.
- Some systems support **nested interrupts** (a higher-priority interrupt can interrupt a lower-priority ISR that's already running); most systems use **interrupt priority levels** to decide what happens when multiple interrupts occur at once.
- **Never put an infinite loop or a `printf` call inside an ISR** — both can be far too slow or risky for time-critical interrupt context (`printf` in particular is often non-reentrant and can block for a long, unpredictable time).

### 48. "Capture Now. Process Later." — The Recommended ISR Pattern

Split responsibilities cleanly between the ISR and your main program:

| ISR (fast, minimal) | Main Loop / Task (does the real work) |
|---|---|
| Capture | Parse |
| Acknowledge | Compute |
| Notify | Update / log |

- **"Short, bounded work keeps the rest of the system responsive."** The ISR just grabs the essential data and flags that something happened; the actual heavy processing happens later, in normal program flow.

### 49. A Flag Is Enough When Events May Coalesce

Example pattern — using a simple shared flag to hand off work from the ISR to the main loop:

```c
// ISR
service_device_cause();
pending = true;
return;
```

```c
// MAIN LOOP
if (take_and_clear(pending)) {
    process_event();
}
do_other_work();
```

- **Critical requirement: `take_and_clear()` must be atomic relative to the ISR** — meaning the read-and-clear operation on the shared flag must not be interruptible halfway through, or you risk a race condition (see next section).

### 50. Shared State Can Change Between Instructions (Race Condition Example)

Demonstration of why unprotected shared variables are dangerous:

| Order | Main Code | ISR |
|---|---|---|
| 1 | Reads `count = 7` | |
| 2 | *(paused, e.g. by an interrupt)* | Increments count → `count = 8` |
| 3 | Writes back `saved_value(7) + 1 = 8` | |

- **Expected result:** 9 (if both operations had applied cleanly)
- **Actual result:** 8 (the main code's increment was silently lost, because it was working from a stale copy of `count`)
- **Key lesson: `volatile` alone is NOT the same as atomicity.** `volatile` only prevents the compiler from caching a variable in a register instead of re-reading it from memory — it does **not** protect a read-modify-write sequence from being interrupted partway through. You need actual atomic operations, or to briefly disable interrupts, when both an ISR and main code modify the same shared variable.

### 51. Handler Setup With the Pico SDK

Recommended sequence, once again "configure first, enable last":

1. **Prepare the peripheral** — configure it and clear any stale status flags.
2. **Install a handler** — use either an exclusive handler, or a shared handler if multiple sources share the same IRQ line.
3. **Enable the event** — allow the peripheral to actually assert its IRQ line.
4. **Enable the NVIC** (Nested Vectored Interrupt Controller) — this final step affects the executing core's ability to actually respond to that IRQ.

**Simplified C example:**
```c
static void uart0_isr(void) {
    while (uart_is_readable(uart0)) {
        uint8_t b = uart_getc(uart0);
        ring_push(b);
    }
}

irq_set_exclusive_handler(UART0_IRQ, uart0_isr);
irq_set_priority(UART0_IRQ, 0x40);
uart_set_irq_enables(uart0, true, false);
irq_set_enabled(UART0_IRQ, true);
```

- **Warning:** enable the source at the SDK/peripheral level, and clear or drain the peripheral source — otherwise the IRQ can fire again immediately, right after you think you've handled it.

### 52. Timers as an Interrupt Source

Model: **CLOCK** (tick, tick, tick...) → **COUNTER** (...997, 998, 999...) → **MATCH** (count reaches a set threshold) → **IRQ** fires, meaning "sample is due."

Common uses: periodic sampling, timeouts, and timestamping.

- **Important subtlety:** the actual timer event time and the moment the ISR *starts executing* are two different points in time — there's always some latency between "the match occurred" and "the CPU is actually running the handler code."

### 53. Debouncing at the Interrupt Level

- Just like with polling, one physical button press can generate a raw signal with many rapid edges.
- Because of this, debounce logic still needs to use **time and recorded state**, not merely a `delay()` call placed inside the ISR itself (which would violate the "keep ISRs short" rule).

### 54. A Higher-Priority IRQ Can Pre-empt a Running ISR

- If a low-priority ISR is running and a higher-priority interrupt request arrives, the CPU will **pause (pre-empt)** the low-priority ISR, run the high-priority one to completion, and only then let the low-priority ISR **resume**.
- **Important nuance: "Priority chooses who waits. It does not create more CPU time."** Pre-emption doesn't give you extra processing capacity — it just changes execution order.

### 55. Too Many Interrupts Can Consume the CPU

Approximate formula: **ISR load ≈ (event rate) × (time spent per ISR call)**

Example illustrating this:
- 1 kHz event rate × 20 µs per ISR call ≈ **2%** CPU load
- 20 kHz event rate × 20 µs per ISR call ≈ **40%** CPU load

- **Caution:** these are illustrative average values only. A low *average* CPU load does **not** by itself prove that all real-time deadlines will actually be met — bursts, jitter, and worst-case timing all matter too.

### 56. Same Fundamentals, Different Entry Machinery (RP2040 vs RP2350)

| Target | IRQ Control Mechanism | Handler Entry Path |
|---|---|---|
| RP2040 (Arm Cortex-M0+) | NVIC | Vector table → handler |
| RP2350 (Arm Cortex-M33) | NVIC | Vector table → handler |
| RP2350 (Hazard3 RISC-V core) | Xh3irq | Trap entry → software dispatch |

- **Takeaway:** the underlying model of event → request → service → resume is universal, even though the specific mechanism for entering the handler differs by CPU architecture.

### 57. Full Takeaway Sequence

**EVENT → REQUEST → SAVE → SERVICE → RESUME**

### 58. A Portable SDK Keeps Application Intent Clear

- Pattern: **BUTTON** (e.g. falling edge) → **SDK SETUP** (select source + callback) → **PLATFORM** (routes internally to NVIC or Xh3irq depending on chip) → **CALLBACK** (records the event).
- Example SDK call: `gpio_set_irq_enabled_with_callback(...)`
- **Key point:** even fully portable SDK source code still needs to be *built* (compiled) specifically for the selected target chip.

### 59. Debugging the Event Path, One Link at a Time

If your interrupt isn't behaving as expected, check each link in the chain individually:

**EVENT? → ENABLED? → HANDLER? → CLEARED?**

| Step | What to check |
|---|---|
| EVENT? | Is the pin/timer/status actually changing as expected? |
| ENABLED? | Is it enabled at the source, controller, AND core level? |
| HANDLER? | Is the correct entry/callback actually installed and being called? |
| CLEARED? | Has the underlying cause actually been serviced/acknowledged? |

- **Practical debugging trick:** toggle a spare GPIO pin at ISR entry and exit, then observe it on an oscilloscope or logic analyser — this makes ISR timing and frequency directly visible.

### 60. RP2040 Interrupt Latency — Tail-Chaining

- When two interrupts of different priority are pending very close together, a **typical processor** without optimisation would fully unstack after the first ISR, then fully re-stack before starting the second ISR — this wastes cycles doing redundant stack operations.
- The **RP2040's Cortex-M0+** implements **tail-chaining** in hardware: if a second interrupt is already pending right as the first ISR finishes, the CPU skips the redundant unstack/restack and jumps almost directly into the second ISR.
- Example savings shown: a typical approach might cost 28 cycles (13 to unstack + 15 to stack again), while tail-chaining reduces this to just 6 cycles — a saving of 22 cycles per transition.

### 61. RP2040 Interrupt Latency — Late Arrival

- Scenario: a **lower-priority** interrupt (IRQ2) is already being stacked/entered when a **higher-priority** interrupt (IRQ1) arrives *late*, just before the lower one starts executing.
- A typical processor would need a second full stacking operation before it can jump to the higher-priority ISR — costing more cycles.
- The RP2040's Cortex-M0+ hardware detects this "late arrival" case and re-routes directly to the higher-priority ISR first, before running the originally-scheduled lower-priority one, saving cycles compared to a naive implementation.

### 62. Exit Check — Questions to Test Your Understanding

1. Which is the source: the timer, or its ISR?
2. Can an event occur while its interrupt is masked?
3. Why might an ISR run again immediately (right after returning)?
4. What kind of work would you deliberately move *out* of an ISR, and why?

### 63. Preview: Interrupt Source Catalogue — Time and Inputs

| Source Family | Typical Cause | Example Use |
|---|---|---|
| GPIO / external IRQ | Rising, falling, or active level | Buttons, encoders, sensor data-ready signals |
| Timer/counter | Compare match, overflow, capture event | Scheduling, timeouts, timestamps |
| PWM | Wrap/cycle boundary, or fault* | Motor-control updates, waveform stepping |
| RTC / low-power timer | Alarm, periodic wake | Scheduled wake-up, timekeeping |
| ADC / comparator / touch* | Result ready, threshold crossed | Measurement, threshold detection |

*Availability and exact event types depend on the specific MCU.

### 64. Preview: Interrupt Source Catalogue — Data and I/O

| Source Family | Typical Cause | Example Use |
|---|---|---|
| UART / serial | RX data ready, TX buffer space, framing error | Console output, GPS, general device links |
| SPI / I²C | FIFO event, transfer completion, bus error | Sensors, memories, displays |
| USB / CAN / Ethernet* | Transfer/packet event, bus event | Connectivity and protocol stacks |
| DMA / accelerators* | Block transfer complete, error | Streaming data, checksums, crypto operations |
| PIO / storage / radio* | FIFO event, custom event, completion | Custom interfaces, buffered I/O |

*Availability and exact event types depend on the specific MCU.

### 65. Preview: System Events Also Change Control Flow

Beyond peripheral interrupts, other system-level events also redirect execution:

| System Event | Purpose | Key Distinction |
|---|---|---|
| Software / inter-core IRQ | Notify another context or CPU core | Often paired with queues or mailboxes |
| OS tick / timer | Scheduling and timeouts | Can be a peripheral timer or a core-local timer |
| Power / clock / NMI* | Wake-up or urgent system condition | Has special enable/masking rules |
| Fault / SVC / ecall | Error handling / system service call | A *synchronous* exception — not a normal I/O IRQ |
| Reset / watchdog* | Recover or restart execution | Usually not a normal "return from interrupt" path |

*Device-specific behaviour. Reset is a fundamentally separate execution path, not a return from an interrupt.

---

## Quick-Reference Summary

- **GPIO** = one digital bit per pin; the circuit sets the voltage, your code gives it meaning.
- Undriven inputs float — always use a pull-up or pull-down resistor.
- Outputs are current-limited — never treat a GPIO pin as a power supply.
- Bitmasks let you touch one bit of a register without disturbing the rest (`|=` to set, `&= ~mask` to clear, `^=` to toggle).
- Debounce = wait for stability before accepting a state change, whether polling or interrupt-driven.
- **PIO** offloads repetitive, timing-critical bit-banging to independent hardware state machines.
- **Interrupts** let hardware notify the CPU of events instead of the CPU repeatedly asking — much more efficient for sparse/unpredictable events, governed by the constraint `h + c ≤ D ≤ T`.
- The interrupt lifecycle is always: **event → request (IRQ) → save context → service (ISR) → restore context → resume**.
- Keep ISRs short: **capture, acknowledge, notify** — then let the main loop do the real work.
- Shared variables touched by both an ISR and main code need atomic protection — `volatile` alone is not enough.
