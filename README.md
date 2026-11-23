# PortalVR protocols and plugins

Public integration documentation for PortalVR's phone controllers, camera tracker, MCP server, and output plugins.

| Reference | Scope |
| --- | --- |
| [CONTROLLER.md](CONTROLLER.md) | iOS and Android phone-controller transports, state packets, commands, and a Python receiver |
| [TRACKER.md](TRACKER.md) | iOS camera-tracker `POSE` packets, clock synchronization, yaw calibration, and a Python receiver |
| [MCP.md](MCP.md) | Local Model Context Protocol server for SteamVR applications: setup, transports, security, and the 25 built-in tools |
| [OUTPUT_PLUGINS.md](OUTPUT_PLUGINS.md) | Authoritative Windows x64 SteamVR output-plugin ABI, installation, lifecycle, and SDK-adapter guide |

## Phone controllers and camera tracker

| App | Install | Connection | Data |
| --- | --- | --- | --- |
| iOS camera tracker | [portalvr.io/tracker](https://portalvr.io/tracker) | USBMux → device TCP port 5005 | Hand/body anchors, optional hand landmarks, acquisition state |
| iOS phone controller | [portalvr.io/controller](https://portalvr.io/controller) | USBMux → device TCP port 5006 | Phone pose, buttons, joystick, tracking state |
| Android phone controller | [portalvr.io/controller](https://portalvr.io/controller) | Android Open Accessory USB | Phone pose, buttons, joystick, tracking state |

These are the links PortalVR's setup QR codes open. [CONTROLLER.md](CONTROLLER.md) and [TRACKER.md](TRACKER.md) cover connection setup, byte-level packet layouts, coordinate conventions, timing, calibration, haptics, and Python receiver examples. The two iOS apps share the USBMux connection and `PUSB` framing described in CONTROLLER.md.

The controller apps share a common packet prefix but currently emit different versions and lengths; do not assume their packets are interchangeable. Headset mode and its transport are outside scope.

The transport references were checked against the app implementations on September 15, 2026. Receiver examples were checked with synthetic packets and fragmented streams; physical-device interoperability has not been tested here. See the [controller](CONTROLLER.md#compatibility-notes) and [tracker](TRACKER.md#compatibility-notes) compatibility notes for differences from older protocol descriptions.

## MCP server

PortalVR's SteamVR driver includes a loopback-only MCP server that lets AI agents and test harnesses observe and drive OpenVR and OpenXR applications running on SteamVR: read poses, inject controller input and poses, capture the composited eye image, and call application-registered tools. Clients connect through the bundled stdio proxy, Streamable HTTP, or legacy HTTP+SSE. The tool surface follows the Meta XR Operator tool names and arguments. [MCP.md](MCP.md) was checked against the implementation on November 22, 2026.

## Output plugins

Output plugins let independent integrators add calibrated projector warp/blend using their own licensed vendor SDK, without rebuilding PortalVR. The plugin reference includes the complete ABI 1.0 header and explains how to use the SDK-free developer kit supplied with compatible PortalVR builds. It covers one mono canvas and one Windows desktop output target, including a Surround/Eyefinity group. Licensed SDK and physical projector validation remain deployment requirements.

The plugin reference was checked against the ABI 1.0 implementation on November 14, 2026. Its [validation record](OUTPUT_PLUGINS.md#compatibility-and-validation-record) distinguishes automated host coverage from the live SDK/projector checks still required for deployment.

This repository contains documentation and embedded examples; PortalVR supplies the host implementation, apps, and standalone plugin developer kit separately.
