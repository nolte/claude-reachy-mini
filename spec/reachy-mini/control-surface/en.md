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

Reachy Mini is a desktop robot from Pollen Robotics / Hugging Face that ships across three platforms: **Reachy Mini** (Wireless, with built-in Raspberry Pi 4 Compute Module and LiFePO4 battery), **Reachy Mini Lite** (tethered to a host computer over USB-C, externally powered), and **Simulation** (software-only, same `ReachyMini` API without real motors). Software-side control follows a **client-server model**: a local daemon owns the hardware connection and the safety layer, while the SDK client (the `reachy_mini` Python package) talks to the daemon over REST/WebSocket. A `ReachyMini` instance is the central entry point and is typically used as a context manager (`with ReachyMini() as mini:`). It exposes motion, sensor, and media functions directly as its own methods and properties (`mini.goto_target(...)`, `mini.imu`, `mini.media`); there are **no** separate `head` / `antennas` / `eyes` / `display` sub-objects.

Motion is expressed in two coordinate systems — **Head Frame** (local to the Stewart platform) and **World Frame** (world-relative, e.g. for `look_at_world(...)`). The SDK ships built-in **safety limits** that prevent self-collision and hardware damage.

Canonical sources: hosted docs <https://huggingface.co/docs/reachy_mini/>, core concepts <https://huggingface.co/docs/reachy_mini/SDK/core-concept>, class source <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>.

### Hardware inventory — actuators

Controllable mechanical axes (verified against the `ReachyMini` class API at <https://huggingface.co/docs/reachy_mini/API/reachymini>):

| Subsystem | DoF | Motors | Representation | Write API | Read API | Value range | Unit | Platforms |
|---|---|---|---|---|---|---|---|---|
| Head | 6 (Stewart platform: 3 rotation + 3 translation) | 6× Dynamixel XL330-M288-T | 4×4 transform matrix; builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees, mm)` | `goto_target(head=...)`, `set_target(head=...)`, `set_target_head_pose(pose)` | `get_current_head_pose() -> np.ndarray` (4×4) | `> ⚠ TBD` per axis | rad / mm | all |
| Antennas (pair) | 2 (1 DoF per antenna) | 2× Dynamixel XL330-M077-T | `List[float]` with two joint angles (left, right) | `goto_target(antennas=...)`, `set_target(antennas=...)`, `set_target_antenna_joint_positions(antennas)` | `get_present_antenna_joint_positions() -> List[float]` | `> ⚠ TBD` | rad | all |
| Body yaw | 1 (base rotation) | 1× custom Dynamixel XC330-M288-PG | `float` | `goto_target(body_yaw=...)`, `set_target(body_yaw=...)`, `set_target_body_yaw(value)`, `set_automatic_body_yaw(enabled)` | part of `get_current_joint_positions()` | `> ⚠ TBD` | rad | all (Wireless **and** Lite, soft-state in Simulation) |

Concrete write-API docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>. Pose builder: <https://huggingface.co/docs/reachy_mini/API/tools>.

Inventory-maintenance requirements:

- **MUST** document each axis with identifier, value range, unit, and default pose as soon as the SDK docs are available
- **MUST** indicate per axis whether the SDK API accepts absolute targets, incremental targets, or both
- **MUST** mark axes that exist only in one hardware variant (e.g. body yaw on Wireless only `> ⚠ TBD`)
- **SHOULD** state the velocity and acceleration limits per axis that follow from the mechanical build (`> ⚠ TBD`)

### Hardware inventory — outputs (non-mechanical)

| Subsystem | Property | Controllability |
|---|---|---|
| Speaker | 5 W @ 4 Ω, single | audio playback via `mini.media.audio.*` ([SDK/media-architecture](https://huggingface.co/docs/reachy_mini/SDK/media-architecture)); concrete codec list `> ⚠ TBD: validate against current SDK media backend` |
| LED ring on the microphone module | LEDs on the ReSpeaker / mic-array board; registers `LED_EFFECT`, `LED_BRIGHTNESS`, `LED_GAMMIFY`, `LED_SPEED` | via `audio_control_utils` ([source](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_control_utils.py)); programmatically writable |
| ~~Display / eye screen~~ | **Reachy Mini ships no programmable eye display.** The "eyes" are mechanical 3D-printed parts (`pp01079_back_big_eye`, `pp01080_back_small_eye`) on the head shell — not controllable. The Pollen control app's "Expressions" are motion compositions of head pose + antenna positions, not display content. | not controllable |

Requirements:

- **MUST** document whether audio playback blocks the behavior tick or runs asynchronously (`> ⚠ TBD: validate against current SDK media backend`)
- **SHOULD** include recommendations for audio-latency measurement and compensation patterns once SDK behavior is verified
- **MUST** model the LED ring as a status / audio-feedback affordance — not as an "eye expression", because it is not located in the eye area of the robot
- **MUST NOT** assume or simulate an "eye display" as a controllable element — Reachy Mini does not ship one

### Hardware inventory — sensors / inputs

| Subsystem | Property | Read API | Docs |
|---|---|---|---|
| Microphone array | 4× PDM MEMS digital, 16 kHz, -26 dB FS, direction-of-arrival capable | via `mini.media` (`MediaManager`) | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), example [`sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Camera | Raspberry Pi v3 wide angle (Sony IMX708, 12 MP, autofocus); concrete stream parameters `> ⚠ TBD: validate against current backend` | via `mini.media.camera` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), example [`take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| IMU (present, confirmed) | accelerometer, gyroscope, quaternion, temperature | `mini.imu` (property → `Dict \| None`) | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini), example [`imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| Position feedback head | current 4×4 pose | `mini.get_current_head_pose() -> np.ndarray` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Position feedback antennas + joints | joint angles | `mini.get_current_joint_positions()`, `mini.get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Media release / acquire | yield camera/mic to external code | `mini.release_media()`, `mini.acquire_media()`, property `mini.media_released` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |

Requirements:

- **MUST** document a read method per actuator so behaviors can fetch the actually-reached pose without blocking the actuator
- **MUST** clarify whether sensor streams come blocking or as async iterators
- **MUST NOT** assume a sensor read is free — read frequency is bounded (see the latency section)

### Hardware platforms and their consequences

Pollen Robotics ships the `ReachyMini` API across three platforms, with the same methods but different compute and actuator profiles:

- **Reachy Mini** (Wireless) — built-in Raspberry Pi 4 Compute Module (CM4104016, 4 GB RAM, 16 GB flash), 2.4–5 GHz dual-band patch antenna, LiFePO4 battery (2000 mAh, 6.4 V, 12.8 Wh); fully self-contained. Docs: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/get_started>
- **Reachy Mini Lite** — tethered to a host computer over USB-C (USB-C does not charge the device), external 6.8–7.6 V power supply; the host carries heavy lifts, same actuator set as Wireless. Docs: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini_lite/get_started>
- **Simulation** — software-only, same `ReachyMini` API without real motors; usable for CI, tests, code shake-out without a device. Constructor argument `use_sim=True`. Docs: <https://huggingface.co/docs/reachy_mini/platforms/simulation/get_started>

Requirements:

- **MUST** make it visible in code which platform a motion is written for; a high-frequency motion designed for Wireless must not silently run on Lite or Simulation
- **SHOULD** provide a way to detect the platform at runtime (capability discovery; `> ⚠ TBD` whether the SDK exposes a direct variant property; the constructor argument `use_sim` is verified to exist)
- **MUST** signal explicitly to the caller when a motion that depends on hardware capabilities (e.g. real audio playback, IMU values) runs in Simulation, instead of silently no-op'ing

### Control layers

Four layers, from direct-write (low level) to narrative (high level), with verified method names:

1. **Direct-set** — target is written immediately, no interpolation
   - methods: `set_target(head, antennas, body_yaw)`, `set_target_head_pose(pose)`, `set_target_antenna_joint_positions(antennas)`, `set_target_body_yaw(value)`
   - docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>
   - usage: only when easing is provided by a higher layer or by your own trajectory — otherwise produces mechanical-feeling motion

2. **Goto-target** — smooth goto over a named duration, SDK handles interpolation
   - method: `goto_target(head, antennas, duration, method, body_yaw)` with `method: InterpolationTechnique`
   - pose builder: `create_head_pose(...)` from `reachy_mini.utils`
   - docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>, source: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/goto.py>
   - playground: <https://huggingface.co/docs/reachy_mini/examples/goto_interpolation_playground>
   - granularity: one call = one motion. Interrupt model `> ⚠ TBD: validate against real hardware`.

3. **Move composition** — reusable motions as subclasses of the `Move` ABC with `duration` and `evaluate(t) -> (head_pose | None, antennas | None, body_yaw | None)`
   - source: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py>
   - playback: `mini.async_play_move(move, play_frequency, initial_goto_duration, sound)` (non-blocking) or `mini.play_move(...)` (blocking); `mini.cancel_move()` stops active playback
   - examples: <https://huggingface.co/docs/reachy_mini/examples/recorded_moves>, <https://huggingface.co/docs/reachy_mini/examples/sequence>
   - easing / composition: `Move` itself has no built-in easing primitives — subclasses supply their own trajectories; sequential / parallel composition is built by consumers

4. **High-level behaviors** — finished methods shipped by the SDK that compose pose + sound + animation
   - methods: `wake_up()`, `goto_sleep()`, `look_at_image(u, v, duration, perform_movement)`, `look_at_world(x, y, z, duration, perform_movement)`
   - docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>, example: <https://huggingface.co/docs/reachy_mini/examples/look_at>
   - prefer them when the task semantically matches the method name — no re-implementation

Requirements:

- **MUST** clearly document for every behavior which layer is the main channel
- **SHOULD** prefer layer 4 (high-level) when the task matches a shipped method — no own `look_at` next to the official one
- **SHOULD** choose layer 3 (Move subclass) for reusable motions — it provides clean lifecycle semantics (`duration`, `evaluate`) and is compatible with audio synchronisation
- **MUST NOT** call layer 1 (direct-set) inside a tick loop without your own trajectory smoothing — produces stutter

### Parameters and value ranges

- **MUST** validate each axis's allowed value range in the SDK; out-of-range values are rejected by the SDK, not silently clamped (expectation toward the SDK; `> ⚠ TBD` whether actually implemented — fall back to a local validation layer if not)
- **MUST** keep units consistent: angles in radians, distances in millimetres, time in seconds — conversions visible at system boundaries
- **SHOULD** offer default poses (rest position) for head and antennas as named constants instead of magic numbers everywhere

### Motion latency and update frequency

- **MUST** state the typical end-to-end latency (software command → mechanical reaction) in the skill body once measured (`> ⚠ TBD ms`)
- **MUST** document update-frequency limits: each platform has a different upper bound (Wireless typically lower than Lite, since Lite offloads to the host; concrete numbers `> ⚠ TBD: validate against real hardware`)
- **MUST NOT** recommend tick frequencies the SDK can't sustain — the visual effect is actuator stutter, not speed

### Dependencies

- **`reachy_mini` SDK version** — pinned in the consuming repo; every motion depends on this version's API shape
- **Device firmware version** — mismatch between SDK and firmware can cause connect or behavior errors; verify on every first connect (`> ⚠ TBD` whether SDK exposes this)
- **Python version** — lower bound per SDK requirement (`> ⚠ TBD`)
- **Hardware platform** — Wireless / Lite / Simulation influences CPU budget and power supply; the actuator set is identical on Wireless and Lite
- **Host connectivity** — Lite needs USB-C to a host plus external 6.8–7.6 V; Wireless needs Wi-Fi (for code sync) and runs from the battery; Simulation needs no host outside the Python process
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
10. **Body-yaw + head-pose synchronicity** — when `automatic_body_yaw=True` (recommended default), the body follows the head pose via IK and offloads work from the Stewart platform's yaw. When both axes are driven manually, do not let body yaw and head yaw move counter to each other — the resulting twist looks unnatural. Docs: [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) (`set_automatic_body_yaw`).

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

### API reference overview (canonical doc links)

| Area | API element | Doc link |
|---|---|---|
| Construction | `ReachyMini(robot_name, host, port, connection_mode, spawn_daemon, use_sim, timeout, automatic_body_yaw, log_level, media_backend, localhost_only)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Pose builder | `create_head_pose(...)` from `reachy_mini.utils` | [`API/tools`](https://huggingface.co/docs/reachy_mini/API/tools) |
| Smooth goto | `goto_target(head, antennas, duration, method, body_yaw)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Direct set | `set_target(...)`, `set_target_head_pose(...)`, `set_target_antenna_joint_positions(...)`, `set_target_body_yaw(...)`, `set_automatic_body_yaw(...)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| State read | `get_current_head_pose()`, `get_current_joint_positions()`, `get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| IMU | property `mini.imu` | [`examples/imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| High-level | `wake_up()`, `goto_sleep()`, `look_at_image(u, v, duration)`, `look_at_world(x, y, z, duration)` | [`examples/look_at`](https://huggingface.co/docs/reachy_mini/examples/look_at) |
| Move ABC | `Move` (source: [`motion/move.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py)), `GotoMove` (source: [`motion/goto.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/goto.py)), `InterpolationTechnique` | [`API/motion`](https://huggingface.co/docs/reachy_mini/API/motion) |
| Move playback | `play_move(move, play_frequency, initial_goto_duration, sound)`, `async_play_move(...)`, `cancel_move()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Recording | `start_recording()`, `stop_recording() -> Optional[List[Dict]]` | [`examples/recorded_moves`](https://huggingface.co/docs/reachy_mini/examples/recorded_moves) |
| Motors | `enable_motors(ids)`, `disable_motors(ids)`, `enable_gravity_compensation()`, `disable_gravity_compensation()` | [`examples/reachy_compliant_demo`](https://huggingface.co/docs/reachy_mini/examples/reachy_compliant_demo) |
| Media manager | property `mini.media`, `release_media()`, `acquire_media()`, property `media_released` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture) |
| REST / WebSocket | daemon API for non-Python clients | [`API/rest-api`](https://huggingface.co/docs/reachy_mini/API/rest-api), [`API/daemon`](https://huggingface.co/docs/reachy_mini/API/daemon), [OpenAPI](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/openapi.json) |
| Audio | `mini.media.audio.*` | [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), [`examples/sound_play`](https://huggingface.co/docs/reachy_mini/examples/sound_play), [`examples/sound_record`](https://huggingface.co/docs/reachy_mini/examples/sound_record), [`examples/sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Camera | `mini.media.camera.*` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`examples/take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| Quickstart | official entry point | [`SDK/quickstart`](https://huggingface.co/docs/reachy_mini/SDK/quickstart) |
| Core concepts | overarching concepts | [`SDK/core-concept`](https://huggingface.co/docs/reachy_mini/SDK/core-concept) |

## Acceptance Criteria
- [ ] Every actuator (head, antennas, body yaw) is listed with DoF, axes, and value range (or TBD)
- [ ] Every output (speaker, LED ring on the mic module) is listed with control modality; the absence of a programmable eye display is named explicitly
- [ ] Every sensor (microphones, camera, IMU, position feedback) is listed with read-API shape
- [ ] Three platforms (Reachy Mini, Reachy Mini Lite, Simulation) are named; differences in actuator set and CPU budget are listed
- [ ] The API reference overview links every controllable area to the hosted docs or the source module
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
- ~~Does the Reachy Mini head have 3 DoF or 6 DoF?~~ **Answered**: 6 DoF (Stewart platform); pose is a 4×4 transform matrix; builder `create_head_pose(x, y, z, roll, pitch, yaw, …)` from `reachy_mini.utils`.
- Which audio codecs does the SDK natively support — PCM/WAV, MP3, OGG? Which codec pipeline is optimal for dance with beat synchronisation? Resolve via [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture).
- Does the SDK offer a capability-discovery API to distinguish Reachy Mini, Reachy Mini Lite, and Simulation at runtime? The constructor argument `use_sim` is verified; an explicit variant property on the instance `> ⚠ TBD`.
- Does the SDK expose brown-out or current-spike telemetry, or does the behavior have to measure it itself?
- What is the canonical rest pose for the emergency stop? Proposal: every axis centred, antennas slightly upright, body yaw 0.
- Which update frequencies are realistic in a behavior tick — 50 Hz, 100 Hz, 200 Hz? Depends on variant and SDK implementation.
- Is there an official animation-authoring tool from Pollen Robotics (timeline editor), and is its output format a recommended composition for our move-primitive layer?
- How does the Stewart-platform head behave mathematically near singularities? Do we need to block singular regions in code?
- Which latency distribution (P50, P95, P99) is typical per layer? Without this distribution, a realistic dance timing is not plannable.
- Should animation principles (anticipation, follow-through, …) move into a separate `motion-design` spec once the patterns grow?
- Are the official Pollen Reachy Mini behaviors (e.g. "Hello", "Curious") released as reference implementations against which we can calibrate our pattern application?
- How far do classical animation patterns translate onto a 6-DoF robot without sliding into uncanny-valley territory? Empirical evaluation on real hardware required.
