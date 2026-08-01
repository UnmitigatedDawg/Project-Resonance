---
Title: Architecture Overview
Version: 0.1
Status: Draft
Owner: Product Team
Last Updated: 2026-08-01
Related Documents:
- PROJECT_BIBLE/02-Product-Philosophy.md
- PROJECT_BIBLE/04-Design-Principles.md
- PROJECT_BIBLE/05-Product-Standards.md
---

# Architecture Overview

> **Every software component should have one clear responsibility.**

The architecture of Project Resonance is designed to support a product that is maintainable, understandable, and enjoyable to evolve.

Like the Product Bible, the architecture favors clarity over cleverness and thoughtful design over unnecessary complexity.

Every architectural decision should ultimately support one objective:

> **Help musicians spend more time creating music and less time managing technology.**

---

# Architectural Philosophy

Project Resonance follows several guiding architectural principles.

- Clear separation of responsibilities.
- Simple, maintainable solutions.
- Incremental evolution.
- Documentation before implementation.
- Testable components.
- Loose coupling.
- High cohesion.

The architecture should remain understandable to future contributors long after the original implementation.

---

# Architectural Pattern

Project Resonance follows the MVVM (Model–View–ViewModel) architectural pattern.

The application is organized into four primary layers.

```
Musician
    ↓
View
    ↓
ViewModel
    ↓
Services
    ↓
Models
    ↓
NUX Mighty Plug Pro MP-3
```

Each layer has one primary responsibility.

---

# Layer Responsibilities

## View

The View presents information to the musician.

It displays the current application state and forwards user interactions to the ViewModel.

Views should contain little or no business logic.

---

## ViewModel

The ViewModel contains the application's presentation logic.

It coordinates user interactions, updates application state, and communicates with Services.

The ViewModel is the primary "brain" behind each screen.

---

## Services

Services perform work on behalf of the application.

Examples include:

- Bluetooth communication
- Tone persistence
- Logging
- Settings
- Future AI capabilities

Services should remain independent of the user interface whenever practical.

---

## Models

Models represent the application's data.

Examples include:

- Tone
- Tone Profile
- Pedal
- Signal Chain
- Device
- User Preferences

Models should contain business data without depending upon user interface implementation.

---

# Design Goals

The architecture should make it easy to:

- Understand the project.
- Add new features.
- Test components independently.
- Replace implementations when necessary.
- Support future expansion.
- Maintain long-term quality.

---

# Engineering Principles

Project Resonance values:

- Readability over cleverness.
- Simplicity over unnecessary abstraction.
- Explicitness over hidden behavior.
- Consistency over novelty.
- Maintainability over short-term convenience.

---

# Looking Ahead

Future architecture documents will describe:

- Folder organization
- Application modules
- Dependency flow
- Bluetooth architecture
- Data persistence
- Testing strategy
- Error handling
- AI integration

This document serves as the foundation for all future engineering decisions.

---

# Decisions

- MVVM adopted as the application architecture.
- Service Layer introduced between ViewModel and Models.
- Documentation-first engineering adopted.
- Single Responsibility Principle established as a foundational guideline.

---

## Open Questions

Should dependency injection be implemented using native Swift techniques or a dedicated dependency injection framework?

---

## Future Ideas

Create detailed architecture diagrams describing the relationships between major application components.

---

## Related Documents

- Product Philosophy
- Design Principles
- Product Standards

---

_Project Resonance • Architecture • Version 0.1 • Status: Draft_
