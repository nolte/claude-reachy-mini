# Motion Sequence: Farewell Wave (`farewell-wave`)

Status: draft

## Context
An open, slightly wistful farewell gesture: like `greeting-wave`, but ends not upright but in a falling pose. Use cases: person leaves the room (camera detects door), "goodbye" voice command, HA trigger "person out of range", transition into `goto_sleep()`.

## Characteristics
- Antenna wave like `greeting-wave` — same left-right offset animation
- Body yaw and head yaw lean toward the person
- **Difference to `greeting-wave`**: after the third wave the pitch lowers instead of going into an up-nod — the pose ends looking-away instead of smiling-at
- Medium tempo (~2.2 s), slightly longer than `greeting-wave` due to the hold

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0.15 | (0, 0, +3, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | small lift |
| 2 | Lean toward person | 0.25 | (0, 0, +3, 0, +5, +10) | (+10, +10) | +8 | `MIN_JERK` | turn-toward |
| 3 | Wave 1 | 0.20 | (0, 0, +3, 0, +5, +10) | (+40, +5) | +8 | `EASE_IN_OUT` | left antenna up |
| 4 | Wave 2 | 0.20 | (0, 0, +3, 0, +5, +10) | (+5, +40) | +8 | `EASE_IN_OUT` | right antenna up |
| 5 | Wave 3 | 0.25 | (0, 0, +3, 0, +5, +10) | (+30, +30) | +8 | `EASE_IN_OUT` | both up — wave peak |
| 6 | Pitch lowering | 0.30 | (0, 0, -2, 0, -10, +8) | (+15, +15) | +5 | `MIN_JERK` | head sinks — farewell character |
| 7 | Hold (saying goodbye) | 0.40 | (0, 0, -3, 0, -12, +5) | (+10, +10) | +3 | `MIN_JERK` | brief hold in the lowered pose |
| 8 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 2.25 s.

### Audio (optional)
A quieter, falling tone ("byyye" — ≤ 700 ms), started with phase 3. Volume quiet to moderate (40). In contrast to `greeting-wave`: falling, not rising tone.

### Idle modulation
None.

### Body yaw and IK
`automatic_body_yaw=True` recommended; same as `greeting-wave`. When the person's position is known: dynamically via `look_at_world`.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.25`. Code-wise, `farewell-wave` can be implemented as a variant of `greeting-wave` — same wave phases, different end phases.
- Pitch -12° in phase 7 is not as deep as in `sad`, just enough for the farewell mood.
- Antenna values follow the same velocity limits as `greeting-wave`.
- Phase 8 (release) ideally hands over to `waiting-idle` or `goto_sleep()`.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "goodbye" / "farewell" / "saying goodbye" (at least 4 out of 5)
- [ ] Total duration is 2.2 ± 0.2 s
- [ ] Three antenna-wave phases are visible
- [ ] Pitch visibly drops below 0° in phase 6 — no up-nod
- [ ] The hold phase 7 shows a visible lowered pose without tipping into `sad`
- [ ] Audio (if enabled) has a falling tone

## Anti-patterns
- Pitch ≥ 0° in the end pose — reads as greeting, not farewell
- Pitch < -20° in phase 7 — reads as `sad`, not a calm goodbye
- Antennas synchronous instead of offset in phases 3/4
- Rising audio tone — wrong affect
- Hold phase 7 longer than 0.6 s — reads as wistful instead of farewell

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Should the pitch in phase 7 go even deeper (e.g. -18°) for a stronger farewell character? Test empirically.
- How is the handover into `waiting-idle` or `goto_sleep()` wired? Skill-layer decision.
- Joint implementation with `greeting-wave` as a parametrised `Move` subclass `WaveMove(direction="hello"|"goodbye")`?
