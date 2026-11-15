# Userspace hardware access

## Contents
1. Driver or userspace?
2. GPIO with libgpiod (v1 and v2)
3. I2C with i2c-dev and i2c-tools
4. SPI with spidev
5. Serial ports, termios and RS-485
6. CAN with SocketCAN
7. Watchdog from an application
8. Sysfs class devices (hwmon, IIO, LED, PWM)
9. Permissions, udev and stable names
10. Worked example — Modbus RTU over RS-485 on a Verdin UART
11. Review checklist

---

## 1. Driver or userspace?

Userspace is right when access is occasional, latency of a few ms is fine, one process owns the device, and the protocol is simple. A crash in userspace restarts a service; a crash in a driver can take down the kernel. Move to a kernel driver (`kernel-drivers.md`) when you need fast IRQ response, DMA, sharing across processes, or integration with an existing subsystem. Before raw I2C/SPI, check whether a DT node would give you a ready hwmon/IIO/RTC device (section 8).

## 2. GPIO with libgpiod (v1 and v2)

The tools changed syntax between libgpiod v1 (most images up to ~2023) and v2. Check with `gpiodetect --version`.

| Task | v1 | v2 |
|---|---|---|
| List chips | `gpiodetect` | `gpiodetect` |
| List lines with names/consumers | `gpioinfo` | `gpioinfo` |
| Find a line by name | `gpiofind SODIMM_206` | `gpioinfo SODIMM_206` (names accepted directly) |
| Read | `gpioget gpiochip3 5` | `gpioget -c gpiochip3 5` or `gpioget SODIMM_206` |
| Write and hold | `gpioset --mode=signal gpiochip3 5=1` | `gpioset -c gpiochip3 5=1` (holds until killed) |
| Wait for edges | `gpiomon gpiochip3 5` | `gpiomon -c gpiochip3 5` |

Prefer **line names** (Toradex BSPs name lines after SODIMM pins) over chip/offset numbers, which change between SoMs.

On Torizon the host OS and the container can differ: Torizon OS 7 (Yocto scarthgap) ships libgpiod v2 tools, but `torizon/debian:3`/`:4` containers are Debian 12 Bookworm, whose `libgpiod2`/`libgpiod-dev`/`gpiod` packages are **libgpiod 1.6.x (v1 API and v1 tool syntax)**. The package name `libgpiod2` is the soname, not the API version; the v2 API comes as `libgpiod3` in Debian 13 Trixie. Code using the v2 API below will not build against Bookworm's `libgpiod-dev`.

C with libgpiod v2:
```c
#include <gpiod.h>

struct gpiod_chip *chip = gpiod_chip_open("/dev/gpiochip3");
struct gpiod_line_settings *s = gpiod_line_settings_new();
gpiod_line_settings_set_direction(s, GPIOD_LINE_DIRECTION_OUTPUT);
gpiod_line_settings_set_output_value(s, GPIOD_LINE_VALUE_INACTIVE);

struct gpiod_line_config *lc = gpiod_line_config_new();
unsigned int offset = 5;
gpiod_line_config_add_line_settings(lc, &offset, 1, s);

struct gpiod_request_config *rc = gpiod_request_config_new();
gpiod_request_config_set_consumer(rc, "myapp");

struct gpiod_line_request *req = gpiod_chip_request_lines(chip, rc, lc);
if (!req) { perror("request"); /* EBUSY: owned by a driver or another process */ }
gpiod_line_request_set_value(req, offset, GPIOD_LINE_VALUE_ACTIVE);
/* ... */
gpiod_line_request_release(req);   /* also released automatically when the process exits */
```
Python: `gpiod` package (v2 bindings) or `python3-libgpiod` (v1) — the APIs differ; pin the version.

Traps:
- `EBUSY` → the line is claimed by a kernel driver (e.g. `gpio-keys`, a regulator) or by another process; `gpioinfo` shows the consumer.
- When the requesting process exits, the line state is not guaranteed — design hardware with pull resistors for the safe state.
- Edge events: use the event API with a timestamp, not a polling loop on `get_value`.

## 3. I2C with i2c-dev and i2c-tools

```
i2cdetect -l                       # list adapters
i2cdetect -y -r 3                  # scan bus 3 using read-byte probing (safer for some chips)
i2cget -y 3 0x48 0x00 w            # read word from register 0
i2cset -y 3 0x20 0x01 0xff         # write — can change hardware state, be careful
i2ctransfer -y 3 w1@0x48 0x00 r2   # write register pointer, repeated start, read 2 bytes
```
`UU` in the scan = a kernel driver owns that address; access through the driver's interface instead.

C, with a combined write-then-read (repeated start):
```c
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <sys/ioctl.h>

int fd = open("/dev/i2c-3", O_RDWR);
uint8_t reg = 0x00, buf[2];
struct i2c_msg msgs[2] = {
    { .addr = 0x48, .flags = 0,        .len = 1, .buf = &reg },
    { .addr = 0x48, .flags = I2C_M_RD, .len = 2, .buf = buf  },
};
struct i2c_rdwr_ioctl_data x = { .msgs = msgs, .nmsgs = 2 };
if (ioctl(fd, I2C_RDWR, &x) < 0) perror("I2C_RDWR");   /* EREMOTEIO = NACK */
```
For simple register access `ioctl(fd, I2C_SLAVE, addr)` then `read`/`write` works; `I2C_SLAVE_FORCE` bypasses a bound driver and is almost always a mistake. Bus speed is set in DT (`clock-frequency`), not from userspace. A stuck bus (SDA held low) is usually recovered by the controller driver if `scl-gpios`/`sda-gpios` recovery is described in DT.

## 4. SPI with spidev

spidev needs a DT child node on the SPI controller. A bare `compatible = "spidev"` is rejected by modern kernels; use a compatible listed in `drivers/spi/spidev.c` of your kernel, or bind by override:
```
echo spidev > /sys/bus/spi/devices/spi1.0/driver_override
echo spi1.0 > /sys/bus/spi/drivers/spidev/bind
```
Toradex images commonly expose `/dev/spidevX.Y` and `/dev/verdin-spi-*` symlinks when the SPI overlay is enabled.

```c
#include <linux/spi/spidev.h>
int fd = open("/dev/spidev1.0", O_RDWR);
uint8_t mode = SPI_MODE_0; uint32_t speed = 1000000;
ioctl(fd, SPI_IOC_WR_MODE, &mode);
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
uint8_t tx[3] = {0x80 | 0x0F, 0, 0}, rx[3];
struct spi_ioc_transfer t = { .tx_buf = (uintptr_t)tx, .rx_buf = (uintptr_t)rx,
                              .len = 3, .speed_hz = speed, .bits_per_word = 8 };
ioctl(fd, SPI_IOC_MESSAGE(1), &t);   /* CS held for the whole message */
```
Test wiring with `spidev_test -D /dev/spidev1.0 -v` (loopback MOSI→MISO). Default transfer buffer is limited (`bufsiz` module parameter, 4096 by default).

## 5. Serial ports, termios and RS-485

i.MX UARTs appear as `/dev/ttymxcN`; Torizon adds `/dev/verdin-uartN` symlinks. Map SODIMM pins → UART from the module datasheet; one Verdin UART is the system console — do not use it for the application.

Raw mode with termios:
```c
int fd = open("/dev/verdin-uart1", O_RDWR | O_NOCTTY);
struct termios t;
tcgetattr(fd, &t);
cfmakeraw(&t);                       /* no echo, no line editing, no CR/LF translation */
cfsetspeed(&t, B115200);
t.c_cflag |= CLOCAL | CREAD;
t.c_cflag &= ~CRTSCTS;               /* flow control off unless wired */
t.c_cc[VMIN] = 0; t.c_cc[VTIME] = 1; /* read returns when any data arrives, or 0 after 100 ms */
tcsetattr(fd, TCSANOW, &t);
```
Forgetting `cfmakeraw` is the classic bug: bytes 0x0D/0x11/0x13 vanish or change.

RS-485 with the UART's RTS driving the transceiver DE pin:
```c
#include <linux/serial.h>
struct serial_rs485 rs = {0};
rs.flags = SER_RS485_ENABLED | SER_RS485_RTS_ON_SEND;   /* DE high while sending */
rs.delay_rts_before_send = 0;
rs.delay_rts_after_send = 0;
if (ioctl(fd, TIOCSRS485, &rs) < 0) perror("TIOCSRS485");  /* ENOTTY: driver lacks RS-485 */
```
The `RTS_ON_SEND`/`RTS_AFTER_SEND` pair sets DE polarity and must match the transceiver wiring. On i.MX (`imx` UART driver) Toradex documents active-low RTS: clear `SER_RS485_RTS_ON_SEND`, set `SER_RS485_RTS_AFTER_SEND`, and set `SER_RS485_RX_DURING_TX` (required on i.MX); RTS delays (`rs485-rts-delay`) are not supported on i.MX8. Or in DT on the UART node: `linux,rs485-enabled-at-boot-time;`, `rs485-rts-active-low;`, `rs485-rx-during-tx;`. On the Verdin Development Board UART_1 is the RS-485 port. Verify DE timing with a scope: the driver must release DE only after the last stop bit has left the shift register; a DE released early truncates the last byte. Electrical side (termination, biasing): a bus/interface hardware skill if installed.

## 6. CAN with SocketCAN

```
ip link set can0 down
ip link set can0 type can bitrate 500000 restart-ms 100        # auto-recover from bus-off
ip link set can0 type can bitrate 500000 dbitrate 2000000 fd on  # CAN FD
ip link set can0 up
ip -details -statistics link show can0                            # state, error counters, bitrate
candump -td -e can0                                               # with error frames
cansend can0 123#DEADBEEF
cangen can0 -g 10                                                 # load test
```
- The i.MX8M Mini has no CAN controller: Verdin iMX8M Mini gets one CAN (CAN_1) from an on-module MCP2518FD on ECSPI3 (`mcp251xfd` driver), assembled only on some variants (e.g. Quad 2GB IT / WB IT); CAN_2 is not available as standard; others need an external SPI CAN controller. The Verdin iMX8M Plus has two native FlexCAN (CAN_1 = `can0`, CAN_2 = `can1`, altnames `verdin-can1`/`verdin-can2`). Check `ip -d link`.
- Bitrate is set once by the system (systemd-networkd `.network` with `[CAN] BitRate=` or a oneshot service), not by every application.
- A CAN interface lives in a network namespace: a container sees it only with `network_mode: host` (or by moving the interface into the container's namespace).
- `ERROR-PASSIVE` / `BUS-OFF` with zero RX usually means no other node acknowledges, missing termination, or wrong bitrate — not a software bug.

Application code: `socket(PF_CAN, SOCK_RAW, CAN_RAW)`, `bind` to the interface index, `read`/`write` `struct can_frame` (or `canfd_frame` with `CAN_RAW_FD_FRAMES`); set filters with `CAN_RAW_FILTER` to reduce wakeups.

## 7. Watchdog from an application

Only if systemd does not already own `/dev/watchdog` (check `systemctl show -p RuntimeWatchdogUSec`; non-zero means systemd owns it — then use `WatchdogSec=` in the service, see `reliability.md`).
```c
#include <linux/watchdog.h>
int fd = open("/dev/watchdog0", O_WRONLY);
int timeout = 30;
ioctl(fd, WDIOC_SETTIMEOUT, &timeout);
for (;;) {
    if (all_health_checks_ok()) ioctl(fd, WDIOC_KEEPALIVE, 0);
    sleep(5);
}
/* clean shutdown only: write(fd, "V", 1); close(fd);  (magic close, ignored if NOWAYOUT) */
```
Kick only after real health checks pass; a dedicated thread that kicks unconditionally defeats the purpose. `WDIOC_GETBOOTSTATUS` tells whether the last reset was a watchdog reset.

## 8. Sysfs class devices (hwmon, IIO, LED, PWM)

```
cat /sys/class/hwmon/hwmon*/name; cat /sys/class/hwmon/hwmon2/temp1_input     # m°C
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw  in_voltage_scale         # value*scale = mV
echo heartbeat > /sys/class/leds/status/trigger
echo 0 > /sys/class/pwm/pwmchip0/export
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period; echo 500000 > .../duty_cycle; echo 1 > .../enable
```
Find devices by `name`, not by index — `hwmon2` can become `hwmon3` after a kernel or DT change. Open sysfs files once and `pread` them in a loop for periodic reads.

## 9. Permissions, udev and stable names

- Run the app as a normal user in groups `gpio`, `i2c`, `spidev`, `dialout` (Torizon's `torizon` user already has them); never run everything as root to "fix" permissions.
- udev rule for a stable name and permissions:
  ```
  # /etc/udev/rules.d/90-myapp.rules
  SUBSYSTEM=="tty", KERNELS=="30860000.serial", SYMLINK+="rs485", GROUP="dialout", MODE="0660"   # i.MX8M Mini UART1; 30880000 = UART3 = Verdin console
  ```
  Find matching attributes with `udevadm info -a -n /dev/ttymxc0`. On Torizon `/etc` is writable (OSTree merges it), but ship rules in the image or container setup, not by hand.

## 10. Worked example — Modbus RTU over RS-485 on a Verdin UART

1. Hardware: transceiver DE/RE from UART RTS; check in the module datasheet that the chosen UART has RTS on the SODIMM.
2. Enable RS-485 in DT (`linux,rs485-enabled-at-boot-time`) or with `TIOCSRS485` at app start.
3. Open with termios raw, 19200 8E1 (`PARENB`, no `PARODD`) per the slave's settings.
4. Use libmodbus: `modbus_new_rtu("/dev/verdin-uart1", 19200, 'E', 8, 1)`, `modbus_rtu_set_serial_mode(ctx, MODBUS_RTU_RS485)`, response timeout ~ 500 ms.
5. In a container: `devices: ["/dev/verdin-uart1"]`, `group_add: ["dialout"]`.
6. Verify the 3.5-character inter-frame gap and DE release on a scope; if responses are cut, check the RTS-after-send delay and the transceiver's enable time.

## 11. Review checklist

- [ ] Each device has exactly one owner (kernel driver or one process); `EBUSY`/`UU` understood.
- [ ] GPIO via libgpiod by line name; safe states guaranteed by hardware pulls.
- [ ] I2C reads that need a repeated start use `I2C_RDWR`; no `I2C_SLAVE_FORCE`.
- [ ] Serial opened raw (`cfmakeraw`), timeouts set, flow control matches wiring; console UART untouched.
- [ ] RS-485 direction handled by the driver, DE timing verified on a scope.
- [ ] CAN bitrate configured by the system; bus-off recovery set; container uses host networking for CAN.
- [ ] Every blocking call has a timeout; every ioctl/read/write return value checked.
- [ ] Runs as a non-root user with the right groups; stable device names via symlinks/udev or sysfs `name`.
