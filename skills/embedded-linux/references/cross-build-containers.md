# Cross-compilation, SDKs, containers and deployment

## Contents
1. Choosing how to build
2. Toolchains and triplets
3. Yocto SDK and sysroots
4. CMake and Meson with a cross toolchain
5. Buildroot and Debian targets
6. Containers on Torizon OS
7. Device access from containers
8. Multi-arch images and deployment
9. Worked example — C++ app reading GPIO and CAN in a container
10. Review checklist

---

## 1. Choosing how to build

| Target OS | Build the app with | Notes |
|---|---|---|
| Torizon OS | Multi-stage Docker build for `linux/arm64` (Debian-based Torizon containers), or the Torizon VS Code extension templates | Host OS is minimal and read-only; apps ship as containers |
| Yocto image | The image's SDK (`populate_sdk`) or a recipe in your layer | SDK must come from the **same** image build as the target |
| Buildroot | `output/host/bin/<triplet>-gcc` with `output/staging` as sysroot, or a package in `BR2_EXTERNAL` | |
| Debian / Ubuntu on the board | Native build on the board, a Debian cross toolchain (`crossbuild-essential-arm64`) with multiarch libs, or an arm64 container | Match the Debian release of the target |

Native builds on the target are slow but give perfectly matching libraries — fine for a quick test, not for a product pipeline.

## 2. Toolchains and triplets

| SoC | Arch | Linux triplet |
|---|---|---|
| i.MX8M Mini/Plus, i.MX93, AM62x, RK3568, RPi 4/5 (64-bit) | AArch64 (Cortex-A53/A55/A72/A76) | `aarch64-linux-gnu` / `aarch64-poky-linux` / `aarch64-buildroot-linux-gnu` |
| i.MX6/i.MX7, STM32MP1, AM335x | ARMv7-A hard-float | `arm-linux-gnueabihf` |

- `arm-none-eabi-*` is for bare-metal MCUs; it cannot build Linux programs.
- Tuning: `-mcpu=cortex-a53` (i.MX8M Mini/Plus). Do not use `-march=native` in a cross build.
- The C library version on the build side must not be newer than the target's: a binary linked against glibc 2.36 will not run on a target with 2.31 ("version GLIBC_2.34 not found"). Check with `ldd --version` on the target and `objdump -T app | grep GLIBC_` on the binary.
- Quick checks: `file app` (ELF 64-bit ARM aarch64, dynamically linked, interpreter `/lib/ld-linux-aarch64.so.1`), `readelf -d app | grep NEEDED`.

## 3. Yocto SDK and sysroots

```
bitbake my-image -c populate_sdk                      # produces tmp/deploy/sdk/*.sh
./poky-glibc-x86_64-my-image-armv8a-*-toolchain-*.sh -d ~/sdk/my-image
source ~/sdk/my-image/environment-setup-*-linux   # name depends on the tune, e.g. cortexa53-crypto
echo $CC                                              # aarch64-poky-linux-gcc ... --sysroot=...
$CC -O2 -g -o app app.c $(pkg-config --cflags --libs libgpiod)
```
- Always use `$CC`, `$CXX`, `$CFLAGS`, `$LDFLAGS` from the environment script; plain `aarch64-linux-gnu-gcc` misses the sysroot.
- A library missing from the SDK sysroot means it is missing from the image too: add the `-dev` package to the image (or `TOOLCHAIN_TARGET_TASK`) and rebuild the SDK.
- Toradex publishes BSP layers and reference images; Torizon apps normally do not use a Yocto SDK but containers.

## 4. CMake and Meson with a cross toolchain

The Yocto SDK environment sets `OE_CMAKE_TOOLCHAIN_FILE`; `cmake` from the SDK picks it up. For a hand-made toolchain:
```cmake
# aarch64.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_SYSROOT /opt/sysroots/target)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)   # run host tools
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)    # but find target libs
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```
```
cmake -B build -DCMAKE_TOOLCHAIN_FILE=aarch64.cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j
```
`pkg-config` must look in the sysroot: `PKG_CONFIG_SYSROOT_DIR=$SYSROOT PKG_CONFIG_LIBDIR=$SYSROOT/usr/lib/pkgconfig:$SYSROOT/usr/share/pkgconfig`. Meson uses a `--cross-file` with the same information.

Keep debug info: build `RelWithDebInfo`, deploy a stripped copy (`$STRIP` or `objcopy --strip-debug`) and keep the unstripped binary for gdb and core analysis.

## 5. Buildroot and Debian targets

- Buildroot: `make menuconfig` → toolchain, packages; rebuild one package with `make <pkg>-rebuild`; root filesystem overlay via `BR2_ROOTFS_OVERLAY`; kernel config fragments via `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES`. The cross compiler is `output/host/bin/*-gcc` and already points at `output/staging`.
- Debian/Ubuntu targets: `dpkg --add-architecture arm64`, install `crossbuild-essential-arm64` and `libfoo-dev:arm64`, or build inside `docker run --platform linux/arm64 debian:<same release>` (slower under QEMU, but exact).

## 6. Containers on Torizon OS

Torizon OS runs Docker; the application is one or more containers started by `docker compose` (the `docker-compose.service` on the device starts `/var/sota/storage/docker-compose/docker-compose.yml` deployed by Torizon Cloud updates).

Multi-stage Dockerfile (build stage on the PC, small runtime stage):
```dockerfile
ARG BASE_VERSION=4            # container major tag: 4 for Torizon OS 7, 3 for Torizon OS 6 (both Debian Bookworm)
FROM --platform=linux/arm64 torizon/debian:${BASE_VERSION} AS build
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential cmake pkg-config libgpiod-dev && rm -rf /var/lib/apt/lists/*
COPY . /src
RUN cmake -S /src -B /build -DCMAKE_BUILD_TYPE=RelWithDebInfo && cmake --build /build -j

FROM --platform=linux/arm64 torizon/debian:${BASE_VERSION}
RUN apt-get update && apt-get install -y --no-install-recommends libgpiod2 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=build /build/app /usr/local/bin/app
USER torizon
CMD ["/usr/local/bin/app"]
```
Check Toradex documentation for the base image tag that matches the Torizon OS release on the board (platform images such as `torizon/debian-imx8:4` for GPU/VPU access). The library ABI inside the container depends on the Debian release, not on the host: Bookworm's `libgpiod2` is libgpiod 1.6 (v1 API), even though the Torizon OS 7 host tools are v2.

## 7. Device access from containers

```yaml
services:
  app:
    image: registry.example.com/acme/app:1.4.2       # pinned tag, never :latest in production
    restart: unless-stopped
    user: torizon
    group_add: [gpio, i2c, spidev, dialout]          # group names resolve inside the image
    devices:
      - /dev/gpiochip0
      - /dev/verdin-i2c1                             # symlink target must also be passed if the app opens the real node
      - /dev/verdin-uart1
    device_cgroup_rules:
      - "c 254:* rmw"                                # only if devices may appear after start; check the major with ls -l
    network_mode: host                               # required for SocketCAN interfaces
    volumes:
      - /sys:/sys:ro                                 # only if reading sysfs (hwmon/IIO)
      - app-data:/var/lib/app
volumes:
  app-data:
```
Traps:
- `--device` passes the node that exists **at container start**; USB devices that re-enumerate need a bind of `/dev` plus a `device_cgroup_rules` entry, or a restart.
- Major numbers for `gpiochip`, `i2c` (89), `spidev`, `ttymxc`/`ttyUSB` and `hidraw` vary; read them with `ls -l /dev/...` on the target before writing cgroup rules.
- Group IDs: `group_add` with a **name** resolves in the container's `/etc/group`; if the container lacks that group, use the numeric GID from the host (`getent group gpio`).
- `privileged: true` works around all of this but also gives the container every device, lets it remount and load modules, and still does not solve the CAN namespace problem. Use it only for short debugging, never in a shipped compose file.
- gdbserver/strace inside a container need `cap_add: [SYS_PTRACE]` (and possibly `security_opt: [seccomp=unconfined]`).
- Logs: write to stdout/stderr; read with `docker logs` or journald (`logging: driver: journald`). Avoid writing log files into the container's writable layer (flash wear, lost on recreate).

## 8. Multi-arch images and deployment

```
docker run --rm --privileged torizon/binfmt                       # or tonistiigi/binfmt; registers QEMU for arm64 on x86 PCs
docker buildx build --platform linux/arm64 -t registry.example.com/acme/app:1.4.2 --push .
```
On Apple Silicon Macs arm64 builds run natively and fast.

Deploy paths, from quick to production:

| Method | Use for |
|---|---|
| `scp app torizon@board:` + run in a debug container | Quick test |
| `docker save img \| ssh torizon@board docker load` | Offline lab test |
| Registry + `docker compose pull && docker compose up -d` on the board | Development fleet |
| Torizon Cloud application update (compose file as an update package) | Production: signed, atomic, rollback |
| Yocto/Buildroot: new image via OTA (SWUpdate, RAUC, Mender) | Production on non-Torizon |

Deploying changes a device; confirm with the user which board/fleet is targeted.

## 9. Worked example — C++ app reading GPIO and CAN in a container

1. Base image tag chosen to match Torizon OS; `libgpiod-dev` in build stage, `libgpiod2` at runtime on Bookworm — that is libgpiod 1.6 (v1 API), so write the GPIO code against v1 or build libgpiod v2 yourself; on Debian 13 Trixie the v2 runtime package is `libgpiod3`. Check `apt-cache policy libgpiod-dev`.
2. Build with `docker buildx --platform linux/arm64`.
3. Compose: `devices: [/dev/gpiochip0]`, `group_add: [gpio]`, `network_mode: host`.
4. CAN bitrate configured on the host by a systemd-networkd file deployed with the OS customization, not by the container.
5. On the board: `docker compose up -d`, `docker logs -f app`.
6. If `gpiod_chip_open` fails with `ENOENT` → device not passed; `EACCES` → group/GID mismatch; `EBUSY` → line owned by a driver.

## 10. Review checklist

- [ ] Toolchain triplet and glibc match the target; `file` and `objdump -T` checked.
- [ ] SDK/sysroot from the same image build as the target, or a container base that matches the OS release.
- [ ] Unstripped binaries and build IDs archived for every release.
- [ ] Containers: pinned image tags, non-root user, explicit devices and groups, no `privileged`, host network only where needed.
- [ ] Persistent data in named volumes or `/var`, not in the container layer or `/usr`.
- [ ] Deployment through an atomic, rollback-capable path for production.
