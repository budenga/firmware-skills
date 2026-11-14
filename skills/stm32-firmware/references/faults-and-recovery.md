# Faults, connection problems and recovery

## Contents
1. HardFault analysis (Cortex-M3/M4/M7/M33)
2. Cortex-M0/M0+ (F0, G0, L0, C0)
3. Capturing faults in the field
4. Common fault causes
5. ST-Link can't connect
6. Option bytes and read protection (RDP)

---

## 1. HardFault analysis (Cortex-M3/M4/M7/M33)

When the CPU faults it pushes R0–R3, R12, LR, PC, xPSR onto the active stack. **The stacked PC is the faulting instruction** (or close to it, for imprecise bus faults).

In the debugger, halted in `HardFault_Handler`:
```
(gdb) p/x $lr                      # EXC_RETURN: bit 2 = 0 → MSP was used, 1 → PSP (RTOS task)
(gdb) set $frame = ($lr & 4) ? $psp : $msp
(gdb) p/x ((uint32_t*)$frame)[6]   # stacked PC
(gdb) p/x ((uint32_t*)$frame)[5]   # stacked LR (caller)
(gdb) info symbol ((uint32_t*)$frame)[6]
(gdb) x/wx 0xE000ED28              # CFSR
(gdb) x/wx 0xE000ED2C              # HFSR
(gdb) x/wx 0xE000ED34              # MMFAR (valid if CFSR.MMARVALID)
(gdb) x/wx 0xE000ED38              # BFAR  (valid if CFSR.BFARVALID)
```
CubeIDE: Window → Show View → Fault Analyzer does the same automatically.

Decode CFSR (0xE000ED28):

| Bits | Name | Meaning |
|---|---|---|
| 0 IACCVIOL / 1 DACCVIOL | MemManage | Instruction/data access violation (MPU, execute-never region) |
| 7 MMARVALID | | MMFAR holds the bad address |
| 8 IBUSERR | BusFault | Instruction fetch from invalid memory (bad function pointer) |
| 9 PRECISERR | | Precise data bus error — BFAR has the address (null/invalid pointer) |
| 10 IMPRECISERR | | Imprecise — PC is after the faulting store. To find it during debug, disable write buffering (`SCnSCB->ACTLR |= SCnSCB_ACTLR_DISDEFWBUF_Msk;` on M3/M4) |
| 12 STKERR / 11 UNSTKERR | | Fault while stacking → **stack overflow** or corrupted SP |
| 15 BFARVALID | | BFAR valid |
| 16 UNDEFINSTR | UsageFault | Undefined instruction (corrupt code, jump into data) |
| 17 INVSTATE | | Thumb bit clear — calling a function pointer that is NULL/garbage or lost bit 0 |
| 18 INVPC | | Bad EXC_RETURN (stack corruption in an ISR) |
| 19 NOCP | | FPU instruction with FPU disabled (`SystemInit` must set CPACR; check `-mfpu`/`-mfloat-abi` match) |
| 24 UNALIGNED | | Unaligned access with UNALIGN_TRP set, or LDM/STRD unaligned (packed structs) |
| 25 DIVBYZERO | | Division by zero with DIV_0_TRP set |

HFSR bit 30 FORCED = a configurable fault escalated to HardFault (look at CFSR); bit 1 VECTTBL = bad vector table read.

Then: `arm-none-eabi-addr2line -e app.elf -f -C <stacked PC>` and `<stacked LR>`.

## 2. Cortex-M0/M0+ (F0, G0, L0, C0)

No CFSR/HFSR/BFAR — every fault is a HardFault. Use the stacked PC/LR (same EXC_RETURN logic, but M0 has no `ITE`; in code use explicit branches). Typical M0 causes: unaligned 16/32-bit access (M0 traps on all unaligned accesses — packed structs, casting `uint8_t*` buffers to `uint32_t*`), null pointer, stack overflow.

## 3. Capturing faults in the field

Untick "Generate IRQ handler" for Hard fault in the `.ioc` (NVIC → Code generation), then (M3/M4/M7/M33):

```c
/* fault.c */
typedef struct { uint32_t magic, pc, lr, cfsr, hfsr, bfar, mmfar; } fault_record_t;
__attribute__((section(".noinit"))) volatile fault_record_t g_fault;   /* needs a NOLOAD .noinit section in the linker script */

void hardfault_c(uint32_t *frame) {
    g_fault.pc    = frame[6];
    g_fault.lr    = frame[5];
    g_fault.cfsr  = SCB->CFSR;
    g_fault.hfsr  = SCB->HFSR;
    g_fault.bfar  = SCB->BFAR;
    g_fault.mmfar = SCB->MMFAR;
    g_fault.magic = 0xDEADFA17u;
#ifdef DEBUG
    __BKPT(0);
#endif
    NVIC_SystemReset();
}

__attribute__((naked)) void HardFault_Handler(void) {
    __asm volatile(
        "tst lr, #4      \n"
        "ite eq          \n"
        "mrseq r0, msp   \n"
        "mrsne r0, psp   \n"
        "b hardfault_c   \n");
}
```
At boot, if `g_fault.magic == 0xDEADFA17`, log the record, then clear `magic`. Point the other fault handlers (MemManage/BusFault/UsageFault) at the same code or enable them separately in `SCB->SHCSR` for finer reporting.

## 4. Common fault causes

| Clue | Likely cause |
|---|---|
| PC = 0 or odd small value, INVSTATE/IBUSERR | Calling a NULL / uninitialised function pointer or callback |
| BFAR ≈ 0x00000000–0x00000100, PRECISERR | NULL pointer dereference (struct member offset) |
| STKERR, SP near RAM start | Stack overflow (big local arrays, deep calls, printf) — enlarge `_Min_Stack_Size` or FreeRTOS task stack, check high-water mark |
| Fault only with `-O2`/`-Os` | Missing `volatile`, uninitialised variable, UB |
| Fault in `HAL_xxx_IRQHandler` | Handle struct not initialised (`MX_xxx_Init` not called before enabling IRQ), or DMA writing into a freed/stack buffer |
| Fault right after enabling FPU math | NOCP: FPU not enabled or float ABI mismatch between libraries |
| H7 fault on DMA buffer access / data looks stale | DMA buffer in DTCM or missing cache maintenance |

## 5. ST-Link can't connect

Work through, in order:
1. Power and wiring: target powered, common GND, SWDIO/SWCLK/NRST connected, VTref connected (stand-alone ST-Link).
2. **Connect under reset**: `STM32_Programmer_CLI -c port=SWD mode=UR` (or CubeIDE debug config → Reset behaviour "Connect under reset"). Needed when firmware reconfigures PA13/PA14 (SWD) as GPIO, enters low power immediately, or disables the debug port.
3. **BOOT0 high** (jumper/button) then reset: the chip boots the system bootloader instead of the user code, so SWD becomes reachable; flash/erase, then put BOOT0 back. On families with nBOOT0/nSWBOOT0 option bits, BOOT0 may be ignored — check option bytes.
4. Lower the SWD frequency (`freq=950` or lower) for long wires.
5. Firmware update the ST-Link (CubeProgrammer → Firmware upgrade).
6. Read protection active → see below.
7. Prevent it next time: in the `.ioc`, SYS → Debug = "Serial Wire" so CubeMX reserves PA13/PA14.

## 6. Option bytes and read protection (RDP)

| Level | Value | Effect | Reversible? |
|---|---|---|---|
| 0 | 0xAA (**0xA5 on STM32F1**) | No protection | — |
| 1 | any other (often 0xBB) | Flash not readable via debug/bootloader; debug still connects | Yes — setting back to the level-0 value **mass-erases** the flash |
| 2 | 0xCC | Debug port and bootloader disabled permanently, option bytes frozen | **No. Permanent** (except newer families like U5 with a password-based regression — check the RM) |

Family differences: F1 uses 0xA5 for level 0 (writing 0xAA there leaves level 1 on); H5 uses "product state" instead of RDP levels. Always read the current option bytes with `-ob displ` before changing them.

Never set RDP level 2 (or write option bytes in general) without the user's explicit, informed confirmation. For production protection, level 1 plus secure boot features of the family (e.g. TrustZone/HDP on U5/H5/L5) is the usual choice; level 2 only when the product truly needs it and the process is validated.

Other option bytes that cause "the board died" reports: watchdog hardware-start (`IWDG_SW` = 0 → watchdog always on), `nBOOT0`/`nSWBOOT0` (boot source), BOR level, write protection (WRP) on sectors you then try to program. Display them with `-ob displ` before changing anything.
