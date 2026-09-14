# INF2004 Embedded Systems Programming — Lecture 1 Study Notes
## Introduction to Embedded Systems Programming

---

## Learning Objectives

By the end of this lecture, you should be able to:

1. Define what an embedded system is, using multiple standard definitions, and place it in the context of Cyber-Physical Systems (CPS) and the Internet of Things (IoT).
2. Distinguish between a microcontroller, a single-board computer, and a general-purpose computer.
3. List and explain the core properties that all embedded systems share.
4. Define "real-time" correctly, and classify systems as non-real-time, soft real-time, or hard real-time based on the consequences of missing a deadline.
5. Identify and differentiate between different embedded hardware options (µC, SoC, ASIC, NPU, µP, DSP, FPGA, GPU).
6. Explain the difference between a microprocessor, a microcontroller, and a System on Chip (SoC).
7. Explain why embedded software quality is uniquely critical, and analyse real-world case studies (737 MAX MCAS, Mars Climate Orbiter, Therac-25, Toyota unintended acceleration, Ariane 5) of what happens when it fails.
8. Understand why the Software Development Life Cycle (SDLC) matters especially for embedded systems.
9. Explain why low-level languages exist alongside high-level languages, and the trade-offs between them.
10. Trace the full build pipeline from C/assembly source code to a running binary on hardware: compiler, assembler, linker, linker script, startup code, loader/bootloader.
11. Explain what C runtime startup code (crt0.s) does and why it must run before `main()`.
12. Explain, in plain English, what a C SDK, a compiler, and a bootloader each do — specifically for the Raspberry Pi Pico.
13. Understand "firmware-critical" C primitives: fixed-width integer types, bitwise register control, `const`/`volatile` qualifiers, pointers for memory-mapped I/O, scope/lifetime/linkage (`static`/`extern`), and struct memory layout — and why these matter more in embedded C than in ordinary application programming.

---

## PART 1: WHAT IS AN EMBEDDED SYSTEM?

### 1. Embedded Systems in Context: Cyber-Physical Systems and IoT

- **Cyber-Physical Systems (CPS):** systems where computation, networking, and physical processes are tightly integrated — think Industry 4.0, smart industry, big data feeding back into physical control.
- **The Internet of Things (IoT):** the broader network of everyday connected devices (smart homes, wearables, drones, appliances) all "talking" to each other and to the cloud.
- Embedded systems are the fundamental building block underlying both of these bigger concepts — every "smart" or "connected" device has an embedded system doing the actual sensing, computing, and controlling.

### 2. Definitions of an Embedded System

There's no single universal definition — the lecture offers several complementary ones:

- **Cambridge Dictionary:** a computing system that does a specific task **inside** a machine or larger electrical system.
- **Prof. Peter Marwedel:** a computer system that is **embedded** into a larger electrical or mechanical system, performing specific dedicated tasks of that larger system.
- **Wikipedia:** a controller with a **dedicated function** within a larger mechanical or electrical system, often with real-time computing constraints.
- **IoT Agenda (TechTarget):** a combination of computer hardware and software, either fixed in capability or programmable, designed for a specific function or functions within a larger system.

**Common thread across all definitions:** embedded systems are *dedicated* (built for one job, not general-purpose), *integrated* (part of something bigger), and often have *timing constraints*.

Examples of devices containing embedded systems: cameras, game machines, smartphones, e-book/tablets, televisions, cars, airlines/aerospace systems, printers, digital signage, amusement machines, broadcasting/medical equipment.

### 3. Microcontroller vs Single-Board Computer vs General-Purpose Computer

| Type | Description |
|---|---|
| **Microcontroller** | Minimalist & embedded — bare chip, breadboard-level, dedicated to one task |
| **Single-Board Computer** | Compact & versatile — a full small computer (e.g., Raspberry Pi) that can run a general OS |
| **General-Purpose Computer** | Powerful & expandable — full desktop/workstation with monitor, keyboard, high-end GPU, etc. |

- These sit on a spectrum from "does one dedicated job with minimal resources" to "does anything, with maximal resources."

### 4. The Analogue and Digital World (Recap from GPIO Lecture)

- The real world is **analogue and continuous**; computers are **digital and discrete**.
- **ADC (Analog-to-Digital Converter):** converts a real-world analogue signal into digitised data for processing.
- **DAC (Digital-to-Analog Converter):** converts digital processed data back into an analogue output signal.
- Embedded systems must mix analogue and digital electronics together to properly represent and act on real-world signals — this is a foundational theme that reappears throughout the whole module (see Lecture 2 notes on GPIO).

### 5. Properties of Embedded Systems

Every embedded system tends to share these core properties:

- **Embedded/integral part of a device** — often bundled with hardware and electromechanical parts, not sold as a standalone general computer.
- **Dedicated to specific tasks** — the tasks are known and assigned at design-time (not decided by the end user later, unlike a general-purpose PC).
- **Uses sensors and actuators** — this is how it interacts with the real, physical world (sensors = input, actuators = output).
- **Strict timing constraints** — because of their interaction with the physical world, often in safety-critical environments.
- **Limited resources** — compared to general-purpose computers, embedded systems often have far less memory, processing power, and energy budget.
- **Based on microcontrollers** — i.e., CPUs with integrated memory and/or peripheral interfaces built into a single chip.

These properties map to the diagram's five themes: **real-time operation, reliability, dedicated function, efficiency, and (implicitly) resource constraint.**

---

## PART 2: DETERMINISM AND TIMING

### 6. What Does "Real-Time" Actually Mean?

- Many embedded systems have **real-time constraints** — but "real-time" does **not** mean "fast." It means something much more precise.
- **Definition: A real-time system is one in which the correctness of the application depends on BOTH its functional results AND the temporal (timing) results — i.e., the time at which the result is produced.**
- **"In other words, a late answer is a wrong answer."**
- Put another way: real-time means the ability to service a request **before a required deadline**. Example: if the deadline is 1 microsecond, the system must service the request within that 1 microsecond — if it consistently can, it qualifies as a real-time system for that requirement.

### 7. Classifying Systems by the Consequence of a Missed Deadline

| Category | Characteristics | Examples |
|---|---|---|
| **Non real-time constraints** | Tasks have no deadlines. A late response usually doesn't significantly affect performance/functionality. Causes no danger. | TV remote control, lawn sprinkler, window lifter |
| **Soft real-time constraints** | Tasks are allowed to occasionally miss deadlines. A late response can *degrade* system performance, but a missed deadline causes no significant danger. | Mobile phone, air-conditioning, refrigerator |
| **Hard real-time constraints** | It is imperative that tasks meet their deadlines at all times. A late response is not considered acceptable even if its *value* is otherwise correct. A missed deadline can have catastrophic consequences. | Nuclear reactor temperature control, airbag controller, flight control system |

- The key distinguishing question for any system: **"What happens if this task is late? Merely annoying, degraded service, or catastrophic?"**

---

## PART 3: EMBEDDED HARDWARE OPTIONS

### 8. Different Embedded Systems Hardware Options

| Hardware | What it is |
|---|---|
| **Microcontroller (µC)** | A self-contained chip combining CPU + memory + peripherals for embedded, dedicated tasks (e.g., NXP LPC214x) |
| **System on Chip (SoC)** | A larger, more integrated chip combining multiple processors/subsystems onto one die (e.g., an Apple A-series chip) |
| **Application Specific Integrated Chip (ASIC)** | A chip custom-designed and hard-wired for one specific application — cannot be reprogrammed for a different task |
| **Neural Processing Unit (NPU)** | A chip specialised for accelerating machine-learning/neural-network workloads |
| **Microprocessor (µP)** | The core CPU chip alone, without integrated memory/peripherals (e.g., bare ARM Cortex core) |
| **Digital Signal Processor (DSP)** | A processor specialised for fast, repetitive mathematical operations on signal data (audio, sensor data, etc.) |
| **Field-Programmable Gate Array (FPGA)** | Reconfigurable hardware logic — you can "rewire" its internal digital circuit itself, rather than just running software on a fixed circuit |
| **Graphics Processing Unit (GPU)** | A processor specialised for massively parallel computation, originally for graphics rendering, now widely used for ML/AI too |

### 9. Microprocessor vs Microcontroller

- A **microprocessor** (e.g., an ARM Cortex-M3 core) is essentially just the CPU core plus a few very tightly coupled support blocks (nested vectored interrupt controller, wake-up interrupt controller, memory protection unit, debug/trace hardware like JTAG and ETM trace).
- A **microcontroller** (e.g., NXP LPC177x/8x) takes that microprocessor core and surrounds it with a much richer set of integrated peripherals on the same chip: flash memory, SRAM, EEPROM, ROM, GPIO, DMA, power control, clock generation, timers, watchdog timer (WDT), real-time clock, various communication interfaces (Ethernet MAC, USB, SD/MMC, CAN, UART, SPI, I2C, PWM), and ADC/DAC.
- **Key takeaway (annotated on the slide): "A microcontroller is just a microprocessor that is more advanced with more integrated features."** In other words, a microcontroller = a microprocessor core + memory + peripherals, all packaged into one chip so it can run standalone without needing lots of external support chips.

### 10. Microcontroller vs SoC (System on Chip)

- A **microcontroller** typically contains **one or two** processor cores, and these are often relatively modest cores (e.g., a single Cortex-M3).
- An **SoC** typically contains **multiple** processor cores, and usually **higher-end** ones — for example, the Exynos 4412 Application Processor slide shows **four ARM Cortex-A9 cores**, each with its own L1 instruction/data cache and NEON SIMD extensions, plus a dedicated multi-core GPU, video decode/encode block, camera interfaces (MIPI DSI/CSI), and a wide range of high-speed I/O (USB 2.0 OTG/HSIC, eMMC, SD, SDIO, SPI, I2C, UART, I2S).
- **Key takeaways (annotated on the slide):**
  - "Microcontrollers may have one or two microprocessors, while SoCs have multiple, and usually higher-end microprocessors — such as Cortex-A9 — compared to Cortex-M3."
  - "An SoC is essentially a combination of multiple embedded systems in one chip" (CPU cluster + GPU + multimedia engines + memory controller + many peripheral interfaces, all on a single die).

---

## PART 4: WHY EMBEDDED SOFTWARE MUST BE DONE RIGHT

### 11. Why Embedded Software Quality Matters So Much

- **Embedded software is described as "the hidden brain" inside the devices we use daily** — from phones and cars to medical equipment and airplanes.
- Its **pervasive** nature ("it has spread deeply into everything") means its quality directly impacts safety, reliability, and business success.

**Why It Matters:**
- **Ubiquity** — it's found in almost every device (washing machines, smartwatches, cars).
- **Mission-Critical** — it can literally be the difference between life and death; it controls systems like pacemakers and airplane autopilots.
- **Core Competency** — it's an essential skill companies rely on for success (example given: Tesla's car software is as essential to the product as its engines).
- **Unique Challenges** — embedded software must work within limited memory and power, and must run reliably without crashing (example: a car airbag must always respond instantly, with zero tolerance for failure).

**The Imperative for Excellence:**
- **Proactive Quality** — don't wait for errors to show up in real-world use. Test and catch problems *during development*. (Example: a phone app bug can often be patched later after release, but a bug in a safety-critical system like an airbag controller absolutely must be fixed *before* the car is ever sold.)
- **Rigorous Standards** — use proven development methods, safety checks, and formal certifications to ensure reliability. (Example: airplane software must follow the aviation standard **DO-178C** to be certified safe before a plane is allowed to fly.)

### 12. Case Study — One Software Mistake Is All It Takes

- **Core message: poorly designed software can severely damage a brand's reputation, and in the worst cases, contribute to a company's collapse.**
- **Example referenced: the Boeing 737 MAX MCAS (Maneuvering Characteristics Augmentation System)** — a flight control software system whose design flaws contributed to fatal crashes and enormous reputational and financial damage to Boeing.
- **Example referenced: the 1999 Mars Climate Orbiter incident** — the spacecraft was lost due to a units-conversion mismatch between two pieces of software (one team used metric units, the other used imperial/English units), illustrated by the classic "Metric, English, whatever..." cartoon. This is a famous cautionary tale about the cost of sloppy interface assumptions between separate software components.

*(For extra context: the Mars Climate Orbiter disintegrated in the Martian atmosphere in September 1999 because one piece of ground software calculated thruster performance data in pound-force seconds, while the navigation software expected the value in newton-seconds — a mismatch that was never caught until it was far too late. It's one of the most-cited examples in software engineering education for why explicit unit/interface contracts matter.)*

### 13. Embedded Software: More Than Just Code

Embedded software development has **unique and significant challenges** that differentiate it from ordinary desktop or web application development, because of the tight integration with hardware and stringent performance requirements.

**Key Challenges:**
- **Resource Constraints** — limited memory, CPU power, and energy budget.
- **Real-Time Requirements** — as discussed above, timing correctness is part of functional correctness.
- **Hardware-Software Co-Design** — hardware and software decisions affect each other and often must be designed together.
- **Concurrency and Interrupts** — multiple things can happen "at once" (interrupts, multiple peripherals), requiring careful coordination.
- **Reliability and Safety** — failures can have real-world physical consequences, unlike most desktop software bugs.
- **Debugging Difficulties** — embedded bugs are often much harder to reproduce and observe than desktop bugs (no easy console, limited visibility into hardware state, timing-dependent behaviour).

**Consequences of Failure:** many embedded systems are **mission-critical**, performing functions where failure is unacceptable — this can lead to **safety risks** and/or **significant financial or business loss**.

### 14. Case Study — The Therac-25 Race Condition (1985–1987)

A classic and extremely important case study in embedded software safety:

**The Bug:**
- A **race condition** between the UI (user interface) software and the control software.
- Triggered only by a **specific, unusual sequence of rapid keystrokes** by the machine operator.
- The result: the machine delivered **lethal radiation overdoses** to patients instead of the intended safe therapeutic doses.

**Reproduction Difficulty:**
- The bug was a classic **"Heisenbug"** — it tended to **disappear whenever someone tried to observe it directly** (e.g., by attaching a debugger, since debugging tools altered the exact timing needed to trigger the race condition).
- It took **years and multiple deaths** before the exact root cause was finally identified.

**High Cost of Failure:**
- **At least six lethal overdoses**, resulting in deaths and severe injuries.
- Multiple lawsuits, severe reputation damage, a complete software redesign, and the introduction of new industry safety protocols as a direct result.
- Immense financial cost (recalls, legal fees, lost business) plus an immeasurable human cost.

**Discussion prompt from the slide: "What testing could have prevented this?"**

*(Additional context for study: the Therac-25 is one of the most cited case studies in software engineering safety courses worldwide. Root causes included removing hardware interlocks that earlier machine models had relied on, poor software architecture that allowed a race condition between the data-entry keyboard task and the treatment-monitor task, and a general absence of rigorous software safety review before deployment.)*

### 15. Case Study — Toyota's "Spaghetti Code" Issue

- Toyota's car software became **famously difficult to maintain and debug** due to its **unstructured design** — a style commonly referred to as **"spaghetti code"** (code so tangled and interdependent that it's hard to trace).
- This complexity became a **significant obstacle during investigations into unintended acceleration incidents**, making it difficult to pinpoint the actual source of the problem.

| Issue | Description |
|---|---|
| **Complexity** | The code was a monolithic, interconnected block without clear modularity, making it difficult to isolate and fix bugs. |
| **Debugging Challenges** | Engineers struggled to trace the flow of logic and identify the source of errors, leading to prolonged and costly investigations. |
| **Safety Concerns** | The entangled nature of the code created an environment where a small change in one area could have unintended and potentially dangerous consequences in another. |

- **Broader lesson: poor software architecture (lack of modularity, unclear structure) is itself a safety risk** — it's not just about individual bugs, but about how hard the *system as a whole* is to reason about and safely change.

### 16. Case Study — The Ariane 5's Fatal Integer Overflow

- A classic, extremely well-documented example of **a single line of code causing catastrophic failure**.
- The **Ariane 5 rocket**, carrying a payload worth **half a billion dollars**, was destroyed just **37 seconds** into its maiden flight.
- **Root cause:** a software bug in the onboard **inertial reference system**.

**"The Bad Line" of Code:**
- The software was originally written to handle a **64-bit floating-point number** representing the rocket's horizontal velocity, and then convert it into a **16-bit signed integer**.
- This code was **reused from the earlier Ariane 4 rocket**, which had a much slower flight profile — so the conversion had never overflowed in that context.
- The specific failing line was:

  ```
  B_H = (float) H_O / 65536.0;
  ```

- The value of `B_H` was stored in a variable that was **too small** to hold it, and the code **failed to perform a simple range check** before the conversion.

**The chain of events that followed:**
1. The overflow caused a crucial variable to hold a **nonsensical value**.
2. The flight control system interpreted this garbage data as a **severe error**.
3. Believing it was in a critical fault state, the system **shut down the rocket's engines**, causing the rocket to veer off course and **self-destruct**.

- **Cost:** half a billion dollars, and a major setback for the European space program — all traceable to a single unchecked numeric conversion.

*(This example is directly relevant to the "Firmware-Critical Primitives" section later in this same lecture — fixed-width integer types, overflow, and truncation are exactly the class of bug that caused this disaster.)*

---

## PART 5: THE SOFTWARE DEVELOPMENT LIFE CYCLE (SDLC) AND LANGUAGES

### 17. What Is the SDLC?

The Software Development Life Cycle is:

- A **structured approach** to software development.
- It **ensures repeatability and quality**.
- It is **critical in embedded systems specifically** due to their safety, timing, and resource constraints — the case studies above (Therac-25, Ariane 5, Toyota, 737 MAX) are essentially all examples of what happens when this discipline is missing or inadequate.

### 18. Low-Level Languages for Different Embedded Hardware

- **Low-level language** (e.g., assembly):
  - Good for **squeezing the best performance** from the underlying hardware.
  - Produces **compact and more optimised code**.
- **High-level language** (e.g., C) is required to improve:
  - **Productivity**
  - **Code abstraction**
  - **Portability**
- **Only a machine understands low-level (machine) language** — so to use a high-level language, you need a **compiler** to translate it down into something the hardware actually understands.
- The layered stack shown: **Hardware → Machine Language → Assembly Language → High-Level Language.**

### 19. Assembly vs C — A Concrete Comparison

Given this simple C function:
```c
int add() {
    int a = 6;
    a = 7 + a;
    return a;
}
```

Compiling it with **ARM GCC 8.2** produces assembly output resembling:
```asm
str fp, [sp, #-4]!
add fp, sp, #0
sub sp, sp, #12
mov r3, #6          ; a = 6
str r3, [fp, #-8]
ldr r3, [fp, #-8]
add r3, r3, #7      ; a = 7 + a
str r3, [fp, #-8]
mov r3, #0
mov r0, r3
add sp, fp, #0
ldr fp, [sp], #4
bx lr
```

- **Takeaway:** even a tiny "6 + 7" style calculation in C expands into a meaningful sequence of low-level assembly instructions (stack frame setup, register moves, arithmetic, stack frame teardown, return) — this is exactly what the compiler is doing "under the hood" on your behalf.

### 20. Compiler vs Assembler

**Compiler:**
- Translates programs written in a human-readable (high-level) language into an equivalent set of opcodes for a particular target processor.
- **Cross compiling:** compiling *on* one machine (the "host") *for* a different target platform (e.g., compiling on your x86 laptop, but generating code that will run on an ARM microcontroller).
- **Native compiling:** compiling for the *same* architecture you're currently running under.
- Examples of real compilers: **GCC, KEIL, IAR, CCS**, etc.

**Assembler:**
- Performs a **one-to-one translation** — each assembly instruction maps directly to one machine instruction (unlike a compiler, which can expand one high-level statement into many machine instructions).

### 21. From C/Assembly Source to a Binary File — The Full Build Pipeline

This is one of the most important diagrams in the lecture. The overall pipeline:

```
C Files (.c) ──compiler──► Object Files (.o) ─┐
                                                 │
Reset Code (reset.s) ──────────────────────────┼──► LINKER ──► Binary File (.bin) ──► Loader/Locator ──► Memory
                                                 │        ▲
Assembly Files (.s) ──assembler──► Object Files (.o) ────┘
                                                 │
Startup Code (crt0.s) ─────────────────────────┘

C Library (.a) / C-SDK  ──────────► also feeds into LINKER
Linker Script (.ld)     ──────────► also feeds into LINKER
```

Step by step:
1. **C source files (.c)** are run through the **compiler**, producing **object files (.o)**.
2. **Assembly source files (.s)** — including special files like **reset code (reset.s)** and **startup code (crt0.s)** — are run through the **assembler**, also producing **object files (.o)**.
3. All these object files, plus any needed **C library / C-SDK** code and a **linker script (.ld)**, are fed into the **LINKER**.
4. The linker combines everything into a single **binary file (.bin)**.
5. A **loader/locator** then places that binary into actual **memory** on the target device, where it can run.

### 22. C Runtime Startup Code (crt0.s)

- **This is the first software that executes after reset — even before `main()` is called.**
- It's a **small block of assembly language code** that prepares the way for the execution of software written in a high-level language (like C).
- Most cross-compilers for embedded systems include an assembly language file for this purpose, commonly called **startup.asm** or **crt0.s** (short for "C runtime, file 0").

**Typical steps performed by startup code, in order:**
1. **Disable interrupts** (so nothing can interrupt the fragile setup process while it's incomplete)
2. **Copy data into RAM** (initialised global/static variables need their starting values copied from flash into RAM)
3. **Zero the uninitialised data area** (the ".bss" section — ensures uninitialised globals start at a known value of 0, as the C standard requires)
4. **Allocate space and initialise the stack** (so function calls, including `main()`, will work correctly)
5. **Create and initialise the heap** (if dynamic memory allocation is used)
6. **Enable interrupts** (now that the system is in a safe, known state)
7. **Call `main`**

**Discussion questions posed by the slide (worth thinking through):**
- *"Purpose of a stack and heap?"* — The **stack** holds function call frames, local variables, and return addresses (grows/shrinks automatically as functions are called and return). The **heap** holds dynamically allocated memory that the programmer explicitly requests and frees (e.g., via `malloc`/`free`), and persists until explicitly released.
- *"Why disable then re-enable the interrupt?"* — Interrupts are disabled during startup because the system is in an incomplete, fragile state (stack not yet set up, memory not yet initialised) — if an interrupt fired and tried to use the stack or access uninitialised globals during this window, it could crash or corrupt memory. Interrupts are only safely re-enabled once the system has been fully initialised and is in a known-good state.

### 23. The Linker

- Takes **one or more object files** generated by a compiler/assembler and **combines them into a single executable file**.
- Resolves **cross-file dependencies** properly — e.g., if you define a variable or function in one `.c` file and use it in another `.c` file, the linker is what connects the two references together.
- Can **link/reuse library files** — both **static** libraries (code copied directly into your final binary) and **dynamic** libraries (linked at load/run time — less common in typical embedded firmware, which usually favours static linking).
- **Every link is controlled by a linker script**, which describes **how the executable is mapped onto memory during execution** (i.e., which memory addresses hold which sections — code, initialised data, uninitialised data, stack, heap, etc.).

### 24. Building an Embedded System Application — In Practice

- The lecture references the classic **xkcd #303 comic** ("My code's compiling" — the programmer's excuse for legitimately slacking off while waiting for a build) as a lighthearted nod to how build pipelines, especially larger ones, can take real time to run.

### 25. "Prof, Explain It to Me in Plain English" — C SDK, Compiler, and Bootloader

**C SDK (Software Development Kit):**
- For the Raspberry Pi Pico, the **C SDK** provides a collection of libraries, functions, and tools that simplify hardware interaction.
- It **abstracts the low-level details** of the microcontroller, letting you focus on writing high-level application code.
- The SDK offers **pre-written code to manage peripherals**, such as GPIO, PWM, UART, and others.

**Compiler:**
- The tool that translates your C source code into machine code (binary) that the Raspberry Pi Pico's processor can actually understand.
- It processes the source code, links it with the necessary libraries from the C SDK, and generates an **executable binary file** — typically in the form of a **.uf2** or **.bin** file for the Pico.

**Bootloader:**
- A small program **pre-installed on the Raspberry Pi Pico itself**.
- It's responsible for **loading the compiled binary into the microcontroller's flash memory and executing it**.
- When the Pico is powered on or reset, the bootloader checks for the presence of a new binary file (e.g., delivered via USB) and writes it to flash memory if one is found. **If no new binary is detected, it simply runs the existing program already stored in flash.**

**The overall flow:** **SOURCE CODE → (via C SDK) → COMPILER → .UF2 file → BOOTLOADER/DEBUGGER (on the Pico) → running program**

**Similar/related terms worth knowing (all describe roughly the same idea from different angles):**
- Application Programming Interface (API)
- Development Framework (e.g., Arduino)
- Board Support Package (BSP)
- Toolkit / Toolchain
- Middleware
- Library

---

## PART 6: FIRMWARE-CRITICAL C PRIMITIVES

*(This section is explicitly labelled "for students who already know C programming" — it focuses on what changes, and what matters more, when ordinary C code interacts directly with hardware registers, interrupts, and other tightly constrained embedded resources.)*

### 26. The Core Idea: "A Hardware Register Is Just Bits With Consequences"

- Every C expression that touches hardware ultimately becomes a **register write** — a specific pattern of bits (e.g., `1 0 1 0 0 1 1 0`) that has a real, physical effect the moment it's written.
- **"The syntax is familiar. The consequences are not."** — the C language rules you already know don't change, but what happens when your code actually executes on real hardware can be very different from what happens on a desktop OS.
- **"Firmware correctness depends on knowing what the compiler, memory, and hardware can each change."**

### 27. The Embedded Mindset — Four Key Themes

| Theme | Covers |
|---|---|
| **1. Representation** | Exact width, signedness, overflow and truncation |
| **2. Bit control** | Masks, shifts, read-modify-write |
| **3. Memory** | Pointers, memory-mapped I/O, alignment and layout |
| **4. Visibility** | `volatile` and `const`, scope and lifetime, interrupt sharing |

These four themes structure the rest of this section.

### 28. Theme 1 — Integers: Choose the Representation Deliberately

- Use **fixed-width types** from `<stdint.h>` (`uint8_t`, `int16_t`, `uint32_t`, etc.) instead of plain `int`, so you know exactly how many bits you're working with — critical when that value maps directly onto a hardware register or bus width.

```c
#include <stdint.h>

uint8_t  duty_pct;    // 0..100
int16_t  temperature; // signed reading
uint32_t tick_count;  // timer ticks
```

- **Width:** use `uint8_t`, `int16_t`, `uint32_t` when the exact number of bits matters (e.g., matching a register's width).
- **Signedness:** use **unsigned** types for bit patterns and non-negative counters; use **signed** types for quantities that genuinely can be negative (like a signed temperature reading).

**Three common traps:**
- **Overflow:** `uint8_t x = 255; x++;` → wraps around to `0` (not 256, since an 8-bit unsigned value can't represent 256).
- **Truncation:** `uint8_t y = 300;` → only the lower 8 bits are retained, silently discarding data (this is exactly the *class* of bug that destroyed the Ariane 5 — a value too large for its storage type).
- **Mixed arithmetic:** combining signed and unsigned operands in the same expression can produce **surprising comparison results**, because the signed value may get silently converted to unsigned before the comparison happens.

**Takeaway: "Treat type selection as part of the hardware interface, not just a coding preference."**

### 29. Theme 2 — Bitwise Operations: Change Only the Intended Bits

Given an 8-bit control register starting as `1 0 1 0 0 1 0 0`, setting bit 3 should change **only** that bit, producing `1 0 1 0 1 1 0 0`.

```c
#define BIT(n) (1u << (n))

REG |=  BIT(3);           // set
REG &= ~BIT(3);           // clear
REG ^=  BIT(3);           // toggle
if (REG & BIT(3)) { /* test */ }
```

- **Mask:** a `1` selects the bit you want to operate on; a `0` leaves the other bits unchanged.
- **Shift:** moves a bit pattern into the correct field position required by the specific register you're working with.
- **Parentheses:** make operator precedence explicit in masks and compound expressions — bitwise operators have notoriously easy-to-misjudge precedence in C, so don't rely on memorising the precedence table.

**Takeaway: "Avoid writing a whole register value when only one field should change"** — i.e., prefer `REG |= BIT(3)` over `REG = 0b00001000`, since the latter destroys every other bit's current state.

*(This directly connects to the "Registers Connect Software to Pin Hardware" and "Change Your Bit Without Replacing the Port" sections from the GPIO lecture — this is the same principle, framed from the language/C side rather than the hardware side.)*

### 30. Theme 4 (introduced early) — Qualifiers: Tell the Compiler What May Change

| Qualifier | Meaning |
|---|---|
| **const** | The program must not modify the object through this name. Useful for lookup tables and read-only configuration. |
| **volatile** | The value may change **outside normal code flow** — such as a hardware register, or a flag shared with an ISR. **Every access must occur exactly as written** (the compiler is not allowed to optimise away or reorder accesses to it). |
| **const volatile** | Software only *reads* the value and does not write it, but hardware may still update it independently — e.g., a read-only hardware status register. |

**Example — a classic volatile use case:**
```c
volatile bool sample_ready = false;

void ADC_IRQHandler(void) {
    sample_ready = true;
}

while (!sample_ready) { /* wait */ }
```
Without `volatile` here, the compiler might assume `sample_ready` never changes inside the `while` loop (since nothing in that loop's visible code flow modifies it) and optimise the loop into an infinite loop that never re-checks memory — a very common and dangerous embedded bug.

**Important limitation: `volatile` does NOT make a multi-step operation atomic.** Shared data touched by both main code and an ISR may still need **interrupt masking, atomic operations, or an RTOS synchronisation primitive** to be safe — this is the exact same lesson demonstrated by the "Shared State Can Change Between Instructions" race-condition example in the Interrupts lecture.

**Takeaway: "Use qualifiers to express access rules, not as a substitute for synchronisation."**

### 31. Theme 3 — Pointers: Connect C Objects to Memory and Peripherals

The chain of reasoning for **memory-mapped I/O**:

**Address** (e.g., `0x40021000`) → **Typed pointer** (e.g., `volatile uint32_t *`) → **Register value** (32 hardware-controlled bits)

```c
#define GPIO_OUT (*(volatile uint32_t *)0x40021000u)

GPIO_OUT |= (1u << LED_PIN);
```

- **Read the declaration "inside out":** this declares "a pointer to a volatile 32-bit unsigned integer, located at a fixed address" — then that pointer is dereferenced to read/write the value actually stored there (which happens to be a hardware register, not ordinary RAM).

**Firmware pointer checklist:**
- Correct address and correct peripheral (easy to mistype/misconfigure — a wrong address means you're silently poking a *different* piece of hardware).
- Correct width and alignment (matching the register's actual size).
- `volatile` for any hardware-changing data (so the compiler doesn't optimise away or cache the access).
- Never dereference an invalid or uninitialised pointer.

**Takeaway: "A pointer is not merely an address. Its type controls how many bytes are accessed and how they are interpreted."** And: **"Wrong pointer = wrong hardware."** — a subtly wrong pointer type or address in firmware doesn't just cause a logic bug, it can literally control the wrong physical device or corrupt memory that overlaps unintended hardware.

### 32. Theme 4 (continued) — Scope, Lifetime, and Linkage: Control Who Owns State

| Storage class | Scope | Lifetime | Typical use |
|---|---|---|---|
| **Local variable** | One block or function | One call | Best default choice for temporary state |
| **static local** | One function | Entire program | Retains its value/state between successive calls to that function |
| **static file-scope** | One source file | Entire program | Hides internal module state from other files (encapsulation) |

**Example — private module state:**
```c
// driver.c
static uint32_t error_count;

uint32_t driver_errors(void) {
    return error_count;
}
```

**Example — intentionally shared state across files:**
```c
// driver.h
extern volatile bool driver_ready;

// exactly one .c file actually defines it:
volatile bool driver_ready = false;
```

**Takeaway: "Prefer private module state. Use `extern` only when shared ownership is intentional and clearly documented."** — uncontrolled global/extern variables scattered across many files are a major source of the "spaghetti code" problem discussed earlier (Toyota case study) — encapsulating state with `static` file-scope variables is one concrete defence against that.

### 33. Theme 3 (continued) — Structs and Buffers: Memory Layout Becomes Part of the Interface

```c
typedef struct {
    uint8_t  id;
    uint32_t timestamp;
    uint16_t value;
} Sample;
```

- The compiler may **insert padding** between fields so that each field lands on a memory address boundary the hardware prefers (its "alignment" requirement) — e.g., the struct's actual in-memory layout might look like: `id | pad | timestamp | value | pad`.
- **Important warning: compiler padding may make `sizeof(Sample)` larger than the simple sum of the field sizes** — you cannot safely assume a struct's size just by adding up its members' individual sizes.

| Concept | Meaning |
|---|---|
| **Alignment** | Some processors require or strongly prefer multi-byte values to sit at aligned memory addresses (e.g., a 4-byte value starting at an address divisible by 4). |
| **Packing** | "Packed" structs can remove the padding gaps, but this may create *slower*, or even *invalid*, unaligned memory accesses on some processors. |
| **Byte order** | Explicitly define serialisation (the exact byte-by-byte layout) when sending data across files, buses, or network packets — **do not transmit raw structs blindly**, since padding, alignment, and byte order (endianness) can all differ between the sender and receiver. |

**Takeaway: "Use explicit serialisation when data crosses a processor, compiler, or communication boundary."**

### 34. Recap — Everything Else From C, in One Slide

A final summary of general good-practice C reminders that still apply in the embedded context:

| Topic | Guidance |
|---|---|
| **Functions and control flow** | Use small functions, clear interfaces, and bounded loops (loops with a guaranteed, predictable end — important for real-time and safety reasoning). |
| **Naming and comments** | Name things to reflect *intent*. Comment on hardware assumptions and non-obvious constraints — future readers (including you) need to know *why*, not just *what*. |
| **Preprocessor** | Use `#include` and guarded headers (include guards). Prefer **typed constants** or **inline functions** over risky macros (macros lack type checking and can have subtle side effects). |
| **Operators** | Use parentheses whenever operator precedence is not immediately obvious — don't rely on memorising precedence tables, especially with bitwise operators. |
| **Coding standard** | Apply one consistent coding standard across the whole module/project (this is part of what prevents "spaghetti code"). |
| **Efficiency** | **Measure before optimising.** Prefer clarity unless there's actual timing, memory, or energy *evidence* that says otherwise. |

**Before any code review, check:** widths • masks • qualifiers • pointer validity • shared-state safety • data layout

**Final takeaway of the lecture: "The goal is not 'more C'. The goal is predictable behaviour at the hardware boundary."**

---

## Quick-Reference Summary

- An **embedded system** = dedicated function + integrated into a larger system + often real-time + resource-constrained + built around a microcontroller.
- **Real-time ≠ fast.** It means correctness depends on *both* the result *and* the time it's delivered. Classify by consequence of a missed deadline: non-real-time (no danger) → soft real-time (degraded performance) → hard real-time (catastrophic failure).
- **Microprocessor → Microcontroller → SoC** is roughly a spectrum of increasing integration: bare CPU core → CPU + memory + peripherals on one chip → multiple high-end CPU cores + GPU + multimedia + peripherals all on one chip.
- Every disaster case study (737 MAX, Mars Climate Orbiter, Therac-25, Toyota, Ariane 5) traces back to a preventable software engineering failure — race conditions, unit mismatches, unstructured code, or unchecked numeric overflow — which is exactly why disciplined SDLC practice matters so much in this field.
- The **build pipeline**: C/assembly source → compiler/assembler → object files → linker (+ linker script + libraries) → binary file → loader/bootloader → running on hardware. **Startup code (crt0.s)** runs first, before `main()`, to safely initialise memory, stack, and interrupts.
- On the Pico specifically: **C SDK** (libraries/abstraction) + **compiler** (source → .uf2/.bin) + **bootloader** (flashes and runs the binary) work together to get your code running.
- **Firmware-critical C:** use fixed-width types to avoid overflow/truncation bugs; use masks (`|=`, `&= ~`, `^=`) to touch only intended register bits; use `volatile` for anything hardware- or ISR-changed (but remember it does **not** provide atomicity); use correctly-typed pointers for memory-mapped I/O; prefer `static` for encapsulated state over uncontrolled globals; and never assume a struct's memory layout without accounting for padding/alignment.
