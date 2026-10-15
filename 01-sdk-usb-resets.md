# 1. USB port resets issued by the SDK

## What users see

Users report, and we observed with an ASI462MC, that on Linux each single-frame exposure is accompanied by a kernel log line such as:

```
usb X-Y: reset high-speed USB device number N using xhci-hcd
usb X-Y: reset SuperSpeed Gen 1 USB device number N using xhci-hcd
```

Most of the time images still arrive. Some users report that after hours or days of operation the camera stops delivering images and only recovers after it is unplugged or power-cycled. [Reported]

- indilib/indi#950 (ASI120MM Mini): one reset per frame regardless of cable or host; not seen during video recording.
- indilib/indi#2361 (ASI676MC, Raspberry Pi, USB 3.0): one reset per frame; after hours to days the camera stops downloading images until it is power-cycled. Reproduced with two cameras on several boards.
- ZWO forum #9937: the same per-frame reset reported for the ASI120MM Mini on Linux.
- INDI forum #8535: a user states that the SDK appears to reset USB before every capture on all of their ZWO cameras; the original poster concluded the reset was not the cause of their particular (USB 2.0 ASI120MC) problem.

## Measurements

All with an ASI462MC and SDK 1.41. [Measured]

**Single frames.** In single-camera runs the number of kernel reset lines equalled the number of exposures exactly, on USB 2.0 and USB 3.0 and for every USB bandwidth setting from 40 to 100:

| Link | Bandwidth | Frames | Resets | Resets per frame |
|---|---|---|---|---|
| USB 3.0 | 40 | 169 | 169 | 1.00 |
| USB 3.0 | 80 | 182 | 182 | 1.00 |
| USB 3.0 | 100 | 185 | 185 | 1.00 |
| USB 2.0 | 40–100 (8 steps) | 61–101 per step | equal to frames | 1.00 |

- Each reset took about 0.15–0.17 s. On USB 3.0 (bandwidth 80, 0.05 s full-frame exposure) it accounted for about 60 % of the snapshot cycle, and snapshots were about 5 times slower than video at the same settings.
- Querying exposure status alone causes no reset; starting an exposure does, whether or not the data is read.
- As a diagnostic only, we intercepted `libusb_reset_device` (with `LD_PRELOAD`) and returned success without resetting: in a USB 2.0 run all snapshots still succeeded and throughput rose by about 31 %. Suppressing resets is not recommended (see [02-usb3-video-stall.md](02-usb3-video-stall.md)).
- Injected reset errors produced no observable change in behaviour.
- A 25-minute run with continuous snapshots produced no failures (1084 frames; this is the same run as the two-camera test below), consistent with reports that failures appear after hours to days.

**Video.** In our tests exactly one reset occurred per video stream. Contrary to what we first assumed, it did not happen when the stream started: aligning kernel timestamps with the SDK call timestamps (same monotonic clock) shows the reset about **1.41 s after `ASIStartVideoCapture`** in 72 out of 72 streams, i.e. while frames are already being transferred. See [02-usb3-video-stall.md](02-usb3-video-stall.md) for the consequences on USB 3.0.

## Where the reset comes from

[Measured] Intercepting calls to the system's `libusb_reset_device` shows that the resets are requested from within the SDK during normal snapshot and video operation, at the times given above; the capture application does not request them.

## Comparison with another vendor

[Measured] In a concurrent test of an ASI camera taking snapshots and a camera from another vendor streaming video on the same host, the kernel logged 1128 resets for 1084 ASI snapshots, i.e. close to one per ASI snapshot and not one per video frame of the other camera. Other vendors were not tested systematically.

## Assessment

- In our tests the reset occurred with every exposure, including the first one after connecting, which indicates that it is part of normal operation rather than a reaction to an error.
- On a healthy link it costs time but rarely causes a failure in short tests.
- Reports of long-run failures that need a power cycle are consistent with a reset occasionally leaving the camera or link in a bad state, but this link has not been demonstrated by us. [Hypothesis]
