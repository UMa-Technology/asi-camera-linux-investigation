# Sources

Dates are those of the thread, issue or commit. Forum content is summarised in our own words; see the original threads for full context.

## Issue trackers

| Source | Topic |
|---|---|
| [indilib/indi#950](https://github.com/indilib/indi/issues/950) | ASI120MM Mini: one USB reset per frame on Linux, not during video |
| [indilib/indi#2361](https://github.com/indilib/indi/issues/2361) | ASI676MC on Raspberry Pi, USB 3.0: per-frame resets; camera stops after hours to days, recovers only by power cycle |
| [indilib/indi#2096](https://github.com/indilib/indi/issues/2096) | Exposure failures with SDK 1.35; fixed by reverting to 1.34 (referenced from INDI forum #14336) |

## ZWO forum

| Source | Topic |
|---|---|
| [#7546 Time Out errors with ASI-178MM / Linux SDK](https://bbs.zwoastro.com/d/7546-time-out-errors-with-asi-178mm-linux-sdk) (2017-11) | Same machine: full speed on Windows, timeouts on Linux; ZWO suggests `usbfs_memory_mb=200` |
| [#9937 Linux SDK ASI120MM Mini USB reset on each frame](https://bbs.zwoastro.com/d/9937-linux-sdk-asi120mm-mini-usb-reset-on-each-frame) | Per-frame USB reset on Linux |
| [#11601 Problems with SDK on Jetson Nano](https://bbs.zwoastro.com/d/11601-problems-with-sdk-on-jetson-nano) (2020-11 to 2020-12) | ASI290MC on Jetson: user replaces the SDK video path with own libusb code; ZWO's reply |

## INDI forum

| Source | Topic |
|---|---|
| [#8142 ASI178 camera performance on Raspberry Pi 4](https://indilib.org/forum/astroberry/8142-asi178-camera-performance-on-rasperry-pi-4.html) | Analysis of the SDK's transfer handling; origin of AsiCamera |
| [#8535 Astroberry Pi 4 and ZWO ASI120MC reset issue](https://indilib.org/forum/astroberry/8535-astroberry-pi-4-and-zwo-asi120mc-reset-issue.html) | USB 2.0 ASI120MC on Raspberry Pi 4; per-capture resets |
| [#14336 ZWO Camera failing exposure](https://indilib.org/forum/ccds-dslrs/14336-zwo-camera-failing-exposure.html) | ASI485MC / ASI1600MM: USB 2.0 fine, USB 3.0 failing; SDK 1.35 → 1.34 |

## Other forums

| Source | Topic |
|---|---|
| [AstroBin #205561](https://app.astrobin.com/forum/topic/205561/zwo-asi2600mc-pro/zwo-2600-mc-2025-version-troubleshooting-missing-frames-and-exposure-failures) (2025-12 to 2026-02) | ASI2600MC Pro 2025 on ASIAIR Plus: missing frames and exposure failures; camera replaced; USB 2.0 cable, firmware reset and supply voltage discussed |
| [Cloudy Nights topic 864144](https://www.cloudynights.com/forums/topic/864144-uh-oh-asi2600mm-exposure-failed/) (2023-02 to 2025-01) | ASI2600MM on ASIAIR Plus: main board replaced; another user's cause was the USB cable |

## Open-source projects

| Source | Topic |
|---|---|
| [seeing-things/zwo](https://github.com/seeing-things/zwo) | `zwo_fixer` (cancel-transfer wait, SDK ≤ 1.18); capture program that bypasses the SDK for video |
| [pawel-soja/AsiCamera](https://github.com/pawel-soja/AsiCamera) | Replacement for the SDK's video transfer layer (SDK 1.15) |
| [indigo-astronomy/indigo](https://github.com/indigo-astronomy/indigo), `indigo_drivers/ccd_asi/README.md` and `REFACTOR.md` | "Exposure failed" checklist; ASI120 duration-change measurements and recovery; temperature reads after init |
