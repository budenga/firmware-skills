# Kernel drivers and modules

## Contents
1. Do you need a driver at all?
2. Existing subsystems and their userspace view
3. Out-of-tree modules — Kbuild and the right headers
4. Building modules on Torizon, Yocto and Buildroot
5. Driver skeleton — I2C client with regmap
6. SPI and platform drivers
7. Exposing data — IIO, sysfs attributes, misc device
8. Concurrency, IRQs and resource management
9. Worked example — interrupt-driven I2C device to IIO
10. Review checklist

---

## 1. Do you need a driver at all?

| Situation | Use |
|---|---|
| Chip has an upstream driver | DT node only (search `git grep "vendor,part" Documentation/devicetree/bindings drivers/`) |
| Simple GPIO-level device (button, LED, buzzer, relay, fan) | Generic drivers: `gpio-keys`, `leds-gpio`, `gpio-beeper`, `pwm-fan`, `gpio-restart` — DT only |
| Occasional register reads, no IRQ latency need | Userspace via i2c-dev / spidev (`userspace-hw-access.md`) |
| Needs IRQ with < ~1 ms reaction, DMA, power management, or must be shared by several processes | Kernel driver |
| Must integrate with an existing framework (input, hwmon, IIO, net, RTC, watchdog) | Kernel driver in that framework, never a custom char device |

A similar chip in the same family often needs only a new `compatible` + chip-info entry in an existing driver.

## 2. Existing subsystems and their userspace view

| Subsystem | Typical devices | Userspace |
|---|---|---|
| GPIO | expanders (PCA953x, MCP23xxx) | `/dev/gpiochipN` (libgpiod) |
| hwmon | temperature, voltage, fan, current monitors (TMP102, INA219, LM75) | `/sys/class/hwmon/hwmonN/*_input` |
| IIO | ADCs, DACs, IMUs, light, pressure | `/sys/bus/iio/devices/iio:deviceN/in_*_raw`, `_scale`; buffered via `/dev/iio:deviceN` |
| input | buttons, touch, keypads, rotary encoders | `/dev/input/eventN` (evdev) |
| LED | indicator LEDs, LED drivers (PCA963x) | `/sys/class/leds/<name>/brightness`, `trigger` |
| PWM | PWM controllers | `/sys/class/pwm/pwmchipN/export` (or consumer drivers like `pwm-backlight`) |
| RTC | DS1307, PCF85063, RV-3028 | `/dev/rtc0`, `hwclock` |
| regulator | PMIC outputs, fixed/gpio regulators | consumers in DT; `regulator_summary` in debugfs |
| watchdog | SoC and external watchdogs | `/dev/watchdogN` |
| net/CAN | MCP251xFD, FlexCAN | SocketCAN `canN` |
| tty/serial | UARTs, SPI-UART bridges (SC16IS7xx) | `/dev/ttyXXX` |

## 3. Out-of-tree modules — Kbuild and the right headers

```make
# Kbuild (or Makefile)
obj-m := mydrv.o
mydrv-y := core.o i2c.o           # multi-file module
ccflags-y := -Wall
```

```make
# Makefile wrapper
KDIR ?= /lib/modules/$(shell uname -r)/build
all:
	$(MAKE) -C $(KDIR) M=$(CURDIR) modules
clean:
	$(MAKE) -C $(KDIR) M=$(CURDIR) clean
```

Cross build against a prepared kernel tree:
```
make -C $KSRC ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- <board>_defconfig   # or the exact .config from the target
make -C $KSRC ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_prepare     # or a full build to get Module.symvers
make -C $KSRC ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- M=$PWD modules
```

The module loads only when it matches the running kernel:
```
uname -r                                  # on target
modinfo -F vermagic mydrv.ko              # must match, incl. SMP preempt mod_unload aarch64
zcat /proc/config.gz > target.config      # use this, not a guessed defconfig
```

| Error at `insmod` | Cause |
|---|---|
| `Invalid module format` + `version magic` in dmesg | Different kernel release string or config |
| `disagrees about version of symbol` | `Module.symvers` from another build (CONFIG_MODVERSIONS) |
| `Unknown symbol` | Depends on a module not loaded (`modprobe` resolves via `depmod`), or symbol is `EXPORT_SYMBOL_GPL` and the module lacks `MODULE_LICENSE("GPL")` |
| `Key was rejected by service` / `module verification failed` | Module signing enforced; sign with the kernel's key or build in-tree |

Install for autoload: copy to `/lib/modules/$(uname -r)/updates/`, run `depmod -a`; DT-based autoload needs `MODULE_DEVICE_TABLE(of, ...)`.

## 4. Building modules on Torizon, Yocto and Buildroot

- **Torizon OS**: `/usr` is read-only and the kernel comes with the OS image. Build with TorizonCore Builder so the module is part of the image:
  ```
  torizoncore-builder kernel build_module ./mydrv --autoload
  ```
  or in `tcbuild.yaml` under `customization: kernel: modules: - source-dir: mydrv/ autoload: yes`. Kernel arguments: `kernel set_custom_args "foo=1"`.
- **Yocto**: write a recipe that `inherit module`; the layer's kernel is used automatically and the module ends up in the image (`KERNEL_MODULE_AUTOLOAD += "mydrv"`). To build from an SDK, the SDK needs `kernel-devsrc` and a `make scripts prepare` in `$SDKTARGETSYSROOT/usr/src/kernel` once.
- **Buildroot**: package with `$(eval $(kernel-module))` in a `.mk`, or put the driver in an external tree (`BR2_EXTERNAL`).
- Prefer in-tree (patch to the vendor kernel) when the driver is long-lived: no vermagic drift on kernel updates.

## 5. Driver skeleton — I2C client with regmap

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/i2c.h>
#include <linux/module.h>
#include <linux/regmap.h>
#include <linux/gpio/consumer.h>

#define REG_ID   0x0F
#define ID_VALUE 0x6A

struct mydev {
	struct regmap *map;
	struct gpio_desc *reset;
};

static const struct regmap_config mydev_regmap = {
	.reg_bits = 8,
	.val_bits = 8,
	.max_register = 0x7F,
};

static int mydev_probe(struct i2c_client *client)
{
	struct device *dev = &client->dev;
	struct mydev *md;
	unsigned int id;
	int ret;

	md = devm_kzalloc(dev, sizeof(*md), GFP_KERNEL);
	if (!md)
		return -ENOMEM;

	md->map = devm_regmap_init_i2c(client, &mydev_regmap);
	if (IS_ERR(md->map))
		return dev_err_probe(dev, PTR_ERR(md->map), "regmap\n");

	md->reset = devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_HIGH);  /* "reset-gpios" in DT */
	if (IS_ERR(md->reset))
		return dev_err_probe(dev, PTR_ERR(md->reset), "reset gpio\n");
	if (md->reset) {
		usleep_range(1000, 2000);
		gpiod_set_value_cansleep(md->reset, 0);   /* logical: deassert, polarity from DT */
	}

	ret = regmap_read(md->map, REG_ID, &id);
	if (ret)
		return dev_err_probe(dev, ret, "read ID\n");
	if (id != ID_VALUE)
		return dev_err_probe(dev, -ENODEV, "unexpected ID 0x%02x\n", id);

	i2c_set_clientdata(client, md);
	return 0;
}

static const struct of_device_id mydev_of_match[] = {
	{ .compatible = "acme,mydev" },
	{ }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

static const struct i2c_device_id mydev_id[] = {
	{ "mydev" },
	{ }
};
MODULE_DEVICE_TABLE(i2c, mydev_id);

static struct i2c_driver mydev_driver = {
	.driver = {
		.name = "mydev",
		.of_match_table = mydev_of_match,
	},
	.probe = mydev_probe,           /* single-argument probe since kernel 6.3; older: probe_new */
	.id_table = mydev_id,
};
module_i2c_driver(mydev_driver);

MODULE_DESCRIPTION("ACME mydev");
MODULE_LICENSE("GPL");
```

- `dev_err_probe` prints the error but stays quiet for `-EPROBE_DEFER` and records the reason in `devices_deferred`.
- `devm_*` resources are released automatically on probe failure and unbind — no `remove` needed for most drivers.
- Check the kernel version for API differences (`probe` signature, `remove` returning `void` since 6.x). Write for the version the board runs.

## 6. SPI and platform drivers

SPI: `devm_regmap_init_spi(spi, &cfg)`, or `spi_write_then_read()` for small transfers. Mode and speed come from DT (`spi-max-frequency`, `spi-cpol`, `spi-cpha`), not hard-coded. Buffers passed to `spi_sync` must be DMA-safe (kmalloc'd, not on the stack).

Platform (memory-mapped block in an FPGA or SoC):
```c
static int myip_probe(struct platform_device *pdev)
{
	void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
	int irq;

	if (IS_ERR(base))
		return PTR_ERR(base);
	irq = platform_get_irq(pdev, 0);
	if (irq < 0)
		return irq;
	/* clocks: devm_clk_get_enabled(&pdev->dev, NULL); */
	return 0;
}
```
Use `readl`/`writel` (or regmap-mmio), never plain pointer dereferences on `__iomem`.

## 7. Exposing data — IIO, sysfs attributes, misc device

Order of preference: existing subsystem (IIO/hwmon/input) → sysfs attributes on the device (`DEVICE_ATTR_RO`, one value per file, use `attribute_groups` in `.dev_groups`) → misc device for streams or commands.

Minimal misc device (gets `/dev/mydev` with a dynamic minor, created by udev):
```c
static ssize_t mydev_read(struct file *f, char __user *buf, size_t len, loff_t *off)
{
	u8 tmp[16];
	size_t n = min(len, sizeof(tmp));
	/* fill tmp under a lock */
	if (copy_to_user(buf, tmp, n))
		return -EFAULT;
	return n;
}
static const struct file_operations mydev_fops = {
	.owner = THIS_MODULE,
	.read  = mydev_read,
	/* .poll for select/epoll, .unlocked_ioctl for commands */
};
static struct miscdevice mydev_misc = {
	.minor = MISC_DYNAMIC_MINOR,
	.name  = "mydev",
	.fops  = &mydev_fops,
};
/* in probe: misc_register(&mydev_misc); in remove: misc_deregister(&mydev_misc); */
```
Never dereference a user pointer directly; always `copy_from_user` / `copy_to_user` and validate lengths. ioctl numbers via `_IOR/_IOW` with a unique magic, and structs with fixed-width types (32-bit userspace on 64-bit kernels needs `compat_ioctl`).

## 8. Concurrency, IRQs and resource management

- I2C/SPI access sleeps → never from hard IRQ context. Use `devm_request_threaded_irq(dev, irq, NULL, handler, IRQF_ONESHOT, ...)`; the threaded handler may sleep.
- Protect shared state with a `mutex` in process context, `spinlock` only if touched from hard IRQ.
- Long work → `workqueue`; periodic polling → `delayed_work`, not busy loops.
- Power: `pm_runtime_*` if the chip has a sleep mode; `suspend/resume` via `DEFINE_SIMPLE_DEV_PM_OPS`.
- Logging: `dev_dbg` (enabled with dynamic debug) for detail, `dev_err` for failures, `dev_*_ratelimited` in paths that can repeat.

## 9. Worked example — interrupt-driven I2C device to IIO

Problem: an accelerometer with a data-ready pin; the vendor gave a userspace script that polls i2c-dev at 100 Hz and misses samples.

1. Search: `git grep -i "lis2dh\|st,lis" drivers/iio` — an upstream ST driver supports it. No new code.
2. Overlay: node on the I2C bus with `compatible = "st,lis2dh12-accel"`, `reg = <0x19>`, `interrupt-parent = <&gpio4>`, `interrupts = <3 IRQ_TYPE_EDGE_RISING>`, pinctrl group for the pad as GPIO input.
3. Check `CONFIG_IIO_ST_ACCEL_3AXIS` in `/proc/config.gz`; if missing, enable in the Yocto kernel config fragment or ask Toradex/the BSP for the module.
4. Verify: `/sys/bus/iio/devices/iio:device0/name`, read `in_accel_x_raw`.
5. Buffered capture with the hardware trigger: enable scan elements, set `buffer/length`, `buffer/enable`, read `/dev/iio:device0` (or `iio_readdev` from libiio). Samples are now driven by the data-ready IRQ.

## 10. Review checklist

- [ ] Checked for an upstream driver / similar chip before writing code.
- [ ] `MODULE_DEVICE_TABLE(of, ...)`, `MODULE_LICENSE("GPL")`, matching DT binding documented.
- [ ] All resources `devm_*`; errors via `dev_err_probe`; `-EPROBE_DEFER` propagated, not swallowed.
- [ ] No sleeping calls in atomic context; threaded IRQ for bus devices; locks around shared state.
- [ ] Buffers for SPI/DMA not on the stack; user pointers only via `copy_*_user` with length checks.
- [ ] Polarity and timing taken from DT, not hard-coded.
- [ ] Built against the exact target kernel; vermagic checked; packaged in the image (TorizonCore Builder / Yocto / Buildroot), not copied by hand.
- [ ] Unbind/rebind and module unload/reload tested without oops or leaks (`kmemleak` if enabled).
