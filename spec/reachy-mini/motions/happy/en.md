# Motion Sequence: Happy (`happy`)

Status: draft

## Context
An expression of joy that reads as "happy" without explanation: Reachy straightens up, perks the antennas like ears, springs twice gently, and eases out. Use cases: positive feedback ("task succeeded"), greeting, response to praise via voice or Home Assistant trigger.

## Characteristics
- Upright, slightly raised head — conveys energy and attention
- Antennas perked up (positive angles on both) — like alert ears
- Two short spring motions with `CARTOON` overshoot — reinforce liveliness
- Body yaw stays mostly centred; only a small wobble between springs
- Overall tempo: fast enough to read as energetic (2.5–3 s), not frantic

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

Pose convention: translation in mm, rotation in degrees, both as offset from the neutral pose (`INIT_HEAD_POSE = np.eye(4)`). Antenna angles in degrees relative to `INIT_ANTENNAS_JOINT_POSITIONS`. Body yaw in degrees. All values respect the `control-surface` constraint pitch/roll ≤ ±90°.

| # | Phase | Duration (s) | Head Δ (x mm, y mm, z mm, roll°, pitch°, yaw°) | Antennas (left°, right°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (dip) | 0.15 | (0, 0, -3, 0, -4, 0) | (-5, -5) | 0 | `MIN_JERK` | small counter-motion before lifting |
| 2 | Lift (main) | 0.40 | (0, 0, +10, 0, +14, 0) | (+30, +30) | 0 | `MIN_JERK` | "chest out, ears perked" |
| 3 | Spring 1 (bounce) | 0.30 | (0, 0, +6, 0, +8, 0) | (+25, +25) | -3 | `CARTOON` | first bounce with subtle roll/yaw play |
| 4 | Spring 2 (smaller bounce) | 0.25 | (0, 0, +9, 0, +12, 0) | (+30, +30) | +3 | `CARTOON` | second, smaller bounce in the opposite direction |
| 5 | Hold with idle | 0.80 | (0, 0, +8, 0, +10, 0) | (+28, +28) | 0 | `MIN_JERK` | idle breathing modulation during hold (see below) |
| 6 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth dissolve to the neutral pose |

Total duration ≈ 2.40 s.

### Audio (optional)
A short cheerful sound (≤ 600 ms), started at the beginning of phase 2, asynchronously via `mini.media.audio.*`. Account for the ~50 ms audio buffer latency — the sound start can be pushed 1–2 frames before the pose change so that audio and motion read as synchronous.

### Idle modulation during hold
During phase 5 a very small sinusoidal modulation on `pitch` (amplitude 1°, frequency 0.3 Hz) and `roll` (amplitude 0.5°, frequency 0.2 Hz, counter-phase). Antennas stay static. Prevents the "frozen" feel and supports the "idle breathing" pattern from `control-surface`.

### Body yaw and IK
`automatic_body_yaw=True` recommended so the body tracks the head yaw smoothly. With manual control, set body yaw exactly as in the table, never counter to the head yaw — otherwise the pose looks twisted (see `control-surface` pattern 10).

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.40` and an `evaluate(t)` that interpolates the six phases via time lookup; per-phase easing from `InterpolationTechnique`. Played via `mini.async_play_move(Happy(), sound="...")`.
- Alternatively as a sequential chain of `mini.goto_target(...)` calls with phase-specific `duration` and `method` — simpler for a first iteration, less suited for tick-accurate audio sync.
- Stewart-platform limits are honoured: pitch +14° and z +10mm sit well inside the IK-reachable envelope.
- Antennas at +30° sit well inside the ±π limit of the `right_antenna` / `left_antenna` joints.

## Acceptance Criteria
- [ ] External observers read the motion as "happy" or "joyful" without a prompt (at least 4 out of 5 observers)
- [ ] Total duration is 2.3 ± 0.2 s
- [ ] No visible pose jumps between phases (easing engages in every phase)
- [ ] Stewart joint limits are not reached in any phase (no daemon-side clamp)
- [ ] Antennas trail the head with a correct phase offset (the "follow-through" pattern from `control-surface`)
- [ ] Idle modulation in phase 5 is visible but subtle — not "nervous tremor"
- [ ] With audio enabled: sound starts at or before phase 2 and ends before phase 6
- [ ] A handover into a follow-up behavior does not break the motion — phase 6 either runs through cleanly or is ended via `cancel_move()`

## Anti-patterns
- `LINEAR` easing in the spring phases (3, 4) — feels mechanical, the character is lost
- Antennas moving synchronously with the head instead of with a small lag — feels stiff (see "follow-through" pattern)
- Pushing body yaw beyond ±10° instead of staying centred — feels "agitated" instead of happy
- Aligning phase durations to tick boundaries (20 ms) — no need for tick alignment, the daemon interpolates
- Loud or long audio (> 1 s) — clashes with the motion tempo

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Plugin references

- Pose values, joint limits, canonical poses (INIT/SLEEP) → [`reachy-mini/motor-positions`](../../motor-positions/en.md)
- Pose composition, IK-vs-mechanical-safety, pitch bleed on roll / heave-up → [`reachy-mini/control-surface`](../../control-surface/en.md) §"Mechanical and electrical limitations"
- Motion contains roll or heave-up components? Compensate pitch explicitly in the target pose (Stewart geometry coupling, live-verified 2026-05-13: roll +25° → −3.8° pitch; z +15 mm → +2.4° pitch)

## Open Questions
- Which concrete audio file is used? Proposal: a short rise borrowed from the SDK's `wake_up` sound as a reference.
- Should there be a "small" and a "large" happy variant (one bounce vs. two)? Depends on trigger context.
- How does the behavior react to a `cancel_move()` mid-phase 3? Plan a safe drift to the neutral pose.
- Which idle modulation frequency and amplitude feel most natural without tipping into "jittery"? Calibrate empirically.
