# Build, flash, debug (ST-Link)

## Contents
1. Tools to install (macOS)
2. Build from the command line
3. PlatformIO for STM32
4. Flash with STM32CubeProgrammer CLI
5. Flash/debug with OpenOCD + GDB
6. Logging: UART, SWV/ITM
7. Size and memory analysis

---

## 1. Tools to install (macOS)

| Tool | Get it | Notes |
|---|---|---|
| STM32CubeMX | st.com (free account) | Recent CubeIDE versions ship without integrated CubeMX — install the stand-alone CubeMX |
| STM32CubeIDE | st.com | Eclipse-based; includes GCC, ST-LINK GDB server, CubeProgrammer CLI |
| STM32CubeProgrammer | st.com | GUI + `STM32_Programmer_CLI` |
| Arm GNU toolchain | `brew install --cask gcc-arm-embedded` or developer.arm.com | `arm-none-eabi-gcc`, `-gdb`, `-size`, `-addr2line`, `-nm` |
| OpenOCD | `brew install open-ocd` | Optional, open-source alternative to ST's GDB server |
| VS Code + "STM32Cube for VS Code" extension | Marketplace | Works with CubeMX CMake projects |
| PlatformIO | VS Code extension or `pip install platformio` | |

Find the CLI if it is not on PATH:
```
find /Applications -name STM32_Programmer_CLI -type f 2>/dev/null
```

## 2. Build from the command line

**CubeMX CMake project** (Project Manager → Toolchain/IDE = CMake):
```
cmake --preset Debug
cmake --build --preset Debug          # → build/Debug/<project>.elf
```
Add your sources/includes in the top-level `CMakeLists.txt` user section, not inside `cmake/stm32cubemx/` (regenerated).

**CubeIDE project, headless** (CI or terminal):
```
/Applications/STM32CubeIDE.app/Contents/MacOS/stm32cubeide \
  --launcher.suppressErrors -nosplash \
  -application org.eclipse.cdt.managedbuilder.core.headlessbuild \
  -data /tmp/cubeide-ws -import "$PWD" -build "<ProjectName>/Debug"
```
Output: `Debug/<ProjectName>.elf`. The exact path to the executable can differ per version; adjust.

## 3. PlatformIO for STM32

```ini
[env:nucleo_g431rb]
platform = ststm32
board = nucleo_g431rb
framework = stm32cube          ; or arduino (STM32duino)
upload_protocol = stlink
debug_tool = stlink
monitor_speed = 115200
build_flags = -Wall -Wextra
```
Commands: `pio run`, `pio run -t upload`, `pio device monitor`, `pio debug`, `pio test -e native`, `pio check`.

Using CubeMX-generated code in PlatformIO:
- `framework = stm32cube` brings PlatformIO's own copy of the HAL. Copy your `Core/` user files into `src/`/`include/` and keep the `.ioc` only as configuration reference — the HAL version may differ from CubeMX's.
- To use your own `stm32xx_hal_conf.h`, set `board_build.stm32cube.custom_config_header = yes` (check the PlatformIO ststm32 docs for the current option names).
- If the project must stay 1:1 with CubeMX (same HAL version, startup file, linker script), prefer the CubeMX CMake/CubeIDE build over PlatformIO.

`framework = arduino` (STM32duino) is good for prototypes; HAL calls still work underneath (`HAL_` functions are available), but timers/interrupts used by the core can conflict with your own use.

## 4. Flash with STM32CubeProgrammer CLI

```
STM32_Programmer_CLI -l                                   # list connected ST-Links
STM32_Programmer_CLI -c port=SWD -w build/Debug/app.elf -v -rst
STM32_Programmer_CLI -c port=SWD -w app.bin 0x08000000 -v -rst   # .bin needs an address
STM32_Programmer_CLI -c port=SWD mode=UR -w app.elf -v -rst      # connect under reset
STM32_Programmer_CLI -c port=SWD -ob displ                # show option bytes (read-only)
```
Destructive — only with the user's explicit OK:
```
STM32_Programmer_CLI -c port=SWD -e all                   # mass erase
STM32_Programmer_CLI -c port=SWD -ob RDP=0xAA             # remove RDP level 1 → triggers mass erase (most families)
STM32_Programmer_CLI -c port=SWD -ob RDP=0xA5             # same on STM32F1 (Blue Pill): level 0 is 0xA5 there — 0xAA would KEEP level 1
```
Always run `-ob displ` first and use the RDP value the tool shows for "level 0" on that family. STM32H5 uses "product state" instead of RDP; check the family's reference manual.
Several ST-Links connected: add `sn=<serial>` to `-c`.

## 5. Flash/debug with OpenOCD + GDB

```
openocd -f interface/stlink.cfg -f target/stm32g4x.cfg \
        -c "program build/Debug/app.elf verify reset exit"
```
Target config names: `stm32f0x.cfg`, `stm32f1x.cfg`, `stm32f4x.cfg`, `stm32g0x.cfg`, `stm32g4x.cfg`, `stm32l4x.cfg`, `stm32h7x.cfg`, ... (list: `ls $(brew --prefix)/share/openocd/scripts/target/ | grep stm32`).

Debug session:
```
openocd -f interface/stlink.cfg -f target/stm32g4x.cfg          # terminal 1 (GDB server on :3333)
arm-none-eabi-gdb build/Debug/app.elf                            # terminal 2
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break app_init
(gdb) continue
(gdb) watch g_state            # stop when it is written (finds corruption)
(gdb) bt                       # backtrace
(gdb) info registers
(gdb) x/8wx $sp                # raw stack
```
Clone ST-Links (cheap "ST-Link V2" dongles) often work with OpenOCD but not always with ST tools; they also lack SWO.

## 6. Logging: UART, SWV/ITM

- **UART** via the ST-Link virtual COM port (Nucleo) — simplest; see `cubemx-hal.md` §3.
- **SWV/ITM printf** (Cortex-M3/M4/M7/M33 with SWO pin, genuine ST-Link V2-1/V3): no extra UART needed.
  ```c
  int __io_putchar(int ch) { ITM_SendChar((uint32_t)ch); return ch; }
  ```
  In CubeIDE: Debug configuration → Debugger → enable Serial Wire Viewer with the correct core clock; open SWV ITM Data Console, enable port 0, start trace. SWO is pin PB3 on most STM32 — do not use it as GPIO.
- **SEGGER RTT** is faster than UART and needs no extra pin. Works with J-Link, and also with a stock ST-Link through OpenOCD ≥ 0.11 (`rtt setup` / `rtt start` / `rtt server start`), pyOCD or probe-rs. Add the SEGGER RTT sources to the project.

## 7. Size and memory analysis

```
arm-none-eabi-size build/Debug/app.elf                     # text = flash code, data = flash+RAM init, bss = RAM
arm-none-eabi-nm --size-sort -S -C build/Debug/app.elf | tail -30   # biggest symbols
arm-none-eabi-addr2line -e build/Debug/app.elf -f -C 0x08001234     # address → function:line
```
Add `-Wl,--print-memory-usage` to the linker flags for a per-region summary in every build. CubeIDE also has the "Build Analyzer" view (memory regions and per-symbol sizes from the map file).
