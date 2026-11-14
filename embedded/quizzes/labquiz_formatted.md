# Lab Quiz Bank (Weeks 1 to 6)

**How to use this:** Read the question, think of your answer, then open the **Show answer** callout to check. Questions are numbered per week, in the same order as your original file.

**Contents**

- [Week 1](#week-1) (7 questions): C pitfalls, operator precedence, compiler warnings
- [Week 2](#week-2) (10 questions): GPIO, UART, string parsing, struct padding
- [Week 3](#week-3) (10 questions): interrupts, timing, polling, `volatile`
- [Week 4](#week-4) (5 questions): PWM, Nyquist, integer division
- [Week 5](#week-5) (10 questions): FreeRTOS, mutexes, priority inversion
- [Week 6](#week-6) (10 questions): optimisation, undefined behaviour, RP2040 performance

---

## Week 1

### Q1

**What does `printf("%d\n", a > b);` print in C?**

- **A.** `true` or `false`
- **B.** Undefined: `>` returns a `bool`
- **C.** `1` or `0`
- **D.** The larger of the two values

> [!success]- Show answer
> **✅ C. `1` or `0`**

### Q2

**How does the compiler parse `mask & 1u << pin == 0`?**

- **A.** `(mask & (1u << (pin == 0)))`
- **B.** `((mask & 1u) << (pin == 0))`
- **C.** `(mask & ((1u << pin) == 0))`
- **D.** `((mask & (1u << pin)) == 0)`

> [!success]- Show answer
> **✅ C. `(mask & ((1u << pin) == 0))`**
>
> C uses a set of rules called operator precedence. From highest to lowest:
>
> 1. `<<` or `>>`, bitwise shift
> 2. `==`, equality
> 3. `&`, bitwise AND

### Q3

**What happens when `count_bits()` is called with `value = -1`?**

```c
uint8_t count_bits(int value) {
    uint8_t count = 0;
    while (value) { count += value & 1; value >>= 1; }
    return count;
}
```

- **A.** Returns 0
- **B.** Returns 32
- **C.** Returns 1
- **D.** Loops forever

> [!success]- Show answer
> **✅ D. Loops forever**
>
> In C, negative ints are stored in two's complement, so -1 is all 1s in a 32-bit int. When `value >>= 1` runs on a signed negative integer, it keeps the sign by shifting a 1 in from the left. Since a 1 is shifted in, the value stays at -1 and the loop never ends.
>
> Fix: copy the value into an unsigned variable first.

### Q4

**GCC reports an error on line 32 but the actual defect is a missing semicolon on line 30. Why?**

- **A.** The parser only fails once it reaches a token that cannot continue the statement
- **B.** Line 30 is inside a macro
- **C.** The error is genuinely on line 32
- **D.** GCC line numbers are off by two in older versions

> [!success]- Show answer
> **✅ A. The parser only fails once it reaches a token that cannot continue the statement**

### Q5

**Why does the original `blink` example do nothing on a Pico W?**

- **A.** The Pico W needs a higher supply voltage for the LED
- **B.** The Pico W has no LED
- **C.** `PICO_DEFAULT_LED_PIN` is undefined for `pico_w`, so the code compiles out
- **D.** `gpio_put()` does not work on the Pico W

> [!success]- Show answer
> **✅ C. `PICO_DEFAULT_LED_PIN` is undefined for `pico_w`, so the code compiles out**

### Q6

**With `int c = 5;`, what does this print?**

```c
printf("%d\n", c++);
printf("%d\n", ++c);
```

- **A.** `5` then `6`
- **B.** `5` then `7`
- **C.** `6` then `6`
- **D.** `6` then `7`

> [!success]- Show answer
> **✅ B. `5` then `7`**

### Q7

**Which of these does `gcc -Wall` report in `if(a=2048) a==0;`?**

- **A.** Both, as two separate warnings
- **B.** Neither, both are valid C
- **C.** The assignment only
- **D.** The comparison only

> [!success]- Show answer
> **✅ A. Both, as two separate warnings**

---

## Week 2

### Q1

**A button is wired with a pull-up resistor. What does `gpio_get()` return while it is pressed?**

- **A.** `1`, because pressing completes the circuit
- **B.** It depends on the button
- **C.** `0`, because pressing connects the pin to ground
- **D.** Undefined until the pin is debounced

> [!success]- Show answer
> **✅ C. `0`, because pressing connects the pin to ground**

### Q2

**`picow_blink_button.c` works even if you delete `gpio_init(BTN_PIN)`. Why is it there anyway?**

- **A.** It is required on the Pico W but not the Pico
- **B.** It has no effect and could be removed
- **C.** Without it the code depends on RP2040 reset defaults rather than stating intent
- **D.** It prevents floating inputs

> [!success]- Show answer
> **✅ C. Without it the code depends on RP2040 reset defaults rather than stating intent**

### Q3

**Why does the command parser forbid `strtok()`?**

- **A.** It keeps its position in a hidden `static`, so it breaks with two links or an interrupt
- **B.** It is not available in the Pico SDK
- **C.** It cannot handle repeated separators
- **D.** It allocates memory

> [!success]- Show answer
> **✅ A. It keeps its position in a hidden `static`, so it breaks with two links or an interrupt**

### Q4

**At 115200 baud, 8N1, roughly how long does one character take on the wire?**

- **A.** 8.7 µs
- **B.** 86.8 µs
- **C.** 868 µs
- **D.** 1.15 ms

> [!success]- Show answer
> **✅ B. 86.8 µs**
>
> - **8N1 protocol:** 10 bits per character in total (1 start bit + 8 data bits + 0 parity bits + 1 stop bit).
> - **Baud rate:** 115,200 baud means 115,200 bits per second.
> - **Time per bit:** 1 / 115,200 ≈ 8.68 µs
> - **Time per character:** 10 × (1 / 115,200) ≈ 0.0000868 s = **86.8 µs**

### Q5

**Why is `payload[5] << 24` undefined behaviour when `payload` is `uint8_t[]`?**

- **A.** `uint8_t` cannot be shifted
- **B.** The shift count exceeds the width of `uint8_t`
- **C.** It is not undefined; it is merely slow
- **D.** It promotes to signed `int`, so any byte > 0x80 shifts into the sign bit

> [!success]- Show answer
> **✅ D. It promotes to signed `int`, so any byte > 0x80 shifts into the sign bit**

### Q6

**Connecting a Pico's UART TX pin directly to its own RX pin creates what, and why is it useful?**

- **A.** A short circuit; it is not useful
- **B.** Nothing, TX cannot drive RX
- **C.** A full-duplex link; it doubles throughput
- **D.** A loopback; it lets you test the UART with one board

> [!success]- Show answer
> **✅ D. A loopback; it lets you test the UART with one board**

### Q7

**Why can you not put an oscilloscope on the Pico W's on-board LED?**

- **A.** The LED is inside the RP2040 package
- **B.** You can, on GP25
- **C.** It is driven by the CYW43 over SPI, not by an RP2040 pin
- **D.** The voltage is too low to measure

> [!success]- Show answer
> **✅ C. It is driven by the CYW43 over SPI, not by an RP2040 pin**
>
> The GPIO-OUTPUT section of your notes mentions that you cannot put one on it.

### Q8

**The parser's test harness asserts `argv[0] == line`. What is that checking?**

- **A.** That `argv` was allocated correctly
- **B.** That the line was tokenised in place, not copied
- **C.** That the line starts with a valid verb
- **D.** That the first word is not empty

> [!success]- Show answer
> **✅ B. That the line was tokenised in place, not copied**
>
> `line` is the memory address of the start of the input string buffer, and `argv[0]` is the memory address of the first parsed token.

### Q9

**`sizeof(reading_t)` is 12, but the wire format specifies 9 payload bytes. Why?**

```c
typedef struct { uint16_t sensor_id; uint8_t status;
                 int16_t temp_c_x10; uint32_t timestamp_ms; } reading_t;
```

- **A.** `sizeof` includes a hidden type tag
- **B.** The compiler inserts padding so each field is naturally aligned
- **C.** `uint32_t` is 8 bytes on ARM
- **D.** The specification is wrong

> [!success]- Show answer
> **✅ B. The compiler inserts padding so each field is naturally aligned**

### Q10

**Passing a plain `char` straight to `toupper()` is undefined behaviour for some inputs. On which build does the problem actually appear?**

- **A.** On the Pico, because `char` is signed on ARM
- **B.** On neither, it is a myth
- **C.** On both equally
- **D.** On the x86 host, because `char` is signed there and unsigned on ARM

> [!success]- Show answer
> **✅ D. On the x86 host, because `char` is signed there and unsigned on ARM**
>
> On ARM, GCC defaults `char` to an unsigned 8-bit int. On x86, GCC makes `char` a signed 8-bit int.

---

## Week 3

### Q1

**Why can a level-triggered interrupt (`GPIO_IRQ_LEVEL_LOW`) hang the system when an edge-triggered one does not?**

- **A.** They are slower to service
- **B.** The condition persists, so the interrupt re-fires continuously and starves the main program
- **C.** Level triggers require a pull-down resistor
- **D.** Level triggers are not supported on the RP2040

> [!success]- Show answer
> **✅ B. The condition persists, so the interrupt re-fires continuously and starves the main program**
>
> The `true` / `false` argument is used when we are setting interrupts on or off:
>
> ```c
> gpio_set_irq_enabled_with_callback(ENCODER_PIN, GPIO_IRQ_EDGE_FALL|GPIO_IRQ_EDGE_RISE, false, &encoder_isr);
> ```
>
> - **Edge triggered** (`GPIO_IRQ_EDGE_RISE`): reacts to momentary events. When the voltage drops (e.g. something touches the sensor), the interrupt fires once. For it to fire again, the voltage must go back up, reset, and drop a second time.
> - **Level triggered** (`GPIO_IRQ_LEVEL_LOW`): reacts to a continuous state. If a wheel slot blocks the sensor for 20 ms, the ISR fires and finishes in 1 µs, then immediately checks again. The sensor is still blocked (20 ms is far longer than 1 µs), so the ISR runs again, causing an infinite loop.

### Q2

**In `gpio_set_irq_enabled_with_callback(2, GPIO_IRQ_EDGE_FALL, true, &gpio_callback)`, what is the fourth argument?**

- **A.** A pointer to a function, so the SDK can call your code later
- **B.** The name of the ISR, resolved by the linker at run time
- **C.** A reference, which C requires for functions
- **D.** The address of a variable holding the callback's result

> [!success]- Show answer
> **✅ A. A pointer to a function, so the SDK can call your code later**

### Q3

**Why is the ultrasonic round-trip time divided by two?**

- **A.** To compensate for sensor latency
- **B.** Because the sound travels to the object and back, so the one-way distance is half
- **C.** To convert microseconds to milliseconds
- **D.** Because the sensor fires two pulses

> [!success]- Show answer
> **✅ B. Because the sound travels to the object and back, so the one-way distance is half**
>
> > ⚠️ The answer line was blank in your original file. B is the option that fits, but worth confirming.

### Q4

**A `while (pulse_count < 20)` loop works at `-O0` and hangs at `-O3`. Why?**

- **A.** `-O3` reorders the ISR
- **B.** The compiler proved nothing in the loop changes `pulse_count`, and hoisted the read out
- **C.** `-O3` disables interrupts
- **D.** The counter overflows at higher speed

> [!success]- Show answer
> **✅ B. The compiler proved nothing in the loop changes `pulse_count`, and hoisted the read out**
>
> No `volatile` keyword (bug 1 task a).

### Q5

**What happens in `while (gpio_get(echo_pin) == 0);` if no echo ever arrives?**

- **A.** It never terminates, and the program hangs with no error
- **B.** It returns 0
- **C.** It times out after 1 second
- **D.** The watchdog resets the board

> [!success]- Show answer
> **✅ A. It never terminates, and the program hangs with no error**

### Q6

**Why must an ISR be kept short?**

- **A.** It runs at high priority and typically masks other interrupts, so everything else waits
- **B.** Long ISRs consume more power
- **C.** ISRs have a hard time limit enforced by the hardware
- **D.** ISRs cannot contain loops

> [!success]- Show answer
> **✅ A. It runs at high priority and typically masks other interrupts, so everything else waits**

### Q7

**Blocking for an echo from a 2 m object costs roughly how much CPU time at 125 MHz?**

- **A.** ~11.7 ms, roughly 1.4 million instructions
- **B.** ~12 µs, about 1500 instructions
- **C.** ~117 ms, roughly 14 million instructions
- **D.** Nothing, the CPU sleeps

> [!success]- Show answer
> **✅ A. ~11.7 ms, roughly 1.4 million instructions**

### Q8

**Which statement about polling intervals is correct?**

- **A.** The interval has no effect on lag
- **B.** A longer interval reduces lag but costs more power
- **C.** A longer interval increases lag but reduces CPU and power cost
- **D.** A shorter interval reduces both lag and CPU cost

> [!success]- Show answer
> **✅ C. A longer interval increases lag but reduces CPU and power cost**

### Q9

**A 32-bit microsecond timer (`time_us_32()`) wraps after approximately how long?**

- **A.** 71 minutes
- **B.** It does not wrap
- **C.** 12 hours
- **D.** 49 days

> [!success]- Show answer
> **✅ A. 71 minutes**

### Q10

**`ENC_IDLE` never appears to work in this switch. Why?**

```c
case ENC_IDLE:
    slot_start_us = now;
    state = ENC_SLOT;

case ENC_SLOT:
    slot_width_us = now - slot_start_us;
    pulse_count++;
    state = ENC_IDLE;
    break;
```

- **A.** `state` should be `volatile`
- **B.** There is no `break` after the first case, so control falls through into the second
- **C.** The cases are in the wrong order
- **D.** `ENC_IDLE` is 0 and evaluates false

> [!success]- Show answer
> **✅ B. There is no `break` after the first case, so control falls through into the second**

---

## Week 4

### Q1

**What determines the RP2040's PWM output frequency?**

- **A.** `pwm_set_gpio_level()`
- **B.** Both together: f = f_sys / (clkdiv × (wrap + 1))
- **C.** `pwm_set_wrap()` alone
- **D.** `pwm_set_clkdiv()` alone

> [!success]- Show answer
> **✅ B. Both together: f = f_sys / (clkdiv × (wrap + 1))**

### Q2

**`uint32_t divider = clock_freq / (freq * 65536);` with `clock_freq = 125e6` and `freq = 100.0f`. What is stored?**

- **A.** `19.073`
- **B.** `20`
- **C.** `19`
- **D.** `0`

> [!success]- Show answer
> **✅ C. `19`**

### Q3

**Sampling a 20 Hz signal every 25 ms means sampling at what rate, relative to Nyquist?**

- **A.** Below Nyquist
- **B.** Nyquist does not apply to square waves
- **C.** Exactly at the Nyquist rate: 40 Hz, precisely 2× the fundamental
- **D.** Well above Nyquist, 25 ms is plenty

> [!success]- Show answer
> **✅ C. Exactly at the Nyquist rate: 40 Hz, precisely 2× the fundamental**

### Q4

**`duty_from_adc(raw, wrap)` is specified so that `duty_from_adc(2048, 999) == 499`. Why does `return (raw / ADC_MAX) * wrap;` return 0?**

- **A.** `ADC_MAX` should be 4096
- **B.** `wrap` is too small
- **C.** `raw / ADC_MAX` is integer division and is 0 for every `raw` below 4095
- **D.** The multiplication overflows

> [!success]- Show answer
> **✅ C. `raw / ADC_MAX` is integer division and is 0 for every `raw` below 4095**

### Q5

**What is `(uint16_t)(duty_cycle * 65536)` when `duty_cycle` is `1.0f`?**

- **A.** `1`
- **B.** `0`
- **C.** `65535`
- **D.** `65536`

> [!success]- Show answer
> **✅ B. `0`**

---

## Week 5

### Q1

**Which is legal inside an ISR?**

- **A.** `xSemaphoreTake(m, portMAX_DELAY)`
- **B.** `vTaskDelay(pdMS_TO_TICKS(1))`
- **C.** `xQueueSendFromISR(q, &v, &woken)`
- **D.** `printf("event\n")`

> [!success]- Show answer
> **✅ C. `xQueueSendFromISR(q, &v, &woken)`**

### Q2

**`configMINIMAL_STACK_SIZE` is measured in what?**

- **A.** Words
- **B.** Kilobytes
- **C.** Stack frames
- **D.** Bytes

> [!success]- Show answer
> **✅ A. Words**

### Q3

**Why use a mutex rather than a binary semaphore to protect a shared resource?**

- **A.** Binary semaphores cannot be used from tasks
- **B.** Mutexes are faster
- **C.** There is no practical difference
- **D.** Mutexes have an owner and support priority inheritance

> [!success]- Show answer
> **✅ D. Mutexes have an owner and support priority inheritance**

### Q4

**In unbounded priority inversion, what is the high-priority task actually waiting for?**

- **A.** A medium-priority task that does not use the resource at all
- **B.** The scheduler tick
- **C.** An interrupt to complete
- **D.** The low-priority task's critical section only

> [!success]- Show answer
> **✅ A. A medium-priority task that does not use the resource at all**

### Q5

**In FreeRTOS, which task runs?**

- **A.** The lowest-numbered priority that is Ready
- **B.** The highest-priority task that is Ready, immediately
- **C.** Tasks run round-robin regardless of priority
- **D.** Whichever has waited longest

> [!success]- Show answer
> **✅ B. The highest-priority task that is Ready, immediately**

### Q6

**A task in the Blocked state consumes how much CPU time?**

- **A.** It depends on its priority
- **B.** None at all
- **C.** A small amount, polling its condition
- **D.** The same as Ready

> [!success]- Show answer
> **✅ B. None at all**

### Q7

**What unit does `vTaskDelay()` take?**

- **A.** Microseconds
- **B.** Seconds
- **C.** Ticks
- **D.** Milliseconds

> [!success]- Show answer
> **✅ C. Ticks**

### Q8

**How many tasks may send to, and receive from, a FreeRTOS message buffer?**

- **A.** One sender, any number of receivers
- **B.** Any number of both
- **C.** Exactly one sender and exactly one receiver
- **D.** Any number of senders, one receiver

> [!success]- Show answer
> **✅ C. Exactly one sender and exactly one receiver**

### Q9

**What is the main advantage of a gatekeeper task over a mutex for a shared `printf`?**

- **A.** It is faster
- **B.** It guarantees message ordering by priority
- **C.** Senders never block on the resource, so no priority inversion or deadlock is possible on it
- **D.** It uses less RAM

> [!success]- Show answer
> **✅ C. Senders never block on the resource, so no priority inversion or deadlock is possible on it**

### Q10

**`xTaskDelayUntil()` fixes which problem, and which does it leave?**

- **A.** Fixes both
- **B.** Fixes drift; leaves jitter
- **C.** Fixes jitter; leaves drift
- **D.** Fixes neither; it is a convenience wrapper

> [!success]- Show answer
> **✅ B. Fixes drift; leaves jitter**

---

## Week 6

### Q1

**`pid.c` never converges even after the syntax errors are fixed. Which omission is most responsible?**

- **A.** `usleep()` is not included
- **B.** The integral term is not clamped
- **C.** `current_value += motor_response;` is missing, so the plant never responds
- **D.** `Kd` is too small

> [!success]- Show answer
> **✅ C. `current_value += motor_response;` is missing, so the plant never responds**

### Q2

**How do you stop the compiler from deleting a benchmark loop?**

- **A.** Add `printf` inside the loop
- **B.** Make the result observable: write to a `volatile` variable
- **C.** Compile at `-O0` and report those numbers
- **D.** Declare the function `static`

> [!success]- Show answer
> **✅ B. Make the result observable: write to a `volatile` variable**

### Q3

**Undefined behaviour often appears only when optimisation is enabled. Why?**

- **A.** Optimisers contain more bugs
- **B.** The compiler may assume UB never happens and optimise on that assumption
- **C.** `-O2` disables run-time checks that `-O0` performs
- **D.** Higher optimisation changes the memory layout

> [!success]- Show answer
> **✅ B. The compiler may assume UB never happens and optimise on that assumption**

### Q4

**Why does this loop take almost no time at `-O2`?**

```c
void timeConsumingTask(void) {
  for (int i = 0; i < 1000000; i++) { int result = i * 2; }
}
```

- **A.** The compiler vectorises the multiplication
- **B.** `result` is unused, so the assignment, then the loop, then the call are all removed
- **C.** `int` multiplication is a single cycle
- **D.** One million iterations is genuinely fast

> [!success]- Show answer
> **✅ B. `result` is unused, so the assignment, then the loop, then the call are all removed**

### Q5

**Which instrument finds "a variable is being corrupted, but I cannot find what writes to it"?**

- **A.** A hardware data watchpoint on that address
- **B.** A breakpoint on every function
- **C.** `printf` after each function
- **D.** Rebuilding at `-O0`

> [!success]- Show answer
> **✅ A. A hardware data watchpoint on that address**
>
> A data watchpoint is a dedicated silicon-level hardware debug feature that monitors a specific memory address for reads or writes.

### Q6

**Why is floating-point arithmetic disproportionately expensive on the RP2040?**

- **A.** The RP2040 clock is too slow
- **B.** The Cortex-M0+ has no FPU, so every operation is a software library call
- **C.** `float` is 64 bits on ARM
- **D.** It is not, the M0+ has a fast FPU

> [!success]- Show answer
> **✅ B. The Cortex-M0+ has no FPU, so every operation is a software library call**

### Q7

**Why is integer division also expensive on the RP2040?**

- **A.** The compiler refuses to optimise division
- **B.** Division is always slower than multiplication on all processors
- **C.** ARMv6-M has no divide instruction, so `/` becomes a library call
- **D.** It is not expensive

> [!success]- Show answer
> **✅ C. ARMv6-M has no divide instruction, so `/` becomes a library call**

### Q8

**`absolute_time_diff_us()` returns what type, and how should it be printed?**

- **A.** `int`, with `%d`
- **B.** `int64_t`, with `PRId64` from `<inttypes.h>`
- **C.** `float`, with `%f`
- **D.** `uint32_t`, with `%u`

> [!success]- Show answer
> **✅ B. `int64_t`, with `PRId64` from `<inttypes.h>`**

### Q9

**A table-driven CRC-8 replaces eight iterations per byte with one. What does it cost?**

- **A.** Nothing, it is strictly better
- **B.** 256 bytes of table storage
- **C.** Accuracy: the table version is approximate
- **D.** It only works for one polynomial length

> [!success]- Show answer
> **✅ B. 256 bytes of table storage**

### Q10

**What is wrong with `compute_pid(setpoint, current_value, &integral, prev_error);` where the parameter is `float *prev_error`?**

- **A.** `&integral` should also be by value
- **B.** Nothing
- **C.** `prev_error` is passed by value where a pointer is expected, a type error
- **D.** The argument order is wrong

> [!success]- Show answer
> **✅ C. `prev_error` is passed by value where a pointer is expected, a type error**
