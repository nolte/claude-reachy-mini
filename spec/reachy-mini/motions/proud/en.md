# Motion Sequence: Proud (`proud`)

Status: draft

## Context
A confident, presenting gesture: Reachy leans slightly back (X negative), straightens fully upright, antennas fully perked, holds the pose long, and makes a small yaw swing ("show it off"). Use cases: difficult task successfully finished, "Done!" after a long process, demo highlight.

## Characteristics
- Full upright: Z +12 mm, pitch +20° (comparable to `happy`'s peak, but held statically)
- X slightly negative (-3 mm) — leans back, "chest out"
- Antennas fully perked (+40°) — like a small crown
- Body yaw initially centred; a single yaw swing during the hold "presents" the pose
- Medium tempo (~2.6 s); `MIN_JERK` with a small `CARTOON` bounce for the rise

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (flinch) | 0.15 | (+2, 0, -2, 0, -5, 0) | (+5, +5) | 0 | `MIN_JERK` | small flinch before lifting |
| 2 | Lift (main move) | 0.40 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `CARTOON` | chest out with a small overshoot |
| 3 | Mini bounce | 0.20 | (-3, 0, +10, 0, +18, 0) | (+38, +38) | 0 | `CARTOON` | settles slightly from the overshoot |
| 4 | Hold (proud) | 0.80 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `MIN_JERK` (static) | hold the pose long |
| 5 | Yaw swing right | 0.30 | (-3, 0, +12, 0, +20, +12) | (+40, +40) | +5 | `EASE_IN_OUT` | show it to the world |
| 6 | Centre | 0.20 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `MIN_JERK` | back to centre |
| 7 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 2.55 s.

### Audio (optional)
A short triumphant tone (≤ 600 ms), started with phase 2 — trumpet-like or a rising chord. Volume moderate to high (60).

### Idle modulation
Phase 4 is static without modulation — pride is amplified by stillness. A quiet idle breath (amplitude 0.5°) is acceptable as a variant but is not the default.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the yaw swing in phase 5 reads softer through IK. Body-yaw moving along (+5° instead of 0°) signals "I am presenting myself".

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.55`, seven phases.
- X translation -3 mm is small but important for the "leaning-back" character.
- Pitch +20° + Z +12 mm + antennas +40° is the maximally expressive pose within the limits — all values respect pitch ≤ +90° and antennas ≤ ±π.
- `CARTOON` easing in phases 2 and 3 makes the lift triumphant; without the overshoot, the pride feels muted.
- Phase 4 (hold) must be long enough (≥ 0.7 s) for the pride to read.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "proud" / "triumphant" / "presenting" (at least 4 out of 5)
- [ ] Total duration is 2.5 ± 0.3 s
- [ ] The lift in phase 2 is visibly `CARTOON`-overshooting
- [ ] The hold (phase 4) is at least 0.7 s long
- [ ] The yaw swing in phase 5 is visible but not exaggerated (≤ +15°)
- [ ] The antenna value during the hold is at +40° (full perk)
- [ ] Audio (if enabled) has a triumphant character

## Anti-patterns
- `MIN_JERK` instead of `CARTOON` in phase 2 — pride feels muted
- Hold (phase 4) shorter than 0.5 s — the pose does not read
- Multiple yaw swings — feels excited, not proud
- Pitch < +15° — the full lift is missing
- Antennas below +30° — the "crown" effect is lost
- Audio with a falling tone — wrong affect

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
- Should the yaw swing in phase 5 always go right, or random left/right?
- Which audio file fits? Proposal: a short trumpet stab or a three-note rising chord.
- For repeated successes, should a "multi-proud" variant with two yaw swings be offered?
- How does `proud` integrate with follow-up behaviors? Proposal: chain into `waiting-idle` for a calm tail.
