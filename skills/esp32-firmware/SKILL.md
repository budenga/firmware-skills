---
name: esp32-firmware
description: ESP32 firmware with PlatformIO — Arduino framework (primary) and ESP-IDF (also inside PlatformIO, or Arduino as an IDF component) — for ESP32, S2, S3, C3, C6. Covers platformio.ini setup, Arduino core 2.x vs 3.x API differences, safe pin selection (strapping/flash/PSRAM/ADC2 pins), FreeRTOS tasks and ISRs, memory/PSRAM, Wi-Fi/BLE, NVS/Preferences, OTA, deep sleep, partitions, flashing over USB/serial, decoding Guru Meditation panics and boot loops. Use this skill whenever the user works on any ESP32 / ESP32-S3 / C3 / C6 board, platformio.ini with espressif32, Arduino sketches for ESP32, idf.py/sdkconfig, or pastes ESP32 serial output (rst:, Guru Meditation, Brownout, watchdog) — even if they only say "a placa reinicia" or "o upload falha". Use together with embedded-c-cpp for general C/C++ quality, testing and architecture.
---

# ESP32 firmware (PlatformIO: Arduino first, ESP-IDF when needed)

Also load `embedded-c-cpp` for coding standards, architecture, unit testing and general debugging. This skill holds the ESP32-specific parts.

## First, identify the project

| Find | Where | Why it matters |
|---|---|---|
| **Exact chip and module** (ESP32-WROOM-32E, ESP32-S3-WROOM-1-N16R8, ESP32-C3-MINI-1...) | `platformio.ini` `board =`, user, board silkscreen | Cores, pins, PSRAM, USB, ADC differ a lot between variants |
| Framework | `framework = arduino`, `espidf`, or `arduino, espidf` | API and configuration model |
| **Arduino core version** (2.x or 3.x) | build log line `framework-arduinoespressif32 @ x.y.z`, `platform =` line, or `ESP_ARDUINO_VERSION_MAJOR` | Several APIs changed in 3.x (LEDC, timers) — code written for the wrong one won't compile |
| Flash / PSRAM size | module name (N16R8 = 16 MB flash, 8 MB octal PSRAM), `board_upload.flash_size` | Partitions, memory placement |
| Board wiring | user/schematic | Avoid strapping pins and pins used by flash/PSRAM/USB/LEDs |

If the variant is unknown and the task touches pins, peripherals, power or memory, find out before writing code — a pin that is fine on an S3 can stop a classic ESP32 from booting.

## Arduino or ESP-IDF?

| Stay with **Arduino** (default) | Move to **ESP-IDF** (in PlatformIO: `framework = espidf`) or **Arduino + IDF** (`framework = arduino, espidf`) |
|---|---|
| Prototypes and most products | Need to change `sdkconfig` options Arduino's precompiled core doesn't expose (rollback OTA, secure boot, flash encryption, PM auto light-sleep, core-dump-to-flash, FreeRTOS settings) |
| Lots of libraries available | Latest IDF features, tight control of memory/performance |
| Fast iteration | Production hardening |

Arduino on ESP32 already runs on FreeRTOS + ESP-IDF, so IDF APIs (`esp_*`, `xTaskCreatePinnedToCore`, `nvs_*`, `esp_sleep_*`, `heap_caps_*`) work from Arduino code. "Arduino as IDF component" (`arduino, espidf`) keeps Arduino libraries and adds `sdkconfig` control — the usual migration path.

Setup, `platformio.ini` templates and the 2.x vs 3.x API changes: `references/platformio-setup.md`.

## Rules that prevent the classic ESP32 failures

1. **Pins**: never use flash pins (GPIO6–11 on ESP32), PSRAM pins (16/17 on WROVER; 33–37 on S3 octal-PSRAM modules) or USB pins when using USB; treat strapping pins (ESP32: 0, 2, 5, 12, 15) as "only with care" — **GPIO12 pulled high at boot makes 3.3 V-flash modules fail to boot**. ADC2 does not work while Wi-Fi is on (classic ESP32): use ADC1 pins. Full tables per variant: `references/pins-and-variants.md`.
2. **Don't block `loop()` or a task forever**: long `while` loops without `delay()`/`vTaskDelay()` trigger the task watchdog; `while (WiFi.status() != WL_CONNECTED)` needs a timeout.
3. **ISRs**: mark `IRAM_ATTR`, keep them tiny, no `Serial.print`, no `delay`, no Wi-Fi calls; hand work to a task (queue / task notification / flag). Shared variables `volatile` + critical section (`portENTER_CRITICAL_ISR` spinlock — the ESP32 is dual-core).
4. **Task stacks are in bytes** in ESP-IDF/Arduino (unlike vanilla FreeRTOS words). Measure with `uxTaskGetStackHighWaterMark(NULL)` (bytes). Arduino `loop()` has 8 KB by default (`SET_LOOP_TASK_STACK_SIZE(16 * 1024);` to change).
5. **Power supply**: Wi-Fi TX draws ~300–500 mA peaks. "Brownout detector was triggered" means fix the supply (regulator, bulk capacitor, cable) — do not disable the brownout detector.
6. **Heap**: avoid `String` concatenation in loops and frequent `new`/`malloc` (fragmentation over days); preallocate. Check `ESP.getFreeHeap()` / `heap_caps_get_minimum_free_size(MALLOC_CAP_8BIT)` over time.
7. **NVS/flash wear**: don't write `Preferences`/NVS on every loop iteration; write on change, with rate limiting.
8. **Irreversible operations**: eFuse burning, secure boot, flash encryption in release mode are permanent. Never run `espefuse.py burn_*`, enable these in `sdkconfig`, or `erase_flash` a device with data, without explicit user confirmation.

## Build, upload, monitor (PlatformIO)

```
pio run -e <env>                       # build
pio run -e <env> -t upload             # flash over USB/serial
pio device monitor -e <env>            # serial monitor (uses monitor_speed / monitor_filters)
pio run -e <env> -t upload -t monitor  # both
pio run -e <env> -t uploadfs           # upload data/ folder as LittleFS/SPIFFS image
pio run -e <env> -t erase              # erase whole flash (asks first: wipes NVS/WiFi creds)
pio run -e <env> -t menuconfig         # ESP-IDF / arduino+espidf only
pio test -e native                     # host unit tests (see embedded-c-cpp)
```
Upload fails with "Failed to connect / Wrong boot mode detected" → hold **BOOT** (GPIO0 / GPIO9 on C3), tap **EN/RST**, release BOOT; check the cable carries data; close any other serial monitor holding the port. Confirm the board is connected and which port before uploading (`pio device list`).

## When it crashes or reboots

Paste-able serial output is the key evidence. Enable `monitor_filters = esp32_exception_decoder` and `build_type = debug` so backtraces are decoded to file:line. Then see `references/panics-and-debug.md` for: Guru Meditation types (LoadProhibited, StoreProhibited, IllegalInstruction, Load/Store access fault on RISC-V), stack canary, task/interrupt watchdog, `rst:` codes, brownout, boot loops, JTAG debugging (`debug_tool = esp-builtin` on S3/C3/C6).

## Reference files

| File | Read when |
|---|---|
| `references/platformio-setup.md` | Creating/fixing `platformio.ini`, choosing platform/core version, Arduino 2.x→3.x API changes, PSRAM, USB-CDC serial, partitions & filesystem, using ESP-IDF in PlatformIO, sdkconfig |
| `references/pins-and-variants.md` | Choosing or reviewing pins, comparing ESP32/S2/S3/C3/C6, ADC, touch, deep-sleep wake pins |
| `references/rtos-memory-power.md` | Tasks, cores, ISRs, queues, watchdog, stack/heap/PSRAM, IRAM, deep/light sleep, battery |
| `references/panics-and-debug.md` | Any crash, reset, watchdog, brownout, boot loop, upload failure, JTAG debug, core dumps, reset reason |
| `references/connectivity-ota-security.md` | Wi-Fi/BLE robustness, NVS/Preferences, OTA updates and rollback, secure boot / flash encryption decisions |
