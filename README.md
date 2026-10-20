# ios-tegu

[![verify](https://github.com/LeR33/ios-tegu/actions/workflows/verify.yml/badge.svg)](https://github.com/LeR33/ios-tegu/actions/workflows/verify.yml)
[![xnu](https://github.com/LeR33/ios-tegu/actions/workflows/xnu.yml/badge.svg)](https://github.com/LeR33/ios-tegu/actions/workflows/xnu.yml)

Apple's iOS 16 running on a **Google Pixel 9a** (tegu, Google Tensor G4).

XNU (xnu-8796, the iOS 16 generation) ported to the Pixel 9a, with the
unmodified iPhone X **iOS 16.7.16 (20H392)** userland on top: launchd, the
system daemons, backboardd, SpringBoard and the stock apps.

This is a port of [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet)
(iOS 16 on a Pixel 9a) adapted to Google's Tensor hardware: the same
XNU patch series, shim architecture and iOS root image pipeline, with the
tegu register map confirmed from the stock **CP3A.260905.009** firmware.

## Status: pre-hardware (kernel builds green, never booted yet)

The kernel compiles clean in CI: `kernel.development.tegu` (Mach-O) is
produced by the `xnu` workflow on every change (milestone 1 of
docs/BRINGUP.md is done). Everything below is a direct port of a proven
architecture, with tegu's confirmed hardware values filled in (see
NOTES.md for the register map and its sources). It has **not run on a
device yet**. The first hardware milestone is listed in docs/BRINGUP.md.

## Target

| Item | Value |
|---|---|
| Device | Google Pixel 9a (`tegu`) |
| SoC | Tensor G4 (`zumapro`): 1x X4 + 3x A720 + 4x A520, PSCI 1.0 |
| Memory | 8 GB @ 0x80000000 |
| GIC | GICv3 (GICD 0x10400000, GICR 0x10440000) |
| Timer | 24.576 MHz generic timer |
| Storage | 128 GB UFS (`samsung,exynos-ufs` @ 0x13200000) |
| Panel | 1080x2406, presented to iOS as an iPhone X panel |
| Stock firmware reference | Android 17, CP3A.260905.009, kernel 6.6 |
| iOS userland | iPhone X (iPhone10,3 / D22AP), iOS 16.7.16 (20H392) |

## How it will boot

```
BootROM -> ABL -> `fastboot boot` shim-boot.img (temporary, nothing flashed)
  -> shim (own MMU, reads the live FDT, loads the payload)
  -> XNU Mach-O + generated Apple Device Tree + boot-args
  -> UFS driver -> HFS+ root in userdata
  -> dyld + shared cache -> launchd -> backboardd + SpringBoard
```

## Build (GitHub Actions, no Linux PC needed)

```bash
gh workflow run verify.yml -R LeR33/ios-tegu    # fast check: all 81 patches apply to xnu-8796.141.3
gh workflow run xnu.yml -R LeR33/ios-tegu      # builds xnu-8796 + patches (macOS ARM runner)
gh workflow run ota.yml -R LeR33/ios-tegu     # fetches stock CP3A.260905.009 images for reference
```

Artifacts produced: `xnu-kernel` (the Mach-O kernel + SHA256SUMS),
`xnu-buildtree` (the patched tree, for local single-file compiles with
`tools/xnu-cc.py`), and the `ota` run's stock images + dumped DTBs.

`verify` (Linux, free) runs on every push touching `xnu/patches/` and proves
the whole series applies cleanly before any macOS build minutes are spent.

Everything hardware-related is tested with `fastboot boot`: temporary, no
flashing, Android stays installed until the iOS root image step.

## Repository layout

- `xnu/patches/` — 81-patch series against xnu-8796 (ported from ios-sweet;
  tegu values applied: GIC, timer, panel; framebuffer address is a runtime
  placeholder pending the live FDT dump — see `xnu/patches/TEGU-PORT.md`)
- `shim/` — bare-metal aarch64 shim (MMU, FDT, console, loader; port of the
  ios-sweet shim, tegu boot-image specifics pending the first fastboot tests)
- `tools/` — `mkadt.py` (tegu ADT), `mkpayload.py`, `mkiosroot.sh`,
  `tegu-root-tweaks.py`, `ota-fetch.sh`, `unpack_bootv4.py`, `fdt-read.py`,
  QEMU test harness
- `docs/BRINGUP.md` — milestones and the on-device debug loop
- `NOTES.md` — confirmed tegu hardware facts (from the stock DTB)
- `docs/reference-sweet-notes.md` — ios-sweet's bring-up history (reference)

## Safety

- Bootloader unlock erases Android user data. Back up the phone first.
- Bootloaders, recovery, modem, persist: never touched. `fastboot boot` only.
- When iOS runs: permissive AMFI, no sandbox, no data protection. Never put
  personal data on it.

Credits: the entire kernel/shim/userland architecture is
[dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet); this repo
adapts it to the Pixel 9a.
