# PortalVR camera-tracker protocol

This document describes how to receive hand tracking from the PortalVR iOS camera tracker (`ios-controller-tracking`) and how to send its supported host commands. Install the app from [portalvr.io/tracker](https://portalvr.io/tracker), the link encoded by PortalVR's tracker-install QR code. It covers the current source implementation. The phone controllers have their own protocol in [CONTROLLER.md](CONTROLLER.md). Headset mode and its transport are outside scope.

All offsets below are in bytes, starting at zero. All multibyte payload fields are **little-endian**, with no alignment padding. `f32` and `f64` mean IEEE-754 binary32 and binary64; `u8/u16/u32/u64` mean unsigned integers. Lengths exclude transport headers unless stated otherwise.

## Contents

- [Connection and framing](#connection-and-framing)
- [Tracker POSE packets](#tracker-pose-packets)
- [Tracker clock synchronization and calibration](#tracker-clock-synchronization-and-calibration)
- [Reference receiver code](#reference-receiver-code)
- [Integration checklist and troubleshooting](#integration-checklist-and-troubleshooting)
- [Compatibility notes](#compatibility-notes)

## Connection and framing

| What the host receives | Device endpoint | Transport framing | Startup |
| --- | --- | --- | --- |
| Camera-derived hand/body anchors, optional 21 hand landmarks, acquisition state and timing | TCP port **5005** through USBMux | `PUSB` | Connect; respond to clock requests |

The tracker observes the user's hands from a camera; a [phone controller](CONTROLLER.md) instead uses the phone itself as the tracked object. The stream is not automatically a desktop gamepad or a VR driver: the receiving program must map measurements into its target API, establish an origin, and handle stale tracking.

The tracker uses the iOS [USBMux connection and PUSB framing](CONTROLLER.md#ios-connection-and-framing) shared with the iOS phone controller, on device port **5005** instead of 5006. The phone listens and the host initiates the connection; a new client replaces the old one. Use a data-capable USB cable, keep the app active, grant camera permission, unlock the device, and trust the host. The tracker enables its listener during camera-view setup. As with the controller, its listener is plain TCP without TLS, discovery, or authentication; use trusted hosts and networks.

The tracker's `POSE` packet version `10` is unrelated to controller packet type `10`.

### Scheduling

The tracker sends at the processing pipeline's rate, with separate packets for left and right. It keeps only the newest unsent packet globally, not a queue per hand. Under backpressure, either side's older packet can be overwritten. Sequence gaps and unequal arrival rates are therefore normal.

## Tracker POSE packets

### Payload layout

Current tracker emission uses versions **`0e`** and **`10`**, both with a 120-byte base and 24-byte joint records:

`payload_length = 120 + 24 * joint_count`

Version `10` carries hand-local metric landmarks when available. Version `0e` is the current fallback without landmarks. A normal full-hand `10` packet has 21 joints and is **624 bytes**; a fallback or status packet is **120 bytes**.

| Offset | Type | Meaning |
| ---: | --- | --- |
| 0 | 4 bytes | ASCII `POSE` (`50 4f 53 45`) |
| 4 | `u8` | Pose version `0e` or `10` |
| 5 | `u8` | Joint count |
| 6 | `u8` | Hand side: `0` status/unknown, `1` left, `2` right |
| 7 | `u8` | Pose flags, below |
| 8 | `u32` | Sequence number, wraps modulo `2^32` |
| 12 | `f64` | Capture timestamp, seconds in device capture timebase |
| 20 | `f32` | Anchor normalized capture X |
| 24 | `f32` | Anchor normalized capture Y |
| 28 | `f32` | Anchor tracker X, meters |
| 32 | `f32` | Anchor tracker Y, meters |
| 36 | `f32` | Anchor tracker Z, meters |
| 40 | `f32` | Anchor confidence |
| 44 | `f32` | Queue latency, ms |
| 48 | `f32` | Preprocess latency, ms |
| 52 | `f32` | Model latency, ms |
| 56 | `f32` | Inference latency, ms |
| 60 | `f32` | Pipeline latency, ms |
| 64 | `f32` | Postprocess latency, ms; builder uses hand-solve + selection |
| 68 | `f32` | Body-depth latency, ms |
| 72 | `f32` | Hand-solve latency, ms |
| 76 | `f32` | Selection latency, ms |
| 80 | `f32` | Time waiting in transmitter, ms |
| 84 | `f64` | Phone send uptime, seconds |
| 92 | `f32` | Estimated host-minus-phone clock offset, ms |
| 96 | `f32` | Latest clock-sync round trip, ms |
| 100 | `u16` | Pipeline dropped-frame count, saturated to 65535 |
| 102 | `u16` | Reserved, zero |
| 104 | `f32` | X stabilization delta, meters in pre-leveling camera coordinates |
| 108 | `f32` | Y stabilization delta, same coordinates |
| 112 | `f32` | Phone pitch hint, radians; **current builder writes zero** |
| 116 | `u32` | Global app-state flags |
| 120 | joint records | `joint_count` records of 24 bytes |

Timing stages overlap; do not sum every latency column to calculate end-to-end delay. Dropped-frame telemetry is distinct from transmitter overwrites and sequence gaps. Sequence increments once per constructed packet, including status packets, shared across both hands. Left is constructed before right in a multi-hand frame; do not rely on adjacent arrivals or sequence parity to identify a hand.

### Flags and missing tracking

| Pose flag | Meaning |
| ---: | --- |
| `01` | Has 3D anchor |
| `02` | Has solved 3D skeleton payload |
| `04` | Synthetic pose began during depth freeze |
| `08` | Canonical tracker-local joints qualification |
| `10` | Hand-landmark source (historical name `handIndexSource`) |
| `20` | TrueDepth capture source |
| `40` | Synthetic/extrapolated pose |
| `80` | Recovery blend is active |

The builder sets `08` only for a complete 21-joint hand-landmark result with measured depth, not synthetic or recovery-blended. Version `10` local coordinates are transformed into tracker axes by the current builder even when `08` is clear; the flag additionally distinguishes the fully qualified measured case. It is not safe to infer that an unflagged `10` payload contains old camera-absolute coordinates.

Synthetic packets set `40`, optionally `04`, and clear `01`, `02`, and `08`. Coordinates may still be present as estimates. Recovery packets set `80`, clear `08`, and currently fall back to `0e` without joints. Treat these conditions as different from a new measured anchor. The packet does not transmit a separate orientation, controller buttons, controller identity, or raw IMU data.

App-state bit **`00000001` = acquiring hands** applies globally, independently of whether a hand is currently visible. While set, the host should show acquisition/settling status rather than a ready-to-recenter prompt. When cleared, the tracker is ready for the host to invite recentering. If set again, return to acquisition state.

When no hand packets are available during acquisition, the pipeline can emit a `10` status packet with side `0`, zero joints/position/confidence, and the acquisition flag. Process its app state, but never submit it as a hand at `(0,0,0)`. Outside acquisition, the absence of hand output need not produce a status packet. Use a per-hand receive timeout to detect disappearance; do not wait for a mandatory "hand lost" message.

### Joint records and anchor meaning

Each record at `120 + 24*i` contains:

| Relative offset | Type | Meaning |
| ---: | --- | --- |
| 0 | `f32` | Normalized capture X |
| 4 | `f32` | Normalized capture Y |
| 8 | `f32` | Confidence |
| 12 | `f32` | Local X meters for version `10` |
| 16 | `f32` | Local Y meters for version `10` |
| 20 | `f32` | Local Z meters for version `10` |

Record index is implicit. The 21-joint hand order is:

| Indices | Landmarks |
| --- | --- |
| 0 | Wrist |
| 1–4 | Thumb CMC, MCP, IP, tip |
| 5–8 | Index MCP, PIP, DIP, tip |
| 9–12 | Middle MCP, PIP, DIP, tip |
| 13–16 | Ring MCP, PIP, DIP, tip |
| 17–20 | Little-finger MCP, PIP, DIP, tip |

In hand mode the primary tracking anchor is **joint 9, middle-finger MCP**, despite the historical `handIndex` naming. Body-derived tracks use a body-wrist/palm estimate and do not supply this hand skeleton. Version `10` joints originate from MediaPipe metric hand-local landmarks, rotated/reflected into tracker coordinates; they are **not absolute positions** and are not automatically relative to the packet anchor.

To place a hand skeleton at the measured anchor, use:

```text
absolute_joint[i] = packet_anchor + local_joint[i] - local_joint[9]
```

This anchors the local shape's middle MCP to the measured position. It is an integration mapping, not an extra transmitted transform. Use the local skeleton's directions to estimate an orientation if required; the app provides no quaternion or specified controller-grip offset. If joints are absent, only the anchor is available, so orientation must come from another source or a host-side model.

Historically `0e` joint XYZ meant camera-space points, whereas `10` introduces hand-local coordinates using the same byte stride. Do not silently treat both as local. The current `0e` builder emits zero joints, so new integrations can support that fallback without reconstructing older skeletons. The `0f` compact variant mentioned in acquisition notes is not emitted by this build.

### Tracker coordinates

Normalized capture coordinates are clamped to `[0,1]` and use the capture image convention, **Y increasing upward** (`v_pixel = (1-y)*(height-1)`), before preview/display orientation mapping. They are not necessarily coordinates in a mirrored on-screen preview.

The anchor's `cameraX/Y/Z` field names are historical: the encoder already transforms them into a pitch-leveled and yaw-corrected tracker frame. Depth starts positive away from the camera. In the landscape canonical mapping, physical camera X is negated, Y is preserved, and Z is preserved. Portrait mappings additionally rotate XY. In exact terms, if a projected physical camera point is `(x,y,z)`, canonical coordinates are:

| Orientation | Canonical camera point |
| --- | --- |
| Landscape left/right | `(-x, y, z)` |
| Portrait | `(y, x, z)` |
| Portrait upside-down | `(-y, -x, z)` |

The builder then applies pitch `p` followed by yaw correction `a`:

```text
x1 = x
y1 = y*cos(p) - z*sin(p)
z1 = y*sin(p) + z*cos(p)
X  = x1*cos(a) - z1*sin(a)
Y  = y1
Z  = x1*sin(a) + z1*cos(a)
```

The result is transmitted. Do **not** level it again using phone pitch; the current pitch hint is intentionally zero to prevent double correction by older receivers. Stabilization deltas remain in the earlier camera XY frame and are diagnostic, not an extra correction to add to the transmitted position. The capture-source flag distinguishes TrueDepth; it is not a calibration matrix. Intrinsics and a transform into the desktop VR world are not transmitted. Establish the latter in your receiver.

## Tracker clock synchronization and calibration

### Clock exchange

The tracker sends a PUSB-wrapped `SYNC` request immediately after connection readiness and then about every **2 seconds**, skipping a tick when a pose write is in flight. The host should reply promptly using one monotonic clock throughout the session. Names containing "Mac" in source fields mean the receiver host, regardless of OS.

| Offset | SYNC request (18 bytes) | SYNR response (34 bytes) |
| ---: | --- | --- |
| 0 | ASCII `SYNC` | ASCII `SYNR` |
| 4 | `u8` version `01` | `u8` version `01` |
| 5 | `u8` reserved `00` | `u8` reserved `00` |
| 6 | `u32` sequence | Echo sequence |
| 10 | `f64` phone send uptime `t0`, seconds | Echo `t0` exactly |
| 18 | — | `f64` host receive uptime `t1`, seconds |
| 26 | — | `f64` host send uptime `t2`, seconds |

Capture `t1` when the request is received and `t2` immediately before sending the response. PUSB-wrap the response. At phone receipt time `t3`, the implementation computes:

```text
RTT = max(0, t3 - t0)
host_minus_phone_offset = (t1 + t2)/2 - (t0 + t3)/2
```

It retains the offset from the lowest-RTT exchange on that connection, while reporting the latest RTT. This implementation does not subtract host processing time from RTT. The outgoing offset and RTT fields are in **milliseconds**, though the sync timestamps are in **seconds**. Zero telemetry before synchronization does not establish zero real offset.

For example, approximate host send time is `send_uptime + offset_ms/1000`. Subtract that from the same host monotonic clock to estimate time since transmission. Avoid claiming absolute sensor latency from a capture timestamp unless you have established that capture's clock alignment too. Clock sync is optional for receiving positions, but required for meaningful cross-device timing estimates. The controller apps do not implement this exchange.

### Tracker yaw calibration

Send a PUSB payload of ASCII **`ORST`** to request yaw calibration. The tracker also accepts framed case-insensitive text `orst`, `orientation_reset`, `reset_orientation`, or `yaw_reset`, and framed JSON such as `{"action":"orientation_reset"}`. JSON keys may be `action`, `event`, `type`, or `command`; values may be the three long aliases. JSON key names are case-sensitive. A payload starting with binary ASCII `ORST` is recognized as well. Like the controller, the tracker also accepts the legacy `[u32 payload_length][payload]` envelope and unframed four-byte `ORST` at a message boundary.

The tracker drops pending unsent pose data, asks its yaw estimator to use recent valid measured left/right hand samples after pitch leveling, and resets its screen-space depth index. The geometric basis is `atan2(right.z-left.z, right.x-left.x)`; the opposite rotation is applied to subsequent anchors and local joints. The implementation uses a filtered sample estimate, caps correction at ±45°, and may wait for more samples or decline an insufficient-sample reset. Hold both hands visible in the intended calibration pose. It does not zero position or create a shared world origin.

There is **no calibration acknowledgement or result packet**. Confirm the effect through subsequent valid measurements; with a suitable calibration pose the corrected hands should have approximately equal Z. This command is distinct from the phone controller's byte `7e` and from its [logging-only `ORST` handling](CONTROLLER.md#host--phone-reset-notifications).

### Tracker launch links

The tracker accepts `https://portalvr.io/tracker` (also with surrounding path slashes). An optional valid JSON payload may be supplied through query keys `payload`, `json`, or `data`, a percent-encoded bare JSON query, or a percent-encoded fragment. It stores/logs the accepted link and posts a local notification. The inspected code does not define a JSON configuration schema or use it to configure the stream endpoint. Do not depend on this as a remote tracking-control API.

## Reference receiver code

The following Python 3 code decodes tracker packets and answers clock requests. It reuses the USBMux and PUSB helpers from the [controller reference receiver](CONTROLLER.md#reference-receiver-code): save that code as `receiver.py` and this code as `tracker_receiver.py` in the same directory. No third-party Python modules are needed. It prints decoded data; a production receiver should retain the newest state per hand, apply freshness policy, and avoid printing every sample.

Run `python3 tracker_receiver.py`. With multiple iOS devices, append the desired serial/UDID. For a pre-existing TCP tunnel, run `python3 tracker_receiver.py tcp HOST PORT`. `USBMUXD_SOCKET_ADDRESS` can override the host daemon endpoint. The example never sends a calibration request automatically.

```python
import struct
import sys
import time

from receiver import connect, frame, read_frame


def pose(p):
    if len(p) < 120 or p[:4] != b"POSE" or p[4] not in (0x0e, 0x10):
        raise ValueError("unsupported tracker packet")
    count, side, flags = p[5:8]
    if len(p) != 120 + 24 * count:
        raise ValueError("invalid joint payload length")
    return dict(version=p[4], side=side, flags=flags,
                sequence=struct.unpack_from("<I", p, 8)[0],
                capture_seconds=struct.unpack_from("<d", p, 12)[0],
                normalized_xy=struct.unpack_from("<2f", p, 20),
                anchor_m=struct.unpack_from("<3f", p, 28),
                confidence=struct.unpack_from("<f", p, 40)[0],
                send_seconds=struct.unpack_from("<d", p, 84)[0],
                host_offset_ms=struct.unpack_from("<f", p, 92)[0],
                acquiring=bool(struct.unpack_from("<I", p, 116)[0] & 1),
                joints=[struct.unpack_from("<6f", p, 120 + 24*i) for i in range(count)])


def tracker_loop(s):
    while True:
        p = read_frame(s)
        received = time.monotonic()
        if p[:4] == b"SYNC" and len(p) == 18 and p[4] == 1:
            # Echo version/reserved, sequence and t0 byte-for-byte.
            response = b"SYNR" + p[4:18] + struct.pack("<dd", received, time.monotonic())
            s.sendall(frame(response))
        elif p[:4] == b"POSE":
            print("tracker", pose(p))
        else:
            print("unhandled payload", p.hex())


if __name__ == "__main__":
    with connect(5005, sys.argv[1:], "usage: tracker_receiver.py [SERIAL] | tcp HOST PORT") as sock:
        tracker_loop(sock)
```

To calibrate the tracker use `sock.sendall(frame(b"ORST"))` on its port-5005 connection. Use one serialized writer if multiple threads send messages, including `SYNR` responses.

### Small wire vectors

| Meaning | Hex bytes |
| --- | --- |
| Framed tracker calibration | `50 55 53 42 01 00 00 00 04 00 00 00 4f 52 53 54` |

A tracker acquisition-only sample has `POSE`, version `10`, side/count zero, and app-state `01 00 00 00` at offset 116; its length is 120.

## Integration checklist and troubleshooting

- **iOS connection refused:** check the app is active, the device is trusted, and select port 5005 for the tracker (5006 is the [controller](CONTROLLER.md)). Select the right USB device; a second receiver can replace your iOS connection.
- **Tracker decoder rejects new packets:** support `0e` and `10`; some bundled desktop code still stops at older versions. Validate the 120-byte base and count-derived length.
- **Tracker depth/orientation seems reversed:** distinguish capture-normalized coordinates, canonical tracker coordinates, and the controller's AR world. Avoid applying leveling twice or treating hand-local joints as absolute.
- **Hands jump during acquisition/recovery:** inspect global acquisition state plus synthetic/recovery flags. Keep per-hand state and freshness separately; a packet for one hand does not refresh the other.
- **Hands stay after disconnection:** release hand state when the connection ends or crosses your timeout. The protocol supplies no universally mandated timeout; choose one for your application and tolerate expected scheduling jitter.
- **Packets arrive but pose is frozen:** inspect tracker flags and capture-timestamp progression. Packet cadence and measurement cadence differ.
- **Reset has no effect:** tracker `ORST` estimates yaw only and needs both hands visible; it has no acknowledgement.

The reference code validates current layouts, not all historical firmware. For production, reject nonfinite measurement values before using them, bound buffers, preserve frame boundaries, tolerate unknown flag bits, and keep a single parser state per connection. Discard partial parser buffers before reconnecting.

## Compatibility notes

The current layouts above were checked against the app implementation on **2026-09-15**. Receiver examples were checked with synthetic packets and fragmented streams; this document does not claim physical-device interoperability testing.

Older tracker decoders accepting only versions `01..0d` must be updated before decoding this tracker build. Use the packet versions and layouts documented here rather than assuming compatibility with older protocol descriptions.
