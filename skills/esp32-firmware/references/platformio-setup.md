# PlatformIO setup for ESP32

## Contents
1. Platform and core versions
2. `platformio.ini` templates (Arduino)
3. Arduino core 2.x → 3.x API changes
4. PSRAM, flash size, partitions, filesystem
5. Native USB serial (S2/S3/C3/C6)
6. ESP-IDF in PlatformIO
7. Arduino as an ESP-IDF component
8. Libraries and reproducibility

---

## 1. Platform and core versions

- `platform = espressif32` (PlatformIO registry) has historically shipped **Arduino core 2.x** (based on ESP-IDF 4.4).
- **Arduino core 3.x** (ESP-IDF 5.x, needed for ESP32-C6/H2/P4 and newer APIs) is commonly used through the community platform **pioarduino**:
  ```ini
  platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
  ```
  Check the current state of both projects before choosing — this changes over time.
- Pin versions in real projects (`platform = espressif32 @ 6.9.0`, or a specific pioarduino release URL) so a rebuild next year produces the same firmware.
- See what you actually got: the build log prints `framework-arduinoespressif32 @ x.y.z`; in code, `ESP_ARDUINO_VERSION_MAJOR`.

## 2. `platformio.ini` templates (Arduino)

```ini
[platformio]
default_envs = s3-debug

[env]                                   ; shared by all envs
platform = espressif32                  ; or pioarduino URL for core 3.x
framework = arduino
monitor_speed = 115200
monitor_filters = esp32_exception_decoder, time
upload_speed = 921600
build_flags =
    -Wall -Wextra
    -DCORE_DEBUG_LEVEL=3                ; 0 none … 5 verbose (ESP_LOGx / log_x output)
lib_deps =
    bblanchon/ArduinoJson @ ^7.0.0

[env:s3-debug]
board = esp32-s3-devkitc-1
build_type = debug                      ; symbols for the exception decoder / JTAG
debug_tool = esp-builtin                ; built-in USB-JTAG on S3/C3/C6

[env:s3-release]
board = esp32-s3-devkitc-1
build_flags =
    ${env.build_flags}
    -DCORE_DEBUG_LEVEL=1

[env:classic]
board = esp32dev                        ; ESP32-WROOM-32 dev boards
debug_tool = esp-prog                   ; needs an external ESP-Prog JTAG adapter

[env:native]                            ; host unit tests (see embedded-c-cpp testing.md)
platform = native
framework =
test_framework = unity
```
Find the board id: `pio boards esp32` (pick the one matching your module's flash/PSRAM, or override as in §4).

## 3. Arduino core 2.x → 3.x API changes

The ones that break most sketches:

| Feature | Core 2.x | Core 3.x |
|---|---|---|
| PWM (LEDC) | `ledcSetup(ch, freq, bits); ledcAttachPin(pin, ch); ledcWrite(ch, duty);` | `ledcAttach(pin, freq, bits); ledcWrite(pin, duty);` (by **pin**) |
| Hardware timer | `t = timerBegin(0, 80, true); timerAttachInterrupt(t, &isr, true); timerAlarmWrite(t, 1000000, true); timerAlarmEnable(t);` | `t = timerBegin(1000000); /* Hz */ timerAttachInterrupt(t, &isr); timerAlarm(t, 1000000, true, 0);` |
| Task watchdog init | `esp_task_wdt_init(timeout_s, panic)` | `esp_task_wdt_config_t cfg = {...}; esp_task_wdt_init(&cfg);` (IDF 5) |
| ESP-IDF underneath | 4.4 | 5.x — many `esp_*`/driver headers renamed (e.g. new ADC oneshot, new RMT, new I2C master driver) |

When the user gives code, check which version it targets before "fixing" it. When writing new code, write for the version the project uses and say which.

## 4. PSRAM, flash size, partitions, filesystem

```ini
[env:s3-n16r8]
board = esp32-s3-devkitc-1
board_upload.flash_size = 16MB
board_upload.maximum_size = 16777216
board_build.partitions = default_16MB.csv     ; or a custom partitions.csv in the project root
board_build.arduino.memory_type = qio_opi     ; S3 with octal PSRAM (R8 modules); qio_qspi for quad PSRAM
build_flags =
    ${env.build_flags}
    -DBOARD_HAS_PSRAM
```
- Check PSRAM at runtime: `psramFound()`, `ESP.getPsramSize()`; allocate with `ps_malloc()` / `heap_caps_malloc(n, MALLOC_CAP_SPIRAM)`.
- Built-in partition tables (Arduino core): `default.csv`, `min_spiffs.csv` (bigger app, OTA), `huge_app.csv` (no OTA), `default_16MB.csv`, ... An app too big for its partition fails at link/upload — pick a table with a bigger app slot rather than removing OTA blindly.
- Custom table example (4 MB, OTA + LittleFS):
  ```
  # Name,   Type, SubType, Offset,   Size
  nvs,      data, nvs,     0x9000,   0x5000
  otadata,  data, ota,     0xe000,   0x2000
  app0,     app,  ota_0,   0x10000,  0x180000
  app1,     app,  ota_1,   0x190000, 0x180000
  spiffs,   data, spiffs,  0x310000, 0xE0000
  coredump, data, coredump,0x3F0000, 0x10000
  ```
  (`spiffs` subtype is also used for LittleFS.) App partitions must be 64 KB aligned.
- Filesystem: `board_build.filesystem = littlefs`; put files in `data/`; `pio run -t uploadfs`.

## 5. Native USB serial (S2/S3/C3/C6)

Boards using the chip's own USB port (no USB-UART chip) need:
```ini
build_flags =
    -DARDUINO_USB_CDC_ON_BOOT=1   ; Serial = USB CDC
    -DARDUINO_USB_MODE=1          ; S3: 1 = HW CDC/JTAG (keeps JTAG), 0 = TinyUSB
```
Many board definitions already set these — check before adding. With USB CDC, wait for the host briefly: `Serial.begin(115200); uint32_t t0 = millis(); while (!Serial && millis() - t0 < 3000) {}`. If the firmware crashes early, the USB port disappears; enter download mode with BOOT+RST to reflash.

## 6. ESP-IDF in PlatformIO

```ini
[env:c6-idf]
platform = espressif32          ; check the platform supports your chip/IDF version
board = esp32-c6-devkitc-1
framework = espidf
monitor_speed = 115200
monitor_filters = esp32_exception_decoder
```
Required files:
```cmake
# CMakeLists.txt (project root)
cmake_minimum_required(VERSION 3.16.0)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```
```cmake
# src/CMakeLists.txt
FILE(GLOB_RECURSE app_sources ${CMAKE_SOURCE_DIR}/src/*.*)
idf_component_register(SRCS ${app_sources})
```
- Entry point is `void app_main(void)` (wrap in `extern "C"` in C++).
- Configuration: `pio run -t menuconfig` writes `sdkconfig.<env>`. Put the options you rely on in **`sdkconfig.defaults`** (committed) so a fresh checkout reproduces them; prefer editing that file over describing menu clicks.
- Extra ESP-IDF components: `idf_component.yml` in `src/` (component manager) or folders in `components/`.

## 7. Arduino as an ESP-IDF component

```ini
[env:s3-arduino-idf]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino, espidf
```
`sdkconfig.defaults` must contain at least:
```
CONFIG_FREERTOS_HZ=1000
CONFIG_AUTOSTART_ARDUINO=y
```
Keeps `setup()`/`loop()` and Arduino libraries while unlocking `sdkconfig` (rollback OTA, PM, core dumps...). Builds are slower (whole IDF compiled).

## 8. Libraries and reproducibility

- Declare every library in `lib_deps` with a version (`owner/Name @ ^x.y.z`), never rely on globally installed ones.
- Check library compatibility with the core version (many libs broke with core 3.x).
- Commit `platformio.ini`, `sdkconfig.defaults`, partition CSVs; ignore `.pio/`.
- `pio pkg update` only on purpose, then retest.
