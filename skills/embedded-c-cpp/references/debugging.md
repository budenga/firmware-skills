# Debugging firmware systematically

## Contents
1. Method
2. Classify the failure
3. Evidence to collect
4. Tools
5. Symptom → likely causes
6. Instrumentation patterns

---

## 1. Method

1. **Reproduce** reliably, or find the conditions that raise the failure rate (temperature, supply voltage, traffic, time running).
2. **Classify** where it fails (table below) — that alone halves the search space.
3. **Collect evidence before changing code**: reset cause, fault registers, last log lines, scope/logic-analyser capture.
4. **One hypothesis at a time**, with a prediction ("if it is stack overflow, the high-water mark of task X is ~0"). Test the prediction.
5. **Instrument, don't rewrite.** Rewriting a subsystem "to be safe" hides the bug.
6. **Fix the cause**, then add a regression test (host test if logic, target test if hardware).
7. Write down cause + fix in the commit message.

## 2. Classify the failure

| Stage | Signs | First suspects |
|---|---|---|
| Build | compiler/linker errors | include paths, wrong part selected, missing generated files, out of flash/RAM (`region overflowed`) |
| Flash | programmer cannot connect/verify | wiring, power, wrong interface (SWD/ICSP/UART boot), boot pins, read-out protection / code protection, debug pins reused as GPIO |
| Boot | nothing runs, boot loop | clock config (crystal missing), brown-out, watchdog too short, vector table / linker script, stack pointer, strapping pins |
| Runtime, deterministic | same failure every time | logic bug, wrong register config, null pointer, buffer overrun |
| Runtime, intermittent | random crashes, rare lockups | race with ISR, stack overflow, uninitialised variable, heap fragmentation, power/EMI, timing margin |

## 3. Evidence to collect

- **Reset cause** register (every MCU has one) — tells watchdog vs brown-out vs software vs power-on.
- **Fault state**: Cortex-M fault registers + stacked PC/LR (see `stm32-firmware`), ESP32 Guru Meditation dump + backtrace (see `esp32-firmware`), PIC stack overflow/underflow flags (STKOVF/STKUNF) and trap vectors on PIC24.
- **Map file**: which function contains the faulting address (`addr2line -e firmware.elf 0x0800_1234`).
- **Stack high-water marks** per task / painted stack.
- **Supply voltage under load** on a scope (Wi-Fi TX bursts and motors cause dips).
- **Bus traces** with a logic analyser for I2C/SPI/UART problems: you see ACK/NACK, clock stretching, framing.
- **Exact versions**: toolchain, framework/HAL, board revision.

## 4. Tools

| Tool | Use |
|---|---|
| Debugger (ST-Link + GDB/CubeIDE, PICkit/ICD + MPLAB X, ESP32 JTAG/USB-JTAG + OpenOCD) | breakpoints, step, inspect memory, **watchpoints** (break when a variable is written — finds memory corruption fast) |
| Serial log | always available; keep it non-blocking (DMA or ring buffer) so logging doesn't change timing |
| GPIO toggle + scope/logic analyser | measure ISR latency, task timing, see if code reaches a point, with near-zero overhead |
| SWO/ITM, SEGGER RTT | fast logging on Cortex-M with the debugger attached |
| Logic analyser (Saleae, cheap FX2 clones + PulseView/sigrok) | decode I2C/SPI/UART/CAN |
| Core dumps (ESP-IDF) | post-mortem analysis of a crash |

## 5. Symptom → likely causes

| Symptom | Check |
|---|---|
| Works in debug build, fails in release | missing `volatile`, uninitialised variables, timing loops, undefined behaviour exposed by optimisation |
| Works with debugger attached, fails without | debug-mode clock/low-power differences, `printf` via semihosting blocking without debugger, power from the probe |
| Crashes after hours/days | heap fragmentation, counter overflow (32-bit ms tick wraps at ~49.7 days; 16-bit much sooner), slow memory leak, flash wear |
| Crashes when adding a `printf` / a local array | stack overflow |
| Value randomly changes | buffer overrun from a neighbour, ISR race, DMA writing to it, stack overflow into data |
| I2C stuck busy | slave holding SDA after a reset mid-transfer → clock out 9 pulses on SCL to free it; missing pull-ups; wrong 7/8-bit address |
| UART garbage | baud mismatch from wrong clock assumption, wrong voltage level, missing common ground |
| Peripheral does nothing | clock to the peripheral not enabled, pin not set to the alternate function, wrong pin mux/PPS, peripheral in reset |
| Interrupt fires once only | flag not cleared, or cleared too late (re-entry), or interrupt re-enable missing |
| Interrupt never fires | global/peripheral enable missing, wrong vector name (typo creates a new function, default handler runs) |
| Random resets under load | brown-out (scope the supply), watchdog starved by a long blocking call |

## 6. Instrumentation patterns

- **Stack painting**: fill the stack with `0xA5` at startup, later scan for the first untouched word to get the real maximum use.
- **Breadcrumbs**: a small array in no-init RAM where code writes a "last reached" ID; survives a watchdog reset, read it at next boot.
- **Fault capture**: in the fault handler, store PC/LR/fault registers to no-init RAM, then reset; report on next boot.
- **Counters**: count overflows, retries, CRC errors, queue-full events; expose them through a debug command. Silent failures become visible.
- **Assert with location**: `#define ASSERT(x) do { if (!(x)) assert_failed(__FILE__, __LINE__); } while (0)`.
