---
name: reachy-mini-sdk
description: Knowledge base and idiom guide for the `reachy_mini` Python SDK from Pollen Robotics / Hugging Face. Activate on any task that imports `reachy_mini`, instantiates the `ReachyMini` class, builds or modifies a Reachy Mini behavior, controls head pan/tilt/roll, drives the antennas, composes Move primitives, or otherwise programs the Reachy Mini desktop robot. Specifically triggers on phrasings like "build a behavior for Reachy Mini", "make Reachy nod / dance / wave the antennas", "open a connection to the Reachy Mini", or any code touching `from reachy_mini import ...`. Do not activate on pure hardware bring-up, on simulation-only tasks (MuJoCo / URDF), or on tasks that only publish a finished behavior — those have their own skills/agents.
tags: [reachy-mini, sdk, python, robotics]
---

# Reachy Mini SDK

Pinned SDK version: **`reachy_mini==<TBD>`** — set this on the first hardware contact and update through a deliberate spec revision afterward.

> ⚠ TBD: The hardware is not on hand yet. Every concrete API shape, signature, and value range below must be verified against the live SDK before being relied on. When unsure, always read the authoritative source (see "Source of truth" below) before generating code.

## When this skill activates

Activate on any of:

- imports of `reachy_mini` or symbols from it
- references to the `ReachyMini` class or its instances
- requests to control head **pan / tilt / roll**, the **antennas**, or to author a **behavior**
- composing **Move** primitives (easing, duration, interpolation) for Reachy Mini

## When NOT to activate

- pure hardware bring-up, calibration, firmware flashing → separate skill
- pure simulation work (MuJoCo / URDF) without SDK contact → separate skill
- publishing a finished behavior to Hugging Face Spaces / Hub → `behavior-publish-hf` (planned)
- audio beat / tempo detection alone → `audio-beat-tracking` (planned)

## Source of truth

The canonical sources are, in order:

1. The SDK source code: <https://github.com/pollen-robotics/reachy_mini>
2. The official Pollen Robotics docs linked from that repo
3. This skill (a curated summary; loses to the source on conflict)

Before producing API-shaped code, **read the relevant module under `pollen-robotics/reachy_mini`** and confirm the exact signature. Do not paste signatures from memory.

## Knowledge base

### Construction & lifecycle

A `ReachyMini` instance owns the connection to the robot. Use a context-manager-style lifecycle so resources are released even on exceptions.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified against reachy_mini==<TBD>
from reachy_mini import ReachyMini

with ReachyMini() as reachy:
    ...  # interact with the robot
```

> ⚠ TBD: Confirm whether the SDK ships a synchronous, asynchronous, or both surfaces, and whether the constructor takes a port / device argument.

### Head motion (pan / tilt / roll)

The head is a 3-DoF joint. Values are angles in a documented range; out-of-range values must be rejected by the SDK, not silently clamped.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified against reachy_mini==<TBD>
reachy.head.goto(pan=0.2, tilt=-0.1, roll=0.0, duration=0.5)
```

> ⚠ TBD: Confirm the exact attribute path (`reachy.head.goto` vs. `reachy.move_head` vs. other), the angle unit (radians vs. degrees), and the legal value ranges per axis.

### Antennas

Two independently driven antennas, typically expressed as left/right angles. Antenna moves often run in parallel with head moves to convey expression.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified against reachy_mini==<TBD>
reachy.antennas.goto(left=0.4, right=-0.4, duration=0.3)
```

> ⚠ TBD: Confirm the actual API name and whether each antenna is its own object or a pair argument.

### Behavior lifecycle

A behavior is a callable artifact with `setup`, an update loop, and `stop`. The SDK defines the contract; user code fills in the body.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified against reachy_mini==<TBD>
class WaveBehavior:
    def setup(self, reachy: "ReachyMini") -> None: ...
    def step(self, reachy: "ReachyMini", dt: float) -> None: ...
    def stop(self, reachy: "ReachyMini") -> None: ...
```

> ⚠ TBD: Confirm the official base class, method names, tick frequency, and how exceptions inside `step` are handled (cancelled? logged? re-raised?).

### Move primitives

Primitives compose head and antenna targets over time with easing and duration. Sequencing and parallel composition are expected.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified against reachy_mini==<TBD>
from reachy_mini import Move  # name TBD

nod = Move(reachy.head, target=dict(pan=0, tilt=0.3, roll=0), duration=0.4, easing="ease_out")
nod.play()
```

> ⚠ TBD: Confirm the actual primitive name(s), the easing vocabulary, and how parallel vs. sequential composition is expressed.

## Failure conditions to handle

- hardware not connected / wrong USB port
- protocol mismatch between SDK and firmware
- exceptions inside a behavior `step` — clean up the connection and surface the error

## Boundaries to neighbouring skills (planned)

- new behavior **scaffolding** → `behavior-scaffold`
- bidirectional **Home Assistant** wiring → `home-assistant-bridge`
- audio **beat / tempo detection** for dance behaviors → `audio-beat-tracking`
- live **on-device test / deploy** of a behavior → agent `reachy-mini-on-device`

These artifacts may not exist yet; defer to them once they do, and do not duplicate their concerns here.

## Drift check

- Re-validate every section against the SDK on each `reachy_mini` major release, or quarterly — whichever comes first.
- When the SDK source disagrees with this skill, the source wins. Update the skill (and the spec) rather than work around it.
- Bump the pinned SDK version explicitly in the header above; never let it silently fall behind.

## Hard rules

- **MUST NOT** paste API signatures from memory or from older Reachy SDKs (Reachy 2, Reachy Pro) without re-verification — the Reachy Mini API is its own surface.
- **MUST NOT** silently clamp out-of-range angles; let the SDK raise.
- **MUST** mark every unverified statement with `> ⚠ TBD: validate against real hardware` so reviewers see exactly what is unconfirmed.
- **MUST** name the verified SDK version next to every code example.
- **MUST** delegate to the neighbouring skills/agents listed above instead of growing this skill into them.
