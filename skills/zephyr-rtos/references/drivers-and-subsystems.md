# Drivers and subsystems

## Contents
1. Driver model in one paragraph
2. GPIO
3. UART — polling, interrupt, async
4. I2C, SPI, ADC, PWM
5. CAN and CAN FD
6. Modbus subsystem
7. Sensor, counter, watchdog
8. Flash map, NVS/ZMS, settings, LittleFS
9. Networking, MQTT/LwM2M, cellular modems
10. Bluetooth LE, USB, power management, retention
11. Worked example — RS-485 Modbus RTU server on Nucleo-F429ZI
12. Review checklist

---

## 1. Driver model in one paragraph

Each enabled devicetree node with a supported `compatible` becomes a `struct device`, initialised at boot in level/priority order. Applications get it with `DEVICE_DT_GET()` (or a `*_dt_spec`), check readiness, and call the subsystem API (`gpio_*`, `uart_*`, `can_*`). The API is the same on STM32, nRF and ESP32; only the overlay differs. Before writing a driver, search `dts/bindings/` for the chip — many sensors, CAN controllers, modems and displays already exist.

## 2. GPIO

```c
static const struct gpio_dt_spec door = GPIO_DT_SPEC_GET(DT_PATH(zephyr_user), door_gpios);
static struct gpio_callback door_cb;
static void door_isr(const struct device *port, struct gpio_callback *cb, uint32_t pins)
{ k_event_post(&locker_ev, EV_DOOR); }                      /* ISR context */

if (!gpio_is_ready_dt(&door)) return -ENODEV;
gpio_pin_configure_dt(&door, GPIO_INPUT);
gpio_pin_interrupt_configure_dt(&door, GPIO_INT_EDGE_BOTH);
gpio_init_callback(&door_cb, door_isr, BIT(door.pin));
gpio_add_callback_dt(&door, &door_cb);
```

- `gpio_pin_set_dt(&spec, 1)` means **active**, not high: polarity comes from `GPIO_ACTIVE_LOW/HIGH` in DT. `gpio_pin_set_raw()` ignores polarity.
- Debounce mechanical door switches in thread context (delayable work), not in the ISR.
- `CONFIG_GPIO=y` must be set explicitly on many STM32 boards from 4.5.

## 3. UART — polling, interrupt, async

| API | Kconfig | Use |
|---|---|---|
| Polling `uart_poll_in/out` | none | Bring-up, tiny debug |
| Interrupt `uart_irq_*` + `uart_fifo_read/fill` | `CONFIG_UART_INTERRUPT_DRIVEN` | Byte streams into a ring buffer; Modbus uses this by default |
| Async `uart_callback_set`, `uart_rx_enable`, `uart_tx` | `CONFIG_UART_ASYNC_API` | DMA-backed, idle-line timeout; best for high rates and modems |

- Interrupt mode: in the callback loop `uart_irq_update()` then `uart_irq_rx_ready()` → `uart_fifo_read()` into `ring_buf`; signal a thread. 4.0+: `uart_irq_tx_ready()` returns the number of bytes that fit (check `> 0`). 4.5: `uart_irq_update()` returns void.
- Async mode: events `UART_RX_RDY` (data in buffer, offset/len), `UART_RX_BUF_REQUEST` (answer with `uart_rx_buf_rsp()`), `UART_RX_BUF_RELEASED`, `UART_RX_DISABLED`, `UART_TX_DONE`. Needs DMA channels in DT on STM32 (`dmas`/`dma-names`).
- Runtime baud change: `uart_configure()` needs `CONFIG_UART_USE_RUNTIME_CONFIGURE` (default y).
- RS-485 direction: STM32 USARTs that have the DE function (F0/F3/F7/G0/G4/L4/H7/U5..., not F1/F2/F4) support hardware DE (`de-enable;` plus `de-assert-time`/`de-deassert-time` in the `st,stm32-uart` node, `<usart>_de_p..` pin in pinctrl). Other SoCs: GPIO toggled by the protocol layer (Modbus `de-gpios`) after `UART_TX_DONE`/TX complete.

## 4. I2C, SPI, ADC, PWM

```c
static const struct i2c_dt_spec rtc = I2C_DT_SPEC_GET(DT_NODELABEL(ext_rtc));
i2c_write_read_dt(&rtc, &reg, 1, buf, sizeof(buf));        /* address comes from reg = <0x68> */

static const struct spi_dt_spec nfc = SPI_DT_SPEC_GET(DT_NODELABEL(nfc0),
        SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB);   /* 3.7: extra delay arg 0 */

static const struct adc_dt_spec vbat = ADC_DT_SPEC_GET_BY_IDX(DT_PATH(zephyr_user), 0);
adc_channel_setup_dt(&vbat);
struct adc_sequence seq = { .buffer = &raw, .buffer_size = sizeof(raw) };
adc_sequence_init_dt(&vbat, &seq); adc_read_dt(&vbat, &seq);
int32_t mv = raw; adc_raw_to_millivolts_dt(&vbat, &mv);

static const struct pwm_dt_spec hold = PWM_DT_SPEC_GET(DT_NODELABEL(lock_pwm));
pwm_set_dt(&hold, PWM_USEC(50), PWM_USEC(20));             /* period, pulse */
```

- I2C addresses are 7-bit in DT (`reg`), not shifted as in STM32 HAL.
- ADC channels are configured in DT child nodes (`channel@3 { zephyr,gain, zephyr,reference, zephyr,acquisition-time, zephyr,resolution }`); check the binding of your ADC for allowed values.
- SPI chip-select is `cs-gpios` on the bus node; timing via `spi-cs-setup-delay-ns`/`spi-cs-hold-delay-ns` (4.3+).
- Solenoid "pull-in then hold" fits PWM: full duty for pull-in time, then reduced duty; enforce maximum on-time in one owner thread.

## 5. CAN and CAN FD

- Controllers: STM32 bxCAN (`st,stm32-bxcan`, classic only), STM32 FDCAN (`st,stm32-fdcan` G0/G4/L5/U5, `st,stm32h7-fdcan` H7), ESP32 TWAI (`espressif,esp32-twai`, classic), nRF52/53/54L have no CAN — use `microchip,mcp251xfd` or `ti,tcan4x5x` over SPI.
- DT: `bitrate = <500000>;`, `bitrate-data = <2000000>;` (FD), `sample-point`, optional `phys = <&transceiver>` (`can-transceiver-gpio` for standby/enable pins). `bus-speed` was deprecated in 3.7 and removed in 4.5.
- Kconfig: `CONFIG_CAN=y`, `CONFIG_CAN_FD_MODE=y` for FD.

```c
const struct device *can = DEVICE_DT_GET(DT_CHOSEN(zephyr_canbus));
can_set_mode(can, CAN_MODE_FD);              /* or CAN_MODE_NORMAL; must be stopped */
can_set_bitrate(can, 500000);                /* or can_calc_timing() + can_set_timing() */
can_set_bitrate_data(can, 2000000);          /* FD data phase */
can_start(can);
struct can_filter f = { .id = 0x100, .mask = CAN_STD_ID_MASK, .flags = 0 };
can_add_rx_filter(can, rx_cb, NULL, &f);     /* or can_add_rx_filter_msgq() */
struct can_frame fr = { .id = 0x101, .dlc = can_bytes_to_dlc(8), .flags = CAN_FRAME_FDF | CAN_FRAME_BRS };
can_send(can, &fr, K_MSEC(100), NULL, NULL);
```

- Bus-off recovery is automatic by default; manual `can_recover()` exists with `CONFIG_CAN_MANUAL_RECOVERY_MODE`. Monitor state with `can_add_state_change_callback()` (4.5+) or `can_set_state_change_callback()` (3.7–4.4, deprecated in 4.5).
- Shell: `CONFIG_CAN_SHELL=y` → `can send`, `can dump` for bring-up.

## 6. Modbus subsystem

- `CONFIG_MODBUS=y` plus one of `CONFIG_MODBUS_ROLE_SERVER`, `_CLIENT`, `_CLIENT_SERVER`. Serial backend uses the UART interrupt API (`CONFIG_UART_INTERRUPT_DRIVEN=y`); async with `CONFIG_MODBUS_SERIAL_ASYNC_API` (4.3+). TCP/raw via `MODBUS_MODE_RAW` + `modbus_raw_*` (you supply the socket).
- DT: a child node with `compatible = "zephyr,modbus-serial"` under the UART, optional `de-gpios` (driver enable, active high when wired directly) and `re-gpios` (receiver enable, active low).
- Parity rule (spec compliant mode): even/odd → 1 stop bit, none → **2 stop bits** automatically. Custom stop bits need `CONFIG_MODBUS_NONCOMPLIANT_SERIAL_MODE` and the field is `stop_bits` (4.2+; `stop_bits_client` and client-only on 3.7).
- The RTU inter-frame timeout is derived from the baud rate (fixed above 38400 baud as the spec says).
- Server: callbacks `coil_rd/wr`, `discrete_input_rd`, `input_reg_rd`, `holding_reg_rd/wr` (and `_fp` float variants) return 0 or `-ENOTSUP` (→ illegal address exception). Client: `modbus_read_holding_regs()`, `modbus_write_holding_reg()`, etc. with `modbus_init_client()`.
- Callbacks run in the Modbus work context (the system workqueue unless `CONFIG_MODBUS_WORKQUEUE`, 4.5+, gives Modbus its own): copy data, do not block.

## 7. Sensor, counter, watchdog

- Sensor API: `sensor_sample_fetch(dev)` then `sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &val)`; `sensor_value_to_double()`. Newer read/decode API (`sensor_read`, RTIO) exists for high-rate sensors; check driver support.
- Counter: hardware timers/RTC as `counter_*` (`counter_set_channel_alarm`, `counter_get_value`), useful for long low-power wakeups.
- Watchdog:

```c
const struct device *wdt = DEVICE_DT_GET(DT_ALIAS(watchdog0));
struct wdt_timeout_cfg cfg = { .window.min = 0, .window.max = 4000, .flags = WDT_FLAG_RESET_SOC };
int ch = wdt_install_timeout(wdt, &cfg);
wdt_setup(wdt, WDT_OPT_PAUSE_HALTED_BY_DBG);
/* feed only from a supervisor that checks every critical thread checked in */
wdt_feed(wdt, ch);
```

STM32 IWDG cannot be stopped once started and has no callback; nRF WDT cannot be reconfigured until reset. The task watchdog subsystem (`CONFIG_TASK_WDT`) multiplexes per-thread software channels onto one hardware watchdog.

## 8. Flash map, NVS/ZMS, settings, LittleFS

- Partitions live in DT under the flash node (`storage_partition`, `slot0_partition`...). API: `flash_area_open(PARTITION_ID(storage_partition), &fa)` (4.4+; `FIXED_PARTITION_ID` on 3.7–4.3), `flash_area_read/write/erase`. 4.4+ boards use `compatible = "zephyr,mapped-partition"` for memory-mapped partitions.
- **NVS**: simple ID→blob store for page-erase flash (STM32, nRF52/53). Needs at least 2 sectors; STM32F4 128 KB sectors make it costly — prefer a part with small pages or external flash.
- **ZMS** (4.0+): successor designed for all memories including RRAM/MRAM (nRF54L) and large sectors. NVS and ZMS headers moved to `zephyr/kvss/` in 4.4 (`zephyr/fs/nvs.h`, `zephyr/fs/zms.h` before; the old paths remain as deprecated wrappers).
- **Settings**: key/value tree on top of NVS, ZMS, file or FCB. `CONFIG_SETTINGS=y` + backend (`CONFIG_SETTINGS_NVS`, `CONFIG_SETTINGS_ZMS` 4.1+); register with `SETTINGS_STATIC_HANDLER_DEFINE`, call `settings_subsys_init()` and `settings_load()`, write with `settings_save_one("app/unit_id", &v, sizeof(v))`. Bluetooth bonds also use it.
- **LittleFS**: power-fail-safe file system for logs/large data, ideally on external QSPI/SPI NOR; auto-mount via a `zephyr,fstab,littlefs` DT node.
- Estimate wear: writes/day × record size vs sector endurance (10k cycles typical for STM32 internal flash; check the datasheet).

## 9. Networking, MQTT/LwM2M, cellular modems

- Stack: `CONFIG_NETWORKING`, `CONFIG_NET_IPV4`, `CONFIG_NET_TCP`, `CONFIG_NET_SOCKETS`, `CONFIG_NET_DHCPV4`. Ethernet on STM32 F4/F7/H7 boards works out of the box (`nucleo_f429zi` has RMII PHY in DT).
- Use `net_if_get_default()`, `net_mgmt` event callbacks (`NET_EVENT_IPV4_ADDR_ADD`) or the connection manager (`CONFIG_NET_CONNECTION_MANAGER`, L4 connected/disconnected events) instead of polling.
- BSD-like sockets (`zsock_*`, or POSIX names with `CONFIG_POSIX_API`). TLS via `CONFIG_NET_SOCKETS_SOCKOPT_TLS` + mbedTLS with credentials in `tls_credential_add()`.
- MQTT client: `CONFIG_MQTT_LIB` (`mqtt_client_init`, `mqtt_connect`, `mqtt_input`/`mqtt_live` in a loop). LwM2M: `CONFIG_LWM2M` with built-in objects and FOTA support.
- **Cellular**: `CONFIG_MODEM=y`, `CONFIG_MODEM_CELLULAR=y`, PPP (`CONFIG_NET_L2_PPP`), modem node under the UART with a vendor compatible (`quectel,bg95`, `quectel,eg25-g`, `simcom,sim7080`, `simcom,a76xx`, `u-blox,lara-r6`, ...; full list in `dts/bindings/modem/`) and `mdm-power-gpios`/`mdm-reset-gpios`. Uses device runtime PM; the `samples/net/cellular_modem` sample shows the version-correct power-up and `net_if_up()` sequence. nRF91 uses its own modem library in NCS instead.

## 10. Bluetooth LE, USB, power management, retention

- **BLE**: `CONFIG_BT=y`, `CONFIG_BT_PERIPHERAL=y`, `bt_enable(NULL)`, advertise with `bt_le_adv_start()`, GATT services with `BT_GATT_SERVICE_DEFINE`. On nRF5340 the controller runs on the network core (sysbuild adds the `hci_ipc` image). ESP32 uses the Espressif controller blob. Store bonds with `CONFIG_BT_SETTINGS`.
- **USB**: use the new stack `CONFIG_USB_DEVICE_STACK_NEXT` (no longer experimental from 4.3; legacy `CONFIG_USB_DEVICE_STACK` deprecated in 4.5). On 3.7 the new stack is still marked experimental. CDC ACM: DT node `compatible = "zephyr,cdc-acm-uart"` under `zephyr_udc0`, `CONFIG_USBD_CDC_ACM_CLASS=y`; see `samples/subsys/usb/cdc_acm` for `usbd_*` init. Assign your own VID/PID for products.
- **Power management**: system PM `CONFIG_PM=y` (idle thread enters SoC sleep states declared in DT `cpu-power-states`), device PM `CONFIG_PM_DEVICE`, runtime PM `CONFIG_PM_DEVICE_RUNTIME` with `pm_device_runtime_get()/put()` around use. Measure with a power analyser; one forgotten enabled UART RX or debug log back-end can keep the SoC awake.
- **Retention / boot mode**: retained RAM regions (`zephyr,retention` nodes) survive warm reset; `CONFIG_RETENTION_BOOT_MODE` + `bootmode_set(BOOT_MODE_TYPE_BOOTLOADER)` then `sys_reboot()` asks MCUboot to stay in serial recovery.

## 11. Worked example — RS-485 Modbus RTU server on Nucleo-F429ZI

Hardware assumption: RS-485 transceiver (e.g., 3.3 V half-duplex) on Arduino D1/D0 = USART6 PG14 (TX) / PG9 (RX), DE and /RE tied together on D9 = PD15; console stays on USART3 (ST-Link VCP). Bias and termination per the bus design.

`boards/nucleo_f429zi.overlay`:
```dts
&usart6 {
    status = "okay";
    current-speed = <19200>;
    modbus0: modbus0 {
        compatible = "zephyr,modbus-serial";
        status = "okay";
        de-gpios = <&gpiod 15 GPIO_ACTIVE_HIGH>;   /* DE=/RE joined: high = transmit */
    };
};
```

`prj.conf`:
```
CONFIG_GPIO=y
CONFIG_SERIAL=y
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_MODBUS=y
CONFIG_MODBUS_ROLE_SERVER=y
CONFIG_LOG=y
CONFIG_MODBUS_LOG_LEVEL_DBG=y        # bring-up only
```

`src/main.c` (skeleton):
```c
#include <zephyr/kernel.h>
#include <zephyr/modbus/modbus.h>
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(app, LOG_LEVEL_INF);

#define NUM_DOORS 16
static uint16_t holding[32];
static K_MUTEX_DEFINE(reg_lock);

static int hr_rd(uint16_t addr, uint16_t *reg)
{
    if (addr >= ARRAY_SIZE(holding)) return -ENOTSUP;
    k_mutex_lock(&reg_lock, K_FOREVER); *reg = holding[addr]; k_mutex_unlock(&reg_lock);
    return 0;
}
static int hr_wr(uint16_t addr, uint16_t reg)
{
    if (addr >= ARRAY_SIZE(holding)) return -ENOTSUP;
    k_mutex_lock(&reg_lock, K_FOREVER); holding[addr] = reg; k_mutex_unlock(&reg_lock);
    /* e.g. addr 0 = "open door N": post to lock_q, never drive the solenoid here */
    return 0;
}
static int coil_wr(uint16_t addr, bool on) { return addr < NUM_DOORS ? post_lock_cmd(addr, on) : -ENOTSUP; }

static struct modbus_user_callbacks cbs = { .holding_reg_rd = hr_rd, .holding_reg_wr = hr_wr, .coil_wr = coil_wr };

int main(void)
{
    const struct modbus_iface_param p = {
        .mode = MODBUS_MODE_RTU,
        .server = { .user_cb = &cbs, .unit_id = 1 },          /* unit id from settings/DIP in a product */
        .serial = { .baud = 19200, .parity = UART_CFG_PARITY_EVEN },
    };
    int iface = modbus_iface_get_by_name(DEVICE_DT_NAME(DT_NODELABEL(modbus0)));
    if (iface < 0 || modbus_init_server(iface, p) != 0) { LOG_ERR("modbus init"); return 0; }
    return 0;   /* server runs from UART ISR + work queue */
}
```

Verify: `west build -b nucleo_f429zi -p always`; check `zephyr.dts` shows `modbus0` under `serial@40011400`; test with a USB-RS-485 adapter and a PC client (e.g., `mbpoll -m rtu -a 1 -b 19200 -P even -r 1 -c 4 /dev/tty...`); scope DE: it must drop within one character time after the last stop bit, or replies collide with the next master frame. Same structure as upstream `samples/subsys/modbus/rtu_server` (whose `app.overlay` uses `&arduino_serial`, which is USART6 on this board, D9 for DE with `GPIO_ACTIVE_LOW` for its shield, and no parity) — set DE polarity for your transceiver wiring.

## 12. Review checklist

- [ ] Every driver used is enabled in DT (`status = "okay"`) and Kconfig, and checked with `*_is_ready*`.
- [ ] GPIO polarity in DT, `gpio_pin_set_dt` used with logical values; door inputs debounced in thread context.
- [ ] UART API mode chosen deliberately; RX overruns and errors handled; RS-485 DE timing measured.
- [ ] CAN uses `bitrate`/`bitrate-data`; bus-off and error states monitored; transceiver standby pin handled.
- [ ] Modbus parity/stop-bit settings match the master; callbacks non-blocking; unit ID configurable.
- [ ] Watchdog fed by a supervisor that checks all critical threads; pause-in-debug set for development.
- [ ] Storage: partition sizes checked against sector size and wear; settings keys versioned.
- [ ] Network/modem reconnect paths tested (cable pull, no SIM, no coverage) without leaking sockets.
- [ ] PM measured on hardware in every operating mode.
