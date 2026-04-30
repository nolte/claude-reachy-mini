# Motion Sequence: Thinking (`thinking`)

Status: draft

## Context
A cyclic "processing…" behavior that runs as long as a background action lasts (voice inference, LLM reply, HA service call). Reachy shows pensive motion instead of looking frozen. Use cases: wait time for voice AI replies, long HA service calls, model inference, "please wait" state.

## Characteristics
- **Loopable**: the behavior is an open-ended cycle with entry and exit phases; the middle phases repeat until a stop signal arrives
- Pitch slightly up (+5°) — pensive, attentive pose
- Cyclic slow roll modulation (-8° → +8° → -8°) — like sorting thoughts
- Antennas slightly perked (+12°) and very faintly vibrating (amplitude 1°, frequency 0.5 Hz) — mental background process
- Body yaw centred with minimal modulation — the body stays calm, the head thinks
- Variable length: one loop cycle ≈ 2.5 s, number of loops as a parameter

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Entry (into the mode) | 0.40 | (0, 0, +3, 0, +5, 0) | (+12, +12) | 0 | `MIN_JERK` | smooth arrival at the thinking pose |
| 2 | Loop: roll left | 0.80 | (0, 0, +3, +8, +5, -3) | (+12 (±1°), +12 (±1°)) | -2 | `MIN_JERK` | sorting roll swing |
| 3 | Loop: roll centre | 0.40 | (0, 0, +3, 0, +5, 0) | (+12 (±1°), +12 (±1°)) | 0 | `MIN_JERK` | brief pass-through centre |
| 4 | Loop: roll right | 0.80 | (0, 0, +3, -8, +5, +3) | (+12 (±1°), +12 (±1°)) | +2 | `MIN_JERK` | mirrored swing |
| 5 | Loop: roll centre | 0.40 | (0, 0, +3, 0, +5, 0) | (+12 (±1°), +12 (±1°)) | 0 | `MIN_JERK` | loop transition or exit |
| 6 | Exit (release) | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth handover to follow-up behavior |

Loop cycle (phases 2–5) = 2.40 s. Entry + exit = 0.80 s. Minimum duration (1 cycle) ≈ 3.20 s.

### Audio (optional)
Very quiet "hmm…" loop or quiet ticking (≤ 1.0 s per cycle), started with phase 2. Volume very low (20). Audio can loop across cycles without re-trigger.

### Idle modulation
Antenna vibration is the idle modulation and runs continuously in loop phases 2–5. Frequency 0.5 Hz, amplitude 1° — very subtle.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the body-yaw offset of ±2° tracks the head roll smoothly. With `False` the pose feels static.

## Implementation notes
- Preferred as a parametrised `Move` subclass: `Thinking(min_loops=1, max_loops=None)`. With `max_loops=None`, the behavior is ended externally via `cancel_move()` once the background task completes.
- `evaluate(t)` must compute phases 2–5 modulo loop duration — the antenna vibration runs continuously while the roll switches between phases.
- If the background task ends quickly (< 2 s), run at least one full loop so the gesture is legible, then jump cleanly into the exit phase.
- Roll ±8° and antenna values sit clearly within the limits.

## Acceptance Criteria
- [ ] External observers read the motion as "thinking" / "processing" / "waiting with activity" (at least 4 out of 5)
- [ ] Entry + one loop + exit take 3.2 ± 0.3 s
- [ ] Roll switches inside the loop read as "sorting"
- [ ] Antenna vibration is subtle but visible
- [ ] Behavior exits cleanly on `cancel_move()` without a jump — the exit drives to the neutral pose
- [ ] Multiple loops run without visible jumps at the loop boundaries
- [ ] Audio (if enabled) loops seamlessly

## Anti-patterns
- Fast roll switches (< 0.4 s per phase) — feels nervous, not pensive
- Antenna vibration without modulation — feels frozen
- `CARTOON` or `LINEAR` easing in the loop phases — wrong character
- Body-yaw modulation > ±5° — feels restless
- Loud audio (volume > 30) — defeats the calm gesture
- Loop duration < 2 s per cycle — too fast to read

## Open Questions
- Should the duration of individual loop phases carry a small random drift so no two cycles read identically? "Timing variation" pattern from `control-surface`.
- Which audio file fits? Proposal: a quiet murmur or a subtle ticking sound.
- How does `thinking` integrate with a status display via the mic-module LED ring (slow pulse)? A combined implementation makes sense.
- How is an abrupt termination (e.g. the background task throws an exception) handled? Proposal: chain into `confused` or `disappointed` depending on outcome.
