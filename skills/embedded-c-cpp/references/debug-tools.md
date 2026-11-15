# Debug and analysis tools

The tools themselves: GDB, OpenOCD, ELF/map inspection, static analysis, sanitizers, and where each one runs in CI. The debugging *method* and symptom tables are in `debugging.md`; Unity, test layout and the basic `pio check`/cppcheck commands are in `testing.md`; Cortex-M fault decoding is in `stm32-firmware` → `references/faults-and-recovery.md`. ESP32 panics and `idf.py monitor` → `esp32-firmware`.

## Contents
1. GDB on a microcontroller
2. Breakpoints and watchpoints
3. Inspecting state
4. OpenOCD
5. ELF and map analysis
6. Static analysis
7. Sanitizers on host builds
8. Where each tool fits in CI

---

## 1. GDB on a microcontroller

Use the GDB that matches the target: `arm-none-eabi-gdb` (Arm GNU Toolchain) or `gdb-multiarch`; `xtensa-esp32-elf-gdb` / `riscv32-esp-elf-gdb` for ESP32; MPLAB X has its own debugger for PIC. Always load the **ELF** (symbols), never the `.bin`/`.hex`.

GDB talks to a **GDB server** that drives the probe:

| Server | Typical start | Default port |
|---|---|---|
| OpenOCD | `openocd -f interface/stlink.cfg -f target/stm32g4x.cfg` | 3333 (per core) |
| ST-LINK_gdbserver (CubeIDE / CubeCLT) | started by CubeIDE, or `ST-LINK_gdbserver -cp <CubeProgrammer bin dir> ...` | 61234 |
| SEGGER J-Link | `JLinkGDBServer -if SWD -device STM32G431KB -speed 4000` | 2331 |
| pyOCD | `pyocd gdbserver -t stm32g431kbux` | 3333 |
| probe-rs | `probe-rs gdb --chip STM32G431KBUx` | 1337 |

Typical session:
```
arm-none-eabi-gdb build/app.elf
(gdb) target extended-remote :3333     # extended-remote: the session survives reset/re-run
(gdb) monitor reset halt               # 'monitor' passes the rest to the server (OpenOCD syntax here)
(gdb) load                             # program every loadable section at its LMA, using the server's flash driver
(gdb) break main
(gdb) continue
```
`load` honours the ELF's load addresses — if the linker script is wrong, `load` writes to the wrong place too. After `load`, reset before running so the core starts from the real vector table, not from wherever GDB set the PC.

**Project GDB script** (commit it, call it explicitly):
```
# tools/debug.gdb — run with: arm-none-eabi-gdb -x tools/debug.gdb build/app.elf
set confirm off
set pagination off
set print pretty on
target extended-remote :3333
monitor reset halt
define reflash
  monitor reset halt
  load
  monitor reset halt
end
define fault
  p/x *(uint32_t*)0xE000ED28
  p/x *(uint32_t*)0xE000ED2C
  info symbol ((uint32_t*)(($lr & 4) ? $psp : $msp))[6]
end
```
A `.gdbinit` in the project directory is **not** executed by default (GDB's auto-load safe-path); prefer `-x file`, or add `add-auto-load-safe-path /path/to/project` to your personal `~/.gdbinit`. Batch use (CI, scripts): `arm-none-eabi-gdb -batch -x tools/flash.gdb build/app.elf`.

IDE front-ends wrap the same thing: CubeIDE (ST-LINK_gdbserver or OpenOCD), VS Code **Cortex-Debug** (`servertype`: `openocd`, `stlink`, `jlink`, `pyocd`; `svdFile` for registers; `rttConfig` for RTT), PlatformIO `pio debug` (uses `debug_tool` / `debug_init_break` from `platformio.ini`).

## 2. Breakpoints and watchpoints

| Command | What it does | Notes on a MCU |
|---|---|---|
| `break file.c:120` / `b func` | Breakpoint | In flash it must be a **hardware** comparator (FPB); GDB switches to one automatically when the server reports a memory map marking the region as flash (OpenOCD, J-Link, pyOCD do; `set breakpoint auto-hw on` is the default). In RAM a software `BKPT` is used |
| `hbreak func` | Force a hardware breakpoint | Use when the server sends no memory map. Not for code in SRAM on M0/M0+/M3/M4: their FPB/BPU only matches the Code region `0x0000_0000–0x1FFF_FFFF` (M7 and Armv8-M FPB cover any address). For RAM functions, set the software breakpoint **after** startup has copied the code, or the copy overwrites it |
| `tbreak func` | One-shot breakpoint | Good for "stop at `main` once" |
| `break uart.c:88 if len > 64` | Conditional | The core still halts every time and GDB evaluates the condition: it is slow and changes timing. For hot code, put the condition in code with `__BKPT(0)` (debug builds only — without a debugger attached `BKPT` escalates to HardFault) |
| `ignore 2 99` | Skip breakpoint 2 the next 99 hits | Same halting cost |
| `commands 2 … end` | Run GDB commands at a hit (`silent`, `p x`, `continue`) | Poor-man's tracepoint; still halts |
| `watch g_state` | Stop when written | Hardware (DWT) — runs at full speed. The core stops **after** the access, so the PC is just past the guilty instruction — look one or two instructions back |
| `rwatch rx_buf[3]` | Stop when read | |
| `awatch g_mode` | Stop on read or write | |
| `watch -l ctx->len` | Watch the **address**, not the expression | Survives leaving the scope where `ctx` was valid |
| `info breakpoints`, `delete`, `disable 3` | Manage | |

Hardware limits (Cortex-M, implementation options vary): breakpoints — M0/M0+: up to 4; M3/M4: 6 instruction comparators; M7: 8 (4 in the reduced debug configuration); watchpoints — M0/M0+: up to 2; M3/M4: 4; M7: 4 (2 reduced). Most STM32 have the full set. A watchpoint covers 1, 2 or 4 aligned bytes, or a larger power-of-two-sized, aligned range using the DWT address mask. When GDB says it cannot insert a breakpoint, you ran out — delete some.

The best memory-corruption tool on a MCU: find the variable that gets clobbered, `watch -l` it, run; the core stops on the guilty store, then `bt`.

## 3. Inspecting state

```
(gdb) info registers                  # r0–r12, sp, lr, pc, xpsr (+ msp, psp, primask, basepri, control with OpenOCD)
(gdb) info all-registers              # includes FPU s0–s31, fpscr
(gdb) p/x $xpsr & 0x1FF               # active exception number: 0 = thread mode, 3 = HardFault, 16+n = IRQ n
(gdb) x/16xw 0x20000000               # 16 words in hex
(gdb) x/8i $pc                        # disassemble around the PC
(gdb) x/s log_buf                     # string
(gdb) p *uart_ctx                     # struct (set print pretty on)
(gdb) p/x arr[0]@32                   # 32 elements from arr[0]
(gdb) p sizeof(big_struct)
(gdb) info symbol 0x08004a1c          # which function contains this address
(gdb) info line *0x08004a1c           # which source line
(gdb) bt                              # backtrace
(gdb) frame 2 / up / down / info locals
```
**Backtrace on Cortex-M**: GDB recognises the `EXC_RETURN` value (`0xFFFFFFxx`) in LR and unwinds through the hardware-stacked frame, printing `<signal handler called>` between the handler and the interrupted code. The backtrace stops or turns to garbage when: the stack is corrupted (often the actual bug), you are in a naked/assembly handler without unwind info, or you stopped in the middle of a function prologue. Then fall back to reading the stacked frame by hand (`faults-and-recovery.md` §1) and `x/32xw $sp` + `info symbol` on values that look like flash addresses.

**RTOS threads**: OpenOCD with `-rtos auto` (or `$_TARGETNAME configure -rtos FreeRTOS` in the config) makes `info threads` / `thread N` / `bt` show every FreeRTOS task. Needs the FreeRTOS symbols present (and for recent FreeRTOS, `uxTopUsedPriority` kept by the linker — see the OpenOCD docs).

**Peripheral registers by name** (from the vendor **SVD** file):
- VS Code Cortex-Debug: `"svdFile": "STM32G431.svd"` → peripheral view with decoded bit fields.
- CubeIDE: SFRs view (SVD included).
- Plain GDB: PyCortexMDebug — `source gdb.py` (from that project), `svd_load STM32G431.svd`, `svd USART2`, `svd USART2 CR1`. Needs a GDB built with Python support.
- Without SVD: `x/wx` the address from the reference manual and decode by hand.

Reading some registers has side effects (reading a UART data register clears RXNE; reading some status registers clears flags). Do not let a debugger view auto-refresh those while you are debugging that peripheral.

## 4. OpenOCD

```
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg                    # server: gdb :3333, telnet :4444, tcl :6666
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c "program build/app.elf verify reset exit"
openocd ... -c "program build/app.bin 0x08000000 verify reset exit"      # raw .bin needs an address; .elf/.hex carry it
```
Configs live in OpenOCD's `scripts/` dir: `interface/` (probe), `target/` (chip), `board/` (both, for dev boards). Write a project `openocd.cfg` instead of long command lines:
```tcl
source [find interface/stlink.cfg]   ;# selects the SWD transport itself; stlink-dap.cfg = newer DAP-direct driver
adapter speed 4000                    ;# lower (e.g. 1000) for long wires or flaky connections
adapter serial <probe-serial>         ;# pick one probe when several are plugged in (OpenOCD >= 0.12)
reset_config srst_only srst_nogate connect_assert_srst   ;# "connect under reset" (needs NRST wired)
source [find target/stm32f4x.cfg]
$_TARGETNAME configure -rtos auto
```

| Command (telnet or `monitor ...`) | Effect |
|---|---|
| `reset run` / `reset halt` / `reset init` | Reset and run / stop at the reset vector / stop and run the target's init scripts (clocks for fast flashing) |
| `halt`, `resume`, `step` | Run control |
| `mdw 0x20000000 16`, `mww 0x40021000 1` | Memory display/write word (also `mdb`, `mdh`) |
| `flash info 0`, `flash erase_sector 0 2 3`, `flash write_image erase app.bin 0x08008000` | Flash operations — erasing is destructive, confirm first |
| `stm32f4x unlock 0` / `stm32l4x option_read` ... | Family-specific option-byte commands; check OpenOCD docs and `stm32-firmware` before touching protection |

**Multiple cores / targets**: OpenOCD opens one GDB port per core (3333, 3334, ...) — dual-core STM32H7/H745, ESP32 (or an SMP view of both cores). For two boards at once, run two OpenOCD instances with different `adapter serial`, `gdb_port`, `telnet_port` and `tcl_port`.

**RTT** (fast logging through the debug probe, no UART): with the SEGGER RTT sources in the firmware,
```
rtt setup 0x20000000 0x8000 "SEGGER RTT"   ;# RAM range to search for the control block
rtt start
rtt server start 9090 0                     ;# then: nc localhost 9090
```
ESP32 uses Espressif's OpenOCD fork (`openocd-esp32`, `board/esp32s3-builtin.cfg` for the built-in USB-JTAG); ESP-IDF's `idf.py openocd` / `idf.py gdb` wrap it.

**Semihosting** (target `printf` and file I/O executed on the host via the debugger):
- Enable: link with `--specs=rdimon.specs -lrdimon`, call `initialise_monitor_handles()` early, `monitor arm semihosting enable` in GDB.
- Every call halts the core while the host services it — milliseconds per call; it destroys real-time behaviour.
- **Without a debugger attached the semihosting `BKPT` instruction faults** (HardFault on Cortex-M). A build with semihosting left in "works with the debugger, hangs or resets without it" (`debugging.md` §5). Use RTT, SWO or UART for logs; keep semihosting for host-side test runners on QEMU or a bench harness, never in release builds.

## 5. ELF and map analysis

| Question | Command |
|---|---|
| How much flash/RAM? | `arm-none-eabi-size build/app.elf` (text+data = flash, data+bss = RAM); `-A` for per-section |
| Per memory region, every build | link with `-Wl,--print-memory-usage` |
| Biggest symbols | `arm-none-eabi-nm --size-sort -S -C -r build/app.elf \| head -30` (`t/T` = code, `d/D` = initialised data, `b/B` = bss, `r/R` = rodata) |
| Where did each section land (VMA/LMA)? | `arm-none-eabi-objdump -h build/app.elf` |
| What will the programmer write? | `arm-none-eabi-readelf -l build/app.elf` (program headers: PhysAddr = load address) |
| Is the vector table first and correct? | `arm-none-eabi-objdump -s -j .isr_vector build/app.elf \| head` (word 0 = `_estack`, word 1 = odd address of `Reset_Handler`) |
| Where is symbol X? | `arm-none-eabi-nm -n build/app.elf \| grep X`, or `readelf -s` |
| Address → function and line | `arm-none-eabi-addr2line -e build/app.elf -f -C -i 0x08004a1c` (`-i` shows inlined frames) |
| Disassembly with source | `arm-none-eabi-objdump -d -S -C build/app.elf > app.lst` (build with `-g`) |
| Who pulled this function in? | map file with `-Wl,-Map=app.map,--cref`: the cross-reference table lists every referrer; "Archive member included to satisfy reference by file (symbol)" at the top explains library pulls (e.g. who dragged in `printf` float support) |
| Size per source file / growth over time | `bloaty build/app.elf -d compileunits`, `puncover` (HTML view with stack usage), or diff `size` output between commits |
| Worst-case stack per function | `-fstack-usage` writes `.su` files; `-Wstack-usage=256` warns; puncover or `avstack.pl` combines them with the call graph (recursion and function pointers are not resolved) |

The ELF used for `addr2line`/GDB must be **exactly** the one that was flashed; keep release ELFs (with `-g`, which costs no flash) archived with the version number. A crash address from the field is useless without them.

Same tools for other toolchains: `xtensa-esp32-elf-*` / `riscv32-esp-elf-*` (plus `idf.py size`, `size-components`, `size-files`), `xc32-size`/`xc32-nm`/`xc32-objdump` for PIC32. XC8/XC16 produce their own map and memory summary (`--summary`, MPLAB X dashboard).

## 6. Static analysis

Order of value for effort:

1. **The compiler** — full warning set from `coding-standards.md` §3 with `-Werror` in CI. Free, zero false-positive triage overhead, catches most conversion/format/shadowing bugs.
2. **GCC `-fanalyzer`** (GCC 10+, best on C, improving each release): path-sensitive checks for NULL dereference, use-after-free, double free, leaks, some buffer overruns. Slower and noisier — run it in a separate CI job on your own sources, not in every developer build.
3. **cppcheck** — basic commands in `testing.md` §7. Make it see what the compiler sees: `cppcheck --project=build/compile_commands.json --enable=warning,style,performance,portability --inline-suppr --error-exitcode=1 -i Drivers -i Middlewares`. `-i` skips those translation units, but vendor headers your own files include are still analysed — silence those with `--suppress='*:*/Drivers/*'` rather than expecting `-i` to do it. Add `--platform=arm32-wchar_t4` (shipped in cppcheck's `platforms/`; `pic8`, `pic8-enhanced`, `pic16` and `avr8` are also shipped, or write a custom platform XML) so type-size checks are right.
   - **MISRA addon**: `cppcheck --addon=misra.json ...` with `misra.json` = `{"script": "misra.py", "args": ["--rule-texts=misra_c_2012_rules.txt"]}`. The rule-text file is your own copy of the headlines (MISRA texts are licensed, not shipped). Open-source cppcheck checks a subset of MISRA C:2012; Cppcheck Premium, PC-lint Plus, Polyspace, Helix QAC, Parasoft or LDRA are what a certification audit expects. Record deviations (rule, location, reason) rather than mass-suppressing.
4. **clang-tidy** — needs `compile_commands.json` (`pio run -t compiledb`; CMake `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`; `bear -- make` for Makefiles). Cross-compile pitfall: clang does not know every `arm-none-eabi-gcc` flag or where newlib headers live. Either:
   - run clang-tidy on the **host test build** database (simplest, covers portable logic), or
   - add `--extra-arg=--target=arm-none-eabi --extra-arg=-Wno-unknown-warning-option` and the GCC sysroot include paths (`arm-none-eabi-gcc -print-sysroot`, `-xc -E -v /dev/null` lists them).
   Useful check groups for firmware: `bugprone-*`, `clang-analyzer-*`, `cert-*`, `misc-*`, `readability-*` selectively; `cppcoreguidelines-*` and `modernize-*` for C++ code. Keep the configuration in a committed `.clang-tidy` with `HeaderFilterRegex` limited to your directories; suppress single findings with `// NOLINT(check-name)` plus a reason.
5. **Clang static analyzer** (`scan-build`) — useful on host builds; overlaps with `clang-analyzer-*` in clang-tidy.

Always exclude vendor/generated code (HAL, CMSIS, ESP-IDF components, `mcc_generated_files/`, CubeMX `Drivers/`) from the reported results — but keep their headers in the include paths, or the analyser guesses types wrong.

## 7. Sanitizers on host builds

Logic that compiles for the PC (see `testing.md` §2–§4) can run under sanitizers that do not exist on a MCU:

| Sanitizer | Flag (GCC/Clang) | Finds |
|---|---|---|
| AddressSanitizer | `-fsanitize=address -fno-omit-frame-pointer` | Buffer overruns (stack, global, heap), use-after-free, use-after-return (`ASAN_OPTIONS=detect_stack_use_after_return=1`) |
| UndefinedBehaviorSanitizer | `-fsanitize=undefined` | Signed overflow, bad shifts, misaligned access, NULL dereference, out-of-range enum/bool, division by zero |
| Clang integer checks | `-fsanitize=integer`, `-fsanitize=implicit-conversion` (Clang only) | Unsigned wrap and **silent truncation** (`uint8_t x = 300`) — exactly the bugs `-Wconversion` warns about, but at run time with real data |
| ThreadSanitizer | `-fsanitize=thread` (not with ASan) | Data races in host simulations that use threads to model ISRs/tasks |
| MemorySanitizer | `-fsanitize=memory` (Clang, Linux only) | Reads of uninitialised memory |

Practical settings:
- Make failures fail the test: `-fno-sanitize-recover=all`, `UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1`.
- Build sanitizer runs with `-O1 -g`: fast enough, readable stack traces.
- LeakSanitizer is not available on every platform (notably some macOS toolchains); run leak checks in the Linux CI job.
- PlatformIO native + ASan quirk on macOS: see `testing.md` §3.
- Remember the host is 64-bit with 32-bit `int`: a sanitizer-clean host run does not prove the 16-bit-`int` PIC build is clean. Use fixed-width types in tested code.

**UBSan on the target** is possible without any runtime: `-fsanitize=undefined -fsanitize-undefined-trap-on-error` (GCC; Clang and recent GCC also accept `-fsanitize-trap=undefined`) turns each check into a trap instruction → UsageFault/HardFault at the offending PC, which `addr2line` resolves. It costs flash and speed; use it in a debug build on bigger parts (Cortex-M4/M7, ESP32), not in release.

## 8. Where each tool fits in CI

| Stage | Runs | Blocks merge? |
|---|---|---|
| Every push | Target build with full warnings and `-Werror`; `--print-memory-usage` / `size` printed and compared with a budget (fail if flash/RAM grows past a threshold) | Yes |
| Every push | Host unit tests under ASan + UBSan (`testing.md` §4, §8) | Yes |
| Every push | cppcheck (and clang-tidy on the host database) on your code only, `--error-exitcode=1` for warning-level findings | Yes, for new findings |
| Nightly / weekly | `-fanalyzer`, `scan-build`, MISRA addon full report, `-fstack-usage` worst-case report | Report, triage; block only on agreed rules |
| Release | Archive ELF + map + version + toolchain version; size diff vs previous release | Yes (artefacts must exist) |
| Hardware-in-the-loop (if a runner has a board) | Flash with OpenOCD/`STM32_Programmer_CLI`/`esptool` from a script, run on-target Unity tests or a scripted smoke test, collect RTT/UART log | Usually yes for release branches |

GDB itself has no CI role except scripted flashing/batch checks; interactive debugging belongs on the bench.
