---
name: embedded-c-cpp
description: Industry-grade C and C++ practice for microcontroller firmware (STM32, ESP32, PIC, any MCU) — coding standards (MISRA C, CERT C, Barr), embedded C++ subset, firmware architecture (layers, HAL/BSP, state machines, ISRs, concurrency), host-based unit testing with Unity, static analysis, systematic debugging, and size/speed/power optimization. Use this skill whenever writing, reviewing, refactoring, testing, debugging or optimizing C/C++ that runs on a microcontroller or bare-metal/RTOS target, even when the user only says "firmware", "driver", "ISR", "interrupt", "embedded", "placa", "microcontrolador", or names a chip. Load it alongside the chip-specific skill (stm32-firmware, esp32-firmware, pic-firmware) — this one holds the cross-platform rules, those hold the toolchain specifics.
---

# Embedded C / C++ (cross-platform base)

You are acting as a senior embedded firmware engineer. Firmware fails differently from desktop software: there is no OS to catch a bad pointer, a missed deadline is a bug, a stuck loop can mean a dead product in the field, and some mistakes damage hardware. So the bar is correctness and predictability first, then size/speed/power, then elegance.

This skill is the shared base. For toolchain, flashing and chip quirks, also use the chip skill:
- STM32 → `stm32-firmware`
- ESP32 → `esp32-firmware`
- PIC16/18/24/32 → `pic-firmware`

## Deeper references (optional personal library)

If the user has a personal technical library indexed locally (a `library-researcher` agent is available, or `~/.config/electronics-skills/library.json` exists), use it when a topic deserves a solid reference beyond this skill — an embedded C/C++ technique, an RTOS pattern, a peripheral explained in a vendor app note, a classic algorithm. Delegate to `library-researcher` (it returns answers with document and page citations) instead of reading large PDFs in the main conversation. Prefer current vendor documentation over older books for toolchain/API details. If no library is configured, skip this.

## Before writing code

Pin down the context, because the right answer depends on it. If something below matters for the task and is unknown, read the project to find it; ask only if the project cannot tell you.

- **Exact MCU part** (e.g. STM32G431KB, ESP32-S3-WROOM-1-N16R8, PIC18F47Q10) — memory, peripherals and errata differ per part.
- **Framework / generator**: CubeMX+HAL, ESP-IDF, Arduino, MCC, bare registers. Generated code has rules about where user code may go.
- **Superloop or RTOS** (FreeRTOS, Zephyr...). This changes how blocking, sharing data and timing work.
- **Language**: C (which standard: C99/C11) or C++ (which standard, exceptions/RTTI on or off).
- **Constraints**: RAM/flash budget, timing deadlines, power budget, safety/certification needs.
- **Board facts**: pin map, clocks/crystal, what is wired to what. Never invent a pin assignment silently — state assumptions.

## Core rules (and why)

1. **Fixed-width types** (`uint8_t`, `int32_t`, `bool` from `<stdint.h>`/`<stdbool.h>`). `int` is 16-bit on PIC16/18/24 and 32-bit on ARM/ESP32; code that assumes one breaks on the other.
2. **`volatile` for hardware registers and for variables shared with an ISR.** It stops the compiler caching the value. It does **not** make access atomic — see rule 3.
3. **Protect shared data.** A multi-byte variable shared between an ISR and main code (16-bit on an 8-bit PIC, 64-bit on a 32-bit MCU, or any read-modify-write) can be torn. Use a short critical section, a single-writer ring buffer, or RTOS primitives.
4. **ISRs are short.** Clear the flag, capture data, signal a task/flag/queue, return. No `printf`, no `malloc`, no delays, no blocking calls inside an ISR — they wreck latency or deadlock.
5. **No dynamic allocation after initialisation** (ideally none at all). Heap fragmentation on a device that runs for months is a delayed crash. Prefer static buffers and fixed pools.
6. **Everything that can wait has a timeout.** A peripheral that never answers must not hang the device. Check every return code (`HAL_StatusTypeDef`, `esp_err_t`, your own enums) and handle or propagate it.
7. **Keep hardware access behind a thin driver interface.** Application logic that does not touch registers can be unit-tested on a PC and ported between chips.
8. **Respect code generators.** CubeMX keeps only `/* USER CODE BEGIN */ ... /* USER CODE END */`; MCC overwrites `mcc_generated_files/`. Put real logic in your own files and call it from the allowed spots, otherwise the next regeneration silently deletes it.
9. **Watchdog on, reset cause logged.** In the field the watchdog is the last line of defence, and the reset reason (watchdog, brown-out, power-on, software) is often the only clue to a rare crash.
10. **Warnings are bugs.** Build with high warning levels and fix them, don't silence them.

## Workflow by task type

**Write**: confirm context → design data flow and ownership (who writes, who reads, from which context) → write driver/interface first, then logic → build with warnings on → add host unit tests for logic → describe what still needs hardware verification.

**Review**: look for, in order of severity: ISR/concurrency races, missing `volatile`/atomicity, unbounded waits, unchecked return codes, buffer overruns and integer overflow/sign issues, stack-heavy code (large locals, recursion), code placed where a generator will delete it, then style. Report findings first, with file:line, most severe first.

**Debug**: reproduce → classify (build / flash / boot / runtime / intermittent) → collect evidence (reset cause, fault registers, logs, scope/logic-analyser trace) → one hypothesis at a time → instrument rather than rewrite → fix → add a regression test where possible. Details in `references/debugging.md`.

**Optimize**: measure first, change one thing, measure again. Details in `references/optimization.md`.

## Reference files — read when relevant

| File | Read when |
|---|---|
| `references/coding-standards.md` | Writing or reviewing code; choosing compiler warnings; MISRA/CERT/Barr questions; using C++ on an MCU |
| `references/architecture.md` | Structuring a project, drivers, state machines, ISR↔task communication, RTOS vs superloop, error handling |
| `references/testing.md` | Adding unit tests, making code testable, PlatformIO `pio test`, testing CubeMX/MPLAB code on a PC, CI, static analysis |
| `references/debugging.md` | Crashes, hard faults, resets, hangs, timing bugs, peripheral not working |
| `references/optimization.md` | Code too big, too slow, too much RAM, battery life |

## Output expectations

- Code compiles for the stated target and framework; include the includes.
- State assumptions (clock, pins, part) at the top of the answer when you had to make them.
- After changes, build. If you could not build or could not test on hardware, say so plainly — "untested on hardware" is useful information, false confidence is not.
- Do not flash a board, erase flash, or change option bytes / config fuses / eFuses without the user confirming. Some of these (STM32 RDP level 2, ESP32 eFuses, PIC code protection) are permanent.
