# Motion Sequence: Waiting Idle (`waiting-idle`)

Status: draft

## Context
The default idle state between other behaviors: no affective content, only a subtle breathing modulation so Reachy does not look "frozen". Use cases: default state between triggers, calm pause after a finished behavior, demo pause. Implements the "idle breathing" pattern from `control-surface`.

## Characteristics
- **Loopable**: runs continuously until an event triggers another behavior
- Pose is the neutral pose with very light breathing modulation
- Z and pitch modulation in phase, frequency 0.25 Hz (a 4 s breath cycle) — like calm human breathing
- Antennas breathe along (very small amplitude)
- Body yaw stationary
- Very slow tempo, very small amplitude — the pose must NOT read as an affect

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Entry (into idle) | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | transition from a previous behavior into idle |
| 2 | Breath cycle (loop) | 4.0 / cycle | (0, 0, 0 (±2 mm), 0, 0 (±1°), 0) | (0 (±2°), 0 (±2°)) | 0 | `MIN_JERK` (idle mod) | sinusoidal breathing: one cycle every 4 s |
| 3 | Exit | 0.40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | smooth handover to follow-up behavior |

Loop-body cycle = 4.0 s. Minimum run (one cycle plus entry + exit) ≈ 4.8 s. In practice the behavior loops indefinitely.

### Audio
**No audio** — idle is silent.

### Idle modulation during phase 2
**Sinusoidal modulation**:
- `z = 2 * sin(2*pi*0.25*t)` — amplitude 2 mm, frequency 0.25 Hz
- `pitch = 1 * sin(2*pi*0.25*t)` — amplitude 1°, in phase with z
- `antenna_left = 2 * sin(2*pi*0.25*t)` — amplitude 2°, in phase
- `antenna_right = 2 * sin(2*pi*0.25*t)` — amplitude 2°, in phase

All modulations are in phase — that is the breath. Roll, yaw, and body yaw stay strictly at 0°.

### Body yaw and IK
Either — body yaw stays a constant 0°.

## Implementation notes
- Preferred as a `Move` subclass with `duration=None` (unbounded runtime). Ended externally via `cancel_move()` when another behavior triggers.
- `evaluate(t)` computes the sinusoidal modulations modulo loop duration (4.0 s) — the breath is continuous with no loop seam.
- Velocity at z = 2 mm × 2π × 0.25 = ~3.1 mm/s — tiny, well below all limits.
- Amplitude must not be raised without losing the "affect-free" character. ±2 mm and ±1° are maxima.
- If the plugin should not hold a constant command link (power saving on Wireless), `waiting-idle` can also be implemented as "periodic breathing every ~10 s" instead of continuous.

## Acceptance Criteria
- [ ] External observers read the motion as "calm" / "alive" / "waiting" (at least 4 out of 5) — not as a specific emotion
- [ ] Breath modulation is visible but subtle enough not to distract
- [ ] Behavior loops indefinitely without a visible seam between cycles
- [ ] Behavior exits cleanly on `cancel_move()` without jumps
- [ ] Roll, yaw, and body yaw stay strictly at 0°
- [ ] No audio
- [ ] Handover to an active behavior (e.g. `alert-listening`) is seamless

## Anti-patterns
- Breath frequency > 0.5 Hz — feels nervous, not calm
- Breath amplitude > ±3 mm or ±2° on pitch — reads as an affective behavior
- Adding roll or yaw modulation — defeats the affect-free idle character
- Static antennas without modulation — feels frozen
- Any audio — idle must be silent

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Should the breath frequency drift slowly (e.g. 0.2–0.3 Hz instead of a fixed 0.25 Hz) so the idle does not feel mechanical? "Timing variation" pattern from `control-surface`.
- How does `waiting-idle` integrate with `mini.disable_motors()` for power saving? Proposal: after 5 min idle, automatically transition into `goto_sleep()`.
- Should breath frequency be reduced in very cool ambient conditions to save servo heat?
