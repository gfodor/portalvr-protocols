# PortalVR protocols

Documentation for building receivers that use PortalVR's iOS camera tracker and iOS/Android phone controller apps.

Read **[PROTOCOLS.md](PROTOCOLS.md)** for connection setup, byte-level packet layouts, coordinate conventions, timing, calibration, haptics, and Python receiver examples.

| App | Connection | Data |
| --- | --- | --- |
| iOS tracker | USBMux → device TCP port 5005 | Hand/body anchors, optional hand landmarks, acquisition state |
| iOS phone controller | USBMux → device TCP port 5006 | Phone pose, buttons, joystick, tracking state |
| Android phone controller | Android Open Accessory USB | Phone pose, buttons, joystick, tracking state |

The controller apps share a common packet prefix but currently emit different versions and lengths. The protocol reference documents these differences; do not assume their packets are interchangeable.

This repository covers phone mode and the camera tracker. Headset mode and its transport are outside scope. It contains documentation and embedded receiver examples, not a desktop VR driver.

The reference was checked against the app implementations on September 15, 2026. Receiver examples were checked with synthetic packets and fragmented streams; physical-device interoperability has not been tested here. See the [compatibility notes](PROTOCOLS.md#compatibility-notes) for differences from older protocol descriptions.
