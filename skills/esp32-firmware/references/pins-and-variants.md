# ESP32 variants and safe pin selection

Tables are for the bare chips/common modules. **Always cross-check the module datasheet and the dev-board schematic** — boards also use pins for the USB-UART bridge, LEDs, buttons, displays and PSRAM.

## Contents
1. Variant comparison
2. Pin rules per variant
3. ADC notes
4. Pin selection checklist

---

## 1. Variant comparison

| | ESP32 | ESP32-S2 | ESP32-S3 | ESP32-C3 | ESP32-C6 |
|---|---|---|---|---|---|
| CPU | 2× Xtensa LX6 240 MHz | 1× LX7 240 MHz | 2× LX7 240 MHz | 1× RISC-V 160 MHz | 1× RISC-V 160 MHz (+ LP core) |
| Wireless | Wi-Fi 4, BT Classic + BLE 4.2 | Wi-Fi 4 only | Wi-Fi 4, BLE 5 | Wi-Fi 4, BLE 5 | Wi-Fi 6, BLE 5, 802.15.4 (Thread/Zigbee) |
| Native USB | no | USB OTG | USB OTG + USB-Serial-JTAG | USB-Serial-JTAG | USB-Serial-JTAG |
| PSRAM | quad (WROVER) | quad | quad or octal | no | no |
| FPU | single precision | no | single precision | **no** | **no** |
| GPIOs | 34 (some input-only) | 43 | 45 | 22 | 31 (QFN40) |
| Use for | Bluetooth Classic (audio, SPP), legacy | USB HID, low cost Wi-Fi | Displays/UI, camera, AI, most new designs | Cheap simple Wi-Fi/BLE nodes | Matter/Thread/Zigbee, Wi-Fi 6 |

## 2. Pin rules per variant

### ESP32 (classic)
| Pins | Rule |
|---|---|
| GPIO6–11 | **Never use** — connected to SPI flash |
| GPIO16, 17 | Used by PSRAM on WROVER modules |
| GPIO0 | Strapping: LOW at reset = download mode. Usually the BOOT button. Avoid driving it low at boot |
| GPIO2 | Strapping: must be LOW/floating to enter download mode (often an LED) |
| GPIO5 | Strapping (SDIO timing); usable after boot |
| **GPIO12 (MTDI)** | Strapping: **HIGH at reset selects 1.8 V flash** → modules with 3.3 V flash fail to boot. Never put a pull-up or a peripheral that pulls it high here |
| GPIO15 (MTDO) | Strapping: LOW silences the boot log; usable after boot |
| GPIO34, 35, 36, 39 | **Input only**, no internal pull-up/down |
| GPIO1, 3 | UART0 TX/RX (programming + Serial) |
| GPIO25, 26 | DAC outputs |
| ADC1: 32–39 | Safe with Wi-Fi |
| ADC2: 0, 2, 4, 12–15, 25–27 | **Not usable while Wi-Fi is active** |
| RTC GPIOs (deep-sleep wake ext0/ext1): 0, 2, 4, 12–15, 25–27, 32–39 | |

Good general-purpose choices: I2C on 21 (SDA) / 22 (SCL); SPI (VSPI) 18 SCK / 19 MISO / 23 MOSI / 5 CS; free GPIOs 4, 13, 14, 16–19 (16/17 not on WROVER/PSRAM modules), 21–23, 25–27, 32, 33.

### ESP32-S2
Strapping 0, 45, 46. Flash 26–32 (avoid). USB D−/D+ on 19/20. ADC2 shares with Wi-Fi — prefer ADC1 (GPIO1–10).

### ESP32-S3
| Pins | Rule |
|---|---|
| GPIO0, 3, 45, 46 | Strapping (0 = BOOT button; 45 = VDD_SPI voltage; avoid) |
| GPIO26–32 | SPI flash / quad PSRAM — avoid |
| **GPIO33–37** | **Used by octal PSRAM on R8 modules (e.g. N8R8, N16R8)** — avoid on those |
| GPIO19, 20 | USB D−/D+ (avoid if using native USB) |
| GPIO43, 44 | UART0 TX/RX |
| ADC1: GPIO1–10 | Preferred; ADC2 (11–20) shares with Wi-Fi |
| RTC GPIOs 0–21 | deep-sleep wake capable |
No input-only pins; no DAC. The GPIO matrix lets most peripherals use most pins.

### ESP32-C3
| Pins | Rule |
|---|---|
| GPIO2, 8, 9 | Strapping (9 = BOOT button; 8 must be HIGH for download mode on many boards) |
| GPIO12–17 | SPI flash — avoid |
| GPIO18, 19 | USB D−/D+ |
| GPIO20, 21 | UART0 RX/TX |
| ADC1: GPIO0–4 | Use these; ADC2 (GPIO5) is not recommended (errata) |
| Deep-sleep wake | GPIO0–5 only |

### ESP32-C6
| Pins | Rule |
|---|---|
| GPIO4, 5, 8, 9, 15 | Strapping (9 = BOOT button) |
| GPIO24–30 | SPI flash — avoid |
| GPIO12, 13 | USB D−/D+ |
| GPIO16, 17 | UART0 TX/RX |
| ADC1: GPIO0–6 | Only ADC on C6 |
| LP GPIO 0–7 | Deep-sleep wake, LP core peripherals |

## 3. ADC notes

- ESP32 ADC is noisy and non-linear near the rails. Use the calibrated millivolt API (`analogReadMilliVolts(pin)` in Arduino; `adc_cali_*` in IDF 5) instead of raw counts × 3.3/4095.
- Attenuation sets the range (≈0–950 mV at 0 dB up to ≈0–3.1 V at 11/12 dB); accuracy is best in the middle.
- Average several samples; add a small capacitor at the pin for high-impedance sources.
- For precision measurement, use an external ADC (ADS1115, etc.).

## 4. Pin selection checklist

When assigning or reviewing pins, verify for each pin:
1. Not a flash/PSRAM pin for this exact module.
2. Not a strapping pin — or, if it must be, the external circuit keeps the required level during reset.
3. Not used by the board (USB-UART, LED, button, display, SD card).
4. Direction possible (input-only pins on classic ESP32).
5. ADC pins on ADC1 if Wi-Fi is used.
6. Deep-sleep wake pins are RTC/LP-capable on this variant.
7. 3.3 V logic only — GPIOs are **not 5 V tolerant**; use level shifting.
8. Current: keep each GPIO under ~20 mA (abs. max 40 mA); drive relays/motors through transistors/drivers with flyback diodes.

Present the result as a table: GPIO | function | device | notes/risks.
