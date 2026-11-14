# Testing firmware

## Contents
1. Strategy: what to test where
2. Make code testable
3. PlatformIO + Unity (ESP32, STM32 on PlatformIO)
4. CMake + Unity on the PC (CubeMX, MPLAB X projects)
5. Fakes and mocks
6. On-target / hardware-in-the-loop tests
7. Static analysis
8. Continuous integration

---

## 1. Strategy: what to test where

| Level | Runs on | Tests | Speed |
|---|---|---|---|
| **Unit (host)** | PC | Pure logic: parsers, protocol framing, state machines, filters, conversions, ring buffers, drivers against a fake bus | Milliseconds — run on every build |
| **Integration (target)** | The MCU | Driver + real peripheral, timing, interrupts | Seconds, needs a board |
| **System / HIL** | Board + test rig | Whole product behaviour, scripted over UART/USB with a PC harness (Python + pyserial) | Minutes |

Most bugs worth catching early live in logic, so the biggest win is host unit tests. Start there even if the project has none: one test file for one module is already useful.

Unity (ThrowTheSwitch) is the default framework: plain C, tiny, used by PlatformIO and ESP-IDF. Ceedling adds CMock auto-mocking on top. GoogleTest/Catch2 are options for C++ code tested on the host.

## 2. Make code testable

- Separate logic from hardware: a function that parses a frame takes a `const uint8_t*`, it does not read the UART register.
- Inject dependencies: drivers receive a bus interface (`i2c_bus_t`, see `architecture.md`), time is passed in (`now_ms`), not read from `HAL_GetTick()` inside.
- Hide vendor headers behind your own: application code includes `bsp_gpio.h`, not `stm32g4xx_hal.h`, so it compiles on a PC.
- Keep generated code (CubeMX/MCC) out of unit tests entirely.

## 3. PlatformIO + Unity

Layout:
```
project/
├── platformio.ini
├── lib/
│   └── protocol/            ← portable logic lives in lib/ so the native env can build it
│       ├── protocol.c
│       └── protocol.h
├── src/main.c               ← hardware entry point, not built for native tests
└── test/
    └── test_protocol/
        └── test_main.c
```

`platformio.ini`:
```ini
[env:native]
platform = native
test_framework = unity
build_flags = -std=c11 -Wall -Wextra

[env:esp32dev]            ; or your STM32 env
platform = espressif32
board = esp32dev
framework = espidf
test_framework = unity
```

`test/test_protocol/test_main.c`:
```c
#include <unity.h>
#include "protocol.h"

void setUp(void) {}
void tearDown(void) {}

static void test_crc_known_vector(void) {
    const uint8_t data[] = {'1','2','3','4','5','6','7','8','9'};
    TEST_ASSERT_EQUAL_HEX16(0x29B1, crc16_ccitt_false(data, sizeof data));
}

static void test_frame_rejects_bad_length(void) {
    const uint8_t frame[] = {0xAA, 0xFF, 0x00};
    TEST_ASSERT_EQUAL(PROTO_ERR_LENGTH, proto_parse(frame, sizeof frame, NULL));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_crc_known_vector);
    RUN_TEST(test_frame_rejects_bad_length);
    return UNITY_END();
}
```

Commands:
```
pio test -e native                 # host tests
pio test -e native -f test_protocol
pio test -e esp32dev               # builds, uploads and runs the same tests on the board
```
For on-target tests with the Arduino framework, the test file uses `setup()` (run tests) and an empty `loop()` instead of `main`. On ESP-IDF use `void app_main(void)`.

The `native` platform needs a host C compiler: on macOS install the Xcode Command Line Tools (`xcode-select --install`).

Practical notes for `pio test -e native`:
- Sanitizers: UndefinedBehaviorSanitizer works (`build_flags = -fsanitize=undefined -fno-sanitize-recover=undefined`, plus the same flag at link time, e.g. via an `extra_scripts` pre-script that appends to `LINKFLAGS`). AddressSanitizer can crash at start-up under the PlatformIO native runner on macOS — use ASan in the CMake harness instead (§4).
- When tests fail, PlatformIO may report the run as `ERRORED` / `SIGINT`; run `.pio/build/native/program` directly to see the plain Unity summary.
- C headers included from C++ (Arduino `.cpp` files) need `#ifdef __cplusplus extern "C" { ... } #endif` guards, or the link fails with mangled-name errors.

## 4. CMake + Unity on the PC (CubeMX, MPLAB X)

CubeIDE and MPLAB X builds are for the target. Add a separate host test build next to them:

```
project/
├── Core/ or project.X/          ← generated + target code
├── app/                         ← portable modules (also compiled into the firmware)
└── tests/
    ├── CMakeLists.txt
    ├── vendor/unity/            ← unity.c, unity.h, unity_internals.h (copied once from the Unity repo)
    └── test_protocol.c
```

`tests/CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.16)
project(firmware_tests C)
set(CMAKE_C_STANDARD 11)
add_compile_options(-Wall -Wextra -g -O0 -fsanitize=address,undefined)
add_link_options(-fsanitize=address,undefined)

add_library(unity vendor/unity/unity.c)
target_include_directories(unity PUBLIC vendor/unity)

add_executable(test_protocol test_protocol.c ../app/protocol.c)
target_include_directories(test_protocol PRIVATE ../app)
target_link_libraries(test_protocol unity)

enable_testing()
add_test(NAME protocol COMMAND test_protocol)
```
Run: `cmake -S tests -B build-tests && cmake --build build-tests && ctest --test-dir build-tests --output-on-failure`

The sanitizers (`-fsanitize=address,undefined`) catch buffer overruns and undefined behaviour on the PC that would be silent corruption on the MCU — one of the best reasons to run logic on the host.

Remember size differences: on PIC16/18/24 `int` is 16-bit. When testing code meant for those targets, avoid `int` in the code under test, or the host test can pass while the target overflows.

## 5. Fakes and mocks

- **Hand-written fake** of a small interface (e.g. `i2c_bus_t` that returns scripted bytes and records writes) — usually the clearest option.
- **FFF (Fake Function Framework)**: single header `fff.h`; `FAKE_VALUE_FUNC(int, i2c_write, uint8_t, const uint8_t*, size_t);` then inspect `i2c_write_fake.call_count`, `arg0_val`, set `return_val`.
- **CMock** (via Ceedling): generates mocks from headers automatically; heavier but scales to big codebases.

Test behaviour (what the driver sends/returns), not implementation details.

## 6. On-target / hardware-in-the-loop tests

- PlatformIO `pio test -e <board_env>` runs Unity on the real MCU and reads results over serial.
- For scripted system tests: firmware exposes a simple command shell over UART/USB; a Python script (pyserial + pytest) sends commands and checks responses and timings.
- Use a logic analyser for protocol timing; save captures with the bug report.

## 7. Static analysis

| Tool | Command |
|---|---|
| PlatformIO built-in | `pio check` (cppcheck by default), `pio check --tool clangtidy`, `pio check --severity=medium` |
| cppcheck | `cppcheck --enable=warning,style,performance,portability --std=c11 --inline-suppr -I app -I Core/Inc app Core/Src` |
| cppcheck MISRA addon | `cppcheck --addon=misra app/` (rule headlines need the MISRA rule-text file you own a licence for) |
| clang-tidy | needs `compile_commands.json`: `pio run -t compiledb`, or CMake `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` |

Run analysers on **your** code; exclude vendor/generated directories or the signal drowns in noise.

## 8. Continuous integration

Minimal GitHub Actions job for a PlatformIO project:
```yaml
name: firmware
on: [push, pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install platformio
      - run: pio test -e native
      - run: pio run
      - run: pio check --fail-on-defect=high
```
For CubeMX/MPLAB projects, run the CMake host tests in CI; building the target needs the vendor toolchain installed in the CI image.
