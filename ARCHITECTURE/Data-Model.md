# Data Model

**Version:** 0.1  
**Status:** Draft  
**Owner:** Product Team  
**Last Updated:** 2026-08-16  

**Related Documents:**
- Architecture-Overview.md
- Application-Architecture.md
- Bluetooth-Architecture.md
- Persistence-Architecture.md
- Engineering-Standards.md

---

# Data Model

> **The data model should describe the musician's world, not the implementation's convenience.**

The Project Resonance data model defines the core concepts the application needs to understand.

These models should reflect the way a guitarist thinks about tones, equipment, pedals, and creative context while remaining suitable for communication with the NUX Mighty Plug Pro MP-3.

The data model is intentionally independent of the user interface.

---

# Design Principles

The Project Resonance data model should:

* Use musical terminology whenever practical.
* Keep hardware-specific details isolated where possible.
* Support all nine MP-3 effect categories.
* Preserve both tone settings and creative context.
* Support future expansion without unnecessary complexity.
* Remain straightforward to serialize, persist, test, and migrate.

---

# Core Model Relationships

At a high level:

```text
ToneProfile
    |
    +-- Tone
    |     |
    |     +-- Pedalboard
    |             |
    |             +-- Pedal
    |             +-- Pedal
    |             +-- Pedal
    |             +-- ...
    |
    +-- InstrumentProfile
    |
    +-- Notes
    |
    +-- Tags
    |
    +-- Snapshots
    |
    +-- CoverImage
```

The `Tone` represents the sound.

The `ToneProfile` represents the complete musical context surrounding that sound.

---

# Tone

A `Tone` represents the complete configuration required to recreate a guitar sound on the MP-3.

Initial properties are expected to include:

```text
Tone
    id
    name
    pedalboard
    devicePresetSlot
    createdAt
    modifiedAt
```

## Responsibilities

A Tone should:

* Have a descriptive name.
* Contain the complete pedalboard configuration.
* Optionally know which MP-3 hardware slot currently contains it.
* Be usable independently from a Tone Profile.

A hardware slot number is metadata.

It is not the Tone's identity.

---

# Tone Profile

A `ToneProfile` enriches a Tone with the musical information needed to remember and recreate the experience.

Initial properties are expected to include:

```text
ToneProfile
    id
    tone
    instrument
    inspiration
    textNotes
    tags
    snapshots
    coverImage
    favorite
    createdAt
    modifiedAt
```

## Purpose

A Tone Profile answers:

> **How did I get this sound, and why did I want to remember it?**

This distinction is central to Project Resonance.

A preset stores settings.

A Tone Profile preserves a musical idea.

---

# Instrument Profile

An `InstrumentProfile` records the guitar and its relevant settings when a Tone Profile is created.

Initial properties may include:

```text
InstrumentProfile
    id
    name
    manufacturer
    model
    pickupSelection
    volumeSetting
    toneSetting
    coilSplitState
    notes
```

Not every field must be populated.

The user should never be required to complete a form before saving a tone.

---

# Pedalboard

The `Pedalboard` represents the ordered signal chain.

```text
Pedalboard
    pedals[]
```

The ordering of the collection represents signal flow.

This is important because the NUX Mighty Plug Pro MP-3 allows supported elements of the signal chain to be reordered.

The model must therefore treat pedal order as data rather than assuming a fixed layout.

---

# Pedal

A `Pedal` represents one active or available signal-processing block.

Initial properties are expected to include:

```text
Pedal
    id
    category
    model
    enabled
    position
    parameters
```

## Pedal Categories

Project Resonance currently recognizes nine MP-3 categories:

1. Gate
2. Compressor
3. EFX
4. Delay
5. Amplifier
6. Cabinet / IR
7. EQ
8. Modulation
9. Reverb

The exact hardware behavior and supported models within each category will be validated against the Mightier Amp implementation and the physical MP-3.

---

# Pedal Parameters

Different pedals expose different controls.

For example:

```text
Compressor
    sustain
    level
```

while:

```text
Delay
    time
    feedback
    mix
```

Project Resonance should not force every pedal into one fixed parameter structure.

The implementation should support pedal-specific parameter definitions while preserving a consistent interface for reading, displaying, and modifying them.

The precise Swift representation will be determined during implementation.

---

# Pedal State

Each pedal has an enabled state.

```text
enabled = true
```

or:

```text
enabled = false
```

This state supports the Play Mode footswitch interaction.

An inactive pedal may remain part of a Tone even when it is bypassed.

Play Mode may visually hide unused signal-chain slots while Tone Studio retains visibility into the complete configuration.

---

# Signal Chain Order

Signal-chain order must be explicitly represented.

For example:

```text
Gate
↓
Compressor
↓
EFX
↓
Amplifier
↓
Cabinet
↓
Delay
↓
EQ
↓
Reverb
```

If EQ is moved after Delay, the model must preserve that ordering.

The user interface should never need to infer pedal order.

---

# Device

A `Device` represents a supported physical amplifier or headphone amplifier.

Initial properties may include:

```text
Device
    id
    name
    model
    firmwareVersion
    batteryLevel
    connectionState
```

Version 1.0 targets:

```text
NUX Mighty Plug Pro MP-3
```

The model should not unnecessarily prevent support for additional devices later.

---

# Connection State

Connection status should be represented explicitly.

Possible states may include:

```text
disconnected
discovering
connecting
connected
reconnecting
failed
```

The exact Swift representation will be defined during implementation.

A clear connection model allows the user interface to respond consistently without understanding Bluetooth internals.

---

# Snapshot

A `Snapshot` preserves a meaningful user-selected version of a Tone Profile.

Initial properties may include:

```text
Snapshot
    id
    name
    tone
    notes
    createdAt
```

Snapshots are intentionally manual.

The working Tone Profile may autosave, but permanent versions are created when the guitarist decides a state is worth preserving.

---

# Inspiration

The `inspiration` field captures the reason or idea behind a tone.

Version 1.0 should support written text.

Future versions may support:

* Voice capture
* Automatic transcription
* Searchable transcription
* Optional audio playback

The model should allow those capabilities to be added later without requiring Tone Profiles to be redesigned.

---

# Cover Image

A Tone Profile may optionally contain a cover image reference.

Version 1.0 should not require cover art.

Future versions may support AI-generated imagery designed to evoke the mood or character of a tone without reproducing copyrighted artwork.

---

# Tags

Tone Profiles may support descriptive tags.

Examples:

```text
80s
Clean
Chorus
Practice
Live
Recording
Favorite
```

Tags should remain flexible rather than being restricted to a fixed taxonomy.

---

# User Preferences

`UserPreferences` represents application-level preferences.

Initial examples include:

```text
keepScreenAwakeWhileConnected
preferredToneSort
hapticsEnabled
```

Only preferences with clear user value should be introduced.

---

# Hardware Preset Slots

The MP-3's hardware preset slots and Project Resonance's Tone Library are separate concepts.

A Tone may exist in the Project Resonance library without being stored in a hardware slot.

Conceptually:

```text
Tone Library
    Unlimited Tones

MP-3
    Limited Hardware Preset Slots
```

A Tone may be assigned or sent to a hardware slot when desired.

This separation prevents hardware storage limits from becoming application storage limits.

---

# Identity

Models that persist over time should have stable identifiers.

Names are editable and should not be used as identity.

For example:

```text
ToneProfile.id
```

remains constant even if:

```text
"The Cars – Let's Go"
```

is later renamed.

---

# Dates and History

Persistent creative objects should record meaningful timestamps where useful.

Examples include:

* Created date
* Modified date
* Snapshot date

Dates should support history and organization without cluttering the musician's experience.

---

# Persistence Considerations

The model should be designed so it can be persisted without depending on the chosen storage technology.

The application should be able to evolve from one persistence mechanism to another without changing the conceptual meaning of:

* Tone
* Tone Profile
* Snapshot
* Instrument Profile
* User Preferences

The detailed persistence strategy is maintained in `Persistence-Architecture.md`.

---

# Bluetooth Considerations

The domain model should not be forced to exactly mirror raw Bluetooth packet formats.

Instead:

```text
Domain Model
      ↕
Protocol Translation
      ↕
Bluetooth Messages
```

The Bluetooth layer should translate between hardware-specific representations and Project Resonance's domain models.

This keeps the rest of the application understandable in musical terms.

---

# Decisions

* Tone and Tone Profile are separate concepts.
* Tone represents sound configuration.
* Tone Profile represents sound plus creative context.
* Pedalboard order is explicit data.
* All nine MP-3 effect categories are represented.
* Pedal parameters may vary by pedal model.
* Hardware preset slots remain separate from the unlimited Tone Library.
* Snapshots are manually created meaningful versions.
* Hardware protocol representation remains separate from the domain model.
* Persistent objects use stable identifiers.

---

## Open Questions

* What exact parameter types are required for every supported MP-3 pedal and amplifier model?
* Which signal-chain reorderings are permitted by the MP-3 firmware?
* How many hardware preset slots should the application expose for the MP-3?
* Should `InstrumentProfile` be reusable across multiple Tone Profiles or copied into each profile?
* Should tags be stored as simple strings or eventually become reusable structured objects?
* What persistence technology best fits the Version 1.0 data model?

---

## Future Ideas

Potential future extensions include:

* Multiple guitars stored as reusable instrument profiles.
* Setlists.
* Tone collections.
* Voice-to-text inspiration capture.
* AI-generated cover art.
* Community tone sharing.
* Additional NUX hardware models.

Future additions should extend the data model without weakening its musician-centered vocabulary.

---

## Related Documents

* Architecture Overview
* Application Architecture
* Bluetooth Architecture
* Persistence Architecture
* Engineering Standards

---

*Project Resonance • Architecture • Version 0.1 • Status: Draft*
