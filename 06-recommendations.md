# 6. Recommendations

These recommendations follow from the evidence in documents 1–5. Where a recommendation rests on a single report or on a hypothesis, it says so.

## For users

### Connection

- Connect the camera directly to the computer if possible, using ZWO's cable or another known-good cable. Avoid long or unpowered extensions.
- Do not connect a camera (including the guide camera) to another camera's USB port; INDIGO's driver README notes these ports are intended for filter wheels and focusers.
- With several ASI cameras on one computer, use moderate USB bandwidth settings (many capture programs do this automatically).
- A thick cable is not necessarily a good one, and users have reported cables becoming unreliable in the cold [Reported]. When a cable is suspected, compare with one that is known to work rather than judging by appearance.

### Power

- Connect the 12 V supply of cooled cameras even when cooling is off.
- Users have reported fewer failures with a supply that stays at 12.0–12.5 V under load, and with a separate supply for the camera when dew heaters, mounts and other loads share the same source. [Reported]
- Give single-board computers the power supply their manufacturer recommends.

### USB 2.0 or USB 3.0

| Use | Suggestion |
|---|---|
| Long-exposure deep-sky imaging | USB 3.0 worked in our snapshot tests and downloads faster. Many capture programs retry failed exposures. |
| Frequent live view or video at modest frame rates | If streams freeze on USB 3.0, try a USB 2.0 port or a USB 2.0 cable. In our tests a 480 Mbps link had no stalls in 120 streams; full-frame snapshot downloads become slower (about 2× in our setup). |
| High-frame-rate planetary video | Needs USB 3.0. If a stream freezes, stop and restart it. |

### When something goes wrong

1. Live view froze or reported a streaming error: restart the stream.
2. The first long exposure after using live view takes much longer than expected or fails: this can be an after-effect of a stalled stream. Some capture software recovers automatically; otherwise disconnect and reconnect the camera in the software.
3. Single frames fail repeatedly:
   - try a known-good cable, then a USB 2.0 cable;
   - check the supply voltage under load;
   - move the guide camera off the main camera's hub;
   - try another camera on the same setup.
4. If only one camera fails on a setup where others work, the camera may need service. Users have reported fixes by ZWO through a firmware reset or a replacement. Contact ZWO support.
5. The camera disappears or no longer responds even after reconnecting in software: unplug USB (and 12 V on cooled cameras), wait a few seconds, reconnect.

## For software developers

[Measured] unless noted.

- **Do not reopen the camera immediately after stopping a stream.** Reopening 7 ms after an aborted stream left the camera unable to enumerate until it was physically replugged (observed once).
- **Do not suppress the SDK's USB resets globally.** With all resets suppressed (as a diagnostic), stalls still occurred and, in our single run, the camera did not recover even after reopening.
- **Recovery that worked in every observed case:** let the next snapshot fail, retry, and reopen (`ASICloseCamera` → `ASIOpenCamera` → `ASIInitCamera`) after the retries. Gain, offset and bandwidth survive `ASIInitCamera`.
- **Bound video reads.** Treat a run of consecutive `ASIGetVideoData` timeouts as a failure and report it, instead of retrying forever: after a stall the stream does not resume by itself.
- **Report stream failures accurately.** If reading frames failed, report the stream as failed even when the subsequent `ASIStopVideoCapture` succeeds.
- **Check return values.** If `ASIStartExposure` fails, retry or fail at once rather than waiting for the full exposure time. If a temperature read fails, do not use the output value. Allow for a 0 reading during the first ~400 ms after `ASIInitCamera` ([Upstream measured]).
- **ASI120 duration changes** ([Upstream measured]): after `ASI_EXP_FAILED`, stop, wait 150 ms, re-apply the duration and restart, up to three times.
- **USB bandwidth:** one choice used by open-source drivers is 80 % of the maximum for a USB 3.0 camera on a USB 2.0 host, and the minimum otherwise. The setting persists in the camera across sessions, so set it explicitly on connect.
- **Approaches that have been discussed publicly but that we have not tested:** handling the video-stream reset differently, or cancelling outstanding transfers before the SDK resets the device (as the AsiCamera project describes); and, at the application level, remembering that a stream stalled so that the next snapshot failure triggers a reopen instead of further retries. Any approach that changes the SDK's own behaviour should be checked against the SDK's license terms first.
- **Regression-test SDK updates** on the cameras you support (see the 1.35 regression in [04-other-causes.md](04-other-causes.md)).
