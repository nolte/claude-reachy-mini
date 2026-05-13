# Motion Sequence: Groove Bob (`groove-bob`)

Status: draft

## Context
A dance building block for rhythmic up-and-down on the music's beat: Reachy "bobs" to the music. Use cases: dance app (the plugin's main use case), reactive motion on beat detection, background animation during audio playback.

## Characteristics
- **BPM-parametrised**: one beat cycle lasts `60/BPM` seconds; one cycle = one down + one up
- Pitch main motion: -8° (down) to +5° (up) — small nod on the beat
- Z translation synchronous: -3 mm (down) to +2 mm (up) — slight ducking
- Antennas slightly perked (+15°), stationary
- Body yaw centred
- **Loopable**: runs as long as the music (n beats), then the outro

## Platform profile

Motion specification applies on all three platforms — **Reachy Mini** (Wireless), **Reachy Mini Lite**, and **Simulation**. The actuator set (Stewart-platform head, two antennas, body yaw) is identical on Wireless and Lite. In simulation every pose and antenna command runs without real motors; any audio portion of this sequence is skipped in simulation without failing the behavior as a whole.

## Components

### Per-beat actuator sequence

Pose values are offsets from the neutral pose. One beat = `T = 60 / BPM` seconds (e.g. 0.5 s at 120 BPM, 0.75 s at 80 BPM).

| Subphase | Duration (relative to T) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennas (l°, r°) | Body yaw (°) | Easing |
|---|---|---|---|---|---|
| Down (beat hit) | 0.40 × T | (0, 0, -3, 0, -8, 0) | (+15, +15) | 0 | `MIN_JERK` |
| Up (off-beat) | 0.60 × T | (0, 0, +2, 0, +5, 0) | (+15, +15) | 0 | `MIN_JERK` |

**Entry** (before the first beat): 0.30 s — from the neutral pose to the "up" pose, in preparation.
**Exit** (after the last beat): 0.40 s — back to the neutral pose.

Minimum run on 1 beat at 120 BPM: 0.30 + 0.5 + 0.40 = 1.20 s.

### Audio
No own audio — the beat comes from the external music source. `groove-bob` reacts to the beat, it does not generate it.

### Idle modulation
No additional modulation — the beat itself is the motion.

### Body yaw and IK
`automatic_body_yaw=True` recommended but has no visible effect since yaw and body yaw stay at 0°.

## Implementation notes
- Preferred as a `Move` subclass with constructor parameters: `GrooveBob(bpm: float, beats: int, lead_time_s: float = 0.0)`. `lead_time_s` allows a phase shift to compensate beat-detection latency.
- BPM range: 60–180. Below 60 the bob feels too slow; above 180 the pitch velocity approaches the joint limit (160 BPM × 13° / 0.4 = ~520°/s ≈ 9 rad/s — close to 8 rad/s; at 180 BPM the down phase is only 0.33×T = 0.11 s and 13° in 0.11 s = 118°/s → 2 rad/s; compute per BPM concretely).
- Beat-detection latency: typically 50–100 ms (audio buffer + detection); compensate via `lead_time_s`.
- `MIN_JERK` easing makes the bob organic; `LINEAR` would feel mechanical, `CARTOON` would distort the beat.
- Synchronicity to the music is critical: the down-beat must land within ±20 ms of the audio beat, otherwise it feels decoupled.

## Acceptance Criteria
- [ ] External observers read the motion without a prompt as "on the beat" / "bobbing along" / "groove" (at least 4 out of 5)
- [ ] At BPM 120, the down-beat is within ±20 ms of the audio beat
- [ ] Pitch swing from +5° to -8° is visible as a "nod"
- [ ] Z modulation is visible but secondary to the pitch motion
- [ ] Behavior loops seamlessly across multiple beats
- [ ] Antennas remain stationary — no antenna bob
- [ ] On `cancel_move()` mid-beat, the behavior settles cleanly into the up-pose and then the neutral pose

## Anti-patterns
- Pitch amplitude > ±15° — feels exaggerated, not groovy
- Antennas bobbing along — defeats the clear pitch-only character
- `LINEAR` or `CARTOON` easing — wrong dance read
- Beat latency > 30 ms — dance feels decoupled
- BPM > 180 — exceeds realistic servo performance and feels panicked
- No lead-time compensation when the audio pipeline has high latency

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
- Which default BPM for unknown music? Proposal: 100 BPM (mid pop dance tempo).
- At very high BPM (≥ 160), should the system automatically switch to half-time bob (every 2 beats)?
- How does `groove-bob` react to a BPM change mid-song? Proposal: seamless phase realignment at the next down-beat.
- Which beat-detection method is assumed? The `audio-beat-tracking` skill (planned) provides it.
