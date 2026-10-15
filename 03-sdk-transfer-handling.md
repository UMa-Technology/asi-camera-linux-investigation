# 3. The SDK's USB transfer handling: community analyses and ZWO's responses

Several people have analysed the Linux SDK's USB transfer behaviour, mostly while chasing low frame rates or timeouts. Their findings are summarised here with attribution and have not been independently verified by us unless noted.

## Error handling after a failed transfer

- **Source:** seeing-things/zwo, `zwo_fixer` (2020–2023).
- **Claim:** after a bulk transfer fails or times out, the SDK cancels it with `libusb_cancel_transfer` but does not check the result. If the transfer had already completed, libusb returns `LIBUSB_ERROR_NOT_FOUND` and no completion callback follows, so the SDK waits for a fixed 500 ms timeout before continuing. `zwo_fixer` hooks the call and, on that error, marks the transfer as finished so the SDK continues immediately.
- **Note:** `zwo_fixer` patches the behaviour of the vendor library at run time; it is described here for completeness, was not used by us, and such patching may conflict with the SDK's license terms.
- **Limits:** according to the project, the fix depends on SDK-internal addresses and supports SDK versions up to 1.18 only; whether the behaviour exists in SDK 1.41 has not been verified. The project later implemented video capture without the SDK's video functions (2023-01).

## Single outstanding transfer, error cascades, copies

- **Source:** INDI forum #8142 and the pawel-soja/AsiCamera project (2020–2021, ASI178MM on a Raspberry Pi 4).
- **Measured by the author:** about 27 fps with spikes of 0.6–1.9 s between frames, against a rated 60 fps.
- **Analysis by the author:** the SDK builds a synchronous flow on top of libusb's asynchronous API (one outstanding transfer at a time); each frame consists of several bulk transfers and one damaged transfer disrupts the following ones ("an avalanche of errors"); each frame is copied about three times.
- **Workaround (as described by the project):** AsiCamera, a replacement for the SDK's video transfer layer. It keeps several transfers outstanding, avoids copies, and cancels its own transfers before the SDK resets the device. Other users reported reaching the target frame rate with ASI178MM and ASI120MM-S on a Raspberry Pi 4; 16-bit mode still dropped frames.
- **Applicability:** written for SDK 1.15; whether it works with SDK 1.41 has not been verified.

## Replacing the SDK's video path

- **Source:** ZWO forum #11601 (2020-11 to 2020-12, ASI290MC, NVIDIA Jetson Nano and Xavier NX).
- **Reported:** kernel warnings (`Event TRB ... with no TDs queued`), frame drops and high CPU load with the SDK's video functions; the user also noted many device resets when something goes wrong. After replacing `ASIStartVideoCapture` / `ASIGetVideoData` / `ASIStopVideoCapture` with their own libusb code using four bulk transfers, they reported 1936×1096 8-bit at 170 fps with no lost frames and much lower CPU load.
- **ZWO's reply** (a ZWO staff member): suggested adjusting the USB bandwidth, separating the receiving thread from processing, and using USB 3.0 rather than USB 2.0; said that bugs in the API, if found, would be fixed; stated that the SDK would not be made open source.

## Linux vs. Windows on the same hardware

- **Source:** ZWO forum #7546 (2017-11, ASI178MM).
- **Reported:** on a dual-boot machine the camera ran at full speed under Windows 10 and had frequent timeouts under Linux with the same ports and cables; each timeout matched an `event TRB ... with no TDs queued` kernel warning.
- **ZWO's reply:** raising `usbcore.usbfs_memory_mb` from the default 16 MB to 200 MB; this improved but did not remove the timeouts for the user. ZWO's engineer considered timeouts to be usually a matter of transmission and camera stability rather than the SDK.

## How this relates to our findings

- The stall in [02-usb3-video-stall.md](02-usb3-video-stall.md) did not resolve in our tests (the stream did not resume), unlike a 500 ms delay, so the cancel-transfer issue alone does not explain it. A port reset while transfers are outstanding is the situation in which that error path would be exercised. [Hypothesis]
- Two independent projects, working on frame-rate problems, chose to implement the video transfer path themselves instead of using the SDK's video functions.
