# MCUboot, DFU and security

## Contents
1. MCUboot with sysbuild
2. Keys and signing
3. Slots and partitions
4. Swap modes
5. Test, confirm, revert
6. MCUmgr / SMP device firmware update
7. Serial recovery and boot mode
8. TF-M and PSA crypto
9. Worked example — DFU over RS-485/UART on STM32
10. Review checklist

---

## 1. MCUboot with sysbuild

```
# sysbuild.conf (app folder)
SB_CONFIG_BOOTLOADER_MCUBOOT=y
SB_CONFIG_BOOT_SIGNATURE_TYPE_ECDSA_P256=y
SB_CONFIG_BOOT_SIGNATURE_KEY_FILE="${APP_DIR}/keys/dev-ec-p256.pem"   # dev key (${APP_DIR} expanded 4.4+; absolute path before)
# SB_CONFIG_MCUBOOT_MODE_SWAP_USING_MOVE=y                             # 4.1+ (4.0: _SWAP_WITHOUT_SCRATCH); pick per section 4 — nucleo_f429zi defaults to SWAP_SCRATCH
```

```bash
west build -b nucleo_f429zi --sysbuild app -p always
west flash            # flashes MCUboot and the signed app
```

- Sysbuild sets `CONFIG_BOOTLOADER_MCUBOOT=y` in the app and passes the key; outputs `build/<app>/zephyr/zephyr.signed.bin` / `.signed.hex` (upload these for DFU) and `build/mcuboot/zephyr/zephyr.elf`.
- MCUboot's own Kconfig goes in `sysbuild/mcuboot.conf`, its DT changes in `sysbuild/mcuboot.overlay` (or `sysbuild/mcuboot/boards/<target>.overlay`).
- 3.7: sysbuild selects MCUboot and the key, but the swap mode is chosen with MCUboot's Kconfig (`sysbuild/mcuboot.conf`, e.g. `CONFIG_BOOT_SWAP_USING_MOVE=y`; MCUboot defaults to move on Nordic or when the board has no `scratch_partition`, otherwise scratch) and must match the app's `CONFIG_MCUBOOT_BOOTLOADER_MODE_*` (default `_SWAP_WITHOUT_SCRATCH`, i.e. move). 4.0 added `SB_CONFIG_MCUBOOT_MODE_*`, which sets both sides consistently.
- Without sysbuild (legacy): build MCUboot from `bootloader/mcuboot/boot/zephyr` separately and set `CONFIG_MCUBOOT_SIGNATURE_KEY_FILE` in the app. Prefer sysbuild.
- App version: `VERSION` file in the app (`VERSION_MAJOR`, `VERSION_MINOR`, `PATCHLEVEL`, `VERSION_TWEAK`) feeds the image header; or `CONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION`.

## 2. Keys and signing

- **The default keys shipped in the MCUboot repo (`root-ec-p256.pem`, `root-rsa-2048.pem`...) are public.** A device built with them accepts anyone's firmware.
- Generate: `imgtool keygen -k prod-ec-p256.pem -t ecdsa-p256` (`pip install imgtool` or `bootloader/mcuboot/scripts/imgtool.py`). Store production keys outside the repo (HSM, CI secret); pass with `-DSB_CONFIG_BOOT_SIGNATURE_KEY_FILE=/secure/path/prod.pem`. Use absolute paths on 3.7–4.3: a relative path is resolved against a different base by each image, and `${APP_DIR}` is only expanded by both images from 4.4.
- Multiple verification keys (4.5+): comma-separated list in `SB_CONFIG_BOOT_SIGNATURE_KEY_FILE`; the first signs, the others are public-only PEMs accepted by the bootloader (key rotation).
- Manual signing (CI, re-sign): `imgtool sign --key prod.pem --header-size 0x200 --align 8 --version 1.4.0+12 --slot-size 0x40000 zephyr.bin app.signed.bin` — header size, alignment and slot size must match the build: `--header-size` = `CONFIG_ROM_START_OFFSET`, `--align` = the flash node's `write-block-size`, `--slot-size` = the `slot1_partition` size; the exact command is in `build.ninja`. Do not add `--pad-header`: a Zephyr image built for MCUboot already reserves the header space, and padding again shifts the vector table.
- Encryption (`SB_CONFIG_BOOT_ENCRYPTION=y`) protects the image in transit/external flash; it needs its own key and is not available with direct-XIP.
- Anti-rollback: MCUboot `CONFIG_MCUBOOT_DOWNGRADE_PREVENTION` (version-based, in `sysbuild/mcuboot.conf`); hardware security counters depend on the SoC (check the MCUboot docs for your platform).

## 3. Slots and partitions

- DT partitions: `boot_partition` (MCUboot), `slot0_partition` (primary, runs), `slot1_partition` (secondary, download), optional `scratch_partition`, `storage_partition` (settings), optional `coredump_partition`. The app links to `zephyr,code-partition` (slot0).
- 4.4+ boards describe them with `compatible = "zephyr,mapped-partition"`; older use `fixed-partitions`. C macros `PARTITION_ID()` (4.4+) vs `FIXED_PARTITION_ID()`.
- Sector layout decides what works: STM32F4 mixes 16/64/128 KB sectors (in-tree `nucleo_f429zi` uses a 128 KB scratch); STM32G4/L4/U5 have uniform 2–8 KB pages; nRF52/53 4 KB pages; nRF54L RRAM has no erase constraint. Slots must start on sector boundaries.
- External flash for slot1 (QSPI/SPI NOR) frees internal flash; MCUboot needs the flash driver too.
- NCS: partitions come from the Partition Manager (`pm_static.yml`) unless the project has migrated to DTS (see `migration-and-vendors.md`).

## 4. Swap modes

| Mode | `SB_CONFIG_MCUBOOT_MODE_...` (4.0+) | Revert | Notes |
|---|---|---|---|
| Swap using offset | `SWAP_USING_OFFSET` | Yes | Default from 4.3 (STM32 boards from 4.4); fastest swap, least wear; secondary slot one sector larger; introduced experimental in 4.1 |
| Swap using move | `SWAP_USING_MOVE` (was `SWAP_WITHOUT_SCRATCH` before 4.1) | Yes | Primary slot one sector larger; no scratch; good on uniform-page flash |
| Swap using scratch | `SWAP_SCRATCH` | Yes | Needs scratch partition; scratch wears fastest; needed on non-uniform sectors (STM32F4) |
| Overwrite only | `OVERWRITE_ONLY` | No | Simple, fast; a bad image cannot roll back — rely on serial recovery |
| Direct-XIP (+ revert) | `DIRECT_XIP`, `DIRECT_XIP_WITH_REVERT` | Optional | Runs from either slot, no copy; build/sign an image per slot |
| RAM load | `RAM_LOAD`, `RAM_LOAD_WITH_REVERT` | Optional | Copies to RAM; for external-flash-only parts |
| Single app | `SINGLE_APP` | No | No secondary slot; update only via serial recovery |
| Firmware updater | `FIRMWARE_UPDATER` | — | Separate updater image does the download |

Change mode only together with the partition layout, and test a full update + revert cycle on hardware.

## 5. Test, confirm, revert

1. New image uploaded to slot1, marked **test** (`mcumgr image test <hash>` or `boot_request_upgrade(BOOT_UPGRADE_TEST)`).
2. Reset → MCUboot swaps, boots the new image once.
3. App runs self-tests (comms up, Modbus responding, door sensors sane), then confirms:

```c
#include <zephyr/dfu/mcuboot.h>
if (!boot_is_img_confirmed() && selftest_ok()) {
    boot_write_img_confirmed();
}
```

4. If the app crashes or the watchdog fires before confirming, the next reset reverts to the previous image.

`CONFIG_MCUBOOT_GENERATE_CONFIRMED_IMAGE=y` produces a pre-confirmed image for factory programming — do not use it for OTA images. `permanent` upgrades (`BOOT_UPGRADE_PERMANENT`) skip the safety net.

## 6. MCUmgr / SMP device firmware update

App side (base from `samples/subsys/mgmt/mcumgr/smp_svr`):

```
CONFIG_MCUMGR=y
CONFIG_NET_BUF=y
CONFIG_ZCBOR=y
CONFIG_CRC=y
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_STREAM_FLASH=y
CONFIG_IMG_MANAGER=y
CONFIG_MCUMGR_GRP_IMG=y
CONFIG_MCUMGR_GRP_OS=y
# transport, pick one or more:
CONFIG_UART_MCUMGR=y              # 4.4+ must be set explicitly with the UART transport
CONFIG_MCUMGR_TRANSPORT_UART=y
# CONFIG_MCUMGR_TRANSPORT_BT=y    # BLE (SMP service)
# CONFIG_MCUMGR_TRANSPORT_UDP=y   # Ethernet/LTE
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2304
```

- UART transport uses the UART chosen as `zephyr,uart-mcumgr`; it can share the console with `CONFIG_MCUMGR_TRANSPORT_SHELL` instead.
- Host tools (see the MCUmgr docs table): `smpmgr` (Python), AuTerm (GUI), `mcumgr-client` (Rust), nRF Connect Device Manager (mobile, BLE), legacy Go `mcumgr`. Typical flow: `image upload app.signed.bin` → `image list` → `image test <hash>` → `reset` → app confirms (or `image confirm`).
- SMP over UDP has no authentication by itself: put it behind DTLS (`udp-dtls.conf` in the sample) or a VPN, and never expose it on a public interface.
- Alternatives: hawkBit client (`CONFIG_HAWKBIT`), LwM2M firmware object, or your own HTTP/MQTT download writing into slot1 with `flash_img_*` (`CONFIG_IMG_MANAGER`) then `boot_request_upgrade()`.

## 7. Serial recovery and boot mode

- MCUboot serial recovery (`sysbuild/mcuboot.conf`): `CONFIG_MCUBOOT_SERIAL=y`, `CONFIG_BOOT_SERIAL_UART=y` (or `CONFIG_BOOT_SERIAL_CDC_ACM=y`), entrance by GPIO (`CONFIG_BOOT_SERIAL_ENTRANCE_GPIO`, button from the `mcuboot-button0` alias), by timeout window (`CONFIG_BOOT_SERIAL_WAIT_FOR_DFU`), by pin reset, or by retained boot mode (`CONFIG_BOOT_SERIAL_BOOT_MODE` + app `bootmode_set(BOOT_MODE_TYPE_BOOTLOADER)`). The MCUboot UART comes from MCUboot's `zephyr,console`/`zephyr,uart-mcumgr` chosen in its overlay.
- It is the field recovery path when an overwrite-only or single-app update fails — keep it unless a security review rules it out.

## 8. TF-M and PSA crypto

- Build the non-secure target (`nrf5340dk/nrf5340/cpuapp/ns`, `nrf54l15dk/nrf54l15/cpuapp/ns`, `nucleo_l552ze_q/stm32l552xx/ns`, `b_u585i_iot02a/stm32u585xx/ns`); TF-M is built automatically (`CONFIG_BUILD_WITH_TFM=y`) and provides PSA Crypto, Internal Trusted Storage, Protected Storage and attestation to the app.
- STM32 TF-M targets use TF-M's own BL2 (MCUboot inside TF-M, `CONFIG_TFM_BL2`); its default signature type moved from RSA-3072 to EC-P256 (3.7 generally; for `b_u585i_iot02a` and `stm32h573i_dk` in 4.5). 4.2 moved STM32 BL2 crypto to software Mbed TLS. Check the board's TF-M doc page for flashing scripts (regression/option bytes) — they write option bytes and TrustZone settings: confirm with the user first.
- Without TF-M, use PSA Crypto through Mbed TLS (`CONFIG_MBEDTLS_PSA_CRYPTO_C`, more Kconfig per version) or the platform driver (Nordic Oberon/CRACEN in NCS). Prefer PSA APIs over raw Mbed TLS calls; Zephyr is moving that way across 4.x.
- Secure storage for credentials: TF-M ITS/PS, or `CONFIG_SECURE_STORAGE` (4.x) without TF-M; never plain settings for private keys.
- Debug port lock (APPROTECT on nRF, RDP on STM32) is a production-line step with irreversible options (nRF53/91 ERASEPROTECT without a firmware unlock path, STM32 RDP level 2); plan recovery (Nordic `--recover` clears APPROTECT by erasing everything) and confirm before applying.

## 9. Worked example — DFU over RS-485/UART on STM32

Goal: update locker controllers on an RS-485 bus without a debugger.

1. Board with uniform flash pages (e.g., STM32G474) → `SB_CONFIG_MCUBOOT_MODE_SWAP_USING_OFFSET=y` (4.3+ default) or move; F4 → swap using scratch with the in-tree layout.
2. App: MCUmgr UART transport on the RS-485 UART is possible but SMP over a multi-drop bus has no addressing — either a point-to-point service port, or tunnel SMP frames inside your Modbus/own protocol using a custom SMP transport. Simpler: service UART/USB connector for DFU, Modbus for operation.
3. Production key from CI secret; dev key in repo only for dev builds; `VERSION` bumped every release.
4. App confirms the image only after Modbus server answered and door inputs read sane; watchdog on, so a hang reverts.
5. Test matrix on hardware: good update; image that crashes at boot (must revert); power cut during upload; power cut during swap (MCUboot must resume); wrong-key image (must be rejected); older version (rejected if downgrade prevention on).

## 10. Review checklist

- [ ] Production signing key not in the repo, not the MCUboot default; key rotation plan.
- [ ] Swap mode matches the flash sector layout; partition sizes and alignment verified in `zephyr.dts`.
- [ ] App confirms only after self-test; watchdog active so a hanging image reverts.
- [ ] Power-cut during upload and during swap tested on hardware.
- [ ] Version in `VERSION`; downgrade policy decided.
- [ ] DFU transport authenticated (BLE pairing, DTLS, physical service port) — SMP itself has no auth.
- [ ] Serial recovery path known and documented for the field.
- [ ] TF-M/APPROTECT/RDP steps written down with recovery procedure; irreversible steps confirmed.
