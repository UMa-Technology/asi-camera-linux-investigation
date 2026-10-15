# 4. Other causes of "Exposure failed"

"Exposure failed" is a symptom with several unrelated causes. The cases below do not involve the video stall described in [02-usb3-video-stall.md](02-usb3-video-stall.md).

## 4.1 ASI120: failures after changing the exposure duration

[Upstream measured] INDIGO's ZWO ASI driver documentation (2026-09, SDK 1.41):

- On an **ASI120MC-S**, shortening the exposure duration, or aborting a long video stream, makes the following snapshots return `ASI_EXP_FAILED`. Sometimes three attempts fail before the fourth succeeds.
- A working recovery: stop the failed exposure, wait 150 ms, set the requested duration again, start again; up to three times.
- The same sequence does not fail on an **ASI294MC Pro**.
- With an ASI462MC we saw no failures when only switching between 0.1 s, 1 s and 5 s snapshots (0 of 3 rounds). [Measured]

The ASI120 family is commonly used as a guide camera, and guiding software often changes the exposure duration (calibration, looping, settling), so this case is relevant for guiding.

INDIGO's maintainers also measured that, for roughly 400 ms after `ASIInitCamera`, reading `ASI_TEMPERATURE` returns success with a value of 0 (ASI294MC Pro). Software that records the temperature right after connecting may store 0 °C.

## 4.2 Setup and environment

INDIGO's ZWO ASI driver README lists these causes (paraphrased):

1. Several ASI cameras on one host: lower the USB bandwidth setting of each (around 40–50); with one camera, 80 or lower may help.
2. Some cooled cameras need their 12 V cooler supply connected for reliable operation even when cooling is off.
3. Lower-grade USB cables cause unstable operation.
4. A second camera connected to another camera's USB port: the README notes that the camera's ports are intended for focusers, filter wheels and similar, not for cameras.
5. Some USB hubs cause the problem; try another hub or a direct connection.
6. Single-board computers with insufficient power supplies (the README suggests at least 5.1 V 2.5 A for a Raspberry Pi 3 and 5.1 V 3 A for a Raspberry Pi 4).

**`usbfs_memory_mb`.** ZWO's forum staff recommended 200 MB (thread #7546). ZWO's udev rule sets it to 200 when an ASI device is plugged in. Rules from other vendors set different values (for example 192 or 256); the effective value depends on which rule ran last. Our tests ran at 200.

## 4.3 Individual camera, firmware, cable, power

[Reported] In these threads the same message was resolved in different ways — a cable, the power supply, or a camera that was repaired or replaced — including on ZWO's ASIAIR controllers:

- **Cloudy Nights, "Uh Oh... ASI2600MM 'Exposure Failed'" (2023-02 to 2025-01, ASIAIR Plus).** Every exposure failed. Replacing cables, changing power ports, a separate 12 V supply and a 20-minute power-off did not help; an ASI533MC on the same cables worked. Outdoor temperature was about −6 °C (22 °F); indoors, on a Mac with ASICap, the camera took a few frames and then locked up. ZWO support concluded the main board needed replacement (warranty repair). Another user in the thread first reduced failures by moving the guide camera off the main camera's USB hub, then eliminated them by replacing the cable between camera and ASIAIR; they observed more failures in the cold and with the cooler at full load. A third user (ASI2600 Duo, 2025-01) had a failure after 68 frames, considered it unrelated to temperature, and sent logs to ZWO.
- **AstroBin #205561 (2025-12 to 2026-02, ASI2600MC Pro 2025 version, ASIAIR Plus).** Exposures sometimes were not saved after the countdown, followed by random "exposure failure", also during autofocus and polar alignment. Three cable brands and lengths, another USB 3.0 port, direct camera power, mains power, saving to internal memory, and a firmware reset did not help; the same setup with an older ASI533MC Pro took over 700 frames without problems. ZWO replaced the camera; the replacement took 360 frames without a problem. Other replies: two users reported improvement with a USB 2.0 cable (one after a single night); one user's problem was solved by a firmware hard reset done remotely by ZWO support; several suggested that 11.4–12.0 V under load is low and recommended 12.0–12.5 V and a separate supply for the camera (the original poster tried a separate supply without effect).

## 4.4 SDK version regressions

[Reported] INDI forum #14336 (around 2024, ASI485MC and ASI1600MM): the cameras worked on USB 2.0, but on USB 3.0 they failed after a few exposures ("Exposure failed after 3 attempts"). One user found exposures of 1.5 s or shorter always failed and 2 s or longer always succeeded. Reverting the SDK from 1.35 to 1.34 fixed it (see indilib/indi#2096). SDK updates should therefore be regression-tested on the cameras in use.
