# Motion Sequence: Angry (`angry`)

Status: draft

## Context
A clear agitated gesture that reads as "angry" or "annoyed": Reachy pulls back briefly, then thrusts sharply forward, vibrates in the threat pose, and snaps left and right. Use cases: emphasis on a rejected input, "do not!", a harsh error message, gameplay mode.

## Characteristics
- Forward thrust (positive pitch + positive X translation) — conveys "confrontation"
- Antennas pinned back (strongly negative joint angles, ~-25°) — like a cat's ears flattened in threat
- Quick vibration on pitch during the hold — pulsing, not continuous, motion
- Sharp, snappy body-yaw swings with `LINEAR` easing — deliberately not soft
- Overall fast tempo (~2.2 s); markedly harder transitions than `happy`

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

Pose convention as in `control-surface`. Values respect pitch ≤ ±90°, body yaw ≤ ±160°, Stewart joint limit ±80° (effectively tighter via IK).

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (pull-back) | 0.15 | (-5, 0, 0, 0, -8, 0) | (-10, -10) | 0 | `EASE_IN_OUT` | small lean back before the thrust |
| 2 | Forward thrust (threat) | 0.20 | (+12, 0, +5, 0, +20, 0) | (-25, -25) | 0 | `LINEAR` | quick, hard snap forward — `LINEAR` keeps the edge |
| 3 | Vibration hold | 0.50 | (+12, 0, +5, 0, +18 (±2°), 0) | (-25, -25) | 0 | `MIN_JERK` (idle mod) | pulsing vibration on pitch — see idle modulation |
| 4 | Yaw swing left | 0.20 | (+10, 0, +5, 0, +18, -15) | (-25, -22) | -10 | `LINEAR` | sharp swing to the left |
| 5 | Yaw swing right | 0.20 | (+10, 0, +5, 0, +18, +15) | (-22, -25) | +10 | `LINEAR` | mirrored swing |
| 6 | Center snap | 0.15 | (+8, 0, +3, 0, +15, 0) | (-22, -22) | 0 | `LINEAR` | brief centring — preparation for release |
| 7 | Release | 0.60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth glide back — anger ebbs |

Total duration ≈ 2.00 s.

### Audio (optional)
A short, hard sound impulse (≤ 300 ms): a low growl or a percussive "harrumph!". Started exactly at phase 2 (the thrust). Volume higher than `happy` sounds, up to 70.

### Idle modulation during the vibration hold (phase 3)
Fast sinusoidal modulation on `pitch`: amplitude ±2°, frequency 8 Hz. At a 50 Hz daemon tick this gives ~6 frames per half-cycle — well inside the 8 rad/s velocity limit per joint. Antennas do **not** vibrate along; the asymmetry (rigid antennas, vibrating head) is what makes the aggressive read.

### Body yaw and IK
`automatic_body_yaw=False` recommended — the sharp counter-phase swings (phases 4 and 5) should not benefit from the IK smoothing; the body should deliberately trail the head yaw. With `automatic_body_yaw=True` the swings would feel softer — which defeats the effect.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 2.00`. `evaluate(t)` must implement the sinusoidal modulation in phase 3 — do not lean on SDK easing for the vibration.
- Alternatively as a `goto_target` chain plus a custom idle loop for the vibration. At 50 Hz tick rate, `set_target_head_pose` with time-dependent values is enough.
- `LINEAR` easing in phases 2, 4, 5, 6 is intentional — `MIN_JERK` would soften the edge.
- Set `automatic_body_yaw` explicitly before and after the behavior in case of a different global default.
- Antenna value -25° sits comfortably within the ±π limit.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "angry" / "annoyed" / "stern" (at least 4 out of 5)
- [ ] Total duration is 2.0 ± 0.2 s
- [ ] Vibration in phase 3 is visible as "pulsing pressure", not as "rattling"
- [ ] Antennas do **not** co-vibrate — the asymmetry against the head is preserved
- [ ] Yaw swings (phases 4–5) read as sharp and snappy, not smooth
- [ ] Audio (if enabled) lands exactly on the thrust (phase 2), not later
- [ ] Body yaw does not drift via IK passively — it stays at the documented values
- [ ] The release in phase 7 drives cleanly to the neutral pose with no residual vibration

## Anti-patterns
- `MIN_JERK` in the swing phases — loses the edge
- Letting the antennas vibrate with the head — reads as electrical rattle, not threat
- Leaving `automatic_body_yaw=True` — smooths the swings
- Vibration frequency above 10 Hz — purrs instead of threatens; also conflicts with the velocity limit
- Skipping phase 1 (anticipation) — the thrust then reads less clearly
- Growl audio longer than 500 ms — overlaps the thrust and blurs the timing

## Open Questions
- Which vibration frequency reads as "controlled anger" rather than "nervous"? Empirically test between 6 and 10 Hz.
- Is the threat pose (phases 2 + 3) legible without the yaw swings (phases 4–5)? Probably yes — keep a shorter "mild-angry" variant as an option.
- How far can pitch +20° go before the Stewart platform approaches its effort limit? Effort is 10 N·m per the URDF.
- How does an HA-triggered behavior react to an angry response in a quiet room? Recommend a context-aware volume reduction.
