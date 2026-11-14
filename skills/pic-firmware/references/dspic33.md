# dsPIC33 Digital Signal Controllers

dsPIC33 parts share the 16-bit PIC24 core (interrupts, traps, PSV, `RCON`, `__attribute__((interrupt))` — see `families.md` §3) and add a DSP engine, faster clocks and control-oriented peripherals (high-speed PWM, fast ADCs, comparators/DACs, op-amps, QEI, CAN-FD). This file covers what is **different**.

## Contents
1. Families and compiler
2. Clock and PLL
3. The DSP engine and fixed-point math
4. Interrupts, context and traps on dsPIC
5. DMA
6. Control peripherals: PWM, ADC, comparators
7. Dual-core dsPIC33CH
8. Tools: MCC, motor control and power libraries
9. Checklist for dsPIC code reviews

---

## 1. Families and compiler

| Family | Core / speed | Typical use | Notes |
|---|---|---|---|
| dsPIC33F (GP, MC, GS) | 16-bit, up to 40 MIPS | Legacy designs | DMA buffers must be in DMA RAM |
| dsPIC33EP / dsPIC33EV | 16-bit, up to 70 MIPS (≤ +85 °C; 60 MIPS on +125 °C extended-temperature parts) | Motor control, general DSP; **EV = 5 V operation**, CAN | |
| dsPIC33CK | 16-bit, up to 100 MIPS, single core | Motor control, digital power, CAN-FD | Recommended for most new 16-bit designs |
| dsPIC33CH | Dual core (main + secondary) | Digital power + communication, dual-motor | Two programs, one chip (§7) |
| dsPIC33A (e.g. dsPIC33AK) | 32-bit DSC core | High-performance control | Newer family — check the datasheet/compiler support carefully |

**Compiler**: **MPLAB XC-DSC** is Microchip's compiler for dsPIC devices (derived from XC16). Existing dsPIC projects may still use XC16; new dsPIC33 projects should use XC-DSC (required for dsPIC33A). Same C syntax for interrupts/attributes as XC16. Include `<xc.h>`; device header is selected from the project's part.

Exact peripherals and register names differ **a lot** between F, E, C and A families (e.g. PWM, ADC and PLL are completely different modules). Always identify the exact part number before writing register code, and prefer MCC Melody drivers where available.

## 2. Clock and PLL

- Instruction clock **FCY = FOSC / 2** (on dsPIC33A, check the datasheet — the core is different).
- Define `FCY` before `#include <libpic30.h>` for `__delay_ms()` / `__delay_us()`.
- Typical start-up: boot on FRC (`FNOSC = FRC`), allow clock switching (`FCKSM = CSECMD`), configure the PLL, then switch:
  ```c
  /* dsPIC33EP example: 7.37 MHz FRC → FOSC ≈ 140 MHz, FCY ≈ 70 MIPS. Values are part-specific. */
  CLKDIVbits.PLLPRE  = 0;     /* N1 = 2 */
  PLLFBD             = 74;    /* M  = 76 */
  CLKDIVbits.PLLPOST = 0;     /* N2 = 2 */
  __builtin_write_OSCCONH(0x01);              /* new oscillator: FRC + PLL */
  __builtin_write_OSCCONL(OSCCON | 0x01);     /* request switch */
  while (OSCCONbits.COSC != 0b001) {}         /* wait for switch */
  while (OSCCONbits.LOCK != 1) {}             /* wait for PLL lock */
  ```
  dsPIC33CK/CH use a different PLL (VCO with `PLLFBD`, `PLLDIV` POST1/POST2 dividers, and separate auxiliary PLL for PWM/ADC). **Let MCC generate the clock code** and verify the resulting FOSC/FCY it reports.
- Every wait loop above should have a timeout in production code; on timeout, fall back to FRC and log it (the `_OscillatorFail` trap also fires on clock failure if fail-safe clock monitor is enabled).
- High-speed PWM and ADC often run from an auxiliary clock (APLL) — their timing does not follow FCY.

## 3. The DSP engine and fixed-point math

Hardware: two 40-bit accumulators (A, B), single-cycle 17×17 multiply, MAC with dual data fetch from X and Y memory, barrel shifter, optional saturation and rounding. Modes are set in `CORCON` (fractional/integer, saturation on A/B, rounding mode).

**Fractional formats**
- **Q15**: `int16_t` interpreted as value/32768 → range −1.0 … +0.99997. Most DSP library functions use Q15 (`fractional` type in `dsp.h`).
- **Q31** for higher precision; accumulators hold 9.31 values (headroom for sums).
- XC16/XC-DSC also support Embedded-C fixed-point types (`_Fract`, `_Accum`, `<stdfix.h>`) — readable code that maps to the DSP engine; check the compiler user guide for enabling and performance.

**Ways to use the DSP engine, easiest first**
1. **Microchip DSP library** (`dsp.h`): FIR, IIR, FFT, vector/matrix operations in Q15. Filter coefficients from a design tool.
2. **libq** (`libq.h`): fixed-point math functions (sin, cos, sqrt, atan2, log) in Q15/Q16 — much faster than `float`.
3. **Compiler builtins** for custom loops:
   ```c
   #include <xc.h>
   /* y = Σ x[i] * h[i] in Q15 — simple form: NULL prefetch pointers mean NO dual fetch,
      x[i]/h[i] are loaded with ordinary moves. For true single-cycle dual fetch, pass
      prefetch pointers into X and Y memory (&xptr, &xval, 2, &yptr, &yval, 2) as shown
      in the compiler user guide's __builtin_mac example. */
   int16_t __attribute__((space(xmemory))) x[N];
   int16_t __attribute__((space(ymemory))) h[N];

   int16_t fir_q15(void) {
       register int acc __asm__("A");
       acc = __builtin_clr();
       for (uint16_t i = 0; i < N; i++) {
           acc = __builtin_mac(acc, x[i], h[i], NULL, NULL, 0, NULL, NULL, 0, 0, 0);
       }
       return __builtin_sac(acc, 0);   /* store with rounding/saturation per CORCON */
   }
   ```
   Builtin signatures are long and part of the compiler user guide — check them there; prefer the DSP library unless profiling shows a need.
4. **Assembly** for the tightest loops (control loops at 100+ kHz): `DO`/`REPEAT` loops, MAC with prefetch.

**Rules**
- **Overflow protection in control loops**: a wrapped PI output can drive a power stage to full duty.
  - Plain C arithmetic on `int16_t`/`int32_t` runs on the ALU and **wraps regardless of CORCON** — clamp the integrator (anti-windup) and the output explicitly in C.
  - `CORCON.SATA/SATB` saturation only applies to DSP-engine operations (MAC/MPY/accumulator ADD/SUB via builtins, `_Sat _Fract` types, DSP library, assembly); `SATDW` (set after reset) only affects the value stored by `SAC`.
  - So: either clamp in C, or do the loop on the DSP engine with saturation enabled — and still clamp the final duty cycle to its safe range.
- Scale signals to use the Q15 range without overflow; document the scaling of every variable (`/* Q15, 1.0 = 50 A */`).
- `float` exists but is slow (software emulation on 16-bit dsPIC33) — keep it out of fast loops. dsPIC33A has a hardware FPU — check before assuming.

## 4. Interrupts, context and traps on dsPIC

Same syntax as PIC24:
```c
void __attribute__((interrupt, no_auto_psv)) _ADCAN0Interrupt(void) { /* name per device header */
    int16_t sample = ADCBUF0;
    /* control loop step */
    IFS5bits.ADCAN0IF = 0;   /* flag register/bit per device */
}
```
- **`shadow` attribute** (`__attribute__((interrupt, shadow, no_auto_psv))`, dsPIC33F/E/C) uses the hardware shadow registers for fast context save of W0–W3 and SR — only one level, so use it on a single, highest-priority ISR (e.g. the control loop).
- **Alternate working register sets (dsPIC33CK/CH)** are a different mechanism: whole W-register sets assigned to interrupt priority levels with the `CTXTn` fields of the `FALTREG` configuration word, used by an ISR via `__attribute__((interrupt, context))` (not `shadow`). **The ISR's priority must match the level assigned to that context** — a mismatch corrupts registers at runtime. Configure with MCC and double-check priorities.
- **DSP state in ISRs**: if an ISR uses the accumulators or changes `CORCON`, and main code also uses the DSP engine, the ISR must preserve that state. Check the compiler user guide for what it saves automatically, save/restore `CORCON` explicitly, and prefer running all DSP code in one context.
- Interrupt priority 7 is not maskable by `DISI`; priority > CPU IPL preempts.
- **Traps to implement** (in addition to PIC24's `_AddressError`, `_StackError`, `_MathError`, `_OscillatorFail`):
  - `_MathError` also covers accumulator overflow when enabled in `INTCON1`:
    - **`COVTE`** (catastrophic overflow past bit 39 → `COVAERR`/`COVBERR`): enable it routinely — it always means a real scaling bug.
    - `OVATE`/`OVBTE` (overflow into the guard bits past bit 31 → `OVAERR`/`OVBERR`): **leave off for normal MAC/FIR code**, because using the guard bits during accumulation is intended. Enable only where the accumulator must never exceed 1.31.
  - `_DMACError` on dsPIC33F/E parts with DMA; dsPIC33CK/CH report such errors through their generic soft/hard trap vectors — check the device header.
  - Newer families add soft/hard trap vectors — check the device header for the exact names.

## 5. DMA

- **dsPIC33F parts with DMA** (not all have it, e.g. GS parts don't): DMA can only access **DMA RAM** — declare buffers with `__attribute__((space(dma)))`.
- dsPIC33E/C: DMA can reach most of data RAM, but check the "DMA" chapter for restrictions on EDS/extended memory and the peripheral's DMA request IDs.
- Ping-pong mode (two buffers) for continuous ADC capture; process one buffer while DMA fills the other.
- Treat buffers being filled by DMA as owned by hardware — no reads until the completion interrupt.

## 6. Control peripherals: PWM, ADC, comparators

- **High-speed PWM** (EP: PWM module with up to ~1 ns resolution on some parts; CK/CH: PWM generators PG1–PGx with PCI fault logic): configure period, dead time, complementary mode, **fault inputs that force outputs to a safe state** — always configure fault/current-limit before enabling outputs on a power stage.
- **ADC triggered by PWM** (trigger at the centre of the PWM period for clean current samples); the ADC-complete interrupt runs the control loop.
- Analog comparators + DACs for cycle-by-cycle current limiting in hardware (no software latency).
- **QEI** for encoder position on motor-control parts.
- Before enabling PWM outputs on a real power stage: verify dead time and fault shutdown on a scope at low voltage / current-limited supply. Code that is "almost right" can destroy MOSFETs.

## 7. Dual-core dsPIC33CH

- Two independent cores: **main** (traditionally "master") and **secondary** ("slave" in older docs), each with its own program flash/RAM, clock and peripherals.
- Two MPLAB X projects: the secondary project is added to the main project as a "secondary" image; the main core loads and starts it at runtime with functions from `libpic30.h` (`_program_secondary(...)`, `_start_secondary()` in current XC versions; older code uses `_program_slave` / `_start_slave`).
- Pin and peripheral ownership between the cores is set in configuration bits / MCC — a pin can only be driven by its owner.
- Inter-core communication: **MSI** (mailboxes + FIFOs with hardware handshakes). Define a small protocol (message IDs, version) and keep it in a shared header.
- Typical split: secondary core runs the fast control loop (digital power, motor), main core runs communication, supervision, logging.
- Debugging: one core at a time per tool session in most setups; plan logging over UART/MSI for the other.

## 8. Tools: MCC, motor control and power libraries

- **MCC Melody** supports dsPIC33CK/CH (and newer); Classic MCC for older E/F parts.
- **Motor control**: motorBench Development Suite (auto-tuning, generates code on the Motor Control Application Framework, MCAF), plus FOC application notes and reference designs per family.
- **Digital power / SMPS**: Digital Compensator Design Tool (DCDT) and the SMPS control library (2P2Z/3P3Z compensators in assembly).
- **MPLAB Data Visualizer / X2C-Scope** to watch control variables live while the loop runs.

## 9. Checklist for dsPIC code reviews

1. Exact part identified; register/peripheral names match its family (F/E/C/A).
2. FCY/FOSC correct and consistent with `FCY` define, MCC clock and PWM/ADC auxiliary clock.
3. Q-format of every fixed-point variable documented; integrator and output explicitly clamped (C arithmetic wraps — CORCON saturation only covers DSP-engine operations).
4. No `float` in fast loops (except dsPIC33A with FPU); no `printf` in ISRs.
5. Control-loop ISR priority and execution time measured (toggle a pin, scope it) — must finish well within the PWM/ADC period.
6. DSP state (accumulators, CORCON) not corrupted between ISR and main context; on CK/CH, `context` ISRs have the priority assigned to their register set.
7. PWM fault inputs / current limits configured before outputs are enabled; safe state on reset and on traps.
8. Trap handlers present, recording the cause; reset cause read at boot.
9. DMA buffers placed correctly for the family.
10. Dual-core: pin ownership, MSI protocol and secondary image loading verified.
