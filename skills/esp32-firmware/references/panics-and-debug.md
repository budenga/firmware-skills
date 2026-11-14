# ESP32 crashes, resets and debugging

## Contents
1. Get a decoded backtrace
2. Guru Meditation error types
3. Other panic/abort messages
4. Boot ROM messages and `rst:` codes
5. Boot loops and "won't start"
6. Upload / connection failures
7. Reset reason at runtime
8. JTAG debugging
9. Core dumps

---

## 1. Get a decoded backtrace

```ini
monitor_filters = esp32_exception_decoder
build_type = debug
```
Then `pio device monitor`. The filter turns `Backtrace: 0x400d1234:0x3ffb1f20 ...` into function + file:line. It needs the **same ELF** that is running on the board — rebuild/re-upload if in doubt.

Manual decode (Xtensa: ESP32/S2/S3):
```
~/.platformio/packages/toolchain-xtensa-esp32*/bin/xtensa-esp32*-elf-addr2line -pfiaC \
  -e .pio/build/<env>/firmware.elf 0x400d1234 0x400d5678
```
RISC-V chips (C3/C6) print a register dump with `MEPC` (faulting PC) and `RA` (return address): decode those two with `riscv32-esp-elf-addr2line`. Find the exact tool name: `ls ~/.platformio/packages/ | grep toolchain`.

Read the first frame in **your** code — frames inside FreeRTOS/IDF are usually victims, not causes.

## 2. Guru Meditation error types

| Message | Meaning | Typical cause |
|---|---|---|
| `LoadProhibited` / `Load access fault` (RISC-V) | Read from invalid address; see `EXCVADDR` / `MTVAL` | `EXCVADDR` ≈ 0x0–0x100 → **NULL pointer** (+ struct offset). Uninitialised pointer, deleted object, failed `malloc` not checked |
| `StoreProhibited` / `Store access fault` | Write to invalid address | Same as above, writing; buffer index out of range |
| `IllegalInstruction` | CPU executed garbage | Corrupted function pointer, stack overflow overwrote return address, calling a callback after the object was freed |
| `InstrFetchProhibited` / `Instruction access fault` | Jump to non-executable address | NULL/garbage function pointer (`PC` = 0 or odd value) |
| `LoadStoreAlignment` | Unaligned 32-bit access | Casting `uint8_t*` buffer to `uint32_t*` / packed structs |
| `IntegerDivideByZero` | Division by zero | Unchecked divisor |
| `Interrupt wdt timeout on CPU0/1` | Interrupts blocked too long | Long ISR, `portENTER_CRITICAL` held too long, spinning in critical section |
| `Cache disabled but cached memory region accessed` | ISR or data in flash used while flash cache off (during flash write) | Missing `IRAM_ATTR` on ISR or functions it calls; const data missing `DRAM_ATTR` |
| `Unhandled debug exception` + `Stack canary watchpoint triggered (taskname)` | **Stack overflow** in that task | Increase that task's stack (bytes), move big locals to static/heap, reduce recursion/printf of large buffers |

## 3. Other panic/abort messages

| Message | Cause |
|---|---|
| `abort() was called at PC 0x...` | Explicit abort: failed `ESP_ERROR_CHECK(...)` (the line above names the error, e.g. `ESP_ERR_NO_MEM`), C++ exception with exceptions disabled, `new` failing |
| `assert failed: <func> <file>:<line> (<expr>)` | IDF/FreeRTOS assertion — the expression tells what was violated (e.g. calling a FreeRTOS API from an ISR, mutex given by non-owner) |
| `E (..) task_wdt: Task watchdog got triggered. The following tasks did not reset the watchdog in time: - IDLE0 (CPU 0)` | Something on that core never yields (busy loop, long blocking call without delay) |
| `Brownout detector was triggered` | Supply drops below ~2.4–2.8 V — usually Wi-Fi TX current peaks. Better regulator (≥500 mA), 100–470 µF bulk capacitor near the module, short thick USB cable, avoid powering motors/LEDs from the same rail. **Don't disable the detector.** |
| `CORRUPT HEAP: Bad head at 0x...` / `heap != NULL && "free() target pointer is outside heap areas"` | Heap corruption: buffer overrun, double free, use-after-free. Enable `CONFIG_HEAP_POISONING_COMPREHENSIVE` (IDF) to catch it closer to the cause |

## 4. Boot ROM messages and `rst:` codes

First line after reset, e.g. `rst:0xc (SW_CPU_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)`.

Classic ESP32 `rst:` values (other chips use similar names, different numbers — read the name):

| Code | Name | Meaning |
|---|---|---|
| 0x1 | POWERON_RESET | Power-on / EN pin |
| 0x3 | SW_RESET | Software system reset (newer chips such as S3 show this for `esp_restart()`) |
| 0x5 | DEEPSLEEP_RESET | Wake from deep sleep |
| 0x7 / 0x8 | TG0WDT_SYS_RESET / TG1WDT_SYS_RESET | Timer-group watchdog (interrupt WDT / task WDT with panic) |
| 0x9 | RTCWDT_SYS_RESET | RTC watchdog |
| 0xc | SW_CPU_RESET | After a panic (normal "crash then reboot") or `esp_restart()` |
| 0xf | RTCWDT_BROWN_OUT_RESET | **Brown-out** |
| 0x10 | RTCWDT_RTC_RESET | RTC WDT during boot (boot stage hang) |

`boot:` value: `0x13`/`0x33` = normal flash boot; `0x3`/`DOWNLOAD_BOOT` + `waiting for download` = GPIO0 was LOW at reset (BOOT pressed or circuit pulling it low).

## 5. Boot loops and "won't start"

| Symptom | Check |
|---|---|
| `invalid header: 0xffffffff` repeating | Flash empty or app at wrong offset; re-upload full image (bootloader + partitions + app), check flash size/mode in `platformio.ini` |
| `flash read err, 1000` | **GPIO12 pulled high** (classic ESP32 → 1.8 V flash), or wrong flash mode/frequency |
| Stuck at `waiting for download` | GPIO0 (classic/S3) or GPIO9 (C3/C6) held LOW at reset |
| Reboots every few seconds with no panic | Brown-out (`rst:0xf`), or watchdog: read the `rst:` code |
| Crash in `setup()` before Serial output | Use `build_type = debug`, increase `CORE_DEBUG_LEVEL`, add early `Serial.begin` + prints; on native-USB boards the crash happens before USB enumerates → use UART0 pins with a USB-UART adapter or JTAG |
| Works on USB power, not on battery | Supply/brown-out, or code waiting for `Serial` (`while(!Serial)` without timeout on native-USB boards) |

## 6. Upload / connection failures

| Error | Fix |
|---|---|
| `Failed to connect to ESP32: Wrong boot mode detected` / `No serial data received` | Manual download mode: hold BOOT, tap EN/RST, release BOOT; then upload. Boards without auto-reset circuit always need this |
| `Could not open port … Resource busy` | Close other serial monitors (Arduino IDE, another `pio device monitor`, screen) |
| Port doesn't appear | Charge-only USB cable; missing driver for CP210x/CH340 on old macOS; native-USB board crashed early → BOOT+RST |
| `A fatal error occurred: MD5 of file does not match` / random failures | Lower `upload_speed` (460800 or 115200), better cable/hub |
| Upload OK, nothing runs, S3/C3 USB port changes after flashing | Normal when firmware switches USB mode; select the new port for the monitor |

## 7. Reset reason at runtime

```cpp
#include <esp_system.h>
const char *resetReasonStr(esp_reset_reason_t r) {
    switch (r) {
    case ESP_RST_POWERON:   return "power-on";
    case ESP_RST_SW:        return "software (esp_restart)";
    case ESP_RST_PANIC:     return "panic/exception";
    case ESP_RST_INT_WDT:   return "interrupt watchdog";
    case ESP_RST_TASK_WDT:  return "task watchdog";
    case ESP_RST_WDT:       return "other watchdog";
    case ESP_RST_DEEPSLEEP: return "deep sleep wake";
    case ESP_RST_BROWNOUT:  return "brownout";
    default:                return "other";
    }
}
// in setup(): log resetReasonStr(esp_reset_reason()) and a boot counter in NVS
```
Store the reason + counter so field units report how often and why they restart.

## 8. JTAG debugging

| Chip | Adapter | `platformio.ini` |
|---|---|---|
| ESP32-S3, C3, C6 (USB-Serial-JTAG) | Just the USB cable to the native USB port | `debug_tool = esp-builtin` |
| ESP32 classic | ESP-Prog (FT2232) on GPIO12–15 (MTDI 12, MTCK 13, MTMS 14, MTDO 15) | `debug_tool = esp-prog` |
| ESP32-S2 | ESP-Prog on GPIO39–42 (MTCK 39, MTDO 40, MTDI 41, MTMS 42) | `debug_tool = esp-prog` |

Then `pio debug --interface=gdb -x .pioinit`, or the VS Code PlatformIO debugger (F5). Set `build_type = debug`. The JTAG pins can't be used by the application while debugging. Hardware breakpoints are limited (2 on Xtensa).

## 9. Core dumps

Save the crash state to flash and analyse later (field units):
- Needs `sdkconfig` (ESP-IDF or Arduino+IDF): `CONFIG_ESP_COREDUMP_ENABLE_TO_FLASH=y`, `CONFIG_ESP_COREDUMP_DATA_FORMAT_ELF=y`, plus a `coredump` partition (64 KB is typical).
- Read it with `idf.py coredump-info` (ESP-IDF) or `esp-coredump info_corefile` (pip package `esp-coredump`) together with the matching ELF.
- Keep every released ELF (archive by version) — without it, backtraces and core dumps cannot be decoded.
