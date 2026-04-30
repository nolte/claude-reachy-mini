# Motion Sequence: Scanning the Room (`scanning`)

Status: draft

## Context
A calm, continuous head sweep from left to right (and back) — like a security-camera pan. Use cases: vision-based person search, "where is…?" reply mode, demo mode for surveying surroundings, initialising `look_at_world` via room exploration.

## Characteristics
- Large continuous yaw sweep (-40° → +40° → -40° → 0°) — no abrupt switches, no "look-and-hold"
- Pitch slightly up (+5°) — camera field is raised
- Antennas perked (+25°) but stationary
- Body yaw tracks the head yaw smoothly (IK-coupled) — the whole body pans
- Long tempo (~4.1 s); `EASE_IN_OUT` only for the sweeps

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (perk) | 0.20 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `MIN_JERK` | reach the scan pose |
| 2 | Sweep left | 1.20 | (0, 0, +3, 0, +5, -40) | (+25, +25) | -25 | `EASE_IN_OUT` | slow swing to the left |
| 3 | Sweep right | 1.80 | (0, 0, +3, 0, +5, +40) | (+25, +25) | +25 | `EASE_IN_OUT` | twice-as-far swing through centre |
| 4 | Sweep to centre | 0.90 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `EASE_IN_OUT` | centring outro |
| 5 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 4.50 s.

### Audio (optional)
A very quiet continuous search beep (≤ 100 ms per beep, every 0.5 s) during phases 2–4. Volume very low (20). Optionally also without audio.

### Idle modulation
No additional modulation — the sweep itself is the motion.

### Body yaw and IK
`automatic_body_yaw=True` recommended — body tracks head smoothly through IK. Since head yaw goes to ±40° and body yaw to ±25°, the relative yaw stays at ±15° (well inside `max_relative_yaw=±65°`). With `False` the head would feel "twisted" against the body.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 4.50`. Ideally compatible with `look_at_world` calls that target a detected point — the scan is then replaced by a tracking behavior.
- Yaw speed in phase 3: 80°/1.8 s = 44°/s ≈ 0.77 rad/s, well below the 8 rad/s joint limit.
- Phase 3 is intentionally longer than phase 2 (1.8 s vs. 1.2 s), because it covers 80°, not 40°.
- With an active vision pipeline: the behavior can be interrupted via `cancel_move()` as soon as the camera detects a target; then immediately call `look_at_world(target)`.

## Acceptance Criteria
- [ ] External observers read the motion as "searching" / "scanning" / "observing" (at least 4 out of 5)
- [ ] Total duration is 4.5 ± 0.4 s
- [ ] The sweep is even, no interim stops
- [ ] Body yaw visibly moves with the head yaw
- [ ] Pitch stays constant at +5° during the sweep
- [ ] Antennas static at +25°
- [ ] On `cancel_move()` mid-sweep, the behavior settles cleanly at the current pose and releases

## Anti-patterns
- Pauses between sweeps — breaks the continuous search feel
- Pitch modulation during the sweep — feels uncertain
- `LINEAR` easing — feels mechanical like a camera pan
- Body yaw not moving along — feels twisted
- Antenna modulation — defeats the calm scan character
- Sweep speed > 60°/s — feels panicked rather than calm

## Open Questions
- Should the scan always start in the same direction (left first), or random?
- With an active vision pipeline, should the scan be automatically replaced by `look_at_world` once a target is detected? Pro: context-relevant; con: increases composition complexity.
- Which audio file fits? Proposal: a quiet sonar-style beep, or no audio.
- Should there be a shorter variant (`scanning-quick`) that only sweeps ±25° and runs ~2 s?
