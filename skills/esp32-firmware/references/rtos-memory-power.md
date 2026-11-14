# FreeRTOS, memory and power on ESP32

## Contents
1. Tasks and cores
2. Interrupts
3. Sharing data
4. Watchdogs
5. Memory: heap, stack, PSRAM, IRAM
6. Sleep and battery

---

## 1. Tasks and cores

- Arduino: `setup()`/`loop()` run in `loopTask` (core 1 on dual-core chips, priority 1, 8 KB stack). Wi-Fi/BT stacks run on core 0.
- Create tasks for independent activities:
  ```cpp
  static void sensorTask(void *arg) {
      for (;;) {
          readSensor();
          vTaskDelay(pdMS_TO_TICKS(100));      // always yield
      }
  }
  void setup() {
      // ARDUINO_RUNNING_CORE is 1 on dual-core chips, 0 on single-core (S2/C3/C6)
      xTaskCreatePinnedToCore(sensorTask, "sensor", 4096 /* bytes */, nullptr, 2, nullptr, ARDUINO_RUNNING_CORE);
  }
  ```
- **Stack sizes are bytes** in ESP-IDF/Arduino. Start with 3–4 KB for simple tasks, 6–8 KB if using `printf`/`Serial.printf`/JSON/TLS; measure `uxTaskGetStackHighWaterMark(NULL)` (bytes free, minimum seen) and trim.
- Pinning: leave core 0 to Wi-Fi/BT for radio-heavy apps; put time-critical work on core 1. **On single-core chips (S2, C3, C6) a core ID of 1 aborts at boot** ("Invalid core ID") — use `xTaskCreate`, `tskNO_AFFINITY`, or `ARDUINO_RUNNING_CORE` so the same code runs on every variant.
- Priorities: higher number = higher priority. A high-priority task that never blocks starves everything below it, including the idle task (→ task watchdog).
- Prefer `vTaskDelayUntil` for fixed-rate loops (no drift).

## 2. Interrupts

```cpp
static QueueHandle_t s_evtQueue;

void IRAM_ATTR onButton() {
    uint32_t now = (uint32_t)esp_timer_get_time();
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(s_evtQueue, &now, &woken);
    if (woken) portYIELD_FROM_ISR();
}

void setup() {
    s_evtQueue = xQueueCreate(16, sizeof(uint32_t));
    pinMode(4, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(4), onButton, FALLING);
}
```
- `IRAM_ATTR` on the ISR and on every function it calls: during flash writes (NVS, OTA, filesystem) the flash cache is disabled and an ISR in flash crashes ("Cache disabled but cached memory region accessed").
- Constant tables used in ISRs: `DRAM_ATTR`.
- No `Serial`, `delay`, `malloc`, Wi-Fi or `ESP_LOGx` in ISRs (`ESP_DRAM_LOGx` exists for emergencies).
- Mechanical buttons bounce: debounce in the task (ignore events < 30 ms apart), not in the ISR.
- For periodic high-rate work prefer `esp_timer` callbacks or hardware timers + task notification.

## 3. Sharing data

- Prefer queues (copy data) and task notifications over shared globals.
- Shared variable between ISR and task on a dual-core chip: a spinlock critical section, not just `noInterrupts()` (that only masks the current core):
  ```cpp
  static portMUX_TYPE s_mux = portMUX_INITIALIZER_UNLOCKED;
  static volatile uint32_t s_count;
  void IRAM_ATTR isr() { portENTER_CRITICAL_ISR(&s_mux); s_count++; portEXIT_CRITICAL_ISR(&s_mux); }
  uint32_t readCount()  { portENTER_CRITICAL(&s_mux); uint32_t c = s_count; portEXIT_CRITICAL(&s_mux); return c; }
  ```
- Between tasks: mutex (`xSemaphoreCreateMutex`) around shared peripherals (I2C bus used by two tasks), or one owner task.
- Arduino `Wire`, `SPI`, `Serial` objects are not designed to be used from several tasks at once — give each bus one owner.

## 4. Watchdogs

| Watchdog | Fires when | Message |
|---|---|---|
| Task WDT | Idle task (or a subscribed task) doesn't run for the timeout (default 5 s) | `E (…) task_wdt: Task watchdog got triggered` + task names |
| Interrupt WDT | An ISR runs too long or interrupts are disabled too long | `Guru Meditation Error: Core 0 panic'ed (Interrupt wdt timeout on CPU0)` |
| RTC WDT | Boot stage hangs | `rst:0x10 (RTCWDT_RTC_RESET)` |

Fix the blocking code; don't just lengthen or disable the watchdog. For your own long-running tasks, subscribe them (`esp_task_wdt_add(NULL)`, then `esp_task_wdt_reset()` in the loop) so a hang in them is detected too.

## 5. Memory: heap, stack, PSRAM, IRAM

- Monitor: `ESP.getFreeHeap()`, `ESP.getMinFreeHeap()`, `ESP.getMaxAllocHeap()` (largest block — fragmentation indicator); IDF: `heap_caps_get_free_size(MALLOC_CAP_8BIT)`, `heap_caps_get_minimum_free_size(...)`, `heap_caps_get_largest_free_block(...)`, `heap_caps_print_heap_info(MALLOC_CAP_DEFAULT)`.
- Log these periodically during soak tests; a steadily dropping minimum = leak, a dropping largest block with stable free = fragmentation.
- Avoid Arduino `String` building in loops — use `snprintf` into fixed buffers, or `reserve()`.
- Large buffers (images, display frame buffers, audio) → PSRAM: `heap_caps_malloc(n, MALLOC_CAP_SPIRAM)`. **DMA buffers must be internal RAM**: `heap_caps_malloc(n, MALLOC_CAP_DMA)` (S3 can DMA from PSRAM only with specific drivers/config).
- TLS (HTTPS/MQTTS) needs ~40–50 KB heap per connection; plan for it.
- IRAM is small; only ISRs and truly time-critical code go there.
- Size analysis: PlatformIO prints RAM/Flash at the end of the build; for details use `.pio/build/<env>/firmware.map`, or with ESP-IDF `idf.py size-components` (in PlatformIO: `pip install esp-idf-size` then `python -m esp_idf_size .pio/build/<env>/firmware.map`).

## 6. Sleep and battery

| Mode | What runs | Wake | Typical current (chip only) |
|---|---|---|---|
| Active, Wi-Fi TX | everything | — | 100–500 mA peaks |
| Modem sleep | CPU on, radio off between beacons | — | ~20–40 mA |
| Light sleep | CPU paused, RAM kept | timer, GPIO, UART | ~0.1–1 mA |
| Deep sleep | RTC/LP domain only, RAM lost (except RTC memory) | timer, RTC/LP GPIO, touch, ULP/LP core | ~5–20 µA |

Deep sleep pattern (Arduino):
```cpp
RTC_DATA_ATTR uint32_t bootCount = 0;          // survives deep sleep (not power loss)

void setup() {
    bootCount++;
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    doMeasurementAndSend();                    // keep the awake time short
    esp_sleep_enable_timer_wakeup(15ULL * 60 * 1000000);   // µs
    esp_deep_sleep_start();                    // never returns; wake = reset → setup()
}
void loop() {}
```
- Wake pins must be RTC/LP-capable for the variant (see `pins-and-variants.md`). Classic ESP32: `esp_sleep_enable_ext0_wakeup(GPIO_NUM_33, 0)`; C3/C6: `esp_deep_sleep_enable_gpio_wakeup(...)`.
- Hold output levels through sleep with `gpio_hold_en()` + `gpio_deep_sleep_hold_en()` where needed.
- **Dev boards rarely reach datasheet deep-sleep current**: USB-UART chip, LDO quiescent current, power LED. Measure on the final hardware.
- Reduce awake time: connect Wi-Fi with static IP and saved channel/BSSID, send, sleep. Each Wi-Fi association costs ~1–3 s at ~100 mA.
- ESP-IDF power management (`CONFIG_PM_ENABLE` + `esp_pm_configure` with light sleep enabled) gives automatic light sleep between tasks — needs ESP-IDF or Arduino+IDF.
