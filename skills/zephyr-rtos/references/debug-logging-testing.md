# Logging, shell, debugging and testing

## Contents
1. Logging
2. Shell
3. Fatal errors and decoding
4. Coredump
5. Debugging with west (OpenOCD, J-Link, pyOCD)
6. Thread analyzer and memory reports
7. Testing — ztest, Twister, native_sim, QEMU, emulators
8. Static analysis
9. Worked example — hard fault in the field
10. Review checklist

---

## 1. Logging

```c
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(lockctrl, CONFIG_LOCKCTRL_LOG_LEVEL);   /* level from your own Kconfig, or LOG_LEVEL_INF */
LOG_INF("door %u open", door);
LOG_ERR("modbus init %d", err);
LOG_HEXDUMP_DBG(buf, len, "rx");
```

| Mode | Kconfig | Behaviour |
|---|---|---|
| Deferred (default) | `CONFIG_LOG_MODE_DEFERRED` | Messages copied to a buffer, processed by the log thread; cheap in ISRs; may drop when the buffer (`CONFIG_LOG_BUFFER_SIZE`) is full |
| Immediate | `CONFIG_LOG_MODE_IMMEDIATE` | Printed in the caller's context; ordered and lossless but slow and blocking — debugging only, never with time-critical ISRs |
| Minimal | `CONFIG_LOG_MODE_MINIMAL` | printk-like, smallest footprint |

- Back-ends: UART (`CONFIG_LOG_BACKEND_UART`), RTT (`CONFIG_USE_SEGGER_RTT` + `CONFIG_LOG_BACKEND_RTT`, needs J-Link or an RTT-capable probe), network, file system. Dictionary logging (`CONFIG_LOG_DICTIONARY_SUPPORT`) sends IDs instead of strings for tiny images; decode with `scripts/logging/dictionary/` tools.
- Levels per module (`CONFIG_<MODULE>_LOG_LEVEL_DBG`) and global `CONFIG_LOG_DEFAULT_LEVEL`. Runtime filtering with `CONFIG_LOG_RUNTIME_FILTERING` and the `log` shell command.
- Deferred mode loses the last messages before a crash: on panic the logger flushes in panic mode, but a hard reset (watchdog) does not. For crash forensics use coredump or a retained-RAM log.
- `%s` arguments located in RAM are copied into deferred messages by current logging (the old `log_strdup()` is gone), but every copied string eats log buffer; log short strings only.

## 2. Shell

`CONFIG_SHELL=y` on the console UART (`zephyr,shell-uart` chosen), or RTT/USB back-ends. Built-ins worth enabling in development: `kernel` (`kernel thread list`/`kernel thread stacks` on 4.x, `kernel threads`/`kernel stacks` on 3.7, `kernel reboot`), `device list`, `log`, `gpio`, `i2c scan`, `can`, `net`, `mcumgr`, `settings`. Own commands:

```c
static int cmd_open(const struct shell *sh, size_t argc, char **argv)
{ shell_print(sh, "opening %s", argv[1]); return post_lock_cmd(atoi(argv[1]), true); }
SHELL_STATIC_SUBCMD_SET_CREATE(sub_locker, SHELL_CMD_ARG(open, NULL, "open <door>", cmd_open, 2, 0), SHELL_SUBCMD_SET_END);
SHELL_CMD_REGISTER(locker, &sub_locker, "Locker commands", NULL);
```

Disable or lock the shell in production builds (it is a full control interface on the UART).

## 3. Fatal errors and decoding

Typical Cortex-M output:

```
E: ***** BUS FAULT *****
E:   Precise data bus error
E:   BFAR Address: 0x0
E: r0/a1:  0x00000000  r1/a2:  0x2000a1b4 ...
E: Faulting instruction address (r15/pc): 0x08004a6e
E: >>> ZEPHYR FATAL ERROR 25: Unknown error on CPU 0
E: Current thread: 0x20000e48 (rs485)
```

1. Decode with the SDK's GDB/addr2line from the exact ELF:
   `arm-zephyr-eabi-addr2line -e build/zephyr/zephyr.elf -f -i 0x08004a6e` (also the stacked `lr`).
   (`riscv64-zephyr-elf-` for ESP32-C3/C6, `xtensa-espressif_esp32s3_zephyr-elf-` for ESP32-S3; check the SDK's bin folder.)
2. Common reason codes: `K_ERR_STACK_CHK_FAIL` (stack overflow → raise stack size, see kernel reference), `K_ERR_KERNEL_OOPS` (`k_oops()`/assert), `K_ERR_KERNEL_PANIC`, CPU exception (bus/usage/mem fault).
3. `CONFIG_EXTRA_EXCEPTION_INFO=y` prints callee-saved registers too. `CONFIG_ASSERT=y` during development catches API misuse earlier (cost: size).
4. Products: `CONFIG_RESET_ON_FATAL_ERROR=y` (4.5+, default n; without it the system halts) and/or (any version) override `void k_sys_fatal_error_handler(unsigned int reason, const struct arch_esf *esf)` to store reason + PC in retained RAM, then `sys_reboot()`. Read the reset cause on boot with `hwinfo_get_reset_cause()`.
5. Keep every released `zephyr.elf` (and `build/zephyr/.config`) — without the exact ELF addresses are meaningless.

## 4. Coredump

`CONFIG_DEBUG_COREDUMP=y` with a back-end:
- `CONFIG_DEBUG_COREDUMP_BACKEND_LOGGING` → hex dump in the log; capture to a file and run `scripts/coredump/coredump_serial_log_parser.py log.txt dump.bin`, then `scripts/coredump/coredump_gdbserver.py build/zephyr/zephyr.elf dump.bin` and attach GDB (`target remote :1234`) to get `bt` and variables.
- `CONFIG_DEBUG_COREDUMP_BACKEND_FLASH_PARTITION` → stored in a `coredump_partition`; retrieve later via shell/MCUmgr. Best choice for field units.
- Memory captured: `CONFIG_DEBUG_COREDUMP_MEMORY_DUMP_MIN` (faulting thread stack) up to linker RAM (check option names in menuconfig for your version).

## 5. Debugging with west (OpenOCD, J-Link, pyOCD)

```bash
west debug                      # build + flash + GDB with the board's default runner
west debug -r jlink             # pick runner: openocd, jlink, pyocd, stm32cubeprogrammer (flash only)
west debugserver -r openocd     # server only; connect from VS Code (cortex-debug) or gdb
west attach                     # attach to running target without reflashing
west rtt                        # RTT terminal (recent versions; check `west rtt --help`)
```

- Build for debugging: `CONFIG_DEBUG_OPTIMIZATIONS=y` (-Og), `CONFIG_DEBUG_THREAD_INFO=y` so OpenOCD (`-rtos Zephyr`) and J-Link (Zephyr RTOS plug-in) list threads (`info threads`).
- Nordic: `nrfutil`/`nrfjprog` runners for flashing, J-Link for debugging (on-board on DKs). ESP32: `west flash` uses esptool; JTAG debug through the built-in USB-JTAG (S3/C3/C6) with Espressif OpenOCD.
- STM32: ST-Link with OpenOCD or `stm32cubeprogrammer` runner; J-Link after converting the ST-Link firmware is the smoothest for RTT.

## 6. Thread analyzer and memory reports

- Thread analyzer: see `kernel-and-concurrency.md` section 9; output lists stack usage and CPU % per thread.
- Size: `west build -t rom_report`, `west build -t ram_report` (tree per symbol/file), `west build -t ram_plot` / `rom_plot` (4.3+), `west build -t puncover` (web UI, `pip install puncover`), `west build -t footprint`. Linker summary at the end of each build shows FLASH/RAM % per region.
- Map file: `build/zephyr/zephyr.map`. Typical wins: lower `CONFIG_LOG_BUFFER_SIZE`, disable `CONFIG_ASSERT`, `CONFIG_SIZE_OPTIMIZATIONS=y` (default `-Os`), drop float printf, shrink net/BT buffer counts.
- Heap: `CONFIG_HEAP_MEM_POOL_SIZE` for `k_malloc`; prefer static allocation or `K_MEM_SLAB_DEFINE` in products. `CONFIG_SYS_HEAP_RUNTIME_STATS` to measure.

## 7. Testing — ztest, Twister, native_sim, QEMU, emulators

```c
#include <zephyr/ztest.h>
ZTEST_SUITE(crc_suite, NULL, NULL, NULL, NULL, NULL);    /* predicate, setup, before, after, teardown */
ZTEST(crc_suite, test_known_vector) { zassert_equal(app_crc16(v, 6), 0x4B37, "crc"); }
```

- Test project: `tests/crc/` with `CMakeLists.txt`, `prj.conf` (`CONFIG_ZTEST=y`), `testcase.yaml` (`tests: app.crc: platform_allow: native_sim ...`). The legacy ztest API (`ztest_test_suite`) is gone; 3.7 and 4.x use `ZTEST_SUITE`/`ZTEST`.
- Run: `west twister -T tests -p native_sim` (CI on Linux), `-p qemu_cortex_m3` for Cortex-M behaviour, `--integration` for the platforms listed in `integration_platforms`, `-C` for coverage.
- Hardware-in-the-loop: `west twister --generate-hardware-map map.yml`, edit, then `west twister --device-testing --hardware-map map.yml -T tests`.
- **native_sim** runs Zephyr as a Linux process (not on macOS hosts; use Docker or a Linux VM/CI). UART appears as a pty; `--stop_at=<seconds>` ends the run. Emulated peripherals: `zephyr,i2c-emul-controller`, `zephyr,spi-emul-controller`, CAN loopback (`zephyr,can-loopback`), flash simulator (`zephyr,sim-flash`), GPIO emulator — sensor drivers with emulators can be tested without hardware.
- QEMU targets (`qemu_cortex_m3`, `qemu_x86`, ...) come with the SDK; `west build -t run`.
- Host unit tests of pure logic: also fine with Unity + PlatformIO native or `find_package(Zephyr COMPONENTS unittest)` (available since 3.1; the old `ZephyrUnittest` package was removed in 4.5) — keep protocol parsers and state machines hardware-independent.

## 8. Static analysis

`west build -b <board> -- -DZEPHYR_SCA_VARIANT=<tool>` with `codechecker`, `sparse`, `gcc` (GCC `-fanalyzer`), `clang`, `eclair`, `coverity`, `cpptest`, `polyspace`, `iar_c_stat`, `iwyu` (availability depends on version; 3.7 has `codechecker`, `cpptest`, `gcc`, `sparse`). `dtdoctor` (4.3+) gives devicetree diagnostics. Add `-Werror` equivalents with `CONFIG_COMPILER_WARNINGS_AS_ERRORS=y` in CI.

## 9. Worked example — hard fault in the field

Unit reboots every few hours; field log shows only boot banners.

1. Add `CONFIG_DEBUG_COREDUMP=y` + `CONFIG_DEBUG_COREDUMP_BACKEND_FLASH_PARTITION=y` with a `coredump_partition` in the overlay, and a `k_sys_fatal_error_handler` that saves reason/PC/thread name to retained RAM before reboot; log `hwinfo_get_reset_cause()` at boot.
2. Next crash: reason `K_ERR_STACK_CHK_FAIL`, thread `modem`. Thread analyzer on a bench unit during reconnect storms shows 97 % use of a 2 KB stack (TLS handshake + logging with floats).
3. Fix: stack to 3 KB, move float formatting out of that thread, keep HW stack protection on. Add a CI test on native_sim that exercises the reconnect path and a release check that the analyzer reports < 75 %.
4. If instead reset cause is watchdog with no fatal error: a thread is blocked; use `kernel thread list` on a bench unit or a task watchdog channel per thread to find which one stopped checking in.

## 10. Review checklist

- [ ] Deferred logging in products; levels per module; no logging in hot ISRs; log buffer sized from measurement.
- [ ] Shell disabled or protected in production images.
- [ ] Fatal handler stores reason, PC and thread before reboot; reset cause logged at boot.
- [ ] Release ELF, `.config`, `zephyr.dts` and frozen manifest archived for every build.
- [ ] Coredump or retained-RAM crash record readable from deployed units.
- [ ] `rom_report`/`ram_report` reviewed; RAM/flash headroom documented.
- [ ] ztest suites run in CI on native_sim (Linux) and, for drivers, on hardware via Twister.
- [ ] SCA variant run in CI; warnings treated as errors.
