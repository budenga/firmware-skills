# firmware-skills

Agent skills and subagents for writing, testing, reviewing, debugging, flashing and optimizing **microcontroller firmware in C/C++** with industry-grade practices. Installable as a Claude Code plugin, or skills-only for any agent that supports the [skills.sh](https://skills.sh) `SKILL.md` format.

| Skill | Covers |
|---|---|
| [`embedded-c-cpp`](skills/embedded-c-cpp/SKILL.md) | Cross-platform base: MISRA C / CERT C / Barr rules, embedded C++ subset, firmware architecture (layers, state machines, ISRs, concurrency), RTOS design (FreeRTOS/CMSIS-RTOS2, priorities, ISR→task, stacks, watchdog supervision), startup code, linker scripts and bootloaders, host unit tests with Unity, GDB/OpenOCD and ELF/map analysis, static analysis and sanitizers, CI, systematic debugging, size/speed/power optimization |
| [`stm32-firmware`](skills/stm32-firmware/SKILL.md) | STM32CubeMX/CubeIDE + HAL/LL and PlatformIO, USER CODE rules, peripheral patterns, low power, flash/option bytes, CLI builds, ST-Link flashing and debugging (CubeProgrammer CLI, OpenOCD, GDB, SWV/RTT), HardFault analysis |
| [`esp32-firmware`](skills/esp32-firmware/SKILL.md) | ESP32 / S2 / S3 / C3 / C6 with PlatformIO — Arduino framework first, ESP-IDF and Arduino-as-IDF-component — core 2.x vs 3.x APIs, safe pin selection, FreeRTOS, memory/PSRAM, deep sleep, Wi-Fi, OTA, panic decoding |
| [`pic-firmware`](skills/pic-firmware/SKILL.md) | Microchip PIC16 / PIC18 / PIC24 / dsPIC33 / PIC32 with MPLAB X, XC8 / XC16 / XC-DSC / XC32 and MCC — config bits, ANSEL/LAT/PPS, interrupts per family, dsPIC DSP engine and control loops, PICkit / Snap / ICD programming |
| [`zephyr-rtos`](skills/zephyr-rtos/SKILL.md) | Zephyr 3.7 LTS / 4.x on STM32, ESP32 and nRF — west and sysbuild, board targets, devicetree and Kconfig, kernel objects, drivers and subsystems (UART, CAN, Modbus, sensors, networking, cellular, BLE, PM), logging/shell, fatal errors and debugging, MCUboot and MCUmgr DFU, Twister/ztest, TF-M, migration from bare-metal/FreeRTOS |
| [`embedded-linux`](skills/embedded-linux/SKILL.md) | Embedded Linux on Toradex Verdin i.MX8M (Torizon OS), Raspberry Pi 4/5 and CM4/CM5, Yocto, Buildroot — boot chain and recovery, device tree and overlays, probe failures, kernel drivers, userspace hardware access (libgpiod, i2c-dev, spidev, RS-485, SocketCAN), cross-builds and containers, debugging and profiling, OTA, read-only rootfs, flash wear |

The chip skills are meant to be used **together with `embedded-c-cpp`**, which holds the shared rules.

## Agents

Subagents for self-contained tasks that benefit from an independent view or a clean context. They load the skills above for their knowledge.

| Agent | Does | Tools |
|---|---|---|
| [`firmware-reviewer`](agents/firmware-reviewer.md) | Independent review of firmware changes before commit/flash: hardware-damage risks, ISR/concurrency, memory safety, timeouts, arithmetic, code-generator regions | Read-only |
| [`firmware-crash-analyst`](agents/firmware-crash-analyst.md) | Root cause of crashes/resets from logs and fault dumps; decodes addresses against the ELF | Read + analysis commands |
| [`firmware-test-writer`](agents/firmware-test-writer.md) | Writes and runs host unit tests (Unity, `pio test -e native` or CMake); reports production bugs instead of silently fixing them | Edits test files only |
| [`datasheet-researcher`](agents/datasheet-researcher.md) | Exact hardware facts from datasheets, reference manuals and errata, with page/section citations | Read + web |
| [`architecture-reviewer`](agents/architecture-reviewer.md) | Critique of a firmware design: execution model, timing/memory budgets, failure modes, updates, power, security, testability | Read-only |
| [`linux-bringup-debugger`](agents/linux-bringup-debugger.md) | Diagnoses embedded-Linux bring-up and runtime problems from boot logs, dmesg, device trees, command output and container configs (Torizon, Raspberry Pi, Yocto); ranked causes with the command that proves each | Read + analysis commands |

No agent flashes devices or changes irreversible settings — that stays with the main session and the user's confirmation.

## Install

### As a Claude Code plugin (skills + agents)

In Claude Code:

```
/plugin marketplace add budenga/firmware-skills
/plugin install firmware-skills@firmware-skills
```

Or from a shell:

```bash
claude plugin marketplace add budenga/firmware-skills
claude plugin install firmware-skills@firmware-skills
```

Plugin components are namespaced: skills appear as `firmware-skills:esp32-firmware`, agents as `firmware-skills:firmware-reviewer`.

### Skills only, with the skills CLI

Works with Claude Code and other agents; does not install the agents.

All skills, globally, for Claude Code:

```bash
npx skills@latest add budenga/firmware-skills -g -a claude-code
```

Only some:

```bash
npx skills@latest add budenga/firmware-skills -g -a claude-code -s embedded-c-cpp -s esp32-firmware
```

Update later:

```bash
npx skills@latest update -g
```

Or copy the folders from `skills/` into `~/.claude/skills/` (global) or `.claude/skills/` (one project), and the files from `agents/` into `~/.claude/agents/`.

Use one method, not both — installing the plugin and the standalone skills loads each skill twice.

## Safety

The skills tell the agent to ask before flashing, erasing, or changing irreversible settings (STM32 RDP level 2, ESP32 eFuses / secure boot / flash encryption, PIC code protection). Firmware guidance is checked against vendor documentation, but always verify pinouts and register details against the datasheet for your exact part.

## Contributing

Each skill is a folder with a `SKILL.md` (frontmatter `name` + `description`, then instructions) and optional `references/` files loaded on demand. Corrections with a source (datasheet, reference manual, vendor docs) are welcome.

## License

[MIT](LICENSE). Parts of `embedded-linux`, `zephyr-rtos` and some `embedded-c-cpp` references were adapted from [mohitmishra786/low-level-dev-skills](https://github.com/mohitmishra786/low-level-dev-skills) (MIT) — see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
