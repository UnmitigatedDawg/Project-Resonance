# MP-3 Protocol Research

**Version:** 0.1
**Status:** Draft
**Owner:** Engineering Team
**Last Updated:** 2026-08-16

**Related Documents:**

* Bluetooth-Architecture.md
* Application-Architecture.md
* Data-Model.md
* Testing-Strategy.md
* Logging-Architecture.md

---

> **Understand the protocol before implementing the protocol.**

This document records technical research into Bluetooth communication with the NUX Mighty Plug Pro MP-3.

The primary research source is the open-source Mightier Amp project.

The purpose of this document is to separate three kinds of knowledge:

1. **Verified from source** — behavior directly represented in Mightier Amp.
2. **Project Resonance interpretation** — how we currently intend to represent that behavior.
3. **Hardware validation required** — behavior that must still be confirmed using the physical MP-3.

This distinction is important because the Mightier Amp implementation is an extremely valuable technical reference, but Project Resonance should verify critical assumptions against the actual hardware before treating them as permanent protocol specifications.

---

# Research Sources

The primary Mightier Amp source files relevant to the MP-3 include:

```text
lib/bluetooth/ble_controllers/BLEController.dart

lib/bluetooth/devices/NuxDeviceControl.dart

lib/bluetooth/devices/NuxMightyPlugPro.dart

lib/bluetooth/devices/NuxConstants.dart

lib/bluetooth/devices/communication/communication.dart

lib/bluetooth/devices/communication/plugProCommunication.dart

lib/bluetooth/devices/presets/PlugProPreset.dart
```

Additional effect and processor definitions will be examined as the protocol implementation progresses.

---

# Device Identity

## Verified from Source

The Mighty Plug Pro device definition identifies the expected Bluetooth advertising name as:

```text
MIGHTY PLUG PRO
```

The same definition identifies:

```text
Hardware preset channels: 7
Effects-chain positions: 9
```

The product is represented internally by Mightier Amp as:

```text
NUX Mighty Plug Pro
```

The device definition also reports battery support as unavailable.

## Project Resonance Interpretation

Version 1.0 should initially scan specifically for the Mighty Plug Pro while keeping device identification isolated enough to permit future NUX hardware support.

The application should represent seven hardware preset slots separately from its unlimited local Tone Library.

The application should represent nine signal-processing positions in the MP-3 pedalboard.

## Hardware Validation Required

Confirm:

* The physical MP-3 advertises exactly as `MIGHTY PLUG PRO`.
* Seven hardware channels are available.
* Nine signal-chain positions are available.
* Battery status cannot be reliably obtained through another supported mechanism.

---

# BLE Transport

## Verified from Source

Mightier Amp defines the Bluetooth MIDI service UUID as:

```text
03b80e5a-ede8-4b33-a751-6ce34ec4c700
```

The BLE MIDI characteristic UUID is:

```text
7772e5db-3868-4112-a1a9-f2669d106bf3
```

These are used as the transport for communication with supported NUX amplifiers.

The BLE controller maintains an outgoing message queue and removes older queued Control Change messages when a newer message for the same controller arrives.

## Project Resonance Interpretation

Project Resonance should use Apple's Core Bluetooth framework to:

1. Scan for the MP-3.
2. Connect to the discovered peripheral.
3. Discover the BLE MIDI service.
4. Discover the BLE MIDI characteristic.
5. Subscribe to notifications.
6. Write outgoing messages through that characteristic.
7. Decode incoming MIDI-style messages.

The exact Core Bluetooth write type should be confirmed during implementation.

## Hardware Validation Required

Confirm:

* The MP-3 exposes the expected BLE MIDI service.
* The expected characteristic supports the required notification behavior.
* The characteristic accepts the write mode selected by the iOS implementation.
* Packet-size behavior is reliable on current MP-3 firmware.

---

# Message Transport Prefix

## Verified from Source

Mightier Amp constructs Control Change messages using two leading BLE MIDI bytes:

```text
0x80 0x80
```

A Control Change message is then represented conceptually as:

```text
0x80
0x80
CONTROL_CHANGE
CONTROL_NUMBER
VALUE
```

Program Change messages use the same two-byte BLE MIDI prefix:

```text
0x80
0x80
PROGRAM_CHANGE
PROGRAM_NUMBER
```

## Project Resonance Interpretation

The two leading bytes should be treated as BLE MIDI transport framing rather than part of the NUX application-level command itself.

Encoding should therefore be separated conceptually:

```text
Application Command
        ↓
NUX / MIDI Message
        ↓
BLE MIDI Framing
        ↓
Core Bluetooth
```

---

# Control Change Messages

## Verified from Source

Real-time effect and parameter operations commonly use MIDI Control Change messages.

The generic message structure is:

```text
[0x80, 0x80, ControlChange, controller, value]
```

Examples of operations implemented through Control Change include:

* Parameter changes.
* Processor enable/bypass.
* Processor selection.
* Master volume.
* Active preset range.
* Drum controls.
* Various device settings.

## Project Resonance Interpretation

Most real-time Tone Studio knob movements will probably become Control Change operations.

Project Resonance should expose meaningful methods such as:

```text
setParameter(...)
setEffectEnabled(...)
```

rather than exposing Control Change numbers to ViewModels.

The MP-3 protocol layer will translate application intent into the appropriate controller/value pair.

---

# Program Change Messages

## Verified from Source

The Plug Pro changes hardware channels using MIDI Program Change.

Conceptually:

```text
[0x80, 0x80, ProgramChange, channel]
```

Mightier Amp's Plug Pro implementation returns a Program Change message when selecting a channel.

## Project Resonance Interpretation

Selecting an MP-3 hardware preset should eventually translate into a Program Change operation.

Tone Library selection and hardware-channel selection remain separate application concepts.

---

# SysEx Messages

## Verified from Source

The Plug Pro uses a device-specific SysEx message format.

Mightier Amp constructs Plug Pro SysEx messages using the following beginning:

```text
0x80
0x80
SysExStart
0x43
0x58
privacy
messageType
direction
```

The payload follows these fields.

The message is terminated using the project's BLE MIDI/SysEx termination representation.

Conceptually:

```text
BLE MIDI Prefix
        ↓
SysEx Start
        ↓
0x43 0x58
        ↓
Privacy
        ↓
Message Type
        ↓
Direction
        ↓
Payload
        ↓
SysEx End
```

## Project Resonance Interpretation

SysEx construction should be centralized in the MP-3 protocol encoder.

No ViewModel, View, or general-purpose Service should manually construct these messages.

---

# Firmware Request

## Verified from Source

Mightier Amp creates a firmware request during connection initialization.

The firmware request uses a short public SysEx-style message beginning with the Plug Pro identifier bytes:

```text
0x43
0x58
```

Firmware information is requested before the remaining connection initialization sequence proceeds.

## Project Resonance Interpretation

Prototype 0.1 should include firmware discovery early in the connection process.

Firmware information will help us identify possible protocol differences between MP-3 firmware revisions.

---

# Full Mightier Amp Initialization

## Verified from Source

The Plug Pro communication implementation declares six connection steps after initial firmware communication.

The sequence is:

```text
Step 0
Request presets

Step 1
Request current channel

Step 2
Request custom IR names

Step 3
Request system settings

Step 4
Request drum data

Step 5
Request microphone/current-state settings
```

Preset retrieval begins by requesting preset index 0 and proceeds through the device's preset set.

## Project Resonance Interpretation

Project Resonance Version 1.0 may eventually need much of this initialization.

Prototype 0.1 does not.

We should deliberately minimize the first prototype handshake.

---

# Prototype 0.1 Initialization

The first Project Resonance hardware prototype should attempt the smallest useful connection sequence:

```text
Discover MIGHTY PLUG PRO
        ↓
Connect
        ↓
Discover BLE MIDI Service
        ↓
Discover BLE MIDI Characteristic
        ↓
Enable Notifications
        ↓
Request Firmware
        ↓
Request Preset 0
        ↓
Receive Preset Data
        ↓
Decode Minimum Required Fields
        ↓
Connection Ready
```

## Prototype Success Criterion

Prototype 0.1 succeeds when Project Resonance can:

1. Discover the user's MP-3.
2. Connect reliably.
3. Receive data from the device.
4. Request one real preset.
5. Decode enough information to demonstrate that the preset is understood.
6. Display successful connection and retrieved tone information.

This is intentionally narrower than reproducing the entire Mightier Amp startup sequence.

---

# Preset Requests

## Verified from Source

Preset retrieval uses Plug Pro SysEx messages.

A preset request is constructed conceptually as:

```text
Privacy: Private
Message Type: PRESET
Direction: Request
Payload: preset index
```

The current hardware channel is requested separately using the current-preset message type.

## Project Resonance Interpretation

The protocol layer should expose higher-level operations such as:

```text
requestPreset(index)
requestCurrentChannel()
```

The caller should not need to understand SysEx message types.

---

# Saving a Hardware Preset

## Verified from Source

Mightier Amp saves the current configuration into a hardware preset index using a device-specific command containing:

```text
save command
preset index
```

## Project Resonance Interpretation

Project Resonance should clearly distinguish:

```text
Save Tone to Library
```

from:

```text
Send / Save Tone to MP-3 Slot
```

The former is application persistence.

The latter is a hardware operation.

---

# Effect Selection and Bypass

## Verified from Source

For the Plug Pro, Mightier Amp combines processor selection and enabled/bypassed state into a value.

The effect index occupies the lower bits.

A `0x40` bit is used in constructing the disabled/bypassed state.

Conceptually:

```text
Enabled:
effectIndex

Bypassed:
effectIndex | 0x40
```

Incoming device data is similarly masked to recover the selected effect index.

## Project Resonance Interpretation

Project Resonance should represent these separately:

```text
Pedal.model
Pedal.enabled
```

The protocol layer is responsible for combining or separating those values when communicating with the MP-3.

This keeps the domain model musical and understandable.

---

# Play Mode Pedal Toggle

The protocol research supports the Play Mode interaction already selected by Product Design:

```text
Tap Pedal
    ↓
Toggle enabled state
    ↓
Encode MP-3 Control Change
    ↓
Send to device
    ↓
Reflect confirmed state
```

The user should never need to enter Tone Studio simply to bypass or engage a pedal.

---

# Hardware Channel Selection

## Verified from Source

The Plug Pro uses Program Change messages for hardware-channel selection.

## Project Resonance Interpretation

The Bluetooth Service should eventually expose:

```text
selectHardwarePreset(slot)
```

rather than:

```text
sendProgramChange(...)
```

Protocol terminology should remain inside the protocol layer.

---

# Signal-Chain Reordering

## Verified from Source

Mightier Amp explicitly supports processor reordering on the Plug Pro.

When sending a new order, it builds a `MODULELINK` SysEx message.

The payload begins with the number of processors and is followed by the processor identifiers in the desired order.

Conceptually:

```text
MODULELINK SET

[
    processorCount,
    processorID,
    processorID,
    processorID,
    ...
]
```

Mightier Amp also records the time of an outgoing reorder because the amplifier sends order information back, and the application needs to distinguish its own recently initiated update from other order information.

## Project Resonance Interpretation

This confirms that our Tone Studio interaction is technically grounded:

```text
Long Press Pedal
        ↓
Lift
        ↓
Drag
        ↓
Other Pedals Move
        ↓
Drop
        ↓
Update Pedalboard Order
        ↓
Send MODULELINK Update
```

The domain model should remain an ordered collection.

The MP-3 protocol layer translates that collection into hardware processor identifiers.

---

# Rapid Parameter Updates

## Verified from Source

Mightier Amp queues outgoing BLE data.

When a new Control Change message is queued, it removes older queued Control Change messages for the same controller.

This effectively prevents stale intermediate values from accumulating during rapid control movement.

## Project Resonance Interpretation

This behavior is highly relevant to knob interaction.

For example, dragging Gain quickly from:

```text
31 → 32 → 33 → 34 → 35 → 36
```

does not necessarily require every intermediate value to reach the MP-3.

Project Resonance should consider a latest-value-wins strategy for rapidly repeated updates to the same parameter.

The final behavior should be validated against the physical device.

---

# Incoming Message Handling

## Verified from Source

Mightier Amp distinguishes incoming message families including:

* SysEx.
* Program Change.
* Control Change.

Program Change can update the selected hardware channel.

Control Change can update:

* Processor state.
* Processor selection.
* Parameters.
* Other device settings.

## Project Resonance Interpretation

Incoming data should flow through:

```text
Core Bluetooth
      ↓
BLE MIDI Decoder
      ↓
MP-3 Response Decoder
      ↓
Domain Update
      ↓
ViewModel
      ↓
View
```

The UI should never parse raw incoming bytes.

---

# Preset Representation

## Verified from Source

Mightier Amp's Plug Pro preset implementation maintains:

* Selected effects.
* Enabled/bypassed states.
* Effect parameter values.
* Master/channel volume.
* Processor order.

Mightier Amp's JSON representation also stores processor data using musical/effect-oriented keys rather than exposing raw packets to the rest of the application.

## Project Resonance Interpretation

This strongly supports the separation already defined in `Data-Model.md`:

```text
Tone
    ↓
Pedalboard
    ↓
Pedal
    ↓
Parameters
```

The Bluetooth representation and the Project Resonance representation should remain separate.

---

# Hardware Preset Count

## Verified from Source

The current Mighty Plug Pro device definition reports:

```text
channelsCount = 7
```

## Project Resonance Interpretation

Version 1.0 should currently assume seven MP-3 hardware slots, while keeping the value device-specific rather than hard-coding it throughout the application.

This resolves one of the open questions in the initial Data Model draft, subject to physical-device confirmation.

---

# Effects-Chain Count

## Verified from Source

The current Mighty Plug Pro device definition reports:

```text
effectsChainLength = 9
```

## Project Resonance Interpretation

This supports the nine-category Tone Studio design.

The UI should not duplicate the number nine throughout the code.

The device definition should provide supported processor information to the rest of the application.

---

# Battery Status

## Verified from Source

The Mighty Plug Pro definition in Mightier Amp reports battery support as false.

The Plug Pro communication class contains battery-request code but marks it as incorrect and exits early when battery support is unavailable.

## Project Resonance Interpretation

Battery percentage is not currently a guaranteed Version 1.0 capability.

Play Mode will show connection status regardless.

Battery information will appear only if hardware testing identifies a reliable supported method.

---

# What We Know With High Confidence

Based on current source research:

* Device BLE name is `MIGHTY PLUG PRO`.
* BLE communication uses the BLE MIDI service and characteristic documented above.
* The device has seven hardware preset channels.
* The device exposes a nine-position effects chain.
* Hardware channel selection uses Program Change.
* Many real-time changes use Control Change.
* Larger Plug Pro operations use device-specific SysEx.
* Presets can be requested individually.
* Hardware presets can be saved.
* Effect enable/bypass is encoded alongside processor selection.
* Signal-chain reordering is explicitly supported.
* Reordering uses `MODULELINK`.
* Outgoing real-time parameter messages benefit from queue coalescing.
* Battery reporting should not currently be assumed.

---

# What Still Requires Hardware Validation

The following should remain open until tested on the physical MP-3:

* Exact Core Bluetooth discovery behavior on iOS.
* Notification subscription behavior.
* Preferred characteristic write type.
* Firmware response format.
* Complete preset response format.
* Packet fragmentation behavior.
* Timing requirements between requests.
* Whether the reduced Prototype 0.1 handshake is sufficient.
* Parameter-update throughput.
* Reorder acknowledgement timing.
* Reconnection behavior.
* Firmware-specific differences.
* Battery reporting possibilities.

---

# Implementation Priorities

Protocol implementation should proceed in this order:

## P0 — Connection

```text
Scan
Connect
Discover Service
Discover Characteristic
Subscribe
Receive Data
```

## P1 — Firmware

```text
Request Firmware
Receive Firmware
Decode Firmware
```

## P2 — First Preset

```text
Request Preset 0
Receive Preset
Decode Minimum Preset Data
```

## P3 — Real-Time Control

```text
Change One Parameter
Toggle One Pedal
Change Hardware Channel
```

## P4 — Full Tone

```text
Decode All Nine Processors
Decode Parameters
Decode Bypass States
Decode Order
```

## P5 — Write Operations

```text
Send Full Tone
Reorder Pedalboard
Save Hardware Preset
```

This sequence deliberately prioritizes learning and risk reduction over feature completeness.

---

# Prototype 0.1 Technical Definition of Done

The Bluetooth/protocol portion of Prototype 0.1 is complete when:

* The iPhone discovers the MP-3.
* Project Resonance connects.
* The BLE MIDI transport initializes.
* The app receives data from the MP-3.
* The app retrieves one real hardware preset.
* At least one meaningful preset property is decoded correctly.
* At least one parameter can be changed from Project Resonance and heard on the connected MP-3.
* Disconnect occurs cleanly.
* Failures generate useful diagnostic logging.

---

# Research Discipline

Whenever new protocol behavior is documented, classify it as:

```text
VERIFIED FROM SOURCE
```

```text
VERIFIED ON HARDWARE
```

```text
PROJECT RESONANCE INTERPRETATION
```

or:

```text
OPEN QUESTION
```

Source research alone should not silently become hardware truth.

Physical testing alone should not become architectural truth without documentation.

Both forms of evidence matter.

---

# Decisions

* Mightier Amp will serve as the primary initial technical reference.
* MP-3 protocol knowledge will be documented before native Swift implementation.
* Verified source behavior will be distinguished from Project Resonance interpretation.
* Critical protocol behavior will be confirmed on physical hardware.
* Prototype 0.1 will use a deliberately reduced initialization sequence.
* Protocol translation will remain separate from the domain model.
* Seven hardware preset channels are the current working device definition.
* Nine effect-chain positions are the current working device definition.
* Signal-chain reordering is confirmed as a supported protocol capability.
* Parameter-message coalescing will be evaluated for rapid knob interaction.
* Battery status remains optional pending hardware verification.

---

## Open Questions

* What exact binary structure is returned for a Plug Pro preset?
* How are all nine processors mapped to device processor identifiers?
* What are the complete Control Change mappings for every processor and parameter?
* What firmware-response fields should Project Resonance retain?
* How does Core Bluetooth deliver longer SysEx responses on iOS?
* Does the MP-3 require timing delays between specific messages?
* Can Prototype 0.1 omit IR, drum, system, and microphone initialization without side effects?
* What acknowledgement behavior should our command queue expect?
* What is the safest strategy for optimistic UI updates?
* Can battery percentage be retrieved reliably despite Mightier Amp marking support unavailable?

---

# Next Step

The next engineering task is to translate the P0 and P1 research into a minimal native Swift design.

That work should define the first implementation contracts for:

```text
BluetoothService
CoreBluetoothManager
MP3Protocol
MP3CommandEncoder
MP3ResponseDecoder
```

The first implementation should remain intentionally small.

The goal is not to implement the entire NUX protocol.

The goal is to establish successful two-way communication with the physical MP-3 as quickly and safely as possible.

---

*Project Resonance • MP-3 Protocol Research • Version 0.1 • Status: Draft*

