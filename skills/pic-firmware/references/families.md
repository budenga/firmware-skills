# PIC families: syntax and specifics

## Contents
1. Family overview
2. PIC16 / PIC18 (XC8)
3. PIC24 / dsPIC (XC16, XC-DSC)
4. PIC32 (XC32)
5. Reset causes per family
6. XC8 memory and optimization tips

---

## 1. Family overview

| Family | Core | Compiler | Int size | HW multiply | Notes |
|---|---|---|---|---|---|
| PIC10/12/16 (baseline, mid-range) | 8-bit | XC8 | `int` = 16-bit | no | Old PIC16: 8-level stack, no LAT on some |
| PIC16F1xxxx (enhanced mid-range) | 8-bit | XC8 | 16-bit | no | 16-level stack, LAT registers, PPS on newer parts |
| PIC18 (K22, K40, K42, Q10, Q43, Q84...) | 8-bit | XC8 | 16-bit | 8×8 | 31-level stack, priority or vectored interrupts (K42/Q), DMA on some Q parts |
| PIC24F/E | 16-bit | XC16 | 16-bit | yes | Software stack in RAM, vectored interrupts with priorities, traps |
| dsPIC33F/EP/EV/CK/CH | 16-bit + DSP engine | MPLAB XC-DSC (older projects: XC16) | 16-bit | yes + MAC, 40-bit accumulators | PIC24 core rules apply; see `dspic33.md` for DSP, PLL, PWM/ADC, dual-core |
| dsPIC33A | 32-bit DSC | MPLAB XC-DSC | 32-bit | yes + FPU | Newer family — see `dspic33.md` and the datasheet |
| PIC32MX/MZ/MK/MM | 32-bit MIPS | XC32 | 32-bit | yes, FPU on MZ EF | Harmony 3 framework; MZ has cache |
| PIC32C / PIC32CM/CZ/CX (Arm Cortex-M) | 32-bit Arm | XC32 | 32-bit | yes | Harmony 3/MCC; Cortex-M rules apply (see stm32-firmware fault analysis for Arm fault registers) |

## 2. PIC16 / PIC18 (XC8)

**Header and clock**
```c
#include <xc.h>
#include <stdint.h>
#define _XTAL_FREQ 32000000UL     /* Fosc in Hz — must match the config/MCC clock */
/* __delay_ms(10); __delay_us(50);  — argument must be a compile-time constant */
```
MCC Melody usually defines `_XTAL_FREQ` in `mcc_generated_files/system/clock.h`; don't redefine it differently.

**Interrupts**
```c
/* PIC16 (single vector) */
void __interrupt() isr(void) {
    if (PIE0bits.TMR0IE && PIR0bits.TMR0IF) {
        PIR0bits.TMR0IF = 0;
        g_tick++;
    }
    if (PIE3bits.RC1IE && PIR3bits.RC1IF) {
        rb_push(RC1REG);              /* reading RCxREG clears RCxIF */
    }
}

/* PIC18 legacy two-level priority (IPEN = 1) */
void __interrupt(high_priority) isr_high(void) { /* ... */ }
void __interrupt(low_priority)  isr_low(void)  { /* ... */ }

/* PIC18 K42/Q-series with vectored interrupts (MVECEN config on) */
void __interrupt(irq(TMR0), base(8)) tmr0_isr(void) { PIR3bits.TMR0IF = 0; /* ... */ }
```
Register/bit names (PIE0/PIR3, RC1REG vs RCREG1 vs U1RXB) vary per part — check the device header. With MCC, prefer registering callbacks (`TMR0_OverflowCallbackRegister(...)` / `..._SetInterruptHandler(...)`) instead of writing the ISR yourself.

Global enable: `INTCONbits.GIE = 1; INTCONbits.PEIE = 1;` (PIC16), `INTCON0bits.GIE = 1` (PIC18 Q), or `ei()` / `di()` macros.

**Atomic read of a 16-bit variable shared with an ISR**
```c
static volatile uint16_t g_tick;
uint16_t tick_get(void) {
    uint16_t t;
    PIE0bits.TMR0IE = 0;      /* mask only this source */
    t = g_tick;
    PIE0bits.TMR0IE = 1;
    return t;
}
```

**Stack and functions**
- No recursion. Avoid long call chains in ISRs.
- XC8 uses a *compiled stack* (autos statically placed from the call graph) by default on many devices. A function called from both main and an ISR gets duplicated or triggers warnings — keep ISR-called helpers separate and small.
- Read the build output: warning **(1393) possible hardware stack overflow detected; estimated stack depth: N**.
- Set `STVREN = ON` so a stack overflow resets the chip (detectable via STKOVF) instead of corrupting execution.

**I/O registers**: `TRISx` (1 = input), `LATx` (write), `PORTx` (read), `ANSELx` (1 = analog, default!), `WPUx` (weak pull-up), `ODCONx` (open drain), `SLRCONx`, `INLVLx` (input level). On parts without LAT: keep a shadow byte and write the whole port.

**PPS (Peripheral Pin Select)** on PPS devices:
```c
/* inputs: <peripheral>PPS = pin code ; outputs: RxyPPS = peripheral code */
/* unlock (interrupts off), assign, lock — MCC generates this */
PPSLOCK = 0x55; PPSLOCK = 0xAA; PPSLOCKbits.PPSLOCKED = 0;
RX1PPS = 0x17;        /* example: RC7 → EUSART1 RX (codes are part-specific!) */
RC6PPS = 0x0F;        /* example: RC6 ← TX1 */
PPSLOCK = 0x55; PPSLOCK = 0xAA; PPSLOCKbits.PPSLOCKED = 1;
```

**Watchdog**: `CLRWDT();` in the main loop. WDT period and enable from config bits (`WDTE`, `WDTCPS`) or `WDTCON0`.

**Sleep**: `SLEEP();` (macro) — wake by interrupt-capable peripherals (WDT, INT pin, IOC, UART on some). Put `NOP();` after `SLEEP();` as the datasheet recommends.

## 3. PIC24 / dsPIC (XC16, XC-DSC)

```c
#include <xc.h>
#define FCY 16000000UL        /* instruction clock = Fosc/2 on PIC24/dsPIC */
#include <libpic30.h>          /* __delay_ms(), __delay_us() — FCY must be defined first */

#pragma config FNOSC = FRCPLL, FWDTEN = OFF   /* example — real names per device */

void __attribute__((interrupt, no_auto_psv)) _T1Interrupt(void) {
    IFS0bits.T1IF = 0;
    g_tick++;
}
```
- Priorities 1–7 in `IPCx` registers; 0 disables. CPU priority in `SR.IPL`.
- Short critical section: `__builtin_disi(0x3FFF);` … `DISICNT = 0;` disables priority ≤ 6 interrupts; or save/raise IPL.
- **Traps** — define them, otherwise the default handler resets silently:
  ```c
  void __attribute__((interrupt, no_auto_psv)) _AddressError(void) { INTCON1bits.ADDRERR = 0; trap_record(1); }
  void __attribute__((interrupt, no_auto_psv)) _StackError(void)   { INTCON1bits.STKERR  = 0; trap_record(2); }
  void __attribute__((interrupt, no_auto_psv)) _MathError(void)    { INTCON1bits.MATHERR = 0; trap_record(3); }
  void __attribute__((interrupt, no_auto_psv)) _OscillatorFail(void){ INTCON1bits.OSCFAIL = 0; trap_record(4); }
  ```
  Address error = word access at an odd address (casting `uint8_t*` to `uint16_t*`, packed structs). Get the faulting address from the stack in the debugger.
- 16-bit reads are atomic; 32-bit (`uint32_t`, `float`) shared with ISRs need protection.
- `const` data in program memory is accessed through PSV/EDS — XC16 handles it; `no_auto_psv` on ISRs that don't read const data saves cycles.
- dsPIC33: everything above applies; DSP engine, clocking, PWM/ADC and dual-core specifics are in `dspic33.md`.

## 4. PIC32 (XC32)

MIPS-based (MX/MZ/MK/MM):
```c
#include <xc.h>
#include <sys/attribs.h>

void __ISR(_TIMER_1_VECTOR, IPL4SOFT) timer1_isr(void) {   /* IPL must match IPC1bits.T1IP = 4 */
    IFS0CLR = _IFS0_T1IF_MASK;                              /* atomic CLR/SET/INV registers */
    g_tick++;
}

void interrupts_init(void) {
    INTCONSET = _INTCON_MVEC_MASK;   /* multi-vector mode */
    __builtin_enable_interrupts();
}
```
- Use the `xxxCLR` / `xxxSET` / `xxxINV` registers for atomic bit changes on SFRs.
- **PIC32MZ cache**: DMA buffers must be coherent — declare `__attribute__((coherent))` or access through KSEG1 (uncached); DMA registers take **physical** addresses (`KVA_TO_PA(ptr)`).
- Most new PIC32 projects use **Harmony 3** (configured through MCC): treat its generated `config/` and `firmware/src/config` like MCC output — application code in `app.c` / your own files.
- Arm-based PIC32C*: Cortex-M; use CMSIS-style handlers and Arm fault analysis.

## 5. Reset causes per family

Read at the top of `main()`, store/log, then reset the flags for next time.

| Family | Register / bits |
|---|---|
| Enhanced PIC16F1 / PIC18 Q/K4x | `PCON0`: `STKOVF`, `STKUNF`, `nWDTWV`, `nRWDT` (WDT), `nRMCLR` (MCLR), `nRI` (RESET instruction), `nPOR`, `nBOR` — active-low bits are cleared by the event; set them back to 1 after reading |
| Older PIC16 | `PCON`: `nPOR`, `nBOR`; `STATUS`: `nTO` (WDT time-out), `nPD` |
| Older PIC18 | `RCON`: `nTO`, `nPD`, `nPOR`, `nBOR`, `nRI`; `STKPTR`: `STKFUL`, `STKUNF` |
| PIC24 / dsPIC | `RCON`: `TRAPR`, `IOPUWR`, `CM`, `EXTR`, `SWR`, `WDTO`, `SLEEP`, `IDLE`, `BOR`, `POR` |
| PIC32MX/MZ | `RCON`: `POR`, `BOR`, `WDTO`, `SWR`, `EXTR`, `CMR`, ... |

Exact names differ per device — check the header.

## 6. XC8 memory and optimization tips

- **Memory summary** at the end of each build shows program and data space used — watch it after every change on small parts.
- `const` tables are placed in program memory automatically (no `PROGMEM`-style keyword needed); reading them costs a few cycles.
- Use the smallest type that fits (`uint8_t` loops on 8-bit cores are much cheaper than `int`).
- Avoid `float`, division and modulo by non-powers-of-two on PIC16 (no hardware divide/multiply): fixed-point and shifts.
- `printf` is large on 8-bit parts; write small formatting helpers or send raw values.
- Bit variables: `__bit` type or bit-fields in a struct for flags; single-bit SFR access (`LATBbits.LATB0 = 1`) compiles to one `BSF`/`BCF` instruction (atomic).
- Optimization levels: since July 2026 Microchip makes all XC compiler optimisations free (XC8 v4.00+ needs no licence key). Older installed versions in Free mode are limited (XC8 typically -O0…-O2; XC16/XC32 typically -O0/-O1), with `-O3`/`-Os` needing PRO. Check the compiler version and the licence mode printed in the build output; upgrading the compiler may unlock `-Os`.
- Large arrays may be split across RAM banks; XC8 handles it but access is slower — keep hot data small.
