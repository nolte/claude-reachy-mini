# Motion Sequence: Soft Headbang (`headbang-soft`)

Status: draft

## Context
An expressive dance building block for rhythmic strong nodding without crossing into Stewart-platform-limit territory: a "soft" headbang for rock / metal / pop music. Use cases: dance app on energetic music, demo mode, "yeah!" reaction in dance sessions.

## Characteristics
- **BPM-parametrised**: one beat cycle = `60/BPM` seconds; per cycle one strong down-bang + one up-recover
- Pitch main motion: -15° (bang) to +5° (recover) — markedly stronger than `groove-bob`
- Antennas slightly perked (+12°), stationary (asymmetry reinforces the bang's hardness)
- Body yaw centred
- Lower BPM range than `groove-bob`, because the bang needs more pitch velocity

## Components

### Per-beat actuator sequence

`T = 60 / BPM`. Asymmetric phase split: fast down (bang), longer up (recover).

| Subphase | Duration (relative to T) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing |
|---|---|---|---|---|---|
| Bang down | 0.30 × T | (0, 0, -5, 0, -15, 0) | (+12, +12) | 0 | `EASE_IN_OUT` |
| Recover up | 0.70 × T | (0, 0, +2, 0, +5, 0) | (+12, +12) | 0 | `MIN_JERK` |

**Entry**: 0.30 s — from neutral pose to up-pose.
**Exit**: 0.40 s — back to neutral pose.

Minimum run for 1 beat at 90 BPM (T = 0.67 s): 0.30 + 0.67 + 0.40 = 1.37 s.

### Audio
No own audio — reacts to external music.

### Idle modulation
None.

### Body yaw and IK
`automatic_body_yaw=True` is fine but has no visible effect.

## Implementation notes
- Preferred as a `Move` subclass `HeadBangSoft(bpm: float, beats: int, lead_time_s: float = 0.0)`.
- BPM range: 60–130. Above that the pitch velocity rises (130 BPM → T = 0.46 s, down-phase = 0.14 s, a 20° pitch swing in 0.14 s = 143°/s ≈ 2.5 rad/s — within 8 rad/s; at 160 BPM it would be ~180°/s ≈ 3.1 rad/s, still OK, but servo heat builds up faster).
- `EASE_IN_OUT` in the down-phase makes the bang harder than `MIN_JERK` without crossing into `LINEAR` rigidity.
- Asymmetric phase split (30/70) produces the typical headbang character: fast hit, longer rebound.
- More than 8 consecutive bangs per sequence: check servo heat — `mini.imu.temperature` (Wireless) or insert a cool-down.

## Acceptance Criteria
- [ ] External observers read the motion as "headbanging" / "rockish" / "strong on the beat" (at least 4 out of 5)
- [ ] Pitch swing reaches -15° to +5° per beat
- [ ] Down-phase is visibly faster than up-phase (asymmetric)
- [ ] Antennas stay stationary — no co-banging
- [ ] At BPM 100, the down-bang is within ±25 ms of the audio beat
- [ ] Behavior loops over multiple beats without jumps
- [ ] On `cancel_move()` mid-bang, the behavior settles into up-pose and the neutral pose

## Anti-patterns
- Pitch amplitude > ±20° — exceeds comfort zone, risks near-singularity
- Symmetric phases (50/50) — loses the headbang character
- Antennas banging along — feels like wild flailing
- `LINEAR` easing in the down-phase — feels violent, not musical
- BPM > 140 — the bang becomes frantic
- More than 16 consecutive bangs without cool-down — servo heat buildup

## Open Questions
- Should variable bang strengths (e.g. every 4th beat amplified) be part of the sequence rather than a fixed amplitude?
- Which upper BPM bound is safe for sustained performance? Test empirically on hardware.
- Should the behavior automatically downshift to `groove-bob` when servo temperature crosses a threshold?
