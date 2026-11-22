pre tribunal prompt for context
I am buddy 2, Lee Ruo Chen.

You are a senior embedded-systems engineer reviewing a finished, working autonomous
line-following robot car (Raspberry Pi Pico W / RP2040, micro T-Kernel RTOS, C, Pico SDK
1.5.1, Barr C standard). Assume the current system works as intended. Your job is to compare
the NEW requirements with the CURRENT system and decide, with evidence, what must change
and what stays the same.

Read first, in this order:
1. START_GUIDE.md, then every Tier 1 and Tier 2 file it lists (HANDOVER.md, TEAM.md,
   INTEGRATION_PHASES.md, INTEGRATION_FIXES.md, config/*.h, src/common/ipc.h,
   src/common/ipc_types.h, src/common/data_repr.h, docs/DATA_REPRESENTATION.md,
   src/rtos/port/hal.h, src/common/period.h, docs/DEBUG.md, app_program/app_main.c,
   ARCHITECTURE.md (design deltas D1..Dn), design.md, specs.md).
2. My own subsystem: the Tier 3 entry for Buddy <N> in START_GUIDE.md, meaning every source
   file, header, test file and roadmap in app_program/subsystem<N>/, plus
   testing/subsystem<N>/TEST_LOG.md and its logs.
3. testing/integration/TEST_LOG.md and testing/system/TEST_LOG.md.
4. docs/trib2/requirements.md and docs/trib2/software_block_diagram.svg.

Shared files are config/*.h, src/common/*, src/rtos/*, app_program/app_main.c and app.h,
CMakeLists.txt, sources.cmake, TEAM.md, ARCHITECTURE.md, design.md and
docs/DATA_REPRESENTATION.md.

This is a read-only assessment. Do not edit code, config or docs, and do not commit. Reply
"Ready" when you have read everything. The next message gives the new requirements and the
report format.


NEW REQUIREMENTS:
<<to be filled>>

Report output path: doc/trib2/buddy<N>.md

explain the reason and provide engineering evidence such as 
	- Requirement values, calculations, measurements, test results, source-code behaviour, interfaces, statecharts, block diagrams, hardware limits, RTOS behaviour, or datasheet specifications.
for every suggestion, list it as "OBVIOUS CHANGE" and other suitable namings, but i need obvious to be there.

Method
1. Review the new requirements against the complete system. Identify what needs to change
   and what should stay unchanged.
2. List the suggestions. Keep the list focused: ordered from highest to lowest impact
3. Tag every suggestion with one type:
   - OBVIOUS CHANGE: the new requirement directly names this change.
   - KNOCK-ON CHANGE: needed because another change forces it (for example, a new speed changes a timing budget).
   - HIDDEN RISK: not stated in the requirement, but needed for safety or correctness.
   - CHECKED, NO IMPACT: an area that looks affected but is not.
   Include every OBVIOUS CHANGE before the others.
4. Give each suggestion exactly one decision:
   - Accept: use the suggestion as proposed.
   - Modify: use the idea, but change the proposed solution.
   - Reject: do not use the suggestion.
   - No change: area was considered but existing design should remain
5. Explain the reason and give engineering evidence: requirement values, calculations with the numbers shown, measurements, test results (test ID), source-code behaviour (file:line, #define value), interfaces, statecharts, block diagrams, hardware limits, RTOS behaviour or datasheet specifications. "The code looks fine" is not evidence.

the list of suggestions should be covered in these areas, not all has to be covered, but it needs to be covered if the new requirements need it to be. Do not miss any areas unless the new requirements do not need them.
Requirements
Tasks / components
State machines
Interfaces / messages
Shared data / concurrency / interrupts
Timing / performance
Memory / resources
Safety / fault handling
Hardware / software interaction
Recovery and safe resumption
Verification / testing

Report format (write to the output path)

#1 [OBVIOUS CHANGE]
What needs to change?:
Decision:
Why?:
Engineering Evidence: 
Alternative considered:(if the decision is modify)

Here is an example of a requirement change affecting a system and its format
#5
Affected Requirement / Component: User Interface - Remaining Toast Time

What needs to change?:
Reduce the remaining-time value from 32 bits to 8 bits.

Decision:
Modify

Why?:
The value can be reduced, but 8 bits are not sufficient if the remaining time is represented in seconds.
The maximum toast cycle is 360 seconds.

Engineering Evidence:
Valid range = 0 to 360 seconds.
361 possible values are required.
8 bits can represent only 256 values.
9 bits can represent 512 values.

Decision:
Use 9 bits for the application representation.

Alternative considered:
8 bits could be used with 2-second resolution, but this would reduce the displayed timer resolution.
The team decided to retain 1-second resolution.

#6
Affected Requirement / Component: Bread Detection Sensor - Bread Present

What needs to change?:
Reduce the bread-presence value.

Suggested by:
Team

Decision:
No Change

Why?:
The bread-presence output already has only two application states.

Engineering Evidence:
0 = NO BREAD
1 = BREAD PRESENT

2 states require exactly 1 bit.
The existing application representation is already 1 bit.

#9
Affected Requirement / Component: Safety - Fault Mask

What needs to change?:
Pack the 5 independent fault flags into one shared application fault field.

Suggested by:
AI

Decision:
Modify

Why?:
Packing the fault flags is appropriate, but different tasks may update different fault bits.
A normal read-modify-write operation could lose another task's fault update.

Engineering Evidence:
Temperature task can set OVERHEAT and TEMPERATURE_SENSOR_FAULT.
Bread-detection task can set BREAD_SENSOR_FAULT.
Heating-control task can set HEATER_FAULT.

Example failure:
Task A reads fault mask = 00000.
Task B reads fault mask = 00000.
Task A sets bit 0 and writes 00001.
Task B sets bit 3 and writes 01000.

The final value becomes 01000 and Task A's OVERHEAT fault is lost.

Decision:
Keep the 5-bit logical fault representation, but updates must use a safe single-writer or synchronised mechanism.
