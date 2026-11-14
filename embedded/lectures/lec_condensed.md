# INF2004 Quiz Study Guide (Lectures 1 to 4)

Quiz format: 50 questions, 30 minutes. That is about 36 seconds per question, so you need to recognize answers fast, not derive them.

**What the quiz bank tells us about the style:**
- Mostly single-answer multiple choice, some "select all that apply", some true/false, a few numeric fill-ins.
- Lots of definition questions ("What is the purpose of the IVT?", "A transducer is...").
- Lots of true/false traps built from one wrong word ("level-triggered can be rising or falling edge", "SPI master can only have one slave").
- A handful of calculations: baud rate/frame time, duty cycle, timer tick count, ADC code, utilization.
- Week 4 questions (RTOS, FSM) lean toward "why" and "which is the best fit".

Sections marked **Quiz hook** are things that showed up in (or look like) the sample quiz.

---

# LECTURE 1: Intro to Embedded Systems

## 1.1 What is an embedded system?

- A computing system dedicated to a **specific task inside a larger** electrical or mechanical system.
- Common thread in every definition: **dedicated, integrated, often has timing constraints**.
- Embedded systems are the building block of **CPS** (cyber-physical systems: computation + networking + physical processes) and **IoT** (connected everyday devices).

**Properties of embedded systems:**
- Embedded as part of a device
- Dedicated to specific tasks (fixed at design time)
- Use **sensors** (input) and **actuators** (output)
- **Strict timing constraints**
- **Limited resources** (memory, CPU, energy)
- Based on **microcontrollers**

> [!tip] Quiz hook
> "Which are properties of ES?" Correct: microcontroller-based, timing constraints, limited resources. Wrong: infinite memory, unlimited processing power, general purpose.

## 1.2 Microcontroller vs single-board computer vs general-purpose computer

| Type | Idea |
|---|---|
| Microcontroller | Minimalist, bare chip, one dedicated job |
| Single-board computer | Compact, versatile, full small computer (Raspberry Pi) |
| General-purpose computer | Powerful, expandable desktop/workstation |

## 1.3 Analogue vs digital

- Real world is **analogue and continuous**. Computers are **digital and discrete**.
- **ADC** converts analogue to digital. **DAC** converts digital to analogue.
- A **transducer** converts a signal from one form to another (e.g. light to voltage). It does not always cross the analogue/digital boundary.

## 1.4 Real-time

- **Real-time does NOT mean fast.** Correctness depends on **both the result and the time it is delivered**. A late answer is a wrong answer.
- Classify by what happens if a deadline is missed:

| Type | Missed deadline | Examples |
|---|---|---|
| Non real-time | No deadlines, no danger | TV remote, lawn sprinkler, window lifter |
| **Soft** real-time | Occasional misses OK, performance **degrades**, no real danger | Mobile phone, air-con, fridge |
| **Hard** real-time | Must **always** meet deadline, miss can be **catastrophic** | Nuclear reactor control, airbag, flight control |

> [!tip] Quiz hook
> "Hard real-time means a missed deadline can have extreme consequences." "Late response degrades the system" is **soft**. "No deadlines" is **non**-real-time.

## 1.5 Hardware options

| Hardware | One-line meaning |
|---|---|
| **µC** (microcontroller) | CPU + memory + peripherals on one chip |
| **µP** (microprocessor) | The CPU core alone, without integrated memory/peripherals |
| **SoC** | Multiple (usually higher-end) cores + GPU + multimedia + peripherals on one die. Basically several embedded systems in one chip |
| **ASIC** | Custom chip hard-wired for one application, cannot be reprogrammed |
| **NPU** | Accelerates machine learning / neural networks |
| **DSP** | Fast repetitive math on signals (audio, sensors) |
| **FPGA** | Reconfigurable hardware logic, you "rewire" the circuit itself |
| **GPU** | Massively parallel computation (graphics, now ML too) |

- **Microcontroller = microprocessor + more integrated features** (flash, SRAM, GPIO, DMA, timers, WDT, RTC, UART/SPI/I2C, ADC/DAC...).
- **Microcontroller vs SoC:** a µC has one or two modest cores (e.g. Cortex-M3). An SoC has multiple, higher-end cores (e.g. Cortex-A9).

> [!tip] Quiz hook
> "Which are typically used for machine learning?" Think NPU, GPU (and FPGA).

## 1.6 Why embedded software quality matters

Core ideas:
- ubiquitous
- mission-critical
- core competency
- unique challenges (limited memory/power, must not crash)
- Be **proactive** (test during development, because you can't patch an airbag after the car is sold)
- Follow **rigorous standards**.

**Embedded challenges:** 
- resource constraints
- real-time requirements
- hardware-software co-design
- concurrency and interrupts
- reliability and safety
- **debugging difficulty**.

## 1.7 SDLC and languages

- **SDLC** = structured approach that ensures repeatability and quality. Critical in embedded due to safety, timing, resource limits.
- **Low-level language** (assembly): best performance, **compact and optimized**.
- **High-level language** (C): better **productivity, abstraction, portability**. Needs a compiler.
- Only a machine understands machine language.

> [!tip] Quiz hook
> "Main characteristic of low-level language?" Compact and optimised. (Not productivity, not easy to understand.)

## 1.8 Compiler, assembler, linker, loader

- **Compiler:** translates high-level code (C) into **object files** (opcodes for a target processor). Examples: GCC, Keil, IAR, CCS.
  - **Cross compiling:** build on one machine (host) for a different target (e.g. x86 laptop to ARM MCU).
  - **Native compiling:** build for the same architecture you run on.
- **Assembler:** translates assembly source (.s) into an **object file**. **One-to-one** translation (one assembly instruction = one machine instruction). It does **not** optimise source code and is **not** the same as a compiler.
- **Linker:** combines object files into **one executable/binary**. Resolves cross-file references. Links static (copied into binary) and dynamic libraries. **Every link is controlled by a linker script** (.ld) that describes how the executable maps onto memory.
- **Loader/bootloader:** puts the binary into memory and runs it.

**Pipeline:**

```
.c files  --compiler-->  .o files  \
.s files  --assembler--> .o files   --> LINKER (+ C library/SDK + linker script .ld + startup crt0.s + reset.s) --> .bin --> loader --> memory
```

> [!tip] Quiz hook
> "Which files can be converted to an object file?" **C file and assembly file** (not hex, not "decoded"). "A compiler translates C into **object files**" (the linker makes the executable).

## 1.9 Startup code (crt0.s)

- First code that runs after reset, **before `main()`**. Small assembly block that prepares the way for C.
- Steps in order:
  1. **Disable interrupts**
  2. **Copy initialised data into RAM** (flash to RAM)
  3. **Zero the uninitialised data** (.bss)
  4. **Set up the stack**
  5. **Set up the heap**
  6. **Enable interrupts**
  7. **Call `main`**
- Interrupts are off at the start because the system is fragile (stack/memory not ready).
- **Stack** = function call frames, locals, return addresses. **Heap** = dynamic memory (`malloc`/`free`).

## 1.10 Raspberry Pi Pico toolchain

- **C SDK:** libraries and tools that abstract the low-level hardware (GPIO, PWM, UART...).
- **Compiler:** C source to executable binary (**.uf2** or **.bin**).
- **Bootloader:** small program **pre-installed on the Pico**. Writes the new binary to flash (e.g. via USB). If no new binary, it runs the existing program in flash.
- Flow: source code, via C SDK, to compiler, to .uf2, to bootloader, to running program.

## 1.11 Firmware-critical C

**Core idea:** a hardware register is just bits with consequences. Four themes: **Representation, Bit control, Memory, Visibility.**

**Integers**
- Use fixed-width types from `<stdint.h>`: `uint8_t`, `int16_t`, `uint32_t`.
- Unsigned for bit patterns and counters. Signed only when negative is meaningful.
- Traps: **Overflow** (`uint8_t x = 255; x++;` gives 0). **Truncation** (`uint8_t y = 300;` keeps lower 8 bits). **Mixed signed/unsigned** comparisons give surprising results.
- Typical sizes: `char` 1 byte, `short` 2 bytes, `int` 4 bytes (on 32-bit ARM), `long long` 8 bytes.

**Operators (high quiz value)**
- `&&` logical AND, `||` logical OR, `&` bitwise AND, `|` bitwise OR, `^` is **XOR (not exponentiation)**, `~` bitwise NOT.

~ not example: ~5 = ~ 00000101 = 11111010 = 250
! is Logical Not: !x, if x == 0, returns true.

- Logical operators return 0 or 1. `17 && 23` = **1**. `17 && 0` = **0**.
- Shift left by n = multiply by 2^n. `0x06 << 1` = `0x0C`.
- shift right by n = divide by 2^n 

**Bit manipulation**
```c
#define BIT(n) (1u << (n))
REG |=  BIT(3);      // set
REG &= ~BIT(3);      // clear
	BIT = 0000 0100, &= here would do REG AND 1111 1011, this clears bit 3.
REG ^=  BIT(3);      // toggle
if (REG & BIT(3)) {} // test
```
- Mask: a `1` selects the bit, a `0` leaves it alone.
- To keep the upper 8 bits of a 16-bit value: **bitwise AND with `0xFF00`**.
- To read bit 5: build the mask `(1 << 5)` and AND it with the register.
- Prefer `REG |= BIT(3)` over `REG = 0x08` (the second destroys all other bits).
- Use parentheses to make precedence explicit.

**Qualifiers**

| Qualifier | Meaning |
|---|---|
| `const` | Program must not modify it (lookup tables, read-only config) |
| `volatile` | Value may change **outside normal code flow** (hardware register, ISR-shared flag). Every access happens exactly as written |
| `const volatile` | Software only reads it, hardware may still change it (read-only status register) |

- **`volatile` does NOT make anything atomic.** Shared data between ISR and main still needs interrupt masking, atomics, or an RTOS primitive.

**Pointers and memory-mapped I/O**
- Address, to typed pointer (`volatile uint32_t *`), to register value.
- `#define GPIO_OUT (*(volatile uint32_t *)0x40021000u)`
- **Wrong pointer = wrong hardware.** Type controls how many bytes are accessed.

**Scope, lifetime, linkage**

| Storage | Scope | Lifetime |
|---|---|---|
| Local | Block/function | One call |
| `static` local | One function | Whole program (keeps value between calls) |
| `static` file-scope | One file | Whole program (hides module state) |

- `extern` only when sharing is intentional. Prefer private module state.

**Structs**
- Compiler may add **padding** for **alignment**, so `sizeof(struct)` can be bigger than the sum of fields.
- **Packing** removes padding but can cause slow or invalid unaligned access.
- Don't send raw structs across a bus/network. Use explicit serialisation (byte order/endianness differs).

**General C good practice:** small functions, bounded loops, name by intent, include guards, prefer typed constants/inline functions over macros, one coding standard, **measure before optimising**.

---

# LECTURE 2: GPIO and Interrupts

## 2.1 GPIO basics

- **GPIO = General Purpose Input Output.** Each pin carries one digital state (about 0 V or 3.3 V on the Pico).
- **"The circuit sets the voltage. The program gives it meaning."**
- **Direction:** input **senses**, output **drives**. Choose direction before reading/driving.
- Each pin's direction is set **individually**. An 8-bit port does NOT need all pins as output or all as input.
- Logic ranges: HIGH above **VIH(min)**, LOW below **VIL(max)**, undefined zone in between. Check the datasheet.
- **Common ground** is required when connecting two circuits (voltage is relative).
- GPIO can do input and output without extra components, but it is a **limited current source/sink, not a power supply**. Never drive a motor directly.
- GPIO pins do **not** always need an external resistor (the MCU has internal pull-ups/pull-downs). An **LED** does need a current-limiting resistor.

**LED current:** `I = (Vsupply - Vled) / R`, e.g. (3.3 - 2.0) / 1k = **1.3 mA**.

## 2.2 Floating pins, pull-ups, pull-downs

- **Floating pin:** an input with nothing driving it. Undefined voltage, picks up noise. **"Nothing connected" is NOT logic 0.**
- Fix with a pull resistor.

| | Open switch | Pressed | Convention |
|---|---|---|---|
| **Pull-up** (to HIGH) | 1 | 0 | **Active-low** |
| **Pull-down** (to GND) | 0 | 1 | **Active-high** |

- The circuit defines the convention. There is no universal "pressed = 1".
- **Push-pull output:** actively drives toward either supply rail (write 1 to the supply voltage, write 0 to GND).

## 2.3 Pin multiplexing

- One physical pin has many possible functions (GPIO, UART, SPI, PWM, PIO...). A **function select** register picks the owner.
- Select exactly **one** owner per pin and check the pin supports it.
- **Multiplex matrix for buttons:** saves pins. N x M switches need only **N + M** GPIO pins instead of N x M.

## 2.4 Registers and masks

- Pin hardware is exposed through registers: **OUT latch** (drives pin), **IN sample** (reads pin), **OE/DIR** (direction).

| Intent | Code | Other bits |
|---|---|---|
| Set bit | `OUT \|= mask` | Preserved |
| Clear bit | `OUT &= ~mask` | Preserved |
| Toggle bit | `OUT ^= mask` | Preserved |
| Replace | `OUT = mask` | **Destroyed** |

- Worked example: `0xA5 | 0x08 = 0xAD` (sets bit 3).
- Read-modify-write on a register shared with an ISR needs an **atomic strategy**.

## 2.5 Configuring GPIO

Sequence: **IDENTIFY, PRELOAD (safe level first), ENABLE, VERIFY.**

```c
// Output
gpio_init(15);
gpio_put(15, 0);
gpio_set_dir(15, GPIO_OUT);

// Input with pull-up
gpio_init(2);
gpio_set_dir(2, GPIO_IN);
gpio_pull_up(2);
```
- Configure once at startup, then read/write in the main loop.
- Software GPIO number (GP2) is not necessarily the physical header pin number.

## 2.6 Edges, bounce, debounce

- **Falling edge** = HIGH to LOW. **Rising edge** = LOW to HIGH. Count a **transition**, not every loop that sees the level.
- **Switch bounce:** mechanical contacts make many rapid edges for one press (a few ms).
- **Debounce:** accept a new state only once it has been **stable for a minimum time** (about **30 ms**; typically tens of ms, not 200+ ms). Algorithm: sample, if changed restart the timer, once stable accept.
- Debouncing in an ISR still needs time and recorded state, not a `delay()` inside the ISR.
- A shorter polling loop reduces blind time but does not guarantee catching short pulses.
- **Handshaking** over GPIO: sender puts data on DATA, asserts **REQ**; receiver captures, asserts **ACK**. Hold data until accepted. Define polarity and a timeout.

## 2.7 PIO (Programmable I/O)

- RP2040/RP2350 feature: tiny independent processors that build **custom hardware interfaces**, run **separately from the CPU**.
- **9 instructions:** `jmp, wait, in, out, push, pull, mov, irq, set`.
- RP2040: **2 PIO blocks, 4 state machines each.**
- Removes CPU-heavy **bit-banging**.
- Use: **SELECT** a state machine, **CONFIGURE** (load program, map pins, set clock divider), **RUN**.

## 2.8 Why interrupts exist

- CPU is busy and a short event happens. How do we not miss it?
- **Polling:** CPU repeatedly asks. Can miss short unlatched pulses.
- **Interrupt:** hardware signals the CPU when the event occurs.
- **An interrupt is not "always a bad thing".** They let the system respond immediately to critical events.
- Interrupts are **not** simple to debug (asynchronous, timing-dependent).

**Formal model**
- T = min time between events, D = deadline, p = polling period, c = CPU time per event, h = hardware latency.
- Polling: **2c <= c + p <= D <= T**, utilisation **u = c / p**.
- Interrupt: **h + c <= D <= T**, utilisation **u = (h + c) / T**.
- Interrupts are usually far more efficient for sparse events.

**Pick the mechanism**

| POLL | INTERRUPT | DMA + INTERRUPT |
|---|---|---|
| Short predictable wait | Sporadic, time-sensitive events | Sustained data streams |

> [!tip] Quiz hook
> "Event with a very short time between occurrences: best choice?" Think about whether the processing fits. For sustained high-rate data, DMA.

## 2.9 Interrupt anatomy

- **Source** (the cause, e.g. timer match), **IRQ** (the request), **ISR** (the code). "A peripheral raises an IRQ. An ISR services it."
- Three phases: **entry (save context), service, return (restore context)**.
- **Three gates** must be open for an ISR to run: **source enabled, controller (NVIC) enabled, CPU global mask/priority allows.**
- **Pending is not running.** Masking doesn't necessarily erase a pending event.
- **Interrupt controller** manages enable, pending, priority.
- **NMI** (non-maskable interrupt): cannot be disabled.

## 2.10 IVT, stack, context

- **IVT (Interrupt Vector Table):** a table at a fixed location holding the **entry point (branch/pointer) for each interrupt/reset type**. When an interrupt occurs, the CPU jumps to that vector. ISR code can live anywhere.
- **RESET** forces the PC to the **reset vector** in the IVT.
- CPU cycle: fetch from PC, execute, increment PC.
- **Context save:** CPU automatically saves **PC and status register (SR)** (and more). Cortex-M pushes **xPSR, PC, LR, R12, R3, R2, R1, R0**.
- **Stack:** RAM region. A CPU **pushes** data onto the stack and later must **pop** it off (reverse order). **SP (stack pointer)** points to the most recently pushed item. Un-popped data corrupts the stack.
- **Full sequence:** event, push context on stack, branch to IVT entry, branch to ISR, ISR restores context (pop), return and resume. Summary: **EVENT, REQUEST, SAVE, SERVICE, RESUME.**

## 2.11 Edges vs levels, pending bit

- **Edge** = a transition (moment). **Level** = a condition that persists. They are different, so "level-triggered interrupts can be rising or falling edge" is **false**.
- **One pending bit is not an event queue.** Three quick edges can produce one service. To count every event, use a hardware counter, FIFO, or bounded queue.
- **Service the cause before returning:** timer (ack status flag), UART RX (drain FIFO), GPIO level (remove condition or mask).
- **You must clear the interrupt flag**, otherwise the ISR fires again immediately. Clear it **before exiting** the ISR (after servicing the cause).

## 2.12 Writing a good ISR

- **Tiny, fast, efficient.** Disturb the rest of the system as little as possible.
- **Never** put an infinite loop or `printf` in an ISR.
- ISRs use **global variables** to communicate (cannot pass normal parameters).
- Interrupts can be **nested**. **Priority levels** decide order. A higher-priority IRQ can **pre-empt** a lower-priority ISR. **"Priority chooses who waits. It does not create more CPU time."**
- **Capture now, process later:** ISR does **capture, acknowledge, notify**. Main loop does **parse, compute, update/log**.
- A simple shared **flag** is enough when events can coalesce. `take_and_clear()` must be **atomic** relative to the ISR.

**Race condition example:** main reads count=7, an interrupt increments to 8, main writes back 7+1=8. Expected 9, got 8. **`volatile` is not atomicity.**

## 2.13 Setting up an interrupt (configure first, enable last)

1. Keep source disabled
2. Set pin/timer/trigger mode
3. Install handler, init shared state
4. Clear stale flags, set priority
5. **Then** enable the source and the NVIC

```c
irq_set_exclusive_handler(UART0_IRQ, uart0_isr);
irq_set_priority(UART0_IRQ, 0x40);
uart_set_irq_enables(uart0, true, false);
irq_set_enabled(UART0_IRQ, true);
```

- **Timer as interrupt source:** clock, counter, match, IRQ. There is **latency** between the match and the ISR actually starting.
- **ISR load = event rate x time per ISR**. 1 kHz x 20 us = **2%**. 20 kHz x 20 us = **40%**. Low average load does not prove deadlines are met.

## 2.14 Debugging chain and RP2040 details

- **EVENT? ENABLED? HANDLER? CLEARED?** (Enabled at source, controller, AND core.)
- Debug trick: toggle a spare GPIO at ISR entry/exit and watch on a scope/logic analyser.
- **Tail-chaining** (Cortex-M0+): if a second interrupt is pending as the first finishes, skip the redundant unstack/restack (28 cycles down to 6, saves 22).
- **Late arrival:** a higher-priority IRQ arriving during stacking of a lower one is taken first.
- **RP2040 (Cortex-M0+) and RP2350 (Cortex-M33):** NVIC with a vector table. **RP2350 Hazard3 (RISC-V):** Xh3irq with trap entry and software dispatch. The model event, request, service, resume is the same.
- Portable SDK call: `gpio_set_irq_enabled_with_callback(...)`. Code must still be built for the specific target.

## 2.15 Interrupt sources (catalogue)

- GPIO/external (buttons, encoders)
- timer (scheduling, timeouts)
- PWM (wrap/fault)
- RTC (alarm, wake)
- ADC (result ready, threshold)
- UART (RX ready, TX space, framing error)
- SPI/I2C (FIFO, completion, bus error)
- USB/CAN/Ethernet
- **DMA** (block transfer complete)
- PIO.

- System events: 
	- software/inter-core IRQ
	- OS tick
	- power/clock/NMI
	- fault/SVC/ecall (synchronous exceptions)
	- **reset/watchdog** (a separate execution path, not a "return from interrupt").

---

# LECTURE 3: Serial Comms, Timers, PWM, ADC

## 3.1 Serial vs parallel

- **Parallel:** many wires, many bits at once, very fast, but wiring/pins/PCB cost scales badly.
- **Serial:** one data line, bits sequentially, **tiny footprint**. It is the standard for chip-to-chip communication.

**Three design variables of any serial protocol:**
1. **Timing:** synchronous (shared clock wire) vs asynchronous (no clock, agreed **baud rate**, start/stop bits)
2. **Data flow:** **simplex** (one way only), **half-duplex** (two way, one at a time), **full-duplex** (simultaneous, separate TX and RX lines)
3. **Topology:** **point-to-point** (1 to 1) vs **bus/multi-drop** (one controller, many peripherals, needs hardware select or software addressing)

- A **synchronous** link has a **separate clock wire**.

## 3.2 UART

- **Universal Asynchronous Receiver-Transmitter.** Technically a **circuit** that converts between parallel and serial, not a protocol.
- **Async, full-duplex (usually), point-to-point.** Speed about **9.6 to 115.2 kbps**. Two wires: **TX and RX, cross-wired** (TX to RX, RX to TX). No clock, no addressing.
- Uses: GPS, RFID, Bluetooth modules, debug consoles.
- Both sides must use the **same baud rate**.
- **Frame:** `idle (HIGH), start bit (LOW), data bits, parity (optional), stop bit(s), idle`.
- **Line idles HIGH. A drop to LOW (start bit) tells the receiver a character is about to be sent.**
- Data is sent **LSB first**.
- **Parity:** even parity makes the total count of 1s (including parity bit) even. A single parity bit **detects an odd number of bit errors (1, 3, 5...)**. It cannot detect even numbers of errors, cannot correct, and doesn't tell the transmitter to resend.
- 5 to 8 data bits, none/even/odd parity, 1 or 2 stop bits.

**Baud rate calculation (explicitly flagged as tested):**
- Bits per character = start + data + parity + stop. Example: 1 + 7 + 1 + 1 = **10 bits**.
- **Baud = characters/sec x bits per character.** 100 chars/s x 10 = **1000 baud**.
- **Frame time = bits per frame / baud.** 10 / 9600 = **1.042 ms**. 
- **Bit time = 1 / 9600 = 0.104 ms.**
## 3.3 SPI

- **Serial Peripheral Interface**, from **Motorola** (1980s).
- **Synchronous, full-duplex, bus (single master, multi-slave).** Speed **10 to 50+ MHz** (fastest of the three).
- Wires: **SCK, MOSI** (Master Out Slave In), **MISO** (Master In Slave Out), **CS/SS** (chip select).
- Acts like a **circular shift register**. Each clock: one bit out on MOSI, one bit in on MISO.
- **CS is active LOW.** Master selects one slave at a time. **An SPI master can have many slaves** (one CS line each).
- **No addressing**, no ACK, no parity.
- **SPI sends MSB first** (UART sends LSB first).
- Uses: displays (LCD/OLED), SD cards, fast ADCs. Fast because of a **dedicated clock and push-pull drivers**.

**SPI modes**

| Mode | CPOL | CPHA | Idle clock | Sample edge |
|---|---|---|---|---|
| 0 | 0 | 0 | Low | Rising |
| 1 | 0 | 1 | Low | Falling |
| 2 | 1 | 0 | High | Falling |
| 3 | 1 | 1 | High | Rising |

- **CPOL** = idle level of clock. **CPHA** = sample on first or second edge. Master must match the slave.
- RP2040: 2 SPI, 4 to 16 bit frames, 8-entry FIFOs.

## 3.4 I2C

- **Inter-Integrated Circuit**, invented by **Philips**.
- **Synchronous, half-duplex, bus (multi-master, multi-slave).** Speed **100 kHz (standard), 400 kHz (fast)**, up to 1 MHz (fast mode plus) on RP2040.
- **Exactly two wires: SCL (clock) and SDA (data).** Over 100 devices possible. **No chip select pins.** It does **not** have separate MOSI/MISO wires.
- **Open-drain** lines pulled HIGH by **external pull-up resistors**. Devices only pull LOW. Pull-ups plus **bus capacitance** limit speed.
- **Addressing:** master sends a **7-bit address** (RP2040 also supports 10-bit). Matching slave replies with **ACK**.
- **Packet:** `START, 7-bit address, R/W bit, ACK, 8-bit data, ACK, STOP`.
- **Clock stretching:** a slow slave holds **SCL LOW** and the master waits.
- **Arbitration:** two masters transmit; open-drain means LOW wins. A master sending 1 but seeing 0 backs off.
- Uses: low-bandwidth sensors (temperature, humidity, accelerometer), RTCs.
- RP2040: 2 I2C, 16-entry FIFOs.

## 3.5 Comparison (memorise)

| | UART | SPI | I2C |
|---|---|---|---|
| Signals | TX, RX | SCK, MOSI, MISO, CS | SCL, SDA |
| Clock | **No (async)** | Yes | Yes |
| Duplex | Full | Full | **Half** |
| Addressing | **No** | No (uses CS) | **Yes (7/10-bit)** |
| Multiple devices | Difficult | Via CS lines | Native |
| Parity / stop bits | **Yes** | No | No |
| ACK/NACK | No | No | **Yes** |
| Speed | Slow | **Fastest** | Medium |
| FIFO (RP2040) | 32-byte | 8-entry | 16-entry |
| Multi-master | No | Possible, uncommon | **Supported** |

> [!tip] Quiz hook
> "Which protocol is asynchronous?" **UART.** 
> "Do we need a slave address for UART?" **No.** 
> "I2C has MOSI/MISO" **false.** 
> "SPI master only one slave" **false.** 
> "Half-duplex means only one side transmits at a time."

## 3.6 Timers

- **Blocking delays** (`sleep()`/`delay()`) trap the CPU in an empty loop. No multitasking, can't react to anything.
- **Hardware timers** count independently, so the **CPU can execute other code while the timer runs concurrently**.
- Inside a microcontroller, a timer is a **counter driven by a clock** (counting clock ticks).
- **Ticks = frequency x time.** A 1 second timer from a 30 MHz clock needs **30,000,000** counts (30 MHz x 1 s).
- **Prescaler:** divides the fast system clock before the counter. Example: 125 MHz / 125 = **1 MHz** tick (1 tick = 1 us). Stops a 16-bit counter overflowing too fast.
- **Counting modes:**
  - **Up-counting (sawtooth):** 0 to max, snap back to 0. Like a stopwatch.
  - **Down-counting:** load max, count to 0, reload. Like a microwave timer. In **free-running** down-count mode the counter wraps back to the **maximum** after 0.
  - **Center-aligned (up/down):** up to max then back down, a **triangle wave**. Used for **symmetrical PWM and motor control**, reduces EMI.
- **Interrupt triggers:**
  - **Overflow:** counter hits its limit and rolls over.
  - **Compare match:** counter equals a value in the **compare register**. Triggers at exact intervals, before overflow.
- **Input capture:** pin edge makes hardware **copy the counter into a capture register** instantly. Measures external events precisely (motor RPM via Hall sensor, IR remote, RC pulse width). **Pulse width = T2 - T1.**
- **Timer-triggered DMA:** timer overflow (e.g. 44.1 kHz) signals the DMA controller, which moves an ADC sample to RAM with **no CPU involvement**. DMA = Direct Memory Access.
- **Timer cascading (odometer):** Timer A overflow clocks Timer B. Two 16-bit timers make an effective **32-bit** timer. (Pico has a native 64-bit timer.)
- **Timer types:**
  - **General purpose:** delays, PWM, waveforms, capture.
  - **RTC:** keeps human time (year to second), ultra-low-power domain, often coin-cell backed. A standard RTC crystal is **32.768 kHz** (2^15 Hz), divided down to **1 Hz**.
  - **Watchdog (WDT):** software must regularly "pet/kick" it. If the code hangs, it hits zero and forces a **hardware reset**.

**Timer traps**

| Trap | Cause | Fix |
|---|---|---|
| **Off-by-one (N-1 rule)** | Registers are zero-indexed, writing 100 divides by 101 | Write **target - 1** |
| **Infinite ISR** | Forgot to clear the timer interrupt flag, ISR re-enters forever | Clear the flag in the ISR |
| **Clock tree illusion** | Assuming timer runs at CPU clock, but peripheral bus has own divider | Check the actual peripheral clock |
| **Runaway timer** | Changing compare target while running, timer already passed it, counts to max and wraps | **Preload/shadow registers** |
| **Torn read** | Multi-byte timer read as two bus reads, rollover in between gives half old, half new (real 0x0100 read as 0x01FF) | Disable interrupts or use hardware lock on high byte |

## 3.7 PWM pulse width modulation

- One digital output switching between HIGH and LOW.
- **Frequency** `f = 1 / T`. **Duty cycle** `D = Ton / T x 100%`. "Frequency sets the rhythm, duty cycle sets the ON share."
- **Average drive = D x supply.**
- **PWM = fake analogue.** The pin stays digital. A load's inertia or filter averages the pulses.
- Why use it: efficient (switch fully ON or OFF wastes less heat than a linear control). Uses: LED brightness, motor speed, power stages.
- **Duty-cycle calculations:** `Period = Ton / duty`. If Ton = 30 ms and duty = 40%, period = **75 ms**. If Ton = 333 ms, period = **832.5 ms**.
- **Two ways to build PWM:**
  - **Dedicated PWM peripheral:** multiple synchronised channels, complementary outputs, **dead-time insertion**, fault/brake inputs. Best for motor control and power conversion.
  - **General-purpose timer output-compare:** best for LEDs, buzzers, servos.
  - Same mechanism underneath: **clocked counter + period + compare + output logic**.
- **Timer to PWM:** **ARR** (auto-reload/period register) sets **frequency**. **CCR** (compare register) sets **duty cycle**.
  - `f_PWM = f_timer / ((PSC+1)(ARR+1))`
  - `D = CCR / (ARR+1)`

## 3.8 ADC

- **12-bit ADC (RP2040):** `N = 4095 x Vin / VREF`. Max 4095 (Vin >= VREF), min 0.
- If the input rises above Vref, the result is **clipped** (saturates at the max code). Inputs must stay between GND and ADC_AVDD.
- **LSB** = VREF / 4096 = 3.3 / 4096 = **0.805 mV**. **Max quantisation error = +/- 0.5 LSB** (about +/-0.403 mV).
- Worked: 1.23 V gives 4095 x 1.23 / 3.3 = **1526**.
- Inside an ADC: **sampling** (discretise in time) and **quantisation** (discretise amplitude into levels). Transduction is done by the sensor, not the ADC.
- **Quantisation** = mapping a continuous value onto a finite number of discrete levels (resolution). It is **irreversible and results in information loss**.
- **RP2040 ADC:** one **12-bit SAR** converter behind a **5-input mux** (GPIO26 to 29 and internal temperature sensor), up to **500 kSPS**, 48 MHz ADC clock, 96 cycles per conversion = **2 us**, **8-entry FIFO**, DMA support.
- **Pipeline: SELECT (AINSEL), ACQUIRE, CONVERT (successive approximation, one bit at a time), PUBLISH (RESULT or FIFO).**
- **Resolution is not accuracy.** RP2040 is about **8.7 ENOB** (effective number of bits). Reasons: reference noise, offset/gain error, nonlinearity, noise.
- **Nyquist:** sample at **fs > 2 x fmax**. Sampling at exactly 2x (**critical sampling**) is dangerous, you can land on zero crossings and read a flat DC line. **Oversample 2.5x to 3x** and trigger with a **hardware timer** (not software delay).
- **CD-quality audio: 44.1 kHz, 16-bit.**
- **Analogue front end:** protect (stay within 0 to ADC_AVDD), **low-pass filter before sampling** (aliasing can't be fixed in software), stable reference/ground. `adc_gpio_init()` sets high impedance, pulls off, digital input off.

**ADC symptom table**

| Symptom | Cause | Fix |
|---|---|---|
| Codes wander at rest | Floating input / noisy reference | Defined source, better grounding |
| First sample wrong | Source didn't settle after channel change | Settling time, buffer |
| False frequency | Aliasing | Analogue low-pass filter, raise sample rate |
| Clips at an end | Input out of range | Scale and protect |
| Channels affect each other | Shared converter, incomplete settling | Slow scan, discard a sample |

**Calibration (two-point):**
- `Gain = (Vhi - Vlo) / (ADChi - ADClo)`
- `Offset = Vlo - Gain x ADClo`
- `Vactual = ADC x Gain + Offset`
- Do it in a **task, not an ISR**. Use fixed-point math.
- Offset error = curve shifted. Gain error = slope wrong.

**RP2040 ADC registers**
- **ADC_CS:** `EN` (bit 0), `TS_EN` (bit 1, temp sensor), `START_ONCE` (bit 2, **self-clears**), `START_MANY` (bit 3, continuous), `READY` (bit 8), `AINSEL` (bits 14:12, 0 to 3 = GPIO26 to 29, 4 = temp sensor).
- **RESULT:** latest conversion, for polling with START_ONCE. Reading it does **not** clear interrupts.
- **FIFO + FCS:** for continuous capture. Reading the FIFO **removes the oldest** sample.
- **FIFO threshold trade-off:** 1 = lowest latency but highest IRQ rate. 4 = balanced. 8 = fewest interrupts but **no overflow margin**.
- **Failure flags:** **OVER** (ADC outran consumer), **UNDER** (read empty FIFO), **ERR** (conversion error).
- **Design rule:** average drain rate must exceed sample rate, and buffer must absorb the longest stall.
- **Golden rule:** polling for occasional reads, IRQ for moderate streams, **DMA for sustained high-rate capture.**
- Common failures: long ISR (`printf`, float), threshold = 1 by habit, reading one sample per ISR call, `sleep_ms` timing jitter, no software buffer, unsafe shared data.

---

# LECTURE 4: RTOS and State Machines

## 4.1 Real-time = predictable

- Correctness = result **and** timing. **Worst-case latency and bounded blocking matter more than average throughput.**
- The RTOS gives mechanisms, but **the designer still has to prove the timing**.
- Ask: "What is the **latest acceptable** response?" (not the average).
- An RTOS: splits work into **tasks**, **waits** instead of polling, **protects** shared resources, uses **kernel messaging**, **controls timing**.

**Six time-critical kernel services:**
1. Task scheduling
2. Task synchronisation (critical sections, semaphores, mutexes)
3. Inter-task communication
4. Managing real-time events (interrupts)
5. Real-time clocks, timers, delays
6. Memory management (avoid leaks and fragmentation so it runs indefinitely)

**μT-Kernel 3.0:** small embedded/IoT RTOS, compact, portable, **IEEE 2050-2018**. API pattern: **`tk_<operation>_<object>`**.
- Operations: `cre` create, `sta` start, `wai` wait, `sig` signal, `loc` lock, `unl` unlock, `snd` send, `rcv` receive, `dly` delay, `slp` sleep, `set`, `get`, `rel` release.
- Objects: `tsk` task, `sem` semaphore, `mtx` mutex, `flg` event flag, `mbf` message buffer, `mbx` mailbox, `mpf` fixed memory pool, `cyc` cyclic handler, `alm` alarm handler.

## 4.2 Tasks vs functions, task states

- **Function:** runs when called, then returns. **Task:** independent flow of control that **normally never returns** (`for(;;)`), waits when idle (BLOCKED, uses no CPU). Task functions typically do **not** call `return()`.
- **Task is the smallest unit of instructions the OS can manage independently.**

| From | To | Trigger |
|---|---|---|
| READY | RUNNING | Scheduler picks highest-priority READY task |
| RUNNING | READY | **Pre-empted** by a higher-priority task |
| RUNNING | BLOCKED | Waiting on semaphore, delay, message... |
| BLOCKED | READY | The thing it waited for happened |
| RUNNING | TERMINATED | Task exits |
| READY/BLOCKED | TERMINATED | Task deleted |

- Only **one task per core** is RUNNING. **TERMINATED is the state a task spends the least time in** (reached once).
- Blocking decides **who is READY**. Scheduling decides **which READY task runs**.

## 4.3 Scheduling and context switch

- **Preemptive priority scheduling:** a higher-priority task becoming READY preempts the running one immediately.
- **Equal priority:** FIFO or **time slicing (round robin)**. (Quiz: three tasks created with the same priority means round-robin.)
- **Priority encodes urgency (tightest deadline), not human "importance".**
- **TCB (Task Control Block)** stores task state (registers, stack pointer, priority, state).
- **Context switch:** save state to TCB, choose next READY task, restore its state. **Cost must be bounded and predictable.**
- Scheduling algorithms (from the quiz bank): **static priority** = RMS (shorter period gets higher priority), DMS (shorter relative deadline). **Dynamic priority** = **EDF** (nearest absolute deadline), **LST** (least slack time).

## 4.4 Race conditions and critical sections

- Two tasks both read 0, add 1, write 1. Expected 2, got 1. **The scheduler didn't cause the bug, it exposed missing synchronisation.** `volatile` does not make it atomic.
- **Keep critical sections short.** While holding protection: **don't block, allocate memory, log, or do slow I/O.** A long critical section on a low-priority task delays high-priority ones.

## 4.5 Synchronisation and communication primitives

| Primitive | Represents | Payload | Owner | Use when |
|---|---|---|---|---|
| **Semaphore** | Count | None | **No owner** (anyone can signal) | N identical resources, producer/consumer |
| **Mutex** | Exclusive ownership | None | **Lock owner only unlocks** | Protect one shared invariant |
| **Event flag** | Condition bits | Bit pattern | None | Wait on ANY/ALL conditions |
| **Message buffer** | Queued bytes | **Copied** data | Kernel owns copy | Variable-size messages, simple lifetime |
| **Mailbox** | Queued **pointers** | Message pointer | Application defined | Large messages, zero-copy |
| **Fixed memory pool** | Fixed blocks | Memory block | Borrower returns | Bounded allocation, no fragmentation |

- **Semaphore:** `tk_wai_sem` (count >= request: decrement, else **block**), `tk_sig_sem` (increment or release a waiter). A consumer **sleeps until data arrives**, no polling.
- **Mutex:** `tk_loc_mtx`, `tk_unl_mtx`. Mutex = ownership + exclusion + wait queue.
- **Event flag:** `tk_set_flg`, `tk_wai_flg`. **`TWF_ORW` = wake on ANY bit**, **`TWF_ANDW` = wake when ALL bits set**. For conditions, not payloads.
- **Message buffer:** `tk_snd_mbf`/`tk_rcv_mbf`. Copy cost grows with size.
- **Mailbox:** `tk_snd_mbx`/`tk_rcv_mbx`. Fast, but **lifetime and who frees the memory** become the hard problem.
- **Fixed pool:** `tk_get_mpf`/`tk_rel_mpf`. Bounded time, **no external fragmentation**. Trade-off: fixed block size wastes space, capacity must cover worst case. This is why RTOS prefers pools over `malloc`.
- Start with semantics, optimise copying later.

## 4.6 Priority inversion

1. Low-priority **L** holds the mutex
2. High-priority **H** waits for it
3. Medium **M** preempts L
4. L can't release, so **H waits indirectly on M**

- **Two conditions needed:** **preemptive multitasking** and a **shared resource** (mutex).
- **Fix: priority inheritance.** L temporarily takes H's priority, M can't preempt, L releases, drops back. μT-Kernel: **`TA_INHERIT`** (or **`TA_CEILING`** = priority ceiling).
- Mars Pathfinder (1997) famously reset repeatedly because of this.

## 4.7 Time services

| Service | API | Behaviour |
|---|---|---|
| **Delay** | `tk_dly_tsk` | Task waits, CPU free for others |
| **Cyclic** | `tk_cre_cyc`/`tk_sta_cyc` | **Periodic release** at fixed rate |
| **Alarm** | `tk_cre_alm`/`tk_sta_alm` | **One-shot** callback at a set time |

- **Cyclic beats delay loops** because delay after each run drifts (work time adds to the period).
- **Build timeouts into the normal design.** Every blocking call has `tmout`. `TMO_FEVR` = wait forever, `TMO_POL` = poll without waiting.
- Cyclic/alarm handlers run like interrupt handlers, so keep them short and just signal a task.

## 4.8 ISR in an RTOS

- ISR: **clear the cause, capture the minimum data, wake a task.** Avoid **blocking, long loops, unbounded work.**
- Blocking calls are not allowed in an ISR. Check the allowed calling context (task vs handler).
- Same "capture now, process later" pattern.

## 4.9 Sensor pipeline example

`ADC ISR --event flag--> Acquire --message buffer--> Filter --mailbox pointer--> Control --semaphore--> Logger`
- Cyclic release 1 ms sets the sampling period. Fixed pool supplies mailbox blocks. **I2C mutex** protects the shared bus.

**For every object define:** invariant, owner, capacity, timeout, worst-case blocking. **Three checks:** mutex or semaphore? Copied data or pointer? What blocks the high-priority task?

## 4.10 Finite state machines (FSM)

- **Correct response = current state + event.** The same event can mean different things in different states.
- **Five parts:** **State** (what mode), **Event** (what happened), **Guard** (is it allowed? a condition that must be true), **Action** (what the system does), **Transition** (where it goes next).
- The filled dot with an arrow marks the **initial state**.
- Example: vending machine. WAITING, insert coin, store credit, HAS CREDIT. Choose snack, **guard: enough credit?**, release item, VENDING. If guard fails, the event is ignored and the state stays.

**Workflow:** CHOOSE, DESIGN, IMPLEMENT, TEST (exercise **every state/event pair**).

**Good fit:** reactive control, operating modes, handshakes, protocols, retries, timeouts, fault recovery. **Signal:** many boolean flags whose combinations secretly describe modes.
**Weak fit:** pure calculation, short linear algorithm, bulk data transformation, many independent activities (**state explosion**, up to 2^N states, so split independent concerns into separate FSMs). Rule of thumb: use an FSM when "what next?" needs **both event and history**.

**Event loop:** receive one event, decide using current state, act quickly, update state, return and wait.
**Never wait inside an action.** Start the work and treat **completion or timeout as a new event**.

**State table:** lists every state x event combination (4 states x 2 inputs = 8 rows). It doubles as your test plan. Example: BTN1 steps 00, 01, 10, 11 then wraps to 00. BTN2 = RESET goes to 00.

**Four implementations:**

| Style | Best for |
|---|---|
| **Lookup table** (`on_next[state]`) | Simple, regular sequences. Can't express guards easily |
| **Switch-case** (`switch(state)`) | **Guards, branches, non-sequential jumps** |
| **Superloop** | Turn inputs into events, call `dispatch(e)`, keep **servicing the watchdog**. Use debounced press events. Loop must stay short |
| **RTOS task + event queue** | One task owns the FSM, blocks on a queue, dispatches each event once |

- Guard example: `if (e == EV_START && ready()) state = S_RUN;`. If `EV_START` occurs while `ready()` is false, the **state stays the same** (no transition).
- **RTOS FSM rules:** **one task contains the FSM, do NOT make one task per state.** **ISRs and timers only enqueue an event, they never run the FSM.** One owner serialises events, so no mutex is needed for the FSM state.
- FreeRTOS names used in the example: `xQueueReceive`, `pdPASS`, `portMAX_DELAY`. Same pattern as μT-Kernel.
- **Filter + FSM:** filter math changes values, the FSM decides **when they can be trusted**. States: **RESET, WARM_UP, VALID, DEGRADED.** VALID enables the output gate. Warm-up matters because IIR/average filters start with empty history.
- **Avoid spaghetti:** **NAME** by stable behaviour (`CONNECTING`, not `STATE_AFTER_STEP_3`), **OWN** (one owner, explicit events, no hidden globals), **DEFINE** (short actions, every state/event result), **TEST** traces (repeat, timeout, invalid event, reset, recovery).

---

# QUICK FORMULA SHEET

| What | Formula | Example |
|---|---|---|
| UART baud | chars/s x bits per frame | 100 x 10 = 1000 baud |
| UART frame time | bits per frame / baud | 10 / 9600 = 1.042 ms |
| UART bit time | 1 / baud | 1 / 9600 = 0.104 ms |
| Timer ticks | f x t | 30 MHz x 1 s = 30,000,000 |
| Prescaled tick | f_sys / prescaler | 125 MHz / 125 = 1 MHz |
| Timer register value | target - 1 | divide by 100 means write 99 |
| Duty cycle | Ton / T x 100% | 30 ms / 75 ms = 40% |
| Period from duty | Ton / duty | 30 / 0.4 = 75 ms |
| PWM frequency | 1 / T, or f_timer / ((PSC+1)(ARR+1)) | |
| PWM duty (timer) | CCR / (ARR+1) | |
| Average PWM drive | D x supply | |
| Pulse width (capture) | T2 - T1 | |
| ADC code | 4095 x Vin / VREF | 4095 x 1.23 / 3.3 = 1526 |
| ADC LSB | VREF / 4096 | 3.3 / 4096 = 0.805 mV |
| Max quantisation error | +/- 0.5 LSB | +/-0.403 mV |
| Two-point gain | (Vhi - Vlo) / (ADChi - ADClo) | |
| Two-point offset | Vlo - Gain x ADClo | |
| Nyquist | fs > 2 x fmax (use 2.5x to 3x) | |
| Polling utilisation | c / p | |
| Interrupt utilisation | (h + c) / T | |
| Polling constraint | 2c <= c + p <= D <= T | |
| Interrupt constraint | h + c <= D <= T | |
| ISR load | event rate x time per ISR | 20 kHz x 20 us = 40% |
| LED current | (Vsupply - Vled) / R | (3.3 - 2.0) / 1k = 1.3 mA |
| Matrix keypad pins | N + M (not N x M) | |
| ASCII case | upper = lower - 0x20 (flip bit 5) | 'a' 0x61 to 'A' 0x41 |

---

# BITWISE QUICK DRILL

- Set bit n: `x |= (1u << n)`
- Clear bit n: `x &= ~(1u << n)`
- Toggle bit n: `x ^= (1u << n)`
- Test bit n: `x & (1u << n)`
- Keep upper byte of 16-bit: `x & 0xFF00`
- Keep lower byte: `x & 0x00FF`
- `0xA5` = `1010 0101`
- `0x06 << 1` = `0x0C`
- `17 && 23` = 1, `17 && 0` = 0, `17 & 23` = bitwise (17 = 10001, 23 = 10111, result 10001 = 17)

---

# TRUE / FALSE TRAP LIST

All of these are **FALSE** (or the opposite is true):

- "Real-time means fast." (It means predictable and on time.)
- "An interrupt is always a bad thing."
- "Interrupts are usually simple to debug."
- "Level-triggered interrupts can be falling or rising edge."
- "An 8-bit GPIO port must be all input or all output."
- "GPIO pins always need an external resistor."
- "Debounce with a delay of more than 200 ms."
- "Nothing connected to a pin means logic 0."
- "`volatile` makes an operation atomic."
- "`^` is exponentiation in C."
- "A compiler translates C straight into an executable." (Into object files. The linker makes the executable.)
- "An assembler is also called a compiler."
- "An SPI master can only have one slave."
- "I2C has separate MOSI and MISO wires."
- "UART needs a slave address."
- "UART uses a clock wire."
- "A single parity bit can correct errors" (it only detects odd-count errors).
- "Resolution equals accuracy."
- "Sample at exactly 2x fmax." (Oversample.)
- "A mutex can be unlocked by any task." (Only the owner. That's a semaphore.)
- "One task per FSM state."
- "An ISR should run the FSM directly." (It should only enqueue an event.)
- "An FSM action should wait for completion." (Completion is a new event.)

All of these are **TRUE**:

- A late answer is a wrong answer.
- SPI sends MSB first, UART sends LSB first.
- Synchronous links have a separate clock wire.
- Half-duplex = one side at a time.
- One pending bit is not an event queue.
- Pending is not the same as running.
- Quantisation is irreversible and loses information.
- The CPU can run other code while a hardware timer runs.
- Task functions normally don't `return`.
- Any task can signal a semaphore, but only the owner unlocks a mutex.
- Priority inversion needs preemptive multitasking and a shared resource.
- Same-priority tasks get round-robin time slicing.
- If a guard fails, the FSM stays in the current state.

---

# BUSES AND MEMORY (appeared in the quiz bank)

These were in the sample quiz, so skim them even though they are lighter in the lecture notes you uploaded:

- **Address bus:** CPU specifies **which location/register** to read or write.
- **Data bus:** carries the **actual data**.
- **Control bus:** carries control/timing signals, including the **read/write strobe** that tells a module whether to read or write.
- **Memory map:** tells the programmer **where registers sit in memory** (base and offset addresses of peripheral registers) and where flash/ROM/RAM regions are.
- **SP (stack pointer):** points to the most recently pushed item on the stack.
- **A CPU can push data onto a stack and later must pop it off.**

---