# Motion Sequence: Shy (`shy`)

Status: draft

## Context
A reserved gesture that reads as "embarrassed" or "self-conscious": Reachy turns away from the trigger, lowers the head slightly, peeks back briefly, and hides away again. Use cases: receiving praise ("you're so smart!"), an awkward demo moment, "this makes me uncomfortable" trigger.

## Characteristics
- Body yaw and head yaw away from the trigger (same direction) — turning away
- Pitch slightly lowered (-10°) — bashful pose
- Antennas slightly lowered (-15°) — folded back
- Small peek-back: a brief glance in phase 3, then turn away again
- Medium tempo (~2.1 s); `MIN_JERK` and `EASE_IN_OUT` only

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (perk) | 0.15 | (0, 0, +1, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | minimal lift, "oh!" |
| 2 | Turn away | 0.40 | (0, 0, -2, 0, -10, -25) | (-15, -15) | +30 | `EASE_IN_OUT` | body and head to one side, pitch down |
| 3 | Mini peek-back | 0.25 | (0, 0, -2, 0, -8, -10) | (-12, -12) | +30 | `EASE_IN_OUT` | brief glance back |
| 4 | Turn away again | 0.30 | (0, 0, -2, 0, -10, -25) | (-15, -15) | +30 | `EASE_IN_OUT` | back away again |
| 5 | Hold (embarrassed) | 0.50 | (0, 0, -3, 0, -12, -25) | (-15, -15) | +30 | `MIN_JERK` (static) | hold the pose |
| 6 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 2.10 s.

### Audio (optional)
A quiet, almost embarrassed beep or muted "hm…" (≤ 350 ms), started with phase 2. Volume quiet (30). Rising tone, in contrast to `disagreeing-shake`.

### Idle modulation
None.

### Body yaw and IK
`automatic_body_yaw=True` recommended — body and head move toward the same side, IK smooths it. Head yaw -25° and body yaw +30° are in **opposite** sign directions — caution: that gives a relative yaw of -55°, just within the ±65° limit.

> ⚠ Note: in the table above `head yaw = -25°` and `body yaw = +30°` are noted as an **opposed turn** — the body rotates clockwise (positive), the head counter. That gives the "body away from trigger, head glancing back" feel. Verify the sign convention before implementation.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.10`, six phases.
- The yaw sign convention is critical — quick test: positive body yaw = body rotates to the right (from above). Validate on the device before implementation.
- The relative yaw between body and head must stay within the limits (≤ ±65°). In phase 2 it is -55° — safe headroom.
- `EASE_IN_OUT` easing in phases 2–4 makes the turn-away organic.
- Pitch -12° in phase 5 is subtle, not as deep as `sad`.
- Variant: a mirrored version (body yaw negative, head yaw positive) for turning away the other way.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "shy" / "embarrassed" / "bashful" (at least 4 out of 5)
- [ ] Total duration is 2.1 ± 0.2 s
- [ ] Body and head both turn away from the trigger
- [ ] Phase 3 (mini peek) reads as "looking back", shorter than phase 2 or 4
- [ ] The hold (phase 5) reads as an "embarrassed pause"
- [ ] Audio (if enabled) sounds reserved, not exclamatory

## Anti-patterns
- Body and head to the same side as the trigger — reads as `peek`, not bashful
- Pitch < -18° — becomes `disappointed`
- Hold phase 5 longer than 0.8 s — feels sulky rather than shy
- Audio loud or exclamatory — defeats the embarrassment
- Multiple peek phases — feels indecisive rather than shy

## Open Questions
- How is the "trigger direction" determined — passed by the caller or via `look_at_image` of the detected person?
- Which audio file fits? Proposal: a muted "hm…" with a rising tone.
- On a repeat trigger, should the turn-away direction be mirrored?
- How does `shy` differ from the planned `disgust`? Proposal: `shy` is social/affective, `disgust` is sensory/aversive.
