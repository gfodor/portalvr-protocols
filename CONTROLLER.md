# PortalVR phone-controller protocol

This document describes how to receive pose and input from the PortalVR iOS and Android **phone controller** apps, and how to send their supported host commands. Install either app from [portalvr.io/controller](https://portalvr.io/controller), the link encoded by PortalVR's phone-setup QR code. It covers the current source implementations, including differences between platforms. The camera tracker has its own protocol in [TRACKER.md](TRACKER.md). Headset mode and its transport are outside scope.

All offsets below are in bytes, starting at zero. All multibyte payload fields are **little-endian**, with no alignment padding. `f32` and `f64` mean IEEE-754 binary32 and binary64; `u8/u16/u32/u64` mean unsigned integers. Lengths exclude transport headers unless stated otherwise.

## Contents

- [Choose an app and connection](#choose-an-app-and-connection)
- [iOS connection and framing](#ios-connection-and-framing)
- [Android phone USB connection](#android-phone-usb-connection)
- [Phone-controller state](#phone-controller-state)
- [Phone-controller commands](#phone-controller-commands)
- [Reference receiver code](#reference-receiver-code)
- [Integration checklist and troubleshooting](#integration-checklist-and-troubleshooting)
- [Compatibility notes](#compatibility-notes)

## Choose an app and connection

| App | What the host receives | Device endpoint | Transport framing | Startup |
| --- | --- | --- | --- | --- |
| [iOS phone controller](https://portalvr.io/controller) (`portal-controller-ios`) | Phone pose, buttons, joystick, virtual-wand mode, tracking status | TCP port **5006** through USBMux | `PUSB` | Connect; state starts automatically |
| [Android phone controller](https://portalvr.io/controller) (`portal-controller-android`) | Same common controller fields, plus a fixed extension | Android Open Accessory bulk endpoints | Raw typed packets, **no length prefix** | Host writes byte `01` to start state streaming |

A phone controller uses the phone itself as the tracked object; the [camera tracker](TRACKER.md) instead observes the user's hands from a camera. Neither stream is automatically a desktop gamepad or a VR driver: the receiving program must map measurements and inputs into its target API, establish an origin, and handle stale tracking.

The controller apps do **not** currently send identical packets: iOS sends version `03`, 67 bytes; Android phone mode sends version `09`, 171 bytes. Their first 67 bytes have the same field layout. The tracker's `POSE` protocol is separate; its version `10` is unrelated to controller packet type `10`.

### Supported connectivity

Use a data-capable USB cable. Keep the chosen app active and grant camera/motion permissions as requested. For iOS, unlock the device and trust the host; the host needs Apple's USBMux service or a compatible implementation. For Android, grant the USB accessory permission when prompted.

The current phone apps are USB-oriented. Android's `BleClient` name and older README references to BLE are historical: scan/connect methods are now stubs and USB owns transmission. The current iOS controller constructs only a `UsbMuxTransportAdapter`. Do not implement an old BLE service or expect a UDP/WebSocket discovery endpoint from these apps.

The iOS listeners use ordinary TCP without TLS and do not explicitly bind to loopback or restrict network interfaces. Consequently, direct TCP to the phone's reachable IP and port may work if iOS permissions and network routing allow it; this is inferred from the listener configuration, not a verified Wi-Fi workflow. There is no app-level LAN discovery or authentication. USBMux is the documented connection path. Use trusted hosts/networks; payload framing provides neither authentication nor encryption.

## iOS connection and framing

The iOS phone controller and the iOS [camera tracker](TRACKER.md) share this connection and framing layer; they differ in device port and payloads.

The **phone listens**, and the host initiates the connection. Each iOS listener keeps one active client; a new client replaces the old connection. No application handshake, subscription message, or state acknowledgement is required.

The controller UI calls its local connection `PORTAL-USB`; this is a synthetic UI name, not a Bluetooth advertisement or a hostname.

### Connecting through USBMux

A host may use an existing USBMux port forwarder or implement the short USBMux setup below. The [reference code](#reference-receiver-code) later in this file implements it using only Python's standard library.

1. Connect to the host USBMux daemon: normally `/var/run/usbmuxd` on macOS/Linux, or `127.0.0.1:27015` with Apple Mobile Device Support on Windows.
2. Send an XML property list with `MessageType="ListDevices"`, `ClientVersionString`, and `ProgName`. Each daemon message has a 16-byte header: four `u32` values, `[total_length_including_header, 1, 8, tag]`.
3. Read the response property list and select an entry in `DeviceList` whose `Properties.ConnectionType` is `USB`. Preserve its numeric `DeviceID`; `Properties.SerialNumber` identifies the device when several are attached.
4. Open a fresh daemon connection. Send a `Connect` property list with `DeviceID` and `PortNumber`, plus the client strings. **PortNumber is the byte-swapped 16-bit device port**, e.g. Python `socket.htons(5006)`, despite the daemon message header being little-endian.
5. Read the daemon response. `Number == 0` means success. That same socket now carries the app's TCP byte stream; stop interpreting bytes as daemon messages.

The daemon protocol and `PUSB` protocol are different layers. Device ports 5005/5006 are not the daemon's port 27015. To reconnect, obtain a current device ID and open a new tunnel. The bundled tracker host transport also honors `USBMUXD_SOCKET_ADDRESS` as a Unix path or `host:port`.

### PUSB frames

Both iOS apps send this envelope around every payload, including control messages:

| Offset | Type | Value |
| ---: | --- | --- |
| 0 | 4 bytes | ASCII `PUSB` (`50 55 53 42`) |
| 4 | `u8` | Framing version `01` |
| 5 | `u8` | Flags, currently `00` |
| 6 | `u16` | Reserved, currently `0000` |
| 8 | `u32` | Payload byte length |
| 12 | bytes | Payload |

Examples: an iOS controller state is 79 bytes including framing; a tracker packet with 21 landmarks is 636 bytes including framing. TCP read boundaries are arbitrary: accumulate bytes, wait for all 12 header bytes, then for the declared payload, and process all complete frames while retaining a partial suffix. Set a reasonable maximum length in your own receiver.

Both apps also accept host messages with the legacy `[u32 payload_length][payload]` envelope. They additionally accept **unframed four-byte `ORST`** at a message boundary. Prefer `PUSB` for new implementations. Plain reset aliases and JSON must be inside an envelope; arbitrary raw text is not a stream framing format.

The app decoders temporarily interpret a partial `PUSB` header shorter than 12 bytes as a legacy length. Once the rest arrives, they re-evaluate the header; a compatible host should simply wait for the full header instead of allocating memory from those first four bytes. Unknown framing versions are not negotiated.

### Scheduling

The iOS controller's default interval is **11 ms** (about 91 packets/s), subject to backpressure and scheduling. It pulls the newest predicted pose at send time. State packets are snapshots, not input events. Repeated pose timestamps are possible even when other fields change. There is no ACK command and no requirement to send keepalives. At most one write is in flight; outgoing control commands have a small queue of up to eight pending writes.

## Android phone USB connection

Android phone mode uses **Android Open Accessory (AOA)**. The desktop is the USB host/accessory; the phone exposes accessory bulk input/output endpoints. A TCP connection to port 5006 is not the Android phone transport.

### Accessory setup

The following standard USB setup precedes Portal's raw packet stream:

1. If needed, issue endpoint-zero vendor IN request **51**, value/index zero; read the supported AOA version as two little-endian bytes.
2. Send vendor OUT request **52** for each identifying string: index is the string ID, value zero, data is NUL-terminated UTF-8, at most 256 bytes.
3. Issue vendor OUT request **53**, value/index zero, no data; wait for re-enumeration.
4. Find the accessory interface's bulk IN and OUT endpoints from descriptors and claim that interface. AOA devices use Google VID `18d1`, commonly PID `2d00` or `2d01`; use the accessory interface, not an unrelated second interface.

These setup operations are defined by [Android's AOA specification](https://source.android.com/docs/core/interaction/accessories/aoa). USB host software such as a libusb-based program must have permission to access the device.

Portal's app requires these **exact, case-sensitive** identity strings:

| String ID | Meaning | Value |
| ---: | --- | --- |
| 0 | Manufacturer | `PortalVR` |
| 1 | Model | `PortalController` |
| 3 | Version | `1.0` |

IDs 2 (description), 4 (URI), and 5 (serial) may describe your host; the app does not use them for matching. PortalVR's own host sends `https://portalvr.io/controller` as the URI, so Android can offer the app's install page when it is not installed. The three required strings are checked both in `accessory_filter.xml` and by `UsbAccessoryManager`.

After the user grants accessory access and the app opens it, write **one raw byte `01`** on the host-to-phone bulk OUT endpoint. Begin reading the phone-to-host bulk IN endpoint continuously. The app starts its send loop on receipt of any nonempty host data, but `01` is the intended host-ready signal. Repeat that startup signal after each new accessory session.

### Raw stream parsing and flow control

There is **no PUSB envelope and no length prefix** on this connection. Parse by packet type at a packet boundary:

| Direction | Leading byte | Total bytes | Meaning |
| --- | --- | ---: | --- |
| Host → phone | `01` | 1 | Host ready |
| Phone → host | `10` | 171 for version `09` | Controller state |
| Phone → host | `7e` | 2 | Recenter request with wand selector |
| Host → phone | `7e` | 1 | Recenter notification; enables the reset-completed state |
| Phone → host | `7f` | 1 | Kill request |
| Host → phone | `20` | 8 | Haptics |
| Phone → host | `02` | 1 | Best-effort deliberate hangup |

Accumulate split packets and consume concatenated packets. Once a state starts, its embedded data can contain any byte value: never scan its floats for command markers. Check the version at offset 1 before choosing a state length. For an unsupported version, stop/reconnect or use an explicitly implemented version schema; skipping arbitrary bytes cannot reliably resynchronize this format.

The phone sends at **8 ms** intervals (about 125 packets/s). This AOA implementation does **not** wait for per-state acknowledgements, even though Android's source retains historical protocol-version comments about ACKs. Do not send an ACK for each state. The writer drops new state snapshots while a write is outstanding and closes the session if a state write stays blocked for about one second. Keep draining the input endpoint even if rendering or processing runs more slowly. Control writes use a separate reliable scheduling path, but still have no delivery acknowledgement.

On a hangup, EOF, detach, or I/O failure, release stale inputs and re-establish the accessory session as needed. `02` is best-effort; its absence does not prove that a connection is alive.

## Phone-controller state

### Common 67-byte layout

This layout is the entire current iOS packet and the prefix of the Android phone packet.

| Offset | Type | Field / meaning |
| ---: | --- | --- |
| 0 | `u8` | Packet type `10` |
| 1 | `u8` | Version: iOS `03`, Android `09` |
| 2 | `3 × f32` | Position `x, y, z`, meters |
| 14 | `4 × f32` | Orientation quaternion **x, y, z, w** |
| 30 | `u32` | Button/touch bitmask |
| 34 | `2 × f32` | Joystick `x, y` |
| 42 | `u64` | Pose timestamp, device-local nanoseconds |
| 50 | `u8` | Virtual-wand mode |
| 51 | `u32` | Feature flags |
| 55 | `u32` | Session-relative timestamp, milliseconds |
| 59 | `u8` | Tracking state, low 2 bits |
| 60 | `u8` | Tracking reason, low 4 bits |
| 61 | `u8` | Interaction mode: **0 for phone mode** |
| 62 | `u8` | Trackpad flags: bit 0 = active |
| 63 | `u16` | Trackpad X, normalized to `0..65535` |
| 65 | `u16` | Trackpad Y, normalized to `0..65535` |

### Android phone extension: bytes 67–170

The Android phone still emits these fields because it uses a shared encoder. Consume all 171 bytes. A phone-mode receiver need not use the auxiliary pose or velocity fields.

| Offset | Type | Phone-mode meaning |
| ---: | --- | --- |
| 67 | `3 × f32` | Unused auxiliary position, `(0,0,0)` |
| 79 | `4 × f32` | Unused auxiliary quaternion, `(0,0,0,1)` |
| 95 | `u32` | Unused auxiliary buttons, `0` |
| 99 | `2 × f32` | Unused auxiliary joystick, `(0,0)` |
| 107 | `u8` | Auxiliary feature flags, copies common flags' low byte |
| 108 | `u8` | Auxiliary-primary indicator, `0` |
| 109 | `2 × u8` | Reserved, `0,0` |
| 111 | `u8` | Trigger amount: `255` when button bit 3 is set, otherwise `0` |
| 112 | `u8` | Grip amount: `255` when button bit 4 is set, otherwise `0` |
| 113 | `2 × u8` | Unused auxiliary trigger/grip, `0,0` |
| 115 | `3 × f32` | Linear velocity slot; phone path does not populate it |
| 127 | `3 × f32` | Angular velocity slot; phone path does not populate it |
| 139 | `3 × f32` | Unused auxiliary linear velocity, zeros |
| 151 | `3 × f32` | Unused auxiliary angular velocity, zeros |
| 163 | `u64` | Unused non-phone status slot, zero in a fresh phone-mode client |

The phone's velocity slots start at zero and are not measurements of velocity. If you need velocity, estimate it from valid timestamped positions in your receiver. Do not interpret a neutral extension as a second independently tracked phone. Modes 4 and 5 below also contain only one measured pose.

### Buttons and feature flags

The button field is a bitmask, **not** the numeric button IDs used internally by the UI.

| Bit | Mask | Meaning |
| ---: | ---: | --- |
| 0 | `00000001` | Action 1: A / X |
| 1 | `00000002` | Action 2: B / Y |
| 2 | `00000004` | Stick click |
| 3 | `00000008` | Trigger (volume up mapping) |
| 4 | `00000010` | Squeeze / grip (volume down mapping) |
| 5 | `00000020` | Menu |
| 6 | `00000040` | iOS: camera-control button; Android: stick touch |
| 7 | `00000080` | Thumbrest touch |
| 8 | `00000100` | Android extended trigger-touch bit |
| 9 | `00000200` | Android extended squeeze-touch bit |
| 10 | `00000400` | Android extended action-1-touch bit |
| 11 | `00000800` | Android extended action-2-touch bit |

Extended touch bits are part of the shared Android mask; phone UI inputs do not promise all of them. In particular, bit 6 is not semantically identical across the two apps. Preserve unknown bits if forwarding, but ignore them when mapping features you do not implement.

New presses are held in outgoing snapshots for at least `ceil(25 ms / TX interval)` sends/builds: normally three frames on iOS and four on Android. A short physical/UI press can therefore appear down after release. Detect rising/falling edges by comparing successive masks; do not trigger a new press for every set bit on every packet. Reset your remembered mask on disconnect.

Common feature flag bit 0 (`1`) means **aim**, and bit 1 (`2`) means **stretch**. iOS emits both according to its state. The current Android phone encoder only sets stretch; aim remains zero. These flags request receiver behavior; the packet does not define a stretch distance or an aim algorithm.

### Virtual-wand selection

| Value | Meaning |
| ---: | --- |
| 0 | Right, persistent |
| 1 | Left, persistent |
| 2 | Right, ephemeral |
| 3 | Left, ephemeral |
| 4 | Dual mirrored |
| 5 | Dual opposed |

The phone can change its virtual hand while remaining on the same connection. Route the common pose and input fields using this byte, not by assuming one fixed hand per device. Ephemeral/persistent describes app-side selection behavior; use the current byte immediately. Dual mirrored/opposed asks the receiver to derive two virtual controllers from the one phone pose. The apps do not transmit a second measured pose, fixed separation, or a complete synthesis algorithm for these modes; choose those mappings in your runtime.

### Joystick and trackpad

Joystick axes are floats with nominal range `[-1,1]` and neutral `(0,0)`; phone UI code applies its dead zone before transmission. The phone touch-joystick paths send **positive X right and positive Y down** in their view coordinates. Invert joystick Y if your target API expects up-positive. Trackpad fields are **absolute normalized coordinates**, not deltas. Divide by 65535. Both encoders' trackpad update paths invert input X and Y, so increasing coordinates represent left/up movement relative to their original input coordinates.

Use trackpad bit 0 as the validity/touch condition. Inactive coordinates may contain the final touch location instead of zero. On a new active touch, establish a baseline; compute differences only within that touch. Android additionally gates trackpad activation on camera-control being enabled and a recenter having been sent or acknowledged. Availability depends on the current UI path.

### Pose coordinates, clocks, and validity

Position is the phone camera's AR world position in meters; quaternion order is `xyzw`, identity `(0,0,0,1)`. The source uses ARKit on iOS and ARCore on Android, with a local session origin. No geographic coordinates or inter-device shared origin is transmitted. Host calibration must align this local world with the application's world.

iOS takes ARKit's camera transform and applies a **local +90° Z rotation** to its quaternion (`q_output = q_ARKit * q_Z90`) before prediction. Position is not rotated by that correction. Android uses its predicted pose or ARCore's display-oriented camera pose. Do not apply the iOS correction again or assume the quaternion is the untouched sensor orientation. The phone stream is suitable for 6DoF use; the [camera tracker](TRACKER.md) stream has different axes and no quaternion.

The pose can be filtered/predicted. On iOS the timestamp is the latest AR frame time aligned to the app's uptime clock (falling back to current uptime when tracking is inactive); predicted pose changes may share a timestamp. Android phone production stamps the sample with `elapsedRealtimeNanos()`. These are **not Unix timestamps and are not synchronized to the host**. The iOS controller does not run the tracker's [clock-sync exchange](TRACKER.md#clock-exchange).

Session milliseconds are calculated separately: from TX-loop start on iOS, and from state-client construction on Android. They wrap modulo `2^32` (about 49.7 days), and are not a packet sequence number. Reconnects need not reset every clock or field. Use local receive time for stream staleness and device time for sample intervals; do not subtract unrelated host and phone clock epochs.

| Tracking state | Meaning |
| ---: | --- |
| 0 | Unknown |
| 1 | Tracking |
| 2 | Limited |
| 3 | Not tracking |

| Tracking reason | Meaning |
| ---: | --- |
| 0 | None |
| 1 | Excessive motion |
| 2 | Insufficient features |
| 3 | Insufficient light |
| 4 | Relocalizing |
| 5 | Initializing |
| 6 | Bad state |

Apply both fields; reason zero alone does not mean valid tracking. Apps can stream status and cached/neutral poses while tracking is unavailable. Decide whether to hold the last pose, fade it, or disable it. Do not mistake continuous packets for fresh optical tracking.

## Phone-controller commands

For iOS, each command below is a PUSB payload. For Android phone USB, send/receive the raw bytes in the direction specified.

### Phone → host: recenter and kill

- iOS recenter: payload **`7e`**, one byte. The host chooses the relevant current wand from controller state.
- Android recenter: payload **`7e HH`**, two bytes, where `HH=00` means **left**, `HH=01` means **right**. This selector is deliberately different from the state packet's wand encoding (`0=right`, `1=left`).
- Both apps: payload **`7f`**, one byte, is a request for the remote runtime to terminate. It is not an automatic local app shutdown. A generic receiver may decline it, but should recognize it explicitly rather than misparse subsequent state bytes.

These commands do not carry poses or timestamps. Implement recenter using a recent valid state and your chosen world alignment. There is no prescribed success response.

### Host → phone: reset notifications

On Android, **raw `7e`**, one byte, marks orientation reset as completed in the app (`orientationResetSentThisSession=true`). It does not itself rotate the phone's AR world. Do not append a hand selector to the host-to-phone form.

On the iOS controller, a PUSB payload `ORST` (ASCII), or a framed case-insensitive text alias `orst`, `orientation_reset`, `reset_orientation`, or `yaw_reset`, is recognized but **only logged**. It does not apply a pose transform. Raw four-byte `ORST` is also accepted by its framing parser. The one-byte host payload `7e` is not handled as a reset by this controller implementation. This differs from both Android and the iOS tracker's [yaw calibration](TRACKER.md#tracker-yaw-calibration).

### Host → phone: haptics

| Offset | Type | Meaning |
| ---: | --- | --- |
| 0 | `u8` | `20` |
| 1 | `u8` | Amplitude `0..255`, scaled to `0..1` |
| 2 | `u8` | Frequency code, currently ignored by phone paths; send `0` |
| 3 | `u8` | Hand mask: bit 0 right, bit 1 left; `3` both |
| 4 | `u32` | Duration in milliseconds |

For example, **`20 80 00 03 64 00 00 00`** requests about half amplitude for 100 ms. Wrap those eight bytes in PUSB on iOS; write them directly on Android. Both phones ignore mask zero, and vibrate their single phone actuator for a nonzero mask without selecting a physical left/right actuator.

Amplitude zero stops/supersedes a pulse. Positive amplitude with duration zero defaults to 50 ms. iOS approximates duration using repeated impact feedback at 35 ms intervals, scales intensity, and caps duration at 500 ms. Android phone vibration clamps positive durations to 40–5000 ms and rate-limits requests. Android decodes the duration into a signed integer internally; keep requests at or below 5000 ms rather than relying on unsigned overflow behavior. Haptics are not acknowledged.

## Reference receiver code

The following Python 3 code is self-contained for the iOS controller stream and also supplies a parser for Android phone data. Save it as `receiver.py`. No third-party Python modules are needed for USBMux. Its transport helpers are reused by the [tracker receiver](TRACKER.md#reference-receiver-code). It prints decoded data; a production receiver should retain the newest state, apply freshness policy, and avoid printing every sample.

Run `python3 receiver.py`. With multiple iOS devices, append the desired serial/UDID. For a pre-existing TCP tunnel, run `python3 receiver.py tcp HOST PORT`. `USBMUXD_SOCKET_ADDRESS` can override the host daemon endpoint. The example never sends kill or reset automatically.

```python
import os
import plistlib
import socket
import struct
import sys


def exact(sock, n):
    data = bytearray()
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise EOFError("connection closed")
        data.extend(chunk)
    return bytes(data)


def daemon_socket():
    target = os.getenv("USBMUXD_SOCKET_ADDRESS")
    if target is None:
        target = "127.0.0.1:27015" if os.name == "nt" else "/var/run/usbmuxd"
    host, sep, port = target.rpartition(":")
    if sep and host and port.isdigit():
        return socket.create_connection((host, int(port)))
    s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    s.connect(target)
    return s


def plist_request(s, fields):
    fields = dict(fields, ClientVersionString="portal-example", ProgName="portal-example")
    body = plistlib.dumps(fields, fmt=plistlib.FMT_XML)
    s.sendall(struct.pack("<4I", 16 + len(body), 1, 8, 1) + body)
    length, version, kind, tag = struct.unpack("<4I", exact(s, 16))
    if not 16 <= length <= 4 * 1024 * 1024:
        raise ValueError("invalid daemon message length")
    return plistlib.loads(exact(s, length - 16))


def usbmux_connect(port, serial=None):
    with daemon_socket() as s:
        result = plist_request(s, {"MessageType": "ListDevices"})
    devices = []
    for entry in result.get("DeviceList", []):
        props = entry.get("Properties", {})
        if props.get("ConnectionType") == "USB":
            if serial is None or props.get("SerialNumber") == serial:
                devices.append(entry)
    if len(devices) != 1:
        names = [d.get("Properties", {}).get("SerialNumber") for d in devices]
        raise RuntimeError(f"select one attached USB device; matching serials: {names}")
    s = daemon_socket()
    result = plist_request(s, {"MessageType": "Connect", "DeviceID": devices[0]["DeviceID"],
                               "PortNumber": socket.htons(port)})
    if result.get("Number") != 0:
        s.close()
        raise RuntimeError(f"USBMux connect failed: {result}")
    return s


def connect(default_port, argv, usage):
    """Open a USBMux tunnel ([SERIAL]) or an existing TCP tunnel (tcp HOST PORT)."""
    if len(argv) == 3 and argv[0] == "tcp":
        return socket.create_connection((argv[1], int(argv[2])))
    if len(argv) <= 1:
        return usbmux_connect(default_port, argv[0] if argv else None)
    raise SystemExit(usage)


def frame(payload):
    return struct.pack("<4sBBHI", b"PUSB", 1, 0, 0, len(payload)) + payload


def read_frame(s):
    first = exact(s, 4)
    if first == b"PUSB":
        version, flags, reserved, size = struct.unpack("<BBHI", exact(s, 8))
        if version != 1:
            raise ValueError("unsupported framing version")
    else:
        size = struct.unpack("<I", first)[0]  # Legacy length envelope.
    if size > 65536:
        raise ValueError("oversized application frame")
    return exact(s, size)


def controller(p):
    sizes = {3: 67, 9: 171}
    if len(p) < 2 or p[0] != 0x10 or sizes.get(p[1]) != len(p):
        raise ValueError("unsupported or truncated controller state")
    v = struct.unpack_from("<BB3f4fI2fQBIIBBBBHH", p)
    result = dict(version=v[1], position=v[2:5], quaternion_xyzw=v[5:9],
                  buttons=v[9], joystick=v[10:12], pose_ns=v[12], wand=v[13],
                  flags=v[14], session_ms=v[15], tracking_state=v[16],
                  tracking_reason=v[17], interaction_mode=v[18],
                  trackpad_active=bool(v[19] & 1), trackpad=(v[20]/65535, v[21]/65535))
    if result["interaction_mode"] != 0:
        raise ValueError("this example handles phone mode only")
    if p[1] == 9:
        result.update(trigger=p[111]/255, grip=p[112]/255)
    return result


def haptics(amplitude=128, hand_mask=3, duration_ms=100):
    if not (0 <= amplitude <= 255 and 0 <= hand_mask <= 3 and 0 <= duration_ms <= 5000):
        raise ValueError("invalid haptics arguments")
    return struct.pack("<BBBBI", 0x20, amplitude, 0, hand_mask, duration_ms)


def ios_loop(s):
    while True:
        p = read_frame(s)
        if p[:1] == b"\x10":
            print("controller", controller(p))
        elif p == b"\x7e":
            print("recenter requested; apply host calibration here")
        elif p == b"\x7f":
            print("kill requested; no action taken by this example")
        else:
            print("unhandled payload", p.hex())


class AndroidPhoneParser:
    """Feed bytes read from the accessory bulk IN endpoint, preserving this object."""
    def __init__(self):
        self.pending = bytearray()

    def feed(self, chunk):
        self.pending.extend(chunk)
        out = []
        while self.pending:
            kind = self.pending[0]
            if kind == 0x10:
                if len(self.pending) < 2:
                    break
                if self.pending[1] != 9:
                    raise ValueError("unsupported Android state version")
                size = 171
            elif kind == 0x7e:
                size = 2
            elif kind in (0x02, 0x7f):
                size = 1
            else:
                raise ValueError(f"unknown Android packet boundary: {kind:02x}")
            if len(self.pending) < size:
                break
            p = bytes(self.pending[:size])
            del self.pending[:size]
            if kind == 0x10:
                out.append(("state", controller(p)))
            elif kind == 0x7e:
                if p[1] not in (0, 1):
                    raise ValueError("invalid reset wand selector")
                out.append(("recenter", "left" if p[1] == 0 else "right"))
            else:
                out.append(("hangup" if kind == 2 else "kill", None))
        return out


if __name__ == "__main__":
    with connect(5006, sys.argv[1:], "usage: receiver.py [SERIAL] | tcp HOST PORT") as sock:
        ios_loop(sock)
```

For Android, the USB-access layer is platform-specific: after the AOA setup above, write `b"\x01"` to bulk OUT, repeatedly pass bulk IN reads into `AndroidPhoneParser.feed`, and dispatch the returned events. Write `haptics(...)` directly to bulk OUT. For iOS controller haptics use `sock.sendall(frame(haptics(...)))`. Use one serialized writer if multiple threads send commands.

### Small wire vectors

| Meaning | Hex bytes |
| --- | --- |
| Android host ready | `01` |
| Android left recenter request | `7e 00` |
| Android right recenter request | `7e 01` |
| iOS framed recenter request | `50 55 53 42 01 00 00 00 01 00 00 00 7e` |
| Raw 100 ms half-amplitude haptics | `20 80 00 03 64 00 00 00` |

For a zeroed controller sample, set quaternion W at offset 26 to float `1.0` (`00 00 80 3f`), type at 0 to `10`, and version at 1 to `03` or `09`; use the corresponding full packet length. For a neutral Android auxiliary quaternion also set W at offset 91 to `1.0`.

## Integration checklist and troubleshooting

- **Connected but no Android data:** verify all three accessory identity strings, permission, an open bulk read loop, and the initial raw `01`. Do not add PUSB framing.
- **iOS connection refused:** check the app is active, the device is trusted, and select port 5006 for the controller (5005 is the [tracker](TRACKER.md)). Select the right USB device; a second receiver can replace your iOS connection.
- **Controller floats are nonsense:** check the version byte at offset 1, packed little-endian layout, and quaternion order. The obsolete 54-byte controller description has different offsets.
- **Inputs stick after disconnection:** clear buttons and touch state when the connection ends or crosses your timeout. The protocol supplies no universally mandated timeout; choose one for your application and tolerate expected scheduling jitter.
- **Packets arrive but pose is frozen:** inspect tracking state and timestamp progression. State cadence and measurement cadence differ.
- **Reset has no effect:** distinguish direction, app, and encoding. Controller `7e` requests host calibration; Android's host-to-phone `7e` only marks the reset complete; iOS controller inbound `ORST` only logs.

The reference code validates current layouts, not all historical firmware. For production, reject nonfinite measurement values before using them, bound buffers, preserve frame boundaries, tolerate unknown flag bits, and keep a single parser state per connection. Release inputs on errors and discard partial parser buffers before reconnecting.

## Compatibility notes

The current layouts above were checked against the app implementations on **2026-09-15**. Receiver examples were checked with synthetic packets and fragmented streams; this document does not claim physical-device interoperability testing.

The old 54-byte/4 ms/BLE controller description does not describe the current apps; the Android phone's current 171-byte size is not optional. Use the packet versions and layouts documented here rather than assuming compatibility with older protocol descriptions.
