---
title: "Application Architecture"
version: "0.1"
status: "Draft"
owner: "Product Team"
last_updated: "2026-08-16"
related_documents:
  - "Architecture-Overview.md"
  - "Data-Model.md"
  - "Navigation-Architecture.md"
  - "Bluetooth-Architecture.md"
  - "Dependency-Injection.md"
  - "Engineering-Standards.md"
---

# Application Architecture

> **Simple enough to understand. Structured enough to grow.**

Project Resonance uses a lightweight MVVM architecture with a dedicated Service Layer.

The architecture separates the user experience, application behavior, domain data, and hardware communication so that each component has a clear responsibility.

The goal is not architectural complexity.

The goal is to make Project Resonance easy to understand, test, maintain, and evolve.

---

# Architectural Model

At the highest level, Project Resonance follows this flow:

```text
Musician
    ↓
Views
    ↓
ViewModels
    ↓
Services
    ↓
Models / Device Communication
    ↓
NUX Mighty Plug Pro MP-3
```

Information also flows in the opposite direction as device state and application state change.

```text
NUX Mighty Plug Pro MP-3
    ↓
Services
    ↓
Models / Application State
    ↓
ViewModels
    ↓
Views
    ↓
Musician
```

The user interface should never need to understand the details of Bluetooth communication.

Likewise, the Bluetooth layer should never need to understand how information is presented on screen.

---

# Application Layer

The Application Layer establishes and coordinates the application as a whole.

Responsibilities include:

* Application startup
* Dependency creation
* Root navigation
* Application lifecycle
* Shared application state

The Application Layer should remain small.

It coordinates the major pieces of Project Resonance without becoming a container for product logic.

---

# View Layer

Views are implemented using SwiftUI.

Views are responsible for presenting information and receiving user interaction.

Examples include:

* Play Mode
* Tone Studio
* Tone Profile
* Settings
* Device connection status

Views may:

* Display state supplied by a ViewModel.
* Send user actions to a ViewModel.
* Perform presentation-specific animation.
* Control layout and visual appearance.

Views should not:

* Communicate directly with the MP-3.
* Perform Bluetooth operations.
* Persist tones directly.
* Contain significant product logic.

A View should primarily answer:

> **What should the musician see right now?**

---

# ViewModel Layer

ViewModels connect the user experience to application behavior.

A ViewModel receives user intentions from a View, coordinates the necessary work, and exposes the resulting state back to the View.

Examples may include:

* `PlayModeViewModel`
* `ToneStudioViewModel`
* `ToneProfileViewModel`
* `SettingsViewModel`

A ViewModel may:

* Turn an effect on or off.
* Change the selected tone.
* Request that a tone be loaded.
* Reorder the signal chain.
* Track whether a Tone Profile has unsaved changes.
* Request communication with the MP-3 through a Service.

ViewModels should not contain low-level Bluetooth implementation details.

A ViewModel should primarily answer:

> **What should happen when the musician does this?**

---

# Service Layer

Services provide capabilities to the rest of the application.

Initial services are expected to include:

## Bluetooth Service

Responsible for:

* Device discovery
* Connection
* Disconnection
* Communication with the MP-3
* Connection state
* Sending commands
* Receiving device responses

## Persistence Service

Responsible for:

* Saving tones
* Loading tones
* Saving Tone Profiles
* Managing snapshots
* Preserving user-created information

## Settings Service

Responsible for:

* Application preferences
* Keep-screen-awake preference
* Other user-configurable application behavior

## Logging Service

Responsible for:

* Diagnostic information
* Connection events
* Errors
* Development troubleshooting

Additional services should be introduced only when a clear responsibility requires them.

A Service should primarily answer:

> **What capability does the application need performed?**

---

# Model Layer

Models represent the domain of Project Resonance.

Initial models are expected to include:

* Tone
* Tone Profile
* Pedal
* Signal Chain
* Amplifier
* Cabinet
* Device
* Connection State
* Snapshot
* User Preferences

Models describe information and relationships without depending on the user interface.

The detailed model design is maintained in `Data-Model.md`.

---

# Feature Organization

Although Project Resonance uses MVVM, the codebase should also remain understandable from a product perspective.

Major user-facing features should be recognizable within the project structure.

Initial features include:

```text
Features/

    PlayMode/

    ToneStudio/

    ToneProfile/

    Settings/
```

Each feature may contain the Views and ViewModels that belong specifically to that experience.

Shared Models and Services should remain outside individual features when they are used across the application.

This provides two useful ways to understand the code:

* **By feature** — what part of the product does this support?
* **By responsibility** — is this a View, ViewModel, Model, or Service?

---

# Proposed Project Structure

The initial Swift project should approximately follow this structure:

```text
ProjectResonance/

    App/
        ProjectResonanceApp.swift
        AppEnvironment.swift

    Features/
        PlayMode/
            PlayModeView.swift
            PlayModeViewModel.swift

        ToneStudio/
            ToneStudioView.swift
            ToneStudioViewModel.swift

        ToneProfile/
            ToneProfileView.swift
            ToneProfileViewModel.swift

        Settings/
            SettingsView.swift
            SettingsViewModel.swift

    Models/
        Tone.swift
        ToneProfile.swift
        Pedal.swift
        SignalChain.swift
        Amplifier.swift
        Cabinet.swift
        Device.swift
        ConnectionState.swift
        Snapshot.swift
        UserPreferences.swift

    Services/
        Bluetooth/
        Persistence/
        Settings/
        Logging/

    Resources/

    Tests/
```

This structure is a starting point rather than a permanent constraint.

New folders or abstractions should be introduced only when the code demonstrates a genuine need for them.

---

# Dependency Direction

Dependencies should generally flow inward from presentation toward reusable application capabilities.

```text
View
  ↓
ViewModel
  ↓
Service
  ↓
Model / External System
```

Views may know about their ViewModels and Models used for presentation.

ViewModels may know about Models and Service interfaces.

Services may know about Models and external technologies such as Core Bluetooth.

Lower-level components should not depend on higher-level user interface components.

In particular:

> **Bluetooth code must never depend on Play Mode or Tone Studio.**

This separation allows hardware communication to evolve independently from the interface.

---

# State Management

SwiftUI should remain the primary mechanism for presenting application state.

State should have a clear owner.

Whenever practical:

* Views own temporary presentation state.
* ViewModels own feature behavior and feature state.
* Services own service-specific state.
* Models represent domain state.

Project Resonance should avoid multiple competing sources of truth for the same information.

---

# Concurrency

Bluetooth communication and other asynchronous work should not block the user interface.

Project Resonance will prefer modern Swift concurrency using:

* `async`
* `await`
* `Task`

Concurrency details will be refined as the Bluetooth architecture is developed.

The architecture should favor understandable asynchronous code over unnecessarily complex concurrency abstractions.

---

# Dependency Injection

ViewModels should receive the Services they require rather than constructing those Services themselves.

For example:

```text
PlayModeViewModel
        ↓
BluetoothService
```

This makes components easier to test and allows implementations to be replaced without changing the user interface.

The detailed approach will be defined in `Dependency-Injection.md`.

---

# Testing

The architecture should make important behavior testable without requiring a physical MP-3 for every test.

Where practical:

* ViewModel behavior should be testable independently.
* Service interfaces should support test implementations.
* Model behavior should be deterministic.
* Bluetooth protocol logic should be testable separately from the physical connection.

The detailed testing approach will be defined in `Testing-Strategy.md`.

---

# Architectural Restraint

Project Resonance should resist architecture for architecture's sake.

New layers, frameworks, protocols, and abstractions should be introduced only when they solve a demonstrated problem.

The preferred solution is the simplest architecture that:

* Preserves clear responsibilities.
* Remains testable.
* Supports foreseeable growth.
* Keeps the code understandable.

If two approaches provide equivalent capability, prefer the easier one to understand.

---

# Decisions

* SwiftUI adopted for the user interface.
* Lightweight MVVM adopted as the primary application pattern.
* Dedicated Service Layer retained alongside MVVM.
* Feature-oriented organization adopted for Views and ViewModels.
* Shared Models and Services remain outside individual features.
* Dependencies flow from presentation toward reusable application capabilities.
* Modern Swift concurrency preferred for asynchronous operations.
* Dependency injection preferred over ViewModels constructing their own Services.
* Architectural restraint established as a project principle.

---

## Open Questions

* What minimum iOS version should Project Resonance support?
* Which Swift observation approach should be used for ViewModels and shared state?
* What exact dependency injection technique should be adopted?
* How should application-wide connection state be shared between features?
* How much of the Mightier Amp codebase can appropriately inform or accelerate our implementation?

These questions should be resolved as close as practical to the implementation that depends on them.

---

## Future Ideas

As Project Resonance grows, evaluate whether individual components should become separate Swift packages.

Modularization should occur only when it produces a clear development or maintenance benefit.

---

## Related Documents

* Architecture Overview
* Data Model
* Navigation Architecture
* Bluetooth Architecture
* Dependency Injection
* Engineering Standards
* Testing Strategy

---

*Project Resonance • Architecture • Version 0.1 • Status: Draft*
