# Startup code, linker scripts and bootloaders

Cortex-M (STM32) first; notes for ESP32 and PIC32 at the end of each part. Fault analysis and the naked HardFault handler live in `stm32-firmware` → `references/faults-and-recovery.md`; this file covers how the image is laid out and how it gets from reset to `main()`.

## Contents
1. Reset sequence and vector table
2. Startup code (reset → `main`)
3. Linker scripts
4. Placing things: no-init RAM, RAM functions, CCM/DTCM, reserved flash
5. Checking the result
6. Bootloaders: layout, image header, update strategy
7. Jumping from bootloader to application
8. Common failures
9. ESP32 and PIC32 notes

---

## 1. Reset sequence and vector table

On reset a Cortex-M core does two reads and nothing else:
1. word 0 of the vector table → **MSP** (initial stack pointer),
2. word 1 → **PC** (`Reset_Handler`, Thumb bit set, so the value is odd).

On most STM32 (F0/F1/F3/F4/G0/G4/L4) the table is read from address 0, which is aliased to flash (`0x0800_0000`), system memory or SRAM depending on BOOT pins / option bytes. On H7 and Armv8-M parts (L5/U5/H5) the reset value of VTOR comes from boot-address option bytes (`BOOT_ADD0/1`, `NSBOOTADD0/1`...), so there is no alias at 0 — the core reads the table directly at that boot address. After that, the core finds the table through **VTOR** (`SCB->VTOR`, M0+/M3/M4/M7/M33; plain Cortex-M0 such as STM32F0 has no VTOR — remap SRAM to 0 via SYSCFG and copy the table there).

Layout: `[0] _estack, [1] Reset_Handler, [2] NMI, [3] HardFault, [4..15] other core exceptions (some reserved), [16..] device IRQs` in NVIC order. The device-specific list comes from the vendor startup file — do not write it by hand.

**VTOR alignment**: the table base must be aligned to its size rounded up to a power of two, minimum 128 bytes. An STM32 with 100 vectors (400 bytes) needs 512-byte alignment; large parts (H7, 150+ vectors) need 1 KB. An application at `0x0800_8000` is always fine; one at `0x0800_8100` may not be.

Unused vectors point to a weak `Default_Handler` (infinite loop). A typo in a handler name (`USART1_IRQHandle`) silently creates a new function and the default handler runs instead — the debugger shows the CPU spinning in `Default_Handler`/`WWDG_IRQHandler` (they often share an address).

## 2. Startup code (reset → `main`)

The vendor `startup_<part>.s` (CubeMX) or `crt0` does, in order:

| Step | Why it matters |
|---|---|
| Set SP (again) | Some bootloaders/debuggers enter without the vector-table SP |
| `SystemInit()` | FPU enable, VTOR (when `USER_VECT_TAB_ADDRESS` is defined), sometimes external memory. Recent ST startup files call it **before** `.data`/`.bss` are initialised, so it must not use initialised globals |
| Copy `.data` from flash (LMA, `_sidata`) to RAM (VMA, `_sdata.._edata`) | Initialised globals |
| Zero `.bss` (`_sbss.._ebss`) | Zero-initialised globals; C guarantees it |
| `__libc_init_array()` | Runs `.preinit_array`, `_init`, `.init_array` → C++ static constructors and `__attribute__((constructor))` functions |
| `main()` | Must never return; startup loops if it does |

**FPU (M4F/M7/M33)**: `SystemInit` sets `SCB->CPACR |= (3UL << 20) | (3UL << 22)` when `__FPU_PRESENT && __FPU_USED`. If code compiled with `-mfloat-abi=hard`/`softfp` runs before this (a C++ constructor, or a custom startup that skips `SystemInit`), the first FP instruction faults with UsageFault NOCP. All objects and libraries must use the same float ABI.

**C++ constructors** run before `main`, i.e. before clocks and peripherals are configured. Keep them trivial; construct hardware objects explicitly in `main` (see `coding-standards.md` §4).

Linker symbols are addresses, not variables — declare them as arrays and never read their "value":
```c
extern uint32_t _sdata[], _edata[], _sidata[];   /* &symbol is the address */
size_t data_bytes = (size_t)((uintptr_t)_edata - (uintptr_t)_sdata);
```

## 3. Linker scripts

Minimal shape of a GNU ld script for an STM32 (CubeMX scripts follow it):

```ld
ENTRY(Reset_Handler)

_Min_Heap_Size  = 0x200;
_Min_Stack_Size = 0x400;

MEMORY
{
  FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 120K   /* last 8 KB reserved, see §4 */
  CONFIG (r)   : ORIGIN = 0x0801E000, LENGTH = 8K
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 32K
}

_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
  .isr_vector : { . = ALIGN(4); KEEP(*(.isr_vector)) . = ALIGN(4); } >FLASH

  .text :
  {
    . = ALIGN(4);
    *(.text) *(.text*)
    *(.glue_7) *(.glue_7t) *(.eh_frame)
    KEEP(*(.init)) KEEP(*(.fini))
    . = ALIGN(4);
  } >FLASH

  .rodata : { . = ALIGN(4); *(.rodata) *(.rodata*) . = ALIGN(4); } >FLASH

  .ARM.exidx : { *(.ARM.exidx* .gnu.linkonce.armexidx.*) } >FLASH   /* C++ unwind tables */

  .preinit_array : { PROVIDE_HIDDEN(__preinit_array_start = .); KEEP(*(.preinit_array*)) PROVIDE_HIDDEN(__preinit_array_end = .); } >FLASH
  .init_array    : { PROVIDE_HIDDEN(__init_array_start = .); KEEP(*(SORT(.init_array.*))) KEEP(*(.init_array*)) PROVIDE_HIDDEN(__init_array_end = .); } >FLASH

  _sidata = LOADADDR(.data);
  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data) *(.data*)
    *(.RamFunc) *(.RamFunc*)          /* functions copied to RAM with .data */
    . = ALIGN(4);
    _edata = .;
  } >RAM AT> FLASH                    /* runs in RAM (VMA), stored in flash (LMA) */

  .noinit (NOLOAD) : { . = ALIGN(4); *(.noinit) *(.noinit*) . = ALIGN(4); } >RAM

  .bss :
  {
    . = ALIGN(4);
    _sbss = .;  __bss_start__ = _sbss;
    *(.bss) *(.bss*) *(COMMON)
    . = ALIGN(4);
    _ebss = .;  __bss_end__ = _ebss;
  } >RAM

  ._user_heap_stack :                 /* only a size check, see below */
  {
    . = ALIGN(8);
    PROVIDE(end = .); PROVIDE(_end = .);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >RAM
}
```
Start from the vendor script for the exact part (CubeMX generates one per device) and change it; do not write one from scratch.

Key points:
- **`KEEP`** protects sections that nothing references from `--gc-sections` (vector table, init arrays, image header, anything found by address only).
- **`>RAM AT> FLASH`** = VMA in RAM, LMA in flash; startup copies it. `LOADADDR()` gives the LMA.
- **`ALIGN`**: keep section boundaries 4-byte aligned so the word-wise copy/zero loops in startup are correct; 8-byte for stack (AAPCS) and for `double`/`uint64_t` data.
- **Heap and stack**: in CubeMX projects the stack grows down from `_estack`; `._user_heap_stack` reserves nothing at run time — it just makes the link fail (`region RAM overflowed`) if static data + heap + stack minimums do not fit. `_sbrk` (`sysmem.c`) hands out heap from `_end` up to `_estack - _Min_Stack_Size`. There is no run-time stack limit on Cortex-M (except Armv8-M Mainline — M33/M55/M85 — with `MSPLIM`/`PSPLIM`); a stack overflow writes silently into `.bss`/heap. Measure (stack painting, `debugging.md` §6) and size `_Min_Stack_Size` from the measurement.
- **Order matters for `.noinit`**: it must sit outside `_sbss.._ebss` and outside anything startup or a bootloader writes, otherwise "survives reset" quietly stops being true.

## 4. Placing things: no-init RAM, RAM functions, CCM/DTCM, reserved flash

**No-init RAM** (fault records, breadcrumbs, boot counters, bootloader↔app mailbox):
```c
__attribute__((section(".noinit"))) volatile uint32_t g_boot_magic;
```
Needs the `NOLOAD` section above. Content is random after power-on; validate with a magic word + CRC. If a bootloader and an application both run, the address must be the same in both linker scripts (fixed-address region in `MEMORY`) and the bootloader must not zero or use it. The fault-record pattern itself is in `stm32-firmware` `faults-and-recovery.md` §3.

**Functions in RAM** (flash programming code that must run while flash is busy, or hot loops on parts with many wait states):
```c
__attribute__((section(".RamFunc"), noinline)) void flash_write_word(uint32_t addr, uint32_t v);
```
ST HAL has `__RAM_FUNC` for this. Everything the function calls must also be in RAM (watch for compiler-inserted library calls like `memcpy` or `__aeabi_uidiv`), and it must not take an interrupt whose handler lives in flash while flash is busy — or the vector table must be in RAM too.

**CCM / DTCM / ITCM** (core-coupled memories, zero wait state, but restrictions):

| Memory | Parts | Notes |
|---|---|---|
| CCM SRAM `0x1000_0000` | F4 (F405/407/429...) | Data only (no code execution); DMA cannot access it — never put DMA buffers there |
| CCM SRAM `0x1000_0000` | F3, G4 | Code and data. F3: no DMA access. G4: also aliased right after SRAM2 (`0x2001_8000` on G47x/G48x, `0x2000_5800` on G431/G441); DMA can reach it **only through that alias**, never at `0x1000_0000` (AN4296 Table 2) — link DMA buffers at the alias address |
| DTCM `0x2000_0000` | F7, H7 | Fast data/stack; on H7 not reachable by DMA1/DMA2 (MDMA only). Good for stack and ISR data |
| ITCM `0x0000_0000` | F7, H7 | Fast code; copy from flash at startup like `.data` |

Pattern: add a `MEMORY` region, a section `.ccmram : { ... } >CCMRAM AT> FLASH` with start/end/load symbols, and add a copy (or zero) loop for it in startup. The extra section is **not** initialised by the stock startup unless you add that loop.

**Reserving flash** (config block, EEPROM emulation, bootloader, image slots): shrink the `FLASH` region and add a named region, as in §3. Rules:
- Align reserved areas to the **erase unit**: 2 KB pages on G0, G431 and most L4 (4 KB on G47x/G48x in single-bank mode, `DBANK = 0` — check the RM for your part), but uneven sectors on F4/F7 (F4: 16/16/16/16/64/128 KB...; F74x/F75x: 32/32/32/32/128/256 KB...). On F4 an EEPROM-emulation area in a 128 KB sector wastes a lot; use two of the small 16 KB sectors near the start — sectors 1 and 2, because sector 0 at `0x0800_0000` must hold the boot vector table (keep the vectors or a small bootloader in sector 0 and start the rest of the code after the emulation sectors).
- Some families erase in banks or have **dual-bank** flash: writing one bank while executing from the other avoids stalls.
- Keep a `.config` section `NOLOAD` (or exclude it from the image) if the programmer must not overwrite field data on reflash; otherwise every `load` wipes the stored settings.
- ST's EEPROM-emulation app notes (per family) and `cubemx-hal.md` in `stm32-firmware` cover the write algorithm.

## 5. Checking the result

The linker only complains about overflow; wrong placement links happily. After any change to the linker script, startup or memory map:

```
arm-none-eabi-gcc ... -Wl,-Map=build/app.map,--cref -Wl,--print-memory-usage
arm-none-eabi-objdump -h build/app.elf          # VMA and LMA per section
arm-none-eabi-readelf -l build/app.elf          # program headers: what the programmer will write, and where
arm-none-eabi-nm -n build/app.elf | grep -E '_estack|_sidata|_sdata|_ebss|g_pfnVectors'
```
Check: `.isr_vector` at the image base (and VTOR-aligned); `.data` VMA in RAM with LMA in flash; nothing in the reserved/config regions; `.noinit` outside `.bss`; RAM functions have a RAM VMA. More tools in `debug-tools.md` §5.

## 6. Bootloaders: layout, image header, update strategy

**First decide whether you need one.** Options, cheapest first:

| Option | When |
|---|---|
| STM32 **system bootloader** (ROM, entered via BOOT0 / option bytes, or by jumping to system memory) | Factory programming and recovery over USART/USB DFU/I2C/SPI/CAN (interfaces vary per part, see ST AN2606). No custom code, but no authentication and needs a pin or a software trigger |
| Existing secure bootloader: **MCUboot** (Zephyr, TF-M, others), ST **SBSFU** / **STM32 Secure Manager** / OEMiROT on newer secure parts | Signed images, A/B slots, rollback — prefer this over writing your own when security matters |
| Custom bootloader | Special transport (RS-485, CAN, proprietary radio) or a tiny part |

**Typical layout** (single application slot + download slot):
```
0x0800_0000  bootloader (16–64 KB, write-protected with WRP option bytes in production)
0x0800_8000  slot A: [image header][vector table][app...]   ← runs here
0x0804_0000  slot B: download area for the new image
             config / EEPROM emulation
```
The application is linked at its slot address (`FLASH ORIGIN = 0x08008000`, or after a header) and sets `SCB->VTOR` to its own vector table — in CubeMX projects by defining `USER_VECT_TAB_ADDRESS` and `VECT_TAB_OFFSET` in `system_<family>.c`, or explicitly at the start of `main`.

**Image header**, written by a post-build script (the compiler cannot know the CRC of its own output):
```c
typedef struct {
    uint32_t magic;          /* identifies a header, e.g. 0x494D4731 */
    uint16_t hdr_version;
    uint16_t hdr_size;       /* vector table starts at slot + hdr_size, keep it VTOR-aligned */
    uint32_t img_size;
    uint32_t fw_version;     /* semantic version or monotonically increasing build number */
    uint32_t security_cnt;   /* anti-rollback counter */
    uint32_t crc32;          /* over the image after the header */
    uint8_t  signature[64];  /* e.g. ECDSA P-256 or Ed25519 over header fields + image hash */
} image_header_t;
```
CRC detects corruption; only a signature (public key inside the write-protected bootloader) detects tampering. Pad the header to the VTOR alignment, or put it right after the vector table at a fixed offset.

**Update strategies**:
- **Overwrite / copy**: download into slot B, verify, copy B → A. Simple; a power cut during the copy must be recoverable, so the bootloader needs a progress record and must be able to resume.
- **A/B (dual slot, execute in place)**: each slot is its own link address (two builds) or uses position-independent code; the bootloader boots the newest valid slot. No copy, but two binaries per release.
- **Swap** (MCUboot style): swap A and B sector by sector using a scratch area or a move; the old image stays available for revert.
- **Hardware bank swap**: some STM32 (dual-bank parts with `BFB2`/`SWAP_BANK` option bits) boot from either bank at the same address, giving A/B with a single build.

**Rollback**: the new image boots as "pending/test". It must confirm itself (write an "image OK" flag) only after it has passed a self-test — peripherals up, communication with the server or main board working. If it resets before confirming (watchdog, crash, power cut), the bootloader reverts to the previous image. A **security counter** stops an attacker installing an older, vulnerable signed image.

Bootloader rules: never erase the only valid image before the new one is verified; validate everything (sizes, addresses, header fields) as untrusted input; keep the bootloader small and rarely changed — it is the one piece you cannot fix in the field.

## 7. Jumping from bootloader to application

The application must start in a state close to reset. Sequence:

```c
#define APP_BASE 0x08008000UL   /* address of the application's vector table */

static bool app_vectors_look_valid(uint32_t base) {
    const uint32_t sp = *(const uint32_t *)base;
    const uint32_t pc = *(const uint32_t *)(base + 4u);
    /* SRAM_END = one past the last RAM byte: _estack is usually exactly that */
    return (sp > SRAM_START) && (sp <= SRAM_END) && ((sp & 7u) == 0u)
        && ((pc & 1u) != 0u) && (pc > base) && (pc < FLASH_END);
}   /* a cheap sanity check, NOT a substitute for CRC/signature verification */

__attribute__((noreturn)) void jump_to_app(void) {
    /* 1. Stop and de-initialise what the bootloader started */
    HAL_RCC_DeInit();            /* back to HSI, PLL off (or leave clocks as the app expects — be consistent) */
    HAL_DeInit();                /* peripheral resets; also de-init USB/DMA/timers you used */

    /* 2. Silence interrupts */
    __disable_irq();
    SysTick->CTRL = 0; SysTick->LOAD = 0; SysTick->VAL = 0;
    for (uint32_t i = 0; i < (sizeof NVIC->ICER / sizeof NVIC->ICER[0]); i++) {
        NVIC->ICER[i] = 0xFFFFFFFFUL;   /* disable */
        NVIC->ICPR[i] = 0xFFFFFFFFUL;   /* clear pending */
    }
    SCB->ICSR = SCB_ICSR_PENDSTCLR_Msk | SCB_ICSR_PENDSVCLR_Msk;  /* clear pending SysTick and PendSV */

    /* 3. Point the core at the application */
    SCB->VTOR = APP_BASE;
    __DSB(); __ISB();

    /* 4. CubeMX applications never call __enable_irq() (PRIMASK is 0 out of a real
          reset); if PRIMASK stays set, SysTick never fires and HAL_Delay() hangs.
          Re-enabling here is safe only because every NVIC IRQ, SysTick and PendSV
          is disabled and nothing is pending (VTOR already points at the app).
          Alternative: leave it set and have the app call __enable_irq() early. */
    __enable_irq();

    /* 5. Load MSP and branch. Both values are read into registers first,
          so nothing touches the stack after MSP changes. */
    const volatile uint32_t *vt = (const volatile uint32_t *)APP_BASE;
    const uint32_t app_sp = vt[0], app_pc = vt[1];
    __asm volatile("msr msp, %0 \n bx %1" :: "r"(app_sp), "r"(app_pc) : "memory");
    for (;;) {}
}
```
Extra points:
- If the bootloader ran an RTOS or used the PSP, also set `CONTROL = 0` (`__set_CONTROL(0); __ISB();`) so the app starts privileged on MSP.
- Disable the MPU and caches the bootloader enabled (H7/F7: `SCB_DisableDCache()`, clean first) unless the app expects them on.
- A **cleaner alternative** to de-initialising everything: write a magic value to no-init RAM or a backup register, `NVIC_SystemReset()`, and have the bootloader jump very early after reset, before it touches any peripheral.
- **Watchdog**: the IWDG cannot be stopped once started. If the bootloader started it, the application must refresh it early (before long init such as clock or radio start-up), or the bootloader must choose a timeout long enough to cover the app's start-up.
- Jumping to the **STM32 system bootloader** from software follows the same steps, with the system-memory address for the part from AN2606; some families also need the memory remap set to system flash first.

## 8. Common failures

| Symptom | Likely cause |
|---|---|
| HardFault immediately after the jump | Application linked at the wrong address (still `0x0800_0000`); stack pointer invalid; branch address without the Thumb bit; app image corrupt — check `readelf -l` of the app ELF and dump the vector table at `APP_BASE` |
| Interrupts run the bootloader's handlers (or `Default_Handler`) | VTOR not set, set too late (after an IRQ was enabled), or overwritten by `SystemInit` with the default offset |
| Some interrupts work, others jump to garbage | VTOR base not aligned to the table size — the low bits are ignored, so the core reads a shifted table |
| App works when flashed alone, not via the bootloader | Different clock state than the app assumes; peripherals/DMA still running; PRIMASK left set; app expects `.noinit`/backup data the bootloader cleared |
| `HAL_Delay()` hangs right after the jump | SysTick or PRIMASK left disabled/set; or SysTick interrupt pending from the bootloader and the app's handler not ready |
| Reset loop a few hundred ms after the update | IWDG started in the bootloader, app does not refresh it early enough |
| UART garbage / USB not enumerating in the app | Peripheral left configured by the bootloader; USB needs a disconnect (pull D+ low or `USB->BCDR`/device disconnect) before the app re-enumerates |
| Board bricked after power cut during update | Only one image and it was erased first; no resumable copy; no rollback |
| App works under the debugger, not from power-on | Debugger `load` set PC/SP itself; at real reset the bootloader's validity check fails or jumps differently |

## 9. ESP32 and PIC32 notes

**ESP32 (ESP-IDF / Arduino)**: you do not write the reset path or linker script by hand. ROM bootloader → second-stage bootloader in flash → application chosen from the partition table (`factory`, `ota_0`, `ota_1`, selected by `otadata`). Memory placement uses attributes and linker fragments instead (`IRAM_ATTR`, `DRAM_ATTR`, `RTC_NOINIT_ATTR` / `__NOINIT_ATTR` for data that survives a reset, `.lf` linker fragment files in ESP-IDF components). OTA, rollback (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` + `esp_ota_mark_app_valid_cancel_rollback()`), anti-rollback, secure boot and flash encryption → `esp32-firmware` (`references/connectivity-ota-security.md`, partitions in `platformio-setup.md`). ESP32 also has MCUboot support through some RTOS ports; use the ESP-IDF bootloader unless there is a reason not to.

**PIC32 (XC32)**: MIPS-based PIC32MX/MZ use XC32's own startup (`crt0`, runs `.data` init, `.bss` clear and `_on_reset`/`_on_bootstrap` hooks) and default linker scripts in the device pack; addresses are virtual (KSEG0 cached `0x9D00_0000` program flash, KSEG1 uncached; reset at `0xBFC0_0000` in boot flash). Bootloaders usually live in boot flash or the start of program flash and the application gets a custom linker script that moves its reset address and exception base (`EBASE` on both; vector spacing via `INTCON.VS` on MX, per-vector `OFFx` offset registers on MZ). Use the Microchip/MCC bootloader examples and their matching application linker script rather than adapting a Cortex-M recipe. Arm-based PIC32C/SAM parts follow the Cortex-M material above. 8/16-bit PIC bootloaders (code offset, interrupt vector remapping) → `pic-firmware`.
