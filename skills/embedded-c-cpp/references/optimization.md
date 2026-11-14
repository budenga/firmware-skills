# Optimizing firmware: size, speed, power

## Contents
1. Measure first
2. Flash (code size)
3. RAM
4. Speed
5. Power
6. Optimization flags per toolchain

---

## 1. Measure first

Optimising without a measurement usually makes code harder to read and no faster. Measure, change one thing, measure again.

| What | How |
|---|---|
| Flash/RAM totals | build summary (`arm-none-eabi-size firmware.elf`, PlatformIO prints it, `idf.py size`, MPLAB X memory usage window / XC8 `--summary`) |
| Who uses flash/RAM | map file; `arm-none-eabi-nm --size-sort -S -C firmware.elf | tail -30`; `idf.py size-components`; `bloaty` on ELF files |
| Execution time | GPIO toggle + scope; Cortex-M3+ cycle counter (`DWT->CYCCNT`); `esp_timer_get_time()`; a free-running hardware timer on PIC |
| Stack | task high-water marks, stack painting |
| Current | multimeter for average, Nordic PPK2 / Joulescope / µCurrent / shunt + scope for profiles |

## 2. Flash (code size)

Biggest wins first:
1. **Linker garbage collection**: `-ffunction-sections -fdata-sections -Wl,--gc-sections` (GCC-based toolchains). Most vendor projects already set it — check.
2. **Optimise for size**: `-Os` (or `-Oz` on newer GCC/Clang). Microchip XC: older compiler versions in Free mode restrict optimisation levels; since July 2026 all levels are free on current versions — check version and licence mode in the build output.
3. **printf family**: full `printf` with float support costs 10–30 KB on ARM. Use newlib-nano (`--specs=nano.specs`), avoid `%f` (print scaled integers), or a small printf (e.g. mpaland/eyalroz printf).
4. **Floating point** on MCUs without an FPU (Cortex-M0/M0+/M3, PIC16/18/24, ESP32-C3) pulls in soft-float libraries — use fixed-point integers (`temp_c_x100`).
5. **Logging strings**: compile out debug logs in release (`LOG_LEVEL` macro), shorten messages.
6. **Duplicate generic code** from templates/macros in C++ — check `nm` for many near-identical instantiations.
7. **LTO** (`-flto`) often saves 5–15%; test thoroughly (it can remove "unused" ISR handlers or code reached only via linker sections — mark them `__attribute__((used))`).
8. Remove unused HAL modules (CubeMX: disable unused peripherals; HAL `_conf.h` module enables).

## 3. RAM

- `const` tables go to flash (ARM, ESP32 `.rodata` in flash, XC8 `const` → program memory). Check the map file that they actually did.
- Right-size task stacks using measured high-water marks (+ ~25% margin), not defaults.
- Right-size buffers to the real worst case; one shared scratch buffer instead of several large locals.
- Avoid large local arrays (stack) — make them `static` if not re-entrant-sensitive, or pass a buffer in.
- ESP32: move large buffers to PSRAM when present (`heap_caps_malloc(n, MALLOC_CAP_SPIRAM)`), keep DMA buffers in internal RAM.
- Packing structures (`__attribute__((packed))`) saves RAM but costs speed and can fault on unaligned access — order members large→small instead.

## 4. Speed

1. **Algorithm and data flow first**: avoid polling loops, copy less, use DMA for bulk transfers (UART, SPI, ADC, memory-to-memory).
2. **Interrupt latency**: keep ISRs short; set priorities so time-critical interrupts preempt others.
3. **Avoid hidden slow operations**: division on cores without hardware divide (Cortex-M0, PIC16/18 — use shifts by powers of 2), `double` instead of `float` (`-Wdouble-promotion`), software float on non-FPU cores, `printf` in hot paths.
4. **Memory placement**: hot code in RAM (`.ramfunc` on STM32, `IRAM_ATTR` on ESP32) when flash wait states or cache misses hurt; enable instruction cache / ART accelerator on STM32.
5. **Lookup tables** for expensive functions (sin, log, CRC) in flash.
6. **Compiler**: `-O2` for speed-critical files only, keep `-Os` elsewhere (per-file flags or `__attribute__((optimize("O2")))` on GCC).
7. **Volatile overuse** prevents optimisation — use it exactly where needed.

## 5. Power

- Sleep whenever idle: `WFI`/Stop/Standby on STM32, light/deep sleep on ESP32, `SLEEP()`/Idle/Doze on PIC. In an RTOS, use tickless idle.
- Lower the clock when full speed isn't needed; batch work then sleep longer ("race to sleep").
- Disable unused peripherals and their clocks; configure unused pins as analog/inputs with defined level (floating inputs draw current).
- Radios dominate on ESP32: Wi-Fi modem sleep / power-save modes, BLE advertising interval, connect less often.
- Watch external parts: LEDs, pull-up resistors held low, sensors left powered, LDO quiescent current.
- Measure the full cycle profile (wake → work → sleep), not just sleep current.

## 6. Optimization flags per toolchain

| Toolchain | Debug build | Release build |
|---|---|---|
| arm-none-eabi-gcc (STM32) | `-Og -g3` | `-Os -g` (keep `-g`: debug info doesn't go into flash) |
| ESP-IDF | `CONFIG_COMPILER_OPTIMIZATION_DEBUG` | `CONFIG_COMPILER_OPTIMIZATION_SIZE` (default for many apps) or `_PERF` |
| PlatformIO | `build_type = debug` | `build_type = release` (default), extra flags via `build_flags` / `build_unflags` |
| XC8 | `-O0`/`-O1` | `-Os` or `-O2` (older Free-mode versions: up to `-O2`) |
| XC16 / XC32 | `-O0`/`-O1` | `-Os` or `-O2` (older Free-mode versions: up to `-O1`) |

Always check the final release build on hardware: optimisation exposes missing `volatile` and timing assumptions.
