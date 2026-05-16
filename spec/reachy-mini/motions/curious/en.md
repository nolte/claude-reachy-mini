# Motion Sequence: Curious (`curious`)

Status: draft

## Context
The classic curiosity gesture: Reachy tilts its head to the side, holds the pose to inspect, tilts to the other side, and rights itself again. Use cases: asking a question, "what next?", response to new input, mode indicator for listening / attention. Pollen's notebook describes this expression directly as "tilt head + asymmetric antennas".

## Characteristics
- A clear roll tilt (±15° to ±20°) as the main feature — that is the canonical curious gesture
- **Asymmetric antennas**: the antenna on the "down" side moves slightly forward, the other backward — reinforces the inspecting read
- Medium tempo (~3.3 s); only `MIN_JERK` and `EASE_IN_OUT`, no `CARTOON`, no `LINEAR`
- Body yaw slightly in the tilt direction — the body "leans along"
- Mini yaw modulation during the hold phases — small inspecting search motion

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Lift (anticipation) | 0.20 | (0, 0, +3, 0, +5, 0) | (+5, +5) | 0 | `MIN_JERK` | small lift as preparation |
| 2 | Tilt left | 0.50 | (0, 0, +5, +20, +6, -8) | (-15, +25) | -5 | `EASE_IN_OUT` | main motion — left-tilted; right antenna forward, left back |
| 3 | Hold left with mod | 0.80 | (0, 0, +5, +20, +6, -8 (±3°)) | (-15, +25) | -5 | `MIN_JERK` (idle mod) | slow yaw modulation — "inspecting" |
| 4 | Tilt right | 0.50 | (0, 0, +5, -20, +6, +8) | (+25, -15) | +5 | `EASE_IN_OUT` | mirrored tilt to the other side |
| 5 | Hold right with mod | 0.60 | (0, 0, +5, -20, +6, +8 (±3°)) | (+25, -15) | +5 | `MIN_JERK` (idle mod) | slightly shorter than phase 3 |
| 6 | Release | 0.60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth dissolve to the neutral pose |

Total duration ≈ 3.20 s.

### Audio (optional)
A short questioning sound ("hmm?" sample, ≤ 400 ms), started with phase 2. Volume quiet to moderate (35–50). A second, lighter variant may optionally accompany phase 4.

### Idle modulation during the holds (phases 3 and 5)
Slow sinusoidal modulation on `yaw` (amplitude 3°, frequency 0.4 Hz) — the pose swings minimally back and forth, as if Reachy were inspecting something. Antennas stay static (the asymmetry is the statement). Pitch and roll keep their static values.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the small body-yaw offset (-5 / +5°) is smoothly tied to the head yaw via IK. Asymmetric antennas need no IK handling, they are set directly.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 3.20`, six phases.
- The asymmetric antennas are the key detail — `set_target_antenna_joint_positions([-15°, +25°])` (phase 2/3) and `[+25°, -15°]` (phase 4/5). Convention reminder: first list element is the left antenna, second is the right.
- Roll ±20° sits clearly within the pitch/roll limit of ±90°.
- The `EASE_IN_OUT` easing in phases 2 and 4 produces the organic "rolling-into" of the motion — `MIN_JERK` would feel too smooth.
- Yaw modulation in phases 3 and 5 is subtle (3° amplitude) — it should read as attentive, not nervous.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "curious" / "questioning" / "interested" (at least 4 out of 5)
- [ ] Total duration is 3.2 ± 0.3 s
- [ ] In the tilt phases, antennas are visibly asymmetric — front and back antenna differ
- [ ] Roll tilt reaches a visible ±15°–20° in phase 2
- [ ] Yaw modulation in the hold phases reads as "inspecting", not "nervous"
- [ ] Phase 4 mirrors phase 2 cleanly (sign flip on roll, yaw, body yaw, antenna values swapped)
- [ ] Audio (if enabled) sounds questioning, not exclamatory

## Anti-patterns
- Symmetric antennas — defeats the Pollen-canonical curious look
- Roll less than ±10° — tilt no longer reads
- `LINEAR` or `CARTOON` easing — wrong character for the gesture
- Hold phases without idle modulation — feels frozen, not inspecting
- Audio with an exclamatory character (loud "ah!") — clashes with the contemplative gesture

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
- Which antenna asymmetry reads strongest as "curious"? -15°/+25° is a first proposal — empirically test between 15° and 35°.
- Should the sequence be single-sided (left tilt or right tilt only) instead of both? Pro: shorter; con: less expressive.
- How does `curious` integrate with `look_at_image()` from the SDK? A look with tilt bias would be a natural future variant.
- Which audio file fits? Proposal: a short rising-tone "hmm?".
