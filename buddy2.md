# Tribunal 2 Impact Assessment — Fault Handling & Invalid-Data Resilience

**Author:** Buddy 2 — Lee Ruo Chen (`2500530`), Owner of Subsystem 2 (Motion Control & Vehicle Controller)  
**Target Platform:** Raspberry Pi Pico W (`RP2040`, dual Cortex-M0+), micro T-Kernel 3.0 RTOS (`KERNEL_TICK_MS = 1`), C11 (`-Wall -Wextra -Werror`, Barr-C standard), Pico SDK 1.5.1  
**Baseline Reviewed:** Active `main` firmware across Subsystems 1–5 (`app_program/subsystem1..5/`), shared IPC/platform (`config/*.h`, `src/common/*`, `src/rtos/port/*`, `app_program/app_main.c`, `app_program/app.h`), and Tribunal 2 baseline (`docs/trib2/requirements.md`, `docs/trib2/software_block_diagram.svg`)  
**Assessment Scope:** Full-system, read-only design review (no firmware/config modifications)  

---

## 1. Requirement Change Under Review

> **New Requirement:** If any part of the system fails or provides invalid data, the system must respond in a safe and predictable way.  
> **Example:** If a system depends on a measurement and that measurement becomes unavailable or invalid, the system should not simply continue as though the value were correct. It should have a defined response, such as continuing with limited operation, stopping the affected function, attempting recovery, or requiring intervention.

---

## 2. Executive Summary & System-Wide Fault-Response Policy

The current car already has strong core fault mechanisms on the highest-priority safety paths (`IF-11` interrupt-safe emergency brake meeting NFR10 at `134 ms < 200 ms`; `P1` Safety 50 ms IMU watchdog; Code 39 strict 29-element decoding; non-blocking telemetry isolation under NFR1). However, systematic inspection across all five subsystems and shared boot/platform code reveals **eight direct (`[OBVIOUS CHANGE]`) gaps** where failed or invalid measurements cause the system to continue as if the data were valid—most critically:

1. **Fabricated encoder speed/distance & ignored stall/encoder faults** in Motion Control (`#1`),
2. **Unconnected subsystem self-test gates and fail-open RTOS error paths** in the Vehicle Controller (`#2`),
3. **Unwired `EVF_SAFETY_FAULT_IR_SENSOR` and silent boot/runtime rail faults** in Line Following (`#3`),
4. **Disconnected/faulty ultrasonic sensor (`no_echo`) treated as clear open space** in Obstacle Scanning & Avoidance (`#4`),
5. **Unvalidated IMU pitch boost, 10 s silent level-reference fallback, and unflagged zero-vector readings** in IMU Processing & Motion (`#5`),
6. **Verified `S1-M03` MQTT reconnect heap exhaustion, link-state flapping, and silent subscribe failure** in Telemetry (`#6`),
7. **Forever-latched IF-12 sensor snapshot valid bits & unchecked `motion_state_t.tick_ms` staleness** in Telemetry (`#7`), and
8. **Fire-and-forget `APP_CHECK()` startup error handling** across `usermain()` (`#8`).

To ensure every failure has a deterministic, non-contradictory outcome across all tasks, responses are standardised into four classes:

| Response Class | Triggering Condition | Mandatory System Action | Recovery / Resumption Rule |
|---|---|---|---|
| **Class A — Emergency Brake & Latch** | Safety/control integrity lost: `LIFTED`, `IMU_WDOG`, IMU zero-vector, `STALL` / `ENCODER` loss, `IR_SENSOR` rail fault, Tier-1 boot/IPC init failure, CPU HardFault/WDT | Immediate `motor_brake_all()`, set `EVF_SAFETY_FAULT_LATCHED` + source bit, transition VC to `VC_SYS_SAFETY_FAULT_LATCHED`, emit `DREP_TEL_FAULT` | Requires explicit `REMOTE_RESET` after `SAFETY_T_STILL_MS` (`500 ms`) at verified zero speed **and** all latching source bits clear; returns to `READY_WAITING` (never auto-drives) |
| **Class B — Stop Affected Function / Refuse Start** | Non-propulsion or pre-start check failed: `subsystem5_sensors_ok() == false`, IR calibration refusal, `OBS_AV_SLOW` mid-scan dropout after confirmed hit | Refuse `REMOTE_START` (hold `INITIALISING`/`FAULT_LATCHED`) or halt avoidance (`OBS_AV_BLOCKED` / `DREP_BYPASS_STOP`) without blind cruising | Clears when hardware health returns and function is explicitly re-started |
| **Class C — Degraded / Limited Operation** | Non-safety-critical function unavailable: Wi-Fi/MQTT outage (`WIFI_LOSS`), transient `1..4` IMU I2C read errors before 50 ms watchdog, stale IF-12 telemetry snapshots | Telemetry buffers events in 16-slot outbox and keeps autonomous driving active; Motion immediately zeroes `hump_boost_mmps` on IMU fault/uncalibrated state; Telemetry clears `*_valid` bits on stale snapshots | Seamless return to full operation once fresh valid samples arrive |
| **Class D — Automatic Recovery Without Uncommanded Motion** | Failed driver/link restores connectivity (`imu_reconfigure()`, CYW43/lwIP reconnect) | Driver re-initialises peripheral, verifies plausible readings (`~1 g`), clears its source bit (`FAULT_IMU_I2C`, `FAULT_IMU_WDOG`, `FAULT_WIFI_LOSS`), and stops there | Never clears `EVF_SAFETY_FAULT_LATCHED` automatically (`#12`) |

### Assessment Summary Table

| Tag | Item IDs | Count | Decisions Breakdown |
|---|---|---|---|
| `[OBVIOUS CHANGE]` | `#1` – `#8` | 8 | 5 `Accept`, 3 `Modify` |
| `[HIDDEN RISK]` | `#9`, `#11`, `#12`, `#13`, `#15` | 5 | 3 `Accept`, 1 `Modify`, 1 `Reject` |
| `[KNOCK-ON CHANGE]` | `#10`, `#14`, `#16` | 3 | 2 `Accept`, 1 `Modify` |
| `[CHECKED, NO IMPACT]` | `#17` – `#22` | 6 | 6 `No change` |
| **Total** | **`#1` – `#22`** | **22** | **10 `Accept`, 5 `Modify`, 1 `Reject`, 6 `No change`** |

---

## 3. Coverage Across All 11 Required Review Areas

| Review Area | Covered By Item(s) | Key Coverage Summary |
|---|---|---|
| **Requirements** | `#14`, `#15`, `#18`, `#21` | Formal FMEA Fault-Response Matrix in `docs/trib2/requirements.md`; NFR14 bring-up budget (`60 s`) vs nominal (`5.0 s`) and 3° vs 5° ultrasonic sweep alignment |
| **Tasks / components** | `#1`, `#3`, `#4`, `#8`, `#21` | `task_motion.c`, `task_vc.c`, `task_lf.c`, `task_obs_scan.c`, `task_obs_avoid.c`, `task_imu.c`, `usermain()` boot tiers, non-blocking P9 Telemetry |
| **State machines** | `#2`, `#3`, `#4`, `#6`, `#10`, `#12` | VC self-test/reset gating; LF `IDLE` vs `FAULT_IR_SENSOR`; `OBS_AV_SLOW` dropout hold; `tel_link` `DEGRADED -> CONNECTED` flapping fix; `LF_ST_SUSPENDED` abort exit fix |
| **Interfaces / messages** | `#4`, `#7`, `#10`, `#13`, `#14` | `EVF_SAFETY` expansion `b0–b9 -> b0–b11` within zero-growth v1 MQTT frames; IF-12 validity expiry; `motion_state_t.tick_ms` freshness; `task_bc.c` `5 ms` `MBF_NAV_IN` send |
| **Shared data / concurrency / interrupts** | `#5`, `#7`, `#11`, `#13`, `#17`, `#22` | `s_vc.state_bits` RMW race protection; eliminating `EVF_SAFETY` shadow caches; `FAULT_IMU_WDOG` clear on recovery; `NET_RX_SLOTS` `STOP` priority; `MTX_MOTION` `TA_INHERIT` |
| **Timing / performance** | `#1`, `#9`, `#14`, `#15`, `#18`, `#19`, `#22` | `500 ms` low-speed stall detection vs `20 ms` loop; `500 ms` HW WDT vs `50 ms` Safety cycle; `100 ms` hold cap; `134 ms < 200 ms` NFR10; `10–566 µs` `SYS-01` jitter |
| **Memory / resources** | `#6`, `#14` | lwIP `MEM_SIZE` `4096 -> 8192 B` (`+4 KB` out of `> 100 KB` free SRAM); `12 B` snapshot timestamps; measured stack margins across all tasks (`620–4276 B` free) |
| **Safety / fault handling** | `#1`–`#5`, `#7`–`#9`, `#11`–`#13`, `#17`–`#21` | End-to-end fault detection, latching, HardFault braking, fail-closed RTOS checks, and safe degraded behaviours across all 5 subsystems |
| **Hardware / software interaction** | `#1`, `#3`, `#4`, `#5`, `#9`, `#15`, `#17`, `#20` | PIO0 encoder readiness, GP26/GP28 ADC rails, HC-SR04 echo stuck-HIGH divider check, LSM303D zero-vector & 4–4.5° mounting offset, RP2040 PWM autonomy + WDT, invert macros |
| **Recovery and safe resumption** | `#2`, `#6`, `#8`, `#10`, `#11`, `#12` | Clean reconnection (`S1-M03`, `imu_reconfigure()`), `LF_ST_SUSPENDED` recovery, preservation of operator-gated `REMOTE_RESET` after stationary verification |
| **Verification / testing** | `#16` | Fault-injection test matrix (`S1-F01..F02`, `S2-F01..F03`, `S3-F01`, `S4-F01`, `S5-F01`, `INT-F01`) and `S1-M03` re-verification |

---

## 4. Detailed Impact Assessment (Ordered Highest → Lowest Impact; Every `[OBVIOUS CHANGE]` First)

---

## #1 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Motion Control)  
**Affected Requirement / Component:** FR5, FR6, NFR2, NFR6 / `app_program/subsystem2/task_motion.c`, `app_program/subsystem2/drv_encoder.c`  
**Area:** Safety / fault handling; Tasks / components; Timing / performance; Hardware / software interaction  

**What needs to change?:**
Act on `EVF_SAFETY_FAULT_STALL` and `EVF_SAFETY_FAULT_ENCODER` by immediately braking the motors and latching safety; lower/speed-scale stall arming so it actually triggers at active bring-up (`5 cm/s`) and low cruise speeds; remove the production bench fallback that fabricates speed and distance when both encoders read zero; and surface PIO encoder init or FIFO poll failures instead of silently returning `0` or stale counts.

**Decision:** Modify  

**Why?:**
Currently, Motion Control actively violates the new requirement in three compounding ways:
1. When a wheel stalls or an encoder fails, `update_fault_flags()` only sets advisory bits `b3`/`b4` on `EVF_SAFETY` — neither Motion, Safety, nor VC ever stops the motors or latches `EVF_SAFETY_FAULT_LATCHED`.
2. Because `MOTION_STALL_DUTY_PM = 500` (50% duty), the stall detector **never even arms** during steady driving at `5 cm/s`, `10 cm/s`, or `15 cm/s`, leaving a jammed motor or unplugged encoder completely undetected indefinitely.
3. When both encoders read `0` (`dl == 0 && dr == 0`), `task_motion.c` deliberately invents fake speed (`±setpoint_cmps * 10`) and integrates fake distance (`distance_cm`), poisoning Barcode speed pairing (`task_bc.c`), VC stationary reset detection (`task_vc.c`), `BC_D_CMD_CM` (`60 cm`) barcode expiration, and MQTT telemetry. On a single-encoder loss, `mean_counts = (|dl| + |dr|) / 2` halves measured travel (causing `2×` over-distance or `180°` over-turns), while `sync_trim` saturates to `±120 pm` and spins the healthy wheel backward.

**Engineering Evidence:**
- **Detected stall/encoder faults never stop motors (`task_motion.c:450-495`):** `update_fault_flags()` sets `EVF_SAFETY_FAULT_STALL` (`1UL << 3`) or `EVF_SAFETY_FAULT_ENCODER` (`1UL << 4`) after `MOTION_STALL_PERIODS = 25U` (`25 × 20 ms = 500 ms`, `task_motion.c:106`). It never calls `stop_internal(false)`, never sets `EVF_SAFETY_FAULT_LATCHED` (`b2`), and never posts `DREP_TEL_FAULT`. `task_safety.c:84` waits exclusively on `EVF_SAFETY_WAIT_MASK = EVF_SAFETY_LIFTED | EVF_SAFETY_IMU_ALIVE` (`src/common/ipc_types.h:96`), and `task_vc.c:195-204` checks only `EVF_SAFETY_FAULT_LATCHED`.
- **Stall detector cannot arm at 5–15 cm/s (worked calculation):**
  - Constants: `ENC_QUAD_MULT = 4U`, `ENC_CPR = 616U` → `MOTION_COUNTS_PER_REV = 2464` (`task_motion.c:63,67`, `config/thresholds.h:123,126`); `WHEEL_CIRC_MM = 207U` (`config/thresholds.h:129`); `MOTION_PERIOD_MS = 20U` (`config/timing.h:76`); `MOTOR_DUTY_MAX_PCT = 95U` (`config/thresholds.h:230`); `MOTION_SPEED_MAX_CMPS = 50U` (`config/thresholds.h:215`).
  - PID gains (`task_motion.c:84-97, 444-446`): `MOTION_PID_KP_PM = 3`, `MOTION_PID_KI_X100 = 10`, `MOTION_PID_KD_PM = 1`, `MOTION_INTEGRAL_MAX = 400`.
  - Feed-forward duty (`task_motion.c:616-617`): `base_pm = (eff_speed_mmps * 95) / 50`. Target counts (`task_motion.c:671-673`): `mag_counts = (eff_speed_mmps * 2464 * 20) / (207 * 1000)`.
  - At active bring-up `5 cm/s` (`50 mm/s`): `base_pm = 95 pm`, `mag_counts = 2464000 / 207000 = 11` counts/period. If both encoders are dead (`dl = dr = 0`), steady-state error `err = 11`, `d_err = 0`, `integral` clamps at `400`: `duty = 95 + (3 × 11) + ((10 × 400) / 100) = 95 + 33 + 40 = 168 pm` (16.8%, or `248 pm` even with `-DMOTION_INTEGRAL_MAX=1200`, `task_motion.c:99-103`).
  - At `10 cm/s` (`100 mm/s`): `base_pm = 190 pm`, `mag_counts = 23`, `duty = 190 + (3 × 23) + 40 = 299 pm` (29.9%).
  - At `15 cm/s` (`150 mm/s`): `base_pm = 285 pm`, `mag_counts = 35`, `duty = 285 + (3 × 35) + 40 = 430 pm` (43.0%).
  - Every steady-state value `168 / 299 / 430 pm` is strictly below `MOTION_STALL_DUTY_PM = 500` (`task_motion.c:154`), so `driving` (`task_motion.c:453-454`) stays `false` and `FAULT_STALL` / `FAULT_ENCODER` never triggers!
- **Single-encoder failure veers & double-travels (`task_motion.c:388-431, 720-744`):** When `dl = 0` and `dr = 11`, `heading_diff = total_dl - total_dr` decreases by `-11` counts/period and saturates `sync_trim` at `-MOTION_SYNC_MAX_PM = -120 pm` (`task_motion.c:94, 728-732`) within `120 ms` (6 periods), collapsing left/right balance, while `mean_counts = (|dl| + |dr|) / 2` halves measured displacement and forces bounded turns/moves to overrun by `2×` up to `MOTION_MANOEUVRE_MAX_MS = 15000 ms` (`task_motion.c:76`).
- **Bench fallback fabricates speed and distance (`task_motion.c:772-780`):** When `(dl == 0) && (dr == 0)`, Motion writes `*p_speed_x10 = ±(setpoint_cmps * 10)` and integrates `s_mot.distance_mm_rem` at commanded speed instead of `0`. Downstream, `task_bc.c:187` pairs fake speed with static/hand-moved barcode edges, `task_vc.c:164-182` sees `speed_x10 != 0` and blocks stationary `RESET`, and `task_tel.c:518` reports false forward velocity/odometry over MQTT.
- **Silent PIO driver failure (`drv_encoder.c:120-123, 149-189`):** If `pio_can_add_program_at_offset(pio0, &quadrature_encoder_program, 0U)` fails, `enc_init()` returns `void` silently with `s_ready = 0U`; `enc_get_count()` silently returns `0`; and when `ENC_FRESH_POLL_MAX` (`32`, line 44) expires without `got == true`, `enc_get_count()` silently returns stale `s_last_raw[wheel]` with zero error counter.

**Modified Decision:**
1. Restrict the `(dl == 0 && dr == 0)` bench fallback (`task_motion.c:772-780`) strictly to `#if APP_TEST` bench builds; in production, report true `*p_speed_x10 = 0` and `distance += 0` whenever `(dl == 0) && (dr == 0)`.
2. Replace `MOTION_STALL_DUTY_PM = 500` in `update_fault_flags()` with a command-and-target-based gate: arm per wheel whenever `(s_mot.flags & MOT_FLG_CMD_ACTIVE) != 0U`, expected wheel target magnitude `>= 3` counts/period (`>= 1.26 cm/s`), and `|duty_w| >= 80 pm` while `dw == 0` for `MOTION_STALL_PERIODS = 25` (`500 ms`, checking each wheel independently so `STEER` asymmetries cannot hide a dead wheel).
3. On `FAULT_STALL` or `FAULT_ENCODER`, execute `stop_internal(false)`, set `EVF_SAFETY_FAULT_LATCHED | EVF_SAFETY_FAULT_STALL/ENCODER`, and emit `tel_post_fault(...)` so VC and operators see the fault and motors halt safely.
4. Expose `enc_stale_count()` and verify `enc_is_ready()` at VC self-test (`#2`).

**Alternative considered:** Running degraded open-loop control on the surviving encoder when only one encoder fails (`dl == 0` XOR `dr == 0`). Rejected because straight-line heading synchronisation (`sync_trim`) requires both wheels' differential counts, turning arcs become unobservable if the dead wheel drags or slips, and continuing on one blind wheel risks driving off the mat or into an obstacle; stopping and latching on either encoder failure is predictable and safe.

---

## #2 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Vehicle Controller)  
**Affected Requirement / Component:** FR4, FR12 / `app_program/subsystem2/task_vc.c`, `app_program/subsystem2/subsystem2.h`  
**Area:** State machines; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Connect all subsystems' boot health checks (`subsystem2_sensors_ok()`, `subsystem3_sensors_ok()`, `subsystem4_imu_found()`, `subsystem5_sensors_ok()`) into VC's self-test gate (`vc_complete_selftest()`); make `is_fault_latched()` and `update_stationary_tracker()` fail **closed** on RTOS/mutex errors; prevent `REMOTE_START` from bypassing `VC_SYS_INITIALISING`; and reject `REMOTE_RESET` while any latching fault-source bit on `EVF_SAFETY` remains active.

**Decision:** Accept  

**Why?:**
The Vehicle Controller owns `VC_SYS_*` arbitration (`FR7`) and safety fault clearing (`FR12`), yet currently its startup gate checks nothing from Subsystems 2–5, its RTOS API error handling assumes "no fault / zero speed" (fail-open), `REMOTE_START` can force a transition out of `INITIALISING` if self-test never passed, and `REMOTE_RESET` can clear `FAULT_LATCHED` even while hardware is still dead.

**Engineering Evidence:**
- **Disconnected self-test gate (`task_vc.c:565-566`):** `vc_task_entry()` executes `vc_init(); vc_complete_selftest(!is_fault_latched());` immediately at `TASK_PRI_VC` (`2`). It never checks `subsystem2_pre_start_brake_ok()` (`subsystem2.h:115`), `motor_is_ready()`, `enc_is_ready()`, `subsystem3_sensors_ok()` (missing in `subsystem3.h`), `subsystem4_imu_found()` (`subsystem4.h:49`), or `subsystem5_sensors_ok()` (`subsystem5.h:48` — whose own docstring states it exists specifically for `vc_complete_selftest()`!). Worse, because `vc` runs at P2 ahead of P8 `imu` and before Safety's `SAFETY_IMU_WDOG_MS = 50 ms` timer expires, VC transitions to `VC_SYS_READY_WAITING` for ~50 ms even when no IMU is present on I2C1.
- **Fail-open safety flag reads (`task_vc.c:195-204`, `task_motion.c:883-887`):** `is_fault_latched()` returns `false` when `tk_ref_flg(EVF_SAFETY, &rflg) != E_OK`. Similarly `task_motion.c:883-887` only brakes if `(tk_ref_flg(EVF_SAFETY, &rflg) == E_OK) && ((rflg.flgptn & EVF_SAFETY_FAULT_LATCHED) != 0U)`. If `ipc_init()` failed or `EVF_SAFETY` returns an error (`E_ID`/`E_NOEXS`), both VC and Motion behave as though no safety fault exists! Failing **closed** (`return true` in `is_fault_latched()`, `else { stop_internal(false); }` in `task_motion.c`) ensures motors stay braked.
- **Fail-open stationary detector (`task_vc.c:158-184`):** `update_stationary_tracker()` initializes `int32_t speed_x10 = 0;` and leaves it `0` if `motion_get_state(&ms) != E_OK` — causing a failed `MTX_MOTION` read or stalled Motion task to look like zero speed, arming `is_stationary_for_reset()` (`SAFETY_T_STILL_MS = 500 ms`, `config/timing.h:89`) even while the car is rolling.
- **`REMOTE_START` bypasses `VC_SYS_INITIALISING` (`task_vc.c:251-254`):** If self-test held VC in `VC_SYS_INITIALISING` (or VC had not finished initializing), receiving `DREP_NAV_REMOTE_START` forcibly promotes `VC_SYS_INITIALISING -> VC_SYS_READY_WAITING -> VC_SYS_NAV_LINE_TRACKING` and sends `DREP_MC_FWD`.
- **Ungated `REMOTE_RESET` (`task_vc.c:217-225`):** `handle_remote_msg()` clears `EVF_SAFETY_FAULT_LATCHED` when `is_stationary_for_reset(now_ms)` is true without inspecting whether any hardware fault-source bits (`EVF_SAFETY_FAULT_IMU_I2C | EVF_SAFETY_FAULT_IMU_WDOG | EVF_SAFETY_FAULT_STALL | EVF_SAFETY_FAULT_ENCODER | EVF_SAFETY_FAULT_IR_SENSOR | EVF_SAFETY_FAULT_US_SENSOR | EVF_SAFETY_FAULT_BOOT_INIT`) are still set. Against a dead IMU, this opens a `50 ms` window (`S1-M02` log row: `READY_WAITING` for ~0.1 s before Safety re-latches) during which a queued `START` could briefly spin motors.

---

## #3 [OBVIOUS CHANGE]
**Buddy:** Buddy 3 (Jin Yong — Subsystem 3: Line Following)  
**Affected Requirement / Component:** FR7, FR8, NFR7 / `app_program/subsystem3/subsystem3.c`, `task_lf.c`, `lf_core.c`, `lf_straddle.c`, `drv_ir.c`  
**Area:** Safety / fault handling; Tasks / components; State machines; Hardware / software interaction  

**What needs to change?:**
Implement `EVF_SAFETY_FAULT_IR_SENSOR` (`b9`) across Subsystem 3 for boot ADC sanity failures, calibration refusal (`LF_CAL_BAD_SENSOR` / `LF_CAL_NO_CONTRAST`), and sustained runtime rail/over-range faults in **both** `lf_straddle.c` and `lf_core.c`; expose `subsystem3_sensors_ok()` for VC self-test; add validity/status reporting to `drv_ir_read()`; and avoid silently freezing `task_lf.c` encoder progress when `motion_get_state()` fails or goes stale.

**Decision:** Accept  

**Why?:**
`EVF_SAFETY_FAULT_IR_SENSOR` (`b9`) is defined in `ipc_types.h` and shown on `docs/trib2/software_block_diagram.svg`, yet is completely dead code. Today, an unplugged, shorted, or saturated IR sensor is logged and ignored at boot; in edge mode (`lf_core.c`), a rail-HIGH sensor (`>= 4079`) normalizes to `1000 pm` dark and causes false steering or false junctions; in straddle mode (`lf_straddle.c`), a rail fault only raises `LF_EV_TRACK_LOST` (dropping VC to `READY_WAITING`, not `FAULT_LATCHED`), permitting repeated restarts on broken sensors.

**Engineering Evidence:**
- **`EVF_SAFETY_FAULT_IR_SENSOR` never set (`src/common/ipc_types.h:93`, `app_program/subsystem3/subsystem3.c:31-67`):** `ipc_types.h:93` defines `#define EVF_SAFETY_FAULT_IR_SENSOR (1UL << 9)` in `EVF_SAFETY_FAULT_MASK`, and `software_block_diagram.svg:71,168` assigns it to `P4 Line Following`. Yet `subsystem3/` never calls `tk_set_flg(EVF_SAFETY, EVF_SAFETY_FAULT_IR_SENSOR)` anywhere. `subsystem3.c:31-56` explicitly comments: `"A fault is logged, never acted on."` `s3_ir_boot_check()` checks `drv_ir_sanity()` against `ADC_STUCK_LOW_RAW = 16`, `ADC_STUCK_HIGH_RAW = 4079`, `ADC_NOISE_MAX_RAW = 400` (`config/thresholds.h:96-102`), prints the string, discards the verdict, and `subsystem3_init()` returns `E_OK` (`subsystem3.c:67`).
- **Calibration failure does not fault-latch (`task_lf.c:218-236`):** When `lf_cal_from_start_pose()` returns `LF_CAL_BAD_SENSOR` or `LF_CAL_NO_CONTRAST`, `task_lf.c` logs it and sets `cal_ok = false`, leaving VC in `READY_WAITING` until `START` triggers one `LF_EV_TRACK_LOST` (`lf_straddle.c:613-616`, `lf_core.c:1024-1027`). It never sets `EVF_SAFETY_FAULT_IR_SENSOR` or emits `DREP_TEL_FAULT`.
- **Edge core lacks runtime rail/saturation detection (`app_program/subsystem3/lf_core.c`):** Because `S3_LF_STRADDLE` defaults to `OFF` (`app_program/subsystem3/sources.cmake:103`), default builds compile `lf_core.c`, which has **no** check for `ADC_STUCK_LOW_RAW`/`ADC_STUCK_HIGH_RAW` or `LF_SD_OVER_RAW` (`3000U`, `lf_straddle.h:224`). A rail-HIGH failure (`>= 4079`) normalizes to `1000 pm` (`LF_ZONE_DARK`) — causing false `JUNCTION` detections (consistent with `testing/subsystem3/TEST_LOG.md` row 66: *"raw far above black counts as DARK, so a lifted nose or a hump can fake a junction"*).
- **Straddle rail fault stops as `TRACK_LOST` instead of sensor fault (`lf_straddle.c:516-519, 592-599, 642-647`):** When `sd_rail()` holds for `LF_SD_FAULT_N = 40U` (`40 × 5 ms = 200 ms`, `lf_straddle.h:253`), `lf_straddle.c` emits `LF_EV_TRACK_LOST` and transitions to `LF_ST_IDLE` without setting `EVF_SAFETY_FAULT_IR_SENSOR` or calling `tel_post_fault()`. VC (`task_vc.c:365-370`) transitions to `VC_SYS_READY_WAITING`, letting an operator immediately click `START` again with dead IR sensors.
- **`drv_ir_read()` returns `0U` on invalid state (`drv_ir.c:36-47`):** Calling `drv_ir_read()` before `drv_ir_init()` (`!s_ready`) or with an invalid channel returns `0U` — indistinguishable from a real `ADC_STUCK_LOW_RAW` conversion.
- **Silent stale `s_ms` copy (`task_lf.c:255-258`):** When `motion_get_state(&s_ms) != E_OK`, `task_lf.c` keeps the stale `s_ms`, freezing distance-based `LF_SD_CONF_CNT`, `LF_SD_BOTHDARK_CNT`, and `LF_D_JCT_POST_CNT` transitions instead of falling back to bounded time-based safety or fault flagging.

---

## #4 [OBVIOUS CHANGE]
**Buddy:** Buddy 5 (Chay Han — Subsystem 5: Obstacle Scanning & Avoidance)  
**Affected Requirement / Component:** FR13, FR14, NFR14, NFR15 / `app_program/subsystem5/subsystem5.c`, `task_obs_scan.c`, `task_obs_avoid.c`, `drv_ultrasonic.c`, `drv_servo.c`  
**Area:** Safety / fault handling; Tasks / components; Hardware / software interaction; Interfaces / messages  

**What needs to change?:**
Separate deterministic ultrasonic/servo hardware failures (echo stuck HIGH before trigger, `us_init()` / `servo_init()` failure, `!subsystem5_sensors_ok()`) from normal open-space echo timeouts (`STATUS_NO_ECHO`); prevent `OBS_AV_SLOW` from resuming straight into a confirmed obstacle when `obs_analyse_fine3()` returns `no_echo`; and fail safely (`OBS_AV_BLOCKED` + stop) rather than timing out into open-loop motion when `motion_get_state()` fails or goes stale.

**Decision:** Modify  

**Why?:**
Currently, an unplugged or miswired HC-SR04 returns `0 mm` ("nothing in range") on every ping, which `task_obs_scan.c` treats identically to an open clear path — allowing the car to drive full speed straight into obstacles. Even worse, if a confirmed obstacle stops the car into `OBS_AV_SLOW` and the subsequent sweep drops echoes, `task_obs_avoid.c` releases straight back to cruise directly toward that obstacle. Finally, `motion_get_state()` failures during `confirm_step()`, legs, and turns silently fall back to open-loop timeouts (`600 ms`, `4000 ms`, `10000 ms`).

**Engineering Evidence:**
- **Boot sensor failure ignored (`subsystem5.c:47-91`):** `subsystem5_init()` verifies servo PWM frequency (`50 ± 2 Hz`), `servo_chan_a_is_low()`, `us_init()`, and `!us_echo_pin_is_high()` (`drv_ultrasonic.c:127-130` explains pulls are disabled precisely so an unplugged/unpowered/no-divider sensor reads stuck HIGH). On failure, `s_sensors_ok = false`, logs `[S5] sensor check FAILED`, returns `E_OK` (`subsystem5.c:90`), and nothing checks `subsystem5_sensors_ok()`.
- **Runtime `no_echo` indistinguishable from clear path (`task_obs_scan.c:946-966, 988`, `ipc_types.h:97,132`):** `record_ping_health()` sets advisory `EVF_SAFETY_STATUS_NO_ECHO` (`b7`) after `OBS_COARSE_BEARING_COUNT = 5` zero pings (`5 × 60 ms = 300 ms`), explicitly excluded from `EVF_SAFETY_FAULT_MASK`. Meanwhile `watch_step()` ignores `mm == 0U`, so a dead sensor keeps `SCAN_WATCH` permanently clear. Note also `drv_ultrasonic.c:89`: echoes longer than `US_ECHO_MAX_US` also overwrite `s_echo_us = 0U`.
- **Mid-scan `no_echo` drives straight into confirmed obstacle (`task_obs_scan.c:1096-1108`, `task_obs_avoid.c:930-937`):** After `watch_step()` confirms an obstacle inside `250 mm` and stops the car, if `finish_fine3_sweep()` fails (`obs_analyse_fine3() == false`), it posts `obs_result_to_profile(NULL, s_seq)` (`distance_cm = 0`, `drep_obs_no_echo(prof) == true`). In `task_obs_avoid.c:930-936`, while in `OBS_AV_SLOW`, seeing `drep_obs_no_echo(prof)` immediately executes `release(now_ms, false, true)` — restoring `VC_SYS_NAV_LINE_TRACKING` (`DREP_MC_FWD`) toward the obstacle that halted the car!
- **Silent `motion_get_state()` failure fallbacks (`task_obs_scan.c:1022-1055`, `task_obs_avoid.c:392-422, 508-519`):**
  - `confirm_step()` (`task_obs_scan.c:1027-1053`): if `motion_get_state(&ms) != E_OK`, `is_still` remains `false`, until `OBS_CONFIRM_MAX_WAIT_MS = 600 ms` (`line 84`) forces standstill confirmation even if the car is still rolling.
  - `snapshot_encoders()` (`task_obs_avoid.c:392-406`): sets `enc[0] = enc[1] = 0` on mutex failure; `leg_fwd_counts()` returns `0U`; `poll_turn()` returns `false`, degrading closed-loop legs/turns into multi-second blind timeouts (`OBS_TURN_MAX_MS`, `OBS_LEG_BACK_MAX_MS`).

**Modified Decision:**
1. Check `us_echo_pin_is_high()` prior to every trigger cycle in `drv_ultrasonic.c` / `task_obs_scan.c` (plus `!us_is_ready()` / `!servo_is_ready()` / `servo_get_pwm_freq_hz()` drift). Whenever echo is stuck HIGH before trigger or drivers failed boot self-test (`!subsystem5_sensors_ok()`), raise `EVF_SAFETY_FAULT_US_SENSOR` (`b10`, see `#14`), post `DREP_TEL_FAULT`, refuse `REMOTE_START`, and command `OBS_AV_BLOCKED` (`abort`).
2. In `OBS_AV_SLOW`, if `finish_fine3_sweep()` cannot form a valid profile (`no_echo`), require an explicit 3-ping ahead verification (`OBS_CONFIRM_SAMPLES = 3`) showing a valid echo beyond `OBS_DETECT_MM + OBS_DETECT_HYST_MM` (`280 mm`) — or at least 2 distinct valid open-range echoes before resuming; if zero echoes persist right after a confirmed close obstacle, transition to `OBS_AV_BLOCKED` and halt cleanly rather than charging blindly.
3. Treat `motion_get_state()` error or stale `ms.tick_ms` (`> MOTION_STALE_MS = 100 ms`) during `confirm_step()`, legs, or turns as a motion-feedback fault: pause/abort to `OBS_AV_BLOCKED` instead of expiring open-loop timeouts.

**Alternative considered:** Setting `EVF_SAFETY_FAULT_LATCHED` whenever `EVF_SAFETY_STATUS_NO_ECHO` asserts (5 consecutive `0 mm` readings = `300 ms`). Rejected because an HC-SR04 aimed down an open track with no obstacle inside `OBS_ECHO_TIMEOUT_MS = 30 ms` legitimately times out with `0 mm` (as documented in `config/timing.h:138`), which would falsely halt the robot on every open straightaway.

---

## #5 [OBVIOUS CHANGE]
**Buddy:** Buddy 4 (Vincent — Subsystem 4: IMU Processing); Secondary: Buddy 2 (Motion Control)  
**Affected Requirement / Component:** FR10, FR11, FR12, NFR12, NFR13 / `app_program/subsystem4/task_imu.c`, `imu_lift.c`, `drv_imu.c`, `app_program/subsystem2/task_motion.c`  
**Area:** Safety / fault handling; Hardware / software interaction; Shared data / concurrency / interrupts  

**What needs to change?:**
Gate Motion Control's IMU hump climb/speed boost on valid, calibrated, non-faulted IMU state (`subsystem4_imu_found()`, `imu_is_level_done()`, clear `EVF_SAFETY_FAULT_IMU_I2C`); replace `update_level()`'s 10 s silent fallback to `0` with an explicit calibration-incomplete/fault condition rather than injecting up to `4.5°` mounting bias; and add physical plausibility / zero-vector sanity checks (`|a|` within plausible bounds, not `(0,0,0)` or rail-saturated) before signalling `EVF_SAFETY_IMU_ALIVE` or arming `imu_lift_set_up()`.

**Decision:** Accept  

**Why?:**
Motion blindly reads `subsystem4_get_pitch_half_deg()` to nearly double bring-up speed (`50 -> 95 mm/s`) and inject `+18%` climb duty without checking whether the IMU exists, has calibrated its boot level offset, or is currently failing I2C reads. Simultaneously, `task_imu.c` silently pretends mounting tilt is `0°` after 10 s on a tilted/moving car (even though the real chassis has a `4.0–4.5°` static bias that exceeds the `3.0°` hump threshold!) and treats an all-zero accelerometer frame `(0,0,0)` as a healthy `IMU_ALIVE` watchdog kick.

**Engineering Evidence:**
- **Unvalidated pitch drives `+90%` speed boost & `+18%` duty in Motion (`task_motion.c:532-608`):** Motion reads `abs_pitch_hd = abs_i32(subsystem4_get_pitch_half_deg())` (`line 532`) and ramps up to `MOTION_HUMP_BOOST_MAX_MMPS = 45U` (`50 -> 95 mm/s`, `lines 167, 595`) plus `MOTION_HUMP_CLIMB_MAX_PM = 180` (`+18%` duty, `lines 171, 606-607`) whenever `abs_pitch_hd >= MOTION_HUMP_PITCH_ON_HD` (`6` = `3.0°`, `line 162`), holding for `MOTION_HUMP_HOLD_TICKS = 6U` (`120 ms`). Motion checks **zero** validity/health flags: if I2C1 disconnects while pitch was `>= 3°`, `s_pitch_half_deg` stays frozen for `IMU_I2C_FAIL_THRESHOLD = 5` reads (`40–50 ms`, `task_imu.c:33`) until Safety's watchdog trips, or indefinitely if transient 1–4 read failures keep resetting `s_consecutive_fail`.
- **Silent 10 s level fallback produces permanent false hump boost (`task_imu.c:612-619`, `S4-U07`):** After `IMU_LEVEL_TIMEOUT = 1000U` samples (`10 s`, `task_imu.c:89`), `update_level()` falls back to `finish_level(0, 0, 0, (s_fz_mg >= 0) ? 1000 : -1000)` and marks `s_level_done = true` with no fault bit. In `testing/subsystem4/TEST_LOG.md` `S4-U07`, the real chassis measured **`8` half-deg (`4.0°`, up to `4.5°`) static pitch when flat**. Falling back to `0` leaves a permanent `+4.0–4.5°` pitch error above BOTH `IMU_HUMP_PITCH_DEG = 3U` (`config/thresholds.h:22`) and `MOTION_HUMP_PITCH_ON_HD = 6U` (`task_motion.c:162`), causing continuous false hump boost (`+45 mm/s`, `+180 pm`) and false `DREP_TEL_HUMP` events on flat ground!
- **All-zero accelerometer vector defeats IMU watchdog & disables lift protection (`task_imu.c:695-764`, `imu_lift.c:75-96`):** `imu_read_accel()` returns `true` whenever I2C bytes transfer (`drv_imu.c:114-122`), without checking for all-zero `(0,0,0)` (powered-down/reset LSM303D outputs `0` before `CTRL1` takes effect, or shorted SDA low reads `0x00`) or identical stuck values across samples. Yet `task_imu.c:764` sets `EVF_SAFETY_IMU_ALIVE` anyway, keeping the 50 ms Safety watchdog satisfied! Worse, if `(0,0,0)` averages into `update_level()`, `imu_lift_set_up()` sets `p_lift->up_mag = 0U` and `p_lift->armed = false` (`imu_lift.c:75-76`), causing `imu_lift_step()` (`imu_lift.c:93-96`) to return `IMU_LIFT_NONE` forever — silently disabling `FR12` lift detection!
- **Unverified reconnect (`task_imu.c:714-734`):** Clears `EVF_SAFETY_FAULT_IMU_I2C` immediately after `imu_reconfigure()` returns `true`, before confirming `WHO_AM_I` + plausible gravity vector (`700 <= mag_mg <= 1300 mg`).

---

## #6 [OBVIOUS CHANGE]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry & Remote Link)  
**Affected Requirement / Component:** FR1, FR2, FR3 / `app_program/subsystem1/lwipopts.h`, `tel_link.c`, `drv_wifi.c`, `task_tel.c`  
**Area:** Recovery and safe resumption; Memory / resources; State machines; Interfaces / messages  

**What needs to change?:**
Fix the confirmed `S1-M03` reconnect lwIP heap exhaustion (`MEM_SIZE = 4096 B`), fix `tel_link.c` flapping between `DEGRADED` and `CONNECTED` on non-publishing wakes, and treat `mqtt_subscribe(TEL_TOPIC_CMD_FILTER)` failure as a connection failure instead of marking the car online while blind to remote `STOP`/`RESET`.

**Decision:** Accept  

**Why?:**
When Wi-Fi drops and returns, Telemetry is currently trapped in a permanent reconnect loop (`S1-M03 FAIL`), its 5 s keep-alive teardown timer is defeated by 9-of-10 non-publishing wakes clearing `DEGRADED`, and a failed subscription leaves the car advertising `car/status = '1'` online while silently unable to receive `car/cmd/stop` or `car/cmd/reset`.

**Engineering Evidence:**
- **Verified `S1-M03` FAIL (`testing/subsystem1/TEST_LOG.md`, 2026-11-02 row 69):**
  - `lwipopts.h:53,55`: `TCP_MSS = 1460`, `TCP_SND_BUF = 2 * TCP_MSS = 2920 B`. `lwipopts.h:30`: `MEM_SIZE = 4096 B`.
  - When `drv_wifi.c:366` calls `mqtt_disconnect(&s_client)`, the closing TCP PCB holds up to `2920 B` of unsent frames in lwIP's `4096 B` heap during FIN/reconnect, leaving `4096 - 2920 = 1176 B < TCP_MSS (1460 B)`. Every subsequent `mqtt_client_connect()` (`drv_wifi.c:201`) fails with `ERR_MEM (-1)`, blocking recovery indefinitely.
  - **Sizing fix:** Raising `MEM_SIZE` to `8192 B` leaves `8192 - 2920 = 5272 B` free (`> 3.6 × TCP_MSS`), costing `+4 KB` out of RP2040's `264 KB` SRAM (current lwIP budget `~16 KB`, `lwipopts.h:29`, with `> 100 KB` unused SRAM headroom).
- **`DEGRADED <-> CONNECTED` flapping (`tel_link.c:112-126`, `task_tel.c:8-14`, `testing/subsystem1/TEST_LOG.md` row 69):** `task_tel.c` wakes every `TEL_WAKE_PERIOD_MS = 20 ms` (`config/timing.h:101`) and only publishes periodic frames every 10th wake (`200 ms`, `TEL_PUBLISH_EVERY_N_WAKES = 10U`). On the 9 intermediate wakes, `s_publish_failed` is `false`. `tel_link.c:113` sees `p_in->net_up && (!p_in->publish_failed) && p_in->outbox_empty` and immediately transitions `DEGRADED -> CONNECTED` (or resets `entered_ms` at line 120), preventing `degrade_timeout_ms` (`5000 ms`) from expiring for `6 s+`. Fix: replace `!publish_failed` recovery with an explicit `publish_succeeded` signal set only when an MQTT publish actually returns `ERR_OK`.
- **Silent subscribe failure (`drv_wifi.c:90-97, 104-113`):** `on_mqtt_connection()` sets `s_mqtt_up = true`, sets `s_stage = NET_STAGE_UP`, publishes `car/status = '1'`, and only then calls `mqtt_subscribe(p_client, TEL_TOPIC_CMD_FILTER, ...)`. If `on_subscribed()` receives `err != ERR_OK`, it only calls `dbg_printf` (`line 95`) and leaves `s_stage = NET_STAGE_UP` — leaving the dashboard believing remote `STOP`/`RESET` works when the broker subscription failed! Fix: set `s_stage = NET_STAGE_UP` and publish `car/status = '1'` only inside `on_subscribed(..., ERR_OK)`, and transition to `NET_STAGE_FAILED` on `err != ERR_OK`.

---

## #7 [OBVIOUS CHANGE]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry)  
**Affected Requirement / Component:** FR1, IF-12, IF-08 / `app_program/subsystem1/task_tel.c`, `src/common/data_repr.h`  
**Area:** Interfaces / messages; Shared data / concurrency / interrupts; Safety / fault handling  

**What needs to change?:**
Expire IF-12 sensor snapshot validity bits (`ir_valid`, `range_valid`, `att_valid`) whenever their producer task stops updating past a bounded timeout, and check `motion_state_t.tick_ms` freshness in `task_tel.c` before publishing `telemetry_motion` as live data.

**Decision:** Accept  

**Why?:**
Currently, once each sensor writes one reading after boot, `TEL_SNAP_VALID` (`1UL << 31`) stays permanently `1` forever. If `task_lf`, `task_obs_scan`, `task_imu`, or `task_motion` hangs or stops producing fresh readings, Telemetry keeps broadcasting the last frozen values on MQTT with `*_valid = 1` every `200 ms`, violating the requirement that invalid/unavailable measurements must not continue looking valid.

**Engineering Evidence:**
- **`TEL_SNAP_VALID` never clears (`task_tel.c:40, 64-66, 126-147, 523-534`):** `tel_set_ir_raw()`, `tel_set_range_cm()`, and `tel_set_attitude()` OR `TEL_SNAP_VALID` into `s_snap_ir`, `s_snap_range`, `s_snap_att` on write and never clear it. `publish_periodic()` (`task_tel.c:532-533`) passes `(ir & TEL_SNAP_VALID) != 0U`, `(range & TEL_SNAP_VALID) != 0U`, `(att & TEL_SNAP_VALID) != 0U` verbatim to `drep_frame_sensor_valid()` (`data_repr.h:851, 864-871`, payload spare bits `[75:73]` of the 10-byte `telemetry_sensor` frame).
- **Zero-wire-cost fix:** Add three single-writer 32-bit timestamps (`s_snap_ir_ms`, `s_snap_range_ms`, `s_snap_att_ms` — `12 B` RAM total). In `publish_periodic()`, require `(now_ms - snap_ms) <= stale_ms` (`IR`: `200 ms` = `40 × 5 ms`; `Attitude`: `100 ms` = `10 × 10 ms`; `Range`: `500 ms` = `8.3 × 60 ms`). Because `drep_frame_sensor_valid()` bits `73..75` already exist (`data_repr.h:851`), the v1 10-byte wire layout stays `100%` unchanged! Note: also suppress `tel_set_attitude()` inside `task_imu.c:777` (or clear validity) if IMU I2C has failed (`#5`).
- **Stale `telemetry_motion` (`task_tel.c:516-519`, `data_repr.h:819-830`):** `publish_periodic()` transmits `ms` whenever `motion_get_state(&ms) == E_OK`, even if `ms.tick_ms` stopped updating (Motion stalled). Skip emitting live motion frames (or mark stale in spare bit `114` of the `120-bit` `15 B` `telemetry_motion` payload, where only `114` bits are currently used) when `(now_ms - ms.tick_ms) > MOTION_STALE_MS` (`200 ms`).

---

## #8 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform Boot)  
**Affected Requirement / Component:** FR4, NFR1, NFR10 / `app_program/app_main.c`, `app_program/app.h`  
**Area:** Tasks / components; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Replace blind `APP_CHECK()` error logging in `usermain()` with tiered boot-failure handling: immediately brake motors and refuse autonomous task startup on Tier-1 critical failures (`ipc_init()`, `subsystem2_init/start()`, `subsystem4_init/start()`), while allowing controlled degraded startup on non-critical Telemetry (`subsystem1`) failure or diagnosable fault-latching on peripheral (`subsystem3`/`subsystem5`) failure.

**Decision:** Modify  

**Why?:**
`APP_CHECK()` only logs `dbg_put_err()` and allows `usermain()` to continue creating and starting tasks even when `ipc_init()` failed (`EVF_SAFETY = 0`, `MBF_NAV_IN = 0`) or `safety` / `motion` failed to start. Continuing startup on broken RTOS objects or missing safety tasks guarantees unpredictable runtime behaviour and fail-open failure modes (`#2`).

**Engineering Evidence:**
- **Fire-and-forget macro (`app_program/app.h:114-122`):** `APP_CHECK(er, p_what)` checks `if (app_check_er_ < (int32_t)E_OK) { dbg_put_err((p_what), app_check_er_); }` and neither returns nor halts.
- **Unchecked boot chain (`app_program/app_main.c:91-110, 123-144`):** All 12 calls in `usermain()` (`ipc_init()`, `dbg_drain_start()`, `subsystem1..5_init()`, `subsystem1..5_start()`) wrap the result in `APP_CHECK(...)`. If `app_start_task()` fails (`E_LIMIT` in `stack_mon_register()`, `tk_cre_tsk()`, or `tk_sta_tsk()`), `subsystemN_start()` returns `< 0`, `APP_CHECK()` prints it, and `usermain()` starts the remaining subsystems anyway (for instance, driving without `safety` or `obs_scan`).
- **`init` stack constraint (`config/priorities.h:122`, `testing/subsystem4/logs/2026-11-19_1625_s4_unit.txt:245`):** `usermain()` executes on the fixed `1 KB` `INITTASK_STKSZ` stack (`532 of 1152 B used, 620 B never used`), so boot error paths must use `dbg_puts` / `dbg_put_err` only (zero `dbg_printf` or large stack locals).

**Modified Decision:**
Partition `usermain()` into three deterministic failure tiers:
- **Tier 1 — Critical Control / IPC (`ipc_init()`, `subsystem2_init/start()`, `subsystem4_init/start()`):** On `er < E_OK`, immediately call `motion_emergency_brake()`, set `EVF_SAFETY_FAULT_LATCHED | EVF_SAFETY_FAULT_BOOT_INIT` (`#14`, if `EVF_SAFETY` was created), log via `dbg_put_err()`, do **not** start Subsystems 3/5, leave `dbg_drain` (and `subsystem1` if IPC succeeded) running for telemetry/console diagnosis, and sleep (`tk_slp_tsk(TMO_FEVR)`).
- **Tier 2 — Navigation Peripherals (`subsystem3_init/start()`, `subsystem5_init/start()`):** On `er < E_OK`, set `EVF_SAFETY_FAULT_LATCHED | EVF_SAFETY_FAULT_BOOT_INIT` so VC locks into `VC_SYS_SAFETY_FAULT_LATCHED` and `START` is permanently refused while `subsystem1` reports the exact fault over MQTT.
- **Tier 3 — Non-Critical Telemetry (`subsystem1_init/start()`):** Log `[ERR]` and continue autonomous startup (matching NFR1 / FR3 isolation), because local safety (`P1` Safety, `P4` LF, `P6/P7` Obstacle) operates without MQTT.

**Alternative considered:** Calling `port_fault_shutdown()` (watchdog reset) on any `APP_CHECK()` failure. Rejected because a persistent hardware/IPC init fault or non-critical Wi-Fi error would cause an infinite reset loop (`10 ms` per reboot) and wipe live USB console diagnostics instead of holding the vehicle safely braked and diagnosable.

---

## #9 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform)  
**Affected Requirement / Component:** FR12, NFR10 / `src/rtos/port/port_fault.c`, `app_program/subsystem2/drv_motor.c`, `app_program/subsystem4/task_safety.c`, `app_program/subsystem2/task_motion.c`  
**Area:** Hardware / software interaction; Safety / fault handling; Timing / performance  

**What needs to change?:**
Call `motor_brake_all()` immediately inside `port_fault_from_handler()` and `port_fault_shutdown()`, and enable the RP2040 hardware watchdog timer (`watchdog_enable`) kicked by `P1` Safety only when both Safety and Motion heartbeat counters are advancing.

**Decision:** Accept  

**Why?:**
This critical hardware hazard is not explicitly named in the requirement change: on the RP2040, PWM slices 4 and 5 (`GP8–GP11`) are autonomous hardware timers that **continue outputting their last non-zero motor PWM duty indefinitely** if the Cortex-M0+ CPU hangs in a HardFault, breakpoint loop, or RTOS deadlock. Target test logs have already documented real `panic()` HardFaults and silent freezes.

**Engineering Evidence:**
- **No motor brake in HardFault / shutdown (`src/rtos/port/port_fault.c:38, 97-111, 134-145`):** Neither `port_fault_from_handler()` nor `port_fault_shutdown()` calls `motor_brake_all()` (or clears PWM slice 4/5 outputs). When debug halt is active (`PORT_FAULT_HALT=1` or `port_fault_halt=1`), `after_record()` spins forever in `for (;;) { __breakpoint(); }` (`port_fault.c:101-104`) while motors keep running at full pre-fault duty! Even in default reset mode, `watchdog_reboot(0U, 0U, PORT_FAULT_RESET_DELAY_MS)` waits `10 ms` (`port_fault.c:38`) — `5 mm` at `50 cm/s` (NFR2) plus coasting until next boot's `subsystem2_init()` drives `GP8–GP11` LOW.
- **No runtime hardware watchdog:** Across the entire codebase, `watchdog_enable()` is never called (`watchdog_reboot()` is only called *inside* `port_fault.c` after a fault already triggers an exception). If a task spinlock or memory corruption stalls the RTOS scheduler without raising HardFault, PWM outputs stay active forever.
- **Verified target crashes (`testing/subsystem4/TEST_LOG.md` rows 81, 96):** Row 81 records `panic()` HardFault inside IRQ 31 (SDK USB stdio background IRQ); row 96 records both a second IRQ 31 HardFault and a silent freeze where output stopped completely without reset.
- **RP2040 Hardware Watchdog Timing:** Enabling a `500 ms` hardware watchdog kicked inside `task_safety.c` (`50 ms` worst-case cycle) only when `motion_release_count` advances (`20 ms` period → `25` Motion cycles per WDT window, `10×` margin) guarantees a full chip reset (`RESETS` clears PWM/IO_BANK0) within `<= 500 ms` on any CPU freeze, while `motor_brake_all()` inside `port_fault.c` stops motors within microseconds of a HardFault.

---

## #10 [KNOCK-ON CHANGE]
**Buddy:** Buddy 5 (Chay Han — Subsystem 5); Secondary: Buddy 3 (Jin Yong — Subsystem 3)  
**Affected Requirement / Component:** FR7, FR14 / `app_program/subsystem5/task_obs_avoid.c`, `app_program/subsystem3/lf_core.c`, `app_program/subsystem3/lf_straddle.c`  
**Area:** State machines; Interfaces / messages; Recovery and safe resumption  

**What needs to change?:**
Eliminate the `LF_ST_SUSPENDED` state-machine deadlock triggered when obstacle avoidance aborts or releases without `EVF_NAV_LINE_REACQUIRED`: allow `st_suspended()` (`lf_core.c`) and `sd_st_suspended()` (`lf_straddle.c`) to exit `LF_ST_SUSPENDED` whenever `!avoid_active` (entering `ON_LINE` if sensor sees line, else bounded `REACQUIRING`), and ensure `abort_manoeuvre()` cleanly stops VC / clears authority so Line Following never drives blind.

**Decision:** Accept  

**Why?:**
Expanding safe aborts in `#4` increases the frequency of `abort_manoeuvre()`. Today, `abort_manoeuvre()` clears `EVF_NAV_AVOID_ACTIVE` without setting `EVF_NAV_LINE_REACQUIRED`, whereas Line Following requires `reacquired && !avoid_active` to ever exit `LF_ST_SUSPENDED` — a deadlock already witnessed on the real car!

**Engineering Evidence:**
- **`abort_manoeuvre()` never sets `REACQUIRED` (`task_obs_avoid.c:567-598`):** Line 597 calls `release(now_ms, false, false)`, where `reacquired || resume_lf` is `false` (`line 571`), so `EVF_NAV_LINE_REACQUIRED` stays `0` while `EVF_NAV_AVOID_ACTIVE` is cleared (`line 580`) and `DREP_NAV_AVOID_RELEASE` is sent (`line 579`).
- **One-way trap in LF (`lf_straddle.c:495-513`, `lf_core.c:893-911`):** Both `sd_st_suspended()` and `st_suspended()` only transition out of `LF_ST_SUSPENDED` under `if (p_in->reacquired && !p_in->avoid_active)`.
- **Observed failure (`testing/subsystem5/TEST_LOG.md` row 104, `2026-11-19_s5_fullchain.txt`):** LF remained stuck in `SUSPENDED` for **340 consecutive log lines** after avoidance released with `reacquired=0`, while VC sent `DREP_MC_FWD` (`task_vc.c:275-276`), driving the car blind until it crashed and tripped the IMU lift detector.
- **Fix:** Inside `st_suspended()`/`sd_st_suspended()`, transition out whenever `!p_in->avoid_active`: if `p_in->reacquired` or a sensor sees the line (`DARK` / contact), enter `ON_LINE`; if `!p_in->reacquired` while still `p_in->running`, enter bounded `REACQUIRING` (`T_search`) rather than hanging indefinitely. In addition, if `abort_manoeuvre()` aborts due to a hard obstacle/sensor block (`OBS_AV_BLOCKED`), send `DREP_NAV_AVOID_BLOCKED` (so VC stops to `READY_WAITING`, `#4`) rather than `DREP_NAV_AVOID_RELEASE` (which restarts `FWD`).

---

## #11 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 4 (Vincent — Subsystem 4), Buddy 1 (Damian — Subsystem 1)  
**Affected Requirement / Component:** FR12, NFR6 / `app_program/subsystem2/task_vc.c`, `task_motion.c`, `app_program/subsystem4/task_safety.c`, `app_program/subsystem1/task_tel.c`  
**Area:** Shared data / concurrency / interrupts; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Protect `s_vc.state_bits` against non-atomic cross-task read-modify-write races on Cortex-M0+; remove local shadow caches of `EVF_SAFETY` bits (`s_mot.fault_bits_set`, `s_wifi_loss_set`) in favor of `tk_ref_flg()` read-before-write; and clear `EVF_SAFETY_FAULT_IMU_WDOG` (`b6`) inside `task_safety.c` when `IMU_ALIVE` resumes (while keeping `EVF_SAFETY_FAULT_LATCHED` set).

**Decision:** Modify  

**Why?:**
1. ARMv6-M (Cortex-M0+) lacks `LDREX`/`STREX` atomic RMW instructions; public APIs mutating `s_vc.state_bits` without disabling dispatch risk silently dropping state/fault flags under preemption.
2. Local `EVF_SAFETY` shadow caches desynchronize whenever resets or test harnesses touch `EVF_SAFETY`, silently suppressing subsequent fault notifications.
3. `task_safety.c` never clears `EVF_SAFETY_FAULT_IMU_WDOG` once the IMU recovers — which both leaves a false persistent watchdog fault on MQTT heartbeats forever after recovery and would permanently block gated `REMOTE_RESET` (`#2`)!

**Engineering Evidence:**
- **Non-atomic RMW on `s_vc.state_bits` (`task_vc.c:79, 108, 134, 175, 222, 479, 567, 605`):** Public API functions `vc_init()`, `vc_complete_selftest()`, and `vc_set_authority()` write `s_vc.state_bits` (`volatile uint16_t`) via read-modify-write (`LDRH` / `ORRS` or `BICS` / `STRH`) without `tk_dis_dsp()`/`tk_ena_dsp()`. If called from `app_test` (P11) or another task while `vc_task_entry` (P2) wakes, preempting mid-RMW overwrites the System FSM state `[3:0]` or authority bit `[4]`.
- **Local `EVF_SAFETY` cache desync (`task_motion.c:219, 480-495`, `task_tel.c:58, 569-573` vs `task_obs_scan.c:747-783`):** Both `task_motion.c` (`s_mot.fault_bits_set`) and `task_tel.c` (`s_wifi_loss_set`) compare against private shadow variables instead of querying `tk_ref_flg(EVF_SAFETY, &rflg)`. `task_obs_scan.c:747-755` explicitly documents why caching fails: if another task or `APP_TEST` clears a bit on `EVF_SAFETY`, the cached writer thinks the bit is still `1` and stops updating it.
- **Stuck `EVF_SAFETY_FAULT_IMU_WDOG` (`task_safety.c:91, 116-120`, `task_vc.c:220`):** Safety sets `EVF_SAFETY_FAULT_IMU_WDOG` on timeout (`line 91`). When `EVF_SAFETY_IMU_ALIVE` resumes (`lines 116-120`), Safety clears `s_wdog_tripped = false` and logs `[SAFETY] IMU heartbeat back (fault stays latched)`, but **never clears `EVF_SAFETY_FAULT_IMU_WDOG`**. Meanwhile `task_vc.c:220` clears ONLY `~EVF_SAFETY_FAULT_LATCHED` (`ARCHITECTURE.md` D9 single-writer-per-bit contract). After any transient IMU unplug & replug (`S4-U14`/`S4-U16`), `b6` remains `1` in every heartbeat until reboot, breaking both dashboard health display and gated `RESET` (`#2`).

**Modified Decision:**
1. Wrap `s_vc.state_bits` mutations inside public/cross-task helpers (or all `s_vc.state_bits` RMWs) with `tk_dis_dsp()` / `tk_ena_dsp()` (cost: ~4 cycles, zero ISR masking).
2. Remove `s_mot.fault_bits_set` and `s_wifi_loss_set`; read current `EVF_SAFETY` state via `tk_ref_flg()` before `tk_set_flg()` / `tk_clr_flg()` (uniform with `task_obs_scan.c:756-783`).
3. In `task_safety.c:116-120`, upon `IMU_ALIVE` returning, execute `tk_clr_flg(EVF_SAFETY, (UINT)~EVF_SAFETY_FAULT_IMU_WDOG)` — preserving Safety's exclusive single-writer ownership of `b6` while keeping `EVF_SAFETY_FAULT_LATCHED` (`b2`) latched until stationary operator `RESET`.

**Alternative considered:** Clearing `EVF_SAFETY_FAULT_IMU_WDOG` inside `task_vc.c` during `REMOTE_RESET`. Rejected because it violates `ARCHITECTURE.md` D9 single-writer ownership per `EVF_SAFETY` bit (`task_safety` owns `b6`; VC owns only `b2`) and would temporarily hide an active dead IMU during the 50 ms watchdog interval after `RESET`.

---

## #12 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 4 (Vincent — Subsystem 4)  
**Affected Requirement / Component:** FR12, `design.md` §5 / `app_program/subsystem2/task_vc.c`, `app_program/subsystem4/task_safety.c`, `app_program/subsystem4/task_imu.c`  
**Area:** Recovery and safe resumption; Safety / fault handling; State machines  

**What needs to change?:**
*(Candidate auto-recovery proposal reviewed and rejected)* Automatically clear `EVF_SAFETY_FAULT_LATCHED` and resume autonomous navigation (`VC_SYS_NAV_LINE_TRACKING`) whenever a transient Class A fault clears by itself (e.g., `imu_reconfigure()` succeeds after `S4-U16` cable reconnect, or stalled wheels become free).

**Decision:** Reject  

**Why?:**
While the new requirement cites *"attempting recovery, or requiring intervention"* as options, allowing auto-recovery to clear a latched Class A safety fault and re-energise the motors without explicit operator intervention would directly endanger the operator and violate `FR12`. If a user picks up the car, reseats a loose I2C/encoder cable, or un-jams a wheel by hand, auto-resuming motor PWM could injure fingers or launch the car uncontrollably off the bench.

**Engineering Evidence:**
- **`docs/trib2/requirements.md` FR12 & `design.md` §5:** Explicitly mandates that lift/overturn (`EVF_SAFETY_LIFTED`) and IMU watchdog (`EVF_SAFETY_FAULT_IMU_WDOG`) faults latch to end the run (`VC_SYS_SAFETY_FAULT_LATCHED`) until an explicit remote `RESET` arrives after `SAFETY_T_STILL_MS = 500 ms` stationary — and `RESET` only returns to `VC_SYS_READY_WAITING`, requiring a separate `REMOTE_START` (`task_vc.c:223-224`) before any torque is applied.
- **Two-tier architecture preserved (`task_imu.c:714-734`, `task_safety.c:116-120`):** Subsystem drivers already perform safe hardware recovery (`imu_reconfigure()` restores I2C sampling; Wi-Fi reconnects) and clear their health bit (`#11`), while keeping `EVF_SAFETY_FAULT_LATCHED` (`b2`) strictly under operator `RESET` control. All Class A propulsion/safety faults (`STALL`, `ENCODER`, `IR_SENSOR`, `IMU_WDOG`, `LIFTED`) must preserve this exact separation.

---

## #13 [HIDDEN RISK]
**Buddy:** Buddy 1 (Damian — Subsystem 1); Secondary: Buddy 2 (Ruo Chen — Subsystem 2), Buddy 3 (Jin Yong — Subsystem 3)  
**Affected Requirement / Component:** FR2, FR9, `TEAM.md` §5 IF-01 / `app_program/subsystem1/drv_wifi.c`, `app_program/subsystem1/task_tel.c`, `app_program/subsystem3/task_bc.c`  
**Area:** Interfaces / messages; Shared data / concurrency / interrupts; Safety / fault handling  

**What needs to change?:**
Guarantee that `REMOTE_STOP` is never silently dropped in `drv_wifi.c`'s 4-slot RX ring (`NET_RX_SLOTS`) or `task_tel.c`'s `MBF_NAV_IN` send path; update `task_bc.c` to post decoded barcode commands using `NAV_MBF_SEND_TMO_MS` (`5 ms`) with counted/logged failure instead of `(void)tk_snd_mbf(..., TMO_POL)`; and mask the 8-bit `drv_bc_overflow_count()` subtraction in `task_bc.c:237`.

**Decision:** Accept  

**Why?:**
During a fault, sensor failure, or command burst, silently dropping the safety-critical `REMOTE_STOP` packet behind 4 other RX slots or a momentarily full `MBF_NAV_IN` leaves the car running despite operator intervention. Meanwhile, `task_bc.c` violates `IF-01` by discarding junction turns on a `0 ms` non-blocking timeout without any error count, and prints an `uint32_t` underflow (`4294967041`) whenever `s_ring.overflow` wraps at 256.

**Engineering Evidence:**
- **Command RX ring overflow drops `STOP` (`app_program/subsystem1/drv_wifi.c:156-160`):** `on_incoming_data()` checks `if ((s_rx_head - s_rx_tail) >= NET_RX_SLOTS)` (`4` slots) and increments `s_rx_dropped++` without prioritizing `car/cmd/stop` or raising an error to Telemetry. Fix: maintain a dedicated `volatile bool s_stop_urgent` flag on `car/cmd/stop` (or evict non-STOP slots) so `STOP` can never be overwritten or dropped by a command burst.
- **Unretried `MBF_NAV_IN` `STOP` drop (`app_program/subsystem1/task_tel.c:333-339`):** If `tk_snd_mbf(MBF_NAV_IN, ...)` times out after `NAV_MBF_SEND_TMO_MS` (`5 ms`), `tel_cmd_receive()` returns `TEL_CMD_POST_FAIL` (`handle_commands()` at line 361 casts `(void)`) and the command is lost forever! Fix: latch `s_pending_stop` in `task_tel.c` and retry every `20 ms` wake until `E_OK` (and after `2` failed wakes = `40 ms`, trigger `motion_emergency_brake()` as fail-safe fallback).
- **Barcode drops `IF-01` turns (`task_bc.c:126` vs `task_lf.c:157`, `task_obs_avoid.c:254-255`):** `task_bc.c:126` executes `(void)tk_snd_mbf(MBF_NAV_IN, &nav, (INT)sizeof(nav), TMO_POL);` — using `TMO_POL` instead of `NAV_MBF_SEND_TMO_MS` (`5U`, `config/timing.h:201`, explicitly allocated in `TEAM.md` §5 & `subsystem3.h:21`) and ignoring failure. Fix: use `(TMO)NAV_MBF_SEND_TMO_MS` and count/log errors on `er != E_OK`.
- **Overflow counter wrap bug (`task_bc.c:237`):** `uint32_t const ovf = drv_bc_overflow_count() - ovf0;` subtracts two `uint8_t` counters promoted to `uint32_t` without `& 0xFFU`; if `drv_bc_overflow_count()` wraps `255 -> 0`, `ovf = 0xFFFFFF01` (`4294967041`), corrupting diagnostics (`line 142`). Mask with `& 0xFFU`.

---

## #14 [KNOCK-ON CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Shared / All Buddies)  
**Affected Requirement / Component:** All FRs/NFRs / `docs/trib2/requirements.md`, `src/common/ipc_types.h`, `src/common/data_repr.h`, `docs/DATA_REPRESENTATION.md`, `tools/dashboard/tel_codec.py`, `app_program/subsystem2/subsystem2.h`, `app_program/subsystem3/subsystem3.h`  
**Area:** Requirements; Interfaces / messages; Memory / resources; Timing / performance  

**What needs to change?:**
Add the Fault-Response Matrix (Section 2 above) to `docs/trib2/requirements.md`; expand `EVF_SAFETY` from 10 to 12 bits (`b10`: `EVF_SAFETY_FAULT_US_SENSOR`, `b11`: `EVF_SAFETY_FAULT_BOOT_INIT`) inside the existing v1 MQTT frame payloads; expose `subsystem2_sensors_ok()` and `subsystem3_sensors_ok()`; define `MOTION_STALE_MS` (`100 ms`) over `motion_state_t.tick_ms`; and update `docs/DATA_REPRESENTATION.md` + `tools/dashboard/tel_codec.py` with verified SRAM and stack budgets.

**Decision:** Modify  

**Why?:**
The new fault detections in `#1–#4` and `#8` add two fault categories not present in `EVF_SAFETY` (`b0–b9`), require standardized public boot-health getters (`subsystem2_sensors_ok()`, `subsystem3_sensors_ok()`) for VC's self-test gate (`#2`), and require `motion_state_t` freshness checks across consumers (`#1`, `#3`, `#4`, `#7`) — while preserving strict `DREP_WIRE_VER = 1` wire framing, zero struct bloat, and RP2040 stack/RAM margins.

**Engineering Evidence:**
- **Zero-byte-growth MQTT wire math (`src/common/data_repr.h:758, 902-920`):**
  - `DREP_FT_HEARTBEAT` (`type 7`): `drep_frame_len(7) = 6 B = 48 bits`. Packed bits = `uptime_ms` (`32 b`) + `sys_state` (`4 b`) + `evf_safety_bits` (`10 b`) = **`46 bits`** → **2 spare bits (`b10`, `b11`) remain!** Raising `EVF_SAFETY_ALL_MASK` from `0x3FFUL` (10 bits) to `0xFFFUL` (12 bits) packs `32 + 4 + 12 = 48 / 48 bits` (`6 B`, **0 extra bytes**).
  - `DREP_FT_EVENT_FAULT` (`type 6`): `drep_frame_len(6) = 5 B = 40 bits`. Packed bits = `ts_ms` (`24 b`) + `10 b` = **`34 bits`** → expanding to `12 b` uses `36 / 40 bits` (`5 B`, `4` spare bits remaining, **0 extra bytes**).
- **Zero-byte-growth `motion_state_t` staleness (`src/common/ipc_types.h:68-73, 119`):** `motion_state_t` is fixed at `16 B` and already carries `uint32_t tick_ms`. Defining `#define MOTION_STALE_MS (100U)` (`5 × MOTION_PERIOD_MS`, matching `MOTION_T_HOLD_MS`) lets every reader (`VC`, `LF`, `ObsScan`, `ObsAvoid`, `Tel`, `IMU`) validate `(uint32_t)(now_ms - ms.tick_ms) <= MOTION_STALE_MS` with **0 B** IPC change.
- **Verified SRAM & Stack Headroom:**
  - SRAM: RP2040 has `264 KB`; kernel pool is `96 KB` (`config/priorities.h:124`), lwIP RAM is `~16 KB` (`lwipopts.h:29`). Total additions across `#1–#14` (`+4096 B` lwIP `MEM_SIZE` in `#6`, `12 B` IF-12 timestamps in `#7`, `< 64 B` fault/WDT state) consume `~4.2 KB`, leaving `> 100 KB` SRAM margin.
  - Measured task stack high-water headroom (worst-case minimum `never used` across `testing/subsystem3/logs/2026-11-16_s3_u14_uturn_hold_run5.txt:604-614` and `testing/subsystem4/logs/2026-11-19_1625_s4_unit.txt:245-253`):
    `init`: **`620 B`** free (of `1152 B`); `vc`: **`1236 B`**; `motion`: **`1316 B`**; `safety`: **`1292 B`**; `lf`: **`1076 B`**; `imu`: **`1468 B`**; `obs_scan`: **`1076 B`**; `obs_avoid`: **`1052 B`**; `tel`: **`4216 B`**; `dbg_drain`: **`1020 B`**. All added checks use `< 64 B` stack; keeping `usermain()` limited to `dbg_puts` / `dbg_put_err` protects `init`'s `620 B` margin.

**Modified Decision:**
Expand `EVF_SAFETY_ALL_MASK` from 10 to 12 bits (`0xFFFUL`, adding `b10` `EVF_SAFETY_FAULT_US_SENSOR` and `b11` `EVF_SAFETY_FAULT_BOOT_INIT`) inside the existing `6 B` heartbeat and `5 B` fault wire frames; add `subsystem2_sensors_ok()` and `subsystem3_sensors_ok()`; use existing `motion_state_t.tick_ms` with `MOTION_STALE_MS = 100U`; update `docs/trib2/requirements.md`, `docs/DATA_REPRESENTATION.md`, and `tools/dashboard/tel_codec.py`.

**Alternative considered:** Widening `DREP_FT_HEARTBEAT` to `7 B` (16-bit safety mask) and adding an explicit `uint32_t valid` word to `motion_state_t` (`16 B -> 20 B`). Rejected because lengthening heartbeat breaks backward compatibility with `DREP_WIRE_VER = 1` decoders when `48` bits already hold `12` fault bits, and `motion_state_t.tick_ms` already solves freshness detection without growing `MTX_MOTION` copy time.

---

## #15 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 3 (Jin Yong — Subsystem 3), Buddy 5 (Chay Han — Subsystem 5)  
**Affected Requirement / Component:** FR5, FR7, FR13, NFR14, NFR15 / `app_program/subsystem2/drv_motor.c`, `drv_encoder.c`, `app_program/subsystem3/sources.cmake`, `app_program/subsystem5/task_obs_scan.c`, `config/timing.h`, `docs/trib2/requirements.md`  
**Area:** Hardware / software interaction; Timing / performance; Requirements  

**What needs to change?:**
Remove the dead-branch hardcoded `(1U)` overrides in `drv_motor.c` and `drv_encoder.c` so `config/pins.h` direction macros take effect; align `sources.cmake` defaults so fault/rail checks (`#3`) are active in both edge and straddle builds; and reconcile the code-vs-spec drift in `OBS_BYPASS_SLOW_BRINGUP` (`60 s` budget vs NFR14 `5.0 s`) and ultrasonic stationary sweep (`3° / 51..129° / 8 ms` vs `docs/trib2/requirements.md` `5° / 50–130° / 12 ms`).

**Decision:** Accept  

**Why?:**
Silent configuration overrides and doc-to-code drift become acute failure hazards during maintenance and verification: swapping a replacement motor or encoder and flipping `config/pins.h` currently has **no effect** in firmware — producing positive-feedback PID runaway! Building without manual CMake cache flags drops straddle checks; and a faulty bypass timeout takes `70 s` (`14×` NFR14's `5.0 s` cap) while `OBS_BYPASS_SLOW_BRINGUP = 1U`.

**Engineering Evidence:**
- **Hardcoded `(1U)` invert overrides (`drv_motor.c:67-71`, `drv_encoder.c:57-61` vs `config/pins.h:71,80`):**
  - `drv_motor.c:67-71` defines `MOTOR_L_EFF_INVERT (1U)` in **both** the `#if MOTOR_L_DIR_INVERT != 0U` branch AND the `#else` branch.
  - `drv_encoder.c:57-61` defines `ENC_R_EFF_INVERT (1U)` in **both** the `#if ENC_R_DIR_INVERT != 0U` branch AND the `#else` branch.
  - Because `config/pins.h:71,80` already sets `MOTOR_L_DIR_INVERT = (1U)` and `ENC_R_DIR_INVERT = (1U)`, toggling those macros to `0U` in `config/pins.h` is silently overridden at compile time!
- **Silent CMake build-mode fallback (`app_program/subsystem3/sources.cmake:103`):** Defaults `S3_LF_STRADDLE` to `OFF` (`lf_core.c`), whereas track builds pass `-DS3_LF_STRADDLE=ON` (`lf_straddle.c`, the only core that currently checks `sd_rail()` and `LF_SD_OVER_RAW = 3000U`, `lf_straddle.h:224`).
- **Avoidance budget & sweep drift (`config/timing.h:153,162,167`, `task_obs_avoid.c:959-960`, `task_obs_scan.c:69-100` vs `docs/trib2/requirements.md` FR13/NFR15):**
  - `OBS_BYPASS_SLOW_BRINGUP = 1U` sets `OBS_BYPASS_BUDGET_MS = 60000U`, so `s_budget_deadline_ms = 60000 + 10000 = 70000 ms` (`70 s`) before a hung bypass fails out (`14×` NFR14's `5.0 s` ceiling, explicitly disabled via `#if !OBS_BYPASS_SLOW_BRINGUP` in `timing.h:237`).
  - `task_obs_scan.c:69-100` runs a stationary `3°` sweep (`51..129°`, `OBS_SWEEP_ECHO_TIMEOUT_MS = 8U`, `4` pings/bearing, `OBS_SWEEP_MIN_HITS = 2U`, `400 mm` cap), whereas `docs/trib2/requirements.md` FR13/NFR15 and `software_block_diagram.svg:84` still specify a `5°` sweep (`50°–130°`, `12 ms` timeout, `3`-echo median). Reconcile the spec and restore `OBS_BYPASS_SLOW_BRINGUP = 0U` for final NFR14 fault-timeout runs.

---

## #16 [KNOCK-ON CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — All Buddies)  
**Affected Requirement / Component:** All Test Suites / `testing/subsystem1..5/TEST_LOG.md`, `testing/integration/TEST_LOG.md`, `testing/system/TEST_LOG.md`, `tests/host/`  
**Area:** Verification / testing  

**What needs to change?:**
Add a dedicated Fault-Injection & Invalid-Data Verification Suite covering host unit tests, per-subsystem `APP_TEST` fault checks, and integrated failure scenarios for every change in `#1–#15`, and re-run/close `S1-M03`.

**Decision:** Accept  

**Why?:**
Current test suites verify nominal driving and IMU lift/I2C-unplug (`S4-U14..U23`), leaving stall at `5 cm/s`, dual-zero encoder behaviour, IR rail faults, ultrasonic stuck-HIGH/no-echo at runtime, IMU zero-vector and 10 s non-flat boot fallback, VC fail-closed/gated-reset checks, HardFault braking + WDT reset, and `S1-M03` Wi-Fi reconnect either untested or currently marked **FAIL**.

**Engineering Evidence:**
- **Existing gaps observed directly in test logs:**
  - `testing/subsystem1/TEST_LOG.md` row 69: `S1-M03` is currently **FAIL** (`ERR_MEM` + `DEGRADED <-> CONNECTED` flapping); no test verifies `on_subscribed()` fail or IF-12 snapshot timeout expiry.
  - `testing/subsystem2/TEST_LOG.md`: `S2-U01..U28` all PASS for nominal motion and `FAULT_LATCHED`, but no test injects a wheel stall at `5 cm/s`, verifies zero-encoder reports `speed == 0` instead of fabricated velocity, or tests `!subsystem*_sensors_ok()` / active-fault `RESET` rejection.
  - `testing/subsystem3/TEST_LOG.md`: No test verifies `EVF_SAFETY_FAULT_IR_SENSOR` latching on ADC rail short (`<= 16` or `>= 4079`) in edge (`lf_core`) and straddle (`lf_straddle`) builds.
  - `testing/subsystem4/TEST_LOG.md`: No test injects `(0,0,0)` accelerometer samples, verifies 10 s non-flat boot refusal, or verifies `EVF_SAFETY_FAULT_IMU_WDOG` clears when heartbeat returns.
  - `testing/subsystem5/TEST_LOG.md`: Row 104 records the open `LF_ST_SUSPENDED` deadlock; no test verifies runtime echo-stuck-HIGH fault latching or `OBS_AV_SLOW` `no_echo` safety hold.
- **Required Fault-Injection Test Additions:**
  - **`S1-M03` (re-run PASS) + `S1-F01` / `S1-F02`:** 20 s AP drop recovery (`MEM_SIZE = 8192`), IF-12 `valid -> 0` after producer freeze, subscribe failure -> reconnect, urgent `STOP` under `NET_RX_SLOTS` overflow.
  - **`S2-F01` / `S2-F02` / `S2-F03`:** Stall / single-encoder unplug at `5 cm/s` latches within `500 ms` with `speed = 0`; VC rejects `START` while any self-test fails and rejects `RESET` while fault source bits remain `1`; HardFault/hang brakes motors immediately and WDT resets within `500 ms`.
  - **`S3-F01`:** IR sensor rail short (`0` or `4095`) sets `EVF_SAFETY_FAULT_IR_SENSOR` + latches VC in both edge and straddle builds.
  - **`S4-F01`:** Zero-vector `(0,0,0)` rejected from `IMU_ALIVE` & trips watchdog in `50 ms`; 10 s non-flat boot inhibits Motion hump boost and flags uncalibrated; `FAULT_IMU_WDOG` clears on reconnect while `FAULT_LATCHED` stays `1`.
  - **`S5-F01` + `INT-F01`:** Ultrasonic echo-HIGH fault halts avoidance (`OBS_AV_BLOCKED`); `OBS_AV_SLOW` fine-sweep `no_echo` holds safely instead of charging obstacle; `abort_manoeuvre()` exits cleanly without `LF_ST_SUSPENDED` deadlock.

---

## #17 [CHECKED, NO IMPACT]
**Buddy:** Buddy 4 (Vincent — Subsystem 4); Secondary: Buddy 2 (Ruo Chen — Subsystem 2)  
**Affected Requirement / Component:** FR12, NFR10, IF-11 / `app_program/subsystem2/drv_motor.c`, `app_program/subsystem4/task_safety.c`  
**Area:** Safety / fault handling; Hardware / software interaction; Shared data / concurrency / interrupts  

**What needs to change?:**
Evaluate whether the `IF-11` emergency brake path (`motion_emergency_brake()` -> `motor_brake_all()`) needs redesign under the new fault requirement.

**Decision:** No change  

**Why?:**
The `IF-11` hardware brake path is already atomic, interrupt-safe, independent of `MTX_MOTION`, works even before PWM slices are initialized, and stops the robot well inside the `200 ms` NFR10 bound. It already provides an immediate, predictable failsafe actuator shutdown for all Class A faults.

**Engineering Evidence:**
- `app_program/subsystem2/drv_motor.c:233-254`: `motor_brake_all()` executes inside `save_and_disable_interrupts()` / `restore_interrupts()`, atomically writing `pwm_set_both_levels(slice, 0U, 0U)` (ROBO PICO LOW-LOW active brake) when `s_ready`, or configuring `GP8–GP11` as GPIO outputs `LOW` when `!s_ready`.
- `app_program/subsystem4/task_safety.c:18-19, 48-53`: Safety (`TASK_PRI_SAFETY = 1`) brakes strictly via `motion_emergency_brake()` and never touches `MTX_MOTION`.
- Verified in `testing/subsystem2/TEST_LOG.md` (`S2-U05` wheels stopped immediately `dL=0, dR=0, braked=1`; `S2-U06` all 4 pins LOW before Motion task start) and `testing/subsystem4/TEST_LOG.md` (`S4-U18`, `S4-U20`, and `S4-U23`: moving wheels stopped **`134 ms`** after lift event vs `200 ms` NFR10 limit = `66 ms` / 33% margin).

---

## #18 [CHECKED, NO IMPACT]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Motion Control)  
**Affected Requirement / Component:** FR6, NFR2, IF-02 / `app_program/subsystem2/task_motion.c`, `src/common/data_repr.h`  
**Area:** Safety / fault handling; Interfaces / messages; Timing / performance  

**What needs to change?:**
Evaluate Motion Control's command hold timeout (`MOTION_T_HOLD_MS`), manoeuvre duration cap (`MOTION_MANOEUVRE_MAX_MS`), hard speed cap (`MOTION_SPEED_MAX_CMPS`), and `drep_mc_valid()` command rejection logic.

**Decision:** No change  

**Why?:**
Motion Control already guarantees that stale/lost upstream command producers (`VC`/`LF`), corrupted `MBF_MOTION_CMD` frames, or stalled bounded manoeuvres cannot keep motors spinning open-endedly or exceed the `50 cm/s` speed ceiling.

**Engineering Evidence:**
- `config/timing.h:78` & `task_motion.c:367-384`: Unbounded commands (`DREP_MC_FWD`/`BACK` with `arg = 0`, `STEER`) brake automatically via `stop_internal(false)` after `MOTION_T_HOLD_MS = 100 ms` (`5 × 20 ms` periods) without a refresh.
- `task_motion.c:76, 375-383`: Bounded manoeuvres (`TURN_L`, `TURN_R`, `UTURN`, `FWD`/`BACK` `arg > 0`) enforce a `MOTION_MANOEUVRE_MAX_MS = 15000 ms` hard stop.
- `src/common/data_repr.h:272, 334-355` & `task_motion.c:318-334`: `drep_mc_valid()` rejects stray bits (`> 19 b`), speeds `> DREP_MC_SPEED_MAX = 50 cm/s` (NFR2), turn angles `> 180°`, and non-zero args on `STOP`/`UTURN`/`BRAKE`, incrementing `s_mot.reject_count`.
- Verified in `testing/subsystem2/TEST_LOG.md`: `S2-U13` (invalid `motion_cmd_t` rejected), `S2-U14` (`100 ms` hold timeout -> `0`), `S2-U15` (`50 cm/s` cap), and `S2-U22` (VC invalid `nav_input_t` rejected via `drep_nav_valid()`).

---

## #19 [CHECKED, NO IMPACT]
**Buddy:** Buddy 4 (Vincent — Subsystem 4: IMU & Safety)  
**Affected Requirement / Component:** FR12, NFR10, NFR13 / `app_program/subsystem4/task_safety.c`, `app_program/subsystem4/imu_lift.c`  
**Area:** Safety / fault handling; Timing / performance  

**What needs to change?:**
Evaluate the core `50 ms` IMU Safety watchdog loop and the `30°` / `0.10 m/s` lift-and-overturn emergency brake latch.

**Decision:** No change  

**Why?:**
The core watchdog and lift/overturn detection architecture already fails closed on IMU loss (`50 ms`), RTOS event-flag wait errors (`er < E_OK` calls `motion_emergency_brake()`), and physical handling/overturn (`50 ms` hold). Its edge-case gaps (all-zero IMU data, 10 s boot fallback, and clearing `FAULT_IMU_WDOG` on recovery) are cleanly handled by `#5` and `#11` without redesigning `task_safety.c`'s core wait loop or `imu_lift.c` thresholds.

**Engineering Evidence:**
- `app_program/subsystem4/task_safety.c:84-114`: `tk_wai_flg(EVF_SAFETY, EVF_SAFETY_WAIT_MASK, (TWF_ORW | TWF_BITCLR), &flg_pattern, (TMO)SAFETY_IMU_WDOG_MS)` (`50 ms`, `config/timing.h:87`) triggers `brake_and_latch()` on both `E_TMOUT` (`FAULT_IMU_WDOG`) and `EVF_SAFETY_LIFTED`, and explicitly fails closed on `er < E_OK` (`lines 101-108`: calls `motion_emergency_brake()`).
- `config/thresholds.h:34-41`: `IMU_LIFT_TILT_DEG = 30U` (`IMU_LIFT_COS_TILT_X1000 = 866U`), `IMU_LIFT_V_MMPS = 100U`.
- Verified in `testing/subsystem4/TEST_LOG.md`: `S4-U14` & `S4-U16` (`50 ms` watchdog trip on I2C unplug and I2C recovery), `S4-U18..U21` (`63/63` host checks + target level lift & roll-over detection), and `S4-U23` (`134 ms` moving-car lift brake `< 200 ms`).

---

## #20 [CHECKED, NO IMPACT]
**Buddy:** Buddy 3 (Jin Yong — Subsystem 3: Barcode Decode)  
**Affected Requirement / Component:** FR8, FR9, NFR8, NFR9 / `app_program/subsystem3/bc_edge.c`, `bc_frame.c`, `bc_decode.c`  
**Area:** Safety / fault handling; Hardware / software interaction  

**What needs to change?:**
Evaluate Code 39 barcode edge glitch filtering, frame boundary validation, and invalid/ambiguous symbol rejection.

**Decision:** No change  

**Why?:**
Barcode decoding already enforces strict syntactic and physical validation on every capture — rejecting optical chatter, overflows, stalls (`< 20 mm/s`), partial scans, missing `*` sentinels, and ambiguous bar/space ratios with zero misreads across host and car tests (once Motion's fake-speed fallback is removed in `#1`).

**Engineering Evidence:**
- `config/timing.h:182, 188, 194`: `BC_GLITCH_US = 2000U` filters hardware comparator chatter (`20–800 µs` measured) below the narrowest physical bar (`3 mm` at `511 mm/s` = `5.9 ms`); `BC_EDGE_RING_LEN = 64U` (`> 2 × 29`-element symbol); `BC_CAPTURE_TIMEOUT_MS = 1000U`.
- `config/thresholds.h:144-151`: `BC_WIDE_RATIO_X100 = 150U` classifies bars and spaces separately against their medians; `BC_MIN_SPEED_MMPS = 20U` rejects stalled reads (`BC_ERR_STALL`).
- Verified in `testing/subsystem3/TEST_LOG.md`: `S3-U01` (`1980 checks, 0 failed`), `S3-U02` (stall/overflow/reversed/partial rejections), `S3-U04` & `S3-U06` (`38` correct, `3` refused, `0` misread on car), and `S3-U07` (unfiltered flicker fails, filtered decodes `100%`).

---

## #21 [CHECKED, NO IMPACT]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry)  
**Affected Requirement / Component:** FR3, NFR1, IF-04 / `app_program/subsystem1/task_tel.c`, `tel_outbox.c`, `config/priorities.h`  
**Area:** Tasks / components; Timing / performance; Safety / fault handling  

**What needs to change?:**
Evaluate whether Telemetry's queue design should be changed or whether `EVF_SAFETY_FAULT_WIFI_LOSS` should halt vehicle navigation when Wi-Fi/MQTT fails.

**Decision:** No change  

**Why?:**
Continuing autonomous operation while Wi-Fi/MQTT is unavailable is the exact intended degraded behaviour under `FR3` and `NFR1`: local vehicle control (`P1–P8`) never blocks on networking (`P9`), local safety stops (`FR12` physical lift-to-stop, `FR13` obstacle stop, `FR7` track-lost stop) remain fully active, and events buffer safely in the 16-slot drop-oldest outbox for recovery (`#6`).

**Engineering Evidence:**
- `app_program/subsystem1/task_tel.c:85-93, 565-585`: `tel_post()` sends to `MBF_TELEMETRY` (`32 × 8 B`) strictly with `TMO_POL` (non-blocking) and counts `s_evt_post_drops`. `update_wifi_loss_bit()` sets `EVF_SAFETY_FAULT_WIFI_LOSS` (`b8`) and queues `DREP_TEL_FAULT` into `s_outbox` without latching `FAULT_LATCHED`.
- `config/priorities.h:24-40`: `TASK_PRI_TEL = 9`, strictly below every control and safety task (`P1–P8`).
- Verified in `testing/subsystem1/TEST_LOG.md`: `S1-U04` (`64` non-blocking posts in `166 µs`, `32` queued + `32` dropped), `S1-U10` (`50` wakes/s, `0` overruns), and `S1-H02` (`outbox` FIFO/drop-oldest bounds).

---

## #22 [CHECKED, NO IMPACT]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform)  
**Affected Requirement / Component:** NFR6, NFR10, NFR13 / `config/priorities.h`, `src/common/ipc.c`  
**Area:** Shared data / concurrency / interrupts; Timing / performance  

**What needs to change?:**
Evaluate whether the RTOS priority assignment or `MTX_MOTION` synchronization model must change to accommodate the new fault checks and emergency-stop responses.

**Decision:** No change  

**Why?:**
The fixed preemptive priority hierarchy (`P1` Safety > `P2` VC > `P3` Motion > `P4` LF > `P5` BC > `P6` ObsScan > `P7` ObsAvoid > `P8` IMU > `P9` Tel > `P10` DbgDrain) paired with priority inheritance (`TA_INHERIT`) on `MTX_MOTION` and Safety never acquiring `MTX_MOTION` already guarantees bounded latency for fault preemption and state access.

**Engineering Evidence:**
- `config/priorities.h:24-45`: Safety (`1`), VC (`2`), and Motion (`3`) outrank all sensor, obstacle, IMU (`8`), telemetry (`9`), and debug drain (`10`) tasks.
- `src/common/ipc.c:176`: `MTX_MOTION` is created with `TA_INHERIT`, preventing priority inversion when lower-priority tasks (`LF` P4, `Obs` P6/P7, `IMU` P8, `Tel` P9) copy `16 B` `g_motion_state`.
- Verified in `testing/system/TEST_LOG.md` `SYS-01`: Across a `798 s` run including `60 s` USB console stress (`633` console drops inside stress), worst-case cyclic period jitter remained **`10 µs` (`safety`) to `566 µs` (`tel`)** against a `1000 µs` deadline limit (`0` missed heartbeats across all tasks).
