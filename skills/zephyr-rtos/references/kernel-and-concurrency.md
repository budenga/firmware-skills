# Kernel and concurrency

## Contents
1. Threads and priorities
2. Choosing the right primitive
3. Workqueues and delayable work
4. Timers
5. Semaphores, mutexes, events
6. Message queues, FIFOs and zbus
7. Interrupts
8. Time, ticks and sleeping
9. Stack sizing and protection
10. Worked example — locker task layout
11. Review checklist

---

## 1. Threads and priorities

- **Lower number = higher priority.** Negative priorities are **cooperative** (never preempted by another thread, only by ISRs; they run until they block or yield). Zero and positive are **preemptible**.
- Defaults: `CONFIG_NUM_COOP_PRIORITIES=16` (−16…−1), `CONFIG_NUM_PREEMPT_PRIORITIES=15` (0…14), `main()` at priority 0 (`CONFIG_MAIN_THREAD_PRIORITY`), system workqueue at −1 (lowest cooperative).
- Use preemptible priorities for application threads. Cooperative only for short, latency-critical sections that must not be interrupted by other threads (and never with long loops).
- Time slicing among equal-priority preemptible threads: `CONFIG_TIMESLICING=y`, `CONFIG_TIMESLICE_SIZE` (ms; default 20 from 4.2, 0 = no limit, i.e. no slicing, on 3.7–4.1 — set it explicitly).

```c
#define RS485_STACK 1536
#define RS485_PRIO  5
static void rs485_thread(void *p1, void *p2, void *p3);
K_THREAD_DEFINE(rs485_tid, RS485_STACK, rs485_thread, NULL, NULL, NULL,
                RS485_PRIO, 0, 0);   /* options, start delay ms */
```

Runtime creation: `K_THREAD_STACK_DEFINE(stack, size)` + `k_thread_create(&thr, stack, K_THREAD_STACK_SIZEOF(stack), entry, p1, p2, p3, prio, 0, K_NO_WAIT)`; name it with `k_thread_name_set()` (`CONFIG_THREAD_NAME=y`) so the debugger and thread analyzer show it.

`main()` may return; the system keeps running. Many apps just initialise in `main()` and return.

## 2. Choosing the right primitive

| Need | Use | Not |
|---|---|---|
| Defer work from ISR, short non-blocking jobs | `k_work` on system workqueue | A thread per job |
| Slow or blocking deferred work (flash, modem AT, network) | `k_work` on a **dedicated** `k_work_q` | System workqueue |
| Periodic or one-shot timeout with work in thread context | `k_work_delayable` | `k_timer` handler doing real work |
| Precise periodic tick, handler trivial | `k_timer` (expiry runs in ISR) | |
| Signal "something happened" (count) | `k_sem` | Busy flag polling |
| Exclusive access to a resource, from threads | `k_mutex` (priority inheritance, recursive) | `k_sem` used as a lock |
| Up to 32 flags, wait for any/all | `k_event` | Several semaphores |
| Fixed-size messages by copy, ISR producer OK | `k_msgq` | |
| Pass pointers to buffers you own, zero-copy | `k_fifo` / `k_lifo` | `k_msgq` with huge items |
| Publish/subscribe between modules | zbus | Global variables + flags |
| Byte stream | `ring_buf` (`sys/ring_buffer.h`) or `k_pipe` | |

## 3. Workqueues and delayable work

```c
static void door_poll_fn(struct k_work *work);
static K_WORK_DELAYABLE_DEFINE(door_poll, door_poll_fn);

static void door_poll_fn(struct k_work *work)
{
    /* runs in workqueue thread context: may take mutexes, log, call drivers */
    read_doors();
    k_work_schedule(&door_poll, K_MSEC(50));   /* re-arm */
}
/* start: k_work_schedule(&door_poll, K_NO_WAIT); stop: k_work_cancel_delayable_sync(&door_poll, &sync); */
```

- `k_work_schedule()` does nothing if already scheduled; `k_work_reschedule()` restarts the delay (debounce pattern).
- Recover the context in the handler with `CONTAINER_OF(k_work_delayable_from_work(work), struct my_ctx, dwork)`.
- Dedicated queue for blocking work:

```c
K_THREAD_STACK_DEFINE(slow_wq_stack, 2048);
static struct k_work_q slow_wq;
k_work_queue_start(&slow_wq, slow_wq_stack, K_THREAD_STACK_SIZEOF(slow_wq_stack), 8, NULL);
k_work_submit_to_queue(&slow_wq, &flash_save_work);
```

- System workqueue stack is `CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE` (default 1024). Bluetooth, sensors, MCUmgr and logging back-ends share it; one blocking handler delays them all.

## 4. Timers

```c
static void tick_fn(struct k_timer *t) { k_sem_give(&tick_sem); }   /* ISR context */
K_TIMER_DEFINE(tick, tick_fn, NULL);
k_timer_start(&tick, K_MSEC(10), K_MSEC(10));   /* first expiry, period */
```

Expiry functions run in the system clock ISR: only ISR-safe calls (`k_sem_give`, `k_work_submit`, `k_msgq_put(..., K_NO_WAIT)`, `k_event_post`). Alternative: no expiry function, and a thread calls `k_timer_status_sync()`.

## 5. Semaphores, mutexes, events

- `k_sem_give()` is ISR-safe; `k_sem_take()` from ISR only with `K_NO_WAIT`.
- `k_mutex` cannot be used from ISRs, must be unlocked by the owner thread, is recursive and applies **priority inheritance** (bounded by `CONFIG_PRIORITY_CEILING`). Use it for shared buses or state structs; keep critical sections short.
- Very short data protection also against ISRs: `k_spinlock` (`k_spin_lock`/`k_spin_unlock`) — on single core it masks interrupts; do not sleep inside.
- `k_event`: `k_event_post()` (OR in), `k_event_set()` (overwrite), `k_event_wait(&ev, mask, reset, timeout)`, `k_event_wait_all()`, `k_event_clear()`. Good for "door 3 opened | power fail | NFC card" style flags.

## 6. Message queues, FIFOs and zbus

```c
struct lock_cmd { uint8_t door; uint8_t action; uint16_t pulse_ms; };
K_MSGQ_DEFINE(lock_q, sizeof(struct lock_cmd), 16, 4);

/* producer (thread or ISR) */
if (k_msgq_put(&lock_q, &cmd, K_NO_WAIT) != 0) { /* full: count and report */ }
/* consumer */
k_msgq_get(&lock_q, &cmd, K_FOREVER);
```

- `k_fifo`: items are your structs whose **first word is reserved** for the kernel (`void *fifo_reserved;`). The item must stay valid until consumed (static, slab or heap, not the producer's stack).
- zbus (`CONFIG_ZBUS=y`): typed channels, listeners (synchronous callback in the publisher's context), subscribers (thread gets a notification and reads the channel), message subscribers (get a copy).

```c
struct door_evt { uint8_t door; bool open; };
ZBUS_CHAN_DEFINE(door_chan, struct door_evt, NULL, NULL,
                 ZBUS_OBSERVERS(logger_lis, backend_sub), ZBUS_MSG_INIT(0));
zbus_chan_pub(&door_chan, &evt, K_MSEC(10));
```

Listeners must be quick and non-blocking. Check the zbus docs for the version in use before using newer features (async listeners, runtime observers).

## 7. Interrupts

- Driver-based code rarely needs `IRQ_CONNECT`; drivers install their own ISRs and give callbacks (GPIO, UART, CAN). Callbacks from drivers run in **ISR context** unless documented otherwise.
- Custom ISR (e.g., a timer peripheral not covered by a driver):

```c
#define MY_IRQ  DT_IRQN(DT_NODELABEL(timers7))
#define MY_PRIO DT_IRQ(DT_NODELABEL(timers7), priority)
IRQ_CONNECT(MY_IRQ, MY_PRIO, my_isr, NULL, 0);   /* compile-time constants */
irq_enable(MY_IRQ);
```

- `IRQ_DIRECT_CONNECT` for minimal-overhead ISRs (no scheduling decision unless you call `ISR_DIRECT_PM()`/return 1); zero-latency IRQs (`CONFIG_ZERO_LATENCY_IRQS`, flag `IRQ_ZERO_LATENCY`) are never masked by the kernel and **must not call any kernel API**.
- In ISRs: only `K_NO_WAIT`; no mutexes, no `LOG_*` in immediate mode, no blocking driver calls. `k_is_in_isr()` tells you where you are.
- On Cortex-M, the kernel reserves priority levels (`CONFIG_ZERO_LATENCY_LEVELS`, PendSV/SVC); use DT priorities given by the board rather than raw NVIC numbers.

## 8. Time, ticks and sleeping

- `k_sleep(K_MSEC(10))`, `k_msleep(10)`, `k_usleep(100)` yield the CPU; timeouts are rounded **up** to ticks.
- `CONFIG_SYS_CLOCK_TICKS_PER_SEC`: default 10000 with tickless kernel, 32768 on most nRF (RTC-based), 31250 with nRF GRTC (nRF54); check `.config`. Tickless (`CONFIG_TICKLESS_KERNEL`, default on most SoCs) means no periodic tick when idle.
- `k_busy_wait(us)` spins: only for sub-tick delays (RS-485 turnaround, sensor start-up of a few µs).
- Timestamps: `k_uptime_get()` (ms, int64), `k_cycle_get_32()` / `k_cyc_to_ns_floor64()` for short intervals.
- Absolute deadlines for periodic loops: `k_timepoint_t`, `sys_timepoint_calc()`, or `K_TIMEOUT_ABS_MS()` (requires `CONFIG_TIMEOUT_64BIT`) to avoid drift.

## 9. Stack sizing and protection

- Development config:

```
CONFIG_THREAD_NAME=y
CONFIG_THREAD_STACK_INFO=y
CONFIG_INIT_STACKS=y
CONFIG_THREAD_ANALYZER=y
CONFIG_THREAD_ANALYZER_AUTO=y
CONFIG_THREAD_ANALYZER_AUTO_INTERVAL=30
CONFIG_HW_STACK_PROTECTION=y        # MPU guard or PSPLIM, arch picks
CONFIG_STACK_SENTINEL=y             # software check when no MPU (costs time)
```

- Read the analyzer output after exercising worst-case paths (all doors, modem reconnect, flash GC, logging bursts). Leave 25–30 % margin. Shell: `kernel thread stacks` (4.0+) / `kernel stacks` (3.7).
- `thread_analyzer_print()` takes a CPU index argument from 4.0 (`thread_analyzer_print(0)`), none on 3.7.
- Common stacks to size: `CONFIG_MAIN_STACK_SIZE` (1024), `CONFIG_ISR_STACK_SIZE` (2048), `CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE`, `CONFIG_LOG_PROCESS_THREAD_STACK_SIZE`, `CONFIG_SHELL_STACK_SIZE`, net/BT stacks. `printf` with floats (`CONFIG_CBPRINTF_FP_SUPPORT`) and mbedTLS are stack-hungry.
- `CONFIG_STACK_CANARIES` adds compiler stack-smashing protection (cost: size and speed).

## 10. Worked example — locker task layout

| Thread / context | Priority | Job |
|---|---|---|
| GPIO ISR callbacks (doors, tamper) | ISR | `k_event_post()` + timestamp, nothing else |
| `lock_ctrl` thread | 2 | Consumes `lock_q`, drives solenoid with PWM/GPIO and a timeout, verifies door sensor |
| Modbus server (subsystem: system workqueue, or its own with `CONFIG_MODBUS_WORKQUEUE` on 4.5+) | default | Register callbacks copy values under a mutex, post commands to `lock_q` |
| `slow_wq` dedicated workqueue | 8 | Settings save, event log to flash |
| Modem / network | per subsystem | LTE/Ethernet, MQTT |
| System workqueue | −1 | Short items only |
| `main` | 0 | Init, then returns |

Rule: hardware actions (solenoid on/off) have one owner thread; everybody else sends it commands. This gives a single place for pulse-length limits and thermal duty-cycle checks.

## 11. Review checklist

- [ ] Every thread has a name, a measured stack size with margin and a justified priority.
- [ ] No blocking calls in ISRs, driver callbacks, `k_timer` expiry functions or zbus listeners.
- [ ] Blocking deferred work runs on a dedicated workqueue, not the system one.
- [ ] Mutexes (not semaphores) protect shared resources; no mutex taken from ISR.
- [ ] Every `K_FOREVER` wait is justified; others have timeouts and handle `-EAGAIN`.
- [ ] Message-queue full and FIFO allocation failures counted and reported.
- [ ] HW stack protection on in all builds; thread analyzer used before release.
- [ ] Periodic loops use absolute deadlines or delayable work, not `k_sleep` drift.
