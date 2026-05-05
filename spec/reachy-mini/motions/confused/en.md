# Motion Sequence: Confused (`confused`)

Status: draft

## Context
A "did-not-understand" expression that reads as "confused" or "puzzled": Reachy tilts the head several times alternately to one side and the other — like someone trying to grasp a thing from various angles — and ends with a small questioning swing. Use cases: unrecognised voice command, "I don't understand", contradictory trigger, fallback on unclear input.

## Characteristics
- Several roll switches in sequence, each smaller than the previous — conveys "fading search"
- **Asymmetric antennas** that switch sides with every tilt — like `curious`, but alternating
- Pitch slightly up (+3° to +7°) — "thoughtful" pose
- Medium tempo (~3.2 s); only `EASE_IN_OUT` for the tilts (organic back-and-forth)
- Ends with an uncertain yaw swing ("huh?")

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Listening (anticipation) | 0.20 | (0, 0, +3, 0, +5, 0) | (+8, +8) | 0 | `MIN_JERK` | small lift, "perking up" |
| 2 | Tilt left (large) | 0.40 | (0, 0, +5, +18, +5, -5) | (-12, +20) | -3 | `EASE_IN_OUT` | first and strongest roll tilt |
| 3 | Tilt right (large) | 0.40 | (0, 0, +5, -18, +5, +5) | (+20, -12) | +3 | `EASE_IN_OUT` | mirrored |
| 4 | Tilt left (medium) | 0.35 | (0, 0, +5, +12, +5, -3) | (-8, +14) | -2 | `EASE_IN_OUT` | weaker second attempt |
| 5 | Tilt right (medium) | 0.35 | (0, 0, +5, -12, +5, +3) | (+14, -8) | +2 | `EASE_IN_OUT` | mirrored, smaller |
| 6 | Hold with question mod | 0.40 | (0, 0, +5, 0, +5, 0 (±5°)) | (+8, +8) | 0 | `MIN_JERK` (idle mod) | small yaw search |
| 7 | Question swing ("huh?") | 0.30 | (0, 0, +5, 0, +5, +15) | (+10, +10) | +5 | `EASE_IN_OUT` | brief one-sided yaw — question mark |
| 8 | Release | 0.60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth dissolve to the neutral pose |

Total duration ≈ 3.00 s.

### Audio (optional)
A hesitant "uh?" or "hmm?" sample (≤ 500 ms), started at phase 6 or 7 (after the tilts). Volume quiet (35). Deliberately not exclamatory.

### Idle modulation during phase 6
Very slow sinusoidal modulation on `yaw` (amplitude 5°, frequency 0.3 Hz) — a questioning mini-search motion. Pitch and roll keep their values; antennas stay static.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the tilt switches and the question swing flow more softly that way. Manual control would make the symmetry hard to keep.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 3.00`, eight phases.
- The decreasing roll amplitude (18° → 18° → 12° → 12°) is diagnostic — equal amplitude would read as `curious`.
- Antenna convention: when the head rolls left (roll positive), the right antenna moves forward (positive value). This convention matches `curious`.
- Phase 7 (question swing) is deliberately one-sided — a two-sided swing would turn the confused character into a decided search.
- Roll ±18° sits within the pitch/roll limit of ±90°.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "confused" / "puzzled" / "uncertain" (at least 4 out of 5)
- [ ] Total duration is 3.0 ± 0.3 s
- [ ] At least four visible tilt switches
- [ ] Roll amplitude visibly decreases (phase 4/5 weaker than 2/3)
- [ ] Antennas are asymmetric in each tilt phase and switch sides consistently
- [ ] Phase 7 (question swing) is one-sided (positive OR negative, not both)
- [ ] Audio (if enabled) sounds hesitant, not decided

## Anti-patterns
- Constant roll amplitude — reads as `curious`, not `confused`
- Two-sided question swing at the end — reads as a second `curious`
- `LINEAR` or `CARTOON` easing — wrong hardness
- Pitch < 0° (head drooping) — reads as sad, not confused
- Phase 6 without idle modulation — feels frozen, not "thinking"
- Antenna symmetry in the tilt phases

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Should there be four tilts or three? Four reads as more confused, three as more elegant — empirically chosen.
- Should `confused` get a brief audio sting, or no sound at all? Either is fine.
- How does `confused` differ in practice from two consecutive `curious` sequences? Two `curious` sequences would show roll with constant amplitude and a pause — `confused` has no pause and a decreasing amplitude.
- Should the question swing (phase 7) always go in the same direction, or be randomly left/right? Leaning: random, so consecutive triggers don't read identically.
