# Motion Sequence: Flinch (`flinch`)

Status: draft

## Context
An extremely fast startle reaction: Reachy snaps back, freezes briefly, and recovers. Harder and shorter than `surprised` — the body stays still, only the head snaps back. Use cases: loud noise (HA sensor), sudden motion in the camera frame, pre-stage of a security alarm.

## Characteristics
- **Fastest** behavior in the repertoire (~1.15 s total)
- X translation negative (-10 mm) — the whole head pulls back
- Z negative (-3 mm), pitch slightly down (-8°) — bashful-defensive pose
- Antennas flat back (-25°) — like ears flattened
- Frozen hold immediately after the snap — shock
- `LINEAR` snap for hardness

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Pre-anticipation | 0.05 | (+1, 0, 0, 0, 0, 0) | (0, 0) | 0 | `LINEAR` | minimal forward flinch |
| 2 | Snap back (flinch) | 0.10 | (-10, 0, -3, 0, -8, 0) | (-25, -25) | 0 | `LINEAR` | very fast hard snap backward |
| 3 | Frozen hold | 0.30 | (-10, 0, -3, 0, -8, 0) | (-25, -25) | 0 | — (static) | shock freeze, no modulation |
| 4 | Mini recovery | 0.20 | (-3, 0, -1, 0, -3, 0) | (-10, -10) | 0 | `MIN_JERK` | small recovery, still slightly back |
| 5 | Release | 0.50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.15 s.

### Audio (optional)
A very short, hard breath or "eh!" sample (≤ 150 ms), started exactly at phase 2. Volume moderate (50).

### Idle modulation
None — phase 3 is explicitly static (frozen hold).

### Body yaw and IK
`automatic_body_yaw=False` — the body must deliberately NOT track the head snap; that is the whole point of a flinch (head only, not body).

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.15`. Very compact but high precision — phase 2 is only 0.10 s and the velocity peaks here.
- Pitch jump from 0° to -8° in 0.10 s = 80°/s ≈ 1.4 rad/s — within 8 rad/s.
- X translation from +1 mm to -10 mm in 0.10 s = 110 mm/s — within IK volume and velocity budget.
- Antenna snap from 0° to -25° in 0.10 s = 250°/s ≈ 4.4 rad/s — just over half the joint velocity limit.
- `LINEAR` easing in phases 1 and 2 is mandatory — `MIN_JERK` would soften the snap.
- Set `automatic_body_yaw=False` before behavior start if it is otherwise the global default.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "flinch" / "startled" / "defensive" (at least 4 out of 5)
- [ ] Total duration is 1.15 ± 0.1 s
- [ ] Phase 2 reads as a visible hard snap (≤ 0.12 s)
- [ ] Frozen hold (phase 3) reads as a visible pause
- [ ] Body yaw stays strictly at 0° throughout the entire behavior
- [ ] Antennas are visibly flattened back (-25°) in phases 2 and 3
- [ ] Audio (if enabled) hits phase 2 exactly

## Anti-patterns
- `MIN_JERK` snap in phase 2 — the hardness is lost, feels like soft `surprised`
- Frozen hold with modulation — defeats the shock pause
- Body yaw co-swinging — disperses the reaction across too many axes
- Antennas positive (perked) — wrong association, opposite of "ears flat"
- Recovery phase longer than 0.3 s — feels like sad lingering
- Pitch positive — wrong direction, the flinch should snap down/back

## Open Questions
- Should the startle pose carry a small roll component (e.g. +5°) to read as "side-defensive"? Test empirically.
- Which audio file fits? Proposal: a short "eh!" or "oh!" beep with a falling tone.
- How does `flinch` react to a repeat trigger within < 1 s? Proposal: ignore the second trigger because the behavior is shorter than typical trigger spacing.
- Can `flinch` chain into `alarm` when the startle source is a security event? Proposal: yes, seamless handover.
