# PortalVR protocols and plugins

Public integration documentation for PortalVR phone/tracker receivers and output plugins.

| Reference | Scope |
| --- | --- |
| [PROTOCOLS.md](PROTOCOLS.md) | Phone/controller and camera-tracker transport protocols |
| [PLUGINS.md](PLUGINS.md) | Authoritative Windows x64 SteamVR output-plugin ABI, installation, lifecycle and SDK-adapter guide |

Output plugins let independent integrators add calibrated projector warp/blend
using their own licensed vendor SDK, without rebuilding PortalVR. The plugin
reference includes the complete ABI 1.0 header and explains how to use the
SDK-free developer kit supplied with compatible PortalVR builds. Licensed SDK and physical projector
validation remain deployment requirements.

Read **[PROTOCOLS.md](PROTOCOLS.md)** for connection setup, byte-level packet layouts, coordinate conventions, timing, calibration, haptics, and Python receiver examples.

| App | Connection | Data |
| --- | --- | --- |
| iOS tracker | USBMux → device TCP port 5005 | Hand/body anchors, optional hand landmarks, acquisition state |
| iOS phone controller | USBMux → device TCP port 5006 | Phone pose, buttons, joystick, tracking state |
| Android phone controller | Android Open Accessory USB | Phone pose, buttons, joystick, tracking state |

The controller apps share a common packet prefix but currently emit different versions and lengths. The protocol reference documents these differences; do not assume their packets are interchangeable.

The transport reference covers phone mode and the camera tracker. Headset mode
and its transport are outside scope. The plugin reference covers one mono canvas
and one Windows desktop output target, including a Surround/Eyefinity group.
This repository contains documentation and embedded examples; PortalVR supplies
the host implementation and standalone plugin developer kit separately.

The transport reference was checked against the app implementations on September 15, 2026. Receiver examples were checked with synthetic packets and fragmented streams; physical-device interoperability has not been tested here. See the [compatibility notes](PROTOCOLS.md#compatibility-notes) for differences from older protocol descriptions.

The plugin reference was checked against the ABI 1.0 implementation on November
14, 2026. Its [validation record](PLUGINS.md#compatibility-and-validation-record)
distinguishes automated host coverage from the live SDK/projector checks still
required for deployment.
