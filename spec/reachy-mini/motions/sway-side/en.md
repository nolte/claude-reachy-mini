# Motion Sequence: Side Sway (`sway-side`)

Status: draft

## Context
A dance building block for rhythmic side-to-side swaying on the beat: Reachy rolls left and right on the beat. Use cases: dance app, reggae / slow-genre motions, background animation during slow-BPM music.

## Characteristics
- **BPM-parametrised**: one beat cycle = `60/BPM` seconds; one sway cycle moves centre → left → centre → right (i.e. one cycle covers two beats)
- Roll main motion: -15° to +15° — visible sway
- Antennas sway along (asymmetric by direction)
- Body yaw tracks roll slightly via IK
- **Loopable**

## Components

### Per 2-beat cycle actuator sequence

A full sway cycle (left + right) corresponds to two beats. `T_cycle = 2 × 60 / BPM`.

| Subphase | Duration (relative to T_cycle) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing |
|---|---|---|---|---|---|
| Roll left (beat 1) | 0.50 × T_cycle | (0, 0, +2, +15, +3, -3) | (-10, +15) | -3 | `MIN_JERK` |
| Roll right (beat 2) | 0.50 × T_cycle | (0, 0, +2, -15, +3, +3) | (+15, -10) | +3 | `MIN_JERK` |

**Entry**: 0.40 s — from neutral pose to centre with slight preparation.
**Exit**: 0.40 s — back to neutral pose via the relevant midpoint.

Minimum run for one cycle at 120 BPM (T_cycle = 1.0 s): 0.40 + 1.0 + 0.40 = 1.80 s.

### Audio
No own audio — `sway-side` reacts to external music.

### Idle modulation
None.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the small body-yaw offset (±3°) tracks the roll. With `False` the body would feel stiff.

## Implementation notes
- Preferred as a `Move` subclass with `SwaySide(bpm: float, beats: int, lead_time_s: float = 0.0)`. `beats` must be even (every 2-beat cycle is one sway cycle).
- BPM range: 50–140 (slower genres). At higher BPM the sway feels frantic.
- Roll velocity: 30° / (T_cycle / 2). At 120 BPM (T_cycle = 1 s, half-cycle = 0.5 s): 60°/s ≈ 1.05 rad/s — well within the 8 rad/s limit.
- Antenna asymmetry follows the dance choreography: with roll left, the right antenna (up) is high, the left (down) is low.
- Beat synchronicity: the peak of roll-left should coincide with beat 1, the peak of roll-right with beat 2.

## Acceptance Criteria
- [ ] External observers read the motion as "swaying" / "on the beat" / "reggae-like" (at least 4 out of 5)
- [ ] Roll swing reaches ±15° per half-cycle
- [ ] Antenna asymmetry is visibly consistent with the roll direction
- [ ] Body yaw moves subtly with the roll
- [ ] At BPM 120, roll peaks alternate every 0.5 s
- [ ] Behavior loops seamlessly
- [ ] On `cancel_move()` mid-cycle, the behavior settles to centre and the neutral pose

## Anti-patterns
- Roll amplitude > ±25° — feels exaggerated, near pitch/roll limits
- Antennas symmetric instead of asymmetric — feels stiff
- `CARTOON` easing — unnatural bouncing
- Body yaw opposed to roll — confusing pose
- BPM > 140 — the sway becomes too fast, no longer reads as "swayey"

## Open Questions
- Should the antenna asymmetry be flipped (roll left → left antenna up)? Test empirically — both variants have appeal.
- How is `sway-side` combined with `groove-bob`? Proposal: parallel `Move` composition possible if the subsystems are orthogonal (pitch vs. roll).
- How does the behavior react to very irregular beats (alternation between 4/4 and 3/4)? Leaning: ignore, accept cycle drift.
