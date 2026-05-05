# Motion Sequence: Disgust (`disgust`)

Status: draft

## Context
The sixth Ekman basal emotion: an aversive turn-away with a small shudder. Reachy pulls back slightly, turns the head to the side, tilts it in the typical disgust roll, and runs a brief shudder over the pose. Use cases: "ew!", a feedback to an unpleasant sensor input (e.g. strong odour via HA air-quality sensor), playful reaction to a wrong input.

## Characteristics
- X translation slightly negative (-5 mm) — pulling back
- Yaw to the side (+25°) — turning away
- Roll +6° — the typical disgust pose component (small tilt)
- Pitch slightly back (-3°) — aversive, not sad
- Antennas flat (-15°) — folded back, defensive
- Small shudder in phase 4 — vibration on roll
- Fast tempo (~1.5 s)

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (mini flinch) | 0.10 | (+1, 0, 0, 0, +1, 0) | (-3, -3) | 0 | `LINEAR` | very small, almost invisible |
| 2 | Disgust reaction | 0.30 | (-5, 0, -2, +6, -3, +25) | (-15, -15) | +5 | `EASE_IN_OUT` | main motion — turn-away with roll |
| 3 | Hold | 0.40 | (-5, 0, -2, +6, -3, +25) | (-15, -15) | +5 | `MIN_JERK` (static) | hold the pose |
| 4 | Mini shudder | 0.20 | (-5, 0, -2, +6 (±2°, 8 Hz), -3, +25) | (-15, -15) | +5 | `MIN_JERK` (idle mod) | small roll vibration — "ugh" |
| 5 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.50 s.

### Audio (optional)
A short clipped "ew" or "yuck!" sample (≤ 250 ms), started with phase 2. Volume moderate (45).

### Idle modulation during phase 4
Sinusoidal modulation on `roll`: amplitude 2°, frequency 8 Hz — short shudder, similar to the `angry` vibration but shorter and on roll instead of pitch. Four shudder cycles in 0.2 s.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the small body yaw (+5°) tracks the head yaw (+25°) via IK softly. Relative yaw +20°, well inside the ±65° limit.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.50`, five phases.
- Roll shudder frequency 8 Hz × ±2° = 32°/s ≈ 0.56 rad/s — well within the 8 rad/s limit.
- Pitch -3° (only slightly negative) is diagnostic — at deeper pitch it reads as `disappointed`.
- Roll +6° is the disgust-typical component and must not be omitted — it differentiates `disgust` from `disagreeing-shake`.
- Variant: a mirrored version (roll -6°, yaw -25°) — disgust to the other side.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "disgust" / "ew" / "rejected" (at least 4 out of 5)
- [ ] Total duration is 1.5 ± 0.2 s
- [ ] Roll component +6° is visible
- [ ] The shudder in phase 4 reads as a brief vibration
- [ ] Antennas flat (-15°) during phases 2–4
- [ ] Audio (if enabled) has an aversive character

## Anti-patterns
- Pitch deeper than -10° — becomes `disappointed`
- Shudder frequency < 5 Hz — feels too slow
- Antennas perked — wrong association
- Roll = 0° — loses the disgust-typical tilt
- Total duration > 2 s — defeats the spontaneous reaction
- `LINEAR` easing in phase 2 — feels aggressive, not aversive

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Two shudders in phase 4 or just one? Multiple shudders read as stronger, but risk tipping into "jittery".
- Which audio file fits? Proposal: a clipped "ew!" or "yuck".
- How does `disgust` differ in practice from `shy`? `shy` is social-reserved, `disgust` is sensory-aversive; the roll shudder is diagnostic for `disgust`.
- Should the turn-away direction be random left/right, or context-dependent?
