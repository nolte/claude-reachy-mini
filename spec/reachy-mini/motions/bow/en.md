# Motion Sequence: Bow (`bow`)

Status: draft

## Context
A formal single bow: Reachy lowers the head in a dignified, slow motion, holds the pose briefly, and rights itself again. Use cases: demo opening or closing, formal greeting of a guest, "thank you" as a reply, "please" as a courtesy gesture.

## Characteristics
- A single, slow pitch sweep down (-25°) — no multi-nod like `agreeing-nod`
- Antennas slightly tucked (-10°) — restrained, not perked
- Body yaw and head yaw centred — formal, straight pose
- Z translation -5 mm during the bow — the whole head lowers, not just the pitch
- Medium tempo (~2.4 s); `MIN_JERK` only, for the dignity

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (lift) | 0.20 | (0, 0, +3, 0, +5, 0) | (-5, -5) | 0 | `MIN_JERK` | small lift as preparation |
| 2 | Deep bow | 0.80 | (0, 0, -5, 0, -25, 0) | (-10, -10) | 0 | `MIN_JERK` | slow, even descent |
| 3 | Hold | 0.40 | (0, 0, -5, 0, -25, 0) | (-10, -10) | 0 | `MIN_JERK` (static) | hold the pose deliberately |
| 4 | Rise | 0.60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | slow return upright |
| 5 | Release (subtle) | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | handover into a follow-up behavior |

Total duration ≈ 2.40 s.

### Audio (optional)
Optionally a quiet, restrained tone with phase 2 (e.g. a whispered "thank you") — volume quiet (25). In most use cases `bow` is silent.

### Idle modulation
None — the silence during the hold (phase 3) carries the dignity of the gesture.

### Body yaw and IK
`automatic_body_yaw=True` recommended; since body and head stay fully centred, the mode has no visible effect.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.40`, five phases — very lean.
- Pitch -25° / 0.80 s = 31°/s ≈ 0.54 rad/s, well below the 8 rad/s joint limit.
- Z translation -5 mm sits comfortably within the IK-reachable envelope.
- Pitch -25° sits below the ±90° limit and close to the -28° pitch in `sad`. For a deeper "Japanese" bow, go to -30° but combine translation and pitch instead of maxing pitch alone.
- Antenna value -10° is subtle and reads as neutral — deliberately not affective.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "bow" / "formal" / "polite" (at least 4 out of 5)
- [ ] Total duration is 2.4 ± 0.2 s
- [ ] Phase 2 is a continuous, even descent — no interim stop
- [ ] Phase 3 (hold) is static without modulation
- [ ] Roll and yaw stay at 0° across the entire sequence
- [ ] Rise in phase 4 is even, no jerks
- [ ] Audio (if enabled) is quiet and unhurried

## Anti-patterns
- Multiple pitch switches — reads as `agreeing-nod`, not `bow`
- Roll or yaw components — defeat the formal pose
- `CARTOON` or `LINEAR` easing — wrong character
- Pitch deeper than -30° — risks the Stewart joint limit and looks overstrained
- Hold (phase 3) shorter than 0.3 s — formal character is lost
- Antennas perked (positive values) — reads as joyful instead of formal

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Should there be variants (`bow-deep` with -30° for special honours, `bow-light` with -15° for everyday courtesy)?
- Which audio file fits? Proposal: quiet "thank you" or "please" as optional audio.
- Should Z translation be lowered more in a deep variant rather than maxing the pitch?
