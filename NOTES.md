# Hardware research notes (Pixel 9a / tegu)

Confirmed facts come from the real stock firmware, build **CP3A.260905.009**
(Android 17, September 2026): the base device trees extracted from the OTA's
`vendor_kernel_boot` partition, plus the vendor_boot cmdline. Source of truth
for everything below: `tools/fdt-read.py` on the dumped DTBs
(re-extract with `tools/ota-fetch.sh` + `tools/unpack_bootv4.py`).
Unverified items are marked *(verify)* — usually from the live device
(`/proc/device-tree` differs from the static tree: runtime-added nodes).

## SoC: Google Tensor G4 ("zumapro", Exynos heritage)

- Root compatible: `google,zumapro`. Serial, pinctrl, PMU blocks are all
  `samsung,exynos-*` — Tensor's peripheral heritage is Exynos, not Qualcomm.
- 8 CPUs in the static tree, all `arm,armv9.2`, PSCI enable-method:
  - cpu@0..cpu@300 — capacity 0xc1: the 4x Cortex-A520 cluster (start here)
  - cpu@400..cpu@600 — capacity 0x39f: the 3x Cortex-A720 cluster
  - cpu@700 — capacity 0x400: the Cortex-X4
- PSCI: `/psci` compatible `arm,psci-1.0` ✓

## Confirmed register map (from the stock DTB)

| Block | Address / value |
|---|---|
| DRAM base | `0x80000000` (`/memory@80000000`; static tree declares 2G, real 8G is filled at runtime — *(verify banks from live dump)* ) |
| GICv3 | GICD `0x10400000`/0x10000, GICR `0x10440000`/0x200000, maint PPI 9 |
| Generic timer | `arm,armv8-timer`, `clock-frequency` **24 576 000 Hz**; PPIs 13/14/11/10 flags 0xff08; vtimer PPI 14 → INTID 30 *(verify at first boot)* |
| UFS host | `samsung,exynos-ufs@13200000`, std reg 0x200; vendor regs `0x13201100`/0x2000, `0x13280000`/0x8000 (PHY), `0x132a0000`/0xa014, `0x13204000`/0x4000, `0x13208000`/0x804; **SPI 466**, level-high; `dma-coherent`; sysreg_ufs `0x13020000` |
| Console UART | `uart@10870000` `samsung,exynos-uart`, stock cmdline: `earlycon=exynos4210,mmio32,0x10870000 console=ttySAC0,115200` *(whether the UART pins are reachable on the production board: unverified)* |
| Ramoops | `ramoops_mem@fd3ff000`, 0x400000 total, console 0x200000, pmsg 0x200000 ✓ |
| Watchdogs | `watchdog_cl0@10060000`, `watchdog_cl1@10070000` (`google,zuma-cl*-wdt`) |
| USB | `usb@11210000` (dwc3), phy `0x11100000`, xhci DMA buffers `0x17BC0000` |
| Display (Exynos DPU) | decon `0x19470000`, dsim `0x19440000`, dpp `0x19900000`... (only needed if we ever drive the panel ourselves; plan is bootloader-fb, like ios-sweet) |
| PMICs | S2MPG14 + S2MPG15 (Samsung), SPMI *(details in the dtbo overlays)* |

## Panel

- 1080x2406 (6.3", ~270 ppi). Patch 0046/0061 already present it as an
  iPhone X panel with a centered 1080x2406 window (ios-sweet approach).
- Bootloader framebuffer address is **runtime** on tegu (allocated by the
  bootloader, unlike sweet's fixed 0x9c000000). The shim must read it from the
  live FDT *(verify: dump `/proc/device-tree` from Android; expect a
  framebuffer node under /chosen or /reserved-memory)*. Placeholder
  `0xf0000000` is used in mkadt.py and patches 0010/0012 until then.

## Boot chain (Android side)

- GKI, kernel 6.6: kernel in `boot.img` (header v4, kernel at offset 4096,
  16.6 MB gzip), vendor ramdisk in `vendor_boot.img` (VNDRBOOT v4,
  **page size 2048**, ramdisk lz4-legacy compressed in 2 fragments),
  overlays in `dtbo.img` (dt_table, 10 uncompressed overlays — mostly PMIC
  rails), and **`vendor_kernel_boot`** carries 4 full base DTBs
  (`google,zumapro`). `init_boot` holds the generic ramdisk.
- Stock vendor_boot cmdline (in the image): `dyndbg=... earlycon=...`
- `androidboot.boot_devices=13200000.ufs` in the ramdisk — confirms UFS path.
- *(verify)* exact `fastboot boot` invocation on tegu: likely needs
  `boot.img` + `--vendor_boot` + `--vendor_kernel_boot`.

## Serial / logging

- No exposed debug UART assumed. Debug loop: framebuffer photo +
  ramoops/pstore pull (`tools/tegu-log-pull.sh` equivalent) +
  fastboot stderr. The ramoops region is confirmed at 0xfd3ff000.

## iOS side (same as ios-sweet)

- XNU xnu-8796 + pinned libpthread/hfs/BearSSL; iPhone X (iPhone10,3) iOS
  16.7.16 (20H392) root; identity D22AP. `mkadt.py` now carries the confirmed
  tegu register values above; `--qemu` values unchanged (QEMU virt).

## Reference

- `docs/reference-sweet-notes.md` — the ios-sweet bring-up history and
  hardware notes (SM7150), kept as the primary reference for how every
  subsystem was solved. This project is a port of
  https://github.com/dipa-studio/ios-sweet to the Pixel 9a.
