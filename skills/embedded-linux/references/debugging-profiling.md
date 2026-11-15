# Debugging and profiling

## Contents
1. First look — logs and state
2. Kernel configuration and module checks
3. Dynamic debug and printk
4. ftrace basics
5. Kernel oops, panic, kgdb and JTAG
6. Remote debugging with gdbserver
7. Core dumps
8. strace and ltrace
9. perf
10. Memory errors and leaks — valgrind, ASan
11. Worked example — app freezes once a day
12. Review checklist

---

## 1. First look — logs and state

```
dmesg -T -l err,warn                    # kernel errors/warnings with wall-clock time
dmesg -w                                # follow
journalctl -b -p warning                # this boot, warnings and worse
journalctl -b -1 -e                     # previous boot (needs persistent journal) — what happened before the reset
journalctl -u myapp -f                  # one service
systemctl --failed
docker logs --tail 200 -f app
cat /proc/cmdline; uname -a; uptime; free -m; df -h; cat /proc/loadavg
```
Reset cause on i.MX8M: the watchdog driver reports it at boot (`dmesg | grep -i -E "wdog|watchdog|reset"`); `WDIOC_GETBOOTSTATUS` from userspace.

## 2. Kernel configuration and module checks

```
zcat /proc/config.gz | grep -E "CONFIG_(SPI_SPIDEV|GPIO_CDEV|CAN_MCP251XFD|DYNAMIC_DEBUG|FTRACE)="
modinfo mcp251xfd               # parameters, firmware, alias, vermagic
lsmod; cat /proc/modules
ls /sys/module/<name>/parameters/
```
`=y` built in, `=m` module, missing = not available (rebuild the kernel or ask the BSP vendor). Without `/proc/config.gz` look for `/boot/config-*` or the Yocto `tmp/work/.../linux-*/build/.config`.

## 3. Dynamic debug and printk

```
mount -t debugfs none /sys/kernel/debug 2>/dev/null
echo 'module spi_imx +p'        > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/can/* +p' > /sys/kernel/debug/dynamic_debug/control
grep spi_imx /sys/kernel/debug/dynamic_debug/control       # what is enabled (=p)
echo 8 > /proc/sys/kernel/printk                            # print everything to console
```
From boot (for probe-time messages): kernel argument `dyndbg="module mydrv +p"` or `mydrv.dyndbg=+p`. Needs `CONFIG_DYNAMIC_DEBUG`. Turn it off afterwards; it is noisy and slows hot paths.

Boot-time diagnosis arguments: `earlycon` (console before the UART driver), `initcall_debug` (time per driver init), `ignore_loglevel`, `loglevel=8`.

## 4. ftrace basics

```
cd /sys/kernel/tracing                     # or /sys/kernel/debug/tracing on older kernels
cat available_tracers
echo 0 > tracing_on; echo > trace
echo function_graph > current_tracer
echo 'mcp251xfd_*' > set_ftrace_filter     # limit scope or the overhead is huge
echo 1 > tracing_on; sleep 2; echo 0 > tracing_on
head -100 trace

# events instead of functions (cheaper):
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/sched/sched_switch/enable
cat trace_pipe
```
`trace-cmd record -p function_graph -g <func>` / `trace-cmd report` does the same with a file you can copy to the PC (KernelShark to view). Use it for: IRQ latency, which process blocks a driver, why a GPIO edge is handled late. Requires `CONFIG_FTRACE` and friends; product kernels sometimes disable it.

## 5. Kernel oops, panic, kgdb and JTAG

- An oops prints `pc : func+0x40/0x120 [module]` and a call trace. Decode with the matching `vmlinux` / `.ko` with debug info:
  ```
  aarch64-linux-gnu-gdb mydrv.ko -ex 'list *(mydrv_probe+0x40)'
  ./scripts/decode_stacktrace.sh vmlinux $KSRC < oops.txt
  ```
- Capture panics that reboot the board: serial console log, or `pstore`/ramoops (`/sys/fs/pstore/` after reboot) if the DT reserves RAM for it. `panic=10` reboots 10 s after a panic instead of hanging.
- kgdb: kernel argument `kgdboc=ttymxc2,115200 kgdbwait`, then `aarch64-linux-gnu-gdb vmlinux` + `target remote /dev/ttyUSB0`. Shares the console UART; useful for stepping a driver's probe.
- JTAG (Lauterbach, J-Link with OpenOCD support for Cortex-A53) is needed for SPL/U-Boot/DDR bring-up and hard hangs where nothing prints. Toradex modules expose JTAG on the SoM; the Verdin dev board has a header.

## 6. Remote debugging with gdbserver

On the target (in the container add `cap_add: [SYS_PTRACE]`, and expose the port or use host networking):
```
gdbserver :2345 /usr/local/bin/app --arg       # start under debugger
gdbserver --attach :2345 $(pidof app)          # attach to running process
```
On the PC with a cross gdb (`gdb-multiarch` on Debian/Ubuntu, `aarch64-poky-linux-gdb` from the Yocto SDK; on macOS use a Linux container or VM):
```
gdb-multiarch build/app                        # unstripped binary, same build as deployed
(gdb) set sysroot /path/to/target-sysroot      # or 'target:' to fetch libs from the board (slow)
(gdb) target remote 192.168.1.50:2345
(gdb) break main
(gdb) continue
(gdb) info threads; thread apply all bt
```
VS Code: `cppdbg` launch config with `miDebuggerServerAddress` and `miDebuggerPath` set to the cross gdb. The Torizon VS Code extension sets this up for its templates.

Mismatched binary or sysroot = wrong line numbers and "Cannot access memory" in library frames. Keep the build ID: `readelf -n app | grep "Build ID"` on both sides.

## 7. Core dumps

```
ulimit -c                                  # 0 = disabled for this shell
cat /proc/sys/kernel/core_pattern          # '|/usr/lib/systemd/systemd-coredump ...' = systemd handles it
coredumpctl list
coredumpctl info myapp                     # signal, backtrace if symbols are on target
coredumpctl dump myapp -o core.myapp       # copy out and analyse on the PC
```
- systemd services: `LimitCORE=infinity` in the unit; containers: `ulimits: { core: -1 }` in compose. `core_pattern` is global to the host kernel, so a container's crash is handled by the host's handler.
- Without systemd-coredump: `echo '/var/crash/core.%e.%p.%t' > /proc/sys/kernel/core_pattern` (directory on a writable, size-limited partition).
- Analyse on the PC with the unstripped binary and the target's libraries:
  ```
  gdb-multiarch build/app core.myapp -ex 'set sysroot /path/to/sysroot' -ex 'bt full' -ex 'info threads'
  ```
- Limit size and count (`/etc/systemd/coredump.conf`: `MaxUse=`, `ProcessSizeMax=`) so dumps cannot fill the flash.

## 8. strace and ltrace

```
strace -f -tt -T -o /tmp/s.txt ./app            # all threads, timestamps, time per syscall
strace -f -e trace=openat,ioctl,read,write -p $(pidof app)
strace -c -f -p $(pidof app)                    # syscall summary (Ctrl-C to print)
strace -e trace=%file ./app 2>&1 | grep ENOENT  # missing files / devices
ltrace -c ./app                                 # library calls (less reliable on aarch64 / PIE)
```
Typical finds: `openat("/dev/spidev1.0") = -1 EACCES` (permissions/group), `ioctl(..., I2C_RDWR) = -1 EREMOTEIO` (NACK), a `read` blocked for seconds (missing timeout), `futex` waits (lock contention), thousands of tiny `write`s (unbuffered logging). In containers: `cap_add: [SYS_PTRACE]`, or strace from the host on the container's PID (`docker top app`).

## 9. perf

```
perf stat -e task-clock,context-switches,cpu-migrations,page-faults ./app
perf top -p $(pidof app)
perf record -g -p $(pidof app) -- sleep 10       # frame-pointer call graphs (build with -fno-omit-frame-pointer)
perf record --call-graph dwarf -p $(pidof app) -- sleep 10   # without frame pointers; larger files
perf report --stdio --sort comm,dso,symbol
```
- `perf` must match the running kernel; on Yocto add `perf` to the image; on Torizon use a debug container with a matching build or a custom image. Hardware PMU counters on i.MX8M need the PMU node and `CONFIG_HW_PERF_EVENTS`; `perf stat` falls back to software events otherwise.
- `echo -1 > /proc/sys/kernel/perf_event_paranoid` (as root, temporarily) to allow non-root profiling and kernel symbols.
- Copy `perf.data` to the PC and use `perf report --symfs=/path/to/sysroot`, or generate flame graphs with `perf script | stackcollapse-perf.pl | flamegraph.pl`.
- Measure on the target, under real load, with the CPU frequency governor known (`cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`).

## 10. Memory errors and leaks — valgrind, ASan

- **valgrind** works on aarch64 (Yocto: add `valgrind`; Debian containers: `apt install valgrind`). It is 20–50× slower — enough for logic, not for real-time paths.
  ```
  valgrind --leak-check=full --track-origins=yes --error-exitcode=1 ./app
  ```
- **AddressSanitizer** in cross builds: compile and link with `-fsanitize=address -fno-omit-frame-pointer -g`; the target needs `libasan` matching the compiler (Yocto: `gcc-sanitizers` package; Debian: `libasan8` or similar). Roughly 2× slowdown, ~2–3× memory — check RAM on a 1–2 GB module. UBSan (`-fsanitize=undefined`) is cheaper and worth running in CI builds too.
- Leak trend in production without tools: watch `VmRSS` in `/proc/<pid>/status` and `smem`/`pmap -x` over days; a steady climb is a leak, a plateau is a cache.
- Run sanitizer builds of the logic on the PC first (host unit tests); target runs catch what depends on real drivers and timing.

## 11. Worked example — app freezes once a day

Symptom: a containerised data logger stops writing after ~20 h; `docker ps` still shows it running.

1. `journalctl -b`, `dmesg -T` around the time: nothing from the kernel → probably userspace.
2. `strace -f -p <pid>` (from the host): every thread in `futex` or one thread in `read(fd=7...)` forever. `ls -l /proc/<pid>/fd/7` → `/dev/verdin-uart1`.
3. The serial port was opened with `VMIN=1, VTIME=0` (blocking forever); the sensor occasionally misses a reply. Fix: `VMIN=0, VTIME=5` or `poll()` with a timeout, plus retry logic.
4. Add a systemd/compose health check and a software watchdog so a future hang is detected and restarts the service, and count timeouts in the logs.
5. If the threads were in `futex`: attach gdbserver, `thread apply all bt` → lock-order inversion between the logger and the uploader thread.

## 12. Review checklist

- [ ] Persistent journal (size-limited) or remote logging so the previous boot can be inspected.
- [ ] Release binaries built with `-g`, stripped on target, unstripped copies archived by build ID.
- [ ] Core dumps enabled, size-capped, retrievable (`coredumpctl` or a crash directory).
- [ ] Debug tools available on demand (debug container or debug image) without shipping them in production.
- [ ] Every blocking read/ioctl has a timeout; hang points identifiable with `strace`.
- [ ] Profiling done on the target under realistic load before optimising.
- [ ] Dynamic debug, ftrace and `perf_event_paranoid` changes reverted after debugging.
