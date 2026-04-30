# Motion Sequence: Pseudo Look-Around Spin (`spin-look-around`)

Status: draft

## Context
A playful show motion that simulates a look-around: body yaw swings to the limit (±150°), head tracks along, then swings the other way and back to centre. Use cases: demo highlight, "look at me" reveal, play mode with spatial attention.

> ⚠ Note: body-yaw limit is `max_body_yaw=±160°` (control-surface spec). A full 360° spin is **mechanically impossible** on Reachy Mini — this spec therefore targets ±150° as a "pseudo-spin" that suggests a look-around.

## Characteristics
- Large body-yaw swing: 0° → -150° → +150° → 0° (over 5 s total)
- Head yaw is intentionally counter-phase to the body so the camera "tracks" a fixed point (relative yaw stays small)
- Pitch slightly up (+5°) — looking, not down
- Antennas perked (+25°), stationary
- Long tempo (~7 s) due to large distances; `EASE_IN_OUT` only

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0.20 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `MIN_JERK` | lift and prepare |
| 2 | Spin left | 1.80 | (0, 0, +3, 0, +5, +50) | (+25, +25) | -150 | `EASE_IN_OUT` | body swings left, head stays "facing forward" relatively |
| 3 | Mini hold left | 0.30 | (0, 0, +3, 0, +5, +50) | (+25, +25) | -150 | `MIN_JERK` (static) | pose hold at the limit |
| 4 | Spin right (through centre) | 2.50 | (0, 0, +3, 0, +5, -50) | (+25, +25) | +150 | `EASE_IN_OUT` | wide swing past centre |
| 5 | Mini hold right | 0.30 | (0, 0, +3, 0, +5, -50) | (+25, +25) | +150 | `MIN_JERK` (static) | pose hold at the limit |
| 6 | Spin to centre | 1.50 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `EASE_IN_OUT` | centring outro |
| 7 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 7.00 s.

### Audio (optional)
A light whirr or "whoosh" during the swing phases 2 and 4. Volume moderate (40). Mind audio latency.

### Idle modulation
None.

### Body yaw and IK
`automatic_body_yaw=False` recommended — the counter-phase head yaw must be set manually, otherwise IK overrides the tracking effect. Body yaw -150° + head yaw +50° = relative yaw +200° (which is **outside** the `max_relative_yaw=±65°` limit!) — so head yaw +50° is **not reachable**.

> ⚠ TBD: validate against real hardware — the maximum head-yaw differential against body yaw at extreme body swings is only ±65°. Phases 2 and 4 must therefore reduce head yaw accordingly: at body yaw -150°, head yaw can range from about -85° down (relative ≤ 65°), practically head yaw between roughly -85° and 0°. The values in the table above are a **draft proposal**; the exact geometry must be computed against the real IK before implementation.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 7.00`. Phase transitions are critical — velocity per phase:
  - Phase 4: body yaw from -150° to +150° in 2.5 s = 120°/s ≈ 2.1 rad/s, within the 8 rad/s limit.
- Head-yaw values are uncertain (see TBD above). Safer default: head yaw stays fixed at 0° relative to the body — i.e. the head always faces "toward body direction", no counter-phase tracking. This sacrifices the tracking effect but is safe.
- The non-tracking variant — head yaw == 0° in every phase — is the recommended default implementation.

## Acceptance Criteria
- [ ] External observers read the motion as "looking around" / "spin" / "impressive turn" (at least 4 out of 5)
- [ ] Body yaw reaches ±150° in phases 2 and 4
- [ ] Phase 4 passes through the centre (continuous, no stop)
- [ ] `automatic_body_yaw` is `False` throughout the entire sequence
- [ ] Mini holds (phases 3, 5) read as visible brief pauses
- [ ] Audio (if enabled) covers the swing phases 2 and 4
- [ ] Return to the neutral pose is clean

## Anti-patterns
- Body yaw beyond ±150° — breaks the safety headroom (joint limit is ±160°)
- `LINEAR` easing — feels jerky
- Counter-phase head yaw without honouring `max_relative_yaw` — IK violation
- Phase 4 with a stop in the middle — breaks the "look-around" feel
- Antenna modulation during the swings — feels restless
- Swing speed > 150°/s — overstresses the body-yaw servo

## Open Questions
- Should the default be the "tracking" variant or the "no-tracking" variant? Leaning: no-tracking for hardware safety.
- Which hold duration is ideal? 0.3 s might be too short — test empirically.
- On a repeat trigger, should the direction be mirrored (start right instead of left)?
- Which audio file fits? Proposal: a pseudo-"whoosh" per swing.
- How does the behavior react when the hardware variant declares a tighter `max_body_yaw` (e.g. Lite with different calibration)? Measure on the device before implementation.
