# Raspberry Pi (Pi 3/4/5, Zero 2 W, CM4/CM5)

## Contents
1. Models, SoCs and the Pi 5 RP1 split
2. Boot: EEPROM bootloader, config.txt, cmdline.txt
3. Overlays: dtoverlay/dtparam, custom overlays, debugging
4. Interfaces: I2C, SPI, UART, RS-485, CAN, PWM, 1-Wire, watchdog, HAT EEPROM
5. Camera and display
6. OS, Python and builds
7. Reliability in products
8. Worked example — RS-485 Modbus + CAN HAT on a Pi 4 vs a Pi 5
9. Common failures
10. Review checklist

Only Pi-specific material lives here. Generic DT and overlay syntax: `boot-and-device-tree.md`. libgpiod, i2c-dev, spidev, termios/`TIOCSRS485`, SocketCAN and watchdog code: `userspace-hw-access.md`. Toolchains and containers: `cross-build-containers.md`. Read-only rootfs, OTA and power-fail rules: `reliability.md`. Authoritative sources: raspberrypi.com/documentation/computers (config.txt, configuration, raspberry-pi, camera, compute-module, os) and the overlays README in github.com/raspberrypi/linux (`arch/arm/boot/dts/overlays/README`, also installed as `/boot/firmware/overlays/README`).

---

## 1. Models, SoCs and the Pi 5 RP1 split

| Board | SoC | Header I/O comes from | Bootloader | Notes |
|---|---|---|---|---|
| Pi 3B/3B+, CM3/3+ | BCM2837 | SoC | ROM + `bootcode.bin` on the FAT partition | 1 GB RAM max; 3B+ has a 60 °C soft thermal limit |
| Zero 2 W | RP3A0 (BCM2710A1) | SoC | as Pi 3 | 512 MB; PL011 on Bluetooth, mini-UART on GPIO 14/15 |
| Pi 4B / CM4 | BCM2711 | SoC | SPI EEPROM | Extra PL011 UART2–5, i2c3–6, spi3–6 by overlay |
| Pi 5 / CM5 | BCM2712 + **RP1** | RP1 (over PCIe) | SPI EEPROM, no `start*.elf` | RTC, power button, 3-pin debug UART, 2× 4-lane CSI/DSI, PCIe |

**What RP1 changes on Pi 5** (GPIO, UART, SPI, I2C, PWM on the header all live in the RP1 south-bridge, not the SoC):
- **gpiochip numbering.** Early Pi 5 kernels exposed the header as `gpiochip4`; current kernels make the RP1 controller `gpiochip0` (label `pinctrl-rp1`) and push the SoC banks to `gpiochip10`+ (raspberrypi/linux PR #6144, 2024). The kernel keeps no alias; Raspberry Pi OS adds a udev symlink `/dev/gpiochip4` → `gpiochip0` for old code, so `gpiodetect` may list the chip twice. Never hard-code the chip: find it by label (`gpiodetect`) or use line names (`gpioinfo GPIO17`; Pi kernels name lines `GPIO<n>`).
- **Register-banging libraries break.** RPi.GPIO, pigpio and old wiringPi poke BCM283x registers and do not work on Pi 5. Use libgpiod, gpiozero (lgpio backend) or the `rpi-lgpio` drop-in shim.
- **`pinctrl` replaces `raspi-gpio`.** `pinctrl get 14` shows mux/pull/level; `pinctrl set 17 op dh` drives a pin for bench tests (it fights the kernel if a driver owns the pin — test only).
- **Bit-banging is slower.** Every GPIO access crosses PCIe; software SPI/1-Wire timing is worse than Pi 4. Use the hardware blocks or `pwm-pio` (RP1 PIO).
- **UARTs.** No mini-UART; UART0–4 are PL011 (disabled by default) plus UART10, the debug UART on the 3-pin JST header (`/dev/ttyAMA10`). Bluetooth has its own dedicated UART, so GPIO 14/15 is free without `disable-bt`.
- **Overlay names.** The firmware's overlay map silently substitutes `uartN` → `uartN-pi5`, `i2c3` → `i2c3-pi5`, `w1-gpio` → `w1-gpio-pi5`, etc. **Same overlay name, different pins**: `uart2` is GPIO 0–3 on Pi 4 but GPIO 4–7 on Pi 5. Always read the README entry for the `-pi5` variant.
- **PWM.** Hardware PWM is RP1 PWM0, four channels: GPIO 12/13/18/19 = channels 0/1/2/3. The stock `pwm`/`pwm-2chan` overlays still work as written: the `pwm` label points at RP1 PWM0 and the RP1 pin driver translates the BCM283x `func` numbers (GPIO 18 + func 2 → `pwm0`). What changes is the channel: GPIO 18 is channel **2** on Pi 5 (channel 0 on Pi 4). Verify the mux with `pinctrl get 18` (expect `pwm0`) and find the chip by its `device` link under `/sys/class/pwm/` (RP1 PWM0 is `1f00098000.pwm`), not by `pwmchipN`.
- **Kernel.** Pi 5 boots `kernel_2712.img` (16 K pages). A few prebuilt binaries/containers assuming 4 K pages crash; `kernel=kernel8.img` is the escape hatch.

## 2. Boot: EEPROM bootloader, config.txt, cmdline.txt

```
Pi 3 / Zero 2 W: ROM → bootcode.bin → start*.elf (GPU) → config.txt → kernel8.img + DTB + overlays
Pi 4 / CM4:      ROM → EEPROM bootloader → start4.elf → config.txt → kernel8.img ...
Pi 5 / CM5:      ROM → EEPROM bootloader (firmware inside) → config.txt → kernel_2712.img ...
```
The GPU firmware, not U-Boot, parses `config.txt`, merges the DTB and overlays, and passes `cmdline.txt` to the kernel.

| Task | Command |
|---|---|
| Bootloader version / update status | `vcgencmd bootloader_version`; `sudo rpi-eeprom-update` |
| Apply latest bootloader (on reboot) | `sudo rpi-eeprom-update -a` — ask first; mask `rpi-eeprom-update.service` in products that pin a version |
| Show / edit bootloader config | `rpi-eeprom-config`; `sudo rpi-eeprom-config --edit` |
| Boot order | `BOOT_ORDER` nibbles read right to left: 1 SD/eMMC, 2 network, 3 rpiboot, 4 USB-MSD, 6 NVMe, f restart. Default `0xf41` (SD, then USB, loop) |
| Effective firmware config | `vcgencmd get_config int`; `vcgencmd get_config str` |
| Firmware log (overlay errors!) | `sudo vclog --msg`; add `dtdebug=1` to `config.txt` for detail |

- **Paths:** Bookworm and later mount the boot FAT at `/boot/firmware/`; `/boot/config.txt` is a stub/legacy path. Edits to the wrong file are silently ignored.
- **`config.txt` filters** let one image serve several boards: `[pi4]`, `[pi5]`, `[cm4]`, `[cm5]`, `[all]` (CM4 also sees `[pi4]`, Pi 500 sees `[pi5]`).
- **`cmdline.txt` is one single line.** A line break truncates it and the root FS is not found. Contains `console=serial0,115200` when the serial console is on — remove it (and disable `serial-getty@`) before giving that UART to an application.
- **A/B updates:** `sudo reboot '0 tryboot'` boots once with `tryboot.txt`; any reset falls back to `config.txt`. `tryboot_a_b=1` in `autoboot.txt` switches whole partitions. This is the Pi-native building block for OTA rollback.

## 3. Overlays: dtoverlay/dtparam, custom overlays, debugging

`dtparam=` sets base-DTB parameters (`i2c_arm`, `spi`, `watchdog`, `uart0`, …); `dtoverlay=name,param=val` loads `/boot/firmware/overlays/name.dtbo`. Order matters: a `dtparam` line after a `dtoverlay` applies to that overlay. Loading an overlay that needs I2C/SPI enables the bus itself.

| Command | Purpose |
|---|---|
| `dtoverlay -a` / `dtoverlay -h mcp2515` / `dtparam -h spi` | List available overlays / show one's parameters |
| `sudo dtoverlay mcp2515-can0 oscillator=12000000` | Load at runtime (configfs) — for bench tests only |
| `dtoverlay -l`; `sudo dtoverlay -r <n>` | List / remove runtime overlays |
| `ls /proc/device-tree/...`; `dtc -I fs /proc/device-tree` | Check what the kernel actually received |

Custom overlay for a TMP102 on the header I2C (there is also a stock `dtoverlay=i2c-sensor,tmp102` — prefer stock overlays when they exist):
```dts
/dts-v1/;
/plugin/;
/ {
    compatible = "brcm,bcm2835";             /* Pi convention for all models, incl. Pi 5 */
    fragment@0 {
        target = <&i2c_arm>;                  /* header I2C (GPIO 2/3), i2c-1 */
        __overlay__ {
            #address-cells = <1>; #size-cells = <0>;
            status = "okay";
            tmp: tmp102@48 { compatible = "ti,tmp102"; reg = <0x48>; };
        };
    };
    __overrides__ { addr = <&tmp>,"reg:0"; };   /* dtoverlay=tmp102-custom,addr=0x49 */
};
```
```
dtc -@ -I dts -O dtb -o tmp102-custom.dtbo tmp102-custom-overlay.dts
sudo cp tmp102-custom.dtbo /boot/firmware/overlays/ && echo 'dtoverlay=tmp102-custom' | sudo tee -a /boot/firmware/config.txt
```
The Pi base DTBs ship with symbols, so `-@` on the overlay is all that is needed. SPI CAN controller: the stock `mcp2515-can0` overlay is the reference pattern — it enables `spi0`, disables `spidev0` (CE0), adds a `fixed-clock` node for the crystal and a `microchip,mcp2515` node with `interrupts = <25 IRQ_TYPE_LEVEL_LOW>`, and exposes `oscillator`, `spimaxfrequency` and `interrupt` overrides. The `oscillator` value **must equal the crystal on the HAT** (8, 12, 16 MHz all exist); a wrong value gives a wrong bitrate and only error frames. `mcp251xfd` (MCP2517FD/2518FD) takes `spi0-0`, `oscillator`, `interrupt`, optional `rx_interrupt`, `xceiver_enable`.

## 4. Interfaces

Enable without the menu (0 = enable, 1 = disable): `sudo raspi-config nonint do_i2c 0`, `do_spi 0`, `do_serial_hw 0`, `do_serial_cons 1`, `do_onewire 0`, `do_overlayfs 0`, `do_ssh 0`. On Pi 5 `do_serial_hw` writes `dtparam=uart0=on`; elsewhere `enable_uart=1`. In a product, prefer writing `config.txt` from the image build so the result is versioned.

| Interface | Pi 3 / 4 / Zero 2 W | Pi 5 | Notes |
|---|---|---|---|
| I2C header | `dtparam=i2c_arm=on` → `/dev/i2c-1`, GPIO 2/3 (1.8 kΩ pull-ups on board) | same | `i2c_arm_baudrate=` sets speed; extra buses `i2c3`–`i2c6` (Pi 4) / `i2c0-pi5`… |
| I2C-0 (GPIO 0/1) | Reserved for the HAT ID EEPROM | same | Do not use for sensors; `force_eeprom_read=0` stops the firmware reading it |
| SPI0 | `dtparam=spi=on` → `spidev0.0` (CE0 GPIO 8), `spidev0.1` (CE1 GPIO 7) | same | `spi0-1cs` frees GPIO 7; `spi1-1cs/2cs/3cs` for SPI1 (CS on GPIO 18/17/16) |
| Primary UART on GPIO 14/15 | Pi 3/4/Zero 2 W: **mini-UART** (`ttyS0`) while Bluetooth has the PL011 | **PL011 UART0** (`dtparam=uart0=on` or `uart0-pi5`) → `ttyAMA0` | `/dev/serial0` is the console UART: GPIO 14/15 on Pi 3/4/Zero 2 W, but `ttyAMA10` (debug header) on Pi 5 — open `/dev/ttyAMA0` explicitly there |
| Console | `serial0` on GPIO 14/15 | `ttyAMA10` on the debug header | Pi 5 with `enable_uart=1` and no debug cable moves kernel logs to GPIO 14/15 |
| Extra UARTs | Pi 4: `uart2`–`uart5` (PL011, rs485 params) | `uart0`–`uart4` → `-pi5` variants, other pins | Resolve node: `ls -l /sys/class/tty/ttyAMA*/device` |
| PWM | `pwm`, `pwm-2chan` (GPIO 12/13/18/19; analogue audio uses PWM) | same overlays → RP1 PWM0, GPIO 12/13/18/19 = channels 0/1/2/3; `pwm-pio` on any GPIO 0–27 | sysfs PWM, see `userspace-hw-access.md` §8 |
| 1-Wire | `dtoverlay=w1-gpio,gpiopin=4` | auto-mapped to `w1-gpio-pi5` | `/sys/bus/w1/devices/28-*/temperature`; external 4.7 kΩ pull-up |
| Watchdog | `bcm2835-wdt`, `/dev/watchdog0`; on by default in current base DT (`dtparam=watchdog=off` disables it) | same driver (BCM2712 PM block) | Hardware max ≈ 16 s (20-bit counter, 1/65536 s ticks). Longer `RuntimeWatchdogSec=` is accepted, but the kernel then pings the hardware itself, so the extra time covers userspace hangs only |

**Mini-UART trap (Pi 3/4/Zero 2 W):** its baud clock is the VPU core clock, it has no parity, no break or framing-error detection and small FIFOs. `enable_uart=1` pins the core clock so the baud rate holds. For Modbus (parity!) or anything above ~115200, use a PL011: `dtoverlay=disable-bt` (Bluetooth off, PL011 back on 14/15; also `sudo systemctl disable hciuart`) or `miniuart-bt` (Bluetooth on the mini-UART, reduced BT throughput), or a Pi 4 `uart2`–`uart5`.

**RS-485:** the `uartN` / `uartN-pi5` overlays take `rs485` (RTS drives DE, implies `ctsrts`), `rs485_invert_rts`, `rs485_rts_on_delay`, `rs485_rts_off_delay` (ms), `rs485_full_duplex`. This sets the DT RS-485 properties so the PL011 driver toggles RTS; equivalent to `TIOCSRS485` from the app. Verify on a scope that DE drops after the last stop bit.

**CAN:** MCP2515 HATs → `mcp2515-can0` (spi0.0) / `mcp2515-can1`, or `mcp2515,spi1-0,...` on other buses; MCP2518FD HATs → `mcp251xfd`. The MCP2515 has two RX buffers and SPI + IRQ latency on Linux; at 500 kbit/s–1 Mbit/s with busy buses it drops frames (`ip -s link` overruns). Use an MCP2518FD for heavy traffic or CAN FD. Bitrate and `restart-ms` set by systemd-networkd, as in `userspace-hw-access.md` §6.

**HAT ID EEPROM:** a 24Cxx at 0x50 on ID_SD/ID_SC (GPIO 0/1) holding vendor info, GPIO setup and a DT overlay the firmware applies automatically. Format, `eepmake`/`eepflash.sh` tools and the Pi 5 "HAT+" variant: github.com/raspberrypi/hats and raspberrypi/utils (`eeptools`). For a product carrier, an EEPROM removes `config.txt` edits from the field process.

## 5. Camera and display

- **Stack:** libcamera + `rpicam-apps` (`rpicam-hello --list-cameras`, `rpicam-still`, `rpicam-vid`; the old `libcamera-*` names are deprecated compatibility links — don't use them in new scripts). Python: Picamera2. The legacy `raspistill`/MMAL stack is gone on 64-bit Bookworm and Pi 5.
- **Detection:** `camera_auto_detect=1` (default) probes official sensors. For third-party modules set `camera_auto_detect=0` and `dtoverlay=imx219` (or `imx477`, `ov5647`, …); on Pi 5/CM add `,cam0` for the CAM0 port. Check sensor I2C with `rpicam-hello --list-cameras` and `dmesg | grep -i imx`.
- **Connectors:** Pi 3/4 use a 15-pin 1 mm-pitch FFC; Pi 5, Zero 2 W and the CM IO boards use 22-pin 0.5 mm. Adapter cables exist; pinouts on the camera hardware page of the official docs — do not guess them.
- **Display:** KMS is the only supported driver (`dtoverlay=vc4-kms-v3d`, default). Legacy `vc4-fkms-v3d` is deprecated — don't use it in new products. DSI panels: `display_auto_detect=1` for the official ones, otherwise e.g. `dtoverlay=vc4-kms-dsi-7inch` (`dsi0` param for the other port). Low CMA (`cma-` params) shows as camera/video buffer allocation failures.

## 6. OS, Python and builds

- **Raspberry Pi OS:** Bookworm (Debian 12) and Trixie (Debian 13, current). Use 64-bit (`aarch64`) on Pi 3/4/5/Zero 2 W. Networking is NetworkManager (`nmcli`); `dhcpcd.conf` and `wpa_supplicant.conf` edits no longer apply. Upgrade between releases by re-imaging, not `apt full-upgrade` across releases.
- **Python (PEP 668):** system `pip install` fails with "externally-managed-environment". Use `apt install python3-xyz`, or `python3 -m venv --system-site-packages .venv` (keeps apt-installed `python3-lgpio`, `python3-picamera2`, `python3-libgpiod` visible). Never `--break-system-packages` in a product.
- **GPIO libraries:** gpiozero (default backend lgpio) works on all models; RPi.GPIO does not on Pi 5 (`rpi-lgpio` is an API-compatible replacement); libgpiod for C/C++ and services.
- **Cross-compiling:** Debian's `aarch64-linux-gnu-gcc` with a sysroot copied from the target (`rsync -a pi:/usr/include pi:/usr/lib pi:/lib sysroot/`) or build in an `arm64` `debian:bookworm`/`trixie` container under QEMU binfmt. Match the target's Debian release or binaries hit "GLIBC_2.xx not found". Details: `cross-build-containers.md`.
- **Docker on the Pi:** install from Docker's Debian repository; device access rules as in `cross-build-containers.md` (CAN needs `network_mode: host`; GPIO needs `/dev/gpiochip0` and the `gpio` group).
- **Product images:** Raspberry Pi OS + `rpi-image-gen`/pi-gen for fast starts; Yocto `meta-raspberrypi` (`MACHINE = raspberrypi4-64`/`raspberrypi5`, `ENABLE_UART`, `RPI_EXTRA_CONFIG`, `KERNEL_DEVICETREE` overlays) or Buildroot (`raspberrypi4_64_defconfig`, `raspberrypi5_defconfig`) for minimal, reproducible, read-only images with A/B OTA.

## 7. Reliability in products

| Risk | Pi-specific facts and mitigations |
|---|---|
| SD wear/corruption | Consumer SD cards die from writes and power cuts. Use `raspi-config nonint do_overlayfs 0` (overlayroot tmpfs, optional RO boot partition), journald `Storage=volatile`, industrial pSLC cards, or a CM4/CM5 with eMMC; NVMe on Pi 5 for heavy writes |
| Power | 5.1 V supply. Pi 4: 3 A. Pi 5: 5 A via USB-PD (official 27 W PSU); on a 3 A supply USB peripherals are limited to 600 mA total and USB boot is off unless `usb_max_current_enable=1`. Feeding 5 V via the GPIO header bypasses protection; on Pi 5 set `PSU_MAX_CURRENT` (e.g. `5000`) in the bootloader config if so |
| Undervoltage | `vcgencmd get_throttled`: bit 0 undervoltage now, 1 ARM freq capped, 2 throttled, 3 soft temp limit; bits 16–19 the same "has occurred since boot". Log it at startup and periodically; non-zero bit 16 explains random SD corruption and USB resets |
| Thermal | Firmware throttles progressively from 80 °C and hard at 85 °C (`vcgencmd measure_temp`). Pi 5 needs the active cooler or a heatsink in an enclosure; fan curve via `fan_temp*` dtparams |
| Time | Pi 5/CM5 have an RTC (`/dev/rtc0`) with a J5 battery connector; use a rechargeable Li-Mn cell and enable charging with `dtparam=rtc_bbat_vchg=3000000`. Pi 3/4: add an I2C RTC (`dtoverlay=i2c-rtc,ds3231`) or rely on NTP |
| Identity | Serial number: `/proc/device-tree/serial-number` (also `Serial` in `/proc/cpuinfo`); model: `/proc/device-tree/model`. Customer OTP rows (`vcgencmd otp_dump`, rows 36–43 on BCM2837/BCM2711, rows 77–84 on BCM2712) are **one-time** — writing needs explicit user approval |
| Watchdog | One owner: systemd `RuntimeWatchdogSec=10s` (keep ≤ 15 s so the hardware timer itself enforces it) + `WatchdogSec=` per service; see `reliability.md` |
| OTA | `tryboot` A/B (section 2), Mender (meta-mender / `mender-convert` for Pi OS images), RAUC (`meta-rauc-community`), SWUpdate. Bootloader EEPROM updates are a separate, riskier channel — gate them |
| CM provisioning | Fit the nRPIBOOT jumper, connect USB, run `rpiboot` from github.com/raspberrypi/usbboot: the eMMC appears as a USB disk (`rpiboot -d mass-storage-gadget64` on CM5, also for CM4). Flash with `rpi-imager` or `dd`/`bmaptool`; `rpi-sb-provisioner` for secure boot at volume. Erasing eMMC and secure-boot key programming are irreversible — confirm first |

## 8. Worked example — RS-485 Modbus + CAN HAT on a Pi 4 vs a Pi 5

Hardware: MCP2515 CAN HAT on SPI0 CE0 with a 16 MHz crystal and INT on GPIO 25 (read your HAT's schematic for both); an RS-485 transceiver whose DE/RE is driven by UART RTS.

```ini
[pi4]
# Bluetooth keeps the PL011 on 14/15 -> use UART3 (TX GPIO4, RX GPIO5, RTS GPIO7)
dtoverlay=uart3,rs485
dtoverlay=spi0-1cs                 # GPIO7 is SPI0 CE1 by default -> free it for RTS
[pi5]
# UART0 on 14/15 is free (BT has its own UART); RTS on GPIO17
dtoverlay=uart0-pi5,rs485
[all]
dtparam=spi=on
dtoverlay=mcp2515-can0,oscillator=16000000,interrupt=25
```
Do **not** reuse `uart3` on the Pi 5: it maps to `uart3-pi5` on different pins. Do not set `enable_uart=1` on the Pi 5 (kernel logs could land on GPIO 14/15), and keep `console=serial0` out of `cmdline.txt` on both.

| Step | Pi 4 | Pi 5 |
|---|---|---|
| Find the RS-485 tty | `ls -l /sys/class/tty/ttyAMA*/device` → the one on `fe201600.serial` (UART3) | `/dev/ttyAMA0` (UART0); confirm the same way |
| Check pin mux | `pinctrl get 4-7` (or `raspi-gpio get` on old images) | `pinctrl get 14,15,17` |
| CAN up | `ip link set can0 up type can bitrate 250000 restart-ms 100`; `dmesg \| grep mcp251x` shows the detected chip | identical; IRQ arrives via RP1 |
| GPIO in the app | `gpiodetect` → `pinctrl-bcm2711` | `gpiodetect` → `pinctrl-rp1` (gpiochip0 on current kernels) |
| Modbus | libmodbus `modbus_new_rtu(dev, 19200, 'E', 8, 1)`; RS-485 already enabled by the overlay | identical |

Verify: `candump can0` sees traffic and `ip -s link show can0` shows no overruns; on a scope, DE covers the whole frame including the stop bit and releases before the slave answers (3.5-character gap).

## 9. Common failures

| Symptom | Likely cause | Check |
|---|---|---|
| Overlay "does nothing" | Edited `/boot/config.txt` instead of `/boot/firmware/config.txt`; overlay name typo; param for wrong overlay | `sudo vclog --msg \| grep -i dtoverlay`; `vcgencmd get_config str` |
| Code works on Pi 4, GPIO fails on Pi 5 | Hard-coded `gpiochip0/4`, RPi.GPIO, pigpio, `/sys/class/gpio` | `gpiodetect`; port to libgpiod/gpiozero |
| Serial garbage / parity errors on Pi 3/4 | Mini-UART (`ttyS0`) with variable core clock or parity frames | `ls -l /dev/serial0`; use `disable-bt` or an extra PL011 |
| Random characters, login prompt on the app UART | Serial console or getty still on it | `cat /proc/cmdline`; `systemctl status serial-getty@ttyAMA0` |
| `can0` missing | Wrong oscillator/IRQ param, SPI disabled, HAT INT on another GPIO | `dmesg \| grep -i mcp25`; `ls /sys/bus/spi/devices` |
| CAN bus errors only | `oscillator` ≠ crystal; no 120 Ω termination | `ip -det -s link show can0`; read the crystal marking |
| RS-485 last byte cut / no reply | DE from GPIO toggled in userspace, or RTS not muxed | `pinctrl get <rts pin>`; scope DE |
| I2C sensor at 0x50 appears, then HAT misbehaves | Device on I2C-0 / ID pins | Move it to i2c-1 |
| Random reboots, SD corruption, USB drops | Undervoltage or thin cable | `vcgencmd get_throttled` bit 16 |
| Performance drops in the enclosure | Thermal throttling | `vcgencmd measure_temp`; bits 2/18 of `get_throttled` |
| Camera "no cameras available" | Ribbon reversed or wrong FFC width; third-party sensor with `camera_auto_detect=1` | `rpicam-hello --list-cameras`; `dmesg \| grep -i -E 'imx\|ov5'` |
| `pip install` refused | PEP 668 externally managed Python | Use a venv with `--system-site-packages` |
| Watchdog never resets the board | Nobody opened `/dev/watchdog` (`RuntimeWatchdogSec` unset/0), or a second owner keeps petting it | `systemctl show -p RuntimeWatchdogUSec`; `wdctl` |

## 10. Review checklist

- [ ] Board, OS release (`/etc/os-release`), kernel and bootloader version recorded; `config.txt` uses `[pi4]`/`[pi5]` filters if one image serves several models.
- [ ] Every `dtoverlay` checked against the README entry **for that model** (`-pi5` variant pins on Pi 5); no pin claimed twice (`pinctrl get`).
- [ ] No hard-coded gpiochip numbers, `/sys/class/gpio`, RPi.GPIO or pigpio; GPIO via libgpiod/gpiozero by line name.
- [ ] Application UART is a PL011, console and getty removed from it; RS-485 DE handled by the driver (`rs485` param or `TIOCSRS485`) and checked on a scope.
- [ ] CAN `oscillator` and `interrupt` match the HAT schematic; bitrate and `restart-ms` set by the system; MCP2518FD if load is high.
- [ ] I2C-0 / GPIO 0–1 left to the HAT EEPROM; product carrier has its own ID EEPROM if field config must be zero-touch.
- [ ] Power supply sized per model (Pi 5: 5 A) and `get_throttled` logged; thermal margin tested in the closed enclosure at maximum ambient.
- [ ] Storage protected: overlayfs/read-only root or eMMC, volatile logs, power-fail-safe writes (`reliability.md`).
- [ ] Watchdog owned by one process with timeout ≤ 15 s; RTC or time source defined.
- [ ] OTA path chosen (tryboot A/B, Mender, RAUC, SWUpdate) with rollback tested; EEPROM bootloader updates pinned or gated.
- [ ] Irreversible actions (OTP writes, secure-boot keys, eMMC erase, EEPROM flash) only with explicit user confirmation.
