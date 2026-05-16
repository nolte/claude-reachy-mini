# Motion Sequence: Surprised (`surprised`)

Status: draft

## Context
A sudden reaction that reads as "surprised" or "startled": Reachy makes a tiny pre-flinch, snaps sharply upward, freezes briefly in the shock moment, micro-trembles, and then orients sideways with a questioning swing. Use cases: unexpected trigger (door sensor, sound), "what was that!", reaction to sudden motion in the camera.

## Characteristics
- Fast, sharp upward snap (positive Z + positive pitch) — conveys a "startle"
- Antennas opened up and outward (strong positive values) — like ears pricking up
- Frozen hold immediately after the snap — the shock freezes the pose briefly
- Optional micro-tremor (small amplitude, short duration)
- Following yaw orientation — "where was it?"
- Very fast tempo (~1.8 s); a hard `LINEAR` snap for the shock moment

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Pre-anticipation (flinch) | 0.05 | (0, 0, -2, 0, -3, 0) | (0, 0) | 0 | `LINEAR` | minimal pre-jump flinch, almost invisible |
| 2 | Upward snap | 0.15 | (0, 0, +15, 0, +18, 0) | (+40, +40) | 0 | `LINEAR` | sharp, fast upward snap |
| 3 | Frozen hold (shock) | 0.40 | (0, 0, +15, 0, +18, 0) | (+40, +40) | 0 | — (static) | pose fully static, no idle modulation |
| 4 | Micro-tremor | 0.20 | (0, 0, +14, 0 (±0.5°), +18 (±0.8°), 0) | (+40, +40) | 0 | `MIN_JERK` (idle mod) | very small tremor on roll and pitch |
| 5 | Yaw orient left | 0.25 | (0, 0, +12, 0, +15, -25) | (+38, +35) | -8 | `EASE_IN_OUT` | "where was it?" — questioning swing |
| 6 | Yaw orient right | 0.25 | (0, 0, +12, 0, +15, +25) | (+35, +38) | +8 | `EASE_IN_OUT` | mirrored search swing |
| 7 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth dissolve to the neutral pose |

Total duration ≈ 1.80 s.

### Audio (optional)
A short sharp sound (≤ 200 ms) like an "oh!" or "hm?" sample, triggered exactly at phase 2 (the snap). Volume moderate (50). Account for the ~50 ms audio buffer latency.

### Idle modulation during the micro-tremor (phase 4)
A very small, fast sinusoidal modulation on `pitch` (amplitude 0.8°, frequency 6 Hz) and `roll` (amplitude 0.5°, frequency 5 Hz, slightly offset). Antennas stay static.

### Body yaw and IK
`automatic_body_yaw=True` for the yaw swings (phases 5–6) — body follows the questioning search smoothly. During phase 2 (snap) body stays at 0° — the startle is a head reaction, not a whole-body reaction.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.80`, compact `evaluate(t)`.
- Phase 2 is the most important frame — velocity check: pitch from -3° to +18° in 0.15 s = ~140°/s ≈ 2.4 rad/s, well inside the 8 rad/s joint limit.
- Antenna motion from 0° to +40° in 0.15 s = ~267°/s ≈ 4.7 rad/s, also inside the limits.
- Phase 3 (frozen hold) is explicitly without idle mod — the shock is the silence between snap and tremor.
- `LINEAR` easing in phases 1 and 2 is mandatory for the startle character.
- Z translation of +15 mm sits comfortably within the IK-reachable envelope.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "surprised" / "startled" / "alert" (at least 4 out of 5)
- [ ] Total duration is 1.8 ± 0.2 s
- [ ] Phase 2 reads as a visible hard snap, not as a soft move (`LINEAR` is mandatory)
- [ ] Phase 3 shows a visible static hold (≥ 0.3 s)
- [ ] The micro-tremor in phase 4 is recognisable but not intrusive
- [ ] Yaw swings (phases 5–6) read as questioning, not controlled — `EASE_IN_OUT` makes the difference
- [ ] Audio (if enabled) hits the snap exactly; late audio defeats the startle effect
- [ ] Release drives cleanly to the neutral pose

## Anti-patterns
- `MIN_JERK` in phase 2 — the snap softens, the surprise character fades
- Phase 1 (pre-anticipation) longer than 0.1 s — ruins the shock, because it telegraphs
- Frozen hold (phase 3) shorter than 0.3 s — the observer sees no hold
- Body yaw away from 0° at the snap (phase 2) — disperses the reaction across too many axes
- Audio later than phase 2 — decouples sound from motion

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
- Is one yaw swing (phase 5 only) enough, or are both required? Test empirically — both is more dramatic.
- Should the micro-tremor (phase 4) be optional? Pro: makes the affect more human. Con: lengthens the behavior.
- Which audio file fits best? Proposal: a short sharp "oh!" or a synthetic alert chime.
- How does the behavior react when triggers come in very quick succession (two surprises back-to-back)? Cancel-and-restart or queue?
