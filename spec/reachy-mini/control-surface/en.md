# Reachy Mini Control Surface and Motion Design

Status: draft

## Context
Anyone writing skills, behaviors, or agents in this plugin needs a canonical reference for *which* elements of the Reachy Mini are actually controllable, *how* they are addressed, and *under which limits* you compose natural-looking motion from them. Without this specification every implementation reconstructs the hardware reality from scratch — usually from half-baked training samples, which on first real device contact either risks the hardware or produces mechanical-feeling motion. This spec consolidates the publicly known hardware and SDK properties of Reachy Mini, describes each control layer the SDK exposes, and for every mechanical and electrical limitation states how it shapes motion composition. It is the normative knowledge base against which the `reachy-mini-sdk` skill validates its snippets, and against which the `behavior-scaffold` skill anchors its templates.

## Goals
- Every controllable element of the Reachy Mini is catalogued with identifier, axes, value range, and unit
- Control layers (pose-goto, move primitives, behavior lifecycle, streaming) are clearly distinguished; a behavior knows which layer fits which task
- Dependencies between hardware variant, SDK version, and firmware are surfaced so an implementation can check them early
- Mechanical, electrical, and latency-bound limitations are documented in a form that can be expressed as a precondition in code
- Patterns for natural, fluid motion (easing, composition, anticipation, follow-through, synchronisation, idle breathing) are spelled out and contrasted with anti-patterns
- All hardware-specific numbers carry a `> ⚠ TBD` marker until verified against the real SDK docs and device

## Non-Goals
- Concrete motion choreographies (the job of individual behaviors / apps)
- Audio beat-tracking pipelines (`audio-beat-tracking`, planned)
- Home Assistant API contract (`home-assistant-bridge`)
- Hardware bring-up, calibration, firmware flashing (separate skills, planned)
- Simulation or URDF model (separate spec)
- Vision pipeline, speech recognition, on-device ML
- Mechanical CAD data or electrical schematics — this spec stays at the software-control level

## Requirements

### Architecture overview

Reachy Mini is a desktop robot from Pollen Robotics / Hugging Face that ships in two variants: **Wired** (cable-tethered, hangs off a host computer) and **Wireless** (with built-in Raspberry Pi 5 and battery). Software-side control runs through the Python SDK `reachy_mini`, which exposes a `ReachyMini` instance as the central entry point. That instance aggregates the controllable subsystems — head, antennas, body rotation, audio output, display, sensors. Motion is expressed in several layers; higher layers use lower layers as an implementation detail.

> ⚠ TBD: validate against pollen-robotics/reachy_mini — the exact aggregation of subsystems on the `ReachyMini` instance (attribute paths, method names) is reconciled against the official SDK on first device contact.

### Hardware inventory — actuators

Controllable mechanical axes:

| Subsystem | DoF | Axes | Value range | Unit | Hardware variant |
|---|---|---|---|---|---|
| Head (Stewart platform) | up to 6 | pan (yaw), tilt (pitch), roll, optionally translation x/y/z | `> ⚠ TBD` per axis | rad / mm `> ⚠ TBD` | Wired + Wireless |
| Left antenna | 1 | rotation around base axis | `> ⚠ TBD` | rad | Wired + Wireless |
| Right antenna | 1 | rotation around base axis | `> ⚠ TBD` | rad | Wired + Wireless |
| Body yaw | 1 | rotation around vertical Z axis | `> ⚠ TBD` | rad | Wireless only `> ⚠ TBD` |

Inventory-maintenance requirements:

- **MUST** document each axis with identifier, value range, unit, and default pose as soon as the SDK docs are available
- **MUST** indicate per axis whether the SDK API accepts absolute targets, incremental targets, or both
- **MUST** mark axes that exist only in one hardware variant (e.g. body yaw on Wireless only `> ⚠ TBD`)
- **SHOULD** state the velocity and acceleration limits per axis that follow from the mechanical build (`> ⚠ TBD`)

### Hardware inventory — outputs (non-mechanical)

| Subsystem | Property | Controllability |
|---|---|---|
| Display ("eyes") | mini LCD/AMOLED, resolution `> ⚠ TBD`, programmable content | images, simple animations, eye-expression layer |
| Speaker | one, power `> ⚠ TBD W` | audio playback (WAV/PCM, further codecs `> ⚠ TBD`) |
| Status LEDs (optional) | count `> ⚠ TBD` | `> ⚠ TBD` whether SDK-controllable or fixed |

Requirements:

- **MUST** model the display subsystem at least as an expression-bearing output layer — motion code may trigger eye expression and pose simultaneously
- **MUST** document whether audio playback blocks the behavior tick or runs asynchronously (`> ⚠ TBD`)
- **SHOULD** include recommendations for audio-latency measurement and compensation patterns once SDK behavior is verified

### Hardware inventory — sensors / inputs

| Subsystem | Property | Read API |
|---|---|---|
| Microphone array | 4 mics `> ⚠ TBD`, directional capture possible | stream / frame pulls `> ⚠ TBD` |
| Camera | one, wide-angle, resolution `> ⚠ TBD`, framerate `> ⚠ TBD` | frame pull / stream `> ⚠ TBD` |
| IMU (if present) | `> ⚠ TBD` whether present and SDK-exposed | `> ⚠ TBD` |
| Position feedback per actuator | geometry-read endpoint per axis | `reachy.head.pose`, `reachy.antennas.left.angle`, etc. — exact API `> ⚠ TBD` |

Requirements:

- **MUST** document a read method per actuator so behaviors can fetch the actually-reached pose without blocking the actuator
- **MUST** clarify whether sensor streams come blocking or as async iterators
- **MUST NOT** assume a sensor read is free — read frequency is bounded (see the latency section)

### Hardware variants and their consequences

- **Wired** — cable-tethered, hangs off a host PC; powerful host, low latency on the USB link; possibly no body yaw `> ⚠ TBD`
- **Wireless** — with built-in Raspberry Pi 5 and battery; CPU and memory budget is bounded, motion code must run leaner; body yaw available `> ⚠ TBD`

Requirements:

- **MUST** make it visible in code which variant a motion is written for; a Wired motion with high update frequency must not silently run on Wireless when CPU isn't enough
- **SHOULD** offer a way to detect the variant at runtime (capability discovery, `> ⚠ TBD` whether the SDK exposes this)
- **MUST NOT** issue body-yaw motions without a variant check on Wired devices

### Control layers

Four layers, from fine-grained (low level) to narrative (high level):

1. **Pose-goto** — directly drive a target pose with duration; SDK handles interpolation and ramp
   - example API: `reachy.head.goto(pan, tilt, roll, duration)` (`> ⚠ TBD`)
   - granularity: one call = one motion
   - own interrupt model: a new goto during a running goto behaves `> ⚠ TBD` (override / queue / reject)

2. **Move primitives** — composable building blocks like nod, shake, look-at, antenna flap
   - carry properties such as easing profile, duration, repetition
   - have a `play()` / `compose()` form that chains parallel or sequentially with other primitives (`> ⚠ TBD`)

3. **Behavior** — stateful, with `setup` / `step` / `stop` hooks and a tick loop
   - read-write loop: every tick reads sensors, decides on move primitives, writes targets
   - tick frequency: `> ⚠ TBD` Hz typical; upper bound governed by CPU / variant
   - owns lifecycle, can trigger emergency stop

4. **Streaming** — continuous pose targets at low latency, without move-primitive wrapping
   - use case: dance on an audio stream, external mocap drivers
   - explicitly needs backpressure handling, otherwise actuator saturation
   - availability `> ⚠ TBD`

Requirements:

- **MUST** decide for every behavior which layer is the main channel; mixing layer 1 and layer 4 without a layer-2 wrapper produces pose jumps
- **SHOULD** recommend layer 2 (move primitives) as the default because it already does easing and composition right
- **MUST NOT** call layer 1 inside a tick loop above `> ⚠ TBD` Hz — that bypasses the layer's easing and produces stutter

### Parameters and value ranges

- **MUST** validate each axis's allowed value range in the SDK; out-of-range values are rejected by the SDK, not silently clamped (expectation toward the SDK; `> ⚠ TBD` whether actually implemented — fall back to a local validation layer if not)
- **MUST** keep units consistent: angles in radians, distances in millimetres, time in seconds — conversions visible at system boundaries
- **SHOULD** offer default poses (rest position) for head and antennas as named constants instead of magic numbers everywhere

### Motion latency and update frequency

- **MUST** state the typical end-to-end latency (software command → mechanical reaction) in the skill body once measured (`> ⚠ TBD ms`)
- **MUST** document update-frequency limits: each hardware variant has a different upper bound (Wireless typically lower than Wired; concrete numbers `> ⚠ TBD`)
- **MUST NOT** recommend tick frequencies the SDK can't sustain — the visual effect is actuator stutter, not speed

### Dependencies

- **`reachy_mini` SDK version** — pinned in the consuming repo; every motion depends on this version's API shape
- **Device firmware version** — mismatch between SDK and firmware can cause connect or behavior errors; verify on every first connect (`> ⚠ TBD` whether SDK exposes this)
- **Python version** — lower bound per SDK requirement (`> ⚠ TBD`)
- **Hardware variant** — Wired vs. Wireless influences actuator set and CPU budget
- **Host connectivity** — Wired needs USB, Wireless needs Wi-Fi (for code sync) and local power
- **System audio stack** — when the behavior plays audio, it depends on the variant's audio stack (PulseAudio / PipeWire `> ⚠ TBD`)

### Mechanical and electrical limitations

- **End stops** — every axis has a mechanical end stop; the SDK should protect against damage, but the implementation must not aim into the end stop in the first place
- **Velocity / acceleration** — linear actuators carry hard velocity and acceleration limits; a command that undercuts them is silently executed slower (`> ⚠ TBD`)
- **Current draw** — many simultaneous motions (head full + both antennas + body) can briefly drop the voltage on battery operation; depending on battery state the system reacts with brown-out protection
- **Thermal budget** — sustained motion at high frequency generates heat; the SDK may expose temperature telemetry (`> ⚠ TBD`)
- **Collisions** — antennas collide with the head at extreme angles; motion must not target such pose combinations (`> ⚠ TBD` which combinations exactly are forbidden)

Requirements:

- **MUST** check the combinatorial collision prohibitions before issuing a motion if the SDK doesn't enforce them itself
- **MUST** avoid current spikes by not driving every actuator at maximum speed simultaneously
- **SHOULD** consume temperature telemetry once the SDK exposes it — pause for a cool-down on threshold breach

### Safety limits

- **MUST** keep an emergency-stop path reachable that ends every running motion and drives to a rest pose — pose definition `> ⚠ TBD: validate against real hardware`
- **MUST** put the device into a safe pose after an unexpected disconnect or program abort, instead of leaving a motion "frozen"
- **MUST NOT** override current, force, or velocity thresholds via SDK parameters without an explicit justification in the behavior

### Patterns for natural, fluid motion

The following principles are translated from classical animation onto a 6-DoF head + 2-antenna system. They are not optional when the result should feel "alive".

1. **Easing (slow-in / slow-out)** — no linear motion. Pose-goto calls without an easing argument are bad because the motion feels mechanical. Default easing of move primitives is `> ⚠ TBD`, but should be at least cubic in/out.
2. **Anticipation** — before a large motion, a tiny counter-motion (example: before nodding down, briefly look up by ~80–150 ms). This makes the main move "readable".
3. **Follow-through and overlapping action** — antennas trail head motion with a small lag (~50–120 ms), not synchronously. When the head stops, antennas may swing slightly afterwards.
4. **Arcs** — motion paths follow arcs, not straight lines. A look-left → look-right through the geometric centre feels mechanical; a slight arc through a small tilt rise feels organic.
5. **Secondary action** — during the main action (e.g. nodding), run a small secondary action (e.g. one antenna twitches faintly). This conveys "aliveness".
6. **Idle breathing** — without an active task the device must not stand fully still. A slow, unobtrusive idle move (~0.2–0.4 Hz) on tilt and roll mimics breathing. Recommended with small amplitude (`> ⚠ TBD`).
7. **Synchronisation antennas + head** — antenna motions coherent with head motion (e.g. "ears pricked up" on a look-at) feel intentional. Incoherent mixing feels noisy.
8. **Audio synchronicity** — for dance or sound reactions: align motion and audio onset to within a few milliseconds, not offset by an asynchronous audio playback (see `audio-beat-tracking`).
9. **Timing variation** — fixed tick steps feel machine-like. Vary move-primitive duration by ~5–15 %, so no two identical moves emerge.
10. **Eye-display synchronicity** — when the display shows eyes: before a look-at motion, let the eyes lead (saccade), then the head follows. Eye motion is much faster than mechanics.

### Anti-patterns (what makes motion feel mechanical)

- linear pose interpolation without easing
- a tick loop running at constant high frequency that ignores layer 2
- moving antennas synchronously with the head instead of with lag
- pose jumps between two `goto()` calls without a stop phase
- idle stillness without a breathing move
- audio over a system player with unknown latency — dance is out of sync
- simultaneous max speed on every actuator — Wireless then brown-outs
- fixed, repetitive move sequences without timing variation
- treating emergency stop as just a log note, not a physical action

### Observability

- **MUST** document a read method per actuator so behaviors can fetch the actually-reached pose (no blind writing)
- **SHOULD** expose latency measurement points per layer (command sent → pose reached)
- **SHOULD** structure telemetry so the `reachy-mini-on-device` agent can derive a PASS/FAIL verdict from it
- **MAY** define a trace format that allows live visualisation of motion in a web dashboard

## Acceptance Criteria
- [ ] Every actuator (head, antennas, body yaw) is listed with DoF, axes, and value range (or TBD)
- [ ] Every output (display, speaker, optional LEDs) is listed with control modality
- [ ] Every sensor (microphones, camera, IMU, position feedback) is listed with read-API shape
- [ ] Both hardware variants (Wired, Wireless) are named; differences in actuator set and CPU budget are listed
- [ ] Four control layers are described with use case, API shape, interrupt model
- [ ] A unified unit system (rad, mm, s) is set
- [ ] Latency and update frequency are listed as required fields, even if the values are TBD today
- [ ] Six dependency fields (SDK, firmware, Python, variant, connectivity, audio stack) are listed
- [ ] Mechanical, electrical, and thermal limitations are named; an emergency-stop path is defined
- [ ] At least 10 patterns for natural, fluid motion are documented
- [ ] At least 9 anti-patterns are documented
- [ ] Observability (position read, latency trace, telemetry) is captured as a requirement
- [ ] Every hardware-specific number carries a `⚠ TBD` marker
- [ ] The `reachy-mini-sdk` skill points at this spec as the canonical knowledge base
- [ ] The `behavior-scaffold` skill points at this spec for move primitives and default easings

## Open Questions
- Does the Reachy Mini head have 3 DoF (purely rotational) or 6 DoF (Stewart platform with translation)? Sources disagree; resolve from the data sheet before first implementation.
- Which audio codecs does the SDK natively support — PCM/WAV, MP3, OGG? Which codec pipeline is optimal for dance with beat synchronisation?
- Does the SDK offer a capability-discovery API to distinguish Wired vs. Wireless at runtime?
- Does the SDK expose brown-out or current-spike telemetry, or does the behavior have to measure it itself?
- What is the canonical rest pose for the emergency stop? Proposal: every axis centred, antennas slightly upright, body yaw 0.
- Which update frequencies are realistic in a behavior tick — 50 Hz, 100 Hz, 200 Hz? Depends on variant and SDK implementation.
- Is there an official animation-authoring tool from Pollen Robotics (timeline editor), and is its output format a recommended composition for our move-primitive layer?
- How does the Stewart-platform head behave mathematically near singularities? Do we need to block singular regions in code?
- Which latency distribution (P50, P95, P99) is typical per layer? Without this distribution, a realistic dance timing is not plannable.
- Should animation principles (anticipation, follow-through, …) move into a separate `motion-design` spec once the patterns grow?
- Are the official Pollen Reachy Mini behaviors (e.g. "Hello", "Curious") released as reference implementations against which we can calibrate our pattern application?
- How far do classical animation patterns translate onto a 6-DoF robot without sliding into uncanny-valley territory? Empirical evaluation on real hardware required.
