# Reliability, boot time and updates

## Contents
1. Boot time — measure, then cut
2. Read-only root filesystem and where data lives
3. OTA updates — OSTree / Torizon, A/B images
4. Rollback and health checks
5. Watchdog with systemd
6. Logging without wearing out the flash
7. Power-fail-safe writes
8. eMMC health and filesystem choice
9. Worked example — making a field device survive power cuts
10. Product review checklist

---

## 1. Boot time — measure, then cut

```
systemd-analyze                         # firmware/loader/kernel/userspace split
systemd-analyze blame | head -20
systemd-analyze critical-chain myapp.service
systemd-analyze plot > boot.svg         # copy to PC, open in a browser
dmesg | grep -E "Freeing unused|Run /sbin/init"   # kernel done at this timestamp
```
Typical cuts, in order of payoff:

| Stage | Action |
|---|---|
| U-Boot | `bootdelay=0` (keep a way to interrupt in manufacturing, e.g. a key or GPIO check); remove network/USB probing from `bootcmd` |
| Kernel | `quiet` on the command line (serial printing at 115200 costs seconds); build drivers you need at boot in, the rest as modules; disable unused subsystems |
| Kernel init | `initcall_debug` + `dmesg` to find slow driver probes (often a driver waiting for a timeout on absent hardware) |
| Userspace | disable unneeded services (`systemctl mask`), avoid `network-online.target` dependencies unless really needed, start the app early with the minimum `After=` |
| Containers | pre-pulled images, small images, no `docker compose pull` at boot |

Record the baseline and each change; boot time regresses silently.

## 2. Read-only root filesystem and where data lives

- **Torizon OS (OSTree)**: `/usr` is read-only and part of the deployment; `/etc` is writable but **3-way merged** on updates (local changes kept, conflicts lose silently — keep app config elsewhere); `/var` is persistent and shared across deployments. App data in Docker volumes (under `/var/lib/docker`) or a bind mount under `/var`.
- **Yocto**: `IMAGE_FEATURES += "read-only-rootfs"`, a separate data partition mounted at `/data` or `/var`, `volatile-binds` for paths that must be writable.
- **Buildroot**: `BR2_TARGET_GENERIC_REMOUNT_ROOTFS_RW` off, data partition in `fstab`.
- Benefits: no corrupted rootfs after power loss, a known state for updates, and tampering is harder.
- Write everything that changes to one well-defined data partition; mount others with `noatime`.

## 3. OTA updates — OSTree / Torizon, A/B images

| Approach | Used by | How |
|---|---|---|
| OSTree (file-level, content-addressed, atomic deployment switch) | Torizon OS, Fedora IoT | Downloads only changed files; new deployment selected at next boot; previous one kept |
| A/B partitions (image-level) | SWUpdate, RAUC, Mender | Write the inactive slot, switch the bootloader flag, roll back on failure |
| Container-only updates | Torizon application updates | New compose file and images; OS unchanged |

On Torizon, Aktualizr on the device talks to Torizon Cloud; OS and application updates are signed (TUF/Uptane). Do not mix in manual `apt`-style changes on the host; they are lost or conflict on the next update.

Before shipping: test update with power cut at each phase (download, install, first reboot), update across two major versions, and update with a full data partition.

## 4. Rollback and health checks

- U-Boot `bootcount` + `bootlimit` + `altbootcmd`: each boot increments a counter; userspace resets it when the system is healthy; exceeding the limit boots the previous deployment/slot. Torizon OS uses this for OS update rollback: after an update Aktualizr sets `upgrade_available=1` and `bootlimit=3`; add health checks as greenboot scripts in `/etc/greenboot/check/required.d/NN-name.sh` (non-zero exit = boot RED, `greenboot-status` shows it). A failing check triggers rollback only while an update is pending; outside an update the device just stays RED.
- "Healthy" must mean the product works, not just that systemd reached `multi-user.target`: the app started, talks to its hardware, and (for connected devices) reached the server.
- Container health: `healthcheck:` in compose plus `restart: unless-stopped`; note that Docker does not restart an **unhealthy** container by itself — use a watchdog service or an orchestrating script for that.

## 5. Watchdog with systemd

System level (hardware watchdog kicked by PID 1; covers kernel and systemd hangs):
```ini
# /etc/systemd/system.conf.d/watchdog.conf  (or system.conf)
[Manager]
RuntimeWatchdogSec=30s
RebootWatchdogSec=10min        # covers a hang during shutdown/reboot
```
Check: `systemctl show -p RuntimeWatchdogUSec`; `wdctl` shows the device, timeout and boot status. Some BSPs already enable this; do not open `/dev/watchdog` from the app if systemd owns it.

Service level (systemd restarts or reboots when the app stops reporting):
```ini
[Unit]
StartLimitBurst=5
StartLimitIntervalSec=10min
StartLimitAction=reboot        # escalate if restarts don't help

[Service]
Type=notify
WatchdogSec=20s
Restart=on-failure
```
```c
#include <systemd/sd-daemon.h>
sd_notify(0, "READY=1");
for (;;) {
    do_work_with_timeouts();
    if (healthy()) sd_notify(0, "WATCHDOG=1");   /* at least every WatchdogSec/2 */
}
```
For a container, run a small host-side systemd service that checks the container's health and is itself watched, or let the app kick through a bind-mounted notify socket (`NOTIFY_SOCKET`) — keep it simple and test that a frozen app really causes a restart.

## 6. Logging without wearing out the flash

- journald: `Storage=volatile` (RAM only) or `persistent` with limits:
  ```ini
  # /etc/systemd/journald.conf.d/size.conf
  [Journal]
  Storage=persistent
  SystemMaxUse=64M
  RateLimitIntervalSec=30s
  RateLimitBurst=1000
  ```
- Docker: `logging: { driver: journald }` or `json-file` with `max-size`/`max-file`; default `json-file` grows without bound.
- Apps: no debug-level logging in production by default; aggregate counters instead of logging every event; never `fsync` per log line.
- Estimate wear: bytes written per day × write amplification (often 2–10 for small writes on eMMC) vs the eMMC's rated TBW. A 100 kB/min logger writes ~50 GB/year before amplification.

## 7. Power-fail-safe writes

Atomic replace of a config or state file:
```c
int fd = open("state.json.tmp", O_WRONLY | O_CREAT | O_TRUNC, 0644);
write_all(fd, buf, len);
fsync(fd);                         /* data on media */
close(fd);
rename("state.json.tmp", "state.json");   /* atomic on POSIX filesystems */
int dfd = open(".", O_RDONLY | O_DIRECTORY);
fsync(dfd);                        /* directory entry on media */
close(dfd);
```
- Without the `fsync` calls, ext4 can leave a zero-length file after a power cut.
- For records, use an append-only log with per-record CRC and sequence numbers, or SQLite with `journal_mode=WAL` and `synchronous=FULL` (or `NORMAL` if losing the last transactions is acceptable).
- Detect power loss early if the hardware allows it (a power-good GPIO into `gpio-keys` or the app) and use the hold-up time (supercap) to finish writes and unmount — then test it with a programmable supply.
- `sync` in a shutdown script is not a strategy; the power cut does not wait for scripts.

## 8. eMMC health and filesystem choice

```
mmc extcsd read /dev/mmcblk0 | grep -i -E "life|eol"    # mmc-utils
# EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A/B: 0x01 = 0-10% used ... 0x0B = exceeded
# EXT_CSD_PRE_EOL_INFO: 0x01 normal, 0x02 warning (80% reserved blocks used), 0x03 urgent
cat /sys/block/mmcblk0/stat                              # sectors written since boot (field 7)
```
- ext4 with `data=ordered` (default) on eMMC/SD; f2fs is an option for write-heavy data partitions; UBIFS on raw NAND (not eMMC).
- Report eMMC life estimation in device telemetry; it is cheap and predicts field failures.
- SD cards: consumer cards lie about flush and wear-level poorly; use industrial cards or boot from eMMC for products.
- Enable `fstrim.timer` (weekly) for eMMC that supports discard, unless the vendor advises otherwise.

## 9. Worked example — making a field device survive power cuts

Starting point: Yocto image, rootfs read-write, app writes `config.ini` in place and logs to `/home/app/log.txt` every second; some units fail to boot after outages.

1. Make the rootfs read-only; add `/data` (ext4, `noatime`) for config and records.
2. Config writes via the atomic temp-file + fsync + rename pattern; keep the previous config as a fallback if the new one fails to parse.
3. Logs to journald with `SystemMaxUse=32M`; app keeps per-hour counters instead of per-second lines.
4. systemd `RuntimeWatchdogSec=30s`, app `Type=notify` + `WatchdogSec=20s`.
5. A/B update with RAUC and U-Boot bootcount; health check marks the slot good only after the app reports ready.
6. Test: 500 random power cuts with a relay-controlled supply during writes and during updates; check boots, data integrity (CRC) and eMMC life fields.

## 10. Product review checklist

- [ ] Boot time measured and within requirement; `quiet`, `bootdelay` and service dependencies reviewed.
- [ ] Rootfs read-only (or OSTree); all mutable data on one partition or volume; `/etc` changes understood under OSTree merge.
- [ ] OTA is atomic, signed, and rolls back on a failed health check; tested with power cuts at every phase.
- [ ] Hardware watchdog active with one owner; app-level watchdog on the main service; restart escalation defined.
- [ ] Logs size-capped and rate-limited; container log drivers bounded; write volume estimated against eMMC endurance.
- [ ] All persistent writes power-fail-safe (fsync + rename, CRC'd records, or a transactional store).
- [ ] eMMC life and reset causes reported in telemetry.
- [ ] Debug access (serial console, SSH keys, U-Boot interrupt) policy decided for production.
