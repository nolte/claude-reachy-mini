# Motion Sequence: Alarm (`alarm`)

Status: draft

## Context
A hard warning signal with pulsing pose and LED pulses on the microphone module. Use cases: security violation detected (HA sensor), fire / smoke / CO alarm, critical system error, hardware emergency-stop announcement.

## Characteristics
- Fast upward snap with pitch vibration (10 Hz, ±5°) — conveys "attention now!"
- Antennas fully perked (+50°) and static — no modulation
- LED ring on the mic module pulses red synchronously with the pitch vibration
- Body yaw centred
- Fast tempo (~2.0 s); a hard `LINEAR` snap

## Platform profile

| Platform | Head vibration | LED pulses | Audio | Current-spike emergency stop |
|---|---|---|---|---|
| Reachy Mini (Wireless) | full | full, synchronised via `audio_control_utils` | full, high (volume 80) | yes, via IMU / daemon data |
| Reachy Mini Lite | full | full, synchronised | full | only daemon-published effort data (no IMU) |
| Simulation | full (pose values) | not available | not available | not available — emergency stop only via logic triggers (timeout, pose out-of-range) |

Implementation consequence: in simulation the behavior must **not** fail due to missing LED or audio subsystems — those channels are optional. On Wireless the multi-channel cue (motion + LED + audio) is **diagnostic** and mandatory because it preserves alarm legibility under stress. On Lite the same applies except for IMU-based emergency-stop triggers.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Snap up (alarm start) | 0.10 | (0, 0, +10, 0, +15, 0) | (+50, +50) | 0 | `LINEAR` | very fast hard snap |
| 2 | Vibration hold (alarm) | 1.50 | (0, 0, +10, 0, +12 (±5°, 10 Hz), 0) | (+50, +50) | 0 | `MIN_JERK` (idle mod) | pulsing pitch + LED red blinking |
| 3 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | gentle outro — alarm ends |

Total duration ≈ 2.00 s.

### Audio (optional, recommended)
A repeating alarm tone (e.g. a two-note beep) during phase 2, started with phase 1. Volume high (80). Audio is not optional in real alarm situations.

### Idle modulation and LED sync during phase 2
**Pitch vibration**: `pitch = 12 + 5 * sin(2*pi*10*t)` — 10 Hz, amplitude 5°. At 50 Hz daemon tick this gives 5 frames per half-cycle — inside the velocity limit (10 Hz × 5° × 2π = ~3.1 rad/s, well under 8 rad/s).

**LED sync**: via `audio_control_utils`, `LED_EFFECT` (red blinking) and `LED_BRIGHTNESS` are modulated in sync with the pitch phase. Concrete register values are `> ⚠ TBD: validate against current ReSpeaker firmware`.

### Body yaw and IK
`automatic_body_yaw=False` recommended — the body is meant to deliberately not co-vibrate with the pitch. The alarm is a head reaction with the body fully extended.

## Implementation notes
- Preferred as a `Move` subclass with parametrisable `duration` — default 2.0 s, but configurable to 5–10 s for longer alarm phases (e.g. CO alarm).
- LED control via `mini.media.audio.*` and the LED registers from `audio_control_utils` — synchronicity with pitch vibration via shared phase.
- Pitch vibration: 10 Hz × ±5° = 200°/s peak ≈ 3.5 rad/s, under the 8 rad/s limit. Higher frequency or amplitude would exceed the limit.
- Phase 1 (snap) deliberately very short (0.10 s) and `LINEAR` — the hardness is the point.
- `automatic_body_yaw` must be explicitly set to False before behavior start.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "alarm" / "warning" / "attention!" (at least 4 out of 5)
- [ ] Total duration is 2.0 ± 0.2 s in default mode
- [ ] The pitch vibration in phase 2 is recognisable as a pulsing beat (10 Hz)
- [ ] The LED ring blinks red in sync
- [ ] Audio (if enabled) is markedly louder than other behaviors
- [ ] Body yaw stays at 0° throughout the entire behavior
- [ ] The behavior can be ended via `cancel_move()` at any time, which immediately resets the LED effect

## Anti-patterns
- `MIN_JERK` snap in phase 1 — the alarm fails to feel urgent
- Pitch-vibration frequency < 6 Hz — reads as weak anger, not alarm
- LED pulsation out of sync with pitch vibration — feels decoupled
- `automatic_body_yaw=True` — body co-vibrates, the alarm is blurred
- Quiet audio — defeats the warning function
- Hold phase without LED sync — the multi-channel cue (motion + LED + audio) is diagnostic

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Which LED-effect register pattern produces the best red blink? `> ⚠ TBD: validate against current ReSpeaker firmware`.
- Should there be severity tiers (`alarm-warning`, `alarm-critical`) with different durations and frequencies?
- How is the alarm extended for longer events without overstressing the pitch joint? Proposal: max duration 10 s, then cooldown.
- Should the behavior automatically use quieter audio in a calm environment (e.g. at night)? Context-dependent via HA sensor.
