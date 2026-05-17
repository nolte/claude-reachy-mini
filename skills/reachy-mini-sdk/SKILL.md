---
name: reachy-mini-sdk
description: >-
  Knowledge base and idiom guide for the `reachy_mini` Python SDK from
  Pollen Robotics / Hugging Face. Activate on any task that imports
  `reachy_mini`, instantiates the `ReachyMini` class, calls
  `goto_target`, `set_target`, `play_move`, `async_play_move`,
  `wake_up`, `goto_sleep`, `look_at_image`, `look_at_world`, accesses
  `mini.imu`, `mini.media`, builds a `Move` subclass, or otherwise
  programs the Reachy Mini desktop robot. Specifically triggers on
  phrasings like "build a behavior for Reachy Mini", "make Reachy nod /
  dance / wave the antennas", "open a connection to the Reachy Mini",
  "look at this point with Reachy", or any code touching `from
  reachy_mini import ...`. Do not activate on pure hardware bring-up,
  on simulation-only tasks without SDK calls, or on tasks that only
  publish a finished behavior — those have their own skills/agents.
tags: [reachy-mini, sdk, python, robotics]
---

# Reachy Mini SDK

Pinned SDK version: **`reachy_mini==1.7.1`** (daemon on a verified Reachy Wireless, 2026-05-13) / **`reachy_mini==1.7.2`** (locally installed in the plugin's IK-helper venv). Update through a deliberate spec revision when Pollen bumps either.

## Source of truth (in order, on conflict)

1. **Hosted SDK docs** — <https://huggingface.co/docs/reachy_mini/>
2. **SDK source code** — <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
3. **Plugin's own normative references** —
   - [`reachy-mini/control-surface`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md) (high-level control architecture and motion composition)
   - [`reachy-mini/motor-positions`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) (per-joint URDF limits, IK polytope, canonical poses verbatim from the SDK source, T1–T8 live-verified targets, recovery triage)
   - [`reachy-mini/motion-anomaly-detection`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motion-anomaly-detection/de.md) (four anomaly classes with binding rules, pre-flight / live / post-hoc detect signals, unified anomaly-event-record schema — this skill is the **pre-flight consumer**: Class A pose-range, Class B pose-delta/dt, Class D local-IK check)
   - [`reachy-mini/daemon-rest-api`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md) (REST surface for non-Python clients)
4. **This skill** — curated summary; loses to the sources above on conflict

Before producing API-shaped code, **open the relevant doc page or source module** and confirm the exact signature. Do not paste signatures from memory.

## When this skill activates

- imports of `reachy_mini` or symbols from it
- references to `ReachyMini`, `goto_target`, `set_target`, `play_move`, `async_play_move`, `Move` subclasses, `look_at_*`, `wake_up`, `goto_sleep`, `mini.imu`, `mini.media`
- requests to control head pose (4×4 matrix), antennas, or body yaw
- composing motion via the `Move` ABC

## When NOT to activate

- pure hardware bring-up, calibration, firmware flashing → separate skill (planned)
- pure simulation work without SDK calls → separate skill (planned)
- publishing a finished behavior to Hugging Face → `reachy-app-publish-hf` (planned)
- audio beat / tempo detection alone → `audio-beat-tracking` (planned)

## Hardware platforms (per the official docs)

| Platform | Notes | Sim constructor |
|---|---|---|
| **Reachy Mini** (Wireless) | Built-in Raspberry Pi 4 Compute Module (CM4) + LiFePO4 battery; full feature set | n/a |
| **Reachy Mini Lite** | Tethered to a host computer; reduced compute on-robot | n/a |
| **Simulation** | Software-only; same `ReachyMini` API, no real motors | `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:` |

Docs: [Wireless](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/get_started) · [Lite](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini_lite/get_started) · [Simulation](https://huggingface.co/docs/reachy_mini/platforms/simulation/get_started)

## Safety limits (canonical in `reachy-mini/control-surface` + `reachy-mini/motor-positions`)

Hard-coded SDK ranges that every code snippet must respect. The high-level table here is for quick orientation; per-joint URDF limits, the IK-polytope vs. mechanical-safety distinction, and live-verified extreme poses live in [`reachy-mini/motor-positions`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md). The motion-composition narrative (frames, easing, anticipation) lives in [`reachy-mini/control-surface`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md). The four-class detection methodology — pre-flight pose-range check (Class A), pose-delta/dt jerk threshold of `0.16 rad/sample` (Class B), antenna-rest-pose deadband (Class C), and local-IK check against URDF limits (Class D) — lives in [`reachy-mini/motion-anomaly-detection`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motion-anomaly-detection/de.md); snippets that drive the head **MUST** apply at least the Class A pre-flight check (Pollen nominal range as binding layer, not the IK polytope).

| Axis | Min | Max | SDK source |
|---|---|---|---|
| Head pitch / roll | −40° | +40° | `analytical_kinematics.py` plus the official hardware datasheet |
| Head yaw | −60° | +60° | as above |
| Head yaw relative to body | — | ±65° | `max_relative_yaw` |
| Body yaw | −155° | +155° | `max_body_yaw=np.deg2rad(160)` |
| Antenna (each) | −180° | +180° | URDF |

**Validity layering — only the innermost layer is binding for live motion:** IK polytope (the analytical solver accepts paths up to ±π per Stewart actuator, since it only honours the `kinematics_data.json` software bound) ⊋ URDF mechanical limits (the asymmetric per-actuator ranges in `robot.urdf`, e.g. stewart_1 −48°/+80°) ⊋ Pollen nominal operations range (±40° pitch/roll above). The full layering and the rationale for why the inner layer is mandatory live in motor-positions Layer 2 §"Three layers of validity". A snippet that targets the head **MUST** guard against the Pollen nominal range, not against the IK polytope — the latter has caused real self-collision on a Reachy Wireless (motor-positions Layer 4 §"Phase-B live incident 2026-05-12"). Do not rely on the daemon to clamp — it raises rather than clips.

**Cross-axis coupling — pitch bleed:** Stewart-platform geometry couples roll and heave-up into pitch. Live-verified 2026-05-13: a pure `roll = +25°` command bleeds **−3.8°** into pitch; a pure `z = +15 mm` command bleeds **+2.4°** into pitch. A motion that needs an isolated roll or heave must compensate the pitch component explicitly in the target pose. Pure pitch, pure yaw, and pure heave-down (T6) do not exhibit measurable bleed. Source: motor-positions Layer 2 §"T1–T8 live verification".

**Antenna rest-pose caveat**: each antenna servo has a per-antenna, sign-asymmetric deadband around `0°` that produces a visible ~±0.5° micro-wobble when the setpoint lands inside it. **SHOULD** keep antenna rest setpoints at `|setpoint| ≥ 5°` (preferably `≥ 10°` to match `INIT_ANTENNAS_JOINT_POSITIONS`); **SHOULD NOT** ease out to all-zero antennas if the robot then sits idle. Full empirical observation matrix + mitigation guidance in [`spec/reachy-mini/control-surface`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md#mechanische-und-elektrische-limitationen).

## API reference (with direct doc / source links)

### Construction & lifecycle

`ReachyMini` is a context manager that owns the connection.

```python
# Source: github.com/pollen-robotics/reachy_mini — verified shape
from reachy_mini import ReachyMini

with ReachyMini() as mini:
    ...  # interact with the robot
```

- `ReachyMini(...)` — constructor: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>
- Class API page: <https://huggingface.co/docs/reachy_mini/API/reachymini>
- Quickstart: <https://huggingface.co/docs/reachy_mini/SDK/quickstart>

### Smooth motion — `goto_target`

The canonical "move smoothly to a target over time" call. Head is a **4×4 transform**, antennas are a **list of two angles in radians**, body yaw is a **float in radians**.

```python
from reachy_mini import ReachyMini
from reachy_mini.utils import create_head_pose

with ReachyMini() as mini:
    mini.goto_target(
        head=create_head_pose(z=10, roll=15, degrees=True, mm=True),
        duration=1.0,
    )
```

- `goto_target(head, antennas, duration, method, body_yaw)` — <https://huggingface.co/docs/reachy_mini/API/reachymini>
- `create_head_pose(...)` (utility) — <https://huggingface.co/docs/reachy_mini/API/tools>
- Interpolation playground example — <https://huggingface.co/docs/reachy_mini/examples/goto_interpolation_playground>

### Direct set — `set_target` / `set_target_*`

Skip interpolation; write the target immediately. Use sparingly — bypassing easing produces mechanical-feeling motion (see the [control-surface spec](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md) anti-patterns).

- `set_target(head, antennas, body_yaw)` — <https://huggingface.co/docs/reachy_mini/API/reachymini>
- `set_target_head_pose(pose: np.ndarray)`
- `set_target_antenna_joint_positions(antennas: List[float])`
- `set_target_body_yaw(body_yaw: float)`
- `set_automatic_body_yaw(enabled: bool)` — IK-driven body yaw

### Pose & state reading

```python
head_pose = mini.get_current_head_pose()              # 4×4 np.ndarray
head_joints, antenna_joints = mini.get_current_joint_positions()
imu = mini.imu                                         # accelerometer / gyroscope / quaternion / temp
```

- `get_current_head_pose()` · `get_current_joint_positions()` · `get_present_antenna_joint_positions()` — <https://huggingface.co/docs/reachy_mini/API/reachymini>
- `mini.imu` (property) — IMU example: <https://huggingface.co/docs/reachy_mini/examples/imu>

### High-level behaviors

```python
mini.wake_up()                          # initial pose + sound + roll animation
mini.goto_sleep()                       # transition to sleep pose
mini.look_at_image(u, v, duration=0.5)  # orient toward a pixel in camera frame
mini.look_at_world(x, y, z, duration=0.5)  # orient toward a 3D point
```

- `wake_up()` · `goto_sleep()` · `look_at_image()` · `look_at_world()` — <https://huggingface.co/docs/reachy_mini/API/reachymini>
- Look-at example — <https://huggingface.co/docs/reachy_mini/examples/look_at>

### Motion composition — the `Move` ABC

A reusable motion is a subclass of `Move` with `duration` and `evaluate(t)`. The SDK plays it via `play_move` / `async_play_move`.

```python
# Source: github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py
from reachy_mini.motion.move import Move

class Nod(Move):
    @property
    def duration(self) -> float: return 0.6
    def evaluate(self, t: float):
        # return (head_pose_4x4_or_None, antennas_list_or_None, body_yaw_or_None)
        ...

mini.async_play_move(Nod())   # non-blocking
mini.play_move(Nod())         # blocking
mini.cancel_move()            # stop active move
```

- `Move` ABC — <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py>
- `GotoMove` and `InterpolationTechnique` — <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/goto.py>
- Recorded moves — <https://huggingface.co/docs/reachy_mini/examples/recorded_moves>
- Motion API page — <https://huggingface.co/docs/reachy_mini/API/motion>

### Motor & gravity compensation

- `enable_motors(ids)` · `disable_motors(ids)` · `enable_gravity_compensation()` · `disable_gravity_compensation()` — <https://huggingface.co/docs/reachy_mini/API/reachymini>
- Compliant-mode demo — <https://huggingface.co/docs/reachy_mini/examples/reachy_compliant_demo>

### Recording & playback

- `start_recording()` · `stop_recording()` — capture trajectory data
- `async_play_move(move, play_frequency, initial_goto_duration, sound)` · `play_move(...)` · `cancel_move()`
- Recorded-moves example — <https://huggingface.co/docs/reachy_mini/examples/recorded_moves>

### Media — camera, microphones, audio

```python
frame = mini.media.camera.get_frame()    # exact API: see media docs
mini.media.audio.play(...)
```

- `mini.media` (property) — <https://huggingface.co/docs/reachy_mini/API/media>
- Media architecture — <https://huggingface.co/docs/reachy_mini/SDK/media-architecture>
- Examples: [take_picture](https://huggingface.co/docs/reachy_mini/examples/take_picture) · [sound_play](https://huggingface.co/docs/reachy_mini/examples/sound_play) · [sound_record](https://huggingface.co/docs/reachy_mini/examples/sound_record) · [sound_doa](https://huggingface.co/docs/reachy_mini/examples/sound_doa)
- `release_media()` / `acquire_media()` — yield camera/mic to external code

### REST / WebSocket API (daemon-side)

For non-Python clients or remote control:

- REST API page — <https://huggingface.co/docs/reachy_mini/API/rest-api>
- OpenAPI schema (Pollen, may lag) — <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/openapi.json>
- Daemon API — <https://huggingface.co/docs/reachy_mini/API/daemon>
- JS SDK — <https://huggingface.co/docs/reachy_mini/SDK/javascript-sdk>
- **Live, authoritative endpoint inventory** — `http://<daemon-host>:8000/openapi.json` (the running daemon), mirrored as a flat existence table in [`reachy-mini/daemon-rest-api`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md). The Pollen-hosted OpenAPI link above can lag the live schema by several point releases — when scripting REST calls, cross-check against the live source.

### Reading live state with the `head_joints` vector

`GET /api/state/full` returns a sparse view by default; to receive the Stewart-joint vector `[body_yaw, stewart_1..stewart_6]`, append `?with_head_joints=true`. Verified live 2026-05-13 on a Reachy Wireless v1.7.1: without that query parameter, `head_joints` comes back as `null` even when the daemon backend is fully alive. The pattern is the same for `with_target_head_pose`, `with_target_head_joints`, `with_target_body_yaw`, `with_target_antenna_positions`, `with_passive_joints`, and `with_doa` — each is opt-in.

### Daemon liveness — what `backend_status.ready` does and does not tell

`GET /api/daemon/status.backend_status.ready` and `.last_alive` are **not synced** to the actual polling-loop progress in `reachy_mini==1.7.1`. The daemon's `self.ready` (a `threading.Event`) is set inside the loop, but never propagated to the JSON-serialised `_status.ready` field. The same desync affects `_status.last_alive`. Authoritative liveness signals are: (1) `mean_control_loop_frequency > 40 Hz` with `nb_error == 0`, (2) `head_joints` populated when requested with `?with_head_joints=true`, and (3) `head_pose` components micro-drift between successive reads. When all three fail, the bus is genuinely hung — see [`reachy-mini/motor-positions`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) §"Stage-3 recovery paths" for the diagnosis flow (`/proc/<pid>/task/*/wchan`, IMU/I²C bypass override, `i2cset` soft-reset, Dynamixel Wizard, Pollen support).

## Method choice — `goto_target` vs. `set_target`

- **`goto_target(head=<4×4>, antennas=[r, l], body_yaw, duration, method)`** — default for choreographed motions ≥ 0.5 s. Built-in interpolation (`MIN_JERK` is the default; `LINEAR`, `EASE_IN_OUT`, `CARTOON` are also available). The call blocks until the move finishes.
- **`set_target(head, antennas, body_yaw)`** — real-time path for high-frequency loops at 50–100 Hz tick rate, **single-owner loop**. No interpolation; you supply each step yourself.
- **MUST NOT** mix `goto_target` and `set_target` from competing call sites — they overwrite each other and produce jerky or stalled motion. Pick one method per behavior; if you need both, sequence them so they never run concurrently.
- Wake/sleep guard: if the robot is in the sleep pose with motors off, motion calls are silently ignored. Bring the head awake first (per `reachy_mini.utils.wake_up` / equivalent) before issuing any pose targets.
- Source: Pollen `motion-philosophy.md` and `control-loops.md`.

## Safe-torque anti-jerk pattern

Toggling motors cold (`enable_motors()` / `disable_motors()` straight) makes the head snap or fall. Pollen's safe-torque sequence is mandatory whenever a behavior toggles motor state:

1. **Before `disable_motors()`** — `goto_target(head=SLEEP_HEAD_POSE, ...)` so the head reaches a mechanically safe pose **under torque** before torque drops.
2. **Before `enable_motors()`** — set the goal to the *current* pose first (a short `goto_target` with `duration ≈ 0.05 s`); only then enable. This prevents motors from snapping to whatever the last commanded goal was.
3. **Mixed-motor states** (some IDs enabled, some not — typically after a partial failure): do not patch up. Fully `disable_motors()` everything, then re-enable sequentially under the same anti-jerk rule.

Source: Pollen `safe-torque.md`.

## Drift check

- Re-validate every API link on each `reachy_mini` major release, or quarterly — whichever comes first.
- When the SDK source disagrees with this skill, the source wins. Update the skill (and the spec) rather than work around it.
- Bump the pinned SDK version in the header above; never let it silently fall behind.

## Boundaries to neighbouring skills (planned)

- new behavior **scaffolding** → `app-scaffold`
- bidirectional **Home Assistant** wiring → `home-assistant-bridge`
- audio **beat / tempo detection** for dance behaviors → `audio-beat-tracking`
- live **on-device test / deploy** of a behavior → agent `reachy-mini-on-device`
- **read-only state inspection** of a running daemon (one-shot REST snapshot, recovery diagnosis) → `reachy-mini-inspect`

## Hard rules

- **MUST NOT** invent `pan / tilt / roll`-style argument lists for head motion — head pose is a **4×4 transform**, built via `create_head_pose(...)` from `reachy_mini.utils`.
- **MUST NOT** paste API signatures from memory or from older Reachy SDKs (Reachy 2, Reachy Pro) — the Reachy Mini API is its own surface.
- **MUST** name the verified SDK version next to non-trivial code examples and link the relevant doc page.
- **MUST** delegate to the neighbouring skills/agents listed above instead of growing this skill into them.
