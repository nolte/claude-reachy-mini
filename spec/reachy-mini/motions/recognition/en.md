# Motion Sequence: Recognition / "Aha!" (`recognition`)

Status: draft

## Context
A quick "I see now" gesture: Reachy makes a brief upward `surprised`-like snap and immediately nods twice in agreement. The combined "small startle + immediate agreement" reads as "I recognised something I accept". Use cases: voice recognition matches a command, "I understood what you mean", transition from `thinking` into a successful action.

## Characteristics
- Fast up-snap like a softened `surprised` — head up, Z lift
- Two pitch nods directly afterward — like an abbreviated `agreeing-nod`
- Antennas perked, slightly offset from head pitch
- Body yaw centred
- Fast tempo (~1.5 s); `EASE_IN_OUT` for the snap, `MIN_JERK` for the nods

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Pre-anticipation | 0.05 | (0, 0, -1, 0, -2, 0) | (+5, +5) | 0 | `LINEAR` | minimal flinch |
| 2 | Aha snap (up) | 0.15 | (0, 0, +12, 0, +18, 0) | (+35, +35) | 0 | `EASE_IN_OUT` | fast up-snap, softer than `surprised` |
| 3 | Mini hold | 0.20 | (0, 0, +12, 0, +18, 0) | (+35, +35) | 0 | `MIN_JERK` (static) | hold the pose — "recognition" |
| 4 | Down-nod 1 | 0.18 | (0, 0, +5, 0, -5, 0) | (+20, +20) | 0 | `MIN_JERK` | first agreement nod |
| 5 | Up between | 0.12 | (0, 0, +6, 0, +5, 0) | (+22, +22) | 0 | `MIN_JERK` | brief swing back |
| 6 | Down-nod 2 | 0.15 | (0, 0, +3, 0, -3, 0) | (+18, +18) | 0 | `MIN_JERK` | second, weaker nod |
| 7 | Hold (understood) | 0.20 | (0, 0, +2, 0, -2, 0) | (+15, +15) | 0 | `MIN_JERK` | held slightly down |
| 8 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.45 s.

### Audio (optional)
A two-part audio: a short "aha!" (≤ 200 ms) on phase 2, followed by a quiet "yes!" (≤ 200 ms) on phase 4. Volume moderate (50). Both rising.

### Idle modulation
None — the sequence is too compact, any modulation would blur it.

### Body yaw and IK
`automatic_body_yaw=True` is acceptable; body stays fully centred.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.45`. The combination snap + nods is the character — phase 3 (mini hold) is the diagnostic pause between both parts.
- `recognition` is explicitly a composition of two affects and should be implemented as such — it can be assembled via a `compose()` pattern from an abbreviated `surprised` and an abbreviated `agreeing-nod`, if a composition API is available.
- Pitch jump from -2° to +18° in 0.15 s = ~133°/s ≈ 2.3 rad/s, inside the joint limit.
- `EASE_IN_OUT` in phase 2 (instead of `LINEAR` like `surprised`) makes the snap softer — `recognition` is not a pure startle.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "aha" / "got it" / "understood now" (at least 4 out of 5)
- [ ] Total duration is 1.5 ± 0.2 s
- [ ] Phase 2 is a clear up-snap but softer than in `surprised`
- [ ] Phase 3 (mini hold) reads as a visible pause before the nods
- [ ] Both nods are visible and the second is weaker than the first
- [ ] Audio (if enabled) has two beats (snap + nod)
- [ ] Handover to a follow-up behavior (e.g. successfully recognised action) is clean

## Anti-patterns
- `LINEAR` snap in phase 2 — reads as a pure `surprised` reaction, recognition character is lost
- Phase 3 (hold) longer than 0.3 s — the sequence falls apart
- Only one nod — feels indecisive
- Roll or yaw components — defeat the clear vertical affect
- Audio only on phase 2 without a follow-up beat — the snap+agree split is lost

## Open Questions
- Should `recognition` have a shorter variant (only snap + one nod, ~1.0 s)?
- Which audio file fits? Proposal: two short tones in rising pitch.
- Is implementation as a "composition of surprised + agreeing-nod" the right shape, or stand-alone? Leaning: stand-alone — the transition phase 3 makes the combination its own gesture.
