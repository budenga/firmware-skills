# Build, program and debug PIC

## Contents
1. Tools (macOS)
2. Headless build
3. Programmers: PICkit 4/5, Snap, ICD
4. Programming from the command line (ipecmd)
5. Debugging
6. "Cannot connect" troubleshooting
7. Unit testing PIC code on the PC

---

## 1. Tools (macOS)

| Tool | Notes |
|---|---|
| MPLAB X IDE (includes IPE) | Default install `/Applications/microchip/mplabx/v6.xx/` |
| XC8 / XC16 (or XC-DSC) / XC32 | Separate installers; `/Applications/microchip/xc8/v3.xx/bin/xc8-cc` etc. Select per project in Project Properties |
| MCC | Plugin inside MPLAB X (Tools → Plugins) |
| Device Family Packs (DFP) | Tools → Packs; the part's DFP version is in `configurations.xml` — mismatched packs cause "device not found" errors |
| MPLAB extensions for VS Code | Microchip's newer VS Code tooling — alternative to the NetBeans-based IDE |

Find installed versions: `ls /Applications/microchip/*/`.

## 2. Headless build

```
MPLABX=/Applications/microchip/mplabx/v6.25            # adjust version
"$MPLABX/mplab_platform/bin/prjMakefilesGenerator.sh" myproduct.X@default   # regenerate makefiles from configurations.xml
make -C myproduct.X CONF=default clean
make -C myproduct.X CONF=default                         # production build
ls myproduct.X/dist/default/production/                  # .hex, .elf, .map
```
`default` is the configuration name (Project Properties → Manage Configurations). The build output ends with a memory summary; the `.map` file has per-symbol sizes and (XC8) the call graph / stack depth.

## 3. Programmers: PICkit 4/5, Snap, ICD

| Tool | Powers target? | High-voltage (HV) programming | Notes |
|---|---|---|---|
| PICkit 5 / PICkit 4 | Yes, small current (check limit, ~50–150 mA) | Yes | General choice; PICkit 5 supports newer parts and Programmer-to-Go |
| MPLAB Snap | **No** (target must be self-powered) | **No** — needs low-voltage programming (LVP) enabled on the part | Cheap; can't recover a part with LVP disabled, and can't program HV-only parts (e.g. old PIC16F8x) at all |
| MPLAB ICD 4 / ICD 5 | Yes | Yes | Faster, more robust, isolated options |
| PICkit 3 | Yes | Yes | Legacy; not supported for many new devices in recent MPLAB X |

ICSP connection: MCLR/VPP (10 kΩ pull-up to VDD, no large capacitor), VDD, VSS, ICSPDAT/PGD, ICSPCLK/PGC. Keep PGD/PGC free of heavy loads (LEDs, capacitors) or isolate them with resistors.

## 4. Programming from the command line (ipecmd)

```
IPE=/Applications/microchip/mplabx/v6.25/mplab_platform/mplab_ipe/bin   # find: find /Applications/microchip -name 'ipecmd*'
"$IPE/ipecmd.sh" -TPPK5 -P18F47Q10 -F"myproduct.X/dist/default/production/myproduct.X.production.hex" -M -Y -OL
```
| Option | Meaning |
|---|---|
| `-TP<tool>` | Programmer: e.g. `PK4`, `PK5`, `ICD4`, `ICD5`, `SNAP`, `PKOB` (on-board) — confirm codes with `ipecmd.sh -?` for your version |
| `-P<part>` | Device without "PIC" prefix: `16F18877`, `18F47Q10`, `24FJ128GA010`, `32MX795F512L` |
| `-F"file.hex"` | Hex file |
| `-M` | Program entire device (`-MP` program memory only, `-ME` EEPROM, `-MC` config) |
| `-Y` | Verify |
| `-E` | Erase (destructive — only when asked) |
| `-OL` | Release from reset after programming (so the code starts) |
| `-W` | Power target from the tool (PICkit/ICD only; not Snap) — check voltage and current needs first |

Confirm the device and programmer with the user before running this; never add code-protect or erase options they didn't ask for.

## 5. Debugging

- Use a **debug build** (Debug Main Project). The debug executive reserves a little program memory, RAM and the PGC/PGD pins; code that uses those pins can't be debugged in-circuit.
- Breakpoints are limited on 8-bit parts (often 1–3 hardware breakpoints; software breakpoints depend on the tool/part). Use them sparingly; use watch windows and the stopwatch (cycle counter).
- MCLR must be enabled (`MCLRE = ON`) for debugging on most parts.
- Code protection must be off.
- **MPLAB X Simulator**: debug logic, timers and some peripherals without hardware, with stimulus files — good for algorithm and ISR-flow checks.
- Logging: UART `printf` or toggling a spare pin and measuring with a scope/logic analyser (lowest overhead).
- Data Visualizer (MPLAB X plugin) plots variables streamed over UART.

## 6. "Cannot connect" troubleshooting

| Message / symptom | Check |
|---|---|
| `Target device ID (0x0) is an invalid device ID` / `Failed to get Device ID` | Power (Snap doesn't power the target), ICSP wiring/order, MCLR pull-up, PGC/PGD loaded by other circuitry, cable length |
| `Device ID mismatch` | Wrong part selected in project, or wrong chip on the board |
| Works once, then can't connect | Firmware reconfigured PGC/PGD pins or disabled MCLR with LVP off — use HV-capable tool (PICkit/ICD) with "Hold in reset"/"Program with VPP first" options; Snap can't recover it |
| `The programmer could not be started: Could not connect to tool hardware` | USB/cable, tool firmware update (MPLAB X updates it on connect), another program holds the tool |
| Programs OK but nothing runs | Config bits (oscillator), `-OL` release from reset missing, MCLR held low |
| Voltage errors | Selected target voltage vs actual; low-voltage parts (1.8 V) vs 5 V settings |

## 7. Unit testing PIC code on the PC

XC8 code can be tested on the PC with Unity + CMake (see `embedded-c-cpp/references/testing.md` §4) if logic is separated from `xc.h`:
- Application modules include only `<stdint.h>`/`<stdbool.h>` and your HAL headers, never `<xc.h>`.
- Remember `int` is 16-bit on PIC16/18/24 and 32-bit on the PC: use fixed-width types in the code under test so overflow behaviour matches.
- XC8-specific keywords (`__bit`, `__interrupt`, `__at`) stay in hardware files only.
- For register-level code, the MPLAB X Simulator with stimulus is the practical alternative.
