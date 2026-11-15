# Build system, west, devicetree and Kconfig

## Contents
1. Workspace, west and the SDK
2. Manifest and topology
3. Board targets (HWMv2)
4. Application layout and configuration files
5. Building, sysbuild, flashing
6. Out-of-tree boards and shields
7. PlatformIO status
8. Devicetree in practice
9. Kconfig in practice
10. Worked example — `__device_dts_ord_N` and "device not ready"
11. Review checklist

---

## 1. Workspace, west and the SDK

```bash
python3 -m venv ~/zephyrproject/.venv && source ~/zephyrproject/.venv/bin/activate
pip install west
west init -m https://github.com/zephyrproject-rtos/zephyr --mr v4.2.0 ~/zephyrproject   # or --mr v3.7.x tag
cd ~/zephyrproject && west update
west zephyr-export                      # registers the Zephyr CMake package
west packages pip --install             # 4.1+; on 3.7/4.0: pip install -r zephyr/scripts/requirements.txt
west sdk install                        # 4.0+; on 3.7 download the SDK bundle and run setup.sh
west blobs fetch hal_espressif          # ESP32 only (Wi-Fi/BLE binary blobs)
```

- **Zephyr SDK versions**: 3.7 pairs with SDK 0.16.x; 4.4 and later require SDK 1.0+ (check `SDK_VERSION` in the zephyr tree for the tested version). Other toolchains via `ZEPHYR_TOOLCHAIN_VARIANT` (`gnuarmemb`, `llvm`, vendor ones); stick to the Zephyr SDK unless there is a reason.
- Host tools: CMake (4.5 needs 3.28+), Ninja, dtc, Python (4.4+ needs 3.12). macOS: Homebrew packages per the Getting Started guide.
- Workspaces are big (all HALs). Trim with `name-allowlist` in your manifest (section 2).

## 2. Manifest and topology

| Topology | What it is | Use when |
|---|---|---|
| T1 (Zephyr is the manifest repo) | `west init` on zephyr itself | Tutorials, samples |
| **T2 (application is the manifest repo)** | Your repo has `west.yml` importing zephyr at a pinned tag | **Products** — one repo pins everything |
| T3 (separate manifest repo) | Manifest-only repo imports zephyr, app and modules | Several products sharing one pinned platform |
| Freestanding app | App folder outside the workspace, `ZEPHYR_BASE` set | Quick experiments; versions not pinned by the app |

```yaml
# west.yml in the application repo (T2)
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v3.7.1              # tag or SHA, never a branch for production
      import:
        name-allowlist:             # only fetch what you use
          - cmsis
          - hal_stm32
          - mcuboot
          - littlefs
  self:
    path: app
```

Then `west init -l app && west update`. Freeze before a release: `west manifest --freeze > frozen.yml`. Module names in the allowlist come from zephyr's own `west.yml` (check there: Cortex-M needs `cmsis_6` instead of `cmsis` from 4.2, Nordic needs `hal_nordic`, ESP32 `hal_espressif`).

## 3. Board targets (HWMv2)

Since 3.7 the board target is `board[@revision][/soc[/cpucluster[/variant]]]`. Board-specific files use the target with `/` replaced by `_`.

| Hardware | Target | Notes |
|---|---|---|
| Nucleo-F429ZI | `nucleo_f429zi` | Single SoC, qualifiers optional |
| Nucleo-L552ZE-Q TF-M | `nucleo_l552ze_q/stm32l552xx/ns` | Non-secure app, TF-M built alongside |
| B-U585I-IOT02A TF-M | `b_u585i_iot02a/stm32u585xx/ns` | |
| ESP32-DevKitC | `esp32_devkitc/esp32/procpu` | 4.2+; on 3.7 `esp32_devkitc_wroom/esp32/procpu` |
| ESP32-S3-DevKitC | `esp32s3_devkitc/esp32s3/procpu` | `appcpu` = second core image |
| nRF52840 DK | `nrf52840dk/nrf52840` | |
| nRF5340 DK app core | `nrf5340dk/nrf5340/cpuapp` | `cpuapp/ns` = with TF-M; `cpunet` = network core |
| nRF54L15 DK | `nrf54l15dk/nrf54l15/cpuapp` | 4.0+; on 3.7 it was `nrf54l15pdk/...` |
| Host | `native_sim`, `native_sim/native/64` | Linux host only (not macOS) |

List with `west boards | grep nucleo`. File names: `boards/nrf5340dk_nrf5340_cpuapp.overlay`, `boards/esp32s3_devkitc_esp32s3_procpu.conf`.

## 4. Application layout and configuration files

```
app/
├── CMakeLists.txt
├── prj.conf                     # Kconfig fragment, all boards
├── app.overlay                  # DT overlay, all boards (ignored if boards/<target>.overlay exists)
├── boards/
│   ├── nucleo_f429zi.conf       # extra Kconfig for this target (merged)
│   └── nucleo_f429zi.overlay    # DT overlay for this target
├── sysbuild.conf                # SB_CONFIG_* (MCUboot etc.)
├── sysbuild/mcuboot.conf        # Kconfig for the MCUboot image
├── VERSION                      # app version, used for image signing
├── Kconfig                      # optional app-specific symbols (must source "Kconfig.zephyr")
└── src/main.c
```

```cmake
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(locker_ctrl)
target_sources(app PRIVATE src/main.c src/modbus_srv.c)
```

Variants without copying files: `-DEXTRA_CONF_FILE=debug.conf`, `-DEXTRA_DTC_OVERLAY_FILE=rs485.overlay`, `-DFILE_SUFFIX=release` (picks `prj_release.conf`, `app_release.overlay` when present), snippets `-S <name>` (e.g. `-S rtt-console` where the board supports it — check `west snippets`/docs). `OVERLAY_CONFIG` is the old name of `EXTRA_CONF_FILE`.

## 5. Building, sysbuild, flashing

```bash
west build -b nucleo_f429zi -p always app            # -p always = pristine
west build -t menuconfig                              # or guiconfig
west build -b nrf5340dk/nrf5340/cpuapp --sysbuild app # MCUboot / multi-image
west config build.sysbuild True                       # make sysbuild the default
west flash                                            # default runner of the board
west flash -r jlink   |  -r openocd  |  -r pyocd  |  -r stm32cubeprogrammer  |  -r nrfutil
west debug  /  west debugserver  /  west attach
west build -d build_dbg -b ... -- -DEXTRA_CONF_FILE=debug.conf   # second build dir
```

- Sysbuild builds several images (app, MCUboot, net core, TF-M) into `build/<image>/`. `SB_CONFIG_*` symbols go in `sysbuild.conf`; per-image Kconfig in `sysbuild/<image>.conf`, overlays in `sysbuild/<image>.overlay`. Pass a CMake variable to one image as `-D<image>_CONFIG_FOO=y`.
- The default build dir is `build/`; with sysbuild the app ELF is `build/<app>/zephyr/zephyr.elf`.
- 4.5 removed the CMake `flash`/`debug` targets (`ninja flash`); use `west flash`/`west debug`.

## 6. Out-of-tree boards and shields

Custom carrier boards belong in your repo, not in a zephyr fork:

```
app/boards/acme/locker_ctrl/
├── board.yml                    # name, full_name, vendor, socs (HWMv2)
├── Kconfig.locker_ctrl          # select SOC_STM32G474XX (or the right SOC_*)
├── locker_ctrl_defconfig        # CONFIG_* defaults
├── locker_ctrl.dts              # includes the SoC dtsi + pinctrl dtsi
└── board.cmake                  # runners (openocd, jlink, stm32cubeprogrammer)
```

Build with `-DBOARD_ROOT=$PWD/app` or set `build: settings: board_root: .` in the repo's `zephyr/module.yml` (the module route is the reliable one with sysbuild, where every image must find the board). Start by copying the closest in-tree board and editing. Add-on PCBs (RS-485 board, display) are **shields**: `boards/shields/<name>/` with `<name>.overlay`, `Kconfig.shield`, `Kconfig.defconfig` (and `shield.yml` on newer versions); use with `--shield <name>`. 3.6 and older used HWMv1 (`board.yml` absent); convert with the board porting guide.

## 7. PlatformIO status

PlatformIO offers `framework = zephyr` on several platforms, but it is maintained separately, **lags upstream by one or more releases** (the package version encodes the Zephyr version: `framework-zephyr` 3.40402.0 = Zephyr 4.4.2, the newest in the registry in November 2026 while upstream was at 4.5.0), does not support sysbuild or the full west module set, and board names follow PlatformIO IDs. Check the shipped version with `pio pkg list` (`framework-zephyr`). Recommendation: use west (plain CLI or VS Code with the nRF Connect / Zephyr extensions) for products; PlatformIO is acceptable for a quick evaluation only.

## 8. Devicetree in practice

```dts
/* boards/nucleo_f429zi.overlay */
/ {
    aliases { lockdrv0 = &lock0; };
    chosen  { zephyr,console = &usart3; };
    outputs {
        compatible = "gpio-leds";             /* any binding with a gpios property */
        lock0: lock_0 { gpios = <&gpioe 9 GPIO_ACTIVE_HIGH>; };
    };
    zephyr,user {                              /* app-only properties, no binding needed */
        door-gpios = <&gpiof 13 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
        io-channels = <&adc1 3>;
    };
};
&i2c1 { status = "okay"; clock-frequency = <I2C_BITRATE_FAST>; };
&spi1 { status = "disabled"; };
```

| Need | Macro / API |
|---|---|
| Node by label / alias / chosen / path | `DT_NODELABEL(lock0)`, `DT_ALIAS(lockdrv0)`, `DT_CHOSEN(zephyr_console)`, `DT_PATH(zephyr_user)` |
| Property | `DT_PROP(node, prop)`, `DT_PROP_OR(node, prop, def)`, `DT_REG_ADDR(node)` |
| Enabled? | `DT_NODE_HAS_STATUS_OKAY(node)` (4.0+), `DT_NODE_HAS_STATUS(node, okay)` (3.7) |
| Device pointer | `DEVICE_DT_GET(node)`, `DEVICE_DT_GET_ONE(compat)`, then `device_is_ready(dev)` |
| GPIO spec | `GPIO_DT_SPEC_GET(node, gpios)`, `GPIO_DT_SPEC_GET_OR(...)`, `gpio_is_ready_dt()` |
| Other specs | `I2C_DT_SPEC_GET`, `SPI_DT_SPEC_GET(node, op)` (3.7–4.2 need a third `delay` arg; optional in 4.3–4.4; removed in 4.5), `ADC_DT_SPEC_GET_BY_IDX`, `PWM_DT_SPEC_GET` |

- Names convert to C tokens: lowercase, `-` and `,` become `_` (`zephyr,user` → `zephyr_user`, `door-gpios` → `door_gpios`).
- **Bindings** (`dts/bindings/**.yaml`) define allowed properties per `compatible`; your own go in `app/dts/bindings/`. A property not in the binding generates no macro.
- **status**: missing means okay; most SoC peripherals are `disabled` in the SoC dtsi and enabled by the board or your overlay.
- **pinctrl**: pins via `pinctrl-0 = <&usart6_tx_pg14 &usart6_rx_pg9>; pinctrl-names = "default";` using labels from the vendor pinctrl dtsi (STM32: `hal_stm32` generated files; Nordic: `&pinctrl { ... psels ... }` groups in the board `-pinctrl.dtsi`; ESP32: `esp32s3-pinctrl.h` macros). Never configure pin muxing in C.
- Outputs: `build/zephyr/zephyr.dts` (final tree), `build/zephyr/include/generated/zephyr/devicetree_generated.h` (macros + ordinal list). Use `-DZEPHYR_SCA_VARIANT=dtdoctor` (4.3+) for hints on DT errors.

## 9. Kconfig in practice

- `prj.conf` lines are `CONFIG_FOO=y` with no spaces and no quotes for bools/ints; strings use quotes.
- Final values: `build/zephyr/.config`; search with `grep CONFIG_UART_ASYNC_API build/zephyr/.config`.
- Assigning an **undefined** symbol aborts the build ("Aborting due to Kconfig warnings"): typo or the module that defines it is not in the workspace.
- "**was assigned the value 'y' but got the value 'n'**" is only a warning: a `depends on` is unmet. In menuconfig (`/` to search) the symbol shows its dependencies and what selects it. Typical: `CONFIG_UART_ASYNC_API` needs a driver with async support (and often DMA in DT); `CONFIG_MODBUS_SERIAL` needs `CONFIG_SERIAL`.
- Board `_defconfig` and DT also set defaults (`default y if DT_HAS_<COMPAT>_ENABLED`): enabling a node often enables its driver automatically, and disabling the node silently disables the driver.
- `west build -t hardenconfig` lists security-relevant deviations.

## 10. Worked example — `__device_dts_ord_N` and "device not ready"

Symptom A (link time): `undefined reference to '__device_dts_ord_42'`. A `DEVICE_DT_GET()` refers to a node for which no driver created a `struct device`.

1. Open `build/zephyr/include/generated/zephyr/devicetree_generated.h`; the header comment lists ordinals: `42  /soc/serial@40011400`.
2. `grep -A6 "serial@40011400" build/zephyr/zephyr.dts`: `status = "disabled"` → enable in the overlay.
3. Status okay but still failing → the driver is off: check `.config` for the driver symbol (`CONFIG_SERIAL`, `CONFIG_I2C`, `CONFIG_GPIO` — on 4.5 STM32 boards `CONFIG_GPIO` is no longer default-on).
4. Driver symbol `y` but node has a `compatible` with no driver in this version → wrong compatible or unsupported peripheral; search `drivers/` for the compatible string (`DT_DRV_COMPAT st_stm32_usart`).
5. Pristine rebuild; confirm the ordinal disappeared.

Symptom B (run time): `device_is_ready()` returns false. The device exists but its init function failed: missing clock or pinctrl, wrong `reg`, a bus device whose parent bus is not ready, an I2C sensor not answering (wrong address, no pull-ups), or init priority ordering. Enable `CONFIG_LOG=y` with the driver's log level at debug (`CONFIG_<SUBSYS>_LOG_LEVEL_DBG=y`, check the exact name in menuconfig) and read the boot log. The shell command `device list` shows which devices are ready.

## 11. Review checklist

- [ ] Manifest pins zephyr and every module to a tag or SHA; `name-allowlist` trims HALs; frozen manifest archived per release.
- [ ] Board target written with full HWMv2 qualifiers; board-specific files named accordingly.
- [ ] No pins, baud rates or addresses in C; all in overlays; overlays use labels (`&usart6`).
- [ ] Every device obtained with `DEVICE_DT_GET`/`*_dt_spec` and checked with `*_is_ready*` before use.
- [ ] `.config` checked for every symbol set in `prj.conf`; no "assigned but got" warnings left.
- [ ] Custom board lives in the app repo (`BOARD_ROOT`), derived from an in-tree board.
- [ ] Build reproducible from a clean checkout with `west update` + `west build -p always`.
