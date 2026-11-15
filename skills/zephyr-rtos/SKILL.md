---
name: zephyr-rtos
description: Zephyr RTOS application development (3.7 LTS and 4.x) on STM32, ESP32 and Nordic nRF52/53/54 — west workspaces and manifests, HWMv2 board targets, sysbuild, prj.conf and overlays, devicetree (DT_ macros, *_dt_spec, pinctrl, __device_dts_ord errors), Kconfig, kernel (threads, workqueues, timers, k_msgq, mutexes, zbus, ISRs, stack sizing), drivers (GPIO, UART, I2C, SPI, ADC, PWM, CAN, Modbus RTU over RS-485, watchdog, flash, NVS/ZMS/settings), networking, LTE modems, BLE, USB, power management, logging and shell, fault decoding, MCUboot and MCUmgr DFU, TF-M, Twister/ztest/native_sim, nRF Connect SDK, and migration from bare-metal or FreeRTOS. Use it whenever a project has west.yml, prj.conf, a .overlay or the user says "Zephyr", "west build", "devicetree", "Kconfig", "nRF Connect", "o device não está pronto", "migrar do FreeRTOS para Zephyr", "sysbuild" or "MCUboot".
---

# Zephyr RTOS (west, devicetree, Kconfig, kernel, MCUboot)

Zephyr is configured, not hand-initialised. Hardware lives in devicetree, features in Kconfig, and both are resolved at build time; most "it does not work" reports are a node that is disabled, a driver that is not enabled or a Kconfig symbol that silently did not set. Read the generated outputs before changing code.

Also load `embedded-c-cpp` for coding standards, architecture and unit testing. For the electrical side of RS-485, CAN or USB load a bus/interface hardware skill (such as `comms-interfaces`) if installed. `stm32-firmware`, `esp32-firmware` hold the CubeMX/PlatformIO way of working; this skill replaces them when the project is Zephyr.

## First, identify the project

| Find | Where | Why it matters |
|---|---|---|
| Zephyr version | `zephyr/VERSION`, `west list zephyr`, `west.yml` revision | 3.7 LTS and 4.x differ in board names, MCUboot modes, partitions, SPI macros, NVS paths |
| Vendor SDK on top | `west.yml` imports `sdk-nrf` (nRF Connect SDK) or plain upstream | NCS has its own versions, Partition Manager, extra Kconfig |
| Topology | Is the app the manifest repo (T2), or a folder inside a workspace (freestanding)? | Decides how you pin versions and add modules |
| Board target | `west build -b ...` history, `build/zephyr/.config` `CONFIG_BOARD_TARGET`, `CMakeCache.txt` `BOARD` | HWMv2 qualifiers (`nrf5340dk/nrf5340/cpuapp/ns`) select core and security domain |
| Sysbuild | `sysbuild.conf`, `sysbuild/` folder, `west config build.sysbuild` | MCUboot, multi-core and TF-M images are built by sysbuild |
| Toolchain | `ZEPHYR_TOOLCHAIN_VARIANT`, Zephyr SDK version | 4.4+ needs Zephyr SDK 1.0 or newer; 3.7 ships with SDK 0.16.x |
| Build system | `west` CLI, VS Code (nRF Connect extension) or PlatformIO `framework = zephyr` | PlatformIO lags upstream by releases and does not cover sysbuild well |

When a version detail matters and the version is unknown, ask or state the assumption ("assuming Zephyr 4.x upstream").

## Rules that prevent the classic failures

1. **Hardware goes in devicetree, features in Kconfig, never in code.** Pins, baud rates, bus speeds, partitions and DE pins belong in an overlay; the driver reads them. Hard-coded pins break on the next board revision and bypass pinctrl.
2. **Get devices with `DEVICE_DT_GET` or a `*_dt_spec`, and check `device_is_ready()` / `*_is_ready_dt()` before first use.** Init failures do not stop boot; an unready device used anyway gives confusing faults later.
3. **Read `build/zephyr/zephyr.dts` and `build/zephyr/.config` before debugging code.** They are the final truth. Overlays and `prj.conf` lines that "do nothing" are visible there in seconds.
4. **Kconfig dependencies are silent.** `CONFIG_X=y` with an unmet `depends on` ends up `n` with a warning that scrolls away. Check the symbol in `.config` and in menuconfig (it shows the unmet dependency).
5. **Pristine-build after changing board, overlays, `CMakeLists.txt` or sysbuild files** (`west build -p always`). Devicetree and Kconfig outputs are generated at CMake configure time and stale build folders lie.
6. **Never block in ISRs, `k_timer` expiry functions or the system workqueue.** Timer expiry runs in interrupt context; a blocking work item stalls every subsystem that shares the system workqueue (Bluetooth, sensors, logging back-ends). Use a dedicated workqueue for slow work.
7. **Size stacks from measurement.** Turn on `CONFIG_THREAD_ANALYZER` and hardware stack protection during development; overflow in Zephyr corrupts a neighbour before it faults.
8. **Pin every west project to a tag or SHA in the manifest.** `revision: main` makes the build depend on the day; reproducible firmware needs a frozen manifest (`west manifest --freeze`).
9. **MCUboot images must be signed with your own key and confirmed by the app.** The default keys in the MCUboot repo are public. An unconfirmed image in a test swap reverts on the next reset, which looks like "the update did not take".
10. **Ask before flashing, erasing, `--recover` on Nordic parts, provisioning keys, or writing option bytes / eFuses.** `west flash --recover` mass-erases nRF parts including UICR (that is also how a locked APPROTECT is cleared, losing all data); ERASEPROTECT on nRF53/nRF91 (blocks `--recover` unless the firmware implements the unlock handshake), STM32 RDP level 2 and ESP32 eFuses (secure boot, flash encryption) are permanent.

## Workflow by task

**New project**: pick the version (3.7 LTS for long-lived products with few new-feature needs; latest 4.x for new SoCs such as nRF54L or newer drivers) → workspace and app skeleton → build a sample for the exact board target → add overlays and Kconfig → bring up one peripheral at a time. See `references/build-west-devicetree-kconfig.md`.

**Device not ready / `__device_dts_ord_N` link error**: find node N in `devicetree_generated.h` → check `status` in `zephyr.dts` → check the driver Kconfig is `y` in `.config` → check `compatible` has a driver → pristine build. Worked example in `references/build-west-devicetree-kconfig.md`.

**Threads, timing, queues, ISRs**: `references/kernel-and-concurrency.md`.

**A peripheral or subsystem (UART, CAN, Modbus, flash storage, network, LTE, BLE, USB, PM)**: `references/drivers-and-subsystems.md`; Modbus RTU worked example is there.

**Crash, hang, logging, size, tests**: `references/debug-logging-testing.md`.

**Bootloader, OTA/DFU, secure boot, TF-M**: `references/mcuboot-dfu-security.md`.

**Coming from bare metal, CubeMX or FreeRTOS, or using NCS / ESP32 / STM32 specifics**: `references/migration-and-vendors.md`.

## Version notes worth knowing

- **3.7** is the current LTS (released July 2024, supported until mid-2029); LTS releases come every 2.5 to 3 years and 4.6 (planned April 2027) is scheduled as the next one, so check the release page before starting a long-lived product.
- **4.x** releases came roughly every four months up to 4.3; from 4.4 (April 2026) the cadence is six months. Several changes break 3.7-era code: board renames (`esp32_devkitc_wroom` → `esp32_devkitc` in 4.2, old alias removed in 4.5), `bus-speed` → `bitrate` for CAN (removed in 4.5), SPI `*_DT_SPEC_GET` delay argument optional in 4.3 and removed in 4.5 (CS timing moves to DT `spi-cs-*-delay-ns`), MCUboot default swap-using-offset (4.3, STM32 in 4.4), NVS/ZMS headers moved to `zephyr/kvss/` (4.4), `FIXED_PARTITION_*` → `PARTITION_*` and `zephyr,mapped-partition` (4.4), CMake `flash/debug` targets removed in favour of `west` (4.5), `CONFIG_GPIO` no longer default-on for most STM32 boards (4.5).
- Always read the migration guide for every release you cross: docs.zephyrproject.org → Releases → Migration guide.

## Reference files

| File | Read when |
|---|---|
| `references/build-west-devicetree-kconfig.md` | Workspace, manifest, SDK, board targets, app layout, sysbuild, overlays, out-of-tree boards/shields, PlatformIO status, DT macros and pinctrl, Kconfig, `__device_dts_ord` diagnosis |
| `references/kernel-and-concurrency.md` | Threads and priorities, workqueues, timers, semaphores/mutexes, message queues, FIFOs, events, zbus, ISRs, stack sizing, ticks and sleeping |
| `references/drivers-and-subsystems.md` | GPIO, UART APIs, I2C, SPI, ADC, PWM, CAN/CAN FD, Modbus RTU (worked example), sensor, counter, watchdog, flash map, NVS/ZMS/settings, LittleFS, networking, LTE modem, BLE, USB, PM, retention |
| `references/debug-logging-testing.md` | Logging modes and back-ends, shell, fatal errors and `addr2line`, coredump, `west debug`, thread analyzer, ROM/RAM reports, Twister, ztest, native_sim, QEMU, static analysis |
| `references/mcuboot-dfu-security.md` | MCUboot with sysbuild, signing, slots and swap modes, confirm/revert, MCUmgr/SMP DFU, TF-M and PSA crypto |
| `references/migration-and-vendors.md` | FreeRTOS/HAL → Zephyr concept map and pitfalls, nRF Connect SDK and Partition Manager, STM32 and ESP32 support level |

## Output expectations

- State the Zephyr version, board target and whether sysbuild is used.
- Give complete overlay + `prj.conf` + code together; a code snippet without its devicetree and Kconfig is half an answer.
- Mark anything version-dependent ("4.4+", "3.7 only") and anything not verified on hardware.
- When unsure whether a Kconfig symbol, binding or API exists in the user's version, say "check" and point to the docs page or to `west build -t menuconfig` / `dts/bindings/` rather than guessing.
- Do not flash, erase, recover or provision keys without the user's confirmation.

## Credits
Parts adapted from mohitmishra786/low-level-dev-skills (MIT License).
