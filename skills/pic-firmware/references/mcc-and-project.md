# MPLAB X project structure and MCC

## Contents
1. Project layout
2. MCC Melody vs Classic
3. Regeneration rules — where your code goes
4. Peripheral patterns with MCC drivers
5. Non-volatile memory (EEPROM / flash)
6. Low power

---

## 1. Project layout

```
myproduct.X/
├── nbproject/                    ← project config (configurations.xml = part, compiler, options). Commit it
│   └── Makefile-*.mk             ← generated; can be regenerated headless
├── mcc_generated_files/          ← MCC output — do not edit (except where MCC marks user regions)
├── myproduct.mc3                 ← MCC configuration — commit it (source of truth like CubeMX .ioc)
├── main.c                        ← thin: SYSTEM_Initialize(); app_init(); while(1) app_task();
├── app/                          ← YOUR code: app.c/.h, protocol, sensor drivers, state machines
└── dist/, build/                 ← output — don't commit
```
Add `app/` files to the project (Source Files → Add Existing Item) so they are in `configurations.xml`; add include directories in Project Properties → XC8 Compiler → Include directories.

## 2. MCC Melody vs Classic

| | Melody | Classic |
|---|---|---|
| Devices | Newer PIC16/18 (Q-series, K42...), many PIC24/dsPIC33, AVR | Older PIC16/18, PIC24, some PIC32 |
| Output | `mcc_generated_files/<peripheral>/src/*.c`, `system/` (clock, pins, config bits, interrupts) | flatter `mcc_generated_files/*.c`, `device_config.c`, `pin_manager.c` |
| APIs | Driver-style with callbacks: `UART1_Write()`, `TMR0_OverflowCallbackRegister()` | `EUSART1_Write()`, `TMR0_SetInterruptHandler()` |
| Config bits | `system/config_bits.c` | `device_config.c` (PIC16/18) |

Use what the project already uses; don't mix flavours in one project.

## 3. Regeneration rules — where your code goes

- Everything in `mcc_generated_files/` can be overwritten on Generate. MCC shows a merge window for files it changed; review it — accepting blindly can drop your edits, rejecting can leave stale config.
- Configure pins, clocks, peripherals, interrupts **in MCC**, then Generate — don't hand-edit generated init code.
- Hook application behaviour through **callbacks** that MCC drivers provide, defined in your own files:
  ```c
  /* app.c */
  static void on_tmr0(void) { g_tick++; }       /* runs in interrupt context: keep it short */
  void app_init(void) {
      TMR0_OverflowCallbackRegister(on_tmr0);   /* Melody; Classic: TMR0_SetInterruptHandler(on_tmr0) */
      INTERRUPT_GlobalInterruptEnable();
  }
  ```
- Pin names: give pins custom names in MCC Pin Grid/Pins view (e.g. `LED_STATUS`), then use generated macros `LED_STATUS_SetHigh()`, `LED_STATUS_Toggle()`, `BUTTON_GetValue()` — survives pin moves.
- Commit `.mc3` and generated files together so a build doesn't depend on someone re-running MCC.

## 4. Peripheral patterns with MCC drivers

**UART (EUSART/UART)**
- Enable the RX interrupt in MCC so received bytes go into the driver's ring buffer; poll `UART1_IsRxReady()` / `EUSART1_is_rx_ready()` and read in the main loop.
- `printf` redirection: MCC has an option "Redirect STDIO to UART" (or implement `putch(char c)` for XC8 / `write()` for XC16).
- Check the baud error % shown by MCC (keep < 2%); wrong `Fosc` = garbage.
- Handle framing/overrun errors (OERR on EUSART needs `CREN` toggled to clear) — the MCC driver has error callbacks.

**I2C**
- Use the MCC I2C Host driver's non-blocking API, check return/error status (NACK, bus collision), always with timeouts.
- 7-bit addresses in MCC APIs (driver handles the R/W bit) — check the driver header to be sure.
- External pull-ups (2.2–4.7 kΩ) required.

**ADC / ADCC**
- Set pin as analog (ANSEL = 1, TRIS = 1), choose positive reference (VDD, FVR), acquisition time per source impedance.
- ADCC (computation ADC on newer PIC16/18) can average/filter in hardware — use it instead of software averaging.
- Use FVR (fixed voltage reference) when VDD is not stable (battery).

**Timers / PWM / CCP**
- Let MCC compute period registers from the requested period; verify the "actual" value it reports.
- Use a timer interrupt for a system tick (1 ms) and do timing with `now - start >= period` comparisons in the main loop.
- Newer parts have CLC, CWG, NCO, SMT peripherals — often a hardware solution exists for what would be a tight software loop.

## 5. Non-volatile memory (EEPROM / flash)

- Parts with data EEPROM: MCC NVM driver (`NVM_UnlockKeySet`, `EEPROM_Write/Read` per flavour). Limited endurance (~100k writes) — write on change only.
- Parts without EEPROM: high-endurance flash (HEF/SAF) rows or regular flash rows via MCC NVM driver; erase a row before writing; reserve the area so code doesn't use it (linker option `-mreserve=` / "ROM ranges" in XC8 options).
- Wrap stored data with a version byte and a checksum/CRC; validate at boot and fall back to defaults.
- Writes stall the CPU on many PIC16/18 — not inside time-critical sections.

## 6. Low power

- `SLEEP();` stops the core; peripherals clocked from LFINTOSC/SOSC/WDT can keep running and wake it.
- Disable unused modules with **PMD** (Peripheral Module Disable) registers — MCC "PMD" module.
- Unused pins: output low, or input with pull-up/analog — never floating digital inputs.
- Lower the clock or use Doze/Idle modes for "awake but slow" phases.
- XLP (eXtreme Low Power) parts reach tens of nA in sleep — only if BOR/WDT/FVR configuration allows (each adds current).
