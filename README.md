# ZWO ASI Cameras on Linux: An Investigation of Exposure Failures and Video Stalls

Findings, test data and recommendations for intermittent "Exposure failed" errors and stalled video streams with ZWO ASI cameras on Linux hosts.

- **Last updated:** 2026-10-16
- **SDK version tested:** ZWO ASICamera2 SDK 1.41 for Linux, the latest release at the time of writing
- **Status:** independent investigation. The authors are not affiliated with ZWO. Corrections and additional test results are welcome.

## Why this document exists

"Exposure failed" reports for ZWO ASI cameras are common across Linux-based capture software (INDI, INDIGO, custom tools) and also from users of ZWO's ASIAIR controllers. The symptom has several independent causes, and advice in forum threads is often specific to one of them. This document collects what is reported publicly, separates measured behaviour from opinion, and adds the results of controlled tests on one camera and host.

## How to read the evidence

Each statement carries one of these labels:

| Label | Meaning |
|---|---|
| **[Measured]** | Our own repeatable measurement; method in [07-test-methodology.md](07-test-methodology.md) |
| **[Upstream measured]** | Measurements published by maintainers of open-source drivers (INDIGO, INDI) |
| **[Reported]** | User reports in forums or issue trackers that we have not reproduced |
| **[Hypothesis]** | An explanation consistent with the evidence but not verified |

**Scope of our own tests:** one ZWO ASI462MC camera and one Linux host, using ZWO's original USB 3.0 cable. Results on other cameras, hosts and kernels may differ.

## Summary

| # | Topic | Finding | Evidence |
|---|---|---|---|
| 1 | USB port reset on every snapshot | With our ASI462MC on Linux, the SDK reset the camera's USB port once per single-frame exposure, on USB 2.0 and USB 3.0; users report the same with other models. Rare long-run failures have been reported, some recoverable only by power-cycling the camera. | [Measured] [Reported] |
| 2 | Video stalls on USB 3.0 | In our tests the SDK also reset the port about 1.41 s after each of 72 video starts, while frames were already being transferred. On a SuperSpeed link, 4 of 60 video streams in the main run (and 6 of 72 across all runs) stopped delivering frames permanently; in 3 of the 4 main-run cases the next single frame then failed until the camera was closed and reopened. The same camera at 480 Mbps (with and without the same hub): 0 stalls in 120 streams. | [Measured] |
| 3 | SDK transfer handling | Community analyses describe a 500 ms wait after a failed transfer, single outstanding transfers, and repeated frame copies in the Linux SDK; independent implementations that bypass the SDK's video functions report higher frame rates. | [Reported] |
| 4 | ASI120 and exposure-duration changes | On an ASI120MC-S, shortening the exposure or aborting a long video stream makes the next snapshots fail with `ASI_EXP_FAILED`; stopping, waiting 150 ms, re-applying the duration and restarting (up to 3 times) recovers. | [Upstream measured] |
| 5 | Setup and environment | Shared USB bandwidth with several cameras, cooled cameras without 12 V power, cable quality, cameras daisy-chained through another camera's USB hub, some USB hubs, insufficient host power (INDIGO driver README and user reports). | [Reported] |
| 6 | Individual camera, firmware, power | Several reports were resolved by replacing the camera or its main board, a ZWO firmware reset, a different USB cable, or a higher supply voltage, including on ASIAIR. | [Reported] |
| 7 | SDK version regressions | One report found SDK 1.35 made short exposures fail on USB 3.0; reverting to 1.34 fixed it. | [Reported] |

## Quick recommendations

For users (details in [06-recommendations.md](06-recommendations.md)):

- **Long-exposure deep-sky imaging:** USB 3.0 worked in our snapshot tests and downloads faster. Many capture programs retry failed exposures.
- **Live view / video without high frame rates:** if streams freeze on USB 3.0, a USB 2.0 port or a USB 2.0 cable avoided the stall in our tests, at the cost of slower full-frame downloads.
- **High-frame-rate planetary video:** needs USB 3.0; restart the stream if it freezes.
- **Persistent failures:** try a known-good cable (and a USB 2.0 cable), check the supply voltage under load, connect the guide camera directly to the host, and compare with another camera. If only one camera fails on an otherwise working setup, contact ZWO support.
- **Camera disappears or no longer responds:** unplug USB (and 12 V for cooled models) and reconnect.

## Documents

| File | Content |
|---|---|
| [01-sdk-usb-resets.md](01-sdk-usb-resets.md) | Per-snapshot and mid-stream USB resets issued by the SDK |
| [02-usb3-video-stall.md](02-usb3-video-stall.md) | Controlled experiment: video stalls on SuperSpeed vs. High-Speed links |
| [03-sdk-transfer-handling.md](03-sdk-transfer-handling.md) | Community analyses of the SDK's USB transfer handling and ZWO's responses |
| [04-other-causes.md](04-other-causes.md) | ASI120 duration changes, environment, hardware/firmware/power, SDK regressions |
| [05-windows-comparison.md](05-windows-comparison.md) | Reports comparing Windows and Linux; what was not tested |
| [06-recommendations.md](06-recommendations.md) | Recommendations for users and for software developers |
| [07-test-methodology.md](07-test-methodology.md) | Test setup, procedures and raw numbers |
| [SOURCES.md](SOURCES.md) | All referenced threads, issues and projects |

## Open questions

- Why a port reset during a SuperSpeed video transfer is sometimes not recovered from has not been isolated (SDK, Linux USB 3.0 stack, host controller, or camera). Only one host and one kernel version were tested.
- Only the ASI462MC was tested by us; cooled and large-sensor models, and the ASI120 family, may behave differently.
- Results on other hosts (x86 PCs, Raspberry Pi, Jetson) and newer kernels would help to generalise or narrow the findings.

## Method and disclaimer

- Our observations come from the SDK's public API, the calls the SDK makes into the system's libusb library, and Linux kernel logs. No vendor software was decompiled, disassembled or modified, and no vendor files are redistributed here.
- Third-party statements are summarised with links to the original posts; short quotations are attributed. The summaries reflect what was posted and are not independently verified unless marked [Measured].
- This document is provided as is, without warranty. It is not an official statement by ZWO or by any software project mentioned. Test results describe the specific hardware and software listed and may not apply to other setups.
- ZWO, ASI and ASIAIR are trademarks of their respective owner. All other product names are trademarks of their respective owners. This work is not affiliated with or endorsed by any of them.

## License

This work is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0). See [LICENSE](LICENSE).
