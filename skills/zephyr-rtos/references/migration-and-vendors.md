# Migration from bare metal / FreeRTOS, and vendor ecosystems

## Contents
1. Should this product move to Zephyr?
2. Concept map — CubeMX/HAL/FreeRTOS → Zephyr
3. Migration pitfalls
4. Migration plan
5. Nordic — nRF Connect SDK
6. STM32 in Zephyr
7. ESP32 in Zephyr
8. Review checklist

---

## 1. Should this product move to Zephyr?

| Zephyr pays off | Stay with HAL/FreeRTOS or ESP-IDF |
|---|---|
| Networking (IPv4/6, TLS, MQTT, LwM2M), BLE, LTE modems, USB, file systems used together | Small single-purpose controller (one UART, some GPIO), already stable |
| Several MCU families sharing one code base (STM32 + nRF + ESP32) | Heavy use of ESP-IDF components (Wi-Fi provisioning, ESP-NOW, esp_http_server) or CubeMX middleware |
| Need for MCUboot DFU, TF-M, PSA crypto, CRA/RED cyber-security evidence | Hard real-time motor/power loops where every cycle is hand-tuned (still possible in Zephyr with zero-latency IRQs, but more effort) |
| Nordic BLE/LTE products (NCS is Zephyr) | Team cannot invest a few weeks in the learning curve |

## 2. Concept map — CubeMX/HAL/FreeRTOS → Zephyr

| Bare metal / CubeMX / FreeRTOS | Zephyr |
|---|---|
| `.ioc` pin and clock config, `MX_xxx_Init()` | Devicetree (board + overlay), pinctrl; drivers init themselves at boot |
| Peripheral enable in CubeMX | `status = "okay"` in DT + subsystem Kconfig (`CONFIG_I2C=y`) |
| `HAL_UART_Transmit`, `HAL_I2C_Mem_Read` | `uart_tx`/`uart_poll_out`, `i2c_write_read_dt` on a `*_dt_spec` |
| `HAL_GPIO_EXTI_Callback` | `gpio_init_callback` + `gpio_add_callback_dt` |
| `HAL_Delay(ms)` | `k_msleep(ms)` (yields) or `k_busy_wait(us)` (spins) |
| `HAL_GetTick()` | `k_uptime_get()` / `k_uptime_get_32()` |
| `xTaskCreate` / task | `K_THREAD_DEFINE` / `k_thread_create` |
| Task priority (higher number = higher priority) | **Lower number = higher priority**; negative = cooperative |
| `vTaskDelay`, `vTaskDelayUntil` | `k_sleep`, absolute timeouts / delayable work |
| `xQueueSend/Receive` (`...FromISR`) | `k_msgq_put/get` (same call in ISR with `K_NO_WAIT`) |
| `xSemaphoreGive/Take`, binary semaphore | `k_sem_give/take` |
| Mutex (`xSemaphoreCreateMutex`) | `k_mutex` (priority inheritance, recursive) |
| Event groups | `k_event` |
| Task notifications | `k_sem`, `k_event`, or `k_thread_resume` patterns |
| Software timers (`xTimerCreate`) | `k_timer` (ISR context) or `k_work_delayable` (thread context) |
| Deferred interrupt handling (`xTimerPendFunctionCallFromISR`) | `k_work_submit` |
| Stream/message buffers | `ring_buf`, `k_pipe` |
| `pvPortMalloc` / heap_4 | `k_malloc` (`CONFIG_HEAP_MEM_POOL_SIZE`), `k_heap`, `K_MEM_SLAB_DEFINE` |
| `configASSERT`, hook functions | `__ASSERT` (`CONFIG_ASSERT`), `k_sys_fatal_error_handler` |
| uxTaskGetStackHighWaterMark | Thread analyzer, `k_thread_stack_space_get()` |
| Emulated EEPROM / custom flash records | Settings over NVS/ZMS |
| Custom bootloader + OTA | MCUboot + MCUmgr (or hawkBit, LwM2M) |
| `printf` over UART | Logging subsystem (`LOG_INF`), `printk` |
| lwIP | Zephyr native IP stack with sockets API |
| CMSIS-RTOS v2 code | `CONFIG_CMSIS_RTOS_V2` compatibility layer (useful as a bridge, not as the end state) |

## 3. Migration pitfalls

- **Priority inversion of the numbering**: copying FreeRTOS priorities as-is makes the least important task the most important.
- **ISR-context callbacks**: Zephyr driver callbacks (GPIO, UART IRQ, CAN RX, timer expiry) run in ISR context exactly like HAL callbacks; the same "no blocking" rule applies, but the APIs look thread-safe and invite misuse.
- **Init order**: drivers initialise before `main()` by level and priority; your code that assumed `MX_xxx_Init()` order must instead check readiness. A device depending on another (sensor on I2C, modem on UART) needs correct init priorities — in-tree drivers handle this, out-of-tree drivers must set them.
- **Clock config moves to DT**: STM32 PLL, bus prescalers and kernel clocks are DT properties (`&pll`, `&rcc`, `clocks = <...>` per peripheral). A wrong `clock-frequency` silently breaks baud rates.
- **Pins configured twice**: a pin also driven by your own register code fights pinctrl; remove all direct register muxing.
- **Direct HAL calls**: allowed on STM32 (`CONFIG_USE_STM32_HAL_<PERIPH>=y` / `CONFIG_USE_STM32_LL_<PERIPH>=y` pull the module sources in), but a peripheral must then not also be owned by a Zephyr driver (disable its DT node). Use for gaps only (e.g., advanced timer features).
- **Interrupt priorities**: assign via DT `interrupts = <irq prio>`; Zephyr reserves levels on Cortex-M; never call kernel APIs from zero-latency IRQs.
- **Flash layout**: MCUboot slots and settings partitions change the app's link address; tools that flashed at 0x08000000 must use `west flash` or the signed hex.
- **Footprint**: a Zephyr hello world with logging, shell and asserts is bigger than a CubeMX blink; measure with `rom_report` and switch off development options in release builds.
- **Time base**: tick-based timeouts round up; for µs timing use hardware timers (counter API) or PWM, not `k_usleep`.

## 4. Migration plan

1. Build `samples/hello_world` and `samples/basic/blinky` for the exact board (or the nearest board + overlay).
2. Write the custom board definition (out-of-tree, section 6 of the build reference) from the schematic: clocks, console, LEDs, buttons, every bus.
3. Port hardware-independent modules unchanged (protocol parsers, state machines) and keep their host unit tests.
4. Replace the HAL layer module by module with Zephyr APIs; one peripheral per step, verified on hardware.
5. Map tasks to threads/workqueues; re-derive priorities; measure stacks with the thread analyzer.
6. Add MCUboot early (it moves the app address and partitions), then DFU, then production hardening (watchdog, fatal handler, logging levels, shell off).
7. Keep the old firmware branch buildable until the Zephyr version passes the full product test.

## 5. Nordic — nRF Connect SDK

- NCS is Nordic's distribution: a west manifest (`sdk-nrf`) importing a Nordic fork of Zephyr (`sdk-zephyr`, an upstream snapshot plus Nordic patches), MCUboot, TF-M, nrfxlib (SoftDevice Controller, nrf_modem for nRF91, crypto back-ends), Matter/Thread/Wi-Fi (nRF70) stacks. NCS version ≠ Zephyr version; each NCS release notes state the Zephyr base. Use NCS for nRF products that need qualified BLE (SoftDevice Controller), nRF91 LTE, nRF70 Wi-Fi or Nordic support.
- Install via nRF Connect for VS Code or Nordic's command-line toolchain/SDK manager (check the current NCS "Installing" page; tools changed across releases). Flash/debug with `nrfutil` (older: `nrfjprog`) and J-Link.
- Since NCS 2.7: HWMv2 is the default board model (HWMv1 removed in NCS 3.0) and all in-SDK samples and applications build with sysbuild by default; in 2.7 applications outside the SDK did not default to sysbuild, so check your NCS version's release notes and `west config build.sysbuild`. Upstream docs apply to NCS with Nordic-specific additions; check the NCS migration guides per version.
- **Partition Manager (PM)**: historically NCS generated flash/RAM partitions from `pm.yml` files and `pm_static.yml`, overriding DT partitions. NCS 3.3 deprecated it (builds warn), the next major release disables it by default and the following removes it; new designs should use DTS partitions except nRF91 Series (check the NCS "Migrating partition configuration from Partition Manager to devicetree" page). Disable with `SB_CONFIG_PARTITION_MANAGER=n`; helper `scripts/pm_to_dts.py` converts existing layouts. Test boot and DFU after migrating — a slot moved by a few KB breaks upgrades of fielded units.
- nRF5340: app core + net core; BLE controller on the net core is added by sysbuild; `/ns` targets use TF-M. nRF54L15: RRAM (no erase), use ZMS rather than NVS; `cpuflpr` RISC-V coprocessor target exists. nRF54H20: multi-core, different memory model; follow Nordic docs closely.
- Access-port protection: `west flash --recover` (nrfutil/nrfjprog) erases the whole chip including UICR — confirm before running.

## 6. STM32 in Zephyr

- Support is broad and maintained by ST and the community: `hal_stm32` module carries STM32Cube HAL/LL; Zephyr drivers mostly use LL. Many Nucleo/Discovery boards are in-tree.
- Clocks in DT: board files set `&clk_hse`, `&pll` (div-m/mul-n/div-p...), `&rcc` (`clock-frequency`, AHB/APB prescalers). Use CubeMX only as a calculator for clock values and pin AF, then transcribe into DT.
- Pins: `<periph>_<signal>_p<port><pin>` labels from `stm32<part>-pinctrl.dtsi` (generated from STM32 Open Pin Data) — if a label does not exist the pin cannot do that function.
- Runners: `stm32cubeprogrammer` (default `west flash` runner on official STM32 boards since 4.0 — install STM32CubeProgrammer; on 3.7 most boards default to `openocd`), `openocd`, `jlink`, `pyocd`. Option bytes (RDP, WRP, nBOOT) are not changed by `west flash`; TF-M board scripts do change them.
- Low power: system PM supported on many series (L4, U5, WB, G0...); check the SoC's DT `cpu-power-states` and docs per series.
- Known gaps to check per series before committing: specific peripheral modes (e.g., HRTIM, some advanced timer features), DMA support per driver, USB device on newer series. Search `drivers/` and the board doc "Supported features" table.

## 7. ESP32 in Zephyr

- Supported SoCs include ESP32, S2, S3, C2, C3, C6, H2, C5, C61, P4 (check the board catalog). Espressif maintains the port; `hal_espressif` carries ESP-IDF low-level code, Wi-Fi/BLE come as binary blobs: run `west blobs fetch hal_espressif` after `west update`.
- Board targets name the core: `esp32s3_devkitc/esp32s3/procpu` (main), `/appcpu` (second core, separate image via sysbuild).
- Boot: "simple boot" (single image, no OTA) or MCUboot via sysbuild (`SB_CONFIG_BOOTLOADER_MCUBOOT=y`), which Espressif docs describe as the path for OTA.
- Wi-Fi: DT node `&wifi { status = "okay"; }` (`espressif,esp32-wifi`), `CONFIG_WIFI=y`, Zephyr Wi-Fi management API (`net_mgmt` `NET_REQUEST_WIFI_CONNECT`) and the `wifi` shell. BLE: `zephyr,bt-hci = &esp32_bt_hci`, Zephyr host stack.
- Monitor: `west espressif monitor`. Flash with `west flash` (esptool). JTAG via built-in USB-JTAG on S3/C3/C6.
- Compared with ESP-IDF: fewer high-level components (no IDF HTTP server/provisioning/ESP-NOW parity). Secure boot (V2) and flash encryption go through the Espressif port of MCUboot, not the IDF bootloader; enabling them burns eFuses on first boot (irreversible) — check the Espressif Zephyr/MCUboot docs for the SoC and confirm before enabling. Up to 4.4 sysbuild forced overwrite-only and unsigned images on Espressif; from 4.5 it uses the generic defaults (swap using offset, signed with the MCUboot dev key — set your own). If the product depends on IDF components, stay on ESP-IDF (see `esp32-firmware`).
- PlatformIO's ESP32 Zephyr support lags upstream; use west.

## 8. Review checklist

- [ ] Decision to migrate documented (features needed, footprint, team effort).
- [ ] Concept mapping done per FreeRTOS object; priorities re-derived with Zephyr numbering.
- [ ] No pin or clock configuration left in C; no peripheral owned by both a Zephyr driver and direct HAL code.
- [ ] Custom board out-of-tree, reviewed against the schematic.
- [ ] Vendor stack choice explicit (upstream vs NCS version; Partition Manager vs DTS partitions).
- [ ] ESP32 blobs fetched in CI; licensing of blobs reviewed.
- [ ] Old firmware still buildable until the Zephyr version passes product tests.
