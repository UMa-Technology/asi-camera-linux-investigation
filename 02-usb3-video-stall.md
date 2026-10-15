# 2. Video stalls on USB 3.0 (controlled experiment)

## Symptom

Shortly after a video stream starts, `ASIGetVideoData` stops returning frames and times out on every call; in our tests the stream did not resume. After such a stall, the next single-frame exposure usually reported `ASI_EXP_FAILED` (status 3; 3 of 4 cases in the main run), and retrying the exposure did not help. Closing and reopening the camera (`ASICloseCamera` → `ASIOpenCamera` → `ASIInitCamera`) restores normal operation. [Measured]

Applications usually present this as a frozen live view, a streaming error, or an "Exposure failed" on the first long exposure after using live view.

## Results

ASI462MC, SDK 1.41, full frame 1936×1096 RAW16. Each "stream" is 25 frames at 0.2 s exposure that ends by itself, followed by one 1 s snapshot. USB bandwidth alternates 40 / 80 / 100 between streams. [Measured]

| Connection | Link speed | Streams | Frames delivered | Stalled streams | Frame timeouts | Failed snapshots afterwards |
|---|---|---|---|---|---|---|
| USB 3.0 port, through a USB hub | 5 Gbps | 60 | 1412 | 4 (plus 1 with scattered timeouts that recovered) | 39 | 3 |
| Separate USB 2.0 port, no hub | 480 Mbps | 60 | 1500 | 0 | 0 | 0 |
| Same USB 3.0 port and hub, USB 2.0 cable | 480 Mbps | 60 | 1500 | 0 | 0 | 0 |

Additional observations:

- **The SDK reset the port about 1.41 s into each stream** (72 of 72 streams in our tests; kernel and SDK timestamps on the same monotonic clock). The reset also happens at 480 Mbps, where it caused no stall.
- **Six of seven streams with timeouts failed at the 4th frame:** frames 1–3 arrive normally, the 4th read starts about 0.9 s into the stream and times out at about 1.8 s, i.e. the 1.41 s reset lands while that frame is in flight.
- **Bandwidth setting:** stalls at 40 / 80 / 100 were 1 / 2 / 1 out of 20 each — no clear dependence at this sample size.
- **Abort vs. natural end:** streams that ended by themselves stalled as often as aborted ones; an earlier impression that "aborting a stream breaks the next snapshot" came from aborted streams that had already stalled.
- **Hub:** with the same hub at 480 Mbps there were no stalls, so the hub alone does not explain it.
- **USB 3.0 link power management (U1/U2):** not active on the test host, so it is not a factor here.
- **`usbfs_memory_mb`:** 200 during all tests (the value ZWO recommends).

## Effect of possible mitigations

[Measured] unless noted.

| Approach | Result |
|---|---|
| Suppress every `libusb_reset_device` call (via `LD_PRELOAD`; diagnostic only) | Stalls became less frequent but did not disappear: 24 clean streams, then a stall after 10 frames. After that stall, two further streams delivered no frames and snapshots kept failing even after reopening the camera. Suppressing all resets also removes the SDK's own recovery. Not a viable fix in our tests. |
| Reopen the camera immediately (7 ms) after every aborted stream (whether those streams had stalled was not logged) | Once, the camera stopped working: at the next routine reset the kernel logged `device firmware changed`, the device disconnected and failed to re-enumerate (`no configurations`), and only a physical replug recovered it. Observed once. |
| Reopen the camera 2 s after every aborted stream | 12 reopens without problems; gain, offset and bandwidth were kept across `ASIInitCamera`. None of these 12 streams had stalled, so this does not show whether a delayed reopen is safe for a stalled camera. |
| Let the next snapshot fail, retry, then reopen (several seconds and several routine resets later) | Recovered the camera in every observed case. Costs about four exposure durations for that snapshot, which is then reported as failed. |

## Interpretation

- [Measured] In these tests video stalls occurred only on the SuperSpeed link, and most stalls coincided with the reset the SDK issues while frames are in flight.
- [Measured] The reset is not the only trigger: stalls still occurred, less often, with resets suppressed.
- [Hypothesis] A port reset during active bulk transfers on a SuperSpeed link occasionally leaves the SDK's transfer pipeline, the camera, or the host controller in a state that requires reinitialisation. Which component is responsible has not been determined; Windows was not tested (see [05-windows-comparison.md](05-windows-comparison.md)).

## Related public analyses

- pawel-soja/AsiCamera (built on SDK 1.15) replaces the SDK's video transfer layer; according to its published source, it cancels all of its own outstanding transfers before the SDK resets the device. It was written for frame-rate problems on a Raspberry Pi 4 rather than for stalls, but it addresses the same situation: a reset while transfers are in flight.
- A ZWO forum user (thread #11601, Jetson Nano / Xavier NX, ASI290MC) wrote that they observed many device resets from the SDK when errors occurred, and reported stable 170 fps after replacing the SDK's video path with their own libusb code.
