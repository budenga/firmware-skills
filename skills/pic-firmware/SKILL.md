---
name: pic-firmware
description: Microchip PIC and dsPIC firmware with MPLAB X IDE, XC8 / XC16 / XC-DSC / XC32 compilers and MPLAB Code Configurator (MCC Melody/Classic) for PIC16, PIC18, PIC24, dsPIC33 (F/EP/EV/CK/CH dual-core/A) and PIC32 — configuration bits, oscillator/PLL, TRIS/LAT/PORT/ANSEL, PPS, interrupts per family, 8/16-bit atomicity and hardware-stack limits, dsPIC DSP engine and Q15 fixed-point, high-speed PWM/ADC control loops, motor control and digital power, XC8 memory and optimization, headless builds, programming and debugging with PICkit 4/5, MPLAB Snap and ICD (ipecmd), reset causes and traps. Use this skill whenever the user mentions any PIC or dsPIC part (PIC16F, PIC18F, PIC24F, dsPIC33EP/CK/CH, PIC32MX/MZ), MPLAB X, MCC, XC8/XC16/XC-DSC/XC32, PICkit, Snap, ICD, `#pragma config`, motorBench, or a .X project folder — even for a short question like "o pino não lê nada no PIC" or "o meu controlo PI no dsPIC oscila". Use together with embedded-c-cpp for general C quality, testing and architecture.
---

# PIC & dsPIC firmware (MPLAB X + XC + MCC; PICkit / Snap / ICD)

Also load `embedded-c-cpp` for coding standards, architecture, unit testing and general debugging. This skill holds the Microchip-specific parts. PIC firmware is written in **C** (XC8 has no C++).

## First, identify the project

| Find | Where | Why it matters |
|---|---|---|
| **Exact part** (PIC16F18877, PIC18F47Q10, PIC24FJ128GA010, dsPIC33CK256MP508, PIC32MX795F512L...) | `nbproject/configurations.xml` (`<targetDevice>`), MCC `.mc3`, user | Registers, peripherals, PPS, interrupt model, memory all differ per part — on dsPIC33 the F, E, C and A families have different PWM/ADC/PLL modules |
| Compiler + version | `configurations.xml` (`<languageToolchain>`, `<languageToolchainVersion>`) | XC8 vs XC16 vs XC-DSC (dsPIC) vs XC32; optimization levels and syntax differ |
| MCC flavour | `mcc_generated_files/` layout, `.mc3` file; Melody (newer: `system/`, `timer/` subfolders, `*_Initialize`) vs Classic | Where config bits and drivers live, API names |
| Clock | MCC clock settings / `#pragma config` (FEXTOSC, RSTOSC / FNOSC / POSCMOD), `_XTAL_FREQ` or `FCY` define | Delays, baud rates, timer periods |
| Programmer/debugger | PICkit 4/5, MPLAB Snap, ICD 4/5 | Power and high-voltage programming capability |

When a register or bit name is uncertain, say which datasheet section to check (or read the device header `pic16f18877.h` etc. in the compiler's include directory) rather than guessing — PIC register names vary between sub-families.

## Rules that prevent the classic PIC failures

1. **Digital input reads 0 → ANSEL.** On most modern PICs, pins power up as **analog**. Clear the `ANSELx` bit (MCC: uncheck "Analog") for every digital input.
2. **Write `LATx`, read `PORTx`.** Writing `PORTx` bits does read-modify-write of the pin levels and can flip other outputs (heavy loads, fast toggling). Parts without LAT registers need a shadow variable.
3. **TRIS: 1 = input, 0 = output.** Easy to invert.
4. **Peripheral Pin Select (PPS)** must map UART/SPI/PWM signals to pins on PPS devices; with `PPS1WAY = ON` it can be unlocked only once after reset — let MCC generate it.
5. **Config bits** decide oscillator, watchdog, MCLR, LVP, brown-out, code protection. Set them in MCC (or the Configuration Bits window → Generate Source Code), never leave them to defaults you did not check. Wrong oscillator = nothing runs or every timing is off.
6. **`_XTAL_FREQ` (XC8) / `FCY` (XC16) must equal the real clock** — `__delay_ms()` and many calculations depend on it.
7. **8-bit atomicity**: on PIC16/18 any `uint16_t`/`uint32_t` shared with an ISR is read in several instructions — disable that interrupt (or GIE) around the read. On PIC24/dsPIC the same applies to 32-bit values.
8. **Hardware call stack is tiny** on 8-bit PICs (8 levels on old PIC16, 16 on enhanced PIC16F1, 31 on PIC18) and the ISR uses levels too. No recursion; watch XC8 warning **(1393) possible hardware stack overflow** and the estimated depth in the build output.
9. **Interrupt flags**: in the ISR, check both enable and flag bits (`if (PIEx.xIE && PIRx.xIF)`), clear the flag, keep it short. Enable peripheral + global interrupts (MCC: `INTERRUPT_GlobalInterruptEnable()`).
10. **MCC regeneration overwrites `mcc_generated_files/`.** Put application code in your own `.c/.h` files; keep `main.c` thin; review the MCC merge dialog instead of accepting blindly.
11. **Irreversible / risky settings**: code protection (CP), write protection, disabling MCLR together with LVP off, oscillator settings that stop the chip. Don't enable code protection or change these on a user's device without explicit confirmation; a Snap cannot recover a part that needs high-voltage programming.

Family-specific syntax (interrupts, delays, config, memory): `references/families.md`.

## dsPIC33 (control, DSP, motor control, digital power)

dsPIC33 uses the PIC24 core syntax plus a DSP engine. Extra rules:
- **FCY = FOSC/2**; PLL setup and clock switching differ per family (EP vs CK/CH) — generate with MCC and verify the reported frequencies, including the auxiliary clock that drives PWM/ADC.
- Use **Q15 fixed-point** (DSP library `dsp.h`, `libq.h`, builtins) instead of `float` in fast loops; document every variable's scaling.
- **Clamp integrators and outputs explicitly in C** — normal C arithmetic wraps; `CORCON` saturation only protects DSP-engine operations.
- Control loop = PWM-triggered ADC interrupt at high priority; measure its execution time on a scope.
- **Configure PWM fault/current-limit inputs before enabling outputs** on any power stage, and test dead time at low voltage first — wrong PWM code destroys hardware.
- Enable the catastrophic accumulator-overflow trap (`COVTE`); leave `OVATE/OVBTE` off for normal MAC code (guard bits are meant to be used).
- dsPIC33CK/CH alternate register sets use `__attribute__((interrupt, context))` with matching ISR priority — not `shadow`.
- dsPIC33CH is dual-core: two projects, pin ownership, MSI mailboxes.

Details: `references/dspic33.md`.

## Build, program, debug

| Action | How |
|---|---|
| Build in IDE | MPLAB X → Clean and Build (hammer+broom). Check the **Memory Summary** and warnings in the output |
| Build headless | `prjMakefilesGenerator.sh <proj>.X@default` then `make -C <proj>.X CONF=default` → `dist/default/production/<proj>.X.production.hex` |
| Program | MPLAB X "Make and Program Device", MPLAB IPE, or `ipecmd` from the command line |
| Debug | MPLAB X Debug Main Project with PICkit/Snap/ICD (debug build), or the **MPLAB X Simulator** for pure logic/timers without hardware |

Programming changes the device: confirm the part and programmer with the user before running `ipecmd`, and never erase/protect without confirmation. Full commands, programmer differences and connection troubleshooting: `references/build-program-debug.md`.

## When it doesn't work

- Nothing runs: config bits (oscillator source, WDT, MCLR), power, MCLR pull-up (10 kΩ), programmer released the device from reset (IPE "Release from Reset" / `-OL`).
- Runs at wrong speed / wrong baud: `RSTOSC`/`FNOSC`, PLL, clock divider (CDIV), `_XTAL_FREQ` mismatch.
- Random resets: read the reset-cause bits at boot (PCON/PCON0/RCON — `references/families.md`), watchdog not cleared, stack overflow (STKOVF), brown-out.
- PIC24/dsPIC sudden reset or hang: install trap handlers (`_AddressError`, `_StackError`, `_MathError`, `_OscillatorFail`) — address errors usually mean a misaligned 16-bit access.

## Reference files

| File | Read when |
|---|---|
| `references/families.md` | Writing/reviewing code for a specific family: interrupt syntax, delays, config bits, memory model, atomicity, reset causes, traps, PIC32 cache/DMA, XC8 optimization and memory tips |
| `references/dspic33.md` | Any dsPIC33 work: families, XC-DSC, PLL/clock switching, DSP engine and Q15 math, ISR context/shadow registers, math traps, DMA, high-speed PWM/ADC control loops, dual-core CH, motorBench/power tools, review checklist |
| `references/mcc-and-project.md` | Project structure, MCC Melody vs Classic, regeneration rules, where to put code, peripheral drivers (UART, I2C, ADC, timers, PWM, NVM/EEPROM), low power |
| `references/build-program-debug.md` | Installing tools, headless builds, `ipecmd` programming, PICkit/Snap/ICD differences, debugging limits, "cannot connect / device ID mismatch" troubleshooting, unit testing PIC code on the PC |
