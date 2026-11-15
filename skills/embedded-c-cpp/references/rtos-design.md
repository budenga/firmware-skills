# RTOS design (RTOS-agnostic, FreeRTOS API)

## Contents
1. RTOS or not
2. Task decomposition and schedulability
3. Priorities, preemption and locking
4. ISR → task communication and Cortex-M priorities
5. Timing: delays, tick, tickless, software timers
6. Memory: allocation, stacks, libc reentrancy
7. Watchdog with many tasks
8. Shared peripherals
9. Debugging: tools and symptom → cause
10. Testing RTOS code
11. CMSIS-RTOS2 (CubeMX) notes
12. API mapping: FreeRTOS ↔ Zephyr ↔ ThreadX ↔ CMSIS-RTOS2
13. Worked example: parcel-locker controller
14. Review checklist

Layering, superloop vs RTOS basics, the SPSC ring buffer and state machines are in `architecture.md`; ESP-IDF specifics (bytes for stack size, cores, `IRAM_ATTR`, task WDT) are in the `esp32-firmware` skill (`rtos-memory-power.md`). This file is the design layer between them. FreeRTOS naming is used; section 12 maps it to other kernels.

---

## 1. RTOS or not

| Choose | When | Cost |
|---|---|---|
| Superloop + ISRs | One or two timing domains, everything fits as non-blocking state machines, 8/16-bit MCU | Cheapest, easiest to reason about; latency = longest loop pass |
| Event-driven (run-to-completion, e.g. active objects on a cooperative kernel or QP) | Many event sources, no long blocking calls | Small RAM (one stack), deterministic; every handler must stay short |
| Preemptive RTOS | Independent activities with different deadlines, vendor stacks that block (TCP/IP, LTE modem, USB, BLE), several people on one codebase | A stack per task, priority bugs, more test effort; ~5–10 KB flash for the kernel |

Determinism: preemption bounds the latency of a high-priority task to "ISR + kernel overhead + longest critical section", independent of what low-priority code does — a superloop cannot promise that. Certification: if IEC 61508/ISO 26262/IEC 62304 is in scope, use a kernel with a certification package (SAFERTOS, ThreadX, Zephyr's safety effort, commercial RTOSes) and budget for it; plain FreeRTOS is not certified.

## 2. Task decomposition and schedulability

Split by **timing and criticality**, not by function: one task per independent deadline or blocking source. Fewer tasks is better — each costs a stack, a priority decision and test paths.

- **Each task blocks on exactly one thing** (its queue, its notification, `xTaskDelayUntil`). A task that waits on a queue *and* polls a flag is two tasks or a design bug.
- **Active object / queue-per-task**: each task owns its data and peripherals, receives typed events in one queue, and runs a state machine per event. No shared globals, so no mutexes; testable without the RTOS (section 10).

```c
typedef enum { EV_OPEN_REQ, EV_TIMEOUT, EV_SENSOR } ev_sig_t;
typedef struct { ev_sig_t sig; uint8_t door; uint16_t arg; } ev_t;   /* copied by value */

static void lock_task(void *arg) {
    lock_ao_t *me = arg;
    ev_t e;
    for (;;) {
        if (xQueueReceive(me->q, &e, pdMS_TO_TICKS(1000)) == pdTRUE) lock_fsm_dispatch(me, &e);
        wdg_checkin(WDG_LOCK);                 /* section 7: wake at least once per second */
    }
}
```

**Rate-monotonic priority assignment**: for periodic tasks with deadline = period, shorter period → higher priority. It is optimal among fixed-priority schemes. With deadlines shorter than periods, use deadline-monotonic (shorter deadline → higher priority).

**Schedulability check** (per task: period `T`, worst-case execution `C`, blocking `B` from lower-priority locks):
- Utilisation bound (Liu & Layland): `U = Σ C/T ≤ n(2^(1/n) − 1)` — 0.828 for 2 tasks, → ln 2 ≈ 0.693 for many. Sufficient, not necessary.
- Response-time analysis (exact for fixed priority): iterate `R = C_i + B_i + Σ_{j higher} ⌈R/T_j⌉·C_j` from `R = C_i` until it converges; schedulable if `R ≤ D_i`. Include ISR load as highest-priority "tasks".
- Measure `C` on target (cycle counter, GPIO toggle + scope, trace tool); do not estimate it.

## 3. Priorities, preemption and locking

- `configUSE_PREEMPTION 1`: the highest-priority ready task always runs. `configUSE_TIME_SLICING 1` (default) round-robins equal-priority ready tasks every tick — useful for background tasks, a source of jitter for anything else. Give timing-critical tasks a unique priority.
- FreeRTOS: higher number = higher priority, idle = 0. Zephyr and ThreadX: **lower number = higher priority**. Do not port priority numbers blindly.

**Priority inversion**: low task L holds a lock, high task H blocks on it, medium task M (needs no lock) preempts L — H now waits for M. On Mars Pathfinder (1997, VxWorks) a low-priority meteorological task held the information-bus mutex, medium-priority communications work preempted it, the high-priority bus-management task missed its deadline and a watchdog reset the lander repeatedly; the fix uploaded from Earth was to enable priority inheritance on that mutex.

| Primitive | Ownership | Priority inheritance | From ISR | Use for |
|---|---|---|---|---|
| Mutex (`xSemaphoreCreateMutex`) | Yes — only the taker gives | Yes | No | Protecting a resource |
| Binary semaphore | No — anyone gives | No | Give: yes | Signalling an event |
| Counting semaphore | No | No | Give: yes | Counting events / pool slots |
| Recursive mutex | Yes, nesting count | Yes | No | Legacy code that re-enters its own lock |
| Task notification | n/a | No | Yes | Fast ISR/task → one task signal |

A semaphore is not a mutex because it has no owner: the kernel cannot know whom to boost, and nothing stops another task releasing a lock it never took. Recursive mutexes hide unclear lock structure — needing one is usually a design smell; each take must be matched by a give.

**Deadlock avoidance**: one global lock order (e.g. `SPI` before `FLASH_META`), never block on a queue while holding a mutex, keep lock scope to a few lines, and always take with a timeout that reports an error (`pdMS_TO_TICKS(50)`, not `portMAX_DELAY`) so a deadlock becomes a logged fault instead of a silent hang.

**Starvation**: a higher-priority task that polls without blocking starves everything below it, including the idle task (which frees memory of deleted tasks and enters tickless sleep). Every task loop must block somewhere.

## 4. ISR → task communication and Cortex-M priorities

Only `...FromISR` APIs may be called from an ISR; they never block and report through `pxHigherPriorityTaskWoken` whether the context switch should happen at ISR exit (`portYIELD_FROM_ISR`). Without that yield the woken task runs at the next tick, adding up to one tick of latency.

**Deferred interrupt processing**: the ISR clears the source, captures minimal data, notifies a high-priority handler task, and returns; the task does the work at task level where it can block, log and be preempted.

```c
static TaskHandle_t s_rx_task;
void USART2_IRQHandler(void) {                         /* idle-line ISR: frame complete */
    BaseType_t woken = pdFALSE;
    uart_clear_idle_flag();
    vTaskNotifyGiveFromISR(s_rx_task, &woken);
    portYIELD_FROM_ISR(woken);
}
static void modbus_task(void *arg) {
    for (;;) {
        if (ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(500)) > 0) modbus_handle_frame();
        wdg_checkin(WDG_MODBUS);
    }
}
```

| Mechanism | Payload | Receivers | Notes |
|---|---|---|---|
| Direct-to-task notification | 32-bit value / count / bits | Exactly one task | Fastest, no extra RAM; cannot queue several values; `configTASK_NOTIFICATION_ARRAY_ENTRIES` gives indexed slots |
| Queue | Fixed-size items, copied | Any | Multiple writers/readers; size it for the worst burst and check the send result |
| Stream buffer | Byte stream | One reader, one writer | UART/DMA byte streams; trigger level wakes reader |
| Message buffer | Variable-length messages | One reader, one writer | Length-prefixed frames; serialise multiple writers yourself |
| Event group | Bits | Many | "Wait for all/any"; set from ISR is deferred via the timer task |

**Cortex-M interrupt priorities** — the most common FreeRTOS bring-up bug:
- NVIC: **lower number = more urgent**; 0 is the highest and the reset default. Only the top `__NVIC_PRIO_BITS` bits are implemented (4 on STM32 Cortex-M3/M4/M7/M33 → 0–15; 2 on Cortex-M0/M0+ parts such as STM32F0/G0/L0 → 0–3, which have no BASEPRI, so FreeRTOS critical sections there mask all interrupts).
- `configMAX_SYSCALL_INTERRUPT_PRIORITY` (the Cortex-M ports read only this name; the alias `configMAX_API_CALL_INTERRUPT_PRIORITY` is used by Cortex-A/R and AArch64 ports, not Cortex-M) is the most urgent level allowed to call `FromISR` APIs, stored already shifted: `(5 << (8 - configPRIO_BITS))`. The kernel masks interrupts at or below that urgency in critical sections; ISRs more urgent (numerically lower, e.g. 0–4) are never masked but **must not call any RTOS API**.
- So an ISR that calls `xQueueSendFromISR` needs NVIC priority numerically **≥ 5** (with the CubeMX default `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY = 5`). Leaving it at 0 corrupts kernel lists at random.
- With `configASSERT` defined, `vPortValidateInterruptPriority()` (Cortex-M3/M4/M7/M33 ports) catches both errors: an API call from a too-urgent ISR, and a priority grouping that assigns bits to sub-priority. STM32 HAL must therefore use `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)` (all bits pre-emption) — `HAL_Init()` sets this by default; do not change it.
- `configKERNEL_INTERRUPT_PRIORITY` (SysTick, PendSV) must be the lowest urgency.

## 5. Timing: delays, tick, tickless, software timers

- `vTaskDelay(n)` waits `n` ticks **from now**, so loop period = work time + n and drifts. For fixed rates use `xTaskDelayUntil(&last, period)` (V10.4.2+, returns whether it actually delayed; `vTaskDelayUntil` is the older void form). `pdMS_TO_TICKS(x)` rounds down — at 100 Hz, `pdMS_TO_TICKS(5)` is 0.
- Tick rate: 1000 Hz gives 1 ms resolution at a small CPU cost; 100 Hz is enough when all timing ≥ 10 ms and saves power. Sub-millisecond timing belongs to hardware timers, not the tick.
- Tickless idle (`configUSE_TICKLESS_IDLE 1` uses SysTick; `2` = your own `portSUPPRESS_TICKS_AND_SLEEP`): the kernel stops the tick when all tasks block long enough (`configEXPECTED_IDLE_TIME_BEFORE_SLEEP`). SysTick stops in deep-sleep modes (e.g. STM32 STOP), so deep sleep needs a low-power timer implementation. Polling tasks defeat tickless entirely.
- **Software timers** run their callbacks in the timer service (daemon) task, at `configTIMER_TASK_PRIORITY`, fed by a command queue of `configTIMER_QUEUE_LENGTH`. Gotchas: if the timer task priority is below busy tasks, callbacks run late; `xTimerStart(t, 0)` from a task or any `...FromISR` timer call fails when the command queue is full — check the return value; event-group sets from ISRs also go through this queue.
- Callbacks must not block (no `vTaskDelay`, no queue waits with timeout, no long I2C transfers): they stall every other timer. Post an event to the owning task instead.

## 6. Memory: allocation, stacks, libc reentrancy

- Prefer **static allocation** (`configSUPPORT_STATIC_ALLOCATION 1`, `xTaskCreateStatic`, `xQueueCreateStatic`, ...): RAM usage is visible in the map file and creation cannot fail at run time. Static allocation also requires `vApplicationGetIdleTaskMemory` (and `vApplicationGetTimerTaskMemory` if timers are on); from V11.0 `configKERNEL_PROVIDED_STATIC_MEMORY 1` makes the kernel supply default versions.
- If dynamic: create everything at start-up, never delete. Heap schemes:

| Scheme | Free | Notes |
|---|---|---|
| heap_1 | No | Deterministic; fine when nothing is ever freed |
| heap_2 | Yes, no coalescing | Legacy; fragments — use heap_4 |
| heap_3 | Yes | Wraps libc `malloc/free` with the scheduler suspended; size from linker heap |
| heap_4 | Yes, coalescing | General default |
| heap_5 | Yes, coalescing | heap_4 across non-contiguous RAM regions (`vPortDefineHeapRegions`) |

- Never allocate in an ISR (`pvPortMalloc` and libc `malloc` are not ISR-safe). Set `configUSE_MALLOC_FAILED_HOOK 1`; monitor `xPortGetMinimumEverFreeHeapSize()`.
- **Stack sizing method**: start generous, run the worst-case paths (error handling, logging, longest protocol frame), read `uxTaskGetStackHighWaterMark(NULL)` (minimum free ever — **words** on Cortex-M FreeRTOS, bytes on ESP-IDF), then trim to used + ~25–50 %. Keep `configCHECK_FOR_STACK_OVERFLOW 2` (checks a fill pattern at the stack end on every switch-out) during development and ideally in production, with `vApplicationStackOverflowHook` logging the task name and resetting. It only catches overflows that land in the guard pattern; for hard protection use the MPU (FreeRTOS-MPU ports or a guard region) or Cortex-M33 `PSPLIM`.
- `printf`/`sprintf` with floats can use 1–2 KB of stack and newlib's shared `_reent` state. Either give each task its own reent (`configUSE_NEWLIB_REENTRANT 1`) or route all formatting through one logging task. STM32CubeMX 6.3+ (also inside CubeIDE): Project Manager → Project → **Thread-safe Settings** — enable multi-thread support and pick a FreeRTOS locking strategy (strategy 4 allows, 5 denies lock use from ISRs; both need `configUSE_NEWLIB_REENTRANT 1`). It generates `newlib_lock_glue.c` and `stm32_lock.h` (ST AN5731); otherwise newlib's `malloc` (used by `printf`) is not protected between tasks.

## 7. Watchdog with many tasks

Kicking the hardware watchdog from the idle task, a timer callback or one random task proves only that *something* runs. Use a supervisor: every monitored task checks in, and the supervisor kicks the WDT only when **all** have checked in within the window.

```c
#define WDG_ALL (WDG_LOCK | WDG_MODBUS | WDG_NFC | WDG_NET)
static EventGroupHandle_t s_wdg;
void wdg_checkin(EventBits_t bit) { xEventGroupSetBits(s_wdg, bit); }

static void supervisor_task(void *arg) {
    for (;;) {
        EventBits_t got = xEventGroupWaitBits(s_wdg, WDG_ALL, pdTRUE /*clear*/, pdTRUE /*all*/,
                                              pdMS_TO_TICKS(2000));
        if ((got & WDG_ALL) == WDG_ALL) { hw_wdg_kick(); }
        else { fault_log_missing(WDG_ALL & ~got); /* stop kicking → WDT resets */ }
    }
}
```
- Tasks that block for long (waiting for a card, a network packet) use a receive timeout shorter than the window so they can check in while idle; a task with a legitimately long operation checks in from inside it.
- Run the supervisor at a high priority so it can log *which* task failed before the reset; a CPU hog below it then shows up as missing check-ins. Hardware WDT timeout > supervisor window with margin. Save the culprit mask in no-init RAM (see `startup-linker-bootloaders.md`).

## 8. Shared peripherals

- **Gatekeeper task**: one task owns the peripheral (display, modem UART, log output); others send requests through its queue. No lock, natural serialisation, and the owner can reorder or batch.
- **Mutex per bus** when several drivers share I2C/SPI: take the bus mutex inside the bus layer (`i2c_bus_t` in `architecture.md`), held for a whole transaction including chip-select, with a timeout. One mutex per bus, never one global "HAL mutex".
- **DMA + notification**: the task starts the transfer, blocks on `ulTaskNotifyTake(pdTRUE, timeout)`, and the DMA-complete ISR calls `vTaskNotifyGiveFromISR`. The CPU is free during the transfer; on timeout, abort the DMA and recover the bus. Mind cache maintenance on Cortex-M7 and DMA-capable memory regions.

## 9. Debugging: tools and symptom → cause

- **Trace**: Percepio Tracealyzer and SEGGER SystemView show every task switch, ISR and kernel call on a timeline — the fastest way to see priority inversion, jitter and starvation. OpenOCD/GDB with the FreeRTOS RTOS awareness lists tasks as threads (`debug-tools.md`).
- **Text stats**: `vTaskList` (v11: `vTaskListTasks` with a buffer length; the old name remains as a macro) needs `configUSE_TRACE_FACILITY` and `configUSE_STATS_FORMATTING_FUNCTIONS`; `vTaskGetRunTimeStats` additionally needs `configGENERATE_RUN_TIME_STATS` and a run-time counter ≥10× the tick rate.
- **Asserts and hooks**: define `configASSERT` to log file/line and halt (debug) or reset (production); enable the stack-overflow and malloc-failed hooks.

| Symptom | Likely cause |
|---|---|
| HardFault in `vPortSVCHandler` / first task start | SVC/PendSV/SysTick vectors not mapped to the port handlers; VTOR wrong after a bootloader jump (start-up reads the initial MSP from the vector table); scheduler started with a corrupt heap/TCB |
| HardFault in `xPortPendSVHandler` | A task stack overflowed into a TCB; a non-`FromISR` API called from an ISR; ISR priority above the syscall limit |
| Stuck in `configASSERT` inside `vPortValidateInterruptPriority` | ISR calling the RTOS is at NVIC priority 0–4 (too urgent) or priority grouping is not all-preemption |
| Random variable corruption, behaviour changes with unrelated edits | Stack overflow (check high-water marks, enable method 2); missing mutex |
| A task never runs | Higher-priority task never blocks; creation failed (heap too small, return value ignored); waits on a queue nobody writes |
| Occasional large latency on a high-priority task | Priority inversion through a binary semaphore used as a lock; long critical section; time slicing with an equal-priority task |
| Events silently lost | `xQueueSend(..., 0)` or `FromISR` send on a full queue with the return value ignored; notification overwritten; timer command queue full |

## 10. Testing RTOS code

- Keep logic RTOS-free: state machines take events and time as parameters (`architecture.md` §4) and are tested deterministically on the PC with Unity (`testing.md`). The active-object pattern makes this natural: test `lock_fsm_dispatch()` with a sequence of events, no tasks involved.
- Wrap the few OS calls behind a thin `os_*` layer (queue send/receive, time, mutex) so host builds link a fake that records calls and controls time.
- For integration tests of real task interaction on a PC, build with the FreeRTOS POSIX/Linux port (or the Windows port); timing is not real-time, so test ordering and protocol, not deadlines.

## 11. CMSIS-RTOS2 (CubeMX) notes

- CubeMX's CMSIS-RTOS2 wrapper (`cmsis_os2.c`) maps `osPriority_t` (`osPriorityNormal = 24`, `osPriorityRealtime = 48`, `osPriorityISR = 56`) almost 1:1 onto FreeRTOS priorities — directly in ST's FreeRTOS ≤ V10.6.2 packages, as `osPriority − 1` in ST's V11.x packages — which is why `configMAX_PRIORITIES` must be 56; `freertos_os2.h` has an `#error` if it is not. Mind the off-by-one when comparing numbers in a debugger or mixing native `xTaskCreate` priorities.
- `osDelayUntil()` in v2 takes an **absolute** tick count; `osDelay()` is relative. Thread flags (`osThreadFlags*`) map onto task notifications.
- Using the wrapper costs a little overhead but not features; you can call native FreeRTOS APIs alongside it. Do not mix both to create the same object type in one module.
- FreeRTOS takes SysTick, so CubeMX asks you to move the **HAL timebase** to a basic timer (TIM6/TIM7 or another free TIM). Its ISR must remain at a priority that can preempt where `HAL_Delay`/`HAL_GetTick` timeouts are used, and must not call RTOS APIs if more urgent than the syscall limit.

## 12. API mapping: FreeRTOS ↔ Zephyr ↔ ThreadX ↔ CMSIS-RTOS2

| Concept | FreeRTOS | Zephyr | ThreadX | CMSIS-RTOS2 |
|---|---|---|---|---|
| Task | `xTaskCreate(Static)` | `k_thread_create`, `K_THREAD_DEFINE` | `tx_thread_create` | `osThreadNew` |
| Queue | `xQueueCreate`, `xQueueSend/Receive` | `k_msgq_init`, `k_msgq_put/get` | `tx_queue_create` (item size in 32-bit words, 1–16 by default) | `osMessageQueueNew`, `osMessageQueuePut/Get` |
| Semaphore | `xSemaphoreCreateBinary/Counting` | `k_sem_init`, `k_sem_give/take` | `tx_semaphore_create` | `osSemaphoreNew` |
| Mutex (inheritance) | `xSemaphoreCreateMutex` | `k_mutex_init` (inherits by default) | `tx_mutex_create(..., TX_INHERIT)` | `osMutexNew` with `osMutexPrioInherit` |
| Event group | `xEventGroupCreate` | `k_event_init`, `k_event_post/wait` | `tx_event_flags_create` | `osEventFlagsNew` |
| Notification | `xTaskNotify`, `ulTaskNotifyTake` | No direct equivalent (per-thread `k_sem`/`k_event`) | No direct equivalent (per-thread semaphore) | `osThreadFlagsSet/Wait` |
| Software timer | `xTimerCreate` | `k_timer_init/start` (callback in ISR context) | `tx_timer_create` | `osTimerNew` |
| Delay until | `xTaskDelayUntil` | `k_sleep(K_TIMEOUT_ABS_MS(t))` (needs `CONFIG_TIMEOUT_64BIT`, default y; `t` = ms since boot) | Not built in: `tx_thread_sleep(next - tx_time_get())` or a periodic timer | `osDelayUntil` (absolute) |
| Priority order | Higher number = higher | Lower = higher; negative = cooperative | 0 = highest | Higher = higher |

Zephyr detail belongs in the `zephyr-rtos` skill.

## 13. Worked example: parcel-locker controller

STM32 (Cortex-M4), FreeRTOS, 1 kHz tick. Inputs: RS-485 Modbus RTU slave to the locker master, NFC reader on SPI, LTE modem on UART, 24 V solenoid locks via MOSFET drivers with door-sensor inputs. Stack sizes in words are **starting points to be measured**, not results.

| Task | Prio | Stack (words) | Blocks on | Sends to | Why |
|---|---|---|---|---|---|
| Supervisor | 6 | 192 | Event group, 2 s window | Hardware WDT, fault log | Must run to name the hung task before reset |
| Lock control | 5 | 256 | Own queue (1 s timeout) | Status events → Modbus queue | Solenoid on-time is a safety limit (coil overheating, 24 V supply sag); pulse timed with a hardware one-shot timer, task only sequences and checks the door sensor |
| Modbus RTU slave | 4 | 384 | Notification from UART idle-line/DMA ISR (0.5 s timeout) | Open commands → lock queue | Master expects a reply within its response timeout; t3.5 frame timing is done in the ISR/timer, not in the task |
| NFC reader | 3 | 384 | `xTaskDelayUntil` 150 ms poll + IRQ notification | Card events → network queue and Modbus register map | Human-scale latency; SPI bus mutex shared with external flash |
| LTE / network | 2 | 1536 | Own queue + modem UART stream buffer | Open commands → lock queue | Slow, long blocking AT/TLS operations, largest stack; must never delay lock or Modbus |
| Idle (+ tickless) | 0 | min | — | — | Frees deleted-task memory, sleeps |

Communication: `UART ISR → (notify) → Modbus → (queue) → Lock`; `NFC → (queue) → Network → (queue) → Lock`; `Lock → (queue) → Modbus` register map owned only by the Modbus task, so no mutex on it. ISRs that call the RTOS sit at NVIC priority ≥ 5; nothing time-critical depends on a software timer. All tasks check in to the supervisor; the lock task also forces the solenoid off on any fault path and at boot (fail-safe state does not depend on a task running).

## 14. Review checklist

- [ ] RTOS justified (section 1); each task has a deadline/blocking reason; tasks block on exactly one thing.
- [ ] Priorities follow rate/deadline-monotonic order; utilisation or response-time checked with measured `C`.
- [ ] Shared resources: owner task or mutex (never a binary semaphore as a lock); lock order documented; all takes have timeouts.
- [ ] Every ISR that calls the RTOS uses `FromISR` + `portYIELD_FROM_ISR` and has NVIC priority numerically ≥ the syscall limit; `NVIC_PRIORITYGROUP_4`; `configASSERT` enabled.
- [ ] Fixed-rate loops use `xTaskDelayUntil`; no blocking in timer callbacks; timer task priority/queue length sized; queue-send results checked.
- [ ] Static allocation or create-once dynamic; no malloc in ISRs; malloc-failed hook on.
- [ ] Stack sizes measured with high-water marks under worst-case paths; overflow check method 2 on; MPU/PSPLIM considered.
- [ ] newlib reentrancy/thread-safety configured, or all formatting in one task.
- [ ] Watchdog kicked only by a supervisor after all tasks check in; culprit logged.
- [ ] HAL timebase moved off SysTick (CubeMX); `configMAX_PRIORITIES` = 56 with the CMSIS-RTOS2 wrapper.
- [ ] Logic testable on the host without the RTOS; trace tool used at least once to confirm timing.

Parts adapted from mohitmishra786/low-level-dev-skills (MIT License).
