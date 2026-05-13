# Reachy Mini Control Surface and Motion Design

Status: draft

## Context
Anyone writing skills, behaviors, or agents in this plugin needs a canonical reference for *which* elements of the Reachy Mini are actually controllable, *how* they are addressed, and *under which limits* you compose natural-looking motion from them. Without this specification every implementation reconstructs the hardware reality from scratch — usually from half-baked training samples, which on first real device contact either risks the hardware or produces mechanical-feeling motion. This spec consolidates the publicly known hardware and SDK properties of Reachy Mini, describes each control layer the SDK exposes, and for every mechanical and electrical limitation states how it shapes motion composition. It is the normative knowledge base against which the `reachy-mini-sdk` skill validates its snippets, and against which the `app-scaffold` skill anchors its templates.

> 🔗 **Deep references.** Per-joint URDF limits, the Stewart asymmetry pattern, canonical poses taken verbatim from the SDK source, T1–T8 live verification, the three-layer validity model (IK polytope ⊋ URDF mechanical ⊋ Pollen nominal), motor mode names with the Placo constraint, backend-health diagnosis, and recovery triage all live in [`reachy-mini/motor-positions`](../motor-positions/en.md). This spec names the *operating ranges*; motor-positions provides the *exact values* and their verification on the real device.

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

| Subsystem | DoF | Motors | Representation | Write API | Read API | Pose / joint limits (verified) | Platforms |
|---|---|---|---|---|---|---|---|
| Head | 6 (Stewart platform: 3 rotation + 3 translation) | 6× Dynamixel XL330-M288-T | 4×4 transform matrix; builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees, mm)` | `goto_target(head=...)`, `set_target(head=...)`, `set_target_head_pose(pose)` | `get_current_head_pose() -> np.ndarray` (4×4) | pitch and roll: ±90° (upright constraint, [`analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)); yaw relative to body: ±65° max (`max_relative_yaw`); translation x/y/z within the IK-reachable envelope (`head_z_offset` from `kinematics_data.json`) | all |
| Antennas (pair) | 2 (1 DoF per antenna) | 2× Dynamixel XL330-M077-T | `List[float]` with two joint angles (left, right) | `goto_target(antennas=...)`, `set_target(antennas=...)`, `set_target_antenna_joint_positions(antennas)` | `get_present_antenna_joint_positions() -> List[float]` | per antenna: -π to +π rad (full rotation), velocity limit 8 rad/s, effort limit 10 N·m ([URDF](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)) | all |
| Body yaw | 1 (base rotation) | 1× custom Dynamixel XC330-M288-PG | `float` | `goto_target(body_yaw=...)`, `set_target(body_yaw=...)`, `set_target_body_yaw(value)`, `set_automatic_body_yaw(enabled)` | part of `get_current_joint_positions()` | ±160° (`max_body_yaw=np.deg2rad(160)`, [`analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)) | all (Wireless **and** Lite, soft-state in Simulation) |

Stewart-platform joint limits (low-level, abstracted by IK; from [`robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)): each of the six actuators `stewart_1`..`stewart_6` has a value range roughly between -1.396 rad (-80°) and +1.396 rad (+80°) — asymmetric per joint —, velocity limit 8 rad/s, effort limit 10 N·m. These are the **hardware bound**; the effective head-pose reachability is tighter and enforced by IK.

**Nominal operations range** (official hardware datasheet, image `dof_table.png` on [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)): the ranges below are the *recommended* operations envelope to compose behaviors against. They are tighter than the kinematic maxima above and ensure that the device reaches the pose reliably.

| Axis | Min | Max |
|---|---|---|
| Tx (head) | −1.5 cm | +2.5 cm |
| Ty (head) | −4 cm | +4 cm |
| Tz (head) | −4 cm | +2.5 cm |
| Rx (head roll) | −40° | +40° |
| Ry (head pitch) | −40° | +40° |
| Rz (head yaw) | −60° | +60° |
| Rz (body yaw) | −155° | +155° |
| R (right antenna) | −180° | +180° |
| R (left antenna) | −180° | +180° |

**Motor IDs on the Dynamixel bus** (from `motors_detail.png` on the same hardware page) — directly usable with `enable_motors(ids)` / `disable_motors(ids)`:

| Subsystem | Label | ID |
|---|---|---|
| Body yaw | Motor F | 10 |
| Stewart 1 | Motor 1 | 11 |
| Stewart 2 | Motor 2 | 12 |
| Stewart 3 | Motor 3 | 13 |
| Stewart 4 | Motor 4 | 14 |
| Stewart 5 | Motor 5 | 15 |
| Stewart 6 | Motor 6 | 16 |
| Right antenna | — | 17 |
| Left antenna | — | 18 |

> **Note on source consistency**: The official hardware page contains two text-vs-image inconsistencies; we follow the prose, not the image labels:
> 1. `motors_detail.png` labels the antenna motors as "XL330 M288-T" and the body motor as "XL330 M288PG-T". The accompanying text correctly names them **XL330-M077-T** (antennas) and **custom XC330-M288-PG** (body), each with a Robotis datasheet link.
> 2. In `electronics.png` the Wireless control electronics is labelled "Wireless Control Board"; the prose calls it "CM4 Controller Board" — both refer to the same component.

Concrete write-API docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>. Pose builder: <https://huggingface.co/docs/reachy_mini/API/tools>.

Inventory-maintenance requirements:

- **MUST** document each axis with identifier, value range, unit, and default pose as soon as the SDK docs are available
- **MUST** indicate per axis whether the SDK API accepts absolute targets, incremental targets, or both
- **MUST** mark axes that exist only on one platform (body yaw is present on Wireless and Lite — Stewart and antennas as well; only the IMU telemetry is Wireless-only)
- **SHOULD** state per-axis velocity and acceleration limits that follow from the mechanical build (velocity limit 8 rad/s per active joint per URDF; acceleration limit not directly stated — bounded by effort 10 N·m and inertia)

### Hardware inventory — outputs (non-mechanical)

| Subsystem | Property | Controllability |
|---|---|---|
| Speaker | 5 W @ 4 Ω, single | audio playback via `mini.media.audio.*` (asynchronous GStreamer pipeline); push API expects `F32LE` samples at 48 kHz, 2 channels (constants: [`AudioBase.SAMPLE_RATE`, `AudioBase.CHANNELS`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_base.py)). `play_sound(file=...)` decodes arbitrary formats via GStreamer `playbin` — WAV, MP3, OGG, FLAC are implicitly reachable. Volume 0–100 ([`SetVolumeCmd`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)) |
| LED ring on the microphone module | LEDs on the ReSpeaker / mic-array board; registers `LED_EFFECT`, `LED_BRIGHTNESS`, `LED_GAMMIFY`, `LED_SPEED` | via `audio_control_utils` ([source](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_control_utils.py)); programmatically writable |
| ~~Display / eye screen~~ | **Reachy Mini ships no programmable eye display.** The "eyes" are mechanical 3D-printed parts (`pp01079_back_big_eye`, `pp01080_back_small_eye`) on the head shell — not controllable. The Pollen control app's "Expressions" are motion compositions of head pose + antenna positions, not display content. | not controllable |

Requirements:

- Audio playback runs **asynchronously** through a GStreamer `appsrc` pipeline (verified in [`audio_gstreamer.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_gstreamer.py)); the behavior tick is not blocked
- **SHOULD** include recommendations for audio-latency measurement and compensation patterns once SDK behavior is verified
- **MUST** model the LED ring as a status / audio-feedback affordance — not as an "eye expression", because it is not located in the eye area of the robot
- **MUST NOT** assume or simulate an "eye display" as a controllable element — Reachy Mini does not ship one

### Hardware inventory — sensors / inputs

| Subsystem | Property | Read API | Docs |
|---|---|---|---|
| Microphone array | 4× PDM MEMS digital, 16 kHz hardware sample rate, -26 dB FS sensitivity, 64 dBA SNR, direction-of-arrival capable; based on the Seeed Studio reSpeaker XMOS XVF3800; mic volume 0–100 ([`SetMicrophoneVolumeCmd`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)) | via `mini.media` (`MediaManager`) | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), example [`sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Camera | Raspberry Pi v3 wide angle (Sony IMX708, 12 MP, autofocus, 120° field of view); mounted in the central bridge lens between the (purely decorative) eyes — the u/v frame in `look_at_image(u, v, …)` references this position, not the eyes; concrete stream parameters `> ⚠ TBD: validate against current backend` | via `mini.media.camera` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), example [`take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| IMU (**Wireless only**) | `accelerometer: list[float]`, `gyroscope: list[float]`, `quaternion: list[float]`, `temperature: float`; data is published by the daemon at 50 Hz ([`ImuDataMsg`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)). On Lite and Simulation, `mini.imu` returns `None`. | `mini.imu` (property → `Dict \| None`) | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini), example [`imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| Position feedback head | current 4×4 pose | `mini.get_current_head_pose() -> np.ndarray` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Position feedback antennas + joints | joint angles | `mini.get_current_joint_positions()`, `mini.get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Media release / acquire | yield camera/mic to external code | `mini.release_media()`, `mini.acquire_media()`, property `mini.media_released` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |

Requirements:

- **MUST** document a read method per actuator so behaviors can fetch the actually-reached pose without blocking the actuator
- **MUST** clarify whether sensor streams come blocking or as async iterators
- **MUST NOT** assume a sensor read is free — read frequency is bounded (see the latency section)

### Hardware platforms and their consequences

Pollen Robotics ships the `ReachyMini` API across three platforms, with the same methods but different compute and actuator profiles:

- **Reachy Mini** (Wireless) — own Wireless Control Board (Raspberry Pi 4 Compute Module CM4104016, 4 GB RAM, 16 GB flash) and Wireless Power Board, LiFePO4 battery (2000 mAh, 6.4 V, 12.8 Wh) with over-charge / over-discharge / over-current / short-circuit protection and a temperature sensor, 2.4–5 GHz dual-band patch antenna (2.79 dBi, omnidirectional); fully self-contained. Back-panel controls: see "Back-panel interface" section. Docs: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/get_started>
- **Reachy Mini Lite** — own Lite Control Board and Lite Power Board (no CM4, no battery), USB-C to a host computer for data plus high-level compute, external 6.8–7.6 V power supply on a separate connector (**not** over USB-C); same actuator set as Wireless (Stewart, antennas, body yaw, mic array, camera, speaker) — only IMU telemetry is Wireless-only. Docs: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini_lite/get_started>
- **Simulation** — software-only, same `ReachyMini` API without real motors; usable for CI, tests, code shake-out without a device. Two paths: (a) `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:` boots a sim daemon in the same Python process, or (b) external daemon via `reachy-mini-daemon --sim` (or, without system deps, `reachy-mini-daemon --mockup-sim --no-media --headless`) plus `with ReachyMini() as mini:` from a second process. Plain `use_sim=True` without `spawn_daemon=True` is **not** a valid path — the client then searches for an external daemon and fails. The full `--sim` path needs GStreamer system packages (`gir1.2-gst-plugins-base-1.0` and friends) and MuJoCo. Docs: <https://huggingface.co/docs/reachy_mini/platforms/simulation/get_started>

Requirements:

- **MUST** make it visible in code which platform a motion is written for; a high-frequency motion designed for Wireless must not silently run on Lite or Simulation
- **SHOULD** provide a way to detect the platform at runtime (capability discovery; `> ⚠ TBD` whether the SDK exposes a direct variant property; the constructor argument `use_sim` is verified to exist)
- **MUST** signal explicitly to the caller when a motion that depends on hardware capabilities (e.g. real audio playback, IMU values) runs in Simulation, instead of silently no-op'ing

### Mechanical profile

From the official hardware datasheet (image `reachy_mini_dimensions.png` on [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)):

- Dimensions (extended, antennas upright): **30 cm height × 20 cm depth × 15.5 cm width**
- Body height in nominal position (without antennas): 27.5 cm
- Antenna height (alone): 14 cm
- Sleeping pos: compact silhouette, ~15.5 cm tall (antennas fold to the sides)
- Mass: **1.475 kg**
- Materials: ABS, PC, aluminium, steel

Behavior implications:

- **MUST** choose look targets in `look_at_world(x, y, z, …)` so that the bridge camera's field of view (120°, ~27.5 cm above the standing surface in nominal pose) covers the target — the eyes are decorative, the camera sits between them
- **MUST** compute placement preconditions from the 20 × 15.5 cm footprint plus extra clearance for antenna motion
- **SHOULD** use `goto_sleep()` for transport and storage commands — the device assumes the compact sleeping pose

### Back-panel interface (Wireless)

Four controls sit on the back of the housing (image `back_interface.png` on [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)):

| Element | Function | Behavior relevance |
|---|---|---|
| **USB-C** | Peripheral **output** (USB stick, USB audio device, …); CM4 acts as USB host | **does not charge the device** — power has its own connector |
| **Power supply** | Dedicated charge / supply connector, 6.8–7.6 V | the actual power input — not via USB-C |
| **On/Off switch** | Physical hardware switch | a hard-off bypasses the SDK emergency-stop path — the device may be left in an unsecured pose if powered off before `goto_sleep()` |
| **LED indicator** | Status LED (power / boot / ready) | the only power-/boot-status surface without API read access — not programmatically queryable |

Requirements:

- **MUST** keep charge / supply logic separate from the USB-C output — they are physically distinct connectors with different functions
- **MUST** call `goto_sleep()` before a planned hard-off, to bring actuators into a safe pose
- **SHOULD** name the on/off switch and the status LED in behavior tutorials as a non-API surface, so consuming skills do not model them as a read source

### Control layers

Four layers, from direct-write (low level) to narrative (high level), with verified method names:

1. **Direct-set** — target is written immediately, no interpolation
   - methods: `set_target(head, antennas, body_yaw)`, `set_target_head_pose(pose)`, `set_target_antenna_joint_positions(antennas)`, `set_target_body_yaw(value)`
   - docs: <https://huggingface.co/docs/reachy_mini/API/reachymini>
   - usage: only when easing is provided by a higher layer or by your own trajectory — otherwise produces mechanical-feeling motion

2. **Goto-target** — smooth goto over a named duration, SDK handles interpolation
   - method: `goto_target(head, antennas, duration, method, body_yaw)` with `method: InterpolationTechnique`
   - pose builder: `create_head_pose(...)` from `reachy_mini.utils`
   - available interpolation modes (verified in [`utils/interpolation.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/utils/interpolation.py)): `LINEAR` (linear), `MIN_JERK` (minimum-jerk trajectory, **default**), `EASE_IN_OUT` (quadratic ease-in/out), `CARTOON` (elastic overshoot)
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

- **MUST** measure typical end-to-end latency (software command → mechanical reaction) against real hardware once available (`> ⚠ TBD: validate against real hardware`)
- The daemon publishes `JointPositionsMsg`, `HeadPoseMsg`, and (on Wireless) `ImuDataMsg` at **50 Hz** ([`io/protocol.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)). Behavior tick loops should align with that — higher frequencies add no value because state reads only refresh every 20 ms anyway
- Audio pipeline latency (GStreamer constants from [`audio_gstreamer.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_gstreamer.py)): sink buffer 50 ms (`PLAYBACK_SINK_BUFFER_TIME_US = 50000`), sink latency 5 ms (`PLAYBACK_SINK_LATENCY_TIME_US = 5000`), gap reset 200 ms (`PLAYBACK_GAP_RESET_NS = 200_000_000`)
- **MUST NOT** recommend tick frequencies above 50 Hz — the daemon does not refresh state faster; higher frequencies are waste or cause actuator stutter
- **SHOULD** align audio-and-motion synchronicity to the ~50 ms audio buffer instead of assuming zero latency

### Dependencies

- **`reachy_mini` SDK version** — pinned in the consuming repo; every motion depends on this version's API shape. Currently: `reachy_mini==1.7.0` (verified via [`pyproject.toml`](https://github.com/pollen-robotics/reachy_mini/blob/main/pyproject.toml))
- **Device firmware version** — mismatch between SDK and firmware can cause connect or behavior errors; verify on every first connect (`DaemonStatus.version` exposes the daemon version)
- **Python version** — `>=3.10` per SDK requirement (verified via `requires-python` in [`pyproject.toml`](https://github.com/pollen-robotics/reachy_mini/blob/main/pyproject.toml))
- **Hardware platform** — Wireless / Lite / Simulation influences CPU budget and power supply; the actuator set is identical on Wireless and Lite
- **Host connectivity** — Lite needs USB-C to a host plus external 6.8–7.6 V; Wireless needs Wi-Fi (for code sync) and runs from the battery; Simulation needs no host outside the Python process
- **System audio stack** — audio playback runs over GStreamer with an OS-specific backend (PulseAudio / ALSA on Linux, WASAPI on Windows, CoreAudio on macOS); no Wireless-vs-Lite difference at the SDK level

### Mechanical and electrical limitations

- **End stops** — every axis has a mechanical end stop; the SDK should protect against damage, but the implementation must not aim into the end stop in the first place
- **Velocity** — 8 rad/s per active joint per [URDF](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf); a command that would move the joint faster is clamped to the limit by the daemon. **Acceleration** — no direct URDF limit; effectively bounded by 10 N·m effort, Stewart-platform inertia, and servo characteristics (`> ⚠ TBD: measure once hardware is on hand`)
- **Current draw** — many simultaneous motions (head full + both antennas + body) can briefly drop the voltage on battery operation; depending on battery state the system reacts with brown-out protection
- **Thermal budget** — sustained motion at high frequency generates heat; the SDK may expose temperature telemetry (`> ⚠ TBD`)
- **Collisions** — **documented on 2026-05-12 by a real self-collision incident:** the analytical IK accepts pose targets up to ±π per Stewart joint (software default from `kinematics_data.json`) and performs no self-collision check (`engine: AnalyticalKinematics, collision check: false` from `GET /api/kinematics/info`). Pose targets outside the Pollen nominal operations range (±40° pitch/roll, ±60° head-yaw) are **IK-mathematically solvable but NOT mechanically safe** — they led to a head-against-body collision on a real Reachy Wireless. Binding for every live motion is the **innermost** of the three validity layers (Pollen nominal); the IK polytope and the URDF mechanical limits are not enough. Details + recovery triage in [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2 §"Three layers of validity" and Layer 4 §"Phase-B live incident 2026-05-12"
- **Pitch bleed on roll and heave-up** — **live-verified 2026-05-13:** the Stewart-platform geometry couples pure-roll and pure-heave-up commands into pitch. Quantitatively: `roll = +25°` pulls pitch down by **−3.8°**; `z = +15 mm` pulls pitch forward by **+2.4°**. Pure pitch, pure yaw, and pure heave-down (negative z) show no measurable bleed. A motion that needs isolated roll or heave must compensate the pitch component explicitly in the target pose — this is not a calibration drift but systematic platform coupling. Source: [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2 §"T1–T8 live verification"

Requirements:

- **MUST** check the combinatorial collision prohibitions before issuing a motion if the SDK doesn't enforce them itself
- **MUST** avoid current spikes by not driving every actuator at maximum speed simultaneously
- **SHOULD** consume temperature telemetry once the SDK exposes it — pause for a cool-down on threshold breach

### Safety limits

- **MUST** keep an emergency-stop path reachable that ends every running motion and drives to a rest pose. Canonical rest pose: `INIT_HEAD_POSE = np.eye(4)` (4×4 identity matrix, head centred) and `INIT_ANTENNAS_JOINT_POSITIONS = [-0.1745, +0.1745]` rad = `[-10°, +10°]` (verified in [`reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)). `SLEEP_HEAD_POSE` (used by `goto_sleep()`) is an alternative — pitch **+24.4°** in xyz-Euler (head tipped forward-down; live-verified 2026-05-13 with real pitch ≈ +26°). Exact matrices and IK joint solutions are in [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 3
- **MUST** put the device into a safe pose after an unexpected disconnect or program abort, instead of leaving a motion "frozen"
- **MUST NOT** override current, force, or velocity thresholds via SDK parameters without an explicit justification in the behavior

### Patterns for natural, fluid motion

The following principles are translated from classical animation onto a 6-DoF head + 2-antenna system. They are not optional when the result should feel "alive".

1. **Easing (slow-in / slow-out)** — no linear motion. `goto_target` accepts a `method` argument; the default is `InterpolationTechnique.MIN_JERK` (minimum-jerk trajectory — already smooth and organic). Available modes: `LINEAR`, `MIN_JERK` (recommended default), `EASE_IN_OUT`, `CARTOON`. Use `LINEAR` only when the motion is meant to feel mechanical.
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
- [ ] The nominal operations range from the official hardware datasheet is given alongside the kinematic maximum range
- [ ] Motor IDs (body, Stewart 1–6, antennas) are tabled and mapped to the `enable_motors` / `disable_motors` API
- [ ] Mechanical profile (dimensions extended + sleeping pos, mass, materials) is listed
- [ ] Back-panel interface (USB-C, power supply, on/off switch, status LED) is documented as its own table
- [ ] Every output (speaker, LED ring on the mic module) is listed with control modality; the absence of a programmable eye display is named explicitly
- [ ] Every sensor (microphones with SNR + chip name, camera with FoV + position, IMU, position feedback) is listed with read-API shape
- [ ] Three platforms (Reachy Mini, Reachy Mini Lite, Simulation) are named; differences in actuator set and CPU budget are listed; Lite has its own Lite Control + Lite Power Board (not "just a dumb cable to the host")
- [ ] Drift risk between official prose and images is captured as a sources note
- [ ] The API reference overview links every controllable area to the hosted docs or the source module
- [ ] Four control layers are described with use case, API shape, interrupt model
- [ ] A unified unit system (rad, mm, s) is set
- [ ] Latency and update frequency are listed as required fields, even if the values are TBD today
- [ ] Six dependency fields (SDK, firmware, Python, variant, connectivity, audio stack) are listed
- [ ] Mechanical, electrical, and thermal limitations are named; an emergency-stop path is defined
- [ ] At least 10 patterns for natural, fluid motion are documented
- [ ] At least 9 anti-patterns are documented
- [ ] Observability (position read, latency trace, telemetry) is captured as a requirement
- [ ] Hardware numbers that are not verifiable from the official docs or SDK source carry a `⚠ TBD` marker
- [ ] The `reachy-mini-sdk` skill points at this spec as the canonical knowledge base
- [ ] The `app-scaffold` skill points at this spec for move primitives and default easings

## References
- Upstream SDK repo (canonical source for every constant and class referenced in the tables above): <https://github.com/pollen-robotics/reachy_mini>
- SDK source tree (`ReachyMini`, IO, Media, Motion, Daemon, Apps, Tools): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Motion module (`Move` ABC, easing modes `MIN_JERK` / `CARTOON`, `goto`, `recorded_move`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- IO protocol (every `*Cmd` / `*Msg` type used in the requirements): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- Media stack (camera, audio, GStreamer pipelines, mic DOA): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- Daemon (status, app lock, REST API): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- API docs (MDX sources for `reachymini`, `media`, `motion`, `daemon`, `apps`, `tools`, `utils`, REST API, OpenAPI schema): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/API>
- SDK concept docs (Quickstart, Core Concept, Apps, Python / JavaScript SDK, Media architecture): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Platform profile docs (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- ~~Does the Reachy Mini head have 3 DoF or 6 DoF?~~ **Answered**: 6 DoF (Stewart platform); pose is a 4×4 transform matrix; builder `create_head_pose(x, y, z, roll, pitch, yaw, …)` from `reachy_mini.utils`.
- ~~Which audio codecs?~~ **Answered**: push API expects `F32LE` at 48 kHz, 2 channels. `play_sound(file=...)` decodes arbitrary formats via GStreamer `playbin`.
- ~~Update frequency?~~ **Answered**: the daemon publishes at 50 Hz; higher tick frequencies are not useful.
- ~~Does Reachy Mini have an IMU?~~ **Answered**: yes, **Wireless only**; fields: accelerometer, gyroscope, quaternion, temperature.
- Does the SDK offer a capability-discovery API to distinguish Reachy Mini, Reachy Mini Lite, and Simulation at runtime? The constructor argument `use_sim` is verified; an explicit variant property on the instance `> ⚠ TBD: read DaemonStatus.no_media / camera_specs_name as proxy`.
- Does the SDK expose brown-out or current-spike telemetry, or does the behavior have to measure it itself?
- ~~What is the canonical rest pose for the emergency stop?~~ **Answered**: `INIT_HEAD_POSE = np.eye(4)` plus `INIT_ANTENNAS_JOINT_POSITIONS` (~10° offset). `SLEEP_HEAD_POSE` is an alternative used by `goto_sleep()`.
- ~~Which tick frequencies are realistic — 50 / 100 / 200 Hz?~~ **Answered**: the daemon publishes at 50 Hz; higher tick frequencies are wasted.
- Is there an official animation-authoring tool from Pollen Robotics (timeline editor), and is its output format a recommended composition for our move-primitive layer?
- How does the Stewart-platform head behave mathematically near singularities? Do we need to block singular regions in code?
- Which latency distribution (P50, P95, P99) is typical per layer? Without this distribution, a realistic dance timing is not plannable.
- Should animation principles (anticipation, follow-through, …) move into a separate `motion-design` spec once the patterns grow?
- Are the official Pollen Reachy Mini behaviors (e.g. "Hello", "Curious") released as reference implementations against which we can calibrate our pattern application?
- How far do classical animation patterns translate onto a 6-DoF robot without sliding into uncanny-valley territory? Empirical evaluation on real hardware required.
