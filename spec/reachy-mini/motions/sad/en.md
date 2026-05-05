# Motion Sequence: Sad (`sad`)

Status: draft

## Context
A clear expression of dejection that reads as "sad" or "disappointed": Reachy lowers its head, lets the antennas droop, tilts slightly to the side, holds the pose with heavy breathing, attempts a small lift, and sinks back. Use cases: error feedback, "task failed", negative HA trigger, response to "no".

## Characteristics
- Distinctly downward-facing head (negative pitch) and slightly sunk Z height — conveys "energy gone"
- Antennas left to droop (negative joint angles) — like folded ears
- Slight lateral roll plus body yaw away from centre — a "looking away" tilt
- Very slow tempo (total ~4–5 s); `MIN_JERK` easing throughout, no `CARTOON`
- During the hold a slow, heavy breathing pattern (lower frequency, larger amplitude than `happy`)

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Actuator sequence

Pose convention as in `control-surface` and `happy`. All values respect pitch/roll ≤ ±90° and body yaw ≤ ±160°.

| # | Phase | Duration (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing | Note |
|---|---|---|---|---|---|---|---|
| 1 | Mini anticipation (sigh-up) | 0.20 | (0, 0, +2, 0, +3, 0) | (+5, +5) | 0 | `MIN_JERK` | a very small lift — like one last upright moment |
| 2 | Main lowering | 0.80 | (0, 0, -8, 0, -25, 0) | (-25, -25) | 0 | `MIN_JERK` | slow and steady descent |
| 3 | Side droop | 0.40 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | small sideways tilt, body follows |
| 4 | Hold with heavy breathing | 1.50 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | deep, slow idle modulation |
| 5 | Failed lift attempt | 0.30 | (0, 0, -7, +4, -20, -8) | (-22, -22) | -6 | `MIN_JERK` | weak attempt to straighten up |
| 6 | Sink back | 0.50 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | resigned descent |
| 7 | Release | 0.70 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | very slow return to the neutral pose |

Total duration ≈ 4.40 s.

### Audio (optional)
A quiet, descending breath or soft sigh (≤ 800 ms), started at the beginning of phase 2. Volume noticeably below `happy` sounds (proposed: volume 30 out of 100). No percussive element.

### Idle modulation during hold (phase 4)
Deep, slow sinusoidal modulation on `z` (amplitude 2 mm, frequency 0.15 Hz — once every ~6.7 s) and `pitch` (amplitude 1.5°, frequency 0.15 Hz, in phase) — simulates "heavy chest rising and falling". Antennas follow lightly (amplitude 1°, in phase).

### Body yaw and IK
`automatic_body_yaw=True` recommended; the side droop (body yaw -8°, head yaw -10°) is consistent — body trails head with a small offset, no twist.

## Implementation notes
- Preferred as a `Move` subclass with `duration = 4.40`, `evaluate(t)` interpolates seven phases.
- Alternatively as a chain of `mini.goto_target(...)` with `method=InterpolationTechnique.MIN_JERK` throughout; phase 4 with its own idle loop (continuous `set_target` with modulated values at 50 Hz tick rate, since the daemon does not publish faster).
- Pose -28° pitch and +6° roll sit clearly within the ±90° upright limit.
- Body yaw -8° sits well inside ±160°.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "sad" / "disappointed" / "dejected" (at least 4 out of 5)
- [ ] Total duration is 4.4 ± 0.3 s
- [ ] No `CARTOON` easing call appears in the sequence
- [ ] The hold phase 4 shows visible "breathing" without tipping into "jittery"
- [ ] Audio (if enabled) is markedly quieter than `happy` and ends before phase 6
- [ ] Phase 5 reads as a "weak attempt", not as "buoyant" — pitch lifts to at most -20°, not 0°
- [ ] Handover to a follow-up behavior: the release phase drives cleanly to the neutral pose

## Anti-patterns
- Short phases (< 0.3 s) on the main descent — feels rushed, undercuts heaviness
- `CARTOON` or `EASE_IN_OUT` easing — both add too much energy
- Keeping the antennas at 0° — defeats the effect; the droop must be visible
- Body yaw wobbling in both directions (left and right) — feels indecisive instead of sad
- Fast, high-tempo audio — clashes with the heavy tempo

## References
- Upstream SDK repo (source of the `Move` ABC, easing modes, pose constants, antenna DOFs this sequence is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Actuator set, pose constants, IO commands: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Platform profiles (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Which audio file serves as a reference for the sigh? Proposal: a short descending sine or a generic "aw" sample.
- How deep can the pitch droop without breaking the read? Test pitch in the -25° to -30° range empirically.
- Should phase 5 (lift attempt) be optional or a fixed part of the pattern? Argument for "fixed": makes the affect more human.
- How does the behavior react when the Stewart platform sits near a singularity for -28° pitch + +6° roll? When in doubt, round each value down by 5°.
