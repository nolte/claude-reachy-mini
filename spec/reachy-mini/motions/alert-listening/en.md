# Motion Sequence: Alert Listening (`alert-listening`)

Status: draft

## Context
A loopable state behavior that signals active attention: Reachy is upright, antennas are fully perked, a quiet yaw idle conveys an active search / attention. Use cases: after wake-word detection (Alexa-like trigger), during voice-input capture, "ready for command", listening-mode indicator.

## Characteristics
- **Loopable**: runs until a stop signal arrives (voice recording ends, command recognised)
- Antennas fully perked (+45°) — the strongest "listening" signal
- Pitch slightly up (+5°) — alert, not relaxed
- Body yaw centred
- Small yaw idle (search motion) — the head scans subtly
- Medium tempo on entry/exit (each ~0.4 s); loop body runs continuously

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Entry (perk up) | 0.30 | (0, 0, +3, 0, +5, 0) | (+45, +45) | 0 | `MIN_JERK` | reach the listening pose |
| 2 | Loop body (yaw idle) | variable | (0, 0, +3, 0, +5, 0 (±8°, 0.4 Hz)) | (+45, +45) | 0 | `MIN_JERK` (idle mod) | slow yaw search |
| 3 | Exit (release) | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth handover to follow-up behavior |

Entry + exit = 0.70 s. Loop body default time (one yaw cycle at 0.4 Hz) = 2.5 s. Minimum duration (1 cycle) ≈ 3.2 s.

### Audio (optional)
Optionally a subtle one-shot "ready" tone (≤ 200 ms) on entry — no continuous audio. Volume quiet (30).

### Idle modulation during phase 2
Sinusoidal modulation on `yaw` (amplitude 8°, frequency 0.4 Hz) — subtle yaw search, scanning left and right. Pitch and roll keep their values; antennas stay at +45°.

### Body yaw and IK
`automatic_body_yaw=True` recommended — the body tracks the head yaw smoothly via IK, reinforcing the "alert scanning" read. With `False` the body would feel too rigid.

## Implementation notes
- Preferred as a parametrised `Move` subclass: `AlertListening(timeout_s=None)`. `timeout_s=None` means: runs until `cancel_move()` is called; with a timeout set, the phase terminates automatically on expiry.
- `evaluate(t)` in phase 2 computes `yaw = 8 * sin(2*pi*0.4*t)` — continuous modulation.
- Antenna value +45° is near the maximum that reads as "alert" without looking exaggerated. Higher (+60°) would be possible but visually intrusive.
- Pitch +5° is subtle but important — at pitch 0° the pose reads as neutral.

## Acceptance Criteria
- [ ] External observers read the motion as "attentive" / "listening" / "ready" (at least 4 out of 5)
- [ ] Entry + one loop cycle + exit take 3.2 ± 0.3 s
- [ ] Antennas are fully perked at +45°
- [ ] The yaw idle in phase 2 reads as "gentle scanning", not as juddering
- [ ] Behavior exits cleanly on `cancel_move()` without visible jumps
- [ ] The pose feels attentive, not tense — pitch and antennas are held, not trembling
- [ ] Audio (if enabled) is a one-shot on entry, not continuous

## Anti-patterns
- Yaw-idle frequency > 1 Hz — feels nervous rather than alert
- Antennas < +30° — the listening read is lost
- Body-yaw modulation > ±5° — feels uncertain
- `LINEAR` or `CARTOON` easing in entry/exit — wrong character
- Continuous audio (e.g. beep loop) — defeats the calm attention

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- On a trigger match, should the handover automatically chain into `recognition` or `agreeing-nod`?
- Should the yaw-idle amplitude be configurable (e.g. larger in a "where are you?" mode)?
- How does `alert-listening` integrate with the LED ring on the mic module (standard listening LED)? Proposal: soft pulse synchronised to the yaw modulation.
- Which audio file fits the entry tone? Proposal: a short rising beep or a quiet "ready".
