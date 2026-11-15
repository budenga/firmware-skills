---
name: embedded-linux
description: Embedded Linux on SoMs and SBCs (Toradex Verdin i.MX8M Mini/Plus with Torizon OS, Raspberry Pi 4/5 and CM4/CM5, Yocto, Buildroot, Debian) — boot chain, U-Boot and recovery, device tree and overlays (i.MX8M pinctrl, TorizonCore Builder, dtbo), probe and deferred-probe failures, kernel modules and drivers (I2C, SPI, platform, Kbuild, vermagic) and existing subsystems first, userspace hardware access (libgpiod, i2c-dev, spidev, RS-485, SocketCAN, watchdog), cross-compilation and Yocto SDKs, Docker device access, gdbserver, core dumps, strace, perf, ftrace, boot time, read-only rootfs and OTA, flash wear and power-fail safety. Use it whenever the target runs Linux rather than bare metal, even if the user only says "a placa não arranca", "o driver não faz probe", "overlay", "o contentor não vê o /dev", "compilar para a placa", "Linux embebido", "Torizon", "Yocto", "i.MX8" or pastes dmesg or a U-Boot log.
---

# Embedded Linux (SoMs, Torizon, Yocto, Buildroot)

Embedded Linux fails differently from an MCU. The hardware description lives in the device tree, not in code; a driver that silently never probes looks exactly like broken hardware; the filesystem can be read-only or rolled back by OTA; and the app often runs in a container that cannot see the device it needs. Find the layer that is failing before changing anything.

Also load `embedded-c-cpp` for general C/C++ quality, and a bus/interface hardware skill (such as `comms-interfaces`) if one is installed, for the electrical side. If a Torizon/embedded-Linux agent is available, delegate board-specific scripting, container and deployment work to it; use this skill for the rules and the diagnosis.

## First, identify the platform

| Find | How | Why it matters |
|---|---|---|
| SoM / SoC and carrier | `cat /proc/device-tree/model`, `cat /proc/device-tree/compatible` | Pin muxing, buses and peripherals differ (i.MX8M Mini has no CAN controller — CAN only via an on-module MCP2518FD on some variants; Plus has two FlexCAN) |
| Distro and build system | `cat /etc/os-release`; `ostree admin status` (Torizon); `/etc/buildroot_version` | Changes how you add packages, kernel modules and config |
| Kernel version and config | `uname -a`; `zcat /proc/config.gz \| grep CONFIG_X` | Out-of-tree modules must match exactly; a missing `CONFIG_` explains a missing device |
| Loaded device tree and overlays | `fw_printenv fdtfile`; Torizon `/sysroot/boot/ostree/torizon-*/dtb/overlays.txt` | Most "hardware does not work" issues are DT issues |
| Where the app runs | `docker ps`, `systemctl status`, compose file | Containers need explicit device, group and namespace access |
| Rootfs writable or not | `mount \| grep ' / '`, `findmnt /usr` | Edits under `/usr` vanish or fail on OSTree and read-only images |
| Console access | debug UART (Verdin UART_3 on the dev board, 115200 8N1), SSH | Without a serial console you cannot see U-Boot or an early kernel panic |

## Rules that prevent the classic failures

1. **Use an existing kernel driver and subsystem before writing one.** A DT node with the right `compatible` gives a tested hwmon, IIO, LED, input, RTC or PWM device with sysfs and udev for free. A custom char driver is weeks of work and maintenance on every kernel update.
2. **GPIO through the character device (libgpiod), never `/sys/class/gpio`.** Sysfs GPIO is deprecated, its numbering is not stable across kernels, and it leaves pins configured when the app crashes.
3. **A pin belongs to one pinctrl group.** On i.MX8M, adding a pad to a new group while the original node still claims it gives "pin already requested" and the second device fails to probe. Disable or re-mux the original node.
4. **Never edit the board `.dts` on the target to test.** Write an overlay (or a TorizonCore Builder customization) so the change is versioned, reproducible and survives updates.
5. **Out-of-tree modules are built against the exact running kernel.** Same version, same config, same `Module.symvers`. "Invalid module format" or "disagrees about version of symbol" is a build mismatch, not a code bug.
6. **Cross-compile with the SDK's environment, not with the host's gcc.** The sysroot carries the target's glibc and headers; linking against host libraries gives binaries that fail with "GLIBC_2.xx not found" or illegal instructions.
7. **Containers get the minimum access they need.** `--device`, `device_cgroup_rules`, `group_add`, `network_mode: host` for CAN. `--privileged` hides the real requirement, breaks the security model and still misses namespaces such as CAN interfaces.
8. **Assume power will be cut at the worst moment.** Write new file, `fsync`, `rename`, `fsync` the directory. Keep logs in RAM or rate-limited. eMMC wears out, and a half-written config bricks a device in the field.
9. **Watchdog has one owner.** Either systemd (`RuntimeWatchdogSec` + per-service `WatchdogSec`) or the app opens `/dev/watchdog`, not both.
10. **Confirm before flashing, writing the eMMC boot partitions, changing U-Boot env or fuses.** i.MX eFuses (boot config, HAB secure boot) are permanent; a bad boot container needs serial-download recovery.

## Workflow by task

**Hardware does not appear**: `dmesg` for the driver name and probe errors → is the node in the live DT (`/proc/device-tree`) with `status = "okay"`? → is the driver built (`/proc/config.gz`, `modinfo`)? → deferred probe (`/sys/kernel/debug/devices_deferred`) → pinmux (`/sys/kernel/debug/pinctrl/*/pinmux-pins`) → only then scope the bus. See `references/boot-and-device-tree.md`.

**Add a peripheral**: check for an upstream binding and driver → write an overlay → test from userspace tools (`i2cdetect`, `gpioinfo`, `candump`) → then application code. Write a driver only if nothing fits (`references/kernel-drivers.md`).

**Write an app that touches hardware**: pick the userspace interface (`references/userspace-hw-access.md`) → cross-build with the SDK or in a container (`references/cross-build-containers.md`) → deploy and debug remotely with gdbserver.

**Crash, hang or slowness**: core dump and `coredumpctl`, `strace`, `perf`, ftrace for kernel-side latency (`references/debugging-profiling.md`).

**Product hardening**: boot time, read-only rootfs and OTA, watchdog, logging, power-fail safety (`references/reliability.md`).

## Reference files

| File | Read when |
|---|---|
| `references/boot-and-device-tree.md` | Boot chain, U-Boot console and env, recovery (uuu, Toradex Easy Installer), DT syntax, i.MX8M pinctrl, overlays (Torizon and generic), probe and deferred-probe debugging |
| `references/kernel-drivers.md` | Choosing a subsystem, out-of-tree modules and Kbuild, vermagic, platform/I2C/SPI drivers with regmap, misc/char devices, TorizonCore Builder module builds |
| `references/userspace-hw-access.md` | libgpiod, i2c-dev, spidev, serial and RS-485, SocketCAN, watchdog, sysfs devices, driver vs userspace decision |
| `references/cross-build-containers.md` | Toolchains, Yocto SDK and sysroots, CMake toolchain, Buildroot, Docker/Compose on Torizon, device access, deploying |
| `references/debugging-profiling.md` | dmesg/journalctl, kernel config, dynamic debug, ftrace, kgdb/JTAG, gdbserver, core dumps, strace/ltrace, perf, valgrind and ASan |
| `references/reliability.md` | Boot time, OSTree/Torizon updates and rollback, read-only rootfs, systemd watchdog, flash wear, power-fail-safe writes, review checklist |
| `references/raspberry-pi.md` | Raspberry Pi 3/4/5, Zero 2 W, CM4/CM5 — Pi 5 RP1 differences, EEPROM boot order, `config.txt`/`cmdline.txt`, `dtoverlay`/`dtparam` and custom overlays, UART/I2C/SPI/CAN/RS-485/1-Wire/PWM/watchdog setup, libcamera and displays, Bookworm (venv, gpiozero/lgpio), SD wear, power/throttling, CM provisioning |

## Output expectations

- State the assumed SoM, kernel version and distro when the user did not give them.
- Give commands that run on the target and say which ones need root or debugfs (`mount -t debugfs none /sys/kernel/debug`).
- Say what was not tested on hardware. Do not flash, write boot partitions, `fw_setenv`, burn fuses or push OTA updates without the user's confirmation.

Credits: Parts adapted from mohitmishra786/low-level-dev-skills (MIT License).
