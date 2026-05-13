# Motion Sequence: Greeting Wave (`greeting-wave`)

Status: draft

## Context
An open, friendly greeting expressed as an antenna wave: Reachy leans slightly toward the person, runs a visible wave through both antennas, and ends with a small nod. Use cases: person-detection trigger, voice "hello Reachy", demo opening, door-opening trigger via HA.

## Characteristics
- Antenna wave as the main statement: left and right offset (not synchronous) — reads like a waving hand
- Body yaw and head yaw lean slightly toward the person — "I see you"
- Final shallow up-nod — a polite micro-bow
- Medium tempo (~2.0 s); `EASE_IN_OUT` for the wave, `MIN_JERK` for the lean

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (perk) | 0.15 | (0, 0, +3, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | small lift |
| 2 | Lean toward person | 0.25 | (0, 0, +3, 0, +5, +10) | (+10, +10) | +8 | `MIN_JERK` | small turn-toward |
| 3 | Wave 1 (left lead) | 0.20 | (0, 0, +3, 0, +5, +10) | (+40, +5) | +8 | `EASE_IN_OUT` | left antenna up, right down |
| 4 | Wave 2 (right lead) | 0.20 | (0, 0, +3, 0, +5, +10) | (+5, +40) | +8 | `EASE_IN_OUT` | right antenna up, left down |
| 5 | Wave 3 (both up) | 0.20 | (0, 0, +3, 0, +5, +10) | (+30, +30) | +8 | `EASE_IN_OUT` | synchronous peak |
| 6 | Polite up-nod | 0.25 | (0, 0, +1, 0, -8, +8) | (+15, +15) | +5 | `MIN_JERK` | small bow toward the person |
| 7 | Pitch back | 0.20 | (0, 0, +2, 0, +3, +5) | (+12, +12) | +3 | `MIN_JERK` | back to neutral upright |
| 8 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.95 s.

### Audio (optional)
A friendly short "hello!" sample (≤ 600 ms), started at phase 3 (first wave). Volume moderate (50). Rising tone.

### Idle modulation
None — the wave is the whole statement, modulation would blur it.

### Body yaw and IK
`automatic_body_yaw=True` recommended; the joint lean of body and head reads organically through IK. When the person's position is known (e.g. via `look_at_world`), the yaw value can be set dynamically instead of the static +10°.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.95`. The antenna wave is the diagnostic detail; the offset values (40°/5°) must be visibly large.
- When `look_at_world(x, y, z)` is available for the turn-toward (person position from camera): combine instead of fixed yaw values.
- Yaw +10° and body yaw +8° sit well within the ±160°/±65° limits.
- Antenna jump from 5° to 40° in 0.2 s ≈ 175°/s ≈ 3 rad/s, inside the 8 rad/s limit.
- When the greeting runs toward a known person and a follow-up behavior is queued (e.g. `alert-listening`), replace phase 8 (release) with a direct handover.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "hello" / "greeting" / "friendly" (at least 4 out of 5)
- [ ] Total duration is 2.0 ± 0.2 s
- [ ] Three visible antenna-wave phases (phases 3–5)
- [ ] Phases 3 and 4 show clear antenna asymmetry (at least 30° difference between left and right)
- [ ] Lean toward the person is visible (yaw and body yaw to the same side)
- [ ] Phase 6 (up-nod) reads as a "small bow", not as a questioning tilt
- [ ] Audio (if enabled) sounds friendly, not exuberant

## Anti-patterns
- Antennas synchronous instead of offset — loses the "wave" character
- Body yaw and head yaw opposed — confusing pose
- Lean too large (> ±20°) — feels intrusive
- `LINEAR` easing in the wave — feels mechanical like a windshield wiper
- Audio with bell-like sound — wrong fit for the social gesture

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
- Should the wave be three-phase or four-phase (one extra wave before phase 5)?
- Which audio file fits as a default? Proposal: the SDK's `wake_up` sound as a reference, custom file for a tailored "hello".
- Should the turn-toward be dynamically tied to the actual person position via `look_at_world`? Pro: more realistic; con: fragile on detection failure.
- How does `greeting-wave` react to a repeat trigger within < 5 s? Proposal: do not retrigger, instead segue into `alert-listening`.
