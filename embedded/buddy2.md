# Tribunal 2 Impact Assessment — Fault Handling & Invalid-Data Resilience

**Author:** Buddy 2 — Lee Ruo Chen (`2500530`), Owner of Subsystem 2 (Motion Control & Vehicle Controller)  
**Target Platform:** Raspberry Pi Pico W (`RP2040`), micro T-Kernel 3.0 RTOS (`KERNEL_TICK_MS = 1`), C11 (Barr-C), Pico SDK 1.5.1  
**Baseline Reviewed:** Active `main` across Subsystems 1–5 (`app_program/subsystem1..5/`), shared files (`config/*.h`, `src/common/*`, `src/rtos/port/*`, `app_program/app_main.c`, `app_program/app.h`), and `docs/trib2/requirements.md` / `software_block_diagram.svg`

---

## 1. Requirement Change Under Review

> **New Requirement:** If any part of the system fails or provides invalid data, the system must respond in a safe and predictable way. Instead of continuing as though an unavailable/invalid value were valid, the system must define a predictable response (limited operation, stopping the affected function, attempting recovery, or requiring operator intervention).

---

## 2. Executive Summary & Fault-Response Policy

Core failsafes (`IF-11` emergency brake at `134 ms < 200 ms`, P1 Safety 50 ms IMU watchdog, Code 39 validation, and non-blocking P9 Telemetry) already hold, but **eight `[OBVIOUS CHANGE]` gaps** still treat dead or invalid data as valid (`#1–#8`), alongside five `[HIDDEN RISK]` items (`#9, #11, #12, #13, #15`) and three `[KNOCK-ON CHANGE]` updates (`#10, #14, #16`). Six existing mechanisms were checked and require no change (`#17–#22`).

| Response Class | Triggers | Required System Action | Recovery / Resumption Rule |
|---|---|---|---|
| **Class A — Brake & Latch** | `LIFTED`, `IMU_WDOG`, IMU zero-vector, `STALL`/`ENCODER`, `IR_SENSOR`, Tier-1 boot fail, HardFault/WDT | `motor_brake_all()`, set `EVF_SAFETY_FAULT_LATCHED` + source bit, VC `SAFETY_FAULT_LATCHED`, emit `DREP_TEL_FAULT` | Stationary `REMOTE_RESET` (`500 ms` zero speed) only when active fault sources clear (`READY_WAITING`) |
| **Class B — Stop Function / Refuse Start** | `!subsystem5_sensors_ok()`, IR cal fail, `OBS_AV_SLOW` post-hit dropout, echo stuck HIGH | Refuse `REMOTE_START` or halt avoidance (`OBS_AV_BLOCKED` / `DREP_BYPASS_STOP`) | Resumes only once hardware health recovers and command is re-issued |
| **Class C — Degraded Operation** | `WIFI_LOSS`, transient IMU `1..4` errors before 50 ms WDT, stale IF-12 snapshots | Keep autonomous drive active (telemetry buffered); zero IMU hump boost immediately; clear stale `*_valid` bits | Automatic seamless return once fresh valid readings resume |
| **Class D — Auto-Recover Without Auto-Move** | `imu_reconfigure()`, Wi-Fi/MQTT reconnect | Verify plausible sensor data (`~1 g`) / subscribe `car/cmd/#` and clear health bit (`FAULT_IMU_I2C`, `FAULT_IMU_WDOG`, `WIFI_LOSS`) | Never auto-clears `EVF_SAFETY_FAULT_LATCHED` (`#12`) |

| Tag | Item IDs | Count | Decisions Breakdown |
|---|---|---|---|
| `[OBVIOUS CHANGE]` | `#1` – `#8` | 8 | 5 `Accept`, 3 `Modify` |
| `[HIDDEN RISK]` | `#9`, `#11`, `#12`, `#13`, `#15` | 5 | 3 `Accept`, 1 `Modify`, 1 `Reject` |
| `[KNOCK-ON CHANGE]` | `#10`, `#14`, `#16` | 3 | 2 `Accept`, 1 `Modify` |
| `[CHECKED, NO IMPACT]` | `#17` – `#22` | 6 | 6 `No change` |
| **Total** | **`#1` – `#22`** | **22** | **10 `Accept`, 5 `Modify`, 1 `Reject`, 6 `No change`** |

---

## 3. Coverage Across All 11 Required Review Areas

| Review Area | Covered By Item(s) | Coverage Summary |
|---|---|---|
| **Requirements** | `#14`, `#15`, `#18`, `#21` | Fault-Response Matrix in `docs/trib2/requirements.md`; NFR14 (`60 s` vs `5.0 s`) & 3°/5° sweep sync |
| **Tasks / components** | `#1`, `#3`, `#4`, `#8`, `#21` | `task_motion`, `task_vc`, `task_lf`, `task_obs_*`, `task_imu`, tiered `usermain()`, P9 isolation |
| **State machines** | `#2`, `#3`, `#4`, `#6`, `#10`, `#12` | VC self-test/reset gates; `FAULT_IR_SENSOR`; `OBS_AV_SLOW` hold; `tel_link` flapping; `LF_ST_SUSPENDED` exit |
| **Interfaces / messages** | `#4`, `#7`, `#10`, `#13`, `#14` | `EVF_SAFETY` `10->12` bits in v1 frames; IF-12 expiry; `tick_ms` freshness; `task_bc` 5 ms send |
| **Shared data / concurrency / interrupts** | `#5`, `#7`, `#11`, `#13`, `#17`, `#22` | `s_vc.state_bits` dispatch guard; drop shadow caches; clear `IMU_WDOG`; `STOP` priority; `TA_INHERIT` |
| **Timing / performance** | `#1`, `#9`, `#14`, `#15`, `#18`, `#19`, `#22` | 500 ms low-speed stall & HW WDT; 100 ms hold; `134 ms < 200 ms` NFR10; `10–566 µs` `SYS-01` jitter |
| **Memory / resources** | `#6`, `#14` | lwIP `MEM_SIZE` `4096->8192 B` (`> 100 KB` free SRAM); `12 B` timestamps; `620–4216 B` free stacks |
| **Safety / fault handling** | `#1`–`#5`, `#7`–`#9`, `#11`–`#13`, `#17`–`#21` | Stall/encoder/IR/US/IMU/boot/HardFault latching and fail-closed checks across S1–S5 |
| **Hardware / software interaction** | `#1`, `#3`, `#4`, `#5`, `#9`, `#15`, `#17`, `#20` | PIO0 encoders, ADC rails, HC-SR04 echo HIGH divider check, LSM303D bias/0 g, PWM + WDT, invert macros |
| **Recovery and safe resumption** | `#2`, `#6`, `#8`, `#10`, `#11`, `#12` | Clean S1 reconnect (`S1-M03`), IMU reconfig + `IMU_WDOG` clear, LF unsuspend, manual `RESET` preservation |
| **Verification / testing** | `#16` | Fault-injection suite (`S1-F01..F02`, `S2-F01..F03`, `S3-F01`, `S4-F01`, `S5-F01`, `INT-F01`) + `S1-M03` |

---

## 4. Detailed Impact Assessment

---

## #1 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Motion Control)  
**Affected Requirement / Component:** FR5, FR6, NFR2, NFR6 / `app_program/subsystem2/task_motion.c`, `drv_encoder.c`  
**Area:** Safety / fault handling; Tasks / components; Timing / performance; Hardware / software interaction  

**What needs to change?:**
Make `EVF_SAFETY_FAULT_STALL` / `FAULT_ENCODER` brake and latch at all driving speeds (`>= 5 cm/s`), remove the production fallback that fabricates speed/distance when both encoders read zero, and report PIO encoder init/poll failures.

**Decision:** Modify  

**Why?:**
Currently `update_fault_flags()` cannot even trigger below `18 cm/s`, never stops motors on a fault, and `task_motion.c` fabricates speed/distance when `dl == 0 && dr == 0`, corrupting Barcode (`task_bc.c`), VC reset (`task_vc.c`), and telemetry. Meanwhile a single dead encoder halves `mean_counts` (`2×` over-travel) and saturates `sync_trim` to `±120 pm` into a spin.

**Engineering Evidence:**
- `task_motion.c:450-495`: sets `EVF_SAFETY_FAULT_STALL` (`b3`) / `FAULT_ENCODER` (`b4`) after `MOTION_STALL_PERIODS = 25U` (`500 ms`, line 106) without calling `stop_internal(false)`, setting `FAULT_LATCHED`, or calling `tel_post_fault()`.
- Stall math at `5 / 10 / 15 cm/s` (`ENC_QUAD_MULT=4`, `ENC_CPR=616` -> `2464` cnt/rev, `WHEEL_CIRC_MM=207`, `20 ms`, `Kp=3`, `Ki_x100=10`, `I_max=400`, `MOTOR_DUTY_MAX_PCT=95`): at `5 cm/s`, `base_pm = 95`, `mag_counts = 11`, max stalled duty = `95 + 3×11 + 40 = 168 pm` (`248 pm` if `INTEGRAL_MAX=1200`); at `10 cm/s`, `190 + 69 + 40 = 299 pm`; at `15 cm/s`, `285 + 105 + 40 = 430 pm` — all `< MOTION_STALL_DUTY_PM = 500` (`task_motion.c:154, 453-454`).
- `task_motion.c:388-431, 720-732`: with `dl=0, dr=11`, `heading_diff` saturates `sync_trim = ±120 pm` in `120 ms` while `mean_counts = (|dl|+|dr|)/2` doubles distance/turns up to `MOTION_MANOEUVRE_MAX_MS = 15000 ms` (line 76).
- `task_motion.c:772-780`: when `(dl == 0) && (dr == 0)`, sets `*p_speed_x10 = ±(setpoint_cmps * 10)` and integrates fake `distance_cm`.
- `drv_encoder.c:120-123, 149-189`: `enc_init()` returns `void` with `s_ready = 0` on PIO error; `enc_get_count()` silently returns `0` when `!s_ready` or stale `s_last_raw` after `ENC_FRESH_POLL_MAX` (`32`).

**Modified Decision:**
Keep the `dl == 0 && dr == 0` bench fallback only under `#if APP_TEST` (report `0` speed/distance in production), and arm per-wheel stall/encoder detection whenever `MOT_FLG_CMD_ACTIVE != 0`, target `>= 3` cnt/period, and `|duty_w| >= 80 pm` for `500 ms`. On trigger, brake (`stop_internal(false)`), latch `EVF_SAFETY_FAULT_LATCHED | FAULT_STALL/ENCODER`, emit `DREP_TEL_FAULT`, and expose `enc_stale_count()` / `enc_is_ready()` for boot self-test.

**Alternative considered:** Degraded single-encoder open-loop driving on `dl == 0` XOR `dr == 0`. Rejected because `sync_trim` cannot compensate differential drift without both wheels, risking off-track veering; braking and latching is safer.

---

## #2 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Vehicle Controller)  
**Affected Requirement / Component:** FR4, FR12 / `app_program/subsystem2/task_vc.c`, `subsystem2.h`  
**Area:** State machines; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Gate `vc_complete_selftest()` on S2–S5 hardware checks, make `is_fault_latched()` and `update_stationary_tracker()` fail closed on RTOS errors, block `REMOTE_START` from bypassing `INITIALISING`, and block `RESET` while fault source bits remain active.

**Decision:** Accept  

**Why?:**
Currently `task_vc.c` promotes itself to `READY_WAITING` without checking any peripheral driver or waiting for IMU startup, treats failed `tk_ref_flg()` / `motion_get_state()` calls as safe/stationary, and lets `START` override `INITIALISING` or `RESET` clear `FAULT_LATCHED` while hardware is still faulted.

**Engineering Evidence:**
- `task_vc.c:565-566`: calls `vc_complete_selftest(!is_fault_latched())` at P2 before P8 IMU/Safety boot settle without checking `subsystem2_pre_start_brake_ok()`, `motor_is_ready()`, `enc_is_ready()`, `subsystem3_sensors_ok()`, `subsystem4_imu_found()` (`subsystem4.h:49`), or `subsystem5_sensors_ok()` (`subsystem5.h:48`).
- `task_vc.c:195-204` & `task_motion.c:883-887`: `is_fault_latched()` returns `false` if `tk_ref_flg(EVF_SAFETY, &rflg) != E_OK`, and Motion only stops if `tk_ref_flg == E_OK` — both fail open on RTOS/IPC errors.
- `task_vc.c:158-184`: `speed_x10` defaults to `0` if `motion_get_state(&ms) != E_OK`, arming `SAFETY_T_STILL_MS = 500 ms` (`timing.h:89`) on a mutex error even while moving.
- `task_vc.c:217-225, 251-254`: `REMOTE_START` forces `INITIALISING -> READY_WAITING -> NAV_LINE_TRACKING`, and `REMOTE_RESET` clears `FAULT_LATCHED` without checking active fault bits (`IMU_WDOG`/`IMU_I2C`/`STALL`/`ENCODER`/`IR_SENSOR`/`US_SENSOR`/`BOOT_INIT`), leaving a `50 ms` unlatched window (`S1-M02`).

---

## #3 [OBVIOUS CHANGE]
**Buddy:** Buddy 3 (Jin Yong — Subsystem 3: Line Following)  
**Affected Requirement / Component:** FR7, FR8, NFR7 / `app_program/subsystem3/subsystem3.c`, `task_lf.c`, `lf_core.c`, `lf_straddle.c`, `drv_ir.c`  
**Area:** Safety / fault handling; Tasks / components; State machines; Hardware / software interaction  

**What needs to change?:**
Set `EVF_SAFETY_FAULT_IR_SENSOR` (`b9`) on boot ADC sanity failures, calibration refusal, and runtime rail/saturation faults in both `lf_straddle.c` and `lf_core.c`, expose `subsystem3_sensors_ok()`, and handle `drv_ir_read()` / `motion_get_state()` failures explicitly.

**Decision:** Accept  

**Why?:**
`EVF_SAFETY_FAULT_IR_SENSOR` is allocated in IPC and shown in the block diagram but never set anywhere in Subsystem 3. Bad IR readings are ignored at boot, normalized to `1000 pm` dark in edge mode (`lf_core.c`) to trigger false junctions, or downgraded to `TRACK_LOST` (`READY_WAITING`) in straddle mode so restarts remain allowed on broken sensors.

**Engineering Evidence:**
- `ipc_types.h:93` & `subsystem3.c:31-67`: `EVF_SAFETY_FAULT_IR_SENSOR` (`1UL << 9`) has zero references in `subsystem3/`; `s3_ir_boot_check()` logs `drv_ir_sanity()` (`ADC_STUCK_LOW_RAW=16`, `ADC_STUCK_HIGH_RAW=4079`, `ADC_NOISE_MAX_RAW=400`, `thresholds.h:96-102`) without acting on faults (`subsystem3_init()` always returns `E_OK`).
- `task_lf.c:218-236`: `LF_CAL_BAD_SENSOR` / `LF_CAL_NO_CONTRAST` never raises `FAULT_IR_SENSOR` or `DREP_TEL_FAULT`.
- `lf_core.c` (default build when `S3_LF_STRADDLE=OFF`, `sources.cmake:103`): has zero runtime `sd_rail()` or `LF_SD_OVER_RAW` checks; raw `>= 4079` maps to dark (`testing/subsystem3/TEST_LOG.md` row 66).
- `lf_straddle.c:516-519, 592-599, 642-647`: `sd_rail()` held for `LF_SD_FAULT_N = 40` (`200 ms`, `lf_straddle.h:253`) only emits `LF_EV_TRACK_LOST` (`READY_WAITING`) without setting `FAULT_IR_SENSOR`.
- `drv_ir.c:36-47` & `task_lf.c:255-258`: `drv_ir_read()` returns `0U` when `!s_ready` (indistinguishable from stuck-low), and `task_lf.c` keeps stale `s_ms` on `motion_get_state() != E_OK`.

---

## #4 [OBVIOUS CHANGE]
**Buddy:** Buddy 5 (Chay Han — Subsystem 5: Obstacle Scanning & Avoidance)  
**Affected Requirement / Component:** FR13, FR14, NFR14, NFR15 / `app_program/subsystem5/subsystem5.c`, `task_obs_scan.c`, `task_obs_avoid.c`, `drv_ultrasonic.c`  
**Area:** Safety / fault handling; Tasks / components; Hardware / software interaction; Interfaces / messages  

**What needs to change?:**
Distinguish deterministic ultrasonic/servo faults (pre-trigger stuck-HIGH echo, failed `subsystem5_sensors_ok()`) from open-space timeouts (`STATUS_NO_ECHO`), stop `OBS_AV_SLOW` from resuming straight into an obstacle on fine-scan `no_echo`, and abort safely to `OBS_AV_BLOCKED` on `motion_get_state()` errors.

**Decision:** Modify  

**Why?:**
Currently an unplugged or unpowered HC-SR04 returns `0 mm` (`STATUS_NO_ECHO`), which is non-faulting (`ipc_types.h:97,132`) and indistinguishable from an open track, letting the car drive full speed into obstacles. Additionally, a mid-scan dropout in `OBS_AV_SLOW` immediately resumes straight cruise toward the confirmed obstacle, while mutex failures fall back to multi-second blind timeouts.

**Engineering Evidence:**
- `subsystem5.c:47-91` & `drv_ultrasonic.c:127-130`: `subsystem5_init()` logs `[S5] sensor check FAILED` when `servo` or `!us_echo_pin_is_high()` fails, yet returns `E_OK` and nothing checks `subsystem5_sensors_ok()`.
- `task_obs_scan.c:946-966, 988`: `EVF_SAFETY_STATUS_NO_ECHO` (`b7`) is set after `5` zero pings (`300 ms`) while `watch_step()` treats `mm == 0U` as clear forever; `drv_ultrasonic.c:89` also collapses `width > US_ECHO_MAX_US` to `0U`.
- `task_obs_scan.c:1096-1108` & `task_obs_avoid.c:930-936`: if `obs_analyse_fine3()` returns `false`, `finish_fine3_sweep()` sends `no_echo`, causing `on_profile()` in `OBS_AV_SLOW` to call `release(now_ms, false, true)` (`FWD` toward the obstacle!).
- `task_obs_scan.c:1027-1055` & `task_obs_avoid.c:392-422, 508-519`: failed `motion_get_state()` lets `OBS_CONFIRM_MAX_WAIT_MS` (`600 ms`) force standstill acceptance and turns closed-loop legs/turns into `OBS_TURN_MAX_MS` (`4000 ms`) / `OBS_LEG_BACK_MAX_MS` (`10000 ms`) blind runs.

**Modified Decision:**
Sample `us_echo_pin_is_high()` before every ping plus `us_is_ready()`/`servo_is_ready()`: on stuck-HIGH echo or boot failure, latch `EVF_SAFETY_FAULT_US_SENSOR` (`b10`, `#14`), emit `DREP_TEL_FAULT`, refuse `START`, and enter `OBS_AV_BLOCKED`. In `OBS_AV_SLOW`, on fine-scan `no_echo`, require 3 confirmed ahead echoes `> 280 mm` before releasing (else hold `OBS_AV_BLOCKED`), and abort to `OBS_AV_BLOCKED` on `motion_get_state()` failure or stale `tick_ms`.

**Alternative considered:** Latching `EVF_SAFETY_FAULT_LATCHED` whenever `EVF_SAFETY_STATUS_NO_ECHO` triggers after `300 ms`. Rejected because an HC-SR04 pointed at open floor past `OBS_ECHO_TIMEOUT_MS` (`30 ms`) legitimately returns zero echoes (`timing.h:138`), which would falsely halt every clear straightaway.

---

## #5 [OBVIOUS CHANGE]
**Buddy:** Buddy 4 (Vincent — Subsystem 4: IMU Processing); Secondary: Buddy 2 (Motion Control)  
**Affected Requirement / Component:** FR10, FR11, FR12, NFR12, NFR13 / `app_program/subsystem4/task_imu.c`, `imu_lift.c`, `drv_imu.c`, `app_program/subsystem2/task_motion.c`  
**Area:** Safety / fault handling; Hardware / software interaction; Shared data / concurrency / interrupts  

**What needs to change?:**
Gate Motion's IMU hump climb/speed boost on valid, level-calibrated, non-faulted IMU health; flag uncalibrated state instead of silently falling back to `0°` offset after 10 s; and reject all-zero `(0,0,0)` / stuck-rail accelerometer vectors before feeding `EVF_SAFETY_IMU_ALIVE` or `imu_lift_set_up()`.

**Decision:** Accept  

**Why?:**
Motion doubles `5 cm/s` speed to `95 mm/s` (`+18%` duty) from raw pitch without checking IMU health or calibration, while `task_imu.c` falls back to `0°` offset despite a measured `4.0–4.5°` mounting tilt above the `3.0°` hump threshold. Meanwhile `(0,0,0)` reads still kick the 50 ms Safety watchdog and permanently zero `up_mag`, disabling `FR12` lift protection.

**Engineering Evidence:**
- `task_motion.c:162-171, 532-608`: applies up to `+45 mm/s` boost and `+180 pm` duty when `|subsystem4_get_pitch_half_deg()| >= 6` (`3.0°`) without checking `subsystem4_imu_found()`, `imu_is_level_done()`, or `EVF_SAFETY_FAULT_IMU_I2C` (`5` consecutive fails = `40–50 ms` freeze before WDT, `task_imu.c:33`).
- `task_imu.c:89, 612-619` & `S4-U07`: `IMU_LEVEL_TIMEOUT` (`1000` samples = `10 s`) calls `finish_level(0, 0, 0, ±1000)` silently; `S4-U07` measured `8` half-deg (`4.0°`, up to `4.5°`) static bias when flat, exceeding `IMU_HUMP_PITCH_DEG = 3U` (`thresholds.h:22`) and `MOTION_HUMP_PITCH_ON_HD = 6U`.
- `task_imu.c:695-764`, `drv_imu.c:114-122`, `imu_lift.c:75-96`: `imu_read_accel()` accepts `(0,0,0)` (`|a| = 0`), setting `EVF_SAFETY_IMU_ALIVE`; if averaged at boot, `imu_lift_set_up()` sets `up_mag = 0U` (`armed = false`), causing `imu_lift_step()` to return `IMU_LIFT_NONE` forever.
- `task_imu.c:714-734`: clears `FAULT_IMU_I2C` immediately after `imu_reconfigure()` without confirming a `~1 g` (`700–1300 mg`) reading.

---

## #6 [OBVIOUS CHANGE]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry & Remote Link)  
**Affected Requirement / Component:** FR1, FR2, FR3 / `app_program/subsystem1/lwipopts.h`, `tel_link.c`, `drv_wifi.c`, `task_tel.c`  
**Area:** Recovery and safe resumption; Memory / resources; State machines; Interfaces / messages  

**What needs to change?:**
Increase `MEM_SIZE` from `4096 B` to `8192 B` to fix the `S1-M03` reconnect `ERR_MEM` loop, require an actual successful publish to leave `TEL_LINK_DEGRADED`, and fail/reconnect the MQTT session if `mqtt_subscribe(TEL_TOPIC_CMD_FILTER)` fails.

**Decision:** Accept  

**Why?:**
During Wi-Fi drops, closing TCP state exhausts lwIP's 4 KB heap and permanently blocks MQTT reconnection, intermediate non-publishing wakes flap `DEGRADED <-> CONNECTED` every 200 ms instead of tripping the 5 s keep-alive teardown, and failed subscriptions leave `car/status = '1'` online while deaf to `STOP`/`RESET`.

**Engineering Evidence:**
- **Verified `S1-M03` FAIL (`testing/subsystem1/TEST_LOG.md` row 69, `lwipopts.h:30,53,55`, `drv_wifi.c:201,366`):** `mqtt_disconnect(&s_client)` keeps a closing PCB holding `TCP_SND_BUF = 2 × 1460 = 2920 B` inside `MEM_SIZE = 4096 B`, leaving `1176 B < TCP_MSS (1460 B)` so `mqtt_client_connect()` fails with `ERR_MEM (-1)`. Raising `MEM_SIZE` to `8192 B` leaves `5272 B` free (`+4 KB` out of `> 100 KB` unused RP2040 SRAM).
- `tel_link.c:112-126` & `task_tel.c:8-14`: `task_tel.c` wakes every `20 ms` and publishes every `10th` wake (`200 ms`); on the 9 non-publishing wakes `!p_in->publish_failed` is true, so `tel_link.c:113` exits `DEGRADED -> CONNECTED` or resets `entered_ms` (`6 s+` flapping, delaying `5000 ms` `degrade_timeout_ms`).
- `drv_wifi.c:90-97, 108-111`: sets `NET_STAGE_UP` and publishes `car/status = '1'` before `on_subscribed()` succeeds, only logging on `err != ERR_OK`.

---

## #7 [OBVIOUS CHANGE]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry)  
**Affected Requirement / Component:** FR1, IF-12, IF-08 / `app_program/subsystem1/task_tel.c`, `src/common/data_repr.h`  
**Area:** Interfaces / messages; Shared data / concurrency / interrupts; Safety / fault handling  

**What needs to change?:**
Expire IF-12 sensor snapshot validity bits (`ir_valid`, `range_valid`, `att_valid`) when producers stop updating, and reject/flag stale `motion_state_t.tick_ms` samples in `task_tel.c`.

**Decision:** Accept  

**Why?:**
Once each sensor writes one value after boot, `TEL_SNAP_VALID` (`1UL << 31`) stays `1` permanently, causing MQTT frames to keep reporting frozen sensor/motion readings as live data after a task or sensor failure.

**Engineering Evidence:**
- `task_tel.c:40, 64-66, 126-147, 523-534`: `s_snap_ir`, `s_snap_range`, `s_snap_att` latch `TEL_SNAP_VALID` forever and pass it straight to `drep_frame_sensor_valid()` (`data_repr.h:851, 864-871`, bits `[75:73]` of the `10 B` `telemetry_sensor` payload).
- Adding three `32-bit` producer timestamps (`12 B` RAM total) and requiring `(now_ms - snap_ms) <= stale_ms` (`IR: 200 ms`, `Att: 100 ms`, `Range: 500 ms`) keeps the `10 B` wire frame unchanged.
- `task_tel.c:516-519` & `data_repr.h:819-830`: `telemetry_motion` uses `114` of `120` bits (`15 B`), allowing stale `(now_ms - ms.tick_ms) > 200 ms` detection without changing frame size.

---

## #8 [OBVIOUS CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform Boot)  
**Affected Requirement / Component:** FR4, NFR1, NFR10 / `app_program/app_main.c`, `app_program/app.h`  
**Area:** Tasks / components; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Replace fire-and-forget `APP_CHECK()` logging in `usermain()` with tiered boot-failure handling that immediately brakes and halts autonomous startup on IPC/Safety/Motion init errors while allowing Telemetry-less degraded driving or diagnosable fault-latching on peripheral errors.

**Decision:** Modify  

**Why?:**
`APP_CHECK()` only logs negative error codes and continues starting tasks even if `ipc_init()` or `safety`/`motion` failed, leaving the vehicle running without core IPC objects or safety supervision.

**Engineering Evidence:**
- `app.h:114-122` & `app_main.c:91-110, 123-144`: `APP_CHECK()` prints `dbg_put_err()` and ignores return codes across `ipc_init()`, `dbg_drain_start()`, and `subsystem1..5_init/start()`, while `app_start_task()` failures leave individual tasks missing.
- `config/priorities.h:122` & `testing/subsystem4/logs/2026-11-19_1625_s4_unit.txt:245`: `usermain()` runs on the `1 KB` `init` stack (`620 B` free of `1152 B`), permitting `dbg_put_err()`/`dbg_puts()` only.

**Modified Decision:**
On **Tier-1 failures** (`ipc_init`, `subsystem2_*`, `subsystem4_*`), call `motion_emergency_brake()`, latch `EVF_SAFETY_FAULT_LATCHED | BOOT_INIT` (`#14`), skip starting S3/S5, and sleep safely with `dbg_drain` alive; on **Tier-2 failures** (`subsystem3_*`, `subsystem5_*`), latch `BOOT_INIT` so VC holds `FAULT_LATCHED` while S1 reports diagnostics; on **Tier-3 failure** (`subsystem1_*`), log and permit degraded autonomous mode (`NFR1`/`FR3`).

**Alternative considered:** Calling `port_fault_shutdown()` (reboot) on any `APP_CHECK()` failure. Rejected because persistent peripheral or Wi-Fi init faults would trigger an endless `10 ms` reset loop and destroy console diagnostics.

---

## #9 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform)  
**Affected Requirement / Component:** FR12, NFR10 / `src/rtos/port/port_fault.c`, `app_program/subsystem2/drv_motor.c`, `task_safety.c`, `task_motion.c`  
**Area:** Hardware / software interaction; Safety / fault handling; Timing / performance  

**What needs to change?:**
Invoke `motor_brake_all()` on entry to `port_fault_from_handler()` / `port_fault_shutdown()`, and enable the RP2040 hardware watchdog (`watchdog_enable`) kicked by `P1` Safety only while both Safety and Motion counters advance.

**Decision:** Accept  

**Why?:**
RP2040 PWM slices 4–5 (`GP8–GP11`) keep driving their last duty autonomously if the CPU hits a HardFault, breakpoint loop, or kernel deadlock, risking motor runaway given already-observed USB/IRQ31 crashes.

**Engineering Evidence:**
- `port_fault.c:38, 97-111, 134-145`: neither `port_fault_from_handler()` nor `port_fault_shutdown()` calls `motor_brake_all()`, leaving PWM running indefinitely under `PORT_FAULT_HALT=1` (`__breakpoint()` loop) or for `10 ms` (`5 mm` at `50 cm/s`) before `watchdog_reboot()`.
- Codebase never calls `watchdog_enable()`; `testing/subsystem4/TEST_LOG.md` rows 81 & 96 record real IRQ 31 HardFaults and a silent freeze without reset. Enabling a `500 ms` HW WDT fed every `50 ms` by Safety when Motion's `20 ms` `release_count` advances bounds CPU-hang runaway to `<= 500 ms` (`10×` margin).

---

## #10 [KNOCK-ON CHANGE]
**Buddy:** Buddy 5 (Chay Han — Subsystem 5); Secondary: Buddy 3 (Jin Yong — Subsystem 3)  
**Affected Requirement / Component:** FR7, FR14 / `app_program/subsystem5/task_obs_avoid.c`, `app_program/subsystem3/lf_core.c`, `lf_straddle.c`  
**Area:** State machines; Interfaces / messages; Recovery and safe resumption  

**What needs to change?:**
Allow `st_suspended()` (`lf_core.c`) and `sd_st_suspended()` (`lf_straddle.c`) to exit `LF_ST_SUSPENDED` whenever `!avoid_active` (entering `ON_LINE` or `REACQUIRING`), and send `DREP_NAV_AVOID_BLOCKED` on fault aborts in `OBS_AV_BLOCKED`.

**Decision:** Accept  

**Why?:**
Increased safety aborts (`#4`) trigger `abort_manoeuvre()`, which currently clears `AVOID_ACTIVE` without setting `LINE_REACQUIRED`, trapping Line Following in `LF_ST_SUSPENDED` while the car drives blind at cruise.

**Engineering Evidence:**
- `task_obs_avoid.c:567-598`: `abort_manoeuvre()` calls `release(now_ms, false, false)`, clearing `EVF_NAV_AVOID_ACTIVE` (`line 580`) with `EVF_NAV_LINE_REACQUIRED == 0` (`line 571`).
- `lf_straddle.c:495-513` & `lf_core.c:893-911`: `LF_ST_SUSPENDED` exits exclusively on `if (p_in->reacquired && !p_in->avoid_active)`.
- `testing/subsystem5/TEST_LOG.md` row 104 (`2026-11-19_s5_fullchain.txt`): LF stayed stuck in `SUSPENDED` for **340 consecutive log lines** while VC drove `FWD` blind until crashing into the IMU lift latch.

---

## #11 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 4 (Vincent — Subsystem 4), Buddy 1 (Damian — Subsystem 1)  
**Affected Requirement / Component:** FR12, NFR6 / `app_program/subsystem2/task_vc.c`, `task_motion.c`, `app_program/subsystem4/task_safety.c`, `app_program/subsystem1/task_tel.c`  
**Area:** Shared data / concurrency / interrupts; Safety / fault handling; Recovery and safe resumption  

**What needs to change?:**
Guard cross-task `s_vc.state_bits` writes against Cortex-M0+ non-atomic RMW races, replace `s_mot.fault_bits_set` / `s_wifi_loss_set` caches with `tk_ref_flg()` checks, and clear `EVF_SAFETY_FAULT_IMU_WDOG` in `task_safety.c` once `IMU_ALIVE` returns (keeping `FAULT_LATCHED` set).

**Decision:** Modify  

**Why?:**
Unsynchronized `uint16_t` RMWs on `s_vc.state_bits` can drop flags under preemption, local `EVF_SAFETY` shadow variables desynchronize when tests clear flags, and a permanently stuck `FAULT_IMU_WDOG` bit (`b6`) falsifies telemetry and blocks gated `RESET` (`#2`) after IMU recovery.

**Engineering Evidence:**
- `task_vc.c:79, 108, 134, 479, 605`: `vc_init()`, `vc_complete_selftest()`, and `vc_set_authority()` mutate `s_vc.state_bits` via non-atomic ARMv6-M RMW without `tk_dis_dsp()`/`tk_ena_dsp()`.
- `task_motion.c:219, 480-495` & `task_tel.c:58, 569-573` cache prior writes locally, unlike `task_obs_scan.c:747-783` which queries `tk_ref_flg()` specifically to prevent external-clear desync.
- `task_safety.c:91, 116-120` & `task_vc.c:220`: Safety sets `FAULT_IMU_WDOG` (`b6`) on timeout and never clears it when `IMU_ALIVE` resumes (`s_wdog_tripped` only), while VC clears only `~EVF_SAFETY_FAULT_LATCHED` (`ARCHITECTURE.md` D9).

**Modified Decision:**
Wrap cross-task `s_vc.state_bits` mutations in `tk_dis_dsp()`/`tk_ena_dsp()`, replace `s_mot.fault_bits_set` and `s_wifi_loss_set` with `tk_ref_flg()` reads, and clear `~EVF_SAFETY_FAULT_IMU_WDOG` inside `task_safety.c:116-120` when `IMU_ALIVE` resumes while preserving `EVF_SAFETY_FAULT_LATCHED` until manual `RESET`.

**Alternative considered:** Clearing `EVF_SAFETY_FAULT_IMU_WDOG` inside `task_vc.c` on `REMOTE_RESET`. Rejected because it violates `ARCHITECTURE.md` D9 single-writer-per-bit ownership (`task_safety` owns `b6`) and masks a dead IMU for up to `50 ms` after `RESET`.

---

## #12 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 4 (Vincent — Subsystem 4)  
**Affected Requirement / Component:** FR12, `design.md` §5 / `app_program/subsystem2/task_vc.c`, `app_program/subsystem4/task_safety.c`, `task_imu.c`  
**Area:** Recovery and safe resumption; Safety / fault handling; State machines  

**What needs to change?:**
*(Evaluated candidate auto-resume change)* Automatically clear `EVF_SAFETY_FAULT_LATCHED` and resume driving once a failed IMU reconnects (`imu_reconfigure()`) or a wheel stall clears.

**Decision:** Reject  

**Why?:**
Auto-clearing a Class A safety latch to restart motor PWM violates `FR12` and risks injury or runaway if an operator triggers recovery while lifting the car, reseating cables, or un-jamming wheels by hand.

**Engineering Evidence:**
- `docs/trib2/requirements.md` FR12 & `design.md` §5 require explicit stationary (`SAFETY_T_STILL_MS = 500 ms`, `timing.h:89`) `REMOTE_RESET` (`-> READY_WAITING`) plus separate `REMOTE_START` (`task_vc.c:223-224`).
- `task_imu.c:714-734` & `task_safety.c:116-120`: subsystem hardware recovery (`imu_reconfigure()`) is already cleanly decoupled from vehicle motion latching (`EVF_SAFETY_FAULT_LATCHED`).

---

## #13 [HIDDEN RISK]
**Buddy:** Buddy 1 (Damian — Subsystem 1); Secondary: Buddy 2 (Ruo Chen — Subsystem 2), Buddy 3 (Jin Yong — Subsystem 3)  
**Affected Requirement / Component:** FR2, FR9, `TEAM.md` §5 IF-01 / `app_program/subsystem1/drv_wifi.c`, `task_tel.c`, `app_program/subsystem3/task_bc.c`  
**Area:** Interfaces / messages; Shared data / concurrency / interrupts; Safety / fault handling  

**What needs to change?:**
Prevent `REMOTE_STOP` drops in `drv_wifi.c`'s `NET_RX_SLOTS` ring and `task_tel.c`'s `MBF_NAV_IN` send path; switch `task_bc.c` from `TMO_POL` to `NAV_MBF_SEND_TMO_MS` (`5 ms`) with failure counting; and mask `uint8_t` wraparound in `task_bc.c:237`.

**Decision:** Accept  

**Why?:**
Losing `REMOTE_STOP` under queue pressure breaks remote safety intervention (`FR2`), while `task_bc.c` silently drops junction turns on a `0 ms` poll and prints bogus overflow counts after 256 overflows.

**Engineering Evidence:**
- `drv_wifi.c:156-160`: drops incoming packets without priority when `(s_rx_head - s_rx_tail) >= NET_RX_SLOTS` (`4`); fix by latching a sticky urgent `car/cmd/stop` slot.
- `task_tel.c:333-339, 361`: unretried `(void)tel_cmd_receive()` drops `REMOTE_STOP` if `MBF_NAV_IN` times out at `5 ms`; fix by retrying `STOP` every `20 ms` wake (and braking via `IF-11` after 2 failures).
- `task_bc.c:126` vs `task_lf.c:157` & `task_obs_avoid.c:254-255`: uses `TMO_POL` instead of `NAV_MBF_SEND_TMO_MS` (`5U`, `timing.h:201`) and discards errors; `task_bc.c:237` omits `& 0xFFU`, printing `4294967041` when 8-bit `s_ring.overflow` wraps `255 -> 0`.

---

## #14 [KNOCK-ON CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — Shared / All Buddies)  
**Affected Requirement / Component:** All FRs/NFRs / `docs/trib2/requirements.md`, `src/common/ipc_types.h`, `data_repr.h`, `docs/DATA_REPRESENTATION.md`, `tools/dashboard/tel_codec.py`, `subsystem2.h`, `subsystem3.h`  
**Area:** Requirements; Interfaces / messages; Memory / resources; Timing / performance  

**What needs to change?:**
Add the Fault-Response Matrix to `docs/trib2/requirements.md`, extend `EVF_SAFETY` to 12 bits (`b10` `FAULT_US_SENSOR`, `b11` `FAULT_BOOT_INIT`) inside existing v1 frames, expose `subsystem2/3_sensors_ok()`, and enforce `MOTION_STALE_MS` (`100 ms`) via `motion_state_t.tick_ms`.

**Decision:** Modify  

**Why?:**
The new fault detections require two additional `EVF_SAFETY` bits, uniform S2/S3 health getters for VC (`#2`), and timestamp freshness checks on `motion_state_t`, all achievable with zero wire-byte or struct-size growth and well within measured SRAM/stack margins.

**Engineering Evidence:**
- `src/common/data_repr.h:758, 902-920`: `DREP_FT_HEARTBEAT` (`6 B = 48 b`) uses `32 + 4 + 10 = 46 b` (**`2` spare bits** fit `b10`/`b11`, `0 B` growth); `DREP_FT_EVENT_FAULT` (`5 B = 40 b`) uses `34 b` -> `36 b` (`4` spare bits left); `ipc_types.h:68-73, 119` (`16 B`) already includes `tick_ms` (`MOTION_STALE_MS = 100U`).
- SRAM: RP2040 `264 KB` SRAM (`96 KB` pool `priorities.h:124`, `~16 KB` lwIP `lwipopts.h:29`); `+4 KB` `MEM_SIZE` (`#6`) + `12 B` timestamps (`#7`) + `< 64 B` fault state leaves `> 100 KB` margin.
- Worst-case measured free stack (`2026-11-16_s3_u14_uturn_hold_run5.txt:604-614`, `2026-11-19_1625_s4_unit.txt:245-253`): `init` `620 B` (`1152 B`), `vc` `1236 B`, `motion` `1316 B`, `safety` `1292 B`, `lf` `1076 B`, `imu` `1468 B`, `obs_scan` `1076 B`, `obs_avoid` `1052 B`, `tel` `4216 B`, `dbg_drain` `1020 B`.

**Modified Decision:**
Expand `EVF_SAFETY_ALL_MASK` to `0xFFFUL` (`12` bits) in existing `6 B`/`5 B` v1 frames, add `subsystem2/3_sensors_ok()`, validate `(now_ms - ms.tick_ms) <= 100 ms`, and sync `docs/trib2/requirements.md`, `DATA_REPRESENTATION.md`, and `tel_codec.py`.

**Alternative considered:** Widening `DREP_FT_HEARTBEAT` to `7 B` (`16`-bit mask) and `motion_state_t` to `20 B` (`valid` word). Rejected to avoid breaking v1 frame parsers when 2 exact spare heartbeat bits and `tick_ms` already exist.

---

## #15 [HIDDEN RISK]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2); Secondary: Buddy 3 (Jin Yong), Buddy 5 (Chay Han)  
**Affected Requirement / Component:** FR5, FR7, FR13, NFR14, NFR15 / `drv_motor.c`, `drv_encoder.c`, `subsystem3/sources.cmake`, `task_obs_scan.c`, `config/timing.h`, `docs/trib2/requirements.md`  
**Area:** Hardware / software interaction; Timing / performance; Requirements  

**What needs to change?:**
Remove hardcoded `(1U)` invert overrides in `drv_motor.c` / `drv_encoder.c`, enforce IR fault safety (`#3`) across default `S3_LF_STRADDLE=OFF` builds, and reconcile `OBS_BYPASS_SLOW_BRINGUP` (`60 s`) and `3°` vs `5°` sweep parameters with `docs/trib2/requirements.md`.

**Decision:** Accept  

**Why?:**
Dead `#else` preprocessor branches silently ignore `config/pins.h` direction changes (causing PID positive-feedback runaway after hardware swaps), default CMake flags compile the unchecked edge core, and `OBS_BYPASS_SLOW_BRINGUP = 1U` delays a wedged avoidance timeout to `70 s` (`14×` NFR14).

**Engineering Evidence:**
- `drv_motor.c:67-71` & `drv_encoder.c:57-61` vs `config/pins.h:71,80`: both `#if` and `#else` branches hardcode `MOTOR_L_EFF_INVERT = (1U)` and `ENC_R_EFF_INVERT = (1U)`.
- `app_program/subsystem3/sources.cmake:103`: defaults `S3_LF_STRADDLE=OFF`, dropping `sd_rail()` / `LF_SD_OVER_RAW` unless `-DS3_LF_STRADDLE=ON` is passed.
- `timing.h:153,162,167,237`, `task_obs_avoid.c:959-960`, `task_obs_scan.c:69-100`: `OBS_BYPASS_SLOW_BRINGUP=1U` sets `s_budget_deadline_ms = 70000 ms` vs `5.0 s` NFR14, while `task_obs_scan.c` sweeps `3°` (`51..129°`, `8 ms`, `2` hits) vs `docs/trib2/requirements.md` FR13/NFR15 (`5°`, `50°–130°`, `12 ms`).

---

## #16 [KNOCK-ON CHANGE]
**Buddy:** Buddy 2 (Ruo Chen — All Buddies)  
**Affected Requirement / Component:** Test Suites / `testing/subsystem1..5/TEST_LOG.md`, `testing/integration/TEST_LOG.md`, `tests/host/`  
**Area:** Verification / testing  

**What needs to change?:**
Add host and on-target fault-injection tests covering every new failure handler (`S1-F01..F02`, `S2-F01..F03`, `S3-F01`, `S4-F01`, `S5-F01`, `INT-F01`) and re-verify `S1-M03` to `PASS`.

**Decision:** Accept  

**Why?:**
Current logs verify nominal runs and IMU unplug/lift (`S4-U14..U23`), leaving `5 cm/s` stalls, dual-zero encoders, IR rail shorts, stuck-HIGH ultrasonic, IMU zero-vectors, VC gating, HardFault braking, and `S1-M03` reconnect (`FAIL`) unverified.

**Engineering Evidence:**
- Logged test gaps: `testing/subsystem1/TEST_LOG.md` row 69 marks `S1-M03` **FAIL**; `testing/subsystem2/TEST_LOG.md` (`S2-U01..U28`) lacks `5 cm/s` stall / zero-encoder / gated `RESET` tests; `testing/subsystem3/TEST_LOG.md` lacks `FAULT_IR_SENSOR`; `testing/subsystem4/TEST_LOG.md` lacks `(0,0,0)` / 10 s flat-boot refusal / `IMU_WDOG` clear; `testing/subsystem5/TEST_LOG.md` row 104 notes the open `LF_ST_SUSPENDED` bug.
- Required additions: `S1-M03` re-run + `S1-F01/F02` (stale expiry, sub fail, `STOP` priority), `S2-F01..F03` (`5 cm/s` stall, fail-closed VC/RESET, HardFault/WDT), `S3-F01` (IR rail in both cores), `S4-F01` (0 g vector, 10 s timeout, `IMU_WDOG` clear), and `S5-F01`/`INT-F01` (echo-HIGH, `no_echo` hold, abort unsuspend).

---

## #17 [CHECKED, NO IMPACT]
**Buddy:** Buddy 4 (Vincent — Subsystem 4); Secondary: Buddy 2 (Ruo Chen — Subsystem 2)  
**Affected Requirement / Component:** FR12, NFR10, IF-11 / `drv_motor.c`, `task_safety.c`  
**Area:** Safety / fault handling; Hardware / software interaction; Shared data / concurrency / interrupts  

**What needs to change?:**
Evaluate whether `motion_emergency_brake()` -> `motor_brake_all()` (`IF-11`) needs modification.

**Decision:** No change  

**Why?:**
The `IF-11` brake path is already atomic, interrupt-safe, `MTX_MOTION`-free, active even pre-`motor_init()`, and stops moving wheels well inside NFR10 (`200 ms`).

**Engineering Evidence:**
- `drv_motor.c:233-254` & `task_safety.c:18-19, 48-53`: executes inside `save_and_disable_interrupts()`, pulling all 4 ROBO PICO channels (`GP8–GP11`) LOW without touching `MTX_MOTION`.
- Verified in `testing/subsystem2/TEST_LOG.md` (`S2-U05`, `S2-U06`) and `testing/subsystem4/TEST_LOG.md` (`S4-U18`, `S4-U20`, `S4-U23`: **`134 ms`** stop `< 200 ms` NFR10 limit).

---

## #18 [CHECKED, NO IMPACT]
**Buddy:** Buddy 2 (Ruo Chen — Subsystem 2: Motion Control)  
**Affected Requirement / Component:** FR6, NFR2, IF-02 / `task_motion.c`, `src/common/data_repr.h`  
**Area:** Safety / fault handling; Interfaces / messages; Timing / performance  

**What needs to change?:**
Evaluate Motion Control's hold timeout, manoeuvre cap, `50 cm/s` cap, and `drep_mc_valid()` validation.

**Decision:** No change  

**Why?:**
Motion already stops unbounded commands if upstream refreshes stall, limits bounded manoeuvres to `15 s`, enforces the `50 cm/s` NFR2 speed ceiling, and drops invalid IPC commands.

**Engineering Evidence:**
- `timing.h:78` & `task_motion.c:76, 318-334, 367-384`: brakes after `MOTION_T_HOLD_MS = 100 ms` (unbounded) or `MOTION_MANOEUVRE_MAX_MS = 15000 ms` (bounded); enforces `DREP_MC_SPEED_MAX = 50 cm/s` (`data_repr.h:272, 334-355`).
- Verified in `testing/subsystem2/TEST_LOG.md`: `S2-U13` (invalid `motion_cmd_t`), `S2-U14` (`100 ms` hold), `S2-U15` (`50 cm/s` cap), and `S2-U22` (VC `drep_nav_valid()`).

---

## #19 [CHECKED, NO IMPACT]
**Buddy:** Buddy 4 (Vincent — Subsystem 4: IMU & Safety)  
**Affected Requirement / Component:** FR12, NFR10, NFR13 / `task_safety.c`, `imu_lift.c`  
**Area:** Safety / fault handling; Timing / performance  

**What needs to change?:**
Evaluate the core `50 ms` IMU Safety watchdog and `30°` / `0.10 m/s` lift-and-overturn detector.

**Decision:** No change  

**Why?:**
Safety (`P1`) already fails closed on `50 ms` IMU heartbeat loss, RTOS wait errors, or `50 ms` sustained lift/overturn (with zero-vector, boot-level, and `IMU_WDOG` clear gaps fixed separately by `#5` and `#11`).

**Engineering Evidence:**
- `task_safety.c:84-114`: `tk_wai_flg()` (`SAFETY_IMU_WDOG_MS = 50 ms`, `timing.h:87`) calls `brake_and_latch()` on `E_TMOUT` / `LIFTED` and brakes directly on `er < E_OK` (`lines 101-108`).
- `thresholds.h:34-41` & `testing/subsystem4/TEST_LOG.md`: `30°` (`cos=866`) / `100 mm/s` thresholds verified by `S4-U14`, `S4-U16`, `S4-U18..U21` (`63/63`), and `S4-U23` (`134 ms < 200 ms`).

---

## #20 [CHECKED, NO IMPACT]
**Buddy:** Buddy 3 (Jin Yong — Subsystem 3: Barcode Decode)  
**Affected Requirement / Component:** FR8, FR9, NFR8, NFR9 / `bc_edge.c`, `bc_frame.c`, `bc_decode.c`  
**Area:** Safety / fault handling; Hardware / software interaction  

**What needs to change?:**
Evaluate Code 39 glitch filtering, framing bounds, and invalid barcode rejection.

**Decision:** No change  

**Why?:**
The decoder pipeline already rejects optical flicker, ring overflows, low speeds (`< 20 mm/s`), partial scans, missing `*` sentinels, and ambiguous bar/space ratios with zero false commands once `#1` removes fake encoder speeds.

**Engineering Evidence:**
- `timing.h:182,188,194` & `thresholds.h:144-151`: `BC_GLITCH_US = 2000U`, `BC_EDGE_RING_LEN = 64U`, `BC_WIDE_RATIO_X100 = 150U`, and `BC_MIN_SPEED_MMPS = 20U` (`BC_ERR_STALL`).
- Verified in `testing/subsystem3/TEST_LOG.md`: `S3-U01` (`1980 checks, 0 failed`), `S3-U02`, `S3-U04`, `S3-U06` (`0` misreads), and `S3-U07`.

---

## #21 [CHECKED, NO IMPACT]
**Buddy:** Buddy 1 (Damian — Subsystem 1: Telemetry)  
**Affected Requirement / Component:** FR3, NFR1, IF-04 / `task_tel.c`, `tel_outbox.c`, `config/priorities.h`  
**Area:** Tasks / components; Timing / performance; Safety / fault handling  

**What needs to change?:**
Evaluate whether Telemetry queues can block control tasks or if `FAULT_WIFI_LOSS` should stop navigation.

**Decision:** No change  

**Why?:**
Non-blocking `TMO_POL` event posting, P9 priority below P1–P8, 16-slot drop-oldest outbox buffering, and continuing local-safety autonomous driving during Wi-Fi loss already satisfy `FR3` and `NFR1` cleanly (`#6` fixes reconnect).

**Engineering Evidence:**
- `task_tel.c:85-93, 565-585` & `config/priorities.h:24-40`: `tel_post()` (`MBF_TELEMETRY`, `32 × 8 B`) never waits (`TMO_POL`) and sets `b8` (`WIFI_LOSS`) without latching `FAULT_LATCHED`.
- Verified in `testing/subsystem1/TEST_LOG.md`: `S1-U04` (`64` posts in `166 µs`), `S1-U10` (`50` wakes/s, `0` overruns), and `S1-H02`.

---

## #22 [CHECKED, NO IMPACT]
**Buddy:** Buddy 2 (Ruo Chen — Shared / Platform)  
**Affected Requirement / Component:** NFR6, NFR10, NFR13 / `config/priorities.h`, `src/common/ipc.c`  
**Area:** Shared data / concurrency / interrupts; Timing / performance  

**What needs to change?:**
Evaluate whether the RTOS task priority table or `MTX_MOTION` synchronization model must change.

**Decision:** No change  

**Why?:**
The fixed `P1..P10` priority hierarchy with `TA_INHERIT` priority inheritance on `MTX_MOTION` (and Safety never taking `MTX_MOTION`) already bounds latency and prevents priority inversion during fault handling.

**Engineering Evidence:**
- `config/priorities.h:24-45` & `src/common/ipc.c:176`: `P1` Safety > `P2` VC > `P3` Motion > `P4–P10`; `MTX_MOTION` uses `TA_INHERIT`.
- Verified in `testing/system/TEST_LOG.md` `SYS-01`: `798 s` run with `60 s` console stress measured **`10–566 µs`** worst-case jitter (`< 1000 µs` limit, `0` missed deadlines).
