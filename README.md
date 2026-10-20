<p align="center">
  <img src="docs/assets/banner.png" alt="ios-tegu banner" width="720">
</p>

<h1 align="center">ios-tegu</h1>

<p align="center">
  <b>Apple's iOS 16 running on the Google Pixel 9a.</b><br>
  The XNU kernel and the unmodified iPhone userland, ported to Tensor hardware.
</p>

<p align="center">
  <a href="https://github.com/LeR33/ios-tegu/actions/workflows/verify.yml"><img src="https://github.com/LeR33/ios-tegu/actions/workflows/verify.yml/badge.svg" alt="verify"></a>
  <a href="https://github.com/LeR33/ios-tegu/actions/workflows/xnu.yml"><img src="https://github.com/LeR33/ios-tegu/actions/workflows/xnu.yml/badge.svg" alt="xnu"></a>
  <a href="https://github.com/LeR33/ios-tegu/releases"><img src="https://img.shields.io/github/v/release/LeR33/ios-tegu?color=blue&include_prereleases" alt="release"></a>
  <a href="https://github.com/LeR33/ios-tegu/stargazers"><img src="https://img.shields.io/github/stars/LeR33/ios-tegu?style=social" alt="stars"></a>
</p>

---

> This is a port of [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet)
> (iOS 16 on a Redmi Note 10 Pro) to the Pixel 9a (`tegu`, Google Tensor G4):
> the same XNU patch series, shim architecture and iOS root image pipeline,
> with tegu's register map confirmed from the stock **CP3A.260905.009** firmware.
> Experimental research project — **not a usable phone OS**.

## 📋 Status

**The kernel compiles green; nothing has run on a device yet.**

| Milestone | State |
|---|---|
| M1 — CI builds XNU for tegu | ✅ **done** ([v0.1.0-m1](https://github.com/LeR33/ios-tegu/releases/tag/v0.1.0-m1)) |
| M2 — shim runs on hardware | 🔜 next ([tracking issue](https://github.com/LeR33/ios-tegu/issues/6)) |
| M3 — XNU boots to a prompt | ⏳ planned |
| M4 — framebuffer console | ⏳ planned |
| M5 — UFS + HFS+ root + iOS image | ⏳ planned |
| M6 — touch, SpringBoard | ⏳ planned |

`kernel.development.tegu` (ARM64 Mach-O, 17.2 MB) is rebuilt on every push by
GitHub Actions — no local toolchain needed. The full 81-patch series is
proven to apply cleanly (`--fuzz=0`) on xnu-8796.141.3.

## 🎯 Target

| Item | Value |
|---|---|
| Device | Google Pixel 9a (`tegu`), 128 GB |
| SoC | Tensor G4 (`zumapro`): 1× X4 + 3× A720 + 4× A520, PSCI 1.0 |
| Memory | 8 GB @ `0x80000000` |
| Interrupts | GICv3 — GICD `0x10400000`, GICR `0x10440000` |
| Timer | 24.576 MHz generic timer |
| Storage | UFS, `samsung,exynos-ufs` @ `0x13200000` (own driver planned) |
| Panel | 1080×2406, presented to iOS as an iPhone X panel |
| Stock firmware ref. | Android 17, CP3A.260905.009, kernel 6.6 |
| iOS userland | iPhone X (iPhone10,3 / D22AP) **16.7.16 (20H392)**, unmodified |

All hardware values above were confirmed from the stock device trees
(extracted by the `ota` workflow — see `NOTES.md`).

## 🔄 How it will boot

```
BootROM → ABL → `fastboot boot` shim-boot.img   (temporary, nothing flashed)
                │
                ▼
        shim: own MMU setup, reads the live FDT,
              loads the payload from the ramdisk
                │
                ▼
        XNU Mach-O + generated Apple Device Tree + boot-args
                │
                ▼
        UFS driver → HFS+ root in userdata → dyld + shared cache
                │
                ▼
        launchd → backboardd + SpringBoard → 🍏
```

## 🚀 Build (GitHub Actions, no Linux PC needed)

Fork the repo, then from any machine with the GitHub CLI:

```bash
gh workflow run verify.yml -R YOU/ios-tegu    # ~3 min: all 81 patches apply to xnu-8796.141.3
gh workflow run xnu.yml   -R YOU/ios-tegu     # ~15 min: kernel.development.tegu (macOS runner)
gh workflow run ota.yml   -R YOU/ios-tegu     # stock CP3A.260905.009 images + DTBs
```

| Workflow | What it proves | Artifacts |
|---|---|---|
| `verify` | the patch series applies cleanly, `--fuzz=0` | — |
| `xnu` | XNU compiles for tegu (`MACHINE_CONFIG=TEGU`) | `xnu-kernel`, `xnu-buildtree` |
| `ota` | stock firmware reference extracts | `tegu-stock-cp3a` (boot v4 + DTBs) |

The kernel image of the latest green build is also attached to
[releases](https://github.com/LeR33/ios-tegu/releases).

To check a single patch locally without spending CI minutes:

```bash
python3 tools/xnu-cc.py   # replays single compiles from a CI log
```

## 📁 Repository layout

```
xnu/patches/     81-patch series against xnu-8796 (tegu port of ios-sweet's)
shim/            bare-metal aarch64 shim: MMU, FDT, console, loader
tools/           mkadt · mkpayload · mkiosroot · unpack_bootv4 · fdt-read
                 tegu-root-tweaks · ota-fetch · xnu-cc · pstore-pull
docs/            BRINGUP.md (milestones) · QEMU_TESTING.md · reference notes
NOTES.md         confirmed tegu hardware facts (from the stock DTBs)
```

## 🛡️ Safety

- Bootloader unlock erases Android user data — **back up the phone first**.
- All hardware tests use `fastboot boot`: temporary, nothing is flashed,
  Android stays installed until the iOS root image step (M5).
- Apple's security model is switched off in the patch series. Do not put
  anything personal on a booted device.

## 🙏 Credits

- **[dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet)** — the
  original iOS 16 / XNU port to Redmi Note 10 Pro that this project ports
  to the Pixel 9a. Its bring-up history is kept in `docs/reference-sweet-notes.md`.
- Apple's XNU and the iPhone userland remain the property of Apple Inc.
  This is a research project; no Apple code is redistributed.

---

<p align="center">
  <sub>Experimental bring-up. Expect slow boots, missing drivers and wonderful crashes.</sub><br>
  <sub>Next milestone: <a href="https://github.com/LeR33/ios-tegu/issues/6">first boot of the shim on hardware</a> ⚡</sub>
</p>
