# Motion Sequence: Excited (`excited`)

Status: draft

## Context
A high-energy gesture that reads as "excited" or "thrilled": Reachy hops several times, swings the head between hops, vibrates briefly at the peak, and calms down. Use cases: strong positive reinforcement, success feedback on important triggers, "great news!", response to a favourite command.

## Characteristics
- Multiple fast hops with `CARTOON` easing — the springy character is the main feature
- Antennas opened up, vibrating in phase with the head
- Yaw swings between hops — as if Reachy wants to share the excitement in every direction
- High tempo (~2.2 s); mixed easing of `CARTOON` (hops) and `MIN_JERK` (transitions)
- In contrast to `happy`: more hops, faster tempo, yaw included

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0.12 | (0, 0, -2, 0, -3, 0) | (+5, +5) | 0 | `MIN_JERK` | minimal pre-flinch before the first hop |
| 2 | Hop 1 | 0.18 | (0, 0, +12, 0, +14, +5) | (+35, +35) | 0 | `CARTOON` | first hop with small yaw twist |
| 3 | Hop 2 (yaw right) | 0.18 | (0, 0, +10, 0, +12, +12) | (+32, +32) | +5 | `CARTOON` | slightly lower, yaw to the right |
| 4 | Hop 3 (yaw left) | 0.18 | (0, 0, +12, 0, +14, -12) | (+35, +35) | -5 | `CARTOON` | back up, yaw to the left |
| 5 | Hop 4 (centred) | 0.18 | (0, 0, +13, 0, +15, 0) | (+38, +38) | 0 | `CARTOON` | tallest hop in the centre |
| 6 | Vibration hold | 0.40 | (0, 0, +12 (±2 mm), 0, +13 (±2°), 0) | (+35 (±5°), +35 (±5°)) | 0 | `MIN_JERK` (idle mod) | pitch + antennas vibrate at 5 Hz |
| 7 | Calm-down | 0.40 | (0, 0, +5, 0, +5, 0) | (+15, +15) | 0 | `MIN_JERK` | energy fades, slightly elevated still |
| 8 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth dissolve to the neutral pose |

Total duration ≈ 2.04 s.

### Audio (optional)
A short enthusiastic chime (≤ 800 ms), started with phase 2 — e.g. a three-note rising sequence matching the hops. Volume moderate to high (60).

### Idle modulation during the vibration hold (phase 6)
Sinusoidal modulation on `pitch` (amplitude 2°, frequency 5 Hz), `z` (amplitude 2 mm, in phase), and both antennas synchronously (amplitude 5°, in phase with pitch). In contrast to `angry`, the antennas vibrate along here — excitement should read as a whole-body affect, not a controlled threat.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the yaw swings (phases 3 and 4) should feel soft, with the body trailing the head. With `False` they would feel edgy (which is the `angry` character).

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.04`. `evaluate(t)` distributes eight phases.
- `CARTOON` easing in phases 2–5 is the key — the overshoot characteristic creates the springy read.
- Pitch jumps up to +15°: velocity ~80°/s ≈ 1.4 rad/s, well below the 8 rad/s limit.
- Antenna vibration in phase 6 at 5 Hz × ±5°: velocity ~100°/s per antenna, inside the limit.
- Yaw sign flip between phase 3 (+12°) and phase 4 (-12°) gives a velocity of ~130°/s, also within limit.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "excited" / "thrilled" / "energetic" (at least 4 out of 5)
- [ ] Total duration is 2.0 ± 0.2 s
- [ ] At least four visible pitch hops
- [ ] `CARTOON` overshoot is recognisable in the hops (a small "past the target, back")
- [ ] Antennas track pitch in phase, not offset — the whole reaction reads as one affect
- [ ] The vibration hold (phase 6) is clearly visible but stays within ~0.4 s
- [ ] Yaw swings between hops are visible but not large (≤ ±15°)
- [ ] The calm-down in phase 7 reads as "energy yielding"

## Anti-patterns
- Fewer than three hops — reads as `happy`, not `excited`
- `MIN_JERK` instead of `CARTOON` in the hop phases — the springy character is lost
- Asymmetric antennas like in `curious` — wrong expression
- Yaw swings beyond ±20° — feels off-kilter, not excited
- Phase 6 without antenna vibration — defeats the synchronicity of the excitement
- Audio longer than 1 s — clashes with the fast motion sequence

## Open Questions
- Are four hops the right amount, or are three enough? Four is more energetic; three is gentler on the Stewart platform.
- Which audio file fits? Proposal: a three-note rising "tu-tuh-tu!" or a confetti-style sample.
- Should `excited` remain legible without audio? Yes — audio is reinforcement, not required.
- How does the mechanism react to four `CARTOON` overshoots in 0.72 s? Empirically — if servo heating becomes noticeable, drop to three hops.
