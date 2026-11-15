# Boot chain, U-Boot and device tree

## Contents
1. Boot chain on i.MX8M (and generic ARM)
2. U-Boot console and environment
3. Recovery when the board does not boot
4. Device tree essentials
5. i.MX8M pin muxing (iomuxc)
6. Overlays — Torizon / TorizonCore Builder
7. Overlays — generic (dtbo + U-Boot, extlinux, configfs)
8. Debugging a device that does not probe
9. Worked example — I2C sensor and a GPIO on a Verdin carrier
10. Review checklist

---

## 1. Boot chain on i.MX8M (and generic ARM)

```
Boot ROM (in SoC, reads boot fuses/pins)
  └─ SPL (U-Boot first stage, runs from on-chip RAM, trains DDR with the LPDDR4 firmware)
      └─ ARM Trusted Firmware (bl31, EL3) [+ optional OP-TEE]
          └─ U-Boot proper (env, boot script, loads kernel + DTB + overlays)
              └─ Linux kernel (unpacks, parses DTB, probes drivers)
                  └─ initramfs (optional; Torizon uses one to mount the OSTree deployment)
                      └─ init = systemd (or BusyBox init on small Buildroot images)
```

- On i.MX8M, SPL + DDR firmware + ATF + U-Boot are packed into one `imx-boot` / `flash.bin` image (imx-mkimage). On a Verdin module it lives in the eMMC boot partition (`/dev/mmcblk0boot0`), not in the user area.
- Each stage prints to the debug UART. Where the log stops tells you the failing stage: nothing at all → boot ROM / boot media / fuses; SPL banner but no "DDR" success → DDR config; U-Boot prompt but "Starting kernel ..." is the last line → wrong DTB, wrong `console=` or a kernel that panics before the console driver (add `earlycon` to the kernel command line).
- Generic boards (Raspberry Pi, Rockchip, TI AM62x) follow the same idea with different first stages; the reasoning about "last line printed" is the same.

## 2. U-Boot console and environment

Connect to the debug UART (Verdin: UART_3, on the Verdin Development Board via the on-board USB-serial; usually 115200 8N1). Press a key during the autoboot countdown.

```
printenv                      # all variables
printenv fdtfile bootcmd      # which DTB, what runs at boot
setenv bootdelay 3            # change (RAM only)
saveenv                       # persist — ask the user first
env default -a; saveenv       # back to defaults — destructive, confirm
run bootcmd                   # continue booting
mmc list; mmc part            # storage
ls mmc 0:1 /                  # files on a partition
fdt addr ${fdt_addr_r}; fdt print /soc   # inspect a loaded DTB
```

From Linux, with `libubootenv` / `u-boot-fw-utils` and a correct `/etc/fw_env.config`:
```
fw_printenv
fw_setenv bootdelay 3         # writes the env partition — confirm first
```
Kernel command line: `cat /proc/cmdline`. On Torizon, extra arguments are set with TorizonCore Builder (`kernel set_custom_args`, or `customization: kernel: arguments:` in `tcbuild.yaml`) or the Toradex `tdxargs` U-Boot variable, not by editing `bootargs` by hand, because the boot script rebuilds it. The mechanism changed in later Torizon OS 7.x releases (uEnv.txt) and is restricted on hardened/secure-boot images — check the docs for the exact version.

## 3. Recovery when the board does not boot

| Situation | Path |
|---|---|
| U-Boot works, Linux does not | Interrupt U-Boot, boot an older OSTree deployment (Torizon rolls back automatically after failed boots), or fix `fdtfile` / overlays |
| Bad overlay on Torizon | In U-Boot `setenv skip_fdt_overlays 1; boot` to boot without overlays, then fix the overlay list and redeploy a corrected image; boot-count rollback only applies while an OS update is pending; last resort, reinstall with Toradex Easy Installer |
| Boot container (SPL/U-Boot) broken | Serial download mode: put the module in recovery (Verdin dev board has a recovery button / jumper), connect USB OTG, run NXP `uuu` or the Toradex recovery script to load Toradex Easy Installer, then reinstall |
| Generic i.MX board | `uuu -b emmc_all flash.bin image.wic` from serial download mode |
| SD-card boards | Reflash the card; keep a known-good card |

Reinstalling wipes the device. Confirm with the user and back up data partitions first.

## 4. Device tree essentials

```dts
/dts-v1/;
#include "imx8mm-verdin.dtsi"               /* SoC + module description */
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/interrupt-controller/irq.h>

/ {
    model = "My carrier with Verdin iMX8M Mini";
    compatible = "acme,mycarrier", "toradex,verdin-imx8mm", "fsl,imx8mm";
};

&i2c4 {                                      /* reference by label, then extend */
    status = "okay";
    temp@48 {
        compatible = "ti,tmp102";            /* binding: Documentation/devicetree/bindings */
        reg = <0x48>;                        /* 7-bit address */
        interrupt-parent = <&gpio4>;
        interrupts = <3 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

| Property | Meaning / trap |
|---|---|
| `compatible` | Most specific first. Must match an `of_device_id` string in a driver that is built |
| `reg` | Address/size cells come from the **parent's** `#address-cells` / `#size-cells`; on I2C it is the 7-bit address |
| `status` | `"okay"` or `"disabled"`. Many SoC nodes are disabled in the `.dtsi` and must be enabled |
| `interrupts` / `interrupt-parent` | Cell format defined by the parent controller (GPIO controllers: line, flags) |
| `*-gpios` | `<&gpio4 3 GPIO_ACTIVE_LOW>` — the active level is in DT, so driver code uses logical values |
| `pinctrl-names` / `pinctrl-0` | Which iomuxc group to apply when the device probes |
| `clocks`, `*-supply`, `resets` | Suppliers; if missing or not ready the consumer defers probe |
| `label:` and `&label` | Phandle references. Overlays need the base DTB built with symbols (`dtc -@`) |

Read the binding YAML in the kernel tree (`Documentation/devicetree/bindings/...`) of the **same kernel version** the board runs; properties change between versions. Validate with `make dt_binding_check` / `make dtbs_check` when building the kernel.

Inspect the live tree:
```
dtc -I fs -O dts /sys/firmware/devicetree/base > live.dts 2>/dev/null
ls /proc/device-tree/soc@0/                     # /proc/device-tree → /sys/firmware/devicetree/base
cat /proc/device-tree/soc@0/bus@30800000/i2c@30a50000/status
```

## 5. i.MX8M pin muxing (iomuxc)

Each pad has one mux setting. A group in the iomuxc node lists pad macro + pad-control value:

```dts
&iomuxc {
    pinctrl_my_gpio: mygpiogrp {
        fsl,pins = <
            MX8MM_IOMUXC_SAI1_RXD1_GPIO4_IO3   0x184   /* illustrative; value = SW_PAD_CTL bits, copy from Toradex .dtsi */
        >;
    };
};
```

- The macro name encodes pad and function (`<PAD>_<FUNCTION>`); macros are in `imx8mm-pinfunc.h` / `imx8mp-pinfunc.h`. Wrong-SoC macros compile but mux the wrong registers.
- The second number sets drive strength, slew, pull-up/down, hysteresis, open drain. The bit layout differs between i.MX8M Mini and Plus — copy a value from the Toradex `.dtsi` for a pad used the same way, and check the `SW_PAD_CTL_PAD_*` register in the SoC reference manual.
- Map carrier signal → SoM pin (Verdin SODIMM number) → SoC pad from the Toradex pinout designer or datasheet. Toradex `.dtsi` files already define groups per SODIMM pin; reuse them.
- See who owns a pad at runtime:
```
cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pinmux-pins | grep -i sai1_rxd1
cat /sys/kernel/debug/gpio                       # requested lines and consumers
```
"pin ... already requested by ..." in `dmesg` means two groups claim the pad — disable or re-mux the other node.

## 6. Overlays — Torizon / TorizonCore Builder

On Torizon OS the base DTBs and overlays live in the OSTree deployment (`/sysroot/boot/ostree/torizon-<hash>/dtb/`), and `overlays.txt` there lists the active ones (`fdt_overlays=a.dtbo b.dtbo`). Editing that file on the device works for a quick test but is unversioned and replaced by the next update; for anything real, build a custom image with TorizonCore Builder on the PC.

Overlay source (start from Toradex `device-tree-overlays` repo examples on the branch matching your Torizon version):
```dts
/dts-v1/;
/plugin/;
#include <dt-bindings/gpio/gpio.h>
#include "imx8mm-pinfunc.h"

/ {
    compatible = "toradex,verdin-imx8mm";    /* overlay is applied only on matching boards */
};

&i2c4 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;
    temp@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};
```

Workflow on the PC (TorizonCore Builder runs in a container; set up its alias per Toradex docs):
```
torizoncore-builder images download --remote-host <board-ip> --remote-username torizon --remote-password ...
# sources on the branch for your Torizon version (e.g. toradex_6.6-2.2.x-imx); `dt checkout` is deprecated
git clone -b <branch> git://git.toradex.com/linux-toradex.git linux
git clone -b <branch> git://git.toradex.com/device-tree-overlays.git device-trees
torizoncore-builder dto list --device-tree linux/arch/arm64/boot/dts/freescale/imx8mm-verdin-wifi-dev.dts
torizoncore-builder dto apply my-overlay.dts        # quick test: compiles and applies to the unpacked image
torizoncore-builder union custom-branch             # commit the changes to an OSTree branch
torizoncore-builder deploy custom-branch --remote-host <ip> --remote-username torizon --remote-password ... --reboot
```
For anything beyond a test, put it in `tcbuild.yaml` and `torizoncore-builder build`:
```yaml
customization:
  device-tree:
    include-dirs:
      - linux/include
    custom: linux/arch/arm64/boot/dts/freescale/imx8mm-verdin-wifi-dev.dts   # base DTB source
    overlays:
      add:
        - my-overlay.dts
```
Commit `tcbuild.yaml` and the `.dts` files; the image is reproducible from them. Deploying reboots the board and changes its OS — confirm with the user.

## 7. Overlays — generic (dtbo + U-Boot, extlinux, configfs)

Compile (the C preprocessor is needed for `#include` and macros):
```
cpp -nostdinc -undef -x assembler-with-cpp -I $KSRC/include -I $KSRC/arch/arm64/boot/dts/freescale \
    my.dts -o my.pre.dts
dtc -@ -I dts -O dtb -o my.dtbo my.pre.dts
```
The **base** DTB must be compiled with `-@` (`DTC_FLAGS += -@` or `make dtbs DTC_FLAGS=-@`), otherwise label references cannot be resolved ("FDT_ERR_NOTFOUND").

Apply in U-Boot:
```
load mmc 0:1 ${fdt_addr_r} board.dtb
fdt addr ${fdt_addr_r}; fdt resize 8192
load mmc 0:1 ${loadaddr} my.dtbo
fdt apply ${loadaddr}
booti ${kernel_addr_r} - ${fdt_addr_r}
```
Distro boot with `extlinux.conf`: `fdtoverlays /dtbs/overlays/my.dtbo` under the label. Raspberry Pi: `dtoverlay=` in `config.txt`. Runtime overlays via configfs exist only with out-of-tree patches on most kernels; do not plan a product on them.

Check after boot that the overlay took effect: look for its nodes in `/proc/device-tree`.

## 8. Debugging a device that does not probe

Work down this list; stop at the first "no".

| Check | Command | If not |
|---|---|---|
| Node present and enabled in the live DT | `ls /proc/device-tree/...`; `cat .../status` | Overlay not applied, wrong label, wrong `compatible` at overlay root |
| Driver built | `zcat /proc/config.gz \| grep TMP102`; `modinfo tmp102` | Enable in kernel config / install module package |
| Module loaded | `lsmod`; `modprobe tmp102` | `MODULE_DEVICE_TABLE(of, ...)` missing → no autoload |
| Device created on the bus | `ls /sys/bus/i2c/devices/`; `ls /sys/bus/platform/devices/` | Parent bus disabled, wrong `reg`, parent missing `#address-cells` |
| Driver bound | `ls -l /sys/bus/i2c/devices/3-0048/driver` | `compatible` mismatch; read `dmesg` for probe error |
| Deferred | `cat /sys/kernel/debug/devices_deferred` | Supplier (clock, regulator, GPIO, PHY) missing or itself not probed |
| Pins | `pinmux-pins`, `dmesg \| grep -i pin` | Conflict or missing `pinctrl-0` |
| Probe error code | `dmesg \| grep -i 3-0048` | `-121` (EREMOTEIO) = NACK → address/wiring/power; `-110` timeout; `-517` = EPROBE_DEFER |

Useful extras: `echo 'module tmp102 +p' > /sys/kernel/debug/dynamic_debug/control` before `modprobe`; `/sys/kernel/debug/clk/clk_summary`; `/sys/kernel/debug/regulator/regulator_summary`. Manual bind/unbind for testing: `echo 3-0048 > /sys/bus/i2c/drivers/tmp102/unbind` then `bind`.

## 9. Worked example — I2C sensor and a GPIO on a Verdin carrier

Goal: TMP102 on Verdin I2C_1 at 0x48, and an output GPIO on a free SODIMM pin.

1. Confirm the bus from Linux before writing DT: `ls -l /dev/verdin-i2c*` (Torizon creates Verdin-named symlinks), then `i2cdetect -y -r <N>` — expect `48`.
2. Write the overlay with the `&i2c4` node above (check in the Toradex `.dtsi` which controller label Verdin I2C_1 uses on your module).
3. For the GPIO, do **not** add a DT node — use libgpiod by line name: `gpioinfo | grep SODIMM_` lists named lines; only add a pinctrl group if the pad is muxed to another function.
4. Apply with `dto apply`, `union`, `deploy <branch> ... --reboot` (or `tcbuild.yaml` + `build`).
5. Verify: `i2cdetect` now shows `UU` at 0x48 (driver owns it); `cat /sys/class/hwmon/hwmon*/name` contains `tmp102`; `cat /sys/class/hwmon/hwmonX/temp1_input` gives milli-°C.
6. The app reads hwmon sysfs — no custom driver, no raw I2C.

## 10. Review checklist

- [ ] Overlay has `/plugin/;`, a root `compatible` matching the board, and uses labels (`&i2c4`), not full paths.
- [ ] Every new node has `status = "okay"`, correct `reg` for the parent's cells, and a binding-valid `compatible`.
- [ ] Pads are muxed once; conflicting nodes disabled; pad-control values copied from a known-good reference for that SoC.
- [ ] GPIO and IRQ polarity expressed in DT flags, not inverted in code.
- [ ] Suppliers (`vdd-supply`, clocks) declared so probe order is correct.
- [ ] Base DTB built with symbols; overlay builds against the same kernel tree version.
- [ ] Change lives in `tcbuild.yaml` / Yocto layer / Buildroot overlay, committed, not hand-edited on the target.
- [ ] Recovery path known (serial console, previous deployment, uuu / Easy Installer) before deploying.
