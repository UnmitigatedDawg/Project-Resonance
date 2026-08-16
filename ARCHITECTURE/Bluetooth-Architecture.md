# Bluetooth Architecture

**Version:** 0.1
**Status:** Draft
**Owner:** Product Team
**Last Updated:** 2026-08-16

**Related Documents:**

* Architecture-Overview.md
* Application-Architecture.md
* Data-Model.md
* Logging-Architecture.md
* Testing-Strategy.md

---

> **Bluetooth should feel invisible to the musician, even when the engineering behind it is not.**

Project Resonance uses Bluetooth Low Energy communication to connect the iPhone application to the NUX Mighty Plug Pro MP-3.

The Bluetooth architecture should isolate device discovery, connection management, protocol translation, and raw message handling from the rest of the application.

Views and ViewModels should never need to understand Bluetooth packets, characteristics, or device-specific command formats.

---

# Primary Objective

The Bluetooth layer should make the rest of Project Resonance think in musical concepts.

The application should be able to request:

* Connect to the MP-3.
* Load the current tone.
* Change an effect parameter.
* Enable or bypass an effect.
* Change the active hardware preset.
* Save a preset.
* Reorder supported effects.
* Disconnect.

The rest of the application should not need to know how those operations are encoded for the hardware.

Conceptually:

```text
Project Resonance
        ↓
Bluetooth Service
        ↓
MP-3 Protocol Layer
        ↓
Core Bluetooth
        ↓
NUX Mighty Plug Pro MP-3
```

---

# Research Basis

Project Resonance will use the open-source Mightier Amp project as the primary technical reference for understanding communication with the Mighty Plug Pro.

Relevant areas of the Mightier Amp codebase include:

```text
lib/bluetooth/

    ble_controllers/

    devices/

        NuxMightyPlugPro.dart

        communication/
            plugProCommunication.dart

        effects/

        presets/
```

Mightier Amp demonstrates a useful separation between:

* Generic Bluetooth transport.
* Device definitions.
* Device-specific communication.
* Effect definitions.
* Preset representations.

Project Resonance should preserve that separation while implementing it natively for iOS.

---

# Native iOS Bluetooth

Project Resonance will use Apple's Core Bluetooth framework for Bluetooth Low Energy communication.

The iPhone acts as the Bluetooth central.

The MP-3 acts as the peripheral.

Conceptually:

```text
iPhone
CBCentralManager
        ↓
CBPeripheral
        ↓
Services
        ↓
Characteristics
        ↓
NUX MP-3
```

Core Bluetooth should remain contained within the Bluetooth implementation layer.

No SwiftUI View should directly reference Core Bluetooth objects.

---

# Bluetooth Service

The application-facing interface should be represented by a Bluetooth Service.

Conceptually:

```text
BluetoothService

    scan()
    connect()
    disconnect()

    loadCurrentTone()
    loadPreset()
    savePreset()

    setEffectEnabled()
    setParameter()
    reorderPedalboard()

    connectionState
```

These names describe intent rather than Bluetooth implementation.

The precise Swift API will evolve during implementation.

---

# Proposed Internal Structure

The Bluetooth portion of the Swift project should approximately follow:

```text
Services/

    Bluetooth/

        BluetoothService.swift

        CoreBluetoothManager.swift

        MP3Device.swift

        MP3Protocol.swift

        MP3CommandEncoder.swift

        MP3ResponseDecoder.swift

        MP3ConnectionState.swift
```

Responsibilities should remain distinct.

---

# BluetoothService

`BluetoothService` is the interface used by ViewModels and other application components.

It should expose meaningful application operations without exposing low-level Core Bluetooth details.

For example:

```text
PlayModeViewModel
        ↓
BluetoothService
        ↓
MP3Protocol
        ↓
CoreBluetoothManager
```

The ViewModel should not know which characteristic receives a command or how a command is encoded.

---

# CoreBluetoothManager

`CoreBluetoothManager` owns Apple's Core Bluetooth interaction.

Responsibilities include:

* Bluetooth availability state.
* Scanning.
* Peripheral discovery.
* Connecting.
* Disconnecting.
* Service discovery.
* Characteristic discovery.
* Characteristic subscription.
* Reading data.
* Writing data.
* Receiving notifications.
* Detecting connection loss.

It should not understand Tone Profiles, Play Mode, or Tone Studio.

---

# MP-3 Device Definition

`MP3Device` describes hardware-specific characteristics of the NUX Mighty Plug Pro.

Initial research identifies the device as advertising under the name:

```text
MIGHTY PLUG PRO
```

Current reference information also identifies:

```text
Hardware preset channels: 7
Effects-chain positions: 9
```

These values should be verified against the physical device during prototype testing before being treated as permanent assumptions.

---

# MP-3 Protocol Layer

`MP3Protocol` represents the device communication rules above the generic Bluetooth transport.

It is responsible for translating between:

```text
Project Resonance Concepts
        ↕
MP-3 Commands and Responses
```

Examples include translating:

```text
Enable Delay
```

into the appropriate device command, or converting an incoming preset response into a Project Resonance `Tone`.

This layer is where most of the useful knowledge obtained from Mightier Amp should be adapted.

---

# Command Encoding

`MP3CommandEncoder` should convert application requests into device-compatible messages.

Examples may include:

* Request current preset.
* Change active channel.
* Change processor model.
* Change parameter value.
* Enable or bypass processor.
* Reorder processor.
* Save preset.

Raw byte construction should remain isolated here whenever practical.

The rest of Project Resonance should never build protocol packets manually.

---

# Response Decoding

`MP3ResponseDecoder` should interpret data received from the device.

It should transform raw responses into meaningful application information such as:

* Connection initialization information.
* Current preset.
* Processor state.
* Parameter values.
* Active hardware channel.
* Firmware information.
* Other supported device state.

Malformed or unexpected data should generate diagnostic information without crashing the application.

---

# Connection State

Bluetooth connection state should be explicit and observable.

Initial states should include:

```text
unavailable
disconnected
scanning
connecting
initializing
connected
reconnecting
failed
```

`initializing` is important.

A Bluetooth connection may technically exist before Project Resonance has discovered the required services, characteristics, and initial MP-3 state.

Play Mode should indicate readiness only when initialization is complete.

---

# Connection Lifecycle

The expected lifecycle is:

```text
Bluetooth Available
        ↓
Scan
        ↓
Discover MP-3
        ↓
Connect
        ↓
Discover Services
        ↓
Discover Characteristics
        ↓
Subscribe / Initialize
        ↓
Request Device State
        ↓
Ready
```

The musician should experience this as simply:

```text
Connecting…
        ↓
Connected
```

Technical complexity belongs inside the service.

---

# Reconnection

Temporary connection loss should not unnecessarily interrupt the musician.

When practical, Project Resonance should attempt graceful reconnection to the previously connected MP-3.

Reconnection behavior should:

* Avoid aggressive endless retry loops.
* Communicate meaningful state to the UI.
* Recover without requiring unnecessary user interaction.
* Preserve unsaved application state.

Exact retry behavior will be defined after testing with the physical device.

---

# Real-Time Editing

Tone Studio changes should feel immediate.

For controls such as knobs, Project Resonance may receive many value changes in rapid succession.

The Bluetooth architecture should therefore support efficient parameter updates without overwhelming the device.

Potential techniques include:

* Coalescing intermediate values.
* Throttling rapid updates when necessary.
* Sending the final value immediately when interaction ends.

Actual behavior should be determined through MP-3 testing rather than premature optimization.

---

# Pedal Bypass

Play Mode allows a musician to tap a pedal to enable or bypass it.

Conceptually:

```text
Pedal Tap
    ↓
PlayModeViewModel
    ↓
BluetoothService
    ↓
MP3Protocol
    ↓
MP-3
```

The UI should update quickly while remaining synchronized with actual device state.

Error handling should prevent the interface from silently displaying a state the hardware did not accept.

---

# Pedalboard Reordering

The MP-3 supports a reorderable signal chain.

Tone Studio will use long-press drag-and-drop to reorder pedals.

The Bluetooth layer must translate the resulting ordered Pedalboard into the appropriate hardware command or commands.

The domain model remains responsible for representing desired order.

The protocol layer remains responsible for communicating that order to the device.

---

# Preset Synchronization

Project Resonance distinguishes between:

```text
Tone Library
Unlimited application storage
```

and:

```text
MP-3 Hardware Presets
7 device channels
```

Bluetooth operations should therefore clearly distinguish:

* Loading a Tone into the application.
* Sending a Tone to the connected MP-3.
* Saving a Tone into a specific MP-3 hardware channel.
* Reading a hardware channel into Project Resonance.

Hardware storage should never define the size of the application's Tone Library.

---

# Source of Truth

While actively connected, device state and application state must remain synchronized.

The precise source-of-truth strategy will be refined during implementation.

As a starting principle:

* User intent updates application state.
* The Bluetooth layer sends the corresponding command.
* Device responses confirm or correct state when available.
* Unexpected external device changes should propagate back into the application.

Project Resonance should avoid allowing the UI and MP-3 to silently disagree.

---

# Error Handling

Bluetooth failures are expected operating conditions, not exceptional programming failures.

Examples include:

* Bluetooth disabled.
* MP-3 powered off.
* Device out of range.
* Connection interrupted.
* Unsupported firmware response.
* Command rejected or not acknowledged.
* Unexpected packet.

Errors should be translated into understandable application states.

The musician should not see raw Bluetooth error codes unless diagnostic logging is intentionally enabled.

---

# Logging

The Bluetooth layer should provide detailed diagnostic logging during development.

Useful events include:

* Scan started.
* Device discovered.
* Connection attempted.
* Connection established.
* Services discovered.
* Characteristics discovered.
* Commands transmitted.
* Responses received.
* Decode failures.
* Disconnect reason.
* Reconnection attempts.

Sensitive or unnecessarily verbose data should not be logged in production.

---

# Testing Without Hardware

The Bluetooth architecture should support a simulated implementation.

Conceptually:

```text
BluetoothService
        ↑
        |
-----------------
|               |
Real            Mock
Bluetooth       Bluetooth
Service         Service
```

A mock service will allow:

* SwiftUI development before hardware is connected.
* ViewModel tests.
* Connection-state simulation.
* Failure testing.
* Faster development.

The physical MP-3 remains essential for protocol validation and integration testing.

---

# Battery Status

Battery reporting for the Mighty Plug Pro should not be assumed.

Current Mightier Amp device definitions do not advertise battery support for this model.

Therefore:

* Play Mode will retain compact connection status.
* Battery percentage will be displayed only if we verify that reliable battery information is available to Project Resonance.
* Lack of battery reporting will not block Version 1.0.

This decision should be revisited during hardware testing.

---

# Firmware Compatibility

Device behavior may vary by firmware version.

Project Resonance should detect or record firmware information when available and avoid assuming all MP-3 firmware revisions behave identically.

Protocol differences should remain contained within the MP-3 protocol layer rather than spreading throughout the application.

---

# Mightier Amp Usage Strategy

Project Resonance will learn from the Mightier Amp implementation rather than porting its entire architecture.

We should identify:

* BLE discovery behavior.
* Device identification.
* Service and characteristic usage.
* MP-3 command formats.
* Response formats.
* Preset encoding.
* Effect parameter mapping.
* Signal-chain reordering behavior.
* Firmware-specific behavior.

That knowledge can then be represented cleanly in native Swift architecture.

This approach allows Project Resonance to benefit from existing open-source knowledge while preserving its own product and engineering design.

---

# Decisions

* Apple Core Bluetooth will provide the native BLE transport.
* Bluetooth implementation remains behind a `BluetoothService` abstraction.
* Core Bluetooth objects do not enter SwiftUI Views.
* Device-specific protocol handling remains separate from generic BLE transport.
* Command encoding and response decoding remain isolated from application UI.
* Explicit connection states will be maintained.
* A mock Bluetooth service will support development and testing without hardware.
* Hardware preset storage remains separate from the Tone Library.
* MP-3 battery display is conditional upon confirmed hardware support.
* Mightier Amp will be used as a technical reference for MP-3 protocol behavior.

---

## Open Questions

* Which BLE service UUIDs and characteristic UUIDs does the MP-3 use?
* Which characteristics are used for command writes and device notifications?
* What initialization exchange is required after connection?
* What acknowledgement behavior exists for commands?
* How are complete presets encoded and transferred?
* How exactly is signal-chain reordering encoded?
* How should rapid knob changes be throttled or coalesced?
* What firmware differences must Version 1.0 support?
* Can reliable battery status be retrieved by another supported mechanism?
* What reconnect strategy provides the best balance between convenience and battery use?

---

## Next Research Step

Before implementing `BluetoothService.swift`, document the MP-3 protocol discovered in the Mightier Amp source.

The next research should focus on:

```text
NuxMightyPlugPro.dart
plugProCommunication.dart
BLEController.dart
MightyBle.dart
PlugProPreset.dart
```

The goal is to identify the precise BLE services, characteristics, initialization sequence, command formats, and response formats required for Prototype 0.1.

---

## Future Ideas

Potential future Bluetooth capabilities include:

* Support for additional NUX hardware.
* Background reconnection where appropriate.
* Bluetooth diagnostics screen.
* Protocol trace export for troubleshooting.

Future capabilities should not complicate the Version 1.0 architecture unless they provide immediate value.

---

## Related Documents

* Architecture Overview
* Application Architecture
* Data Model
* Logging Architecture
* Testing Strategy

---

*Project Resonance • Architecture • Version 0.1 • Status: Draft*
