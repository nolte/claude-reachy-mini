# Motion Sequence: Peek (`peek`)

Status: draft

## Context
A small playful gesture: Reachy peeks sideways — body and head turn slightly to the same side, Z lifts, the head tilts briefly "searching" and returns. Use cases: playful reaction to a person, hide-and-seek in a demo, "look at this!" trigger, draw attention out of idle without a big gesture.

## Characteristics
- Body and head yaw to the same side (aligned, no twist) — shows a clear viewing direction
- Z slightly lifted (+5 mm) — like "going on tiptoe"
- Small pitch +5° and roll 0° — curious, but not as strong as `curious`
- Mini yaw search motion during the hold — small left-right within the chosen direction
- Medium tempo (~2.0 s)

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (small) | 0.15 | (0, 0, +2, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | minimal lift |
| 2 | Peek-out (sideways) | 0.40 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `EASE_IN_OUT` | aligned peek to the side |
| 3 | Mini pause | 0.30 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `MIN_JERK` (static) | brief stop to look |
| 4 | Search motion | 0.30 | (0, 0, +5, 0, +5, +20 (±5°)) | (+15, +15) | +20 | `MIN_JERK` (idle mod) | small yaw sway to search |
| 5 | Mini hold | 0.20 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `MIN_JERK` (static) | brief stop — found? |
| 6 | Pull back | 0.40 | (0, 0, +2, 0, +3, +5) | (+8, +8) | +5 | `MIN_JERK` | yaw and Z fade out |
| 7 | Release | 0.30 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 2.05 s.

### Audio (optional)
A quiet questioning beep or "hm?" tone (≤ 300 ms), started with phase 2. Volume quiet (30).

### Idle modulation during phase 4
Sinusoidal modulation on `yaw` (amplitude 5°, frequency 1.0 Hz) — fast, short search. Pitch and roll keep their values; antennas stay static.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the aligned rotation of body and head reads softer through IK. The `+25° head yaw` combined with `+20° body yaw` yields a relative yaw of +5° — well within `max_relative_yaw=65°`.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.05`. `evaluate(t)` with sinusoidal modulation in phase 4.
- Body yaw + head yaw both on the same side — different from `curious` (where body goes in the tilt direction and head goes in the yaw direction offset) and different from `disagreeing-shake` (where body stays fixed).
- Yaw +25° for head and +20° for body sit well within the ±65° / ±160° limits.
- Z translation +5 mm sits in the IK envelope.
- Variant: a mirrored version (negative yaw values) for peeking the other way — useful as a random choice on repeat triggers.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "peek" / "looking out" / "peering" (at least 4 out of 5)
- [ ] Total duration is 2.0 ± 0.2 s
- [ ] Body yaw and head yaw point visibly the same way
- [ ] Z lift in phase 2 is visible
- [ ] Phase 4 clearly shows the small search motion
- [ ] Phases 3 and 5 are static without modulation
- [ ] Audio (if enabled) sounds questioning, not exclamatory

## Anti-patterns
- Body yaw and head yaw opposed — feels twisted, not peek
- Pitch < 0° — reads as sad, not curious
- Roll component — becomes `curious`
- Phase 4 search frequency above 2 Hz — feels nervous
- `CARTOON` easing in phase 2 — feels springy, not stealthy

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
- Should the peek always go to the same side or random left/right?
- Which audio file fits? Proposal: a quiet questioning beep.
- How does `peek` interact with `look_at_world` for a detected person? Merging would be context-relevant.
- If the search yields no hit (target not visible), should a follow-up `confused` be triggered?
