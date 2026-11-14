# INF2004 Embedded Systems Programming — Lecture 3 Study Notes
## Serial Digital Communication (UART/SPI/I2C) • Timers • PWM • ADC

---

## Learning Objectives

By the end of this lecture, you should be able to:

1. Explain why serial communication replaced parallel communication in most embedded systems, and describe the trade-off between the two.
2. Identify the three core design variables that define any serial protocol: timing, data flow direction, and topology.
3. Explain synchronous vs asynchronous timing, and the roles of a clock line, baud rate, start bits, and stop bits.
4. Distinguish simplex, half-duplex, and full-duplex communication, and point-to-point vs bus topologies.
5. Explain UART's configuration, wiring, framing, and baud rate calculation, including flow control and long-distance variants (RS-232/RS-485).
6. Explain SPI's configuration, shift-register operating principle, chip-select mechanism, and the four SPI modes (CPOL/CPHA).
7. Explain I2C's configuration, open-drain bus mechanism, addressing scheme, clock stretching, and multi-master arbitration.
8. Compare UART, SPI, and I2C directly on speed, wiring, addressing, and appropriate use cases, and choose the right protocol for a given application.
9. Explain why blocking software delays (`sleep()`/`delay()`) are harmful in embedded systems, and how hardware timers solve this.
10. Explain the role of a prescaler in converting a fast system clock into a usable timer tick rate.
11. Explain up-counting, down-counting, and center-aligned (up/down) timer counting modes and when each is used.
12. Distinguish overflow interrupts from compare-match interrupts, and explain input capture and timer-triggered DMA.
13. Explain timer cascading for building larger effective timers, and describe general-purpose timers, RTCs, and watchdog timers.
14. Identify common timer configuration traps: off-by-one prescaler errors, unacknowledged ISR flags causing infinite re-entry, clock tree assumption errors, unsynchronised dynamic updates ("runaway timer"), and torn multi-byte reads.
15. Define Pulse Width Modulation (PWM), calculate frequency and duty cycle, and explain how PWM approximates an analogue voltage using a digital pin.
16. Explain the two ways to generate PWM (dedicated PWM peripheral vs general-purpose timer output-compare) and the shared underlying mechanism.
17. Explain the ADC's core conversion math (resolution, quantisation error), the SELECT → ACQUIRE → CONVERT → PUBLISH pipeline, and the difference between resolution and accuracy (including ENOB).
18. Explain the Nyquist sampling limit and the danger of "critical sampling" causing a signal to alias into a flat DC line.
19. Explain why a clean analogue front end (protection, filtering, stable reference/ground) is a prerequisite for good ADC readings, and diagnose common ADC symptom/cause/fix scenarios.
20. Explain firmware calibration of an ADC using the two-point method, and the RP2040's specific ADC register model (ADC_CS, RESULT, FIFO/FCS) and its polling vs FIFO-interrupt vs DMA usage patterns.

---

## PART 1: SERIAL DIGITAL COMMUNICATION (W3.1)

### 1. The Hardware Constraint — Why Serial Exists

**Parallel Communication:**
- Uses **multiple dedicated wires** (8 or more data wires).
- Sends **multiple bits simultaneously**, one per wire.
- **Extremely fast**, but the physical footprint (number of wires, connector size, PCB routing complexity) scales badly as more bits or more devices are added.

**Serial Communication:**
- **Vastly reduced physical footprint** — sends bits **sequentially** over a single data line.
- Has become the **universal standard for chip-to-chip communication** in modern embedded systems, precisely because it trades a bit of raw speed for a massive reduction in wiring, pin count, and cost.

*(Context: this is the same fundamental trade-off you'll see repeated in real systems — e.g., old parallel ATA/SCSI hard drive cables vs modern serial SATA/USB, or old parallel printer ports vs modern USB.)*

### 2. The Three Core Variables

**Every specific serial protocol is simply a unique combination of three fundamental engineering choices:**

1. **TIMING** — how the receiver knows *when* to read each bit.
2. **DATA FLOW** — which direction(s) data can travel.
3. **TOPOLOGY** — how many devices are connected, and how they're wired together.

Once you understand these three variables, you can understand *any* serial protocol — including ones not covered in this lecture — by simply asking "what choice did this protocol make for each of the three variables?"

### 3. Variable 1: Timing

**Synchronous:**
- Devices **share a dedicated physical Clock Wire (SCLK)**.
- The receiver knows exactly when to read the data line because it watches for the clock line to transition (commonly **samples on the rising edge**).
- Because timing is handled by a physical wire, synchronous protocols can typically run much faster and more reliably.

**Asynchronous:**
- **No shared clock line.**
- Both devices must be **pre-configured to the exact same speed** — the **baud rate**.
- Uses **"Start" and "Stop" bits** wrapped around the actual data payload, so the receiver can detect exactly when a new piece of data begins and ends, purely from the timing of edges on the single data line.

### 4. Variable 2: Direction of Data Flow

| Mode | Description |
|---|---|
| **Simplex** | A strict one-way street. Data only ever flows from the transmitter to the receiver. |
| **Half-Duplex** | A two-way street, but only **one device can talk at a time** over a shared line. |
| **Full-Duplex** | **Simultaneous** two-way communication. Requires **dedicated TX and RX lines** (separate physical wires for each direction). |

### 5. Variable 3: Topology

| Topology | Description |
|---|---|
| **Point-to-Point** | A direct, dedicated 1-to-1 connection. Simple and secure, but **doesn't scale well** if you need to connect many devices to a single microcontroller. |
| **Bus (Multi-drop)** | **One Controller** communicates with **multiple Peripherals** on shared wires. Requires either **hardware selection pins** or **software addressing** to route messages to the correct device. |

---

## PART 2: THE "BIG THREE" PROTOCOLS

*"Applying our framework to UART, SPI, and I2C to deeply understand how they work."*

### 6. UART: Configuration & Purpose

**UART = Universal Asynchronous Receiver-Transmitter.**

**Important clarification:** UART is **not technically a protocol** — it's a **physical circuit** built into a microcontroller that translates data between **parallel** form (inside the chip) and **serial** form (on the wire).

**The Configuration:**
- **Timing:** Asynchronous
- **Direction:** Full-Duplex (usually)
- **Topology:** Point-to-Point
- **Typical Speed:** 9.6 kbps – 115.2 kbps

**Why choose UART?**
- **Minimalism.** Only **two wires** (TX and RX) are the bare minimum needed for two devices to talk back and forth simultaneously. By dropping the clock wire entirely, you save a physical pin on the microcontroller.
- **Real-World Uses:** because it operates at slower, reliable speeds without needing a clock, it's ideal for **legacy sensors (GPS, RFID), Bluetooth modules, and simple debug consoles**.

### 7. UART: How It Works

- Because there's no clock wire to keep the two sides synchronised, **both the sender and receiver must be pre-configured to talk at the exact same speed** — the **Baud Rate** (e.g., 9600 or 115200 bits per second).
- Devices are **cross-wired**: TX (transmit) on one device connects to RX (receive) on the other, and vice versa.
- **The line is held HIGH when idle.** A drop to LOW signals the **start** of a transmission.

**Frame structure (left to right on the wire):** `idle → start bit → data bits → parity (optional) → stop bit → idle`

**Important detail about bit order — annotated correction:** UART transmits its data bits **LSB (Least Significant Bit) first**. So if the waveform diagram shows the data bits appearing in the order `1 1 0 0 1 0 1 0`, that sequence is read **starting from the LSB**, meaning the actual byte value (written conventionally MSB-first) is the **reverse** of that display order — the annotation confirms the image actually represents the byte `0101 0011`.

### 8. Baud Rate Calculation for UART

**⚠️ This calculation type will be tested.**

Example frame breakdown:
- **Start bit** (1 bit)
- **Data bits:** 7 bits shown as `1 1 0 0 1 0 1`, transmitted **LSB first**
- **Parity bit:** 1 bit — here it's **even parity** (the parity bit is chosen so the total count of 1-bits, including the parity bit itself, is even)
- **Stop bit:** 1 bit (can be configured as 2 stop bits instead)

**Reading the actual character:** Because the data bits are sent LSB-first, reversing the displayed sequence gives the true byte value, which the slide identifies as **ASCII 'S' (shown on the slide as 0x52, though note: ASCII 'S' is actually 0x53 — worth double-checking this specific value against the original slide/lecturer, as 0x52 is actually ASCII 'R'. The bit-order annotation confirming `0101 0011` = `0x53` matches 'S', so treat 0x53 as the intended value).**

**The core baud rate formula and worked example:**

- This frame format uses **7 data bits + 3 control bits** (1 start + 1 parity + 1 stop) = **10 bits per character** total.
- If the transmission rate is **100 characters per second**, then:

  $$\text{Baud Rate} = 100 \times 10 = 1000 \text{ baud/s (i.e., 1000 bps)}$$

- **General formula to remember:** `Baud Rate = (characters per second) × (total bits per character, including start/parity/stop overhead)`

### 9. UART: Variations & Considerations

**1. Flow Control — What if Device A sends data faster than Device B can process it?**
- Without protection, **buffers overflow** and data is lost.
- **Hardware Flow Control:** adds two extra wires — **RTS (Request to Send)** and **CTS (Clear to Send)** — so devices can signal each other to pause.
- **Software Flow Control:** sends special control characters (**XON/XOFF**) *inside* the normal data stream to pause/resume transmission, without needing any extra wires.

**2. Distance vs Noise (RS-232 / RS-485):**
- **Standard UART** uses **TTL logic levels** (0V and 3.3V/5V), which are **highly susceptible to electrical noise** over long cables.
- **RS-232:** uses **higher voltage swings** (e.g., -15V to +15V) to increase noise immunity and usable cable range.
- **RS-485:** converts the UART signal into **differential signalling** over twisted-pair wires, allowing reliable communication over **thousands of feet** even in noisy industrial environments.

### 10. RP2040 UART Features

| Category | Details |
|---|---|
| **Peripherals** | 2× UART (UART0, UART1) |
| **Data Length** | 5 to 8 data bits |
| **Parity** | None, Even, or Odd |
| **Stop Bits** | 1 bit or 2 bits |
| **TX/RX FIFOs** | 32-byte transmit FIFO, 32-byte receive FIFO |
| **Interrupt Support** | TX FIFO level, RX FIFO level, Receive timeout, Error conditions |
| **DMA Support** | Yes |
| **Clock** | Flexible baud-rate divider |
| **Flow Control** | Hardware CTS/RTS supported |

---

### 11. SPI: Configuration & Purpose

**SPI = Serial Peripheral Interface**, developed by **Motorola in the 1980s** as a high-speed backbone for embedded systems.

**The Configuration:**
- **Timing:** Synchronous
- **Direction:** Full-Duplex
- **Topology:** Bus (Single-Master, Multi-Slave)
- **Typical Speed:** 10 MHz – 50+ MHz

**Why choose SPI?**
- **Raw Speed.** Because it has a **dedicated clock line** and **push-pull drivers**, massive amounts of data can flow continuously. The separate transmit and receive lines mean data flows **both ways simultaneously**.
- **Real-World Uses:** applications requiring enormous data pipelines like **high-resolution LCDs, OLED displays, SD cards, and fast ADCs**.

### 12. SPI: How It Works

- SPI acts like a **circular shift register** shared between master and slave.
- On **every clock pulse** generated by the Master:
  - One bit is pushed out on **MOSI** (Master Out, Slave In).
  - Simultaneously, one bit is pulled in on **MISO** (Master In, Slave Out).
- The Master uses a dedicated **Chip Select (CS)** pin, which is **active LOW**, to "wake up" the specific slave it wants to talk to — all other slaves ignore the bus traffic while their CS line is inactive (HIGH).
- **Important bit-order distinction (annotated on slide): UART sends its LSB first, while SPI sends its MSB first.** This is a key difference to remember between the two protocols.

### 13. SPI Modes — Clock Polarity and Phase (CPOL/CPHA)

- Because there's **no single universal standard** for exactly how SPI should behave at the clock-edge level, different manufacturers design their chips slightly differently. **You must configure the Master to match the Slave's expected behaviour.**

- **CPOL (Clock Polarity):** is the clock line idling **HIGH (1)** or **LOW (0)** when nothing is being sent?
- **CPHA (Clock Phase):** is data captured on the **first edge** (transition) of the clock, or the **second edge**?

This creates **4 distinct SPI Modes**:

| Mode | CPOL | CPHA | Idle Clock | Sample Edge |
|---|---|---|---|---|
| 0 | 0 | 0 | Low | Rising |
| 1 | 0 | 1 | Low | Falling |
| 2 | 1 | 0 | High | Falling |
| 3 | 1 | 1 | High | Rising |

### 14. RP2040 SPI Features

| Category | Details |
|---|---|
| **Peripherals** | 2× SPI (SPI0, SPI1) |
| **Operation Modes** | Master Mode, Slave Mode |
| **Data Format** | 4 to 16 data bits per frame; MSB-first or LSB-first; supports SPI Modes 0, 1, 2, 3 (CPOL = 0 or 1, CPHA = 0 or 1) |
| **Signals** | SCK (Serial Clock), TX/MOSI (Master Out Slave In), RX/MISO (Master In Slave Out), CSn/SS (Chip Select) |
| **TX/RX FIFOs** | 8-entry transmit FIFO, 8-entry receive FIFO |
| **Interrupt Support** | TX FIFO level, RX FIFO level, RX timeout, Receive overrun, Receive underrun, Transmit overflow |
| **DMA Support** | DMA request for TX FIFO, DMA request for RX FIFO |
| **Flexible Clock Generation** | Programmable SPI clock divider, up to system clock limitations |
| **Multi-Master Support** | Supported by hardware; external arbitration required by system design |
| **Full-Duplex** | Simultaneous transmit and receive — one bit shifted out and one bit shifted in per clock cycle |

---

### 15. I²C: Configuration & Purpose

**I²C = Inter-Integrated Circuit** (pronounced "I-squared-C"), **invented by Philips Semiconductor** to allow multiple chips on a TV board to communicate easily using minimal wiring.

**The Configuration:**
- **Timing:** Synchronous
- **Direction:** Half-Duplex
- **Topology:** Bus (Multi-Master, Multi-Slave)
- **Typical Speed:** 100 kHz (Standard) – 400 kHz (Fast)

**Why choose I²C?**
- **Pin Conservation.** You can connect **over 100 devices** to a microcontroller using **exactly two wires** — **SCL (clock)** and **SDA (data)**. No dedicated Chip Select pins are required at all (unlike SPI).
- **Real-World Uses:** because physical pull-up resistors limit the switching speed, it's perfect for **low-bandwidth sensors** that don't need to send much data — **temperature, humidity, accelerometers**, and **Real-Time Clocks (RTCs)**.

### 16. I²C: How It Works

- I²C uses **"open-drain" lines pulled HIGH** by external pull-up resistors. **Devices can only actively pull the lines LOW** — they never actively drive them HIGH (the resistor does that passively).
- To talk to a specific slave, the **Master broadcasts a 7-bit software address** over the data line.
- **Only the slave with that matching hardcoded address responds** with an **ACK** (acknowledge), and data transfer then begins.

**I²C Data Packet Structure (on the SDA line):**

`START → Address (7-bit) → Read/Write bit → ACK → Data Payload (8-bit) → ACK → STOP`

### 17. I²C: Advanced Concepts

**1. Clock Stretching:**
- Normally, the **Master controls the clock (SCL) entirely**.
- However, if a Slave is **too slow** to process incoming data (e.g., an ADC that's still taking a reading), the **Slave can physically hold the SCL line LOW**.
- The Master sees the clock line held low and **waits**. Once the Slave is ready, it releases the SCL line, which floats back HIGH via the pull-up resistor, and the Master resumes generating the clock.
- **This allows slow devices to pause fast masters** — a built-in flow-control mechanism unique to I²C.

**2. Multi-Master Arbitration:**
- **What if two Masters try to talk on the bus at the exact same time?**
- Because the lines are **open-drain**, pulling a line LOW always "wins" over letting it float HIGH.
- **Both masters monitor the SDA line as they transmit.** If Master A tries to send a '1' (letting the line float HIGH) but actually sees the line go LOW instead, it knows Master B is simultaneously transmitting a '0'. **Master A instantly backs off** and lets Master B finish — this is how the bus avoids actual electrical collisions.

### 18. RP2040 I²C Features

| Category | Details |
|---|---|
| **Peripherals** | 2× I²C (I²C0, I²C1) |
| **Signals** | SCL (Serial Clock), SDA (Serial Data) |
| **Operation Modes** | Controller (Master) Mode, Target (Slave) Mode, Multi-Master Capable |
| **Addressing** | 7-bit Addressing, 10-bit Addressing |
| **Data Format** | 8-bit Data Bytes, ACK/NACK Handshaking, START / REPEATED START / STOP Conditions |
| **TX/RX FIFOs** | 16-entry transmit FIFO, 16-entry receive FIFO |
| **Interrupt Support** | TX FIFO level, RX FIFO level, START detection, STOP detection, Read request detection, RX overrun, RX underrun, TX overrun, Abort conditions |
| **DMA Support** | DMA request for TX FIFO, DMA request for RX FIFO |
| **Bus Features** | Standard Mode (100 kbps), Fast Mode (400 kbps), Fast Mode Plus (1 Mbps), Clock Stretching, Arbitration Support |

---

### 19. Protocol Selection: Speed vs Application

| | **UART (Slow)** | **I²C (Medium)** | **SPI (Fast)** |
|---|---|---|---|
| **Speed** | ~9.6 kbps – 115.2 kbps | 100 kHz – 400 kHz | 10 MHz – 50+ MHz |
| **Ideal for** | Legacy sensors (GPS, RFID), simple debug consoles | Environmental sensors (temperature, humidity), RTCs | LCD screens, OLEDs, SD cards, high-speed ADCs |
| **Why?** | Lack of a shared clock makes high-speed synchronisation harder — chosen when **wiring simplicity** matters more than throughput | Relies on pull-up resistors to bring the voltage back HIGH; **bus capacitance** physically limits how fast the signal can cleanly switch | **Push-pull hardware drivers** forcefully drive the lines HIGH and LOW; combined with a dedicated clock wire, this allows massive continuous data pipelines |

### 20. RP2040 Digital Communications — Full Comparison Table

| Feature | UART | SPI | I²C |
|---|---|---|---|
| **RP2040 Peripherals** | 2 | 2 | 2 |
| **Signals** | TX, RX | SCK, MOSI, MISO, CS | SCL, SDA |
| **Clock Required** | No | Yes | Yes |
| **Duplex** | Full | Full | Half |
| **Addressing** | No | No | Yes (7/10-bit) |
| **Multiple Devices** | Difficult | Via CS lines | Native bus addressing |
| **Data Size** | 5–8 bits | 4–16 bits | 8-bit bytes |
| **Parity** | Yes | No | No |
| **Stop Bits** | Yes | No | No |
| **ACK/NACK** | No | No | Yes |
| **FIFO** | 32-byte | 8-entry | 16-entry |
| **DMA** | Yes | Yes | Yes |
| **Multi-Master** | No | Possible but uncommon | Supported |
| **Typical Speed** | kbps–Mbps | Mbps–tens of Mbps | 100 kbps–1 Mbps |

---

## PART 3: TIMERS — UNBLOCKING THE CPU (W3.2)

### 21. The Software Delay Problem

- **Blocking Code:** functions like `sleep()` or `delay()` are **entirely blocking**. The CPU is trapped **running an empty loop** just checking the time, over and over.
- **Zero Multitasking:** while blocked this way, the microcontroller **cannot** read sensor data, process other inputs, or respond to emergencies — it is completely unavailable for anything else during the entire delay period.
- *(Illustrated on the slide with a pizza, a microwave, and an alarm clock — everyday examples of "waiting for time to pass" that map onto why hardware timers matter: you don't want your entire kitchen frozen solid just because the microwave is counting down.)*

### 22. The Gear Reducer: The Prescaler

- **Timers require a clock signal to increment.** However, system clocks are extremely fast (e.g., **125 MHz**). A 16-bit timer counting at 125 MHz would **overflow in a fraction of a millisecond** — far too fast to be useful for most everyday timing tasks.
- **The Prescaler** mathematically **divides** the high-frequency system clock down before it ever reaches the actual counting register.

**Worked example:**
- If you set a **Prescaler of 125**, the actual Timer Register only counts up **once for every 125 system clock ticks**, slowing the effective tick rate down to a very usable **1 MHz** (i.e., 1 tick = 1 microsecond).

**Flow:** `SYSTEM CLOCK (e.g. 125 MHz) → PRESCALER (divider, N) → TIMER TICK (e.g. 1 MHz) → COUNTER REGISTER`

### 23. Timer Config: Up & Down Counting

**Up-Counting (Sawtooth) — the standard mode:**
- The hardware **starts at 0**, increments up to a maximum value, and then **instantly resets back to 0**.
- **Analogy: a stopwatch counting elapsed time up from 0:00.**

**Down-Counting (Reverse Sawtooth):**
- You **load a maximum value**, and the hardware **decrements** down to 0 before **automatically reloading** the maximum value again.
- **Analogy: a microwave timer counting down to 0.**

### 24. Timer Config: Center-Aligned (Up/Down)

- **Up/Down Counting (Center-Aligned):** instead of snapping back to zero at the peak, the timer counts **up to MAX**, turns around, and **gracefully counts back down to zero** — repeating this cycle.
- **The Triangle Wave:** this creates a perfectly **symmetrical triangle wave** in hardware.
- **Why use this?** This is the "secret weapon" for **Motor Control** and generating **symmetrical PWM (Pulse Width Modulation)**. It aligns pulses perfectly in the center of each period, which **drastically reduces electromagnetic noise** and makes motors run smoother compared to a simple edge-aligned sawtooth approach.

### 25. When Do We Get Interrupted?

- The hardware timer signals the CPU via an **Interrupt Service Routine (ISR)** — but exactly *when* does it "pull the alarm"?

**1. Overflow Interrupt:**
- Fires when the counter hits its **absolute limit and rolls over** (the peak of the sawtooth).

**2. Compare Match Interrupt:**
- You load a specific number into a **"Compare Register."**
- As the counter runs, hardware **constantly compares** the live count against that stored value.
- **When they match, it fires an interrupt** — this allows you to trigger events at **exact intervals**, potentially well *before* an overflow would ever happen. This gives you much finer control over precisely when an event fires, rather than being limited to only "when the counter wraps around."

### 26. Input Capture (The Digital Stopwatch)

- **While Output Compare *creates* events, Input Capture *measures* external events with microscopic precision.**
- **How it works:** a GPIO pin is routed **directly to a timer** at the hardware level. When the pin changes state (e.g., goes HIGH), the hardware **instantly copies the timer's current count** into a special **"Capture Register"** and fires an interrupt.
- **The Analogy:** a runner breaking a laser beam at a finish line. The laser automatically records the exact time on a clipboard **before** the referee (the CPU) has a chance to react — the timing is captured entirely in hardware, immune to any software/interrupt-handling delay.
- **Use Cases:** measuring **motor RPM (via Hall effect sensors)**, decoding **IR remote signals**, reading **RC receiver pulse widths**.
- The pulse width between two captured events is simply: **Pulse Width = T2 − T1** (the difference between two consecutive capture-register values).

### 27. Timer-Triggered DMA (The Invisible Hand)

- **Direct Memory Access (DMA)** allows data to move around the chip **without the CPU lifting a finger** — this represents the ultimate evolution of non-blocking code (going one step further than even interrupts, since the CPU doesn't even need to run a short ISR).
- **How it works:** configure a timer to overflow at a specific rate (e.g., **44.1 kHz**, a common audio sample rate). Instead of firing a CPU interrupt, the overflow event sends a **hardware signal directly to the DMA controller**.
- The **DMA controller** then grabs one sample from the **Analog-to-Digital Converter (ADC)** and writes it into **RAM**, entirely independently — the CPU can remain completely asleep or doing other work the whole time.
- **Use Cases:** **high-quality audio sampling**, reading continuous sensor streams, driving addressable LEDs like **NeoPixels (WS2812B)** with **zero CPU overhead**.

### 28. Timer Cascading (The Odometer)

- Sometimes a **16-bit timer** (which only counts up to **65,535**) isn't large enough, even with a prescaler, to track time over days or weeks without wrapping around too often.
- **How it works:** hardware timers can be **chained together**. **Timer A** counts the system clock directly. **When Timer A overflows, its output is routed as the clock input to Timer B.**
- **The Analogy:** an analogue car **odometer**. When the "miles" wheel completes a full rotation, it physically clicks the "tens of miles" wheel forward by one notch.
- This technique allows creating an **effective 32-bit timer** out of two smaller 16-bit timers.
- **Note (Pico-specific):** the Raspberry Pi **Pico has a native 64-bit hardware timer**, so cascading isn't strictly necessary there — but this cascading technique is **crucial for other common chip families like AVR, PIC, and STM32**, which typically only offer smaller native timer widths.

### 29. Specialized Timer Types

| Type | Description |
|---|---|
| **General Purpose** | The standard workhorse. Used for triggering delays, creating PWM signals, generating output waveforms, and capturing the timing of incoming digital signals. |
| **Real-Time Clock (RTC)** | Designed strictly to keep **human time** (Year, Month, Day, Hour, Min, Sec). Often runs on a **separate, ultra-low-power domain**, sometimes with a **coin cell battery backup**, so it keeps ticking even when the rest of the chip is powered off. |
| **Watchdog Timer (WDT)** | A **safety failsafe**. Software must continuously reset ("**pet**" or "kick") this down-counting timer. **If the code crashes** and stops petting it, the timer hits zero and **forces a hard hardware reboot** — a critical last-resort recovery mechanism for unattended embedded systems. |

### 30. Dynamic Updates: The Runaway Timer

- **The Mistake:** dynamically changing the **Compare Match target** while the timer is **actively running**, without using **Shadow Registers**.
- **The Issue:** if the timer has **already passed** the *new* target value by the time you write it, the hardware will **miss the match entirely**. It will then blindly count all the way to its absolute maximum limit, overflow back to zero, and **only then** finally hit the new target.
- **The Impact:** this causes a **massive, unexpected glitch in timing** — a skipped beat in an RTOS scheduler, or a violent, audible stutter in a PWM-driven motor controller.
- **The Fix:** **always enable Preload/Shadow Registers.** The hardware buffers your newly-written value internally and only applies it cleanly **exactly at the next rollover point**, rather than immediately and unpredictably mid-cycle.

### 31. 16-bit Reads on 8-bit Buses: Torn Reads

- **The Mistake:** reading a **multi-byte running timer** directly, where the CPU has to read the lower half, then the upper half, **sequentially** (as two separate bus transactions).
- **The Issue:** if the timer happens to **roll over perfectly in the middle** of your two separate reads, you end up "stitching together" **half of the old value and half of the new value** — producing a completely nonsensical result.

**Worked example:**
| Time | Action | Register value | What you actually read |
|---|---|---|---|
| T0 | Timer is at `0x00FF` | Low=`00`, High=`FF`* | — |
| T1 | CPU reads Low Byte | Low=`00`, High=`FF` | `[--, FF]` |
| T2 | **Timer Rolls Over!** | Low=`01`, High=`00` | — |
| T3 | CPU reads High Byte | Low=`01`, High=`00` | `[01, FF]` |

*(**Note on the table layout:** the slide's boxes list Low/High byte pairs per row — at T0 the timer holds `00FF`; by T3, after the rollover, the true current value is `0100`, but because the CPU already captured the stale high byte `FF` from before the rollover and only the low byte `01` is fresh, the two mismatched halves get stitched together.)*

- **Result:** the CPU thinks the timer is at **`0x01FF` (511)**, when the real value at that moment was actually just **256**.
- **The Fix:** briefly **disable global interrupts**, or better, utilise the hardware's **built-in locking mechanism** — many chips automatically **freeze the upper byte's shadow register** the exact moment you read the lower byte, guaranteeing you get a clean, **atomic read** of the whole value.

### 32. Configuration Traps — Three Classic Timer Bugs

**1. Off-by-One (The N−1 Rule):**
- **Hardware registers are zero-indexed.** If you want a prescaler that divides by 100, and you naively write the literal value `100` into the register, it will actually **divide by 101** (because counting from 0 to 100 inclusive is 101 total steps).
- Over time, this small error accumulates into **massive timing drift**.
- **Fix: always use `Target − 1`** when writing to this class of zero-indexed register.

**2. The Infinite ISR:**
- On bare-metal C, you must **manually clear the timer's hardware interrupt flag** at the end of the ISR.
- **If you forget:** the CPU exits the ISR, immediately sees the flag is still set, and **instantly re-enters** the same ISR again — **freezing the main program forever** in an infinite interrupt loop, since `main()` never actually gets to run.

**3. Clock Tree Illusions:**
- Engineers often calculate timer periods **assuming the timer runs at the main CPU clock speed** (e.g., 100 MHz).
- However, **peripheral buses often have their own independent hardware clock dividers** — meaning, for example, that "Timer 2" might actually be running at only **50 MHz** internally, even though the CPU core itself runs at 100 MHz. **Always verify the actual peripheral clock feeding a specific timer**, rather than assuming it matches the core clock.

---

## PART 4: PULSE WIDTH MODULATION (PWM)

### 33. What Is Pulse Width Modulation (PWM)?

PWM is described in terms of **one digital output** that switches repeatedly between HIGH and LOW, characterised by two key parameters:

**Frequency — how often the pulse repeats:**

$$f_{PWM} = \frac{1}{T}$$

where **T** is the total length of one period.

**Duty Cycle — the share of each period spent HIGH:**

$$D = \frac{T_{on}}{T} \times 100\%$$

**Summary: "Frequency sets the rhythm; duty cycle sets the ON share."**

### 34. PWM = Fake Analog Voltage

- **Same voltage levels. Different ON time.** The pin never actually outputs an intermediate voltage — **the pin stays digital** the entire time; it only ever switches cleanly between LOW and HIGH.
- What changes between 25%, 50%, and 75% duty cycle examples is purely **how much time** is spent HIGH vs LOW within each repeating period — **more ON time → more average energy** delivered to whatever is connected.
- A **load, its own physical inertia, or an external filter** averages these rapid pulses out over time, which is what creates the *illusion* of a smooth, intermediate analogue voltage to the outside world.

**Approximate average drive delivered:**

$$\text{Average drive} \approx D \times \text{supply level}$$

**Why use PWM at all?**
- **Efficient switching avoids wasting the excess voltage as heat** in a linear (analogue) control element — a transistor either fully ON (near-zero resistance, near-zero heat) or fully OFF (no current, no heat) wastes far less energy than one held in a partially-conducting linear state.
- Common applications: **LEDs** (brightness control), **Motors** (speed control), **Power stages** (voltage regulation).

### 35. Configuring PWM — Two Approaches

**Dedicated PWM peripheral (built for richer waveform control):**
- Offers **multiple synchronised channels** and precise update timing.
- Often adds **complementary outputs and dead-time insertion** (useful for driving motor H-bridges safely, preventing both transistors in a leg from ever being on simultaneously).
- May include **fault/brake inputs** for immediate emergency shutdown.
- **Best fit:** motor control and power-conversion stages.
- **Note:** the exact peripheral name varies by MCU family.

**General-purpose timer PWM (uses timer output-compare behaviour):**
- A **counter** defines position within each PWM period.
- A **period register** defines when the counter rolls over.
- A **compare register** defines when the output pin's state actually changes.
- **Best fit:** LEDs, buzzers, servos, and other straightforward drive tasks.
- **Note:** usually configured as just one operating mode of an otherwise ordinary general-purpose timer.

**Underlying unifying principle: "Same mechanism underneath: clocked counter + period + compare + output logic."** Whether you use a dedicated PWM peripheral or a general-purpose timer in output-compare mode, the fundamental building blocks are identical.

### 36. A Timer Becomes PWM When Compare Events Drive the Output Pin

**Typical Edge-Aligned Up-Count Mode**, illustrated with three key register names: **ARR** (Auto-Reload Register — the period register) and **CCR** (Capture/Compare Register — the compare register):

1. **Count:** timer clock ticks advance the counter upward.
2. **Reset:** the period value (ARR) ends one cycle and instantly starts the next — this is what sets the overall PWM **frequency**.
3. **Compare:** the compare match (against CCR) changes the output pin's state — this is what sets the PWM **duty cycle**.

**Formulas (exact form varies with counting mode and MCU family):**

$$f_{PWM} = \frac{f_{timer}}{(PSC+1)(ARR+1)}$$

$$D = \frac{CCR}{ARR+1}$$

Where **PSC** is the prescaler value. **"The bridge: timer period sets PWM frequency; compare sets duty cycle."**

---

## PART 5: ANALOG & DIGITAL CONVERSION — ADC (W3.3)

### 37. 12-bit ADC Core Math

- The **RP2040 ADC** converts an analogue input voltage into its **12-bit digital representation**.
- The **upper reference** is **V_REF** (typically 3.3V or 5V) and the **lower reference is Ground** (typically 0V).

**Core conversion formula:**

$$N_{ADC} = (2^{12} - 1) \times \frac{V_{in}}{V_{REF}}$$

- **Max:** $N_{ADC} = 4095$ when $V_{in} \geq V_{REF}$
- **Min:** $N_{ADC} = 0$ when $V_{in} \leq 0V$

- **Quantisation error** is the unavoidable difference between the actual, true analogue voltage and the nearest digital code value the ADC can represent — typically **up to ±0.5 LSB (Least Significant Bit)**.

**Worked practice calculations (V_REF = 3.3V):**

- **Q1: What is the digital output when a potentiometer is set to 1.23V?**
  $$N_{ADC} = 4095 \times \frac{1.23V}{3.3V} \approx 1526$$

- **Q2: What is the LSB resolution?**
  $$\frac{3.3V}{4096} = 0.000805V = 0.8mV$$

- **Q3: What is the maximum Quantisation error?**
  $$\pm (\tfrac{1}{2} \times \text{LSB}) \approx \pm 0.403 \text{ mV}$$

### 38. The ADC Acquires Voltage Before Conversion

The **RP2040 uses one 12-bit SAR (Successive Approximation Register) converter** behind a **five-input multiplexer** — meaning one physical converter is shared and time-sliced across five possible inputs.

**The Four-Step Pipeline:**

1. **SELECT** — `AINSEL` chooses which input to read: GPIO26–29, or the internal temperature sensor.
2. **ACQUIRE** — the chosen input **charges** the ADC's internal sampling circuitry.
3. **CONVERT** — the **Successive Approximation** algorithm resolves the result **one bit at a time**.
4. **PUBLISH** — the `RESULT` register updates, or the sample is pushed into the **FIFO**.

**Timing:**
- **Maximum-rate timing:** 48 MHz ADC clock, **96 cycles per conversion**.
- **One complete result** takes **2 µs**.
- **Takeaway: "The signal must settle before bit decisions can be trusted."** — if the input voltage is still changing while the SAR algorithm is resolving individual bits, the result will be corrupted/inaccurate.

### 39. Twelve Output Bits Do Not Guarantee Twelve Accurate Bits

- **Resolution** describes **code spacing** (how finely the output scale is divided). **Accuracy** describes **closeness to the true voltage**. These are two genuinely different properties, and a high resolution does not automatically imply high accuracy.

**Ideal 12-bit scale:**
- **4096** possible codes, ranging from **0 to 4095**.
- At 3.3V: **1 LSB ≈ 0.806 mV**.
- **Practical tip:** repeat the same DC measurement many times — the **spread** of readings reveals **noise**, while the **average** of those readings reveals **bias** (systematic offset error).

**Why measured accuracy is typically lower than the ideal resolution suggests:**
- **Reference:** noise or drift on the reference voltage itself changes every resulting code.
- **Offset + gain:** the whole transfer curve can shift up/down (offset error) or tilt (gain error).
- **Nonlinearity:** the actual code steps are not perfectly uniform in real silicon.
- **Noise:** repeated readings of the exact same input voltage will spread across several different codes.
- **Effective resolution:** the **RP2040 is specified at only 8.7 ENOB** (Effective Number Of Bits) — meaning that in practice, only about 8.7 of its 12 nominal bits are actually "clean," useful, trustworthy bits; the remainder is effectively noise.

### 40. The Zero-Crossing Trap (Nyquist Limit)

- **Nyquist's theorem states we must sample at $f_s > 2f_{max}$** (strictly greater than twice the highest frequency component present in the signal) to be able to faithfully reconstruct it.
- **"Critical Sampling"** — sampling at *exactly* 2× the signal frequency — is **dangerous** in real embedded systems.
- **If your sampling perfectly aligns with the signal's zero-crossings**, the ADC will read **all zeros**, every single time, purely by unlucky coincidence of timing.
- **The reconstructed signal then appears as a completely flat DC line (0V)**, entirely losing the original waveform — even though a real, non-zero AC signal was actually present the whole time.
- **Solution: always oversample** (e.g., at **2.5× to 3×** the signal's maximum frequency, rather than the bare theoretical minimum of 2×), and **use hardware timers, never software delays,** to trigger each ADC conversion at a precise, jitter-free interval.

### 41. Good ADC Software Cannot Rescue a Bad Analog Front End

**"Condition the signal before it reaches GPIO26–29."**

**The Pipeline:** `SOURCE (sensor/potentiometer) → PROTECT (keep input within 0...ADC_AVDD) → FILTER (low-pass before sampling) → STABILISER (drive the sampling circuitry) → CONVERT (acquire and convert)`

**GPIO setup — use `adc_gpio_init()`:**
- Configures the pin for **high impedance**
- **Pulls disabled**
- **Digital input disabled** (so the digital input buffer doesn't interfere with, or draw current from, the analogue signal)

**Low-pass filter:**
- **Limit analogue bandwidth before sampling.** Frequencies that would otherwise **alias** into your measurement **cannot be removed later in software** — filtering must happen in the analogue domain, before digitisation, or the aliased information is permanently lost.

**Reference + ground:**
- **ADC codes are ratios relative to `ADC_AVDD`** (the ADC's own analogue supply voltage reference). **Noise on the reference or on the ground return path will appear directly as measurement noise** in your readings, even if the actual input signal itself is perfectly clean.

### 42. Most "Bad ADC" Readings Begin Before the Conversion Starts

**"Diagnose the symptom, then check the electrical cause before rewriting firmware."**

| Symptom | Likely Cause | Fix |
|---|---|---|
| **Codes wander at rest** | Floating input or noisy reference | Add a defined source; improve grounding and decoupling |
| **First sample is wrong** | Source did not settle after a channel change | Allow settling time; buffer high-impedance sources |
| **Waveform has a false frequency** | Input bandwidth exceeds the sample plan (aliasing) | Add an analogue low-pass filter; raise the sample rate |
| **Readings clip at an endpoint** | Input is outside the valid ADC range | Scale and protect the input; never exceed `ADC_AVDD` |
| **Channels influence each other** | Shared converter plus incomplete settling between channel switches | Slow the scan, discard a settling sample, or buffer |

*(RP2040-specific limit: ADC inputs must remain strictly between ground and ADC_AVDD at all times.)*

### 43. Firmware Calibration

- **The Problem:** silicon is imperfect. Manufacturing tolerances introduce **Offset Errors** (the whole curve is shifted up or down) and **Gain Errors** (the slope is too steep or too shallow compared to ideal).

- **Two-Point Method:** calculate the true slope (gain) and intercept (offset) using two **known** reference voltages — one low, one high:

  $$\text{Gain} = \frac{V_{hi} - V_{lo}}{ADC_{hi} - ADC_{lo}}$$

  $$\text{Offset} = V_{lo} - (\text{Gain} \times ADC_{lo})$$

  Then, at runtime, correct any raw reading with:

  $$V_{actual} = (ADC \times \text{Gain}) + \text{Offset}$$

- **RTOS Implementation guidance:** perform this calibration math **inside a task, not inside the ISR** (calibration math is comparatively slow and doesn't belong in time-critical interrupt context). **Use fixed-point math** to avoid slow floating-point operations, which can be a real performance concern on cores without a hardware floating-point unit.

### 44. The RP2040 ADC — Full Hardware Overview

**"The RP2040 ADC samples five sources at up to 500 kSPS."**

- **12-bit conversion:** 0–4095 output codes.
- **Throughput:** up to **500 kSPS** (kilo-samples per second).
- **External inputs:** GPIO26–29 (channels 0–3).
- **Internal input:** the on-chip **temperature sensor** (channel 4).
- **Buffering:** an **8-entry FIFO**.
- **Streaming:** **DMA** supports fully **CPU-independent** transfer of samples out of the FIFO.

**Internal signal path:** `Input mux → 12-bit converter (SAR) → RESULT register or FIFO`, with internal blocks including a **Sample-and-Hold (S/H)**, a **Comparator**, **Control Logic**, the **Successive Approximation Register (SAR)** itself, and an internal reference **DAC**.

**Summary: "Five selectable sources share one converter; read samples directly or stream them through the FIFO to DMA."**

### 45. ADC_CS — Controls Power, Channel, and Conversion Mode

**Use the control/status register (`ADC_CS`) to enable the ADC, select an input, start conversions, and check readiness.**

| Bit(s) | Field | Purpose |
|---|---|---|
| 0 | **EN** | Power on the ADC and enable its clock. |
| 1 | **TS_EN** | Power on the internal temperature sensor. |
| 2 | **START_ONCE** | Start one conversion; the bit **self-clears** afterward. |
| 3 | **START_MANY** | Run conversions **continuously**. |
| 8 | **READY** | 1 when ready for a new conversion; 0 while busy. |
| 14:12 | **AINSEL** | Select input: GPIO26–29 (values 0–3) or the temperature sensor (value 4). |

**Typical configuration sequence:**
1. Enable **EN**
2. Select **AINSEL**
3. Set either **START_ONCE** or **START_MANY**
4. Check **READY**, or read from the **FIFO**

**Key detail:** `START_ONCE` **self-clears** automatically after one conversion; `START_MANY` **keeps the converter running continuously** until you explicitly stop it.

### 46. RESULT Is for Polling; FIFO Is for Continuous Capture

**Single polling mode — the RESULT register:**
- Holds the **most recent 12-bit conversion result**.
- **Reading RESULT does NOT clear any interrupt flags.**
- **Best matched to manual `START_ONCE` conversions** — i.e., you explicitly request one sample at a time and read it back synchronously.

**Streaming mode — FCS + FIFO registers:**
- The ADC provides an **8-entry FIFO buffer**.
- **FCS** (FIFO Control/Status) controls **enable, threshold, underflow, and overflow status**.
- **Reading the FIFO removes the oldest 12-bit sample** (standard FIFO "pop" behaviour).
- **The FIFO decouples the CPU from continuous `START_MANY` acquisition** — samples pile up in the FIFO at their own pace, and the CPU can drain them whenever it's convenient, rather than needing to keep up in perfect real time.

**Rule of thumb: "Use RESULT for occasional synchronous reads; use FIFO when samples arrive faster than the CPU should block for."**

### 47. Conversion Completion Timing Diagrams

- At the ADC's maximum rate of **500 kSPS**, a full-rate sample completes **every 2 µs**.

**Single conversion (START_ONCE):**
- `START_ONCE` is pulsed → `READY` drops LOW while converting → after **+2 µs**, `READY` rises again and `RESULT` updates with the new sample. **"READY rises when the new RESULT is valid."**

**Continuous conversion (START_MANY + FIFO):**
- `START_MANY` stays asserted → the converter repeatedly runs `CONVERT` cycles back-to-back → each completed conversion triggers a `FIFO PUSH`, incrementing `FIFO LEVEL`.
- `DREQ`/`IRQ` (the DMA request / interrupt request signal) **asserts whenever `FIFO LEVEL ≥ THRESH`** (the configured threshold), and de-asserts again once the FIFO is drained back down below that threshold via a `FIFO read`.
- **"FIFO LEVEL counts unread samples (0–8); requests assert while LEVEL ≥ THRESH."**

### 48. The 8-Entry FIFO Does Not Prevent Data Loss

**"Reliable capture requires the consumer to keep pace with the ADC."**

**The data flow:** `ADC PRODUCER (up to 500 kS/s, one sample every 2 µs) → 8-ENTRY FIFO (LEVEL = unread samples, 0...8) → CONSUMER (IRQ: CPU drains FIFO, or DMA: moves samples to memory)`

**Threshold choice trade-offs:**
- **Threshold = 1:** lowest latency, but the **highest IRQ rate** (interrupts fire very frequently, since even a single new sample immediately triggers one).
- **Threshold = 4:** a balanced choice — reasonable batching with reasonable margin before overflow.
- **Threshold = 8:** maximum batching (fewest interrupts), but **no overflow margin left at all** — if the consumer is even slightly late, samples are immediately lost.

**Failure signals to watch for:**
- **OVER:** the producer (ADC) outran the consumer — the FIFO filled up completely and new samples were dropped.
- **UNDER:** software tried to read from an **already-empty** FIFO.
- **ERR:** a conversion error can accompany a given piece of data in the FIFO.

**Design rule: "Average drain rate must exceed the sample rate, and the buffer must absorb the longest processing stall."** — even if your *average* processing speed is fast enough, a single unusually long stall (e.g., a slow `printf`, a blocking call, a higher-priority interrupt) can still overflow the small 8-entry buffer if it lasts longer than the buffer can absorb.

### 49. Polling Blocks the Loop; FIFO Interrupts Keep Acquisition Moving

**Polling — synchronous read (simple control flow; the CPU waits for `adc_read()`):**

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"

int main(void)
{
  stdio_init_all();
  adc_init();
  adc_gpio_init(26);
  adc_select_input(0);

  while (true)
  {
    uint16_t result = adc_read();
    float voltage = result * (3.3f / 4096.0f);
    printf("Raw: %u, V: %.2f\n", result, voltage);
    sleep_ms(100);
  }
}
```
**Best for occasional samples and straightforward logic.**

**FIFO interrupt — continuous read (the ADC runs continuously; the ISR drains queued samples):**

```c
#include "hardware/adc.h"
#include "hardware/irq.h"

void adc_isr(void)
{
  while (!adc_fifo_is_empty())
  {
    uint16_t result = adc_fifo_get();
    // Process or buffer quickly
  }
}

int main(void)
{
  adc_init();
  adc_gpio_init(26);
  adc_select_input(0);
  adc_fifo_setup(true, false, 1, false, false);
  irq_set_exclusive_handler(ADC_IRQ_FIFO, adc_isr);
  adc_irq_set_enabled(true);
  irq_set_enabled(ADC_IRQ_FIFO, true);
  adc_run(true);

  while (true) tight_loop_contents();
}
```
**Best for sustained capture without blocking the main loop.**

### 50. Slow Firmware Turns ADC Samples Into Data Loss

**"Keep the sampling path deterministic and move expensive work elsewhere."**

| Common Failure | Why It Hurts |
|---|---|
| **Long ISR** — formatting, floating point, or `printf` delays FIFO draining | **Software-delay timing** — `sleep_ms()` adds jitter and does not guarantee precise cadence |
| **Threshold = 1 by habit** — interrupt overhead can dominate CPU time at high sample rates | **No software buffer** — temporary processing stalls immediately become FIFO overflow |
| **Read only one sample** per ISR call — queued samples remain, and the IRQ may immediately re-trigger | **Unsafe shared data** — the ISR and main code race on shared counters, indices, or sample arrays |

**Golden rule: "Use polling for occasional reads, IRQ for moderate streams, and DMA for sustained high-rate capture."**

### 51. Debug ADC Systems by Testing One Uncertainty at a Time

**"A six-step lab turns vague 'noisy ADC' reports into measurable evidence."**

1. **KNOWN DC** — apply a safe, known voltage and predict the ideal expected code.
2. **REPEAT** — capture 1,000 samples; calculate the mean, min, max, and spread.
3. **REFERENCE** — compare `ADC_AVDD` and ground behaviour directly with a multimeter.
4. **TIMING** — change the sample rate and look for settling or aliasing changes.
5. **STRESS** — deliberately slow down the consumer and observe whether `FIFO OVER` occurs.
6. **COMPARE** — measure the CPU cost and jitter of polling, IRQ, and DMA approaches against each other.

**Exit criterion: "You can explain every unexpected code using electrical, timing, or software evidence."**

*(Lab safety reminder from the slide: always keep every ADC input voltage strictly between ground and `ADC_AVDD`.)*

---

## Quick-Reference Summary

- **Serial beats parallel** for chip-to-chip communication because of drastically reduced wiring, at the cost of raw per-instant throughput. Every serial protocol is just a combination of **timing** (sync/async), **data flow** (simplex/half/full-duplex), and **topology** (point-to-point/bus).
- **UART:** async, full-duplex, point-to-point, 2 wires (TX/RX), needs matching baud rate, sends **LSB first**, framed with start/stop/parity bits. Slow but minimal pin count.
- **SPI:** sync, full-duplex, bus (single-master/multi-slave), needs SCLK+MOSI+MISO+CS, sends **MSB first**, 4 modes via CPOL/CPHA. Fastest of the three.
- **I²C:** sync, half-duplex, bus (multi-master/multi-slave), only 2 wires (SCL/SDA), open-drain with pull-ups, 7-bit addressing, supports clock stretching and arbitration. Best for many low-speed sensors on minimal pins.
- **Timers:** never block the CPU with `sleep()`/`delay()` in real embedded designs. A **prescaler** slows the fast system clock to a usable tick rate. Timers can count up, down, or center-aligned (triangle wave, good for symmetric PWM). Interrupts fire on **overflow** or **compare match**. **Input capture** timestamps external edges in hardware; **timer-triggered DMA** moves data with zero CPU involvement. Watch for off-by-one prescaler errors, unacknowledged ISR flags, wrong clock-tree assumptions, unshadowed dynamic updates, and torn multi-byte reads.
- **PWM:** a digital pin that fakes an analogue voltage by varying duty cycle at a fixed frequency; underlying mechanism is always counter + period register + compare register, whether via a dedicated PWM peripheral or a general-purpose timer's output-compare mode.
- **ADC:** converts analogue voltage to a 12-bit code via SELECT → ACQUIRE → CONVERT → PUBLISH. Resolution (4096 codes) ≠ accuracy (RP2040 is really only ~8.7 ENOB). Never sample at *exactly* 2× the Nyquist rate — oversample instead, using a hardware timer trigger. Fix bad readings at the analogue front end (protection, filtering, stable reference/ground) before touching firmware. Use `RESULT` for occasional polled reads, the `FIFO` (with IRQ or DMA) for sustained capture — and always ensure your consumer can drain samples faster than the ADC produces them.
