# Lecture Quiz Bank (Weeks 1 to 4)

**How to use this:** Read the question, think of your answer, then click **Show answer** to check. Questions are numbered per week, in the same order as your original file.

**Contents**

- [Week 1](#week-1) (33 questions): C basics, bitwise ops, embedded system basics
- [Week 2](#week-2) (38 questions): GPIO, interrupts, timers, buses
- [Week 3](#week-3) (34 questions): ADC, timers, serial comms, duty cycle
- [Week 4](#week-4) (18 questions): RTOS, tasks, finite state machines

---

## Week 1

### Q1

**To mask out the lower 8 bits of a 16-bit value and keep the upper 8 bits in their original position, what is the first recommended operation?**

- **A.** Bitwise AND with `0xFF00`
- **B.** Left shift by 8 bits
- **C.** Logical AND with `0xFF00`
- **D.** Bitwise OR with `0x00FF`
- **E.** Right shift by 8 bits

> [!success]- Show answer
> **✅ A. Bitwise AND with `0xFF00`**
>
> Bitwise vs logical: logical AND returns a boolean (1 or 0), giving 1 only if both operands are true. Bitwise AND works bit by bit. ANDing a bit with 1 preserves it, and ANDing with 0 clears it.
>
> Example: `1011 1010 xxxx xxxx` AND `0xFF00` = `1011 1010 0000 0000`

### Q2

**An Assembler ...**

- **A.** does a one-to-one with a compiler
- **B.** optimizes the source code.
- **C.** translates an assembly source file into an object file
- **D.** is also called a Compiler

> [!success]- Show answer
> **✅ C. translates an assembly source file into an object file**

### Q3

**`17 && 23` = ?**

- **A.** 40
- **B.** 6
- **C.** 1
- **D.** 0

> [!success]- Show answer
> **✅ C. 1**
>
> `&&` is the logical AND operator, so it returns 1.

### Q4

**When monitoring a specific bit, for example, bit 5 of register P0, what expression creates a mask to target that bit?**

- **A.** `(1 << 5)`
- **B.** `(P0 | 5)`
- **C.** `(5 << 1)`
- **D.** `(P0 & 5)`

> [!success]- Show answer
> **✅ A. `(1 << 5)`**
>
> `(1 << 5)` creates the mask `0010 0000`. ANDing this mask with the register (bitwise AND) lets us read bit 5.

### Q5

**If a register contains `0x06` and a left shift operation is performed by 1 position (`0x06 << 1`), what is the expected result?**

- **A.** `0x03`
- **B.** `0x07`
- **C.** `0x0E`
- **D.** `0x0C`

> [!success]- Show answer
> **✅ D. `0x0C`**
>
> `0x06` = `0000 0110`. Shifting left by 1 gives `0000 1100`, which is `0x0C`.

> 📝 **Remember:** `||` is logical OR, `&&` is logical AND, `|` is bitwise OR, `&` is bitwise AND.

### Q6

**What is the typical size of a `short int` data type?**

- **A.** 1 byte
- **B.** 8 bytes
- **C.** 2 bytes
- **D.** 4 bytes

> [!success]- Show answer
> **✅ C. 2 bytes**

### Q7

**Which of the following best defines an Embedded System (ES)?**

- **A.** A standalone software application.
- **B.** A computing system dedicated to a specific task within a larger electrical system.
- **C.** A system that only uses analogue electronics.
- **D.** A general-purpose computer for various tasks.

> [!success]- Show answer
> **✅ B. A computing system dedicated to a specific task within a larger electrical system.**

### Q8

**Based on the lecture, which file(s) can be converted to an object file? (can be more than 1)**

- **A.** decoded file
- **B.** c file
- **C.** assembly file
- **D.** hex file
- **E.** b and c

> [!success]- Show answer
> **✅ E. b and c** (the C file and the assembly file)
>
> In the build pipeline, the compiler converts C files (`.c`) into object files (`.o`), and the assembler converts assembly files (`.s`) into object files (`.o`).

### Q9

**A Compiler ...**

- **A.** translate source codes in 'C' language into an executable file.
- **B.** arranges all the '.c' files in the folder
- **C.** is also called an Assembler.
- **D.** translate source codes in 'C' language into object files

> [!success]- Show answer
> **✅ D. translate source codes in 'C' language into object files**
>
> The compiler compiles C source files into object files. It is the linker that then takes those object files and produces the final executable binary.
>
> > ⚠️ Your original file had the answer as "a", but the explanation matches option D. Worth double-checking against your lecturer's key.

### Q10

**Which is the main characteristic of low-level language?**

- **A.** Productivity
- **B.** Compact and optimised
- **C.** Do not need a compiler/assembler
- **D.** Very easy to understand

> [!success]- Show answer
> **✅ B. Compact and optimised**
>
> Low-level languages such as assembly are used to squeeze out performance and produce compact, highly optimized code. High-level languages like C are chosen for productivity, abstraction, and portability.

### Q11

**Which of the following is true about "hard real-time constraints"?**

- **A.** Missed deadline can have extreme consequences.
- **B.** Tasks have no deadlines.
- **C.** Typically non-critical systems.
- **D.** A late response can degrade a system.

> [!success]- Show answer
> **✅ A. Missed deadline can have extreme consequences.**
>
> In a hard real-time system, missing a deadline is unacceptable and can lead to catastrophic consequences (e.g. flight control, airbags). Degraded performance from a late response is **soft** real-time. No deadlines means **non**-real-time.

### Q12

**Which of these are properties of embedded systems?**

- **A.** Infinite memory
- **B.** Based on microcontrollers
- **C.** Able to handle certain timing constraints
- **D.** Unlimited processing power

> [!success]- Show answer
> **✅ B and C**
>
> Embedded systems operate under strict timing constraints and are typically built on microcontrollers with limited memory and processing resources.

### Q13

**Which of the following is False?**

- **A.** Exponentiation (`^`)
- **B.** Modulo (`%`)
- **C.** Division (`/`)
- **D.** Addition (`+`)

> [!success]- Show answer
> **✅ A. Exponentiation (`^`)**
>
> In C, `^` is the bitwise XOR operator, not exponentiation. `%`, `/`, and `+` are all valid arithmetic operators.

### Q14

**Performing a left shift operation on a number by N bits is equivalent to what arithmetic operation?**

- **A.** Dividing the number by N
- **B.** Multiplying the number by 2 to the power of N
- **C.** Multiplying the number by N
- **D.** Dividing the number by 2 to the power of N

> [!success]- Show answer
> **✅ B. Multiplying the number by 2 to the power of N**
>
> Left shifting by N bits (`x << N`) moves every bit position left, which equals x × 2^N.

### Q15

**Which of the following are properties of Embedded Systems?**

- **A.** They often have strict timing constraints.
- **B.** They interact with the real world using sensors and actuators.
- **C.** They are dedicated to handle specific tasks.
- **D.** They use abundant resources compared to general-purpose computers.

> [!success]- Show answer
> **✅ A, B, C**
>
> Embedded systems work with limited resources, not abundant resources.

### Q16

**How many different bitwise operators are there? (Assume it's for Pico C SDK)**

- **A.** 1
- **B.** 2
- **C.** 3
- **D.** 4
- **E.** 5
- **F.** 6
- **G.** 7
- **H.** 8
- **I.** 9
- **J.** too many to count

> [!success]- Show answer
> **✅ F. 6**
>
> Standard C defines exactly 6 bitwise operators:
>
> - `&` (AND)
> - `|` (OR)
> - `^` (XOR)
> - `~` (NOT)
> - `<<` (left shift)
> - `>>` (right shift)

### Q17

**Which best describes the pattern that any embedded system should follow?**

- **A.** input-process-output
- **B.** process-output-input
- **C.** input-output-process
- **D.** process-input-output

> [!success]- Show answer
> **✅ A. input-process-output**

### Q18

**The operator (`!`) is used for Bitwise Complement (Negation)?**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> In C, `!` is the logical NOT operator. The bitwise complement (negation) operator is the tilde `~`.

### Q19

**Which of the following properties of embedded systems are true? (2 correct answers)**

- **A.** Dedication to a wide range of tasks
- **B.** Utilization of high memory and computing power
- **C.** Microcontrollers are typically used rather than microprocessors
- **D.** Necessity for strict timing constraints

> [!success]- Show answer
> **✅ C and D**
>
> Embedded systems are dedicated to specific single tasks (not a wide range), and operate under constrained memory and power (not abundant resources).

### Q20

**Which one is an intermediate file produced by the compiler?**

- **A.** Objective file
- **B.** Objection file
- **C.** Object file
- **D.** Binary file

> [!success]- Show answer
> **✅ C. Object file**
>
> Compilers translate source files into intermediate `.o` / `.obj` object files before the linker combines them into an executable binary.

### Q21

**How many bytes of memory does a `char` data type typically require in most C compilers?**

- **A.** 1 byte
- **B.** 8 bytes
- **C.** 2 bytes
- **D.** 4 bytes

> [!success]- Show answer
> **✅ A. 1 byte**
>
> In standard C, `sizeof(char)` is defined as exactly 1 byte (8 bits).

### Q22

**Which type of variable retains its value between function calls and is stored in RAM?**

- **A.** Local variable
- **B.** Const variable
- **C.** Static variable
- **D.** Volatile variable

> [!success]- Show answer
> **✅ C. Static variable**
>
> A static local variable lasts for the entire program run and keeps its value between calls. It lives in data/BSS memory in RAM, not on the temporary stack frame.

### Q23

**In C, what numeric value represents TRUE in a conditional expression?**

- **A.** 0
- **B.** -1
- **C.** 1
- **D.** Any non-zero value

> [!success]- Show answer
> **✅ D. Any non-zero value**
> B and C as well

### Q24

**A startup code typically ...**

- **A.** initialize the Heap
- **B.** initialize the Stack
- **C.** calls the main function
- **D.** reset all the memory
- **E.** return to the main function

> [!success]- Show answer
> **✅ A, B, C**

### Q25

**The Linker ...**

- **A.** loads the executable file to the memory for execution.
- **B.** executes a file
- **C.** combines one or more object files into a single executable file
- **D.** archives the data files into a single executable file

> [!success]- Show answer
> **✅ C. combines one or more object files into a single executable file**
>
> The linker takes one or more object files from the compiler/assembler, plus libraries, and combines them into an executable binary. The **loader** is what loads it into memory.

### Q26

**Choose the programming language that is the lowest level.**

- **A.** Assembly
- **B.** C
- **C.** Object
- **D.** Fortran

> [!success]- Show answer
> **✅ A. Assembly**
>
> Assembly is a low-level language with a direct, one-to-one mapping to machine instructions. It sits below high-level compiled languages like C and Fortran.

### Q27

**Which operator is used to find the remainder of an integer division?**

- **A.** `/`
- **B.** `*`
- **C.** `^`
- **D.** `+`
- **E.** `-`
- **F.** `%`

> [!success]- Show answer
> **✅ F. `%`**
>
> The modulo operator `%` gives the remainder of an integer division.

### Q28

**Which data type is usually 4 bytes and primarily used for measuring things like weight, distance, or temperature on a Pico W?**

- **A.** `int`
- **B.** `char`
- **C.** `double`
- **D.** `long double`
- **E.** `float`

> [!success]- Show answer
> **✅ E. `float`**

### Q29

**What is a "Heisenbug" as described in the context of the Therac-25 case study?**

- **A.** A bug that disappears or changes behavior when observed or debugged.
- **B.** A bug that is easy to reproduce and fix.
- **C.** A bug caused by hardware malfunction.
- **D.** A bug that explodes
- **E.** A bug related to network communication.

> [!success]- Show answer
> **✅ A. A bug that disappears or changes behavior when observed or debugged.**

### Q30

**What is the typical size of a `short int` data type?**

- **A.** 8 bytes
- **B.** 1 byte
- **C.** 2 bytes
- **D.** 4 bytes

> [!success]- Show answer
> **✅ C. 2 bytes**

### Q31

**To mask out the lower 8 bits of a 16-bit value and keep the upper 8 bits in their original position, what is the first recommended operation?**

- **A.** Logical AND with `0xFF00`
- **B.** Right shift by 8 bits
- **C.** Bitwise OR with `0x00FF`
- **D.** Bitwise AND with `0xFF00`
- **E.** Left shift by 8 bits

> [!success]- Show answer
> **✅ D. Bitwise AND with `0xFF00`**

### Q32

**Which is/are the typical product development process?**

- **A.** Testing
- **B.** Deployment
- **C.** Implementation
- **D.** All of the above

> [!success]- Show answer
> **✅ D. All of the above**

### Q33

**If a register contains `0x06` and a left shift operation is performed by 1 position (`0x06 << 1`), what is the expected result?**

- **A.** `0x0E`
- **B.** `0x03`
- **C.** `0x0C`
- **D.** `0x07`

> [!success]- Show answer
> **✅ C. `0x0C`**
>
> Step by step:
>
> 1. Convert to binary: `0x06` = `0000 0110` (decimal 6)
> 2. Shift left by 1 bit (`<< 1`): `0000 0110 << 1` = `0000 1100`
> 3. Convert back to hex: `0000 1100` = `0x0C`

---

## Week 2

### Q1

**In microcontrollers, GPIO stands for**

- **A.** Global Purpose Input/Output
- **B.** General Purpose Input/Output
- **C.** General Public Input/Output
- **D.** Global Public Input/Output

> [!success]- Show answer
> **✅ B. General Purpose Input/Output**

### Q2

**The watchdog timer (WDT) is designed to restart a system if the software crashes**

*True or False?*

> [!success]- Show answer
> **✅ True**

### Q3

**Why do modern microcontrollers often have "multiplexed" or "multi-function" pins?**

- **A.** To efficiently utilize a limited number of physical pins by allowing them to serve different functions under software control
- **B.** To provide a redundant set of pins in case of hardware failure
- **C.** To increase the overall clock speed of the microcontroller
- **D.** To reduce power consumption by sharing tasks across multiple processors

> [!success]- Show answer
> **✅ A**

### Q4

**GPIO can be used to implement an SPI bus**

*True or False?*

> [!success]- Show answer
> **✅ True**

### Q5

**A CPU can ___ data onto a stack and later it must ___ it off again**

*(Fill in the blanks)*

> [!success]- Show answer
> **✅ push, pop**

### Q6

**Inside a microprocessor, a "timer" is**

- **A.** A counter that can run down or up
- **B.** Software that implements a stopwatch function
- **C.** A built-in clock unit
- **D.** An ISR that is triggered by the RTC

> [!success]- Show answer
> **✅ A. A counter that can run down or up**

### Q7

**An RTC crystal or oscillator allows a CPU to**

- **A.** Provide a signal that can be exactly divided into 1 second
- **B.** Generate its main system clock
- **C.** Improve the timing of its ISRs
- **D.** Connect to peripherals that also have an RTC

> [!success]- Show answer
> **✅ A. Provide a signal that can be exactly divided into 1 second**
>
> An RTC allows accurate time keeping and scheduled wake ups.

### Q8

**Which best describes the pattern that any embedded system should follow?**

- **A.** input-output-process
- **B.** process-input-output
- **C.** process-output-input
- **D.** input-process-output

> [!success]- Show answer
> **✅ D. input-process-output**

### Q9

**Which of the following properties of embedded systems are true?**

- **A.** Necessity for strict timing constraints
- **B.** Utilization of high memory and computing power
- **C.** Dedication to a wide range of tasks
- **D.** Microcontrollers are typically used rather than microprocessors

> [!success]- Show answer
> **✅ A and D**

### Q10

**ISR code should ideally be**

- **A.** written with a proper user interface
- **B.** written in a high level language
- **C.** only executed at boot time
- **D.** small and efficient

> [!success]- Show answer
> **✅ D. small and efficient**

### Q11

**GPIO pins can support which of the following (without additional components):**

- **A.** Analogue input and digital output
- **B.** Analogue and digital input and output
- **C.** Digital input and output
- **D.** Analogue output only

> [!success]- Show answer
> **✅ C. Digital input and output**
>
> Standard GPIO pins work purely in the digital domain (reading or driving binary high/low states). Handling analogue signals directly needs dedicated on-chip peripherals like an ADC or DAC, or external circuitry.

### Q12

**GPIO pins always need to be used with an external resistor**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> Many microcontrollers include configurable internal pull-up and pull-down resistors that can be enabled in software, so an external resistor is not always required.

### Q13

**In which circumstance would it be better to NOT use GPIO pins to communicate with a peripheral:**

- **A.** When the peripheral is a long distance away
- **B.** When the peripheral specifies a very unusual custom interfacing protocol
- **C.** When you can use a built-in bus like I2C or SPI
- **D.** When the peripheral is read-only, and you can't write to it

> [!success]- Show answer
> **✅ C. When you can use a built-in bus like I2C or SPI**
>
> Dedicated hardware communication peripherals (I2C, SPI, UART) are much better than manual GPIO software bit-banging. The hardware handles timing, serialization, and buffering automatically, without putting a heavy load on the CPU.

### Q14

**Why should you only clear the interrupt flag before exiting an ISR?**

- **A.** To tidy up the stack before you leave
- **B.** To tell the user that the code has ended
- **C.** Because ISRs cannot use global variables
- **D.** To prevent the ISR immediately re-running

> [!success]- Show answer
> **✅ D. To prevent the ISR immediately re-running**
>
> If the pending interrupt flag is not cleared before exiting, the interrupt controller still sees the condition as active and triggers the ISR again as soon as execution returns to the main program.

### Q15

**Based on the lecture, which file(s) can be converted to an object file? (can be more than 1)**

- **A.** hex file
- **B.** assembly file
- **C.** c file
- **D.** decoded file
- **E.** b and c

> [!success]- Show answer
> **✅ E. b and c** (the assembly file and the C file)
>
> In standard embedded C toolchains:
>
> - A C file (`.c`) is compiled (or compiled to assembly and then assembled) into a relocatable object file (`.o` / `.obj`).
> - An assembly file (`.s` / `.asm`) is directly assembled into an object file.
> - A hex file (e.g. Intel HEX) is an end-stage output generated **after** linking object files. It is not an input used to build an object file.

### Q16

**How is the specific function of a multiplexed pin typically selected on a microcontroller?**

- **A.** Automatically by the microcontroller based on the connected external device
- **B.** It is hard-wired during manufacturing and cannot be changed
- **C.** By physically rewiring the pin to a different internal circuit
- **D.** Through software configuration, such as writing to function select registers

> [!success]- Show answer
> **✅ D. Through software configuration, such as writing to function select registers**
>
> Pin multiplexing is controlled by dedicated hardware routing that software configures (function select or alternate function registers). This lets software assign a physical pin to GPIO, UART, SPI, or PWM as needed.

### Q17

**Regarding the direction of a General Purpose Input/Output (GPIO) pin, which statement is true?**

- **A.** Input and output operations always occur on separate, dedicated pins
- **B.** A GPIO pin must be permanently set as either input or output during manufacturing
- **C.** A single GPIO pin can simultaneously be configured as both an input and an output
- **D.** Pins can be changed between input and output modes, but not simultaneously

> [!success]- Show answer
> **✅ D. Pins can be changed between input and output modes, but not simultaneously**
>
> A GPIO pin's direction is dynamic and set through direction registers (e.g. DIR or OE). Software can toggle it between input and output at runtime, but the driver hardware cannot be an active driven output and a high-impedance sensing input at the same instant.

### Q18

**Which of the following internal buses does the CPU use to specify which location or register it wishes to read from/write to:**

- **A.** USB
- **B.** Address bus
- **C.** Data bus
- **D.** Control bus

> [!success]- Show answer
> **✅ B. Address bus**
>
> - The **Address bus** carries the memory or register address the CPU wants to access.
> - The **Data bus** transfers the actual data.
> - The **Control bus** carries control and timing signals (e.g. read/write strobe, bus clock).

### Q19

**Level-triggered interrupts can be falling or rising edge.**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> Rising and falling edges are **edge-triggered** interrupts (transitions LOW → HIGH or HIGH → LOW). Level-triggered interrupts watch a continuous state or voltage condition (held active-HIGH or active-LOW).

### Q20

**Which of the following communication bus tells an internal module to read or to write?**

- **A.** CPU bus
- **B.** Control bus
- **C.** Address bus
- **D.** Data bus

> [!success]- Show answer
> **✅ B. Control bus**
>
> The Control bus carries operational signals and commands between the CPU and other modules, including the read/write (R/W) strobe lines that say whether an operation is a read or a write.

### Q21

**`17 && 0` = ?**

- **A.** TRUE
- **B.** 0
- **C.** FALSE
- **D.** 17
- **E.** 1
- **F.** -17

> [!success]- Show answer
> **✅ B and C (0 and FALSE)**
>
> In C, `&&` is the logical AND operator. Any non-zero value is logically true (so 17 is true), and 0 is logically false. Logical operators return 1 for true and 0 for false. Since one operand is false, the result is 0.

### Q22

**An interrupt is always a bad thing for a CPU**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> Interrupts are essential for real-time responsiveness. They let the CPU run tasks or enter low-power sleep while hardware peripherals work on their own, interrupting the CPU only when immediate attention is needed.

### Q23

**You would normally debounce an input key with a delay of more than 200 milliseconds**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> Switch bounce usually lasts a few milliseconds up to about 20 to 30 ms. A debounce delay is commonly set around 10 to 50 ms. Waiting more than 200 ms adds noticeable lag and could miss rapid key presses.

### Q24

**An 8-bit GPIO port must be configured to have all 8 pins as "all output" or "all input"**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> GPIO direction registers (such as DIR or TRIS) are bit-addressable with bitmasks. Individual pins in the same 8-bit port can be set independently as inputs (0) or outputs (1).

### Q25

**In the lecture, we defined the utilisation for an interrupt as u = (h+c)/T. In general, utilisation is the percentage of time when the processor is busy and h is the timing overhead for handling the interrupt. Suppose the inter-arrival time between two events is 500 cycles and the processor needs 18 cycles for either a pop or a push. For an ISR which requires 20 cycles for completion, what is its utilisation?**

- **A.** 7.6
- **B.** 11.2
- **C.** 11.6
- **D.** 4

> [!success]- Show answer
> **✅ B. 11.2**
>
> 1. **Interrupt overhead (h):** servicing an interrupt needs context saving on entry (push) and context restoring on exit (pop).
>    h = push cycles + pop cycles = 18 + 18 = **36 cycles**
> 2. **ISR execution time (c):** c = **20 cycles**
> 3. **Total active cycles per event:** h + c = 36 + 20 = **56 cycles**
> 4. **Utilisation (u):** u = (h + c) / T = 56 / 500 = 0.112 = **11.2%**

### Q26

**What is the purpose of an interrupt vector table (IVT)?**

- **A.** To store the code that causes interrupts to happen
- **B.** To store the ISR code for handling different interrupts
- **C.** To store code that prevents interrupts from happening
- **D.** To hold the address of ISRs which can handle interrupts

> [!success]- Show answer
> **✅ D. To hold the address of ISRs which can handle interrupts**
>
> The IVT does not hold the actual ISR code. It holds pointers (function addresses / branch targets) that tell the CPU where each ISR lives in memory.

### Q27

**The internal memory map of a microcontroller tells the programmer**

- **A.** When the programmer can save and load files from
- **B.** The location of registers in memory
- **C.** How to get a signal from an input pin to an output pin
- **D.** The current amount of free RAM space in the device

> [!success]- Show answer
> **✅ A and B**
>
> - **The location of registers in memory:** memory maps define the base and offset addresses for all hardware peripheral / special function registers (SFRs).
> - **Where the programmer can save and load files / code from:** memory maps lay out the non-volatile storage regions (Flash, ROM, EEPROM) where programs, constant tables, or embedded file system images (like LittleFS or FAT) are stored and accessed.

### Q28

**Interrupts are usually really simple to debug**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> Interrupts are asynchronous and non-deterministic. They bring concurrency issues, timing jitter, re-entrancy risks, and race conditions, so bugs are intermittent and hard to reproduce.

### Q29

**Most microcontrollers multiplex their GPIO pins with other functions**

*True or False?*

> [!success]- Show answer
> **✅ True**
>
> To save silicon area and package pin count, microcontrollers multiplex their physical pins across several internal peripherals (GPIO, UART, SPI, I2C, PWM) using configurable function select registers.

### Q30

**If an event is happening with a very short duration in-between 2 given occurrences, which is the best choice to take? (Assume we don't care about the power consumption)**

- **A.** Polling
- **B.** Interrupts
- **C.** Nested Interrupts
- **D.** Masking

> [!success]- Show answer
> **✅ A. Polling**
>
> When events come very close together, the overhead of interrupt context switching (pushing/popping registers on entry and exit) overwhelms the CPU. Since power is not a concern, tight polling avoids the stacking/latency overhead and reliably catches rapid consecutive states.

### Q31

**Which of the following are typically used for Machine Learning? (can be more than 1)**

- **A.** GPU
- **B.** MPU
- **C.** NPU
- **D.** MCU

> [!success]- Show answer
> **✅ A and C**
>
> GPUs and NPUs have massively parallel architectures built for matrix and vector math, so they are the main accelerators for machine learning. MPUs and MCUs can run lightweight inference (e.g. TinyML) but are not the typical dedicated ML hardware.

### Q32

**`17 && 23` = ?**

- **A.** TRUE
- **B.** 1
- **C.** 40
- **D.** 6
- **E.** 0
- **F.** FALSE

> [!success]- Show answer
> **✅ A and B (1 and TRUE)**
>
> In C, `&&` is the logical AND operator. Any non-zero integer is true, so 17 (true) and 23 (true) give true. By the C specification, logical operations produce an integer result of 1 for true.

### Q33

**A multiplex matrix is used to connect buttons to a microcontroller because**

- **A.** The layout of the QWERTY keys means you must use a matrix
- **B.** It saves on the number of pins needed
- **C.** It is a safe way to debounce input signals
- **D.** It allows LEDs to display when a key is pressed

> [!success]- Show answer
> **✅ B. It saves on the number of pins needed**
>
> A matrix lays out switches in intersecting rows and columns. Instead of 1 dedicated GPIO pin per switch (N × M pins), a matrix scans N × M switches with only N + M GPIO pins.

### Q34

**The SP is**

- **A.** a register that indicates the end of the stack
- **B.** a stack placeholder the programmer uses to remember where the stack is in memory
- **C.** a variable that contains data for a short time
- **D.** the "stack push" indicator

> [!success]- Show answer
> **✅ A. a register that indicates the end of the stack**
>
> The Stack Pointer (SP) is a hardware register that points to the current top (end boundary) of the active stack in memory.

### Q35

**How is the specific function of a multiplexed pin typically selected on a microcontroller?**

- **A.** It is hard-wired during manufacturing and cannot be changed
- **B.** By physically rewiring the pin to a different internal circuit
- **C.** Automatically by the microcontroller based on the connected external device
- **D.** Through software configuration, such as writing to function select registers

> [!success]- Show answer
> **✅ D. Through software configuration, such as writing to function select registers**
>
> Microcontrollers multiplex several internal peripheral lines onto shared physical pins. The role of each pin (GPIO, UART, SPI, PWM) is set at runtime by writing to dedicated function select registers.

### Q36

**ISR code should ideally be**

- **A.** written in a high level language
- **B.** small and efficient
- **C.** only executed at boot time
- **D.** written with a proper user interface

> [!success]- Show answer
> **✅ B. small and efficient**
>
> An ISR pauses normal program flow, so it should be as short, fast, and minimal as possible. Capture the necessary data and clear flags, then leave heavy computation to the main loop.

### Q37

**What is the purpose of an interrupt vector table (IVT)?**

- **A.** To store code that prevents interrupts from happening
- **B.** To hold the address of ISRs which can handle interrupts
- **C.** To store the ISR code for handling different interrupts
- **D.** To store the code that causes interrupts to happen

> [!success]- Show answer
> **✅ B. To hold the address of ISRs which can handle interrupts**
>
> The IVT holds pointers/addresses to the routines that service each interrupt, not the ISR code itself.

### Q38

**A CPU can __ data onto a stack and later it must __ it off again**

*(Fill in the blanks)*

> [!success]- Show answer
> **✅ push, pop**

---

## Week 3

### Q1

**Nyquist's theorem states that perfect reconstruction may be possible if the signal is sampled how fast?**

- **A.** At least 48kHz
- **B.** As fast as the ADC can sample
- **C.** With at least 2x as many bits as the signal should have
- **D.** At least 2x faster than the highest frequency component

> [!success]- Show answer
> **✅ D. At least 2x faster than the highest frequency component**

### Q2

**A synchronous communications link has a separate clock wire**

*True or False?*

> [!success]- Show answer
> **✅ True**
>
> It is the SCLK wire. Synchronous devices share a dedicated, separate clock wire.

### Q3

**An SPI master can only have one slave**

*True or False?*

> [!success]- Show answer
> **✅ False**

### Q4

**Select the two conditions that are required for Priority Inversion to occur.**

- **A.** Inheritance
- **B.** Preemptive Multitasking
- **C.** Shared Resource
- **D.** Blocking

> [!success]- Show answer
> **✅ B and C**

### Q5

**Which one of the following can convert a lower-case alphabet to an upper-case alphabet (e.g. 'a' --> 'A')**

- **A.** `upper_alphabet = lower_alphabet - 20;`
- **B.** `lower_alphabet = upper_alphabet + 0x32;`
- **C.** `lower_alphabet = upper_alphabet - 0x20;`
- **D.** `upper_alphabet = lower_alphabet - 0x20;`

> [!success]- Show answer
> **✅ D. `upper_alphabet = lower_alphabet - 0x20;`**

### Q6

**The count-down timer free-running mode means**

- **A.** Counter wraps back around to maximum after counting down to 0
- **B.** Counter counts down from maximum to 0 and then stops automatically
- **C.** Counter counts down to 0 then begins counting up
- **D.** Counter wraps back around to 0 after hitting a maximum value

> [!success]- Show answer
> **✅ A. Counter wraps back around to maximum after counting down to 0**
>
> In down-counting free-running mode, when the counter hits zero it automatically rolls over to its maximum hardware value (e.g. `0xFFFFFFFF` for 32-bit). It does not reload a custom period or stop.

### Q7

**Inside a microcontroller, a "timer" is a/an ...**

- **A.** counter that can run down or up.
- **B.** software that implements a stopwatch function.
- **C.** built-in clock unit.
- **D.** ISR that is triggered by the RTC

> [!success]- Show answer
> **✅ A and C**
>
> - **"counter that can run down or up":** this describes the register and counting hardware at the silicon level.
> - **"built-in clock unit":** in embedded terminology, hardware timers/counters count as internal, on-chip timing units rather than software implementations.

### Q8

Look at the timer diagram:

![Timer diagram: count ramps down from max to 0 repeatedly, with a flag set each time it reaches 0](data:image/png;base64,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)

1. Counter wraps around to the maximum value after counting down to zero.
2. Flag/Interrupt set when the counter wraps around 1st clock edge after zero.

**Timer_32 is configured to which mode?**

- **A.** One-shot Mode
- **B.** Periodic Timer Mode
- **C.** Continuous Mode
- **D.** Free Running Mode

> [!success]- Show answer
> **✅ D. Free Running Mode**
>
> When the counter automatically wraps to its hardware maximum upon reaching zero and keeps repeating, without halting (one-shot) or loading a custom preload value (periodic), it is in Free Running Mode.

### Q9

**CPU can execute other code while the timer is running concurrently.**

*True or False?*

> [!success]- Show answer
> **✅ True**
>
> Hardware timers are independent on-chip peripherals. They count ticks without CPU intervention, so the processor can run other tasks or sleep until an interrupt occurs.

### Q10

**A 1Hz clock is**

- **A.** directly used as the main clock
- **B.** used by the Real Time Clock
- **C.** generated using a crystal that emits a signal at 32.768kHz
- **D.** generated using a counter that counts till 2^15

> [!success]- Show answer
> **✅ B. used by the Real Time Clock**
>
> - **"used by the Real Time Clock":** true. 1 Hz (1-second period) is the standard base tick for calendar timekeeping.
> - **"generated using a crystal that emits a signal at 32.768kHz":** true. Standard low-power RTC circuits use an external 32.768 kHz tuning-fork crystal divided down to 1 Hz.
> - **"generated using a counter that counts till 2^15":** this is almost certainly 2^15 (32,768), because 32,768 Hz ÷ 2^15 = 1 Hz.
>
> > ⚠️ Your original file lists only "b" as the answer, but the reasoning says C and D are also true. Worth double-checking against your lecturer's key.

### Q11

**Using a single parity bit when transmitting a 7-bit character over EIA/RS-232, you can**

- **A.** Correct an error of 1 bit in the received character
- **B.** Indicate to the transmitter that it must re-send the character
- **C.** Detect when 1 or 3 bits are received in error
- **D.** Detect when 2 or 4 bits are received in error

> [!success]- Show answer
> **✅ C. Detect when 1 or 3 bits are received in error**
>
> A single parity bit changes state if an **odd** number of bits flip (1, 3, 5, ...). It cannot detect an even number of flips (they cancel out the parity check). It also does not give enough information to locate bits for error correction or to trigger automatic retransmission.

### Q12

**If an input signal to an ADC raises slightly above Vref, then it is**

- **A.** Clipped
- **B.** Destroyed
- **C.** Ignored until the voltage drops below Vref
- **D.** Measured as a 0

> [!success]- Show answer
> **✅ A. Clipped**
>
> When the analog input exceeds the reference voltage, the conversion result saturates at the maximum digital value, so the reading is clipped at the upper end.

### Q13

![Square wave: A is the full period, B is the high time](data:image/png;base64,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)

**'A' must be ___ to make a 40% duty-cycle, if 'B' is 30ms. (ensure you include the units too)**

*(Fill in the blank)*

> [!success]- Show answer
> **✅ 75ms**
>
> Duty cycle = B / A × 100%
>
> 40% = 30ms / A, so A = 30 / 0.4 = **75 ms**

### Q14

**How is data transmitted onto the serial console?**

- **A.** One word at a time
- **B.** One sentence at a time
- **C.** One byte at a time
- **D.** One string at a time

> [!success]- Show answer
> **✅ C. One byte at a time**
>
> At the hardware and serial console driver level (UART), characters are framed, buffered, and sent across the wire byte by byte.

### Q15

**How is data received from the serial console?**

- **A.** One byte at a time
- **B.** One word at a time
- **C.** One string at a time
- **D.** One sentence at a time

> [!success]- Show answer
> **✅ A. One byte at a time**
>
> Serial console reception buffers incoming UART frames into the hardware RX FIFO one character (one byte) at a time.

### Q16

**The I2C protocol has separate wires for transmit and receive (called MOSI and MISO)**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> MOSI and MISO are the dedicated data lines for **SPI**. I2C uses a single bidirectional data wire called SDA (along with the clock wire SCL).

### Q17

**A transducer**

- **A.** Always crosses between analogue and digital domains
- **B.** Is a digital device used to measure voltage
- **C.** Converts a signal from one form into another
- **D.** Channels the energy emitted by the Tesseract

> [!success]- Show answer
> **✅ C. Converts a signal from one form into another**
>
> This is the basic physical definition of a transducer: converting physical energy (light, temperature, pressure, motion) into an electrical signal, or vice versa.

### Q18

**Which of the following statements is true?**

- **A.** An example of an asynchronous communication is Rendezvous and a synchronous communication is Message Queue.
- **B.** An example of a synchronous communication is Rendezvous and an asynchronous communication is Message Queue.
- **C.** Both Rendezvous and Message Queue are examples of asynchronous communication.
- **D.** Both Rendezvous and Message Queue are examples of synchronous communication.

> [!success]- Show answer
> **✅ B. Rendezvous is synchronous, Message Queue is asynchronous.**
>
> - A **Rendezvous** is synchronous because the sender and receiver must both reach the synchronization point and wait for each other before data transfer happens.
> - A **Message Queue** is asynchronous because the sender can drop a message into the queue buffer and carry on without waiting for the receiver to consume it.

### Q19

**We need to provide a slave address for the UART communication protocol.**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> UART is a point-to-point connection between two endpoints (TX cross-wired to RX) with no hardware or software addressing. Slave addressing belongs to bus protocols like I2C (7-bit or 10-bit addresses).

### Q20

**Half-duplex transmission between unit A and unit B is where**

- **A.** Units A and B take turns to transmit
- **B.** Only Unit A can transmit, Unit B can only receive
- **C.** Units A and B transmit at the same time
- **D.** Unit A sends half of the data, then unit B sends the other half

> [!success]- Show answer
> **✅ A. Units A and B take turns to transmit**
>
> In half-duplex, data can travel in both directions over the shared medium, but only one unit can transmit at any given time.

### Q21

**Which one of the following can convert an upper-case alphabet to a lower-case alphabet (e.g. 'A' --> 'a')**

- **A.** `lower_alphabet = upper_alphabet + 0x32;`
- **B.** `lower_alphabet = upper_alphabet + 0x20;`
- **C.** `upper_alphabet = lower_alphabet + 0x20;`
- **D.** `upper_alphabet = lower_alphabet - 20;`

> [!success]- Show answer
> **✅ B. `lower_alphabet = upper_alphabet + 0x20;`**
>
> In ASCII, uppercase 'A' is `0x41` (65) and lowercase 'a' is `0x61` (97). The difference is 97 - 65 = 32 = `0x20` (this is bit 5). Adding `0x20` to an uppercase character gives the lowercase one.

### Q22

**A signal that ranges from 0V to 3.3V is fed into a 7-bit ADC. What is the representation of sampled analog voltage of 0.12V? (Assume the error is between 0 to 1 least significant bit)**

- **A.** `0x12`
- **B.** `001 0010b`
- **C.** `000 0110b`
- **D.** `0x04`

> [!success]- Show answer
> **✅ D. `0x04`**
>
> For a 7-bit ADC with Vref = 3.3 V, the number of quantization intervals is 2^7 = 128 (or 2^7 - 1 = 127 code transitions).
>
> - **LSB step** = 3.3 V / 128 ≈ **0.02578 V** (25.78 mV)
> - For Vin = 0.12 V: N = 0.12 V / 0.02578 V ≈ **4.654 LSBs**
> - An error between 0 and 1 LSB means standard truncation (floor).
> - floor(4.654) = 4 = **`0x04`**

### Q23

**What is the typical sample rate and data size of a CD-quality audio signal?**

- **A.** 20-bits and 24kHz
- **B.** 44.1kHz and 16-bits
- **C.** 256kHz and 32-bits
- **D.** 8-bits and 48kHz

> [!success]- Show answer
> **✅ B. 44.1kHz and 16-bits**
>
> The standard Red Book CD-DA specification defines audio sampled at 44.1 kHz with 16 bits per sample.

### Q24

**Select which of the following occurs within an ADC (tick all that apply):**

- **A.** Inversion
- **B.** Transduction
- **C.** Sampling
- **D.** Quantisation

> [!success]- Show answer
> **✅ C and D (Sampling and Quantisation)**
>
> - **Sampling:** the ADC's sample-and-hold circuit discretizes the continuous-time analog signal at specific clock intervals.
> - **Quantisation:** the ADC converts the sampled voltage amplitude into discrete digital binary codes.
> - Transduction is done outside the ADC by a physical sensor (transducer), and inversion is an analog op-amp / signal conditioning function. Neither is a core ADC process.

### Q25

**A 7-bit ASCII data is transmitted with 1 start bit, 1 stop bit and 1 parity bit at a baud rate of 9600. How fast is the 7-bit ASCII data being transmitted?**

- **A.** 0.104 ms
- **B.** 1.042 ms
- **C.** 0.115 ms
- **D.** 1.146 ms

> [!success]- Show answer
> **✅ B. 1.042 ms**
>
> - **Total bits per character frame:** 1 (start) + 7 (data) + 1 (parity) + 1 (stop) = **10 bits**
> - **Full character frame time:** T_frame = 10 bits / 9600 bps ≈ 0.00104167 s = **1.042 ms**
> - **Bit time:** T_bit = 1 bit / 9600 bps ≈ 0.00010417 s = **0.104 ms**

### Q26

**An ADC should quantise a continuous signal differently to how it quantises a single value.**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> The quantisation rule and transfer function of an ADC are memoryless and fixed. Each instantaneous voltage sample maps onto the discrete scale the same way, whether it is an isolated DC voltage or part of a time-varying waveform.

### Q27

**An RTC crystal or oscillator allows a CPU to**

- **A.** provide a signal that can be exactly divided into 1 second
- **B.** improve the timing of its ISRs
- **C.** connect to peripherals that also have an RTC
- **D.** directly drive its main system clock

> [!success]- Show answer
> **✅ A. provide a signal that can be exactly divided into 1 second**
>
> RTC crystals run at 32.768 kHz (2^15 Hz). Passing this through a 15-stage binary divider (2^15) gives an exact 1 Hz tick (1 second period) used for calendar tracking.

### Q28

**Quantization process is generally __________ and results in information __________ .**

- **A.** reversible, gain
- **B.** reversible, loss
- **C.** irreversible, loss
- **D.** irreversible, gain

> [!success]- Show answer
> **✅ C. irreversible, loss**
>
> Quantisation maps an infinite range of continuous analog voltages onto a finite set of discrete digital levels. Many different analog voltages round to the same binary code, so the original values cannot be recovered without quantisation error. That makes it lossy and irreversible.

### Q29

**A _____ count is used to obtain a 1 second timer from a 30MHz clock? (enter numerical answer)**

*(Fill in the blank)*

> [!success]- Show answer
> **✅ 30000000**
>
> Ticks = Frequency × Time = 30 MHz × 1 s = **30,000,000**

### Q30

**When referring to an ADC, quantisation means**

- **A.** The number of discrete levels used to represent a sample
- **B.** The biggest (maximum) value that the sample can reach
- **C.** The sample rate at which a sample is converted to digital
- **D.** The sample is converted to analogue by a DAC

> [!success]- Show answer
> **✅ A. The number of discrete levels used to represent a sample**
>
> Quantisation sets the resolution (the number of digital codes/steps, e.g. 2^12 = 4096 levels for a 12-bit ADC) used to approximate the continuous sampled voltage.

### Q31

![Square wave: A is the full period, B is the high time](data:image/png;base64,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)

**If 'B' is 333ms, what is 'A' to make a 40% Duty-Cycle?**

- **A.** 666ms
- **B.** 667ms
- **C.** 555ms
- **D.** 832.5ms

> [!success]- Show answer
> **✅ D. 832.5ms**
>
> Duty cycle = B / A × 100%
>
> 0.40 = 333 ms / A, so A = 333 ms / 0.40 = **832.5 ms**

### Q32

**Which digital communication protocol is asynchronous?**

- **A.** UART
- **B.** SPI
- **C.** I2C
- **D.** All of them.

> [!success]- Show answer
> **✅ A. UART**
>
> UART (Universal Asynchronous Receiver-Transmitter) has no shared clock line. Devices sync frame timing using pre-set baud rates and start/stop framing bits. SPI and I2C both use a dedicated clock line (SCLK / SCL), so they are synchronous.

### Q33

**A task can acquire more than one lock (Mutex).**

*True or False?*

> [!success]- Show answer
> **✅ True**
>
> A single task in an RTOS can acquire several mutexes in sequence to guard several shared resources at once (as long as lock ordering is kept consistent to avoid deadlocks).

### Q34

**In EIA/RS-232, what indicates that a character is about to be transmitted**

- **A.** The chip select (CS) line
- **B.** A start bit
- **C.** The header is transmitted first
- **D.** Transmit indicator wire

> [!success]- Show answer
> **✅ B. A start bit**
>
> In UART/RS-232, the line idles HIGH (mark state). A character transmission begins when the line drops LOW for exactly one bit period, which is the start bit.

---

## Week 4

### Q1

**If a task calls the following function, it will always be placed into a "blocked" state for 250ms: `vTaskDelay(250);`**

*True or False?*

> [!success]- Show answer
> **✅ False**
>
> In FreeRTOS, `vTaskDelay(ticks)` takes the delay in system **ticks**, not milliseconds. Unless the tick rate (`configTICK_RATE_HZ`) is exactly 1000 Hz (1 tick = 1 ms), passing 250 will not delay for 250 ms. To delay for an exact number of milliseconds regardless of tick rate, convert with the macro `pdMS_TO_TICKS(250)`.

### Q2

**A task is the smallest sequence of instructions that can be managed independently by an/a ....**

- **A.** scheduler
- **B.** interrupt
- **C.** inter-process communication (IPC)
- **D.** task control block (TCB)

> [!success]- Show answer
> **✅ A. scheduler**
>
> By the standard OS/RTOS definition, a task (or thread) is the smallest independently executable sequence of instructions managed and dispatched by the scheduler. The TCB only stores the task's context/state. The scheduler does the actual managing, preemption, and CPU allocation.

### Q3

**Which service/feature is the most important for multi-tasking to work in an EOS?**

- **A.** memory management
- **B.** inter-process communication
- **C.** context switch
- **D.** polling

> [!success]- Show answer
> **✅ C. context switch**
>
> Multi-tasking on a single core needs the kernel to save the execution state (registers, program counter, stack pointer) of the running task and restore the state of another. Without context switching, the CPU cannot switch between tasks or interleave them.

### Q4

**Why should an FSM action avoid waiting for an operation to complete?**

- **A.** Waiting causes the state variable to be deleted
- **B.** Waiting prevents the FSM from handling other events promptly
- **C.** Waiting changes the priority of every RTOS task
- **D.** Waiting automatically resets the FSM

> [!success]- Show answer
> **✅ B. Waiting prevents the FSM from handling other events promptly**
>
> FSM actions must be non-blocking and quick. If an action waits for an operation to finish, the state machine stalls and cannot process incoming inputs, alarms, or urgent events. The standard pattern is to start the work in the action and handle its completion or timeout as a separate, later event.

### Q5

**How should an FSM normally be structured within an RTOS application?**

- **A.** Use one owning task that receives and dispatches events
- **B.** Allow every event source to modify the state directly
- **C.** Create one task for every state
- **D.** Execute the entire FSM inside an interrupt service routine

> [!success]- Show answer
> **✅ A. Use one owning task that receives and dispatches events**
>
> An FSM in an RTOS should live in a single owning task fed by an event queue. One owning task serializes event handling, which removes race conditions on the state variable without needing mutexes. One task per state is an anti-pattern (wastes memory, adds scheduling overhead). Running the whole FSM in an ISR breaks the rule to keep ISRs minimal and non-blocking.

### Q6

**Which application is the strongest fit for a finite state machine?**

- **A.** Sorting a fixed array
- **B.** Multiplying two matrices
- **C.** Converting temperatures from Celsius to Fahrenheit
- **D.** Managing communication retries and timeouts

> [!success]- Show answer
> **✅ D. Managing communication retries and timeouts**
>
> FSMs suit event-driven, reactive systems with sequences of operating modes, protocols, handshakes, timeouts, and fault handling, where the next behaviour depends on past history. Maths (matrix multiplication, temperature conversion) and linear algorithms (sorting) are plain data-flow pipelines with no persistent modes or asynchronous events.

### Q7

**Which of the following are considerations when selecting an RTOS? [Select 2]**

- **A.** Features
- **B.** Decompilers
- **C.** Debuggers
- **D.** Machine Learning

> [!success]- Show answer
> **✅ A and C**
>
> Key criteria include supported **Features** (preemptive scheduling, synchronization primitives, memory protection, memory footprint, peripheral/networking stacks) and toolchain support like **Debuggers** (RTOS-aware debugging to watch task stacks, states, and execution traces). Decompilers are reverse-engineering tools, and Machine Learning is an application domain, not a core OS selection criterion.

### Q8

**Task functions typically do not call `return()`**

*True or False?*

> [!success]- Show answer
> **✅ True**
>
> An RTOS task is normally an independent, continuous thread written as an infinite loop (`for (;;)` or `while (1)`) that waits on events and processes work. A task function must never simply `return;`. Exiting without deleting itself through a kernel call (e.g. `vTaskDelete(NULL)` or `tk_ext_tsk()`) leads to undefined behaviour or a crash, because the processor tries to pop a return address that does not exist off the task stack.

### Q9

**Which Task State occurs the least amount of time?**

- **A.** Terminated
- **B.** Ready
- **C.** Blocked
- **D.** Running

> [!success]- Show answer
> **✅ A. Terminated**
>
> Tasks keep moving between Running, Ready, and Blocked during normal operation. The Terminated (or Dormant/Dead) state is reached only once, when a task exits or is deleted, so it is by far the least frequent state.

### Q10

**What primarily determines how a finite state machine responds?**

- **A.** The current event only
- **B.** The current state and event
- **C.** The current state only
- **D.** The number of previous events

> [!success]- Show answer
> **✅ B. The current state and event**
>
> An FSM's behaviour depends on both memory and input: **Next response = current state + event.** The current state gives the context (operating mode), and the event triggers the action and state transition.

### Q11

**In a state machine, what is the purpose of a guard?**

- **A.** To record the current state
- **B.** To generate an event
- **C.** To determine whether a transition is permitted
- **D.** To perform the transition's action

> [!success]- Show answer
> **✅ C. To determine whether a transition is permitted**
>
> A guard is a boolean condition that must be true for an event to trigger a transition. If the guard is false, the transition is blocked, and the machine ignores the event or stays in its current state.

### Q12

**What should an interrupt service routine or timer callback do when it detects an FSM-related event?**

- **A.** Wait until the FSM becomes idle
- **B.** Enqueue or notify the owning task of the event
- **C.** Run the entire state machine directly
- **D.** Create a new task for the event

> [!success]- Show answer
> **✅ B. Enqueue or notify the owning task of the event**
>
> ISRs and timer callbacks must stay short and non-blocking. Following "capture now, process later", an ISR should never run the FSM logic directly or block waiting for idle. It captures the event, enqueues/signals it to the single task that owns the FSM, and returns immediately.

### Q13

**Which Priority Driven Scheduling Algorithm is dynamic-priority based?**

- **A.** Deadline Monotonic Scheduling (DMS)
- **B.** Least Slack Time (LST)
- **C.** Rate Monotonic Scheduling (RMS)
- **D.** Earliest Deadline First (EDF)

> [!success]- Show answer
> **✅ B and D (LST and EDF)**
>
> - **Static / fixed-priority:** priorities are set before execution from fixed task parameters. RMS gives higher priority to higher-frequency (shorter period) tasks. DMS gives higher priority to shorter relative deadlines.
> - **Dynamic-priority:** priorities are recomputed at runtime. EDF gives the highest priority to the task with the nearest absolute deadline. LST gives the highest priority to the task with the smallest slack (t_deadline - t_remaining).
>
> If the question is single-choice, **EDF** is the classic textbook dynamic-priority algorithm.

### Q14

**Which of the following statements is true based on the following code snippet?**

```c
xTaskCreate(avg_task, "AvgThread", configMINIMAL_STACK_SIZE, NULL,
            TEST_TASK_PRIORITY, NULL);
xTaskCreate(simpleaverage, "TempThread", configMINIMAL_STACK_SIZE, NULL,
            TEST_TASK_PRIORITY, NULL);
xTaskCreate(printfunction, "AvgThread", configMINIMAL_STACK_SIZE, NULL,
            TEST_TASK_PRIORITY, &printtask);
```

- **A.** Priority-based scheduling will occur
- **B.** The program will crash at the instruction: `vTaskStartScheduler()`
- **C.** The program will crash at the instruction: `for( ; ; )`
- **D.** Round-robin scheduling will occur

> [!success]- Show answer
> **✅ D. Round-robin scheduling will occur**
>
> All three tasks are created with the same priority (`TEST_TASK_PRIORITY`). When tasks share a priority in FreeRTOS, the scheduler shares CPU time among them using round-robin time slicing (switching on every system tick). The duplicate task name `"AvgThread"` does not cause a crash. FreeRTOS only uses names as debug strings.

### Q15

**Which of the following is NOT a characteristic of a Real-Time Operating System (RTOS)?**

- **A.** High throughput for batch processing
- **B.** Minimal interrupt latency
- **C.** Deterministic response time
- **D.** Priority-based scheduling

> [!success]- Show answer
> **✅ A. High throughput for batch processing**
>
> An RTOS puts predictability, determinism, bounded context-switch times, and minimal interrupt latency ahead of raw throughput. Maximizing batch throughput is typical of general-purpose operating systems (Linux/Windows), where average speed matters more than worst-case latency guarantees.

### Q16

**Consider the following FSM rule:**

```c
case S_IDLE:
    if (e == EV_START && ready())
        state = S_RUN;
```

**What happens if `EV_START` occurs while `ready()` returns false?**

- **A.** The FSM moves to an error state
- **B.** The FSM remains in `S_IDLE`
- **C.** The FSM restarts automatically
- **D.** The FSM moves to `S_RUN`

> [!success]- Show answer
> **✅ B. The FSM remains in `S_IDLE`**
>
> `ready()` acts as a guard condition. With the logical AND (`&&`), if `ready()` is false the whole condition is false. With no matching else branch, no state assignment runs, so the FSM stays in `S_IDLE`.

### Q17

**How should an FSM normally be structured within an RTOS application?**

- **A.** Create one task for every state
- **B.** Execute the entire FSM inside an interrupt service routine
- **C.** Use one owning task that receives and dispatches events
- **D.** Allow every event source to modify the state directly

> [!success]- Show answer
> **✅ C. Use one owning task that receives and dispatches events**
>
> In an RTOS, an FSM should sit inside a single owning task that takes events from a queue or notification. This keeps state transitions strictly serialized, avoiding race conditions and the need for complex mutex synchronization on the state variable.

### Q18

**Which application is the strongest fit for a finite state machine?**

- **A.** Converting temperatures from Celsius to Fahrenheit
- **B.** Sorting a fixed array
- **C.** Managing communication retries and timeouts
- **D.** Multiplying two matrices

> [!success]- Show answer
> **✅ C. Managing communication retries and timeouts**
