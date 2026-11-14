# INF2004 Embedded Systems Programming: Lecture 4 Study Notes
## Real-Time Operating Systems (W4.1) and State Machines (W4.2)

---

## Learning Objectives

By the end of this lecture, you should be able to:

1. Explain why "real-time" means predictable, not fast, and why worst-case latency matters more than average throughput.
2. Describe what an RTOS does: split work into tasks, wait instead of poll, protect shared resources, pass messages, and control timing.
3. List the six time-critical kernel services an RTOS coordinates.
4. Explain the difference between a function and a task, and trace a task through its states (READY, RUNNING, BLOCKED, TERMINATED).
5. Explain preemptive priority scheduling and why a context switch must have a bounded cost.
6. Explain how task interleaving creates race conditions, and why critical sections must be short.
7. Choose correctly between a semaphore, mutex, event flag, message buffer, mailbox, and fixed memory pool.
8. Explain priority inversion and how priority inheritance limits it.
9. Use time services (delay, cyclic handler, alarm handler) instead of busy waiting.
10. Explain why an ISR should stay short and defer work to a task.
11. Read basic μT-Kernel 3.0 API names (`tk_cre_tsk`, `tk_wai_sem`, `tk_loc_mtx`, and so on) and understand the pattern behind them.
12. Describe how several RTOS primitives combine into a real design (the sensor pipeline).
13. Define a finite state machine (FSM) and its five building blocks: state, event, guard, action, transition.
14. Decide when an FSM is a good fit and when it is a bad fit.
15. Design an FSM with a state table, and implement it three ways: bare-metal lookup table, switch-case, and superloop.
16. Implement an FSM inside a single RTOS task fed by an event queue.
17. Use an FSM to control when filtered sensor data can be trusted.
18. Avoid "state machine spaghetti" using naming, ownership, short actions, and full state/event testing.

---

# PART 1: REAL-TIME OPERATING SYSTEMS (W4.1)

**Agenda of the lecture:** predictable timing, synchronisation tools, μT-Kernel APIs, common mistakes, application example.

## 1. Real-Time = Predictable, Not "Fast"

- Correctness depends on **both** the computed result **and when it is delivered**. This repeats the definition from Lecture 1: a late answer is a wrong answer.
- A deadline may be **hard**, **soft**, or **none**. The consequence of being late is what differs (catastrophic, degraded, or harmless).
- **Worst-case latency and bounded blocking matter more than average throughput.** A system that is fast 99.9% of the time but occasionally stalls for 50 ms can still fail a 5 ms deadline.
- **The RTOS provides mechanisms. The application designer still has to prove the timing assumptions.** Using an RTOS does not make your system real-time by magic.

**Design question to ask every time:** *What is the latest acceptable response, and NOT the average response?*

## 2. What an RTOS Does For You

An RTOS makes concurrency and timing **explicit**:

| Idea | Meaning |
|---|---|
| Split work into **Tasks** | Each job gets its own independent flow of control |
| **Wait** instead of polling | A task sleeps until something happens, instead of looping and checking |
| **Protect** shared resources | Use mutexes, semaphores, and critical sections |
| Use **kernel messaging** | Tasks talk through kernel-managed objects, not loose global variables |
| **Control timing** | Delays, periodic release, and timeouts are built in |

This is the natural next step after Lectures 2 and 3. You already saw polling, interrupts, timers, and race conditions. An RTOS packages the fixes into one consistent toolkit.

## 3. The Six Time-Critical Kernel Services

An RTOS coordinates six services:

1. **Task scheduling.** Executes "parallel" tasks on one processor by scheduling them, and shares the processor fairly.
2. **Task synchronisation.** Critical sections, semaphores, monitors, mutual exclusion.
3. **Inter-task communication.** Queuing of requests and data between tasks.
4. **Managing and servicing real-time events.** Interrupts and their hand-off to tasks.
5. **Real-time clocks, timers, and delays.**
6. **Memory management.** Critical so the device can run indefinitely without a reboot. The concerns are memory leaks and memory fragmentation.

**Extra point from the slide:** an embedded system has many parts besides the CPU and memory (peripherals, buses, devices). The OS must manage these as **resources** too.

## 4. μT-Kernel 3.0: The Example RTOS

The lecture uses **μT-Kernel 3.0** as its concrete example.

**Why this one:**
- Designed for small embedded and IoT-edge systems.
- Compact and configurable.
- Portable across processor families.
- Compliant with **IEEE 2050-2018**.
- Official API and source are publicly documented (TRON Forum).

**Software stack (top to bottom):**

```
APPLICATION TASKS   sensor_task()   filter_task()   logger_task()
μT-Kernel RTOS      scheduler • sync • IPC • time • memory
BSP + Device Drivers
microcontroller
```

**API naming pattern: `tk_<operation>_<object>`.** Once you see the pattern, the names are easy to decode.

| Call | Meaning |
|---|---|
| `tk_cre_tsk` | create a task |
| `tk_wai_sem` | wait for tokens (semaphore) |
| `tk_loc_mtx` | lock a mutex |
| `tk_snd_mbf` | send copied data (message buffer) |

Decoding tip: `cre` = create, `sta` = start, `wai` = wait, `sig` = signal, `loc` = lock, `unl` = unlock, `snd` = send, `rcv` = receive, `dly` = delay, `slp` = sleep, `set` = set, `get` = get, `rel` = release. `tsk` = task, `sem` = semaphore, `mtx` = mutex, `flg` = event flag, `mbf` = message buffer, `mbx` = mailbox, `mpf` = fixed memory pool, `cyc` = cyclic handler, `alm` = alarm handler.

## 5. A Task Repeatedly Waits and Works

- A **function** runs when called, then returns.
- A **task** runs independently. It normally never returns.
- A task **waits when idle** (it is BLOCKED and uses no CPU).
- An **event makes the task READY again**.

Typical task shape:

```c
void some_task(void) {
    for (;;) {
        wait_for_event();   // block here until signalled
        process();          // do the work
    }                       // loop back and wait again
}
```

Compare this with the superloop in Part 2. The RTOS task blocks efficiently. The superloop checks continuously.

## 6. Task States

A task moves between states in response to events:

| From | To | Trigger |
|---|---|---|
| (start) | **READY** | startup |
| **READY** | **RUNNING** | scheduler picks it (highest priority READY task) |
| **RUNNING** | **READY** | **pre-emption** by a higher-priority task |
| **RUNNING** | **BLOCKED** | task is blocked (waiting on a semaphore, delay, message, and so on) |
| **BLOCKED** | **READY** | task is placed in the ready queue (the thing it waited for happened) |
| **RUNNING** | **TERMINATED** | task exits |
| **READY** / **BLOCKED** | **TERMINATED** | task is deleted |

Key point: only **one** task per core is RUNNING. Blocking decides who is READY, and scheduling decides which READY task runs.

## 7. Preemptive Scheduling

- A **higher-priority task becomes READY** after an interrupt, signal, or message.
- The scheduler **preempts** the lower-priority running task immediately.
- **Within equal priorities**, an implementation may use **FIFO** or **time slicing** (round robin).
- **Priority assignment encodes urgency, not "importance" in a human sense.** The task with the tightest deadline should get the highest priority, even if it feels less important than a task that logs data.

The lecture's diagram shows three tasks with different priorities. The high-priority task interrupts the lower ones whenever it becomes READY, and the lower ones resume afterwards.

## 8. Context Switch Must Have a Bounded Cost

- Each task has a **TCB (Task Control Block)**. The TCB holds the saved state of the task (registers, stack pointer, priority, current state).
- A context switch does three things:
  1. **Save minimal state** of the current task (Task T0) into its TCB.
  2. **Choose the next READY task.**
  3. **Restore its state** from its TCB (Task T1).
- The cost of this must be **bounded and predictable**, because it adds to every response time. This is the same idea as the interrupt context save from Lecture 2, applied to whole tasks instead of ISRs.

## 9. Interleaving Turns a Shared Resource Into a Race

Shared variable `count = 0`. Both Task A and Task B want to add 1.

| Step | Task A | Task B |
|---|---|---|
| 1 | read 0 | |
| 2 | | read 0 |
| 3 | add 1 | add 1 |
| 4 | write 1 | write 1 |

- **Expected: 2. Actual: 1.**
- Both tasks read the same old value, so the result depends on timing.
- **"A scheduler did not 'cause' the bug. It exposed the missing synchronisation."** The bug was always there. Preemption just made it visible.
- This is the same read-modify-write hazard as the ISR example in Lecture 2 (expected 9, got 8). Remember: `volatile` does not make it atomic.

## 10. Critical Sections Must Be Short

Structure:

```
compute (unprotected) → lock → read/update shared state → unlock → continue (unprotected)
```

**Rules while holding protection:**
- Do **not** block.
- Do **not** allocate memory.
- Do **not** log.
- Do **not** do slow I/O.

**Why:** shorter protected paths reduce blocking on other tasks, which makes **worst-case latency easier to bound**. A long critical section on a low-priority task can delay a high-priority one.

## 11. Semaphore: Countable Availability

A semaphore holds a **count**.

| Operation | API | Behaviour |
|---|---|---|
| **WAIT** | `tk_wai_sem(semid, 1, tmout)` | If count ≥ request: decrement and continue. Otherwise: **block** in the semaphore's wait queue. |
| **SIGNAL** | `tk_sig_sem(semid, 1)` | Increment the count, or release a waiting task. |

- Example from the slide: count = 2, three tasks T1, T2, T3. Two get tokens, the third waits in the wait queue.
- **Use for:** resource counts (N identical resources), producer/consumer, event occurrences.
- **Not ownership.** Any task may signal a semaphore. It does not track who "holds" it.

**Semaphore lets a consumer sleep until data arrives.**
- Consumer calls `tk_wai_sem` and goes **BLOCKED**.
- Producer produces an item and calls `tk_sig_sem`.
- The signal makes the consumer **READY**, and it processes the item.
- **No polling loop. No wasted processor time while the queue is empty.**

## 12. Mutex: Ownership Plus Mutual Exclusion

- `tk_loc_mtx(mtxid, tmout)`: acquire. Block if another task owns it.
- `tk_unl_mtx(mtxid)`: release. **Only the owning task unlocks.**
- **Mutex = ownership + exclusion + wait queue.**
- Example from the slide: low-priority Task L owns the mutex that protects shared state. High-priority Task H tries to acquire it and must wait.

**Semaphore vs mutex, plainly:**
- Semaphore: "how many are available?" Anyone can signal.
- Mutex: "who has the lock right now?" Only the owner releases. Use it to protect **one shared invariant**.

## 13. Priority Inversion and Priority Inheritance

**The problem (without protocol):**
1. Low-priority **L** holds the mutex.
2. High-priority **H** becomes READY and waits for the mutex.
3. **Medium-priority M** becomes READY. It is higher than L, so it preempts L and runs.
4. L cannot finish and release the mutex while M runs. So **H waits indirectly for M**, even though H has higher priority than M.

**The fix (priority inheritance):**
- While H waits, **L temporarily inherits H's priority**. So M cannot preempt it.
- L finishes its critical section and releases the mutex.
- L drops back to its normal priority, and H runs.
- In μT-Kernel this is selected with the mutex attribute **`TA_INHERIT`** (or **`TA_CEILING`** where appropriate, which is the priority ceiling protocol).

Extra context: this exact bug famously caused repeated system resets on the Mars Pathfinder mission in 1997. It was fixed by enabling priority inheritance on the mutex.

## 14. Event Flags: Wait for Combinations of Conditions

- One flag object holds a **bit pattern** (for example 8 bits, b0 to b7).
- **Producers set bits:** `tk_set_flg(flgid, pattern)`.
- A task waits for a pattern with `tk_wai_flg(flgid, waiptn, wfmode, &p_flgptn, tmout)`.
- The wait mode chooses the condition:
  - **`TWF_ORW`**: wake on **ANY** of the requested bits.
  - **`TWF_ANDW`**: wake only when **ALL** of the requested bits are set.
- **Best for state or condition notification. Not for carrying a payload.**
- Example use: "ADC done" OR "timeout" OR "stop requested" all set different bits, and one task waits on the combination.

## 15. Message Buffers: Copy Variable-Length Data

- Sender calls `tk_snd_mbf`. Receiver calls `tk_rcv_mbf`.
- The **kernel buffer** stores `length + bytes` and **copies the payload**.
- Features: variable message sizes, sender and receiver may block, **the kernel owns queued bytes**, capacity and max size are configured.
- **Simple ownership. Copying cost grows with payload size.**

## 16. Mailboxes: Queue Pointers, Not Payloads

- Sender and receiver use `tk_snd_mbx` / `tk_rcv_mbx`.
- Only the **pointer** (address) to an application-owned message is queued. Example: address `0x20001240`.
- **Fast handoff:** no payload copy.
- **Dangers:**
  - The message's lifetime must **outlive the queue**.
  - Producer and consumer must **agree who releases the memory**.
- **"Zero-copy shifts the hard problem to ownership and lifetime."**

## 17. Fixed Memory Pools: Bounded Allocation

Example: a pool of **8 × 64-byte blocks** (B0 to B7).

- `tk_get_mpf(mpfid, &blk, tmout)`: take a block.
- `tk_rel_mpf(mpfid, blk)`: return it.

| Benefits | Trade-offs |
|---|---|
| Bounded search and allocation time | Fixed block size wastes internal space |
| **No external fragmentation** | Capacity must cover the **worst-case** demand |
| Waits or times out when empty | |

This is why RTOS designs prefer pools over `malloc`. `malloc` can take unpredictable time and fragment the heap.

## 18. Choose Primitives by What the Wait Means

| Primitive | Represents | Payload | Ownership | Use when |
|---|---|---|---|---|
| **Semaphore** | Count / occurrence | None | No owner | N identical resources; producer/consumer |
| **Mutex** | Exclusive ownership | None | Lock owner | Protect one shared invariant |
| **Event flag** | Condition bits | Bit pattern | No payload owner | Wait for ANY/ALL conditions |
| **Message buffer** | Queued bytes | Copied data | Kernel owns copy | Variable-size messages; simple lifetime |
| **Mailbox** | Queued pointers | Message pointer | Application-defined | Large messages; zero-copy handoff |
| **Memory pool** | Fixed blocks | Memory block | Borrower returns | Bounded allocation without fragmentation |

**Rule from the slide:** *start with semantics. Optimise copying or wake-up behaviour only after the design is correct.*

## 19. Time Services Avoid Busy Waiting

| Service | API | Behaviour |
|---|---|---|
| **DELAY** | `tk_dly_tsk` | Task goes RUN → WAIT Δt → READY. The CPU is free for other tasks meanwhile. |
| **CYCLIC** | `tk_cre_cyc` / `tk_sta_cyc` | **Periodic release** at 0T, 1T, 2T, 3T and so on. Good for fixed-rate work. |
| **ALARM** | `tk_cre_alm` / `tk_sta_alm` | **One-shot callback** at a set time. |

- **Timeouts belong in the normal design, not only in error recovery.** Every blocking call takes a `tmout` argument. Decide what should happen if the event never comes.
- Extra: in the T-Kernel family, `TMO_FEVR` means wait forever and `TMO_POL` means poll without waiting. Avoid waiting forever on anything that can fail.
- Why cyclic beats a delay loop: `tk_dly_tsk` after each run drifts, because the work time adds to the period. A cyclic handler releases at a fixed rate regardless of how long the work takes. (This links to the "software delay adds jitter" point from the ADC lecture.)
- Cyclic and alarm handlers run like interrupt handlers, so keep them short and only signal a task.

## 20. Keep the Interrupt Handler Short; Defer Work to a Task

```
HARDWARE device event
   → INTERRUPT CONTEXT: ISR acknowledge + capture
   → (signal / flag / message)
   → TASK CONTEXT: high-priority task processes data
```

| In the ISR (do) | Avoid |
|---|---|
| ✓ Clear the cause | ✕ Blocking |
| ✓ Capture the minimum data | ✕ Long loops |
| ✓ Wake a task | ✕ Unbounded work |

This is the "capture now, process later" pattern from Lecture 2, now with an RTOS doing the wake-up (semaphore signal, event flag set, or message send from the ISR).

## 21. Application Example: A Sensor Pipeline

Five tasks and an ISR combine several primitives:

```
ADC ISR --(event flag)--> Acquire --(message buffer)--> Filter --(mailbox pointer)--> Control --(semaphore)--> Logger
                                                                   ^ fixed pool          ^ I²C mutex
cyclic release: 1 ms
```

| Edge / object | Role |
|---|---|
| Cyclic release, **1 ms** | Drives the sampling period |
| **Event flag** (ADC ISR → Acquire) | ISR reports "sample ready" |
| **Message buffer** (Acquire → Filter) | Copies raw sample data |
| **Mailbox pointer** (Filter → Control) | Passes a larger result without copying |
| **Fixed pool** | Supplies the blocks that mailbox messages point to, with bounded allocation |
| **I²C mutex** | Protects the shared I²C bus (one device at a time) |
| **Semaphore** (Control → Logger) | Counts items waiting to be logged |

**Design rule: label each edge with the invariant or ownership rule it enforces.**

## 22. Making Waiting and Ownership Explicit: The Summary

- Scheduling chooses the **highest-priority READY task**. Blocking determines **who is READY**.
- **Semaphore = count. Mutex = ownership. Event flag = condition bits.**
- **Message buffer = copied bytes. Mailbox = queued pointers.**
- **Cyclic/alarm** services express time. **ISRs defer work.** **Fixed pools bound allocation.**
- For every object, define: **invariant, owner, capacity, timeout, and worst-case blocking.**

**Three checks before you finish a design:**
1. Mutex or semaphore?
2. Copied data or pointer?
3. What blocks the high-priority task?

## 23. Appendix: μT-Kernel 3.0 API Map

| Group | Calls |
|---|---|
| **Tasks** | `tk_cre_tsk` create, `tk_sta_tsk` start, `tk_dly_tsk` delay, `tk_slp_tsk` sleep |
| **Sync** | `tk_cre_sem` semaphore, `tk_wai_sem` wait, `tk_sig_sem` signal, `tk_cre_mtx` mutex, `tk_loc_mtx` lock, `tk_unl_mtx` unlock |
| **IPC + time + memory** | `tk_set_flg` / `tk_wai_flg`, `tk_snd_mbf` / `tk_rcv_mbf`, `tk_snd_mbx` / `tk_rcv_mbx`, `tk_cre_cyc` / `tk_sta_cyc`, `tk_cre_alm` / `tk_sta_alm`, `tk_get_mpf` / `tk_rel_mpf` |

**Warning from the slide:** the exact service profile and **allowed calling context** (task vs handler) are implementation and specification questions. Check them before coding. Some calls that are fine in a task (anything that blocks) are not allowed in an ISR.

---

# PART 2: STATE MACHINES (W4.2)

**Title:** *State Machine: Structuring Event-Driven Behaviour in Embedded Systems.*

## 24. The Simplest State Machine

The opening example is an Off/On toggle.
- Two states: **Off** and **On**.
- One event: **buttonpressed**.
- Pressing the button in Off goes to On. Pressing it in On goes to Off.
- The **filled dot with an arrow** marks the **initial state** (Off).

Same event, different response. That is the whole point of an FSM.

## 25. Current State + Event Determines the Response

**Correct Response = Current State + Event.**

The button press alone does not tell you what to do. You also need to know what mode you are in. An FSM stores that "remembered mode" explicitly instead of hiding it in scattered flags.

## 26. The Five Parts That Make Behaviour Explicit

| Part | Question it answers |
|---|---|
| **STATE** | What mode you're in |
| **EVENT** | What just happened |
| **GUARD** | Is it allowed? (a condition that must be true) |
| **ACTION** | What the system does |
| **TRANSITION** | Where it goes next |

## 27. Worked Example: A Vending Machine

Follow one purchase from start to finish:

| Current state | Event | Guard | Action | Next state |
|---|---|---|---|---|
| WAITING | Insert coin | (none) | Store credit | HAS CREDIT |
| HAS CREDIT | Choose snack | **Enough credit?** | Release item | VENDING |
| VENDING | Item drops | (none) | Return change | READY AGAIN |

**At every step: current state + event → action + next state.**
**The next state becomes the starting point for the following event.**

Note the guard: if the credit is not enough, the choose-snack event is ignored and the machine stays in HAS CREDIT.

## 28. When to Use an FSM: The Four-Step Workflow

| Step | What to do |
|---|---|
| **CHOOSE** | Identify behaviour that benefits from explicit states |
| **DESIGN** | Name states, events, guards, actions, and transitions |
| **IMPLEMENT** | Use the same core pattern every time |
| **TEST** | Exercise **every state/event pair** |

## 29. State Machine Checklist

| Category | Details |
|---|---|
| **Strong fit** | Reactive control, operating modes, handshakes, protocols, retries, timeouts, lifecycle and fault recovery |
| **Useful signal** | **Many flags whose combinations secretly describe modes.** If you have `is_connecting`, `is_online`, `has_error` as separate booleans, that is an FSM in disguise |
| **Weak fit** | Pure calculation, a short linear algorithm, bulk data transformation, or **many independent activities that create state explosion** |
| **Rule of thumb** | If "what should happen next?" needs **both event and history**, consider an FSM |

**State explosion, explained:** if you put N independent yes/no concerns into one FSM, you can need up to 2^N states. Split independent concerns into separate FSMs before the state count multiplies.

## 30. Handle One Event at a Time

The event-handling loop for any FSM:

1. **Receive** one event.
2. **Decide** using the current state.
3. **Act** quickly.
4. **Update** the state.
5. **Return** and wait for the next event.

**Never wait inside an action.** Start the work, then handle **completion or timeout as a new event.**

Example: to connect to a network, the action calls `net_start()` and returns at once. A later LINK_UP or TIMEOUT event tells the FSM what happened. The FSM never sits in a blocking loop waiting for the link.

## 31. Example: Two Buttons Select a 2-Bit LED State

- Two LEDs (LED 2 and LED 1) display a 2-bit value.
- **BTN 1** steps the LEDs through the four patterns in order: `00 → 01 → 10 → 11`, then wraps back to `00`.
- **BTN 2** is **RESET** and jumps straight back to `00`.

## 32. An FSM With Four Explicit States

- States: **00, 01, 10, 11** (the numbers are the LED 2, LED 1 output).
- **BTN1** moves to the next state (00→01→10→11).
- **RESET** returns to 00 from any state.

## 33. A State Table Removes Ambiguous Branches

A state table lists **every** state and event combination. Here: 4 states × 2 inputs = 8 rows.

| Current state | LED 2 | LED 1 | Input | Next state | LED 2 | LED 1 |
|---|---|---|---|---|---|---|
| 00 | 0 | 0 | BTN 1 | 01 | 0 | 1 |
| 00 | 0 | 0 | BTN 2 | 00 | 0 | 0 |
| 01 | 0 | 1 | BTN 1 | 10 | 1 | 0 |
| 01 | 0 | 1 | BTN 2 | 00 | 0 | 0 |
| 10 | 1 | 0 | BTN 1 | 11 | 1 | 1 |
| 10 | 1 | 0 | BTN 2 | 00 | 0 | 0 |
| 11 | 1 | 1 | BTN 1 | 00 | 0 | 0 |
| 11 | 1 | 1 | BTN 2 | 00 | 0 | 0 |

- **BTN2 = RESET** (annotated on the slide).
- **BTN 1 in state 11 wraps to 00.**
- Why a table helps: there is no "what happens if...?" left unanswered. Every state/event pair has a defined result. This is also your **test plan**.

## 34. Implementation 1: Bare-Metal (Lookup Table)

```c
typedef enum { S00, S01, S10, S11 } State;
typedef enum { EV_NONE, EV_NEXT, EV_RESET } Event;

static State state = S00;
static const State on_next[] = { S01, S10, S11, S00 };

static void dispatch(Event e)
{
    if (e == EV_RESET) {
        state = S00;
    } else if (e == EV_NEXT) {
        state = on_next[state];
    }
    leds_write((uint8_t)state);
}
```

| Idea | What it means |
|---|---|
| **Name the choices** | Two enums define every valid state and incoming event |
| **Remember one state** | The `state` variable starts at S00 and stores where the machine is now |
| **Choose the next state** | RESET returns to S00. NEXT uses the lookup table to advance |
| **Update the output** | After every event, the LEDs are refreshed to show the new state |

Details worth understanding:
- `on_next[]` is indexed by the current state. `on_next[S01]` gives `S10`. Because enums count from 0, S00=0, S01=1, S10=2, S11=3.
- That numbering is why `leds_write((uint8_t)state)` works. The state value **is** the 2-bit LED pattern.
- `static const` means the table is private to the file, read-only, and can live in flash.
- `EV_NONE` means "nothing happened". It is not dispatched.
- Best when the machine is a simple sequence and the transitions are regular.

## 35. Implementation 2: Switch-Case

```c
static void dispatch(Event e)
{
    switch (state) {
    case S_IDLE:
        if (e == EV_START && ready()) state = S_RUN;
        else if (e == EV_FAULT)       state = S_ERROR;
        break;
    case S_RUN:
        if (e == EV_DONE)             state = S_READY;
        else if (e == EV_FAULT)       state = S_ERROR;
        break;
    case S_READY:
        if (e == EV_START)            state = S_RUN;
        else if (e == EV_RESET)       state = S_IDLE;
        break;
    case S_ERROR:
        if (e == EV_RESET && fault_cleared()) state = S_IDLE;
        break;
    }
    output_apply(state);
}
```

| Idea | What it means |
|---|---|
| **Current state chooses the case** | `switch(state)` runs only the rules for the state the machine is in now |
| **Input and guards decide** | An event causes a transition only when that state's condition is satisfied (`ready()`, `fault_cleared()` are guards) |
| **Transitions are non-linear** | A state may jump to RUN, READY, ERROR, or IDLE. There is no fixed sequence |
| **Other events do nothing** | If no rule matches, the current state is kept, and the next event can still be handled |

**Transition table for this machine:**

| State | Event (guard) | Next state |
|---|---|---|
| S_IDLE | EV_START (if `ready()`) | S_RUN |
| S_IDLE | EV_FAULT | S_ERROR |
| S_RUN | EV_DONE | S_READY |
| S_RUN | EV_FAULT | S_ERROR |
| S_READY | EV_START | S_RUN |
| S_READY | EV_RESET | S_IDLE |
| S_ERROR | EV_RESET (if `fault_cleared()`) | S_IDLE |

Use switch-case when the machine has **guards, branches, and non-sequential jumps**. The lookup table style cannot express guards easily.

## 36. Implementation 3: Superloop

```c
int main(void)
{
    board_init();
    leds_write((uint8_t)state);

    for (;;) {
        Event e = EV_NONE;
        if (btn1_pressed()) e = EV_NEXT;
        if (btn2_pressed()) e = EV_RESET;

        if (e != EV_NONE) dispatch(e);
        service_watchdog();
    }
}
```

| Idea | What it means |
|---|---|
| **Collect events** | Each loop checks the buttons and turns a press into NEXT or RESET |
| **Let the FSM decide** | `dispatch(e)` handles the event using the machine's current state |
| **Keep servicing** | The **watchdog** still runs every loop, even when no event occurs |

Notes:
- The loop only **turns inputs into events**. The FSM logic stays inside `dispatch()`.
- `service_watchdog()` is the "pet the watchdog" idea from Lecture 3. If the loop ever hangs, the watchdog forces a reset.
- If both buttons are pressed in the same pass, the second assignment wins, so **RESET overrides NEXT**. This is a design decision to be aware of.
- For real buttons, `btn_pressed()` should return a **debounced press event**, not the raw level (see Lecture 2: count an accepted transition, not every loop that sees LOW).
- Superloop limits: every check runs every pass, so the loop must stay short. That is the "blind time" issue from Lecture 2.

## 37. Implementation 4: FSM Inside an RTOS Task

**Big picture:** events are **queued**. One task wakes, runs the FSM, and updates the driver.

```
EVENT SOURCES               QUEUE / NOTIFICATION            ONE COMMUNICATION TASK
START, LINK_UP, LINK_DOWN,  -------event------->  1 WAIT for event → 2 DISPATCH to FSM → 3 ACT on driver
TIMEOUT, RETRY              (task blocks efficiently)     FSM states: DOWN, CONNECTING, ONLINE, ERROR
                                                          current state + event + guard → action + next state
```

- The task **owns the driver and the connection state**.
- **KEY IDEA: one task contains the state machine. Do not create one task per state.** Tasks are for independent flows of work. States are modes of one flow.

**FSM simplifies logic by limiting each state's options.** Each state accepts only a small set of valid events and leads only to a small set of valid next states.

### The handler stays ordinary C

```c
typedef enum { DOWN, CONNECTING, ONLINE, ERROR } ConnState;
static ConnState s = DOWN;

static void on_event(Event e)
{
    switch (s) {
    case DOWN:
        if (e == START) { net_start(); s = CONNECTING; }
        break;
    case CONNECTING:
        if (e == LINK_UP)        s = ONLINE;
        else if (e == TIMEOUT)   s = ERROR;
        break;
    case ONLINE:
        if (e == LINK_DOWN)      s = DOWN;
        break;
    case ERROR:
        if (e == RETRY) { net_start(); s = CONNECTING; }
        break;
    }
}
```

**Transition table from the code:**

| State | Event | Action | Next state |
|---|---|---|---|
| DOWN | START | `net_start()` | CONNECTING |
| CONNECTING | LINK_UP | (none) | ONLINE |
| CONNECTING | TIMEOUT | (none) | ERROR |
| ONLINE | LINK_DOWN | (none) | DOWN |
| ERROR | RETRY | `net_start()` | CONNECTING |

| Idea | What it means |
|---|---|
| **State is memory** | `s` records the current connection mode |
| **Switch narrows choices** | Only the current state's case can run |
| **Events permit moves** | Each case accepts only its valid transitions |
| **Keep action + update together** | The driver call and the next state remain easy to trace |

**Note on the slides:** the state diagram on the "FSMs simplify logic" slide uses simplified labels (START, UP, RETRY, DROP, TIMEOUT) and its arrows do not match the code exactly. For example, the diagram shows ONLINE going to ERR on RETRY, while the code moves ONLINE to DOWN on LINK_DOWN and ERROR to CONNECTING on RETRY. Treat the **code and the table above** as the precise version, and the diagram as a rough illustration. It is worth confirming with your lecturer if it comes up in an assessment.

### The task loop blocks, then dispatches exactly once

```c
void CommsTask(void *arg)
{
    Event e;

    for (;;) {
        if (xQueueReceive(event_q, &e, portMAX_DELAY) == pdPASS) {
            on_event(e);
            publish_state(s);   // optional
        }
    }
}

// ISR or timer: enqueue an event only;
// do not run the FSM in the callback.
```

| Idea | What it means |
|---|---|
| **Block efficiently** | The queue sleeps the task until an event arrives (`portMAX_DELAY` = wait as long as needed) |
| **Dispatch exactly once** | Each received event calls `on_event(e)` one time |
| **Keep one owner** | Only this task runs the FSM, so **events stay serialised** |

- Because only one task touches `s`, **no mutex is needed** for the FSM's own state. That is a big benefit of the "one owner" rule.
- **ISRs and timers only enqueue an event.** They never run the FSM directly. This is "capture now, process later" again.
- Note the API style here: `xQueueReceive` and `pdPASS` are **FreeRTOS** names, while Part 1 used μT-Kernel. The FSM pattern is the same in any RTOS. Only the queue calls change.

## 38. FSMs Decide When Filtered Values Can Be Trusted

**Keep the mathematics and the control logic separate.**

Structure of the filtering component (math and control tested separately):

```
SENSOR SAMPLES → NUMERICAL FILTER (FIR / IIR / median / average) → FILTERED VALUE → OUTPUT GATE (publish / hold) → TRUSTED OUTPUT
                                                                                          ^ VALID enables output
CONTROL EVENTS: RESET, ENOUGH SAMPLES, SENSOR LOST, OUTLIER BURST → CONTROL FSM
CONTROL FSM states: RESET → WARM_UP → VALID → DEGRADED
```

| FSM state | Meaning | How you get there |
|---|---|---|
| **RESET** | Filter cleared, nothing trusted | Reset event |
| **WARM_UP** | Samples are arriving but the filter is not yet settled | Samples arrive |
| **VALID** | Output is stable and trustworthy. **VALID enables the output gate** | Enough samples, stable |
| **DEGRADED** | Output is no longer trusted | Sensor lost or outlier burst |

- The numerical filter **transforms samples**. The FSM **decides whether the result may be used**.
- Why warm-up matters: an IIR filter or moving average starts with empty history, so its first outputs are misleading. Publishing them would feed bad data to the controller.
- **KEY IDEA: filter math changes values. The FSM controls when those values may be used.**
- This is also a good example of a **weak-fit-for-FSM problem (pure math) paired with a strong-fit problem (mode control)**. Only the control part becomes an FSM.

## 39. Preventing State-Machine Spaghetti

| Rule | Meaning |
|---|---|
| **NAME** | Name states by **stable behaviour**, not temporary code locations. Good: `CONNECTING`. Bad: `STATE_AFTER_STEP_3` |
| **OWN** | Give **one owner** explicit events, not hidden global flags |
| **DEFINE** | **Keep actions short.** Define **every state/event result** |
| **TEST** | Test **traces**: repeats, timeout, invalid event, reset, and recovery |

This ties back to the Toyota "spaghetti code" case study in Lecture 1. An FSM keeps behaviour readable only if you keep the discipline above.

## 40. Summary: Make State Explicit, Events Finite, Actions Short

| Step | Guidance |
|---|---|
| **CHOOSE** | Use an FSM when **remembered mode changes the correct response** |
| **STRUCTURE** | **State + event + guard → short action + next state** |
| **PLACE** | Keep it inside **one owning loop or RTOS task** |
| **SCALE** | **Split independent concerns before states multiply** |
| **VERIFY** | **Complete the table.** Test normal, invalid, timeout, and recovery paths |

---

## Quick-Reference Summary

**RTOS**
- Real-time means predictable. Design for the **worst case**, not the average.
- Task states: READY, RUNNING, BLOCKED, TERMINATED. Highest-priority READY task runs, and higher priority preempts lower.
- Context switch saves and restores task state via the **TCB**, and its cost must be bounded.
- Two tasks doing read-modify-write on shared data will race. Protect it with a **short critical section**.
- **Semaphore** = count (no owner). **Mutex** = ownership (owner unlocks). **Event flag** = ANY/ALL condition bits. **Message buffer** = copied bytes. **Mailbox** = pointers (watch lifetime). **Fixed pool** = bounded allocation, no fragmentation.
- **Priority inversion:** M starves H by delaying L. Fix with **priority inheritance** (`TA_INHERIT`) or ceiling (`TA_CEILING`).
- Use **delay / cyclic / alarm** instead of busy waiting. Build timeouts into the normal design.
- **ISR:** clear cause, capture minimum, wake a task. No blocking, no long loops.
- For every object define: invariant, owner, capacity, timeout, worst-case blocking.

**State machines**
- **Correct response = current state + event.**
- Five parts: **state, event, guard, action, transition.**
- Good fit: modes, protocols, handshakes, retries, timeouts, fault recovery. Weak fit: pure math, short linear code, many independent activities.
- **Handle one event at a time. Never wait inside an action.** Completion and timeout are new events.
- Implementations: **lookup table** (simple sequences), **switch-case** (guards and jumps), **superloop** (poll inputs into events, keep servicing the watchdog), **RTOS task** (queue in, one owner task, ISRs only enqueue).
- **One task holds the FSM. Do not make one task per state.**
- Use an FSM to gate **when filtered data is trusted** (RESET → WARM_UP → VALID → DEGRADED).
- Avoid spaghetti: **name by behaviour, one owner, short actions, define every state/event pair, test traces.**
