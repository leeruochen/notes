# Bug Hunt Drill 1 — RP2040 Hardware, Wire Parsing & C Traps

Tags: `#embedded-c` `#rp2040` `#cortex-m0plus` `#bughunt` `#quiz-prep`

> [!info] System Specification (Target Context)
> - **Target:** RP2040 (32-bit ARMv6-M, little-endian), compiled at `-O3`.
> - **Wire Packet Format (`FRAME_PAYLOAD = 8`):** `[0xA5][LEN][PAYLOAD (8 bytes)][CHECKSUM (1 byte)]`
>   - Bytes `0..1`: `sensor_id` (`uint16_t`, big-endian)
>   - Bytes `2..3`: `voltage_mv` (`uint16_t`, big-endian)
>   - Bytes `4..7`: `timestamp_ms` (`uint32_t`, big-endian)
>   - `CHECKSUM`: 8-bit sum of the 8 payload bytes only.
> - **Hardware:** `adc_dma_isr()` fires asynchronously when a conversion completes; ADC input `1` (`GP27`) reads a 12-bit value (`0..4095`) against a `3300 mV` reference.

---

## 1. Complete Fixed Code (With Inline Fix Comments)

> [!success] Fixed Implementation (`drill1_fixed.c`)
> ```c
> #include <stdint.h>
> #include <stdbool.h>
> #include <stdio.h>
> #include "pico/stdlib.h"
> #include "hardware/adc.h"
> 
> #define FRAME_SOF     0xA5u
> #define FRAME_PAYLOAD 8u
> 
> typedef struct {
>     uint16_t sensor_id;
>     uint16_t voltage_mv;
>     uint32_t timestamp_ms;
> } packet_t;
> 
> // FIX 1 (Missing volatile): adc_dma_isr() writes this asynchronously outside
> // read_millivolts()'s control flow. Without volatile, -O3 hoists the read out
> // of the while loop into a register once and hangs forever.
> static volatile bool sample_ready = false;
> 
> void adc_dma_isr(void)
> {
>     sample_ready = true;
> }
> 
> uint16_t read_millivolts(void)
> {
>     while (!sample_ready) {
>         tight_loop_contents();
>     }
>     sample_ready = false;
> 
>     // FIX 2 (Stateful peripheral order): RP2040 has one ADC behind a multiplexer.
>     // Select input channel 1 BEFORE calling adc_read(), otherwise adc_read()
>     // converts whichever channel was selected previously.
>     adc_select_input(1);
>     uint16_t raw = adc_read();
> 
>     // FIX 3 (16-bit intermediate truncation): raw * 3300u reaches 13,513,500,
>     // which exceeds uint16_t max (65,535) for raw >= 20. Keep the intermediate
>     // product in uint32_t and narrow to uint16_t only after dividing by 4095u.
>     return (uint16_t)(((uint32_t)raw * 3300u) / 4095u);
>     //uint16_t scaled = raw * 3300u; return scaled / 4095u; originally
> }
> 
> char *format_status(uint16_t mv)
> {
>     // FIX 4 (Dangling pointer to stack local): A non-static local array is
>     // destroyed when the function returns (GCC often replaces the return with NULL).
>     // Making it static keeps it alive (note: non-reentrant; passing a caller-owned
>     // buffer `char *buf, size_t size` is preferred if multiple tasks call this).
>     static char msg[32];
>     snprintf(msg, sizeof(msg), "ADC1 = %u mV", mv);
>     return msg;
> }
> 
> bool parse_packet(const uint8_t *buf, uint16_t buf_len, packet_t *out)
> {
>     if (buf_len < 3u || buf[0] != FRAME_SOF) {
>         return false;
>     }
> 
>     uint8_t len = buf[1];
> 
>     // FIX 5 (Untrusted wire length / out-of-bounds read): Never trust len from
>     // an external packet before indexing buf[2 + i] or buf[2 + len].
>     // - len != FRAME_PAYLOAD ensures the 8 payload bytes exist and match the spec.
>     // - (uint16_t)len + 3u > buf_len prevents reading past buf_len (cast to uint16_t
>     //   avoids 8-bit overflow if len >= 253).
>     if (len != FRAME_PAYLOAD || (uint16_t)len + 3u > buf_len) {
>         return false;
>     }
> 
>     // NOTE (Not a bug): uint8_t sum intentionally wraps modulo 256 because the
>     // spec defines CHECKSUM as an 8-bit sum of the payload bytes.
>     uint8_t sum = 0;
>     for (uint8_t i = 0; i < len; i++) {
>         sum += buf[2 + i];
>     }
> 
>     if (sum != buf[2 + len]) {
>         return false;
>     }
> 
>     out->sensor_id  = ((uint16_t)buf[2] << 8) | (uint16_t)buf[3];
>     out->voltage_mv = ((uint16_t)buf[4] << 8) | (uint16_t)buf[5];
> 
>     // FIX 6 (Unaligned access + wrong endianness + strict aliasing):
>     // *(uint32_t *)&buf[6] fails in three ways on RP2040:
>     //   1. &buf[6] is not 4-byte aligned -> HardFault on Cortex-M0+.
>     //   2. RP2040 is little-endian, reversing the big-endian wire bytes.
>     //   3. Casts away const and violates strict aliasing.
>     // Assemble byte-by-byte after casting each byte to uint32_t before shifting.
>     out->timestamp_ms = ((uint32_t)buf[6] << 24) |
>                         ((uint32_t)buf[7] << 16) |
>                         ((uint32_t)buf[8] << 8)  |
>                         ((uint32_t)buf[9]);
>     return true;
> }
> 
> uint32_t compute_buffer_sum(const uint8_t *data, uint16_t count)
> {
>     uint32_t total = 0;
> 
>     // FIX 7 (Loop counter narrower than bound): uint8_t i wraps from 255 back
>     // to 0, making `i < count` true forever whenever count >= 256. Match the
>     // counter width (uint16_t) to count.
>     for (uint16_t i = 0; i < count; i++) {
>         total += data[i];
>     }
>     return total;
> }
> ```

---
# Bug Hunt Drill 2 — PWM, IIR Filters, PID & FreeRTOS Pipelines

Tags: `#embedded-c` `#rp2040` `#freertos` `#pid` `#bughunt` `#quiz-prep`

> [!info] System Specification (Target Context)
> * **Target & Kernel:** RP2040 (`SYS_CLK_HZ = 125,000,000`), FreeRTOS with `configTICK_RATE_HZ = 1000` (`1 tick = 1 ms`) and `configMINIMAL_STACK_SIZE = 256` words (`1024` bytes).
> * **PWM Slice:** Hardware counter runs from `0` up to `wrap` **inclusive** (`wrap + 1` ticks per period).
> * **IIR Low-Pass Filter:** Implements $y_n = y_{n-1} + \frac{x_n - y_{n-1}}{16}$ (`ALPHA_SHIFT = 4`), tracking both rising and falling inputs smoothly without wrapping or stalling.
> * **Discrete PID Controller:** Computes $u(t) = K_p e(t) + K_i \int e(t)\,dt + K_d \frac{de(t)}{dt}$ with step `dt` in seconds, driving `measured` toward `setpoint`.
> * **Telemetry Pipeline:** `sensor_task` samples an `int16_t` temperature (in tenths of a degree) every **50 ms** without drift and delivers **every** sample to both `window_task` and `control_task`.

## 1. Complete Fixed Code (With Inline Fix Comments)

> [!success] Fixed Implementation (`drill2_fixed.c`)
> ```c
> #include <stdint.h>
> #include <stdbool.h>
> #include <stdio.h>
> #include "FreeRTOS.h"
> #include "task.h"
> #include "message_buffer.h"
> 
> #define SYS_CLK_HZ       125000000u
> #define SAMPLE_PERIOD_MS 50u
> #define ALPHA_SHIFT      4u
> 
> // FIX 1 (Two readers on one message buffer): A FreeRTOS message buffer is
> // strictly single-producer, single-consumer (1 sender, 1 receiver). Sharing one
> // buffer splits samples between tasks and corrupts the single waiting-task slot.
> // Create one dedicated message buffer per consumer task.
> static MessageBufferHandle_t window_mbuf;
> static MessageBufferHandle_t control_mbuf;
> 
> /* 1. PWM wrap calculation */
> uint16_t calc_pwm_wrap(uint32_t target_hz, float clk_div)
> {
>     uint32_t ticks = (uint32_t)((float)SYS_CLK_HZ / (clk_div * (float)target_hz));
> 
>     // FIX 2 (PWM wrap off-by-one): The RP2040 PWM counter counts 0..wrap
>     // INCLUSIVE, which equals (wrap + 1) clock ticks per period. Subtract 1.
>     return (uint16_t)(ticks - 1u);
> }
> 
> /* 2. Single-channel IIR low-pass filter */
> uint32_t iir_filter(uint32_t x)
> {
>     // NOTE (Valid C - NOT a bug): Because this is a single-channel filter called
>     // by one owner, `static uint32_t y = 0;` is required to retain state across calls.
>     static uint32_t y = 0;
> 
>     // FIX 3 (Unsigned underflow + floor-rounding deadband):
>     // 1. Subtracting uint32_t values when x < y wraps around to ~4.29e9 and shoots
>     //    the filter upward. Cast both operands to signed int32_t first.
>     // 2. Signed `diff >> 4` rounds toward -infinity (15 >> 4 == 0), stalling 15
>     //    counts short of the target when rising. Round away from zero when positive.
>     int32_t diff = (int32_t)x - (int32_t)y;
>     int32_t step = (diff >= 0)
>                  ? (diff + (int32_t)((1u << ALPHA_SHIFT) - 1u)) >> ALPHA_SHIFT
>                  : (diff >> ALPHA_SHIFT);
>     y += (uint32_t)step;
>     return y;
> }
> 
> /* 3. Discrete PID controller step (dt in seconds) */
> float compute_pid(float setpoint, float measured,
>                   float *integral, float *prev_error, float dt)
> {
>     float Kp = 1.5f, Ki = 0.25f, Kd = 0.05f;
> 
>     // FIX 4 (Inverted error sign): `measured - setpoint` produces negative feedback
>     // when below target, driving the actuator in the wrong direction (positive-
>     // feedback runaway). Always compute `setpoint - measured`.
>     float error = setpoint - measured;
> 
>     // FIX 5 (Missing discrete dt scaling): `dt` was passed into compute_pid()
>     // and never used. Discrete integration multiplies by `dt` (area), and
>     // discrete differentiation divides by `dt` (slope).
>     *integral += error * dt;
>     float derivative = (error - *prev_error) / dt;
> 
>     // NOTE (Valid C - NOT a bug): `prev_error` and `integral` are `float *`
>     // pointers passed from the caller so state persists across iterations.
>     // `*prev_error` and `*integral` properly dereference those pointers.
>     *prev_error = error;
> 
>     return (Kp * error) + (Ki * (*integral)) + (Kd * derivative);
> }
> 
> /* 4. Sensor producer task: must run every 50 ms without drift */
> void sensor_task(void *pvParameters)
> {
>     // NOTE (Valid C - NOT a bug): FreeRTOS requires every task function to accept
>     // `void *pvParameters`. Casting `(void)pvParameters;` intentionally silences
>     // the unused-parameter warning when NULL is passed in xTaskCreate().
>     (void)pvParameters;
> 
>     // FIX 6 (Wrong delay unit + cumulative timing drift):
>     // `vTaskDelay(1)` delays for 1 tick (1 ms at 1000 Hz, 50x too fast) and
>     // adds loop execution time to every period. Record the initial tick count
>     // once and use `vTaskDelayUntil()` with `pdMS_TO_TICKS(SAMPLE_PERIOD_MS)`.
>     TickType_t next_wake = xTaskGetTickCount();
> 
>     for (;;) {
>         int16_t sample = read_temp_tenths();
> 
>         // FIX 7 (Sending pointer size instead of variable size):
>         // `sizeof(&sample)` is 4 bytes (pointer size on 32-bit ARM), whereas
>         // `sample` is 2 bytes (`int16_t`). Sending a 4-byte message causes a
>         // 2-byte `xMessageBufferReceive()` to return 0 forever without reading.
>         xMessageBufferSend(window_mbuf,  &sample, sizeof(sample), 0);
>         xMessageBufferSend(control_mbuf, &sample, sizeof(sample), 0);
> 
>         vTaskDelayUntil(&next_wake, pdMS_TO_TICKS(SAMPLE_PERIOD_MS));
>     }
> }
> 
> /* 5. Consumer task A: processes every sample */
> void window_task(void *pvParameters)
> {
>     (void)pvParameters;
>     int16_t val = 0;
>     for (;;) {
>         if (xMessageBufferReceive(window_mbuf, &val, sizeof(val), portMAX_DELAY) == sizeof(val)) {
>             process_window(val);
>         }
>     }
> }
> 
> /* 6. Consumer task B: runs PID and prints status once per second */
> void control_task(void *pvParameters)
> {
>     (void)pvParameters;
>     int16_t val = 0;
>     float integral = 0.0f, prev_err = 0.0f;
>     TickType_t last_print = xTaskGetTickCount();
> 
>     for (;;) {
>         if (xMessageBufferReceive(control_mbuf, &val, sizeof(val), portMAX_DELAY) == sizeof(val)) {
>             float u = compute_pid(25.0f, (float)val / 10.0f, &integral, &prev_err, 0.05f);
>             TickType_t now = xTaskGetTickCount();
>             if ((TickType_t)(now - last_print) >= pdMS_TO_TICKS(1000)) {
>                 printf("Temp=%.2f C, Control=%.2f\n", (double)val / 10.0, (double)u);
>                 last_print = now;
>             }
>         }
>     }
> }
> 
> void app_start(void)
> {
>     window_mbuf  = xMessageBufferCreate(256);
>     control_mbuf = xMessageBufferCreate(256);
> 
>     xTaskCreate(sensor_task,  "sensor", 512,  NULL, 4, NULL);
>     xTaskCreate(window_task,  "window", 512,  NULL, 3, NULL);
> 
>     // FIX 8 (Stack too small for printf with %f): `configMINIMAL_STACK_SIZE`
>     // is 256 words (1024 bytes), which overflows when `printf` formats floats
>     // (`%.2f`). Allocate 1024 words (4096 bytes) for tasks calling float printf.
>     xTaskCreate(control_task, "ctrl",   1024, NULL, 3, NULL);
>     vTaskStartScheduler();
> }
> ```

# Bug Hunt Drill 3 — Quiz Format Practice (MCQ, Multi-Select, Fill-in-the-Blank & Short Answer)

Tags: `#embedded-c` `#rp2040` `#cortex-m0plus` `#freertos` `#bughunt` `#quiz-prep`

> [!info] Drill Focus
> This note mirrors the exact quiz structure—**Multiple-Choice (MCQ)**, **Multi-Select**, **Fill-in-the-Blank**, and **Short Answer**—covering bitwise precedence, integer promotion, `-O3` compiler optimisations, binary wire serialisation, ISR torn reads, and FreeRTOS concurrency traps.

---

## 1. Multiple-Choice Questions (MCQs)

### Q1. Bitwise Operator Precedence (`is_bit_clear`)

> [!bug] Broken Code
> ```c
> bool is_bit_clear(uint32_t reg, unsigned bit)
> {
>     return (reg & 1u << bit == 0);
> }
> ```

* **Answer:** **C** — `is_bit_clear(0x04u, 2)` and `is_bit_clear(0x04u, 3)` **both return `false`**.
* **Why it fails:** In C, operator precedence from highest to lowest is `<<`, then `==`, then `&`. The compiler parses the expression as `reg & ((1u << bit) == 0)`. Because `1u << bit` is a power of two (never `0`), `(1u << bit) == 0` always evaluates to `0`, collapsing the entire expression to `reg & 0` (`false`).

> [!success] Fixed Code
> ```c
> bool is_bit_clear(uint32_t reg, unsigned bit)
> {
>     return (reg & (1u << bit)) == 0;
> }
> ```

---

### Q2. Integer Promotion in Byte Unpacking (`unpack_be32`)

> [!bug] Broken Code
> ```c
> uint32_t unpack_be32(const uint8_t p[4])
> {
>     return (p[0] << 24) | (p[1] << 16) | (p[2] << 8) | p[3];
> }
> ```

* **Answer:** **B** — Invokes **Undefined Behaviour** whenever `p[0] >= 0x80`.
* **Why it fails:** Operands narrower than `int` (`uint8_t`) undergo **integer promotion** to signed 32-bit `int` before `<<` executes. Shifting `0x80` left by 24 places produces `0x80000000`, which shifts a `1` into the sign bit and overflows signed `int`.

> [!success] Fixed Code
> ```c
> uint32_t unpack_be32(const uint8_t p[4])
> {
>     return ((uint32_t)p[0] << 24) |
>            ((uint32_t)p[1] << 16) |
>            ((uint32_t)p[2] <<  8) |
>            ((uint32_t)p[3]);
> }
> ```

---

### Q3. FreeRTOS Synchronisation: Mutex vs. Binary Semaphore

* **Scenario:** `Task_High` (priority 4) and `Task_Low` (priority 1) share an SPI bus guarded by a **binary semaphore**, while `Task_Med` (priority 2) runs CPU-bound work without touching SPI.
* **Answer:** **B** — **Unbounded priority inversion**.
* **Why it fails:** A binary semaphore has no concept of ownership or priority inheritance. While `Task_Low` holds the semaphore and `Task_High` blocks waiting for it, `Task_Med` can preempt `Task_Low` indefinitely, indirectly stalling `Task_High`.
* **Exact Fix:** Guard the shared bus with a **mutex** (`xSemaphoreCreateMutex()`), which implements **priority inheritance** (temporarily boosting `Task_Low` to priority 4 until it releases the lock), or move SPI ownership into a dedicated **gatekeeper task** fed by a queue.

---

### Q4. Compiler Optimisation (`-O3` Empty Delay Loop)

> [!bug] Broken Code
> ```c
> void sensor_settle_delay(void)
> {
>     for (uint32_t i = 0; i < 6000; i++) {
>         ;
>     }
> }
> ```

* **Answer:** **B** — At `-O3`, GCC compiles the entire function down to a single `bx lr` instruction (immediate return, $0\ \mu\text{s}$ delay).
* **Why it fails:** The loop modifies no `volatile` state and produces no observable side effects, so the optimiser deletes it completely.
* **Exact Fix:** Use a hardware-timer delay (`busy_wait_us_32(500)` or `sleep_us(500)` on the RP2040 SDK), which guarantees the minimum duration regardless of optimisation level or clock settings.

---

## 2. Multi-Select Questions

### Q5. UART Binary Framing & Struct Transmission (`send_status`)

> [!bug] Broken Code
> ```c
> typedef struct {
>     uint16_t node_id;
>     uint8_t  flags;
>     uint32_t uptime_ms;
> } status_frame_t;
>
> void send_status(uart_inst_t *uart, const status_frame_t *st)
> {
>     uart_set_translate_crlf(uart, true);
>     const uint8_t *raw = (const uint8_t *)st;
>     for (size_t i = 0; i < sizeof(status_frame_t); i++) {
>         uart_putc(uart, raw[i]);
>     }
> }
> ```

* **Correct Selections:** **A, B, and C**
  * **A (Struct alignment padding):** `sizeof(status_frame_t)` is **8 bytes** instead of 7 because the compiler inserts 1 padding byte after `flags` so `uptime_ms` starts on a 4-byte boundary.
  * **B (Wrong endianness):** Casting `st` to `const uint8_t *` transmits multi-byte fields in the RP2040's native **little-endian** memory order instead of the protocol's big-endian wire order.
  * **C (UART CRLF translation):** `uart_set_translate_crlf(uart, true)` injects an extra `0x0D` (`\r`) byte before any binary payload byte equal to `0x0A` (`\n`), stretching the frame and breaking receiver alignment.

> [!success] Fixed Code
> ```c
> void send_status(uart_inst_t *uart, const status_frame_t *st)
> {
>     uart_set_translate_crlf(uart, false);
>
>     uint8_t wire[7] = {
>         (uint8_t)(st->node_id >> 8),
>         (uint8_t)(st->node_id),
>         st->flags,
>         (uint8_t)(st->uptime_ms >> 24),
>         (uint8_t)(st->uptime_ms >> 16),
>         (uint8_t)(st->uptime_ms >>  8),
>         (uint8_t)(st->uptime_ms)
>     };
>
>     for (size_t i = 0; i < sizeof(wire); i++) {
>         uart_putc_raw(uart, wire[i]);
>     }
> }
> ```

---

### Q6. GPIO Capture ISR & Shared 64-Bit Timestamp (`capture_isr`)

> [!bug] Broken Code
> ```c
> static volatile uint32_t pulse_width_us  = 0;
> static volatile uint64_t last_capture_us = 0;
> static volatile uint32_t last_edge_us    = 0;
> static volatile bool     in_pulse        = false;
>
> void capture_isr(uint gpio, uint32_t events)
> {
>     uint32_t now = time_us_32();
>     if (now > last_edge_us + 500u) {
>         last_edge_us = now;
>         switch (in_pulse) {
>             case false:
>                 last_capture_us = time_us_64();
>                 in_pulse = true;
>             case true:
>                 pulse_width_us = now - (uint32_t)last_capture_us;
>                 in_pulse = false;
>                 break;
>         }
>     }
> }
>
> uint64_t get_last_timestamp(void)
> {
>     return last_capture_us;
> }
> ```

* **Correct Selections:** **A, B, and C**
  * **A (Timer rollover addition trap):** `now > last_edge_us + 500u` overflows near the $\approx 71.6\text{-minute}$ ($2^{32}\ \mu\text{s}$) rollover of `time_us_32()`. Use wrap-safe unsigned subtraction: `(uint32_t)(now - last_edge_us) > 500u`.
  * **B (Missing `break;` fallthrough):** Omitting `break;` at the end of `case false:` causes immediate fallthrough into `case true:` on the first edge, setting `pulse_width_us = 0` and resetting `in_pulse = false`.
  * **C (64-bit torn read on a 32-bit core):** `volatile` prevents register caching but **does not provide atomicity**. On a 32-bit Cortex-M0+, reading `uint64_t last_capture_us` takes two 32-bit `ldr` instructions; if `capture_isr` fires between them, `get_last_timestamp()` returns half of the old timestamp and half of the new one.

> [!tip] Clarifying Why `pulse_width_us` Needs `volatile` (Option E)
> Whether `pulse_width_us` is written in `case true:` or read inside `capture_isr()` does not determine `volatile`. What matters is that `capture_isr()` is an **asynchronous interrupt handler** never called directly by `main()`. Any variable written inside an ISR and read outside that ISR (such as in `main()`) **must** be qualified with `volatile` so `-O3` cannot cache its value in a register.

> [!success] Fixed Code
> ```c
> #include "hardware/sync.h"
>
> void capture_isr(uint gpio, uint32_t events)
> {
>     uint32_t now = time_us_32();
>     if ((uint32_t)(now - last_edge_us) > 500u) {
>         last_edge_us = now;
>         switch (in_pulse) {
>             case false:
>                 last_capture_us = time_us_64();
>                 in_pulse = true;
>                 break;
>             case true:
>                 pulse_width_us = now - (uint32_t)last_capture_us;
>                 in_pulse = false;
>                 break;
>         }
>     }
> }
>
> uint64_t get_last_timestamp(void)
> {
>     uint32_t saved = save_and_disable_interrupts();
>     uint64_t snapshot = last_capture_us;
>     restore_interrupts(saved);
>     return snapshot;
> }
> ```

---

## 3. Fill-in-the-Blanks (Exact Code Fixes)

### Q7. Signed Right-Shift Infinite Loop
```c
uint8_t count_ones(int value)
{
    uint32_t v = (uint32_t)value;
    uint8_t count = 0;
    while (v) {
        count += v & 1u;
        v >>= 1;
    }
    return count;
}
```
> [!warning] Common Mistake: Using `int8_t` Instead of `uint32_t`
> Casting to `int8_t` fails in two ways: (1) `int8_t` is still **signed**, so `-1` (`0xFF`) performs an **arithmetic right shift** that copies the `1` sign bit forever in an infinite loop; (2) narrowing a 32-bit `int` to 8 bits discards the upper 24 bits and returns `8` instead of `32`. Always cast to unsigned `uint32_t` so `>>= 1` performs a **logical shift** (filling bit 31 with `0`).

### Q8. Heap String Duplication
```c
char *dup_label(const char *src)
{
    char *dst = malloc(strlen(src) + 1);
    if (dst != NULL) {
        strcpy(dst, src);
    }
    return dst;
}
```
* **Why:** `strlen(src)` excludes the null terminator `'\0'`; `+ 1` prevents a 1-byte heap buffer overflow when `strcpy()` writes `'\0'`.

### Q9. Ring-Buffer Moving Average Warm-Up
```c
float moving_avg_c = (float)sum / (float)fill / 10.0f;
```
* **Why:** Dividing by `fill` (the number of samples collected so far, $1 \le \text{fill} \le \text{RING\_LEN}$) rather than constant `RING_LEN` prevents under-reporting during the initial warm-up window, and floating-point division preserves tenths of a degree.

### Q10. Atomic 64-Bit Read in `main()`
```c
uint32_t saved = save_and_disable_interrupts();
uint64_t snapshot = shared_ts;
restore_interrupts(saved);
```
* **Why:** Masks interrupts across the two 32-bit `ldr` instructions needed to read a `uint64_t` on Cortex-M0+, preventing a torn read.

---

## 4. Short-Answer Model Solutions

### Q11. Uninitialised CRC & Unused Ready Flag
* **Why it fails:**
  1. An uninitialised local `uint8_t crc;` starts from indeterminate leftover stack garbage instead of `0x00`, causing unpredictable checksum failures depending on prior stack usage.
  2. If `crc8()` is called before `crc_init()`, `crc_table[256]` is still all zeros, so every table lookup returns `0` and `crc8()` outputs `0x00` for every input (accepting any corrupted frame whose CRC byte happens to be `0`).
* **Exact Fix:** Initialise `uint8_t crc = 0x00u;` at the start of `crc8()`, and add `if (!crc_ready) crc_init();` before the lookup loop (or precompute `static const uint8_t crc_table[256]` in flash).

### Q12. Running Mean Precision Drift
* **Why it fails:** A 32-bit IEEE-754 `float` has only a **24-bit mantissa** ($2^{24} = 16,777,216$). As `acc` grows without bound (crossing $2^{24}$ after roughly 55 minutes at $10\ \text{Hz}$), small incoming sample increments can no longer be represented accurately and get rounded off, causing the mean to drift away from the true value.
* **Exact Fix:** Accumulate the sum in an exact **`int64_t acc`** and the sample count in a **`uint64_t n`**, then convert to `double` only at the moment of division (`(float)((double)acc / (double)n / 10.0)`).

### Q13. FreeRTOS Queue Timeouts & Software Timers
* **Why `portMAX_DELAY` in a software timer callback is dangerous:** FreeRTOS software timer callbacks run in the context of the **timer service task** (`Tmr Svc`). Blocking indefinitely on a full queue stalls the timer service task and freezes **every software timer in the system**.
* **Why timeout `0` in a normal task loses data:** Passing a `0`-tick timeout makes `xQueueSend()` return immediately with `errQUEUE_FULL` whenever `print_q` is full, silently discarding the message unless the return value is checked and handled.

---
# Bug Hunt Drill 4 — GPIO Wiring, Interrupts, Timing & Shared Task State

Tags: `#embedded-c` `#rp2040` `#cortex-m0plus` `#freertos` `#bughunt` `#quiz-prep`

> [!info] System Specification (Target Context)
> 
> - **Architecture & Build:** RP2040 (32-bit ARMv6-M, little-endian), compiled at `-O3`.
>     
> - **GPIO Wiring:**
>     
>     - `BTN_PIN` (`GP14`) is a push-button wired to **GND** with an **internal pull-up resistor** (`gpio_pull_up`).
>         
>     - `ENC_PIN` (`GP15`) connects to a 20-slot optical wheel encoder. A slot pulse starts on a **falling edge** (`HIGH -> LOW`) and ends on a **rising edge** (`LOW -> HIGH`), with slot width measured between those two transitions.
>         
> - **Timing:** `time_us_32()` is a 32-bit free-running microsecond counter (wraps every $\approx 71.6\text{ minutes}$); `time_us_64()` is a 64-bit microsecond counter.
>     
> - **Tasks & Shared Data:**
>     
>     - `encoder_isr()` measures slot widths, increments `edge_count`, updates `last_edge_ts`, and sends slot widths to `width_queue`.
>         
>     - `logger_task` (priority 3) receives slot widths from `width_queue` and tracks the average slot width.
>         
>     - `monitor_task` (priority 2) waits until `TARGET_EDGES` occur, reads the final 64-bit timestamp, polls the push-button, and tracks the average battery voltage ($\text{mV}$).
>
## 4. Complete Fixed Code (With Inline Fix Comments)

> [!success] Fixed Implementation (`drill4_fixed.c`)
> ```c
> #include <stdint.h>
> #include <stdbool.h>
> #include <stdio.h>
> #include "pico/stdlib.h"
> #include "hardware/gpio.h"
> #include "hardware/sync.h"
> #include "FreeRTOS.h"
> #include "task.h"
> #include "queue.h"
> 
> #define BTN_PIN       14u
> #define ENC_PIN       15u
> // NOTE (Valid C - NOT a bug): 500 us (0.5 ms) is an appropriate debounce window
> // for a 20-slot optical encoder (allows up to ~6,000 RPM). 50,000 us (50 ms) would
> // be too long and would swallow valid optical pulses.
> #define DEBOUNCE_US   500u
> #define TARGET_EDGES  40u
> 
> typedef enum {
>     ENC_IDLE = 0,
>     ENC_IN_SLOT
> } enc_state_t;
> 
> static QueueHandle_t width_queue;
> 
> // FIX 1 (Missing volatile on ISR-shared counter): encoder_isr() increments
> // edge_count asynchronously while monitor_task polls it in a while loop.
> // Without volatile, -O3 hoists the read into a register once and spins forever.
> static volatile uint32_t    edge_count       = 0;
> static volatile uint64_t    last_edge_ts     = 0;
> static volatile uint32_t    last_debounce_us = 0;
> static volatile uint32_t    slot_start_us    = 0;
> static volatile enc_state_t enc_state        = ENC_IDLE;
> 
> // FIX 2 (Shared static state across two tasks + 32-bit overflow):
> // A helper with `static uint32_t acc` and `static uint32_t count` called by both
> // logger_task (widths in us) and monitor_task (battery in mV) mixes two unrelated
> // data streams together, creates a data race on preemption, and risks 32-bit
> // overflow. Eliminate the shared helper and give each task its own local 64-bit state.
>// uint32_t update_running_avg(uint32_t sample)
>uint32_t update_running_avg(uint32_t sample) { 
>static uint32_t acc = 0; 
>static uint32_t count = 0; 
>acc += sample; 
>count++; 
>return acc / count; }
>// ^^^^ this code is wrong, update_running_avg
>
>/* GPIO Interrupt Service Routine */
> void encoder_isr(uint gpio, uint32_t events)
> {
>     (void)gpio;
>     (void)events;
>     uint32_t now = time_us_32();
> 
>     // FIX 3 (Timer rollover addition trap): `now > last_debounce_us + DEBOUNCE_US`
>     // overflows 32 bits near the ~71.6-minute time_us_32() rollover (or locks out
>     // edges for 71 minutes if `now` wraps first). Use wrap-safe unsigned subtraction.
>     if ((uint32_t)(now - last_debounce_us) > DEBOUNCE_US) {
>         last_debounce_us = now;
>         last_edge_ts = time_us_64();
>         edge_count++;
> 
>         switch (enc_state) {
>             case ENC_IDLE:
>                 slot_start_us = now;
>                 enc_state = ENC_IN_SLOT;
>                 // FIX 4 (Missing break fallthrough): Without `break;`, the very
>                 // first edge falls straight into ENC_IN_SLOT, computing
>                 // width_us = now - now = 0 and resetting enc_state to ENC_IDLE.
>                 break;
> 
>             case ENC_IN_SLOT: {
>                 uint32_t width_us = now - slot_start_us;
> 
>                 // FIX 5 (Blocking queue call inside an ISR): An ISR cannot block
>                 // (portMAX_DELAY is illegal in interrupt context). Use the non-blocking
>                 // ISR-safe API and yield if a higher-priority task was woken.
>                 BaseType_t xHigherPriorityTaskWoken = pdFALSE;
>                 xQueueSendFromISR(width_queue, &width_us, &xHigherPriorityTaskWoken);
>                 portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
> 
>                 enc_state = ENC_IDLE;
>                 break;
>             }
>         }
>     }
> }
> 
> /* High-priority consumer task (priority 3) */
> void logger_task(void *pvParameters)
> {
>     (void)pvParameters;
>     uint32_t width = 0;
>     uint64_t width_sum = 0;
>     uint64_t width_count = 0;
> 
>     for (;;) {
>         if (xQueueReceive(width_queue, &width, portMAX_DELAY) == pdPASS) {
>             width_sum += width;
>             width_count++;
>             uint32_t avg_w = (uint32_t)(width_sum / width_count);
>             printf("Slot width: %lu us (avg: %lu us)\n",
>                    (unsigned long)width, (unsigned long)avg_w);
>         }
>     }
> }
> 
> /* Lower-priority monitor task (priority 2) */
> void monitor_task(void *pvParameters)
> {
>     (void)pvParameters;
>     uint64_t batt_sum = 0;
>     uint64_t batt_count = 0;
> 
>     while (edge_count < TARGET_EDGES) {
>         tight_loop_contents();
>     }
> 
>     // FIX 6 (Live ISR + 64-bit torn read on a 32-bit core):
>     // 1. Disable the encoder IRQ once TARGET_EDGES is reached so the ISR stops
>     //    overwriting last_edge_ts behind monitor_task's back.
>     // 2. Even if an IRQ were still active, reading a 64-bit uint64_t on Cortex-M0+
>     //    takes two 32-bit `ldr` instructions that an ISR can split (torn read).
>     //    Wrap the read in a critical section for guaranteed atomicity.
>     gpio_set_irq_enabled_with_callback(
>         ENC_PIN, GPIO_IRQ_EDGE_FALL | GPIO_IRQ_EDGE_RISE, false, &encoder_isr);
> 
>     uint32_t saved = save_and_disable_interrupts();
>     uint64_t completed_at = last_edge_ts;
>     restore_interrupts(saved);
> 
>     printf("Reached target edges at %llu us\n", (unsigned long long)completed_at);
> 
>     for (;;) {
>         // FIX 7 (Inverted active-low pull-up button logic): BTN_PIN is wired to
>         // GND with gpio_pull_up() enabled. Unpressed = HIGH (1), Pressed = LOW (0).
>         if (gpio_get(BTN_PIN) == 0) {
>             uint32_t batt_mv = read_battery_mv();
>             batt_sum += batt_mv;
>             batt_count++;
>             uint32_t avg_mv = (uint32_t)(batt_sum / batt_count);
>             printf("Button pressed! Battery avg: %lu mV\n", (unsigned long)avg_mv);
>         }
> 
>         // NOTE (Valid C - NOT a bug): Polling a manual push-button every ~100 ms
>         // does not require drift-free scheduling; vTaskDelay() is completely fine here.
>         vTaskDelay(pdMS_TO_TICKS(100));
>     }
> }
> 
> void system_init_and_start(void)
> {
>     stdio_init_all();
> 
>     gpio_init(BTN_PIN);
>     gpio_set_dir(BTN_PIN, GPIO_IN);
>     gpio_pull_up(BTN_PIN);
> 
>     gpio_init(ENC_PIN);
>     gpio_set_dir(ENC_PIN, GPIO_IN);
> 
>     // FIX 8 (Missing rising edge on two-edge state machine): A slot pulse starts on
>     // a falling edge and ends on a rising edge, and encoder_isr toggles between
>     // ENC_IDLE and ENC_IN_SLOT on every edge. Enabling FALL only measures full
>     // periods (falling-to-falling) instead of slot widths and halves the count rate.
>     gpio_set_irq_enabled_with_callback(
>         ENC_PIN, GPIO_IRQ_EDGE_FALL | GPIO_IRQ_EDGE_RISE, true, &encoder_isr);
> 
>     width_queue = xQueueCreate(16, sizeof(uint32_t));
>     xTaskCreate(logger_task,  "logger",  1024, NULL, 3, NULL);
>     xTaskCreate(monitor_task, "monitor", 1024, NULL, 2, NULL);
>     vTaskStartScheduler();
> }
> ```

# Bug Hunt Drill 5 — Bitwise Operators, UART Sync, Scaling & Interrupt Modes

Tags: `#embedded-c` `#rp2040` `#cortex-m0plus` `#uart` `#bughunt` `#quiz-prep`

> [!info] System Specification (Target Context)
> * **Architecture & Build:** RP2040 (32-bit ARMv6-M, little-endian), compiled at `-O3` with `-Wall -Wextra`.
> * **Emergency Stop Button (`STOP_BTN_PIN`):** Wired to **GND** with an internal **pull-up resistor** (`gpio_pull_up`). Pressing and holding the button pulls the pin LOW (`0`) for as long as the user holds it.
> * **UART Binary Frame (`FRAME_LEN = 8`):** `[0xAA][6 bytes payload][1 byte checksum]`. Occasional electrical noise can inject stray bytes onto the line between frames.
> * **Closed-Loop Voltage Control:** Reads a 12-bit ADC value (`0..4095`), converts it to millivolts (`0..3300 mV`), and adjusts PWM `duty` (`0..wrap`) toward `TARGET_MV` (`1650 mV`).

---

## 1. Complete Fixed Code (With Inline Fix Comments)

> [!success] Fixed Implementation (`drill5_fixed.c`)
> ```c
> #include <stdint.h>
> #include <stdbool.h>
> #include <stdio.h>
> #include "pico/stdlib.h"
> #include "hardware/gpio.h"
> #include "hardware/uart.h"
> 
> #define STOP_BTN_PIN  10u
> #define UART_ID       uart1
> #define FRAME_SOF     0xAAu
> #define FRAME_LEN     8u
> #define TARGET_MV     1650u
> 
> static volatile bool estop_triggered = false;
> 
> void estop_isr(uint gpio, uint32_t events)
> {
>     (void)gpio;
>     (void)events;
>     estop_triggered = true;
> }
> 
> /* 1. Check whether fault bit 'bit' (0..31) is 0 */
> bool fault_is_clear(uint32_t status, unsigned bit)
> {
>     // FIX 1 (Bitwise operator precedence): In C, `==` binds tighter than `&`.
>     // Without parentheses, `status & 1u << bit == 0` parses as
>     // `status & ((1u << bit) == 0)`, which evaluates to `status & 0` (always false).
>     return ((status & (1u << bit)) == 0);
> }
> 
> /* 2. Count how many fault bits are set in a 32-bit status word */
> uint8_t active_fault_count(int status_word)
> {
>     // FIX 2 (Signed right-shift infinite loop): If bit 31 is set, `status_word`
>     // is a negative signed `int`. Right-shifting a negative signed integer performs
>     // an arithmetic shift that copies the `1` sign bit forever (`0xFFFFFFFF`).
>     // Cast to unsigned `uint32_t` first so `>>= 1` is a logical shift (fills with 0).
>     uint32_t v = (uint32_t)status_word;
>     uint8_t count = 0;
>     while (v) {
>         count += (uint8_t)(v & 1u);
>         v >>= 1;
>     }
>     return count;
> }
> 
> /* 3. Reverse all 32 bits of 'flags' (bit 0 <-> bit 31, etc.) */
> uint32_t mirror_flags(uint32_t flags)
> {
>     uint32_t out = 0;
> 
>     // FIX 3 (Out-of-range shift Undefined Behaviour): `i <= 32` allows `i == 32`,
>     // causing `flags >> 32` (shift >= type width) and `<< (31 - 32)` (`<< -1`),
>     // both of which invoke Undefined Behaviour. Loop strictly over 0..31 (`i < 32`).
>     for (int i = 0; i < 32; i++) {
>         out |= ((flags >> i) & 1u) << (31 - i);
>     }
>     return out;
> }
> 
> /* 4. Map 12-bit ADC reading (0..4095) linearly onto 0..wrap */
> uint16_t calc_pwm_duty(uint16_t adc_raw, uint16_t wrap)
> {
>     // FIX 4 (Divide-before-multiply truncation): `(adc_raw / 4095u)` is integer
>     // division and evaluates to 0 for every reading below 4095. Multiply first
>     // in 32-bit unsigned arithmetic to avoid 16-bit overflow, then divide last.
>     return (uint16_t)(((uint32_t)adc_raw * wrap) / 4095u);
> }
> 
> /* 5. Nudge PWM duty toward TARGET_MV and log status */
> void control_step(uint16_t mv, uint16_t wrap)
> {
>     static uint16_t duty = 0;
> 
>     // FIX 5 (Unsigned subtraction underflow / positive feedback runaway):
>     // `TARGET_MV` is `1650u` (unsigned). When `mv > 1650`, `TARGET_MV - mv` wraps
>     // modulo $2^{32}$ to a huge positive number, and `delta < 0` can never be true
>     // for an unsigned `uint32_t`—driving `duty` to 100% permanently. Use `int32_t`.
>     int32_t delta = (int32_t)TARGET_MV - (int32_t)mv;
> 
>     if (delta < 0) {
>         if (duty > 0) duty--;
>     } else if (delta > 0) {
>         if (duty < wrap) duty++;
>     }
> 
>     float volts = mv / 1000.0f;
> 
>     // FIX 6 (printf format specifier mismatch): `volts` promotes to a 64-bit
>     // `double` in variadic arguments, whereas `%d` reads a 32-bit `int`, printing
>     // garbage for `volts` and corrupting `duty` and `wrap` printed after it.
>     printf("ADC=%u mV (%.2f V), duty=%u/%u\n", mv, (double)volts, duty, wrap);
> }
> 
> /* 6. Poll UART and process 8-byte binary frames */
> void uart_rx_poll(void)
> {
>     static uint8_t  buf[FRAME_LEN];
>     static uint8_t  n = 0;
>     static uint32_t last_byte_us = 0;
> 
>     uint32_t now = time_us_32();
>     if (n > 0 && (uint32_t)(now - last_byte_us) > 50000u) {
>         n = 0;
>     }
> 
>     while (uart_is_readable(UART_ID)) {
>         last_byte_us = time_us_32();
>         buf[n++] = (uint8_t)uart_getc(UART_ID);
> 
>         // FIX 7 (Receiver never hunts for Start-of-Frame): Blindly counting 8 bytes
>         // means a single stray noise byte shifts the frame window permanently.
>         // Discard any leading byte that does not match FRAME_SOF (0xAA).
>         if (n == 1 && buf[0] != FRAME_SOF) {
>             n = 0;
>         }
> 
>         if (n == FRAME_LEN) {
>             process_frame(buf, FRAME_LEN);
>             n = 0;
>         }
>     }
> }
> 
> void system_init(void)
> {
>     stdio_init_all();
> 
>     gpio_init(STOP_BTN_PIN);
>     gpio_set_dir(STOP_BTN_PIN, GPIO_IN);
>     gpio_pull_up(STOP_BTN_PIN);
> 
>     // FIX 8 (Level-triggered interrupt hang): Holding the button keeps the pin
>     // LOW (`0`). With `GPIO_IRQ_LEVEL_LOW`, the hardware re-fires `estop_isr`
>     // continuously as soon as it returns, starving the main loop. Trigger on
>     // the falling edge (`GPIO_IRQ_EDGE_FALL`) instead.
>     gpio_set_irq_enabled_with_callback(STOP_BTN_PIN, GPIO_IRQ_EDGE_FALL, true, &estop_isr);
> }
> ```

# Bug Hunt Drill 6 — Memory Off-by-Ones, Shared Rings, ISR Latency & PID Loops

Tags: `#embedded-c` `#rp2040` `#freertos` `#pid` `#bughunt` `#quiz-prep`

System Specification (Target Context)

- **Target & Kernel:** RP2040 (32-bit ARMv6-M), FreeRTOS with `configTICK_RATE_HZ = 1000` (`1 tick = 1 ms`).
- **CRC-8 Table:** Must hold 256 entries covering every possible byte value (`0x00..0xFF`).
- **Moving Average Window:** `RING_LEN = 8` samples. During startup (before 8 samples arrive), it must average the actual number of samples collected so far (`ring_fill`).
- **Closed-Loop PID Task (`pid_task`):** Runs every **100 ms** (`dt = 0.1` s), computing 𝑢 =𝐾𝑝𝑒 +𝐾𝑖∫𝑒 𝑑𝑡 +𝐾𝑑𝑑𝑒𝑑𝑡 to drive `current_pos` from `0.0` toward `setpoint = 100.0`.

## 1. Complete Fixed Code (With `MISSED:` Indicators & Inline Comments)

Fixed Implementation (`drill6_fixed.c`)

```c
#include <stdint.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "FreeRTOS.h"
#include "task.h"

#define PULSE_PIN  16u
#define RING_LEN   8u

// FIX 1 [CAUGHT] (Global array 1 entry too small): crc_init() writes indices
// 0..255 (256 total entries). Size 255 causes an out-of-bounds write at i == 255.
static uint8_t crc_table[256];
static volatile uint32_t isr_pulses = 0;

// FIX 2 [CAUGHT] (Shared global ring buffer across tasks): Instead of sharing
// unprotected global ring[] state between sensor_task and pid_task, keep the
// ring buffer local to sensor_task (its sole owner) so no data race occurs.
static int32_t ring[RING_LEN]; 
static uint8_t ring_head = 0; 
static uint8_t ring_fill = 0;
// ^^^^ wrong

/* 1. Build CRC-8 lookup table */
void crc_init(void)
{
    for (int i = 0; i < 256; i++) {
        uint8_t c = (uint8_t)i;
        for (int b = 0; b < 8; b++) {
            c = (c & 0x80u) ? (uint8_t)((c << 1) ^ 0x07u) : (uint8_t)(c << 1);
        }
        crc_table[i] = c;
    }
}

/* 2. Allocate and copy a status tag string on the heap */
char *create_tag(const char *label)
{
    // FIX 3 [CAUGHT] (Heap string off-by-one): strlen(label) excludes the null
    // terminator '\0'. Allocate strlen(label) + 1 bytes so strcpy() does not
    // write 1 byte past the end of the heap block.
    char *copy = malloc(strlen(label) + 1);
    if (copy != NULL) {
        strcpy(copy, label);
    }
    return copy;
}

/* 3. Local moving-average helper for sensor_task */
static float compute_window_avg(const int32_t *ring, uint8_t ring_fill)
{
    if (ring_fill == 0) return 0.0f;
    int32_t sum = 0;
    for (uint8_t i = 0; i < ring_fill; i++) {
        sum += ring[i];
    }

    // FIX 4 — MISSED (Dividing by constant RING_LEN instead of ring_fill):
    // During warm-up (when fewer than 8 samples have arrived), dividing by
    // RING_LEN (8) averages unpopulated zero slots and under-reports the value.
    // Divide by the actual sample count `ring_fill`.
    return (float)sum / (float)ring_fill;
}

/* 4. GPIO Interrupt Service Routine */
void pulse_isr(uint gpio, uint32_t events)
{
    (void)gpio;
    (void)events;
    isr_pulses++;

    // FIX 5 — MISSED (printf inside an ISR): Formatted UART printf() takes
    // milliseconds (hundreds of thousands of CPU cycles), blocking other
    // interrupts, risking re-entrancy collisions with task printf() calls, and
    // causing timing Heisenbugs. Never call printf() inside an ISR.
}

/* 5. Discrete PID calculation (dt in seconds) */
float compute_pid(float setpoint, float current_pos,
                  float *integral, float *prev_error, float dt)
{
    float Kp = 2.0f, Ki = 0.2f, Kd = 0.02f;
    float error = setpoint - current_pos;

    *integral += error * dt;
    float derivative = (error - *prev_error) / dt;

    // FIX 6 [CAUGHT] (Wrong derivative state tracked): Storing current_pos
    // instead of error makes (error - *prev_error) subtract position from error.
    *prev_error = error;

    return (Kp * error) + (Ki * (*integral)) + (Kd * derivative);
}

/* 6. Sensor acquisition task */
void sensor_task(void *pvParameters)
{
    (void)pvParameters;
    TickType_t next = xTaskGetTickCount();

    // Local ring buffer owned exclusively by sensor_task
    int32_t ring[RING_LEN] = {0};
    uint8_t ring_head = 0;
    uint8_t ring_fill = 0;

    for (;;) {
        int32_t raw = read_sensor_data();

        ring[ring_head] = raw;
        ring_head = (uint8_t)((ring_head + 1u) % RING_LEN);
        if (ring_fill < RING_LEN) {
            ring_fill++;
        }

        float avg = compute_window_avg(ring, ring_fill);
        (void)avg;

        vTaskDelayUntil(&next, pdMS_TO_TICKS(50));
    }
}

/* 7. Closed-loop PID motor control task */
void pid_task(void *pvParameters)
{
    (void)pvParameters;
    float setpoint    = 100.0f;
    float current_pos = 0.0f;
    float integral    = 0.0f;
    float prev_error  = 0.0f;
    float dt          = 0.1f;

    TickType_t next = xTaskGetTickCount();

    for (;;) {
        float u = compute_pid(setpoint, current_pos, &integral, &prev_error, dt);
        float motor_step = u * 0.05f;

        // FIX 7 — MISSED (Open-loop plant / missing feedback update):
        // Without adding `motor_step` to `current_pos`, the simulated motor never
        // moves (`current_pos` stays 0.0f forever) and the PID loop never converges.
        current_pos += motor_step;

        printf("u=%.2f pos=%.2f\n", (double)u, (double)current_pos);

        // FIX 8 — MISSED (Wrong unit & float truncation in usleep(dt)):
        // `dt` is 0.1f seconds, while usleep() takes integer microseconds—truncating
        // 0.1f to 0 us and spinning continuously without yielding the CPU.
        // In a FreeRTOS task, use vTaskDelayUntil() with pdMS_TO_TICKS(100).
        vTaskDelayUntil(&next, pdMS_TO_TICKS(100));
    }
}

void app_main(void)
{
    stdio_init_all();
    crc_init();

    gpio_init(PULSE_PIN);
    gpio_set_dir(PULSE_PIN, GPIO_IN);

    // NOTE (Valid C - Decoy): Single-edge `GPIO_IRQ_EDGE_RISE` is valid when
    // merely counting pulses (`isr_pulses++`). Both edges (`FALL | RISE`) are only
    // needed when measuring pulse width between a start edge and an end edge.
    // Likewise, disabling the IRQ is only needed if a task stops at a target count
    // and needs a frozen snapshot of shared variables.
    gpio_set_irq_enabled_with_callback(PULSE_PIN, GPIO_IRQ_EDGE_RISE, true, &pulse_isr);

    // NOTE (Valid C - Decoy): Giving `sensor_task` 1024 words (4 KB) is harmless
    // headroom. Too *little* stack (e.g., 256 words with float printf) is the bug.
    xTaskCreate(sensor_task, "sensor", 1024, NULL, 3, NULL);
    xTaskCreate(pid_task,    "pid",    1024, NULL, 2, NULL);
    vTaskStartScheduler();
}
```
# Embedded C Bug Hunt — Simple Cheat Sheet

Tags: `#embedded-c` `#rp2040` `#freertos` `#bughunt` `#quiz-prep`

## 1. ADC & Byte Packing / Unpacking

- **ADC Select Before Read:**
    
    - Always call `adc_select_input(ch)` **before** `adc_read()`.
        
    - Why: The RP2040 has one ADC behind a switch. Reading first reads the _old_ channel.
        
- **Multiply First, Divide Last (Prevent Truncation & Overflow):**
    
    - Do: `return (uint16_t)(((uint32_t)raw * 3300u) / 4095u);`
        
    - Why: Dividing first (`raw / 4095`) rounds down to `0`. Storing `raw * 3300` in a `uint16_t` overflows `65,535`. Keep the middle step in `uint32_t`.
        
- **Packing vs. Unpacking Bytes (Endianness):**
    
    - **Packing** (splitting a `uint32_t` into wire bytes, MSB first) uses **right shifts (`>>`)**:
        
        - `val >> 24`, `val >> 16`, `val >> 8`, `val`
            
    - **Unpacking** (combining wire bytes back into a `uint32_t`) uses **left shifts (`<<`)** with `(uint32_t)` casts before shifting:
        
        - `((uint32_t)b[0] << 24) | ((uint32_t)b[1] << 16) | ((uint32_t)b[2] << 8) | (uint32_t)b[3]`
            
    - **Never** cast a raw pointer like `*(uint32_t *)&buf[2]`: it causes an unaligned **HardFault** on Cortex-M0+ and flips little-endian/big-endian order.
        
- **No Dangling Pointers:**
    
    - Never return a normal local array (`char msg[32]; return msg;`)—it disappears when the function ends.
        
    - Fix: Pass a buffer in from the caller, or use `static char msg[32];` (only safe if one task calls it).
        

## 2. PWM, PID & FreeRTOS Tasks

- **When to Subtract `- 1` on Ticks:**
    
    - **Only for PWM `wrap`:** `wrap = total_ticks - 1` (because the PWM hardware counts `0..wrap` _inclusive_, which is `wrap + 1` ticks).
        
    - **Not for FreeRTOS delays:** Use `pdMS_TO_TICKS(100)` as-is.
        
- **Signed vs. Unsigned Subtraction:**
    
    - Use **signed (`int32_t`)** when subtracting values or errors that can go negative (filter `x - y`, controller `target - mv`). Unsigned math wraps around to a huge positive number (`4.29e9`).
        
    - Keep **unsigned (`uint32_t`)** when subtracting **timestamps** (`(uint32_t)(now - last_us)`). Unsigned subtraction handles timer rollover safely.
        
- **PID Controller Checks:**
    
    - **Error sign:** Always `setpoint - measured` (reversing it causes runaway feedback).
        
    - **Time step (`dt`):** Multiply integral by `* dt` and divide derivative by `/ dt`.
        
- **1-to-1 Message Buffers:**
    
    - A FreeRTOS message buffer allows **1 sender and 1 receiver only**. If 2 tasks need the data, create 2 separate buffers and send to both.
        
- **Check `sizeof(...)` Carefully:**
    
    - Use `sizeof(sample)` (2 bytes), **never** `sizeof(&sample)` (4 bytes, the size of a pointer).
        
    - Never use `sizeof(struct)` for wire packet lengths—structs contain hidden compiler padding bytes.
        
- **FreeRTOS Task Stack Sizes (Measured in Words = 4 Bytes):**
    
    - `256` words (`configMINIMAL_STACK_SIZE` = 1 KB): Too small if the task calls `printf` with floats (`%f`).
        
    - `512` words (2 KB): Good for normal tasks without float `printf`.
        
    - `1024` words (4 KB): Safe size for tasks calling `printf` with floats (`%f`).
        

## 3. ISRs, GPIO & Timing

- **Pull-Up vs. Pull-Down Button Logic:**
    
    - **Pull-up (`gpio_pull_up` + GND):** Unpressed = `1`, Pressed = `0`.
        
    - **Pull-down (`gpio_pull_down` + 3.3V):** Unpressed = `0`, Pressed = `1`.
        
- **Interrupt Edge Modes:**
    
    - To measure a pulse width (start + end), enable **both** `GPIO_IRQ_EDGE_FALL | GPIO_IRQ_EDGE_RISE`. if the code has 2 state toggle like IDLE to ACTIVE
    - use single edge, so either edge fall or edge rise if ISR simply counts events or sets flags
        
    - Avoid `GPIO_IRQ_LEVEL_LOW` on buttons—it re-fires continuously while the button is held down and freezes the CPU., use edge fall for pull up, edge rise for pull down
        
- **How to Differentiate Debounce Windows:**
    
    - **Mechanical button:** `20,000–50,000 µs` (20–50 ms) to ignore physical switch bounce.
        
    - **Optical wheel encoder:** `~500 µs` (0.5 ms). A 50 ms window is too long and will skip real wheel pulses.
        
- **Wrap-Safe Timer Checks:**
    
    - Never add to a timestamp: `now > last_us + DEBOUNCE_US` breaks when `time_us_32()` rolls over at ~71.6 minutes.
        
    - Always subtract: `(uint32_t)(now - last_us) > DEBOUNCE_US`.
        
- **Keep ISRs Fast & Non-Blocking:**
    
    - Never call `printf()` or blocking queue functions (`xQueueSend(..., portMAX_DELAY)`) inside an ISR. Use `xQueueSendFromISR`.
        
- **Disable IRQs & Prevent Torn Reads:**
    
    - Disable the IRQ once you finish counting target edges so it stops changing data in the background.
        
    - Reading a `uint64_t` on a 32-bit CPU takes two instructions. Wrap the read in `save_and_disable_interrupts()` / `restore_interrupts()` so an ISR cannot fire halfway through and split the value (**torn read**).
    - dont disable a level triggered isr
    - dont disable if the system is meant to be running continuously forever
        

## 4. General C & Loop Traps

- **`volatile` for ISR-Shared Variables:**
    
    - Any variable updated in an ISR and read in `main()` or a task **must** be marked `volatile`, or `-O3` will cache it in a register and loop forever.
        
    - _(Note: `volatile` stops register caching, but does **not** make 64-bit reads atomic.)_
        
- **No Shared `static` Variables Across Tasks:**
    
    - A helper function with `static` local variables (or a global array without a mutex) gets corrupted if two tasks call it. Give each task its own local variables.
        
- **Validate Packet Lengths First:**
    
    - Before reading payload bytes using a wire length `len = buf[1]`, check that `len` matches the expected size and `(uint16_t)len + 3u <= buf_len`.
        
- **Match Loop Counter Width to the Limit:**
    
    - `for (uint8_t i = 0; i < count; i++)` loops forever if `uint16_t count >= 256`, because `uint8_t i` wraps from `255` back to `0`. Use `uint16_t i`.
        
- **Prevent Drift in Periodic Tasks:**
    
    - Use `vTaskDelayUntil(&next, pdMS_TO_TICKS(ms))` instead of `vTaskDelay()` for periodic sensor sampling so code execution time does not add drift.
        
- **Check `switch` Cases for `break;`:**
    
    - Make sure every `case` ends with `break;` so execution does not fall straight into the next `case`.