# Firmware architecture

## Contents
1. Layers
2. Superloop vs RTOS
3. ISR ↔ main/task communication
4. State machines
5. Error handling and resilience
6. Timing without blocking

---

## 1. Layers

```
app/        product behaviour, state machines         → portable, unit-tested on PC
services/   protocols, logging, storage, scheduling   → mostly portable
drivers/    sensor/actuator drivers (BME280, motor)    → depend on a bus interface, not on a chip
bsp/        board: pins, buses, clocks for THIS board  → chip + board specific
vendor/     HAL / ESP-IDF / MCC generated code         → never edited by hand (or only in allowed regions)
```

Dependencies point **down only**. The key seam is the interface between drivers and the bus:

```c
/* i2c_bus.h — what a sensor driver needs, nothing more */
typedef struct {
    int (*write_read)(void *ctx, uint8_t addr,
                      const uint8_t *tx, size_t tx_len,
                      uint8_t *rx, size_t rx_len, uint32_t timeout_ms);
    void *ctx;
} i2c_bus_t;
```

The BSP implements it with HAL/ESP-IDF/MCC; tests implement it with a fake. The BME280 driver then compiles and is tested on a PC, and moves from STM32 to ESP32 to PIC32 unchanged.

## 2. Superloop vs RTOS

| Superloop (+ interrupts) | RTOS (FreeRTOS, Zephyr) |
|---|---|
| PIC16/18, small STM32, simple products | ESP32 (always — ESP-IDF and Arduino run on FreeRTOS), connectivity stacks, many independent activities |
| Everything non-blocking, cooperative | Tasks may block on queues/semaphores |
| Easy to reason about timing | Need to think about priorities, stack per task, priority inversion |

Superloop pattern:
```c
for (;;) {
    process_uart_rx();     /* each returns quickly, never blocks */
    sensor_fsm_step();
    ui_fsm_step();
    watchdog_kick();
    sleep_until_interrupt();  /* optional: WFI / SLEEP() */
}
```

RTOS rules of thumb:
- One task per independent activity with its own timing, not one task per function.
- Communicate with queues (copy data) rather than shared globals + mutexes.
- Give each task a measured stack size (check the high-water mark), not a guess.
- Only one task owns each peripheral; others send it requests.
- Avoid equal-priority busy loops; something that polls must `vTaskDelay`.

## 3. ISR ↔ main/task communication

Options, from simplest:

1. **Flag**: `static volatile bool s_rx_ready;` set in ISR, cleared in main. Fine for single events.
2. **Single-producer/single-consumer ring buffer**: ISR writes `head`, main reads `tail`. No lock needed if indices are atomic-sized for the CPU (8-bit on PIC16/18!) and written once per operation.
   ```c
   #define RB_SIZE 64u                       /* power of two */
   static volatile uint8_t rb_buf[RB_SIZE];
   static volatile uint8_t rb_head, rb_tail; /* uint8_t: atomic on every MCU */

   void uart_rx_isr(uint8_t byte) {          /* producer */
       uint8_t next = (uint8_t)((rb_head + 1u) & (RB_SIZE - 1u));
       if (next != rb_tail) { rb_buf[rb_head] = byte; rb_head = next; }
       else { rb_overflow++; }               /* count, don't block */
   }
   bool rb_pop(uint8_t *out) {               /* consumer */
       if (rb_tail == rb_head) return false;
       *out = rb_buf[rb_tail];
       rb_tail = (uint8_t)((rb_tail + 1u) & (RB_SIZE - 1u));
       return true;
   }
   ```
   On multi-core (ESP32 dual core) or with compiler reordering concerns, add memory barriers or use the RTOS queue instead.
3. **RTOS queue / task notification** from the ISR using the `FromISR` API variants, then request a context switch (`portYIELD_FROM_ISR`).
4. **DMA + half/full-complete interrupts** for streams (UART, ADC, I2S): the ISR only tells the task which half is ready.

Critical sections must be short (microseconds). Prefer disabling only the one interrupt source involved over disabling all interrupts.

## 4. State machines

Use explicit state machines for anything with modes, protocols or sequences — they make behaviour reviewable and testable.

```c
typedef enum { ST_IDLE, ST_MEASURE, ST_WAIT_RESULT, ST_ERROR } sensor_state_t;

void sensor_fsm_step(sensor_ctx_t *c, uint32_t now_ms) {
    switch (c->state) {
    case ST_IDLE:
        if (now_ms - c->last_ms >= PERIOD_MS) { start_conversion(c); c->t0 = now_ms; c->state = ST_WAIT_RESULT; }
        break;
    case ST_WAIT_RESULT:
        if (conversion_done(c))            { read_result(c); c->last_ms = now_ms; c->state = ST_IDLE; }
        else if (now_ms - c->t0 > TIMEOUT_MS) { c->errors++; c->state = ST_ERROR; }
        break;
    case ST_ERROR:
        recover_bus(c); c->state = ST_IDLE;
        break;
    default:
        c->state = ST_ERROR;               /* corrupted state → recover, never ignore */
        break;
    }
}
```
- Time comparisons as `now - start >= period` with unsigned types survive counter wrap-around; `now >= start + period` does not.
- Pass time in (`now_ms`) instead of reading a global clock inside — makes the FSM testable.
- For complex products, use hierarchical state machines or an active-object framework (e.g. QP/C) rather than nested `if`s.

## 5. Error handling and resilience

- Return status codes from every fallible function; reserve `assert` for programming errors (things that should be impossible), with an assert handler that logs file/line and resets (in production) or halts (in debug).
- Classify errors: *recoverable* (retry with back-off, reinit peripheral), *degraded* (keep running without the feature, report), *fatal* (log and reset).
- At boot: read and log the **reset cause**, a **boot counter**, and the **firmware version** (embed `git describe` at build time). Store the last fault info in a no-init RAM section or flash so it survives the reset.
- Independent watchdog fed from the main loop or a supervisor task that checks all other tasks are alive — not from a timer interrupt (that keeps a hung app alive).
- Enable brown-out detection; flash writes during brown-out corrupt data.
- Validate everything that comes from outside (UART frames, BLE, network, sensor values out of physical range).
- Persistent data: version + CRC on every stored structure; two copies or a journal if power can fail mid-write.

## 6. Timing without blocking

- Replace `delay(500)` in loops with time-stamped checks or RTOS delays — a blocking delay in a superloop starves everything else.
- Microsecond timing: hardware timers, input capture, or the CPU cycle counter, not software loops (they change with optimisation level).
- Debounce buttons in software (sample every 5–10 ms, require N stable samples) or with a timer, not by delaying in the ISR.
