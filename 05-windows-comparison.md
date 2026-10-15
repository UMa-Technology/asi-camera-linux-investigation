# 5. Windows comparison

## What has been reported

- ZWO forum #7546 (2017-11, ASI178MM): on a dual-boot computer the camera ran at full speed under Windows 10 and had frequent timeouts under Linux, using the same USB ports and cables. [Reported]
- Reports of "Exposure failed" also exist from users of ZWO's ASIAIR controllers (see [04-other-causes.md](04-other-causes.md)), so the symptom is not limited to third-party Linux software.

## What we did not do

We did not test the cameras on Windows and did not analyse ZWO's Windows driver or the Windows SDK. This document therefore makes no statement about how the Windows software behaves internally.

## What can be said

- On Linux, the stalls we measured required a SuperSpeed link and most coincided with a USB port reset issued during an active video stream (see [02-usb3-video-stall.md](02-usb3-video-stall.md)).
- If the same cameras are indeed more reliable on Windows, possible explanations include differences in how the SDK behaves on each platform, or in how each operating system's USB 3.0 stack handles a reset during active bulk transfers. [Hypothesis] Distinguishing these would require testing the same camera and host under both systems.
