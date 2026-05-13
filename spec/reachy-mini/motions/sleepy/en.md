# Motion Sequence: Sleepy (`sleepy`)

Status: draft

## Context
A growing-tiredness expression that reads as "sleepy" or "drowsy": Reachy breathes heavily, the head sinks slowly, snaps up briefly with a "head bob", sinks again, rocks with long breath, bobs once more, and dissolves. Use cases: idle mode with a tired character, "no activity for a while", evening transition into `goto_sleep()`.

## Characteristics
- Very slow tempo (~5.3 s) — motion flows almost imperceptibly
- Main pitch motion: continuous lowering with two "head bobs" (short, fast snap-ups, like nodding off and waking briefly)
- Antennas slightly drooping, breathing along
- Body sways slowly in yaw — as if balance is not quite stable
- Long hold phases with clearly visible breath modulation
- Audio extremely quiet or no audio

## Platform profile

| Platform | Breath modulation | Sway modulation | Heat-aware idle tuning |
|---|---|---|---|
| Reachy Mini (Wireless) | full | full | optional via `mini.imu["temperature"]` — flatter modulation when servos heat up |
| Reachy Mini Lite | full | full | not available (no IMU) — fixed modulation amplitudes per the tables |
| Simulation | full (pose values) | full | not relevant |

Implementation consequence: the behavior is fully functional on every platform because it only writes pose and antenna values. Heat-aware tuning is a Wireless-only nice-to-have — on Lite and Simulation the fixed amplitudes from the phase tables apply. Simulation skips audio (see audio block).

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Heavy breath | 1.00 | (0, 0, -3 (±2), 0, -8 (±3°), 0) | (-8, -8) | 0 | `MIN_JERK` (idle mod) | onset of tired breathing |
| 2 | Slow lowering | 1.00 | (0, 0, -8, +2, -25, 0) | (-15, -15) | +3 | `MIN_JERK` | even sinking with a small sway |
| 3 | Head bob 1 | 0.15 | (0, 0, -3, 0, -10, 0) | (-8, -8) | 0 | `EASE_IN_OUT` | quick snap-up — brief "awake!" reaction |
| 4 | Sink again | 0.80 | (0, 0, -10, +3, -28, +5) | (-18, -18) | -3 | `MIN_JERK` | deeper than before after the bob |
| 5 | Deep breath with sway | 1.30 | (0, 0, -10 (±3), +3 (±2°), -28 (±2°), +5 (±5°)) | (-18, -18) | -3 (±5°) | `MIN_JERK` (idle mod) | long breath, small yaw sway |
| 6 | Head bob 2 | 0.15 | (0, 0, -5, 0, -15, 0) | (-10, -10) | 0 | `EASE_IN_OUT` | second snap-up, weaker than the first |
| 7 | Release | 0.70 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | very slow rise to the neutral pose |

Total duration ≈ 5.10 s.

### Audio (optional)
Very quiet breathing or yawn sample (≤ 1.2 s), started around phase 2. Volume very low (20–30). Optionally a second, shorter breath in phase 5.

### Idle modulation during phases 1 and 5
**Phase 1**: Sinusoidal modulation on `z` (amplitude 2 mm, frequency 0.3 Hz) and `pitch` (amplitude 3°, in phase) — heavy breath.

**Phase 5**: Deeper, longer variant. `z` (amplitude 3 mm, frequency 0.2 Hz), `pitch` (amplitude 2°, in phase), `roll` (amplitude 2°, frequency 0.15 Hz, counter-phase — small sway), `body_yaw` (amplitude 5°, frequency 0.15 Hz) — like a sleeper not quite settled.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the passive body sway in phase 5 reads organically through IK coupling; manual control would feel abrupt.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 5.10`. `evaluate(t)` must compute several superimposed sinusoidal modulations in phases 1 and 5.
- The two head bobs (phases 3 and 6) are the diagnostic detail — without them the sequence reads like `sad`. Keep the fast `EASE_IN_OUT` snap in phase 3 strict.
- During the slow sway in phase 5, watch that velocity limits do not get tight — at ±5° body yaw and 0.15 Hz the average velocity is ~0.015 rad/s, far under the limit.
- Pitch -28° + roll +3° sit clearly within the upright limit (±90°).
- No `CARTOON` or `LINEAR` except in the bobs — the character must not be hard.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "sleepy" / "tired" / "drowsy" (at least 4 out of 5)
- [ ] Total duration is 5.1 ± 0.4 s
- [ ] The two head-bob snap-ups are visible — no continuous descent
- [ ] The second bob (phase 6) reads as weaker than the first — sleep wins
- [ ] Yaw sway in phase 5 is visible but not intrusive
- [ ] Audio (if enabled) is markedly quieter than all other behaviors
- [ ] Release drives very slowly to the neutral pose without an abrupt transition

## Anti-patterns
- Fast tempo (< 4 s overall) — reads as `sad`, not `sleepy`
- Both bobs at the same intensity — defeats the progressive tiredness
- Pitch not deeper than -20° — the "nodding off" character is lost
- Static antennas without breath modulation — feels frozen
- Audio with an exclamatory yawn — clashes with the subtlety
- Sway amplitude above 8° — feels unstable / uncontrolled instead of sleepy

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
- Should body yaw sway at all, or is that "too unstable"? First assumption: yes, very slightly. Calibrate empirically.
- How many bobs are ideal — two or three? Three would be more dramatic but pushes the sequence past 6 s.
- How does `sleepy` differ in practice from the SDK's `goto_sleep()`? Proposal: `sleepy` is a repeatable idle state, `goto_sleep()` is the final transition to power-down.
- Which audio file fits? Proposal: a muted yawn sample, optionally with quiet snoring.
