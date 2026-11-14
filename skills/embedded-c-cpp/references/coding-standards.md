# Coding standards for embedded C and C++

## Contents
1. Which standard to follow
2. Rules that matter most in practice
3. Compiler warnings per toolchain
4. C++ on microcontrollers
5. Naming and file layout

---

## 1. Which standard to follow

| Standard | What it is | Use it as |
|---|---|---|
| **MISRA C:2012** (consolidated as MISRA C:2023) | Rules for safety-critical C; de-facto industry reference | Default guideline for C firmware. Full compliance only if a customer/certification requires it |
| **CERT C** (SEI) | Security-focused rules (overflow, strings, undefined behaviour) | Complement for anything connected (Wi-Fi, BLE, OTA) |
| **Barr Group Embedded C Coding Standard** | Practical style + bug-prevention rules | Good house style when no other standard is imposed |
| **MISRA C++:2023** / AUTOSAR C++14 | C++ subsets for critical systems | Guideline when writing C++ firmware |

Do not claim "MISRA compliant" without a checker report and a deviation record. Say "follows MISRA-style rules" instead.

## 2. Rules that matter most in practice

These are the ones that prevent real field bugs (MISRA/CERT rule numbers in brackets where useful):

- **Fixed-width integer types**, no plain `int`/`char` for data whose size matters [MISRA Dir 4.6].
- **No implicit narrowing or sign conversion.** `uint8_t x = a + b;` where `a + b > 255` silently wraps. Integer promotion turns `uint8_t` into `int` before arithmetic — watch comparisons and shifts.
- **Shifts**: never shift by ≥ width or shift a negative value; use unsigned for bit operations (`1U << n`, `1UL << 20` on 16-bit targets).
- **No undefined behaviour**: signed overflow, out-of-bounds, uninitialised reads, aliasing violations, modifying a variable twice between sequence points.
- **Every `switch` has a `default`**; every `if … else if` chain ends with `else` (even if it only logs an error) [MISRA 16.4, 15.7].
- **Single point of definition** for constants: `#define`/`enum`/`static const` with units in the name (`TIMEOUT_MS`, `VREF_MV`).
- **Function-like macros only when a function cannot do it**; prefer `static inline`. Fully parenthesise macro arguments.
- **No recursion** [MISRA 17.2] — stack is small and bounded analysis needs it gone. PIC16/18 hardware stacks make this a hard limit.
- **No dynamic memory** after init [MISRA Dir 4.12].
- **Check every return value** of functions that can fail [MISRA Dir 4.7].
- **Bounded loops / waits**: every `while (!flag)` gets a timeout or watchdog-safe design.
- **`const` everything that does not change** — on many MCUs it also moves data from RAM to flash.
- **`static` for file-local functions and variables** — limits coupling and helps the optimiser.
- **Avoid bit-fields for hardware registers in portable code** — layout is implementation-defined. Vendor headers that use them are fine on that vendor's compiler.

## 3. Compiler warnings per toolchain

**GCC / Clang (STM32 arm-none-eabi, ESP32 xtensa/riscv, PIC32 XC32 is GCC-based)**

```
-Wall -Wextra -Wshadow -Wdouble-promotion -Wformat=2 -Wformat-truncation
-Wundef -Wcast-align -Wconversion -Wsign-conversion -Wnull-dereference
-Wstrict-prototypes (C only) -Wmissing-prototypes (C only)
-fno-common
```
- `-Wconversion` is noisy on vendor HAL code: enable it for your own files only (per-file flags, or a separate library target).
- `-Wdouble-promotion` catches accidental `double` math on single-precision FPUs (Cortex-M4F/M7, ESP32-S3), which is much slower.
- Treat warnings as errors in CI (`-Werror`) once the codebase is clean.

**XC8 (PIC10/12/16/18)**: `-Wall` plus `-Wpedantic`; read the "advisory" messages too — XC8 reports pointer and stack issues as warnings. XC8 is not GCC; many GCC flags do not exist.

**XC16 (PIC24/dsPIC)**: GCC-derived; `-Wall -Wextra` work.

## 4. C++ on microcontrollers

C++ is a good fit for 32-bit MCUs (STM32, ESP32, PIC32) when used as a *zero-overhead* language. Arduino is already C++.

**Use**
- `constexpr`, `enum class`, `static_assert`, `std::array`, templates for type-safe drivers, references instead of nullable pointers.
- RAII for critical sections, mutex locks and peripheral claims:
  ```cpp
  class CriticalSection {
  public:
      CriticalSection()  { primask_ = __get_PRIMASK(); __disable_irq(); }
      ~CriticalSection() { __set_PRIMASK(primask_); }
  private:
      uint32_t primask_;
  };
  ```
- Static polymorphism (templates / CRTP) when virtual-call overhead or vtables in RAM matter; plain virtual interfaces are fine elsewhere and make mocking easy.
- The Embedded Template Library (ETL) for fixed-capacity containers instead of `std::vector`/`std::string`.

**Avoid**
- Exceptions and RTTI on small targets: build with `-fno-exceptions -fno-rtti` (ESP-IDF has exceptions off by default; STM32 projects should set it).
- Heap-using STL (`std::vector`, `std::string`, `std::function` with captures, `std::map`), `<iostream>` (huge).
- Global objects with non-trivial constructors whose order matters (static initialisation order fiasco); construct hardware objects explicitly in `main`/`app_main`.

**Mixing with C generators (CubeMX, MCC)**: keep generated `main.c` as C. Put the application in `app.cpp` exposing `extern "C" void app_main_loop(void);` and call it from a USER CODE block.

PIC8/16-bit (XC8, XC16): use C. XC8 has no C++ support.

## 5. Naming and file layout

- One module = `module.h` (public API, include guard, minimal includes) + `module.c` (private `static` helpers).
- Prefix public symbols with the module name: `uart_init()`, `uart_write()`; types `uart_config_t`.
- Units in names: `delay_ms`, `voltage_mv`, `temp_c_x10`.
- Hardware-specific code in `bsp/` or `drivers/`, portable logic in `app/` or `lib/`, tests in `test/`.
- Version the inputs to generators (`.ioc`, `.mc3`, `sdkconfig.defaults`, `platformio.ini`), not build output.
