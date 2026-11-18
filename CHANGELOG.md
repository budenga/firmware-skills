# Changelog

All notable changes to this project are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/): new skills, agents or references bump the minor version; corrections bump the patch version.

## [1.1.0] - 2026-11-15

### Added
- Skill `embedded-linux`: Toradex Verdin i.MX8M / Torizon OS, Raspberry Pi 3/4/5 and CM4/CM5, Yocto and Buildroot — boot chain and recovery, device tree and overlays, probe failures, kernel drivers, userspace hardware access (libgpiod, i2c-dev, spidev, RS-485, SocketCAN), cross-builds and containers, debugging and profiling, OTA and read-only rootfs.
- Skill `zephyr-rtos`: Zephyr 3.7 LTS and 4.x on STM32, ESP32 and nRF — west, sysbuild, devicetree, Kconfig, kernel, drivers and subsystems, MCUboot and DFU, Twister/ztest, migration from FreeRTOS.
- `embedded-c-cpp` references: `startup-linker-bootloaders`, `debug-tools`, `rtos-design`.
- Agent `linux-bringup-debugger`: evidence-based diagnosis of embedded-Linux bring-up and runtime problems.
- `THIRD_PARTY_NOTICES.md` with the MIT notice for parts adapted from [mohitmishra786/low-level-dev-skills](https://github.com/mohitmishra786/low-level-dev-skills).

### Changed
- `firmware-reviewer`, `firmware-crash-analyst` and `architecture-reviewer` now load the RTOS, bootloader, Zephyr and Linux material when the project uses it.
- All new content was fact-checked against official sources and tested on planted-bug cases before release.

## [1.0.0] - 2026-11-15

### Added
- Skills `embedded-c-cpp`, `stm32-firmware`, `esp32-firmware` (PlatformIO Arduino first, ESP-IDF also) and `pic-firmware` (PIC16/18/24/32 and dsPIC33).
- Agents `firmware-reviewer`, `firmware-crash-analyst`, `firmware-test-writer`, `datasheet-researcher` and `architecture-reviewer`.
- Claude Code plugin and marketplace manifests; MIT license.

[1.1.0]: https://github.com/budenga/firmware-skills/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/budenga/firmware-skills/releases/tag/v1.0.0
