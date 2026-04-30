# Motion Sequence: Disappointed (`disappointed`)

Status: draft

## Context
A milder form of `sad`: Reachy makes a single, medium-strength pitch dip, lets the head droop slightly, and stays there without the lift attempt of `sad`. Reads as "aw…" or "not as good as hoped". Use cases: a task only partially successful, follow-up behavior on an unrecognised voice command, "no" reply to a question.

## Characteristics
- **Difference to `sad`**: only a single pitch dip (no multi-step lowering), no lift attempt, less deep pose
- Pitch -15° to -18° (between `sad`'s -28° and the neutral pose) — moderate droop
- Antennas slightly lowered (-10°), not fully drooping
- Body yaw slightly off (-5°) — turned away, not sad-looking-away
- Medium tempo (~2.6 s); `MIN_JERK` only

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (mini lift) | 0.20 | (0, 0, +2, 0, +3, 0) | (+5, +5) | 0 | `MIN_JERK` | small lift |
| 2 | Dip down | 0.50 | (0, 0, -3, 0, -15, -3) | (-10, -10) | -2 | `MIN_JERK` | single medium-strength pitch drop |
| 3 | Slight droop | 0.50 | (0, 0, -5, +2, -18, -5) | (-12, -12) | -3 | `MIN_JERK` | pose deepens slightly |
| 4 | Hold with light breath | 0.80 | (0, 0, -5 (±1), +2, -18 (±1°), -5) | (-12, -12) | -3 | `MIN_JERK` (idle mod) | moderate breath, not heavy |
| 5 | Release | 0.60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | gentle return upright |

Total duration ≈ 2.60 s.

### Audio (optional)
A quiet sigh (≤ 500 ms), started with phase 2. Volume quiet (30). Shorter and less heavy than the `sad` sigh.

### Idle modulation during phase 4
Very light sinusoidal modulation on `z` (amplitude 1 mm, frequency 0.3 Hz) and `pitch` (amplitude 1°, in phase) — moderate breath, not the heavy breathing of `sad`.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the slight body yaw -3° reads softer through IK.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.60`, five phases — lean.
- Pitch -18° sits clearly between `sad` (-28°) and `agreeing-nod` (-12°) — medium strength.
- The absence of a lift attempt (unlike `sad`) is diagnostic — otherwise it reads like an abbreviated `sad`.
- Body yaw -3° is subtle; with `False` it would barely show — let IK coupling carry it.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "disappointed" / "aw" / "not great" (at least 4 out of 5)
- [ ] Total duration is 2.6 ± 0.3 s
- [ ] Pitch dip is visible (≥ -15°), but not as deep as in `sad`
- [ ] No lift attempt anywhere in the sequence
- [ ] Idle modulation in phase 4 is moderate, not heavy breathing
- [ ] Audio (if enabled) is shorter than the `sad` sigh
- [ ] Return to the neutral pose is gentle

## Anti-patterns
- Pitch deeper than -22° — becomes `sad`
- Adding a lift attempt like in `sad` — lengthens the sequence and defeats the moderate affect
- `CARTOON` easing — wrong hardness
- Antennas drooping as far as in `sad` (-25°) — reads as weak `sad`
- Audio with a loud or long sigh — over-paints the affect

## Open Questions
- Should body yaw drift at all, or stay strictly centred? A small turn-away makes the affect more human, but staying centred would be cleaner.
- Which audio file fits? Proposal: a short falling tone, quieter than `sad`.
- How does `disappointed` differ in practice from `sad` when both are triggered consecutively? Proposal: a `disappointed` directly followed by a `sad` is legitimate and reads as escalation.
