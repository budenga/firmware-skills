---
name: stm32-firmware
description: STM32 firmware development with STM32CubeMX / STM32CubeIDE + HAL/LL and with PlatformIO — project structure and USER CODE rules, peripherals (UART/I2C/SPI/ADC/timers/DMA), clocks, low power, flash/option bytes, building from the command line, flashing and debugging with ST-Link (STM32CubeProgrammer CLI, OpenOCD, GDB, SWV), and HardFault analysis. Use this skill whenever the user works on any STM32 / Nucleo / Discovery / Blue Pill board, a `.ioc` file, CubeMX/CubeIDE, HAL_ functions, ST-Link, or sees a HardFault — even if they just say "o meu STM32 não arranca" or paste a HAL call. Use together with embedded-c-cpp for general C/C++ quality, testing and architecture.
---

# STM32 firmware (CubeMX/HAL + PlatformIO, ST-Link)

Also load `embedded-c-cpp` for coding standards, architecture, unit testing and general debugging. This skill holds the STM32-specific parts.

## First, identify the project

Read the repo before writing code. These facts change the answer:

| Find | Where |
|---|---|
| Exact part (e.g. STM32G431KBT6) and family (F0/F1/F4/G0/G4/L4/H7/U5...) | `.ioc` (`Mcu.UserName`), `platformio.ini` `board =`, startup file name |
| Project type | `.ioc` + `.project`/`.cproject` → CubeIDE; `.ioc` + `CMakeLists.txt`/`CMakePresets.json` → CubeMX CMake; `platformio.ini` → PlatformIO |
| HAL vs LL vs registers | `Drivers/STM32xx_HAL_Driver`, `#include "stm32xx_ll_*.h"`, CubeMX "Advanced settings" per peripheral |
| RTOS | `Middlewares/Third_Party/FreeRTOS`, `cmsis_os.h` |
| Clocks | `SystemClock_Config()` in `main.c`, `.ioc` RCC section; Nucleo boards usually feed HSE 8 MHz from the ST-Link (HSE **bypass**) |
| Probe | ST-Link (on-board on Nucleo/Discovery, or ST-Link V2/V3 stand-alone) |

Family matters a lot: cores (M0+/M3/M4F/M7/M33), FPU or not, flash write granularity (half-word on F1, double-word on G4/L4, flash-word on H7), DMA (DMAMUX on G0/G4/L4+/H7), caches (F7/H7). When unsure about a register or peripheral detail, say what to check in the reference manual (RM) instead of guessing.

## The CubeMX rules that prevent lost work

CubeMX regenerates `main.c`, `stm32xx_it.c`, `stm32xx_hal_msp.c` and init files every time. Only text between `/* USER CODE BEGIN x */` and `/* USER CODE END x */` survives.

- Put application logic in your own files (`App/app.c`, `App/app.h`, or `app.cpp` for C++) and call `app_init()` / `app_loop()` from `USER CODE BEGIN 2` and the `while (1)` USER CODE block.
- Change pins, clocks, DMA, interrupts and peripheral parameters **in the `.ioc`**, then regenerate — not by editing `MX_xxx_Init()` (it gets overwritten, and the `.ioc` stops matching the code).
- Commit the `.ioc` file. It is the source of truth for hardware configuration.
- Enable "Generate peripheral initialization as a pair of .c/.h files per peripheral" in Project Manager → Code Generator: smaller `main.c`, cleaner diffs.
- Details and C++ integration: `references/cubemx-hal.md`.

## Building, flashing, debugging

Pick the path matching the project; full commands in `references/build-flash-debug.md`.

| Project | Build | Flash (ST-Link) | Debug |
|---|---|---|---|
| CubeIDE | IDE, or headless `stm32cubeide ... -build` | IDE, or `STM32_Programmer_CLI -c port=SWD -w app.elf -v -rst` | CubeIDE debugger (ST-LINK GDB server) |
| CubeMX CMake (VS Code) | `cmake --preset Debug && cmake --build --preset Debug` | STM32_Programmer_CLI or OpenOCD | VS Code STM32 extension / OpenOCD + GDB |
| PlatformIO | `pio run -e <env>` | `pio run -e <env> -t upload` (`upload_protocol = stlink`) | `pio debug` / VS Code (`debug_tool = stlink`) |

Flashing changes the board: confirm with the user which board/probe is connected before running an upload, and never run mass erase, option-byte or read-protection commands without explicit confirmation (RDP level 2 is permanent).

## HAL usage — the mistakes that cost the most time

- **I2C addresses** are passed left-shifted: `HAL_I2C_Master_Transmit(&hi2c1, 0x76 << 1, ...)`.
- **HAL callbacks run in interrupt context** (`HAL_UART_RxCpltCallback`, `HAL_GPIO_EXTI_Callback`, `HAL_TIM_PeriodElapsedCallback`): keep them short, no `HAL_Delay`, no blocking transmits.
- **`HAL_Delay` / `HAL_GetTick` need SysTick**: calling them from a higher-priority ISR or with interrupts disabled hangs. With FreeRTOS, set SYS → Timebase Source to a basic timer (TIM6/TIM7) in CubeMX.
- **UART reception**: use `HAL_UARTEx_ReceiveToIdle_DMA` (variable-length frames) or interrupt mode into a ring buffer; restart reception in the callback and in `HAL_UART_ErrorCallback` (an overrun stops reception silently).
- **DMA buffers on F7/H7** (data cache): clean before TX, invalidate after RX, 32-byte aligned and sized; on H7, DMA1/2 cannot reach DTCM RAM — place buffers in D2/AXI SRAM.
- **Check the return value** of every HAL call (`HAL_OK`, `HAL_ERROR`, `HAL_BUSY`, `HAL_TIMEOUT`); a `HAL_BUSY` usually means a previous transfer never finished.
- **Timeouts**: never `HAL_MAX_DELAY` in production paths.
- More patterns (UART DMA, ADC DMA, timers/PWM, EXTI, low power, flash writes, watchdog, reset cause): `references/cubemx-hal.md`.

## When it crashes or won't start

- HardFault / MemManage / BusFault / UsageFault → `references/faults-and-recovery.md` (read CFSR/HFSR, stacked PC, decode with `addr2line`).
- ST-Link can't connect ("No target found", "DEV_TARGET_HELD_UNDER_RESET") → connect under reset; check SWD pins were not reconfigured as GPIO in the `.ioc`; BOOT0; RDP level → `references/faults-and-recovery.md`.
- Hangs in `SystemClock_Config` → HSE not present/bypass mismatch, wrong PLL, flash latency too low.
- Peripheral silent → clock enabled? pin AF correct? `MX_xxx_Init` actually called and in the right order (DMA init before the peripheral using it)?

## Personal library (optional)

If the user has a personal technical library indexed locally (a `library-researcher` agent is available), delegate to it for STM32 reference-manual chapters, ST application notes, HAL/LL and timer/DMA/low-power techniques that go beyond this skill — it answers with document and page citations. Treat older books as background and prefer current vendor documentation for API/toolchain details. Skip if no library is configured.

## Reference files

| File | Read when |
|---|---|
| `references/cubemx-hal.md` | Writing/reviewing code in a CubeMX project, C++ integration, peripheral patterns, low power, flash/EEPROM emulation, watchdog, reset cause |
| `references/build-flash-debug.md` | Toolchain install, building from CLI, PlatformIO setup for STM32, flashing, GDB/OpenOCD, printf/SWV logging, size analysis |
| `references/faults-and-recovery.md` | HardFault and other faults, board won't connect, option bytes and read protection, recovering a bricked-looking board |
