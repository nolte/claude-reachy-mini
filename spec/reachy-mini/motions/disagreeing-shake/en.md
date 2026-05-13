# Motion Sequence: Disagreeing / Head Shake (`disagreeing-shake`)

Status: draft

## Context
The canonical disagreement gesture that reads as "no" or "not OK": Reachy shakes the head two to three times left-right, with a slightly downward pitch ("sceptical"). Use cases: rejection of a voice command, "no, that won't work", a negative reply, detected safety violation.

## Characteristics
- Clear yaw alternation on the head axis — three switches (left–right–left), with decreasing amplitude
- Pitch slightly negative (-5°) throughout the sequence — conveys a sceptical, slightly turned-away attitude
- Antennas slightly lowered, stationary
- Body yaw does **not** track the head — canonically in a head shake the body stays still, only the head negates
- Short tempo (~1.8 s); `EASE_IN_OUT` for the yaw switches (organic back-and-forth)

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (scepticism) | 0.15 | (0, 0, 0, 0, -5, 0) | (-5, -5) | 0 | `MIN_JERK` | small pitch down — defensive prep |
| 2 | Yaw left 1 (large) | 0.22 | (0, 0, 0, 0, -5, -22) | (-5, -5) | 0 | `EASE_IN_OUT` | first and strongest swing left |
| 3 | Yaw right 1 (large) | 0.22 | (0, 0, 0, 0, -5, +22) | (-5, -5) | 0 | `EASE_IN_OUT` | matched right swing |
| 4 | Yaw left 2 (medium) | 0.20 | (0, 0, 0, 0, -5, -16) | (-5, -5) | 0 | `EASE_IN_OUT` | second swing, weaker |
| 5 | Yaw right 2 (medium) | 0.20 | (0, 0, 0, 0, -5, +16) | (-5, -5) | 0 | `EASE_IN_OUT` | mirrored, weaker |
| 6 | Yaw left 3 (subtle) | 0.18 | (0, 0, 0, 0, -5, -8) | (-5, -5) | 0 | `EASE_IN_OUT` | third swing, clearly smaller |
| 7 | Center | 0.15 | (0, 0, 0, 0, -5, 0) | (-5, -5) | 0 | `MIN_JERK` | centre on 0° yaw, pitch still slightly down |
| 8 | Release | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | softly to the neutral pose |

Total duration ≈ 1.72 s.

### Audio (optional)
A short "mm-mm" sample (≤ 350 ms, with two clear beats like a negation), started at phase 2. Volume moderate (40). Not exclamatory.

### Idle modulation
No idle modulation — the motion is compact and decisive, any modulation would dilute it.

### Body yaw and IK
**Important**: `automatic_body_yaw=False` for this behavior. The whole point of the head shake is that the body stays still while only the head negates. With `automatic_body_yaw=True` the body would track the head — that blurs the affect. This is the only behavior in the catalogue where we deviate from the "auto body yaw" default; explicitly set state before and after the behavior.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 1.72`, eight phases.
- Before behavior start: `mini.set_automatic_body_yaw(False)`. At behavior end (or in the `stop` hook): restore to the previous state.
- The decreasing yaw amplitude (22° → 22° → 16° → 16° → 8°) is diagnostic; equal amplitude reads as mechanical.
- Yaw switch from -22° to +22° in 0.22 s = 200°/s ≈ 3.5 rad/s, well inside the 8 rad/s joint limit.
- Pitch -5° constant throughout the sequence (except phase 8) — set on every frame in `evaluate(t)`.
- Roll = 0° is mandatory — even a slight roll turns the shake into a wondering gesture.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "no" / "rejecting" / "disagreeing" (at least 4 out of 5)
- [ ] Total duration is 1.7 ± 0.2 s
- [ ] Three visible yaw swings, with clearly decreasing amplitude
- [ ] Body yaw stays at 0° throughout the entire behavior — `automatic_body_yaw` is actively `False`
- [ ] Pitch holds -5° in phases 1–7 — slightly sceptical attitude
- [ ] Roll stays strictly at 0°
- [ ] Antennas do not move (apart from the single lowering in phase 1)
- [ ] Audio (if enabled) has two clear beats, matching the "mm-mm"

## Anti-patterns
- Leaving `automatic_body_yaw=True` — body yaw passively rotates along, the affect disappears
- Introducing a roll component — turns into `confused`
- Positive pitch during the shake — defeats the sceptical attitude
- More than three yaw swings — reads as frantic rejection or dance
- `LINEAR` easing in the yaw phases — feels mechanical like a metronome
- Audio with an exclamatory "NO!" — clashes with the calm scepticism

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Plugin references

- Pose values, joint limits, canonical poses (INIT/SLEEP) → [`reachy-mini/motor-positions`](../../motor-positions/en.md)
- Pose composition, IK-vs-mechanical-safety, pitch bleed on roll / heave-up → [`reachy-mini/control-surface`](../../control-surface/en.md) §"Mechanical and electrical limitations"
- Motion contains roll or heave-up components? Compensate pitch explicitly in the target pose (Stewart geometry coupling, live-verified 2026-05-13: roll +25° → −3.8° pitch; z +15 mm → +2.4° pitch)

## Open Questions
- Should there always be three swings, or two depending on context? Two is more concise, three is more emphatic.
- Which audio file fits? Proposal: a neutral "mm-mm" with a falling tone.
- How does the behavior react when the caller already set `automatic_body_yaw=False` beforehand? Do not change anything in that case and do not restore at the end — respect caller state.
- Should an optional, very small final yaw-centre wobble (phase 7 with a micro modulation) round off the read?
