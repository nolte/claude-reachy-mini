# Motion Sequence: Agreeing / Nod (`agreeing-nod`)

Status: draft

## Context
A short, clear agreement gesture that reads as "yes" or "understood": Reachy nods downward two to three times, stays centred, and returns to the neutral pose. Use cases: confirmation of voice commands, "OK / understood" as a reply to a question, brief positive feedback without a large gesture.

## Characteristics
- Three clear nods on the pitch axis, each slightly weaker than the previous (typical human nodding)
- Antennas slightly raised (+8°) but stationary — they are not part of the nod
- Body yaw stays strictly centred — the entire affect is pitch-only
- Short tempo (~1.8 s), `MIN_JERK` for soft transitions, no `CARTOON`

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (micro-lift) | 0.15 | (0, 0, +2, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | small lift as preparation |
| 2 | Nod 1 down | 0.22 | (0, 0, 0, 0, -12, 0) | (+8, +8) | 0 | `MIN_JERK` | first and strongest nod |
| 3 | Up again | 0.18 | (0, 0, +2, 0, +5, 0) | (+8, +8) | 0 | `MIN_JERK` | not all the way back to 0° |
| 4 | Nod 2 down | 0.18 | (0, 0, 0, 0, -10, 0) | (+8, +8) | 0 | `MIN_JERK` | second nod, slightly weaker |
| 5 | Up again | 0.15 | (0, 0, +1, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | a little less lift |
| 6 | Nod 3 down (subtle) | 0.15 | (0, 0, 0, 0, -7, 0) | (+8, +8) | 0 | `MIN_JERK` | third, clearly smaller nod |
| 7 | Hold | 0.20 | (0, 0, 0, 0, -3, 0) | (+8, +8) | 0 | `MIN_JERK` | held slightly down — "confirmed" |
| 8 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.63 s.

### Audio (optional)
A brief neutral "mhm" sample (≤ 250 ms), started at phase 2 (the first nod). Volume quiet (35). Not exuberant.

### Idle modulation
No idle modulation in this behavior — the motion is compact enough that any hold modulation would break the read.

### Body yaw and IK
`automatic_body_yaw=True` is acceptable; with body yaw at 0° throughout the sequence it has no visible effect. With `False` nothing changes either.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.63`, eight phases — very compact.
- The decreasing pitch amplitude (-12° → -10° → -7°) is diagnostic; equal amplitude reads as mechanical.
- Pitch motion from -12° to +5° in 0.18 s = ~94°/s ≈ 1.6 rad/s, well below the joint velocity limit.
- Strictly only `MIN_JERK` easing — anything else reads as either too hard (`LINEAR`) or too silly (`CARTOON`).
- Roll and yaw always 0° — a single roll offset would shift the gesture toward `curious` or `confused`.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "yes" / "agreeing" / "understood" (at least 4 out of 5)
- [ ] Total duration is 1.6 ± 0.2 s
- [ ] Three visible nods, each weaker than the previous
- [ ] Roll and yaw stay at 0° across the entire sequence
- [ ] Antennas do not move (apart from the single lift in phase 1)
- [ ] Audio (if enabled) is neutral, not exuberant
- [ ] Transition to the neutral pose is seamless, not jolty

## Anti-patterns
- More than three nods — reads as exaggerated agreement or a dance step
- Antennas nodding along — defeats the clarity of the gesture
- Roll or yaw components — blur the nod character
- `CARTOON` easing — turns the nod into a spring motion
- Pitch not deep enough (< -8° on the first nod) — the gesture becomes invisible
- Hold (phase 7) longer than 0.4 s — feels stiff

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
- Should there always be three nods, or two depending on the context? Two would be minimal.
- Which audio file fits? Proposal: a short neutral "mhm" — deliberately not "yes!" or "exactly!".
- Should the hold (phase 7) be optional? Pro: smoother handover into a follow-up behavior; con: lower legibility.
