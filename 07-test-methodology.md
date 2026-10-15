# 7. Test methodology and raw data

## Setup

| Item | Value |
|---|---|
| Camera | ZWO ASI462MC (USB 3.0 camera, uncooled, color) |
| SDK | ZWO ASICamera2 1.41 for Linux |
| Host | one Linux host |
| USB 3.0 path | host USB 3.0 port → USB hub → camera |
| USB 2.0 path | a separate USB 2.0 port, no hub |
| Cable | ZWO original USB 3.0 cable; a USB 2.0 A–B cable for the "same hub at 480 Mbps" run |
| `usbfs_memory_mb` | 200 |
| USB 3.0 LPM | not active |
| Capture | an application using the SDK's public API; full-size frames (1936×1096), RAW16 |

## Instrumentation

- **SDK call log.** A small shared library loaded with `LD_PRELOAD` wraps the SDK's public C functions (`ASIStartExposure`, `ASIGetExpStatus`, `ASIGetVideoData`, `ASIStartVideoCapture`, `ASIStopVideoCapture`, `ASIOpenCamera`, `ASIInitCamera`, `ASICloseCamera`, `ASIGetControlValue`, `ASISetControlValue`). Each call is logged with `CLOCK_MONOTONIC` time, thread id, arguments and result. The same library can inject failures (for example a failed `ASIStartExposure`, `ASI_EXP_FAILED` status, a failed temperature read, video read timeouts, a failed open) to exercise error paths.
- **Reset counting and timing.** A second preload library wraps `libusb_reset_device` to count, time, or suppress resets. Independently, the kernel's own `usb ...: reset ...` lines are counted with `dmesg`; the raw `dmesg` timestamps use the same monotonic clock as the call log, which allows aligning resets with SDK calls to the millisecond.
- **Stall definition.** A stream is "stalled" when `ASIGetVideoData` times out repeatedly and no further frame arrives (the capture application stops after several consecutive timeouts). A stream with timeouts that later resumes is counted separately.

## Experiments

### A. Per-frame resets (single frames)

Continuous single-frame exposures, counting kernel reset lines per frame, at USB bandwidth 40–100, on USB 2.0 and USB 3.0. Also with resets suppressed, and with injected reset errors. Results: [01-sdk-usb-resets.md](01-sdk-usb-resets.md).

### B. Video stall vs. link (main experiment)

Per round: set bandwidth (40, 80, 100 in rotation) → start a 25-frame stream at 0.2 s exposure → wait for it to end → take one 1 s snapshot. 60 rounds per connection.

| Connection | Bandwidth | Streams | Hard stalls | Recovered streams with timeouts | Frame timeouts | Frames | Failed snapshots |
|---|---|---|---|---|---|---|---|
| USB 3.0, 5 Gbps, via hub | 40 | 20 | 1 | 0 | 8 | 478 | 1 |
| | 80 | 20 | 2 | 0 | 16 | 456 | 2 |
| | 100 | 20 | 1 | 1 | 15 | 478 | 0 |
| Separate USB 2.0 port, 480 Mbps | 40 / 80 / 100 | 20 each | 0 | 0 | 0 | 500 each | 0 |
| Same USB 3.0 port and hub, USB 2.0 cable, 480 Mbps | 40 / 80 / 100 | 20 each | 0 | 0 | 0 | 500 each | 0 |

In the 480 Mbps runs the kernel logged 121 resets per 60 rounds (one per stream, one per snapshot, one at connect), i.e. the SDK's mid-stream reset still happened.

### C. Reset timing within a stream

72 USB 3.0 streams (the 60 of run B, plus 6 aborted streams of D1 and 6 20-frame streams of D2): the reset occurred 1.411–1.415 s after `ASIStartVideoCapture` in every stream. Of the 7 streams with frame timeouts (6 hard stalls and 1 that recovered), 6 first timed out on the 4th read (read started ~0.9 s, timed out ~1.8 s after stream start); the 7th (a D1 stream) first timed out on the 24th read, just before it was aborted.

### D. Abort vs. natural end

| Run | Before the snapshots | Rounds with a failed snapshot |
|---|---|---|
| D1 | continuous 0.2 s stream, aborted after ~4 s | 2 of 3, later 1 of 6 |
| D2 | finite stream ending by itself (10 frames / 20 frames) | 0 of 3 / 1 of 6 |
| D3 | D1, then disconnect and reconnect | 0 of 3 |
| D4 | stream ended by injected (not real) frame timeouts, with or without a reconnect | 0 of 3, 0 of 3 |
| D5 | only exposure-duration changes, no video (0.1 → 1 → 5 → 0.1 s) | 0 of 3 |

In the rounds where every video read was logged (all except the first three D1 rounds), every failed round had a real frame timeout in the preceding stream, and no round without one failed. Injected timeouts (D4) never caused failures because the camera itself had not stalled. In the failing rounds the first temperature read after the stream returned an implausible value (0.0–0.3 °C instead of ~32 °C).

### E. Mitigation trials

| Trial | Rounds | Outcome |
|---|---|---|
| Reopen immediately (≤ 7 ms) after every aborted stream (stalls not logged in this run) | 3 before it was stopped | one round still failed the second snapshot; in one round the camera stopped working (`device firmware changed`, disconnect, `no configurations`, `unable to enumerate USB device` despite the kernel's port power cycle); recovered only by a physical replug |
| Reopen 2 s after an aborted stream | 12 | no failures; none of these streams had stalled; gain/offset/bandwidth (150/20/40) unchanged after `ASIInitCamera` |
| All resets suppressed (B procedure; diagnostic only) | 27 | 24 clean streams; stall after 10 frames in round 25; rounds 26–27 no frames at all, snapshots failing through reopen; run stopped |
| Retry-then-reopen recovery | all failed snapshots after stalls | all recovered; that snapshot reported failed after ~4 × exposure time |

### F. Error-path checks (injected)

With the call-log library injecting failures, a capture application following the recommendations in [06-recommendations.md](06-recommendations.md) behaved as follows:

| Injected failure | Result |
|---|---|
| `ASIStartExposure` fails once, 15 s exposure | retried at once; image after 15.6 s |
| Persistent `ASI_EXP_FAILED` | 3 retries with a 0.150 s pause and re-applied duration, then reopen, then an error is reported |
| Temperature read fails | last valid value kept, error state reported |
| Consecutive video read timeouts | stream reported as failed even though the stop call succeeded |
| `ASIOpenCamera` fails once | the next connection attempt succeeds |

## Limitations

- One camera model, one host, one kernel. Stall rates are from 60 streams per condition.
- The video stream used 0.2 s exposures; higher frame rates were not tested.
- The immediate-reopen failure was seen once.
- All observations come from the SDK's public API, calls from the SDK into the system's libusb library, and kernel logs. No vendor binary was decompiled, disassembled or modified.
