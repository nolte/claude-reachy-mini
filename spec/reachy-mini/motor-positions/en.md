# Reachy Mini motor positions, limits, and canonical poses

Status: draft

## Context

Anyone moving a Reachy Mini needs two kinds of answers that today are scattered across docs or not answered at all: *which position is each individual motor allowed to reach?* and *which combinations of those positions are actually valid — physically reachable, mechanically safe, solvable by the SDK IK?* The [`reachy-mini/control-surface`](../control-surface/en.md) spec answers the first question at a high level (inventory, nominal operations range) and characterises the Stewart joint limits only as "asymmetric per joint". Concrete per-motor values, canonical rest poses with their exact joint vectors, and the combinatorial conflict shapes are intentionally out of its scope. This spec fills that gap.

It is consumed by the [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) skill when snippets are validated against limits, by [`dance-choreography`](../../claude/dance-choreography/en.md) when composing extreme poses, by [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/en.md) when sanity-checking state reads, and by any future `app-scaffold` template as the source of truth for pose defaults. Values are verified as of 2026-05-12 against a running Wireless daemon and the Pollen SDK source on [`main`](https://github.com/pollen-robotics/reachy_mini).

One headline finding up front: the daemon reports `engine: AnalyticalKinematics, collision check: false`. The SDK validates a pose **only** against the inverse-kinematics polytope, not against self-collision, antenna mechanical end-stops, or cable harness clearance. A pose the IK accepts is *kinematically reachable* — not automatically *mechanically safe*. The spec keeps that distinction explicit throughout.

## Goals

- For every motor / joint, document the exact values from the live URDF — lower limit, upper limit, velocity, effort — with a clear note that the URDF (mechanical limit) takes precedence over `kinematics_data.json` (software limit ±π)
- Mirror the canonical poses from the SDK source verbatim (`INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS`, `SLEEP_HEAD_POSE`, `SLEEP_ANTENNAS_JOINT_POSITIONS`, plus the hard-coded joint vectors for each), including the `wake_up` and `goto_sleep` trajectories
- Separate the three layers of validity cleanly: joint limit (URDF), kinematic reachability (IK polytope), mechanical safety (empirical)
- Name the known conflict shapes that follow from the Stewart-platform geometry so a composition can flag them as "IK-unsolvable" early, without consulting the device
- Tag every value with date and source so a later URDF or SDK change surfaces immediately in a follow-up audit

## Non-Goals

- Motion composition, easing profiles, anticipation / follow-through — belongs to [`reachy-mini/control-surface`](../control-surface/en.md) §"Motion design"
- Concrete choreographies — belongs to [`reachy-mini/motions/`](../motions/) (one file per motion) and to the [`dance-choreography`](../../claude/dance-choreography/en.md) skill
- Implementation guidance for a skill or agent — this spec is a normative knowledge base, not an operations recipe
- Audio, vision, or LED control — other subsystems
- A self-collision algorithm or URDF-based mesh check — explicitly absent from the SDK; until that is retrofitted, the spec stays at "known risks" enumeration
- Hardware bring-up, calibration, firmware flash — separate skills (planned)
- Simulation-only values when they diverge from hardware — the sim shares the URDF limits, but electromechanical behaviour (effort, velocity under load) is sim-irrelevant

## Requirements

### Layer 1 — joint limits per motor (from the live URDF)

Values fetched on 2026-05-12 from `http://reachy-mini.local:8000/api/kinematics/urdf`, generated from [`src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf) (onshape-to-robot pipeline).

#### Active joints (steerable, mechanical limits)

| Joint | Type | Min (rad) | Max (rad) | Min (deg) | Max (deg) | Velocity (rad/s) | Effort (N·m) | Motor |
|---|---|---|---|---|---|---|---|---|
| `stewart_1` | revolute | −0.8378 | +1.3963 | **−48°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_2` | revolute | −1.3963 | +1.2217 | **−80°** | **+70°** | 8 | 10 | XL330-M288-T |
| `stewart_3` | revolute | −0.8378 | +1.3963 | **−48°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_4` | revolute | −1.3963 | +0.8378 | **−80°** | **+48°** | 8 | 10 | XL330-M288-T |
| `stewart_5` | revolute | −1.2217 | +1.3963 | **−70°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_6` | revolute | −1.3963 | +0.8378 | **−80°** | **+48°** | 8 | 10 | XL330-M288-T |
| `right_antenna` | revolute | −π | +π | −180° | +180° | 8 | 10 | XL330-M077-T |
| `left_antenna` | revolute | −π | +π | −180° | +180° | 8 | 10 | XL330-M077-T |
| `yaw_body` | revolute | −2.7925 | +2.7925 | **−160°** | **+160°** | 8 | 10 | XC330-M288-PG (custom) |

#### Stewart asymmetry pattern

The six Stewart actuators are arranged in mirrored pairs. That produces a strict asymmetry pattern that the `control-surface` spec only hinted at:

| Pair | Joints | Lower–Upper (deg) | Interpretation |
|---|---|---|---|
| **A** | `stewart_1`, `stewart_3` | **−48° / +80°** | More travel "up/inward", less "down/outward" |
| **B** | `stewart_4`, `stewart_6` | **−80° / +48°** | Mirror image of pair A |
| **C-1** | `stewart_2` | **−80° / +70°** | Nearly symmetric, slightly biased downward |
| **C-2** | `stewart_5` | **−70° / +80°** | Mirror image of `stewart_2` |

Consequence: a head pose that drives stewart_1 to +80° also taxes stewart_4 in the same direction; because stewart_4 can only go to +48° there, the symmetric maximum head deflection toward one side is **tighter** than toward the other. The effective pitch/roll range therefore depends on the yaw direction.

#### Software limit (`kinematics_data.json`) vs. mechanical limit (URDF)

The [`assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json) file lists `limits: [-π, +π]` for every Stewart motor (i.e. ±180°). That is **not** the mechanical limit — it is a software default bound on the IK solver. **The URDF is authoritative.** Anyone validating a value against limits uses the URDF table above, not the JSON.

#### Passive joints (kinematically required, not steerable)

The URDF additionally declares 21 passive revolute joints (`passive_1_x/y/z` … `passive_7_x/y/z`) with limits `±π`, velocity `1e+08` (effectively unbounded), and effort `10` N·m. Those are the ball joints of the Stewart attachment, required to close the kinematic loops. They never appear in the REST API or the Python SDK as a steerable quantity — they are set implicitly by the IK and are listed here only for completeness.

### Layer 2 — inverse kinematics and workspace

Engine: `AnalyticalKinematics` (Rust core with Python bindings, source: [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)). Live identification: `GET /api/kinematics/info` returns `{"engine":"AnalyticalKinematics","collision check":false}`.

#### Kinematic parameters

From [`assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json):

| Parameter | Value | Meaning |
|---|---|---|
| `motor_arm_length` | 40 mm | Motor horn lever length |
| `rod_length` | 85 mm | Rigid rod motor-horn → platform |
| `head_z_offset` | 177 mm | Vertical offset from the Stewart base frame to the head frame; the IK adds this to any z-component internally |

#### IK-internal safety thresholds

From `analytical_kinematics.py`, in the `inverse_kinematics_safe` path (active when `automatic_body_yaw=True`):

| Constant | Value | Meaning |
|---|---|---|
| `max_relative_yaw` | `np.deg2rad(65)` = **±65°** | Maximum yaw angle of the head frame *relative* to the body yaw; if a target pose exceeds it, the body yaw is rotated along |
| `max_body_yaw` | `np.deg2rad(160)` = **±160°** | Maximum body yaw; identical to the URDF limit on `yaw_body` |

In the `automatic_body_yaw=False` path, the body yaw is set by the caller and the IK fails when the Stewart solution falls outside the URDF polytope — no auto-adjustment.

#### Workspace polytope

The reachable head-pose space is the set of all 4×4 transforms whose inverse-kinematics Stewart solution falls inside the URDF limits. There is no closed-form expression because the six Stewart limits are asymmetric and the solver is an analytic Stewart inversion. Empirically — backed by the `control-surface` doc and the Pollen datasheet table (`platforms/reachy_mini/hardware`) — the following counts as the **nominal operations range**:

| Axis | Min | Max | Source |
|---|---|---|---|
| Head roll (Rx) | −40° | +40° | Pollen `dof_table.png` |
| Head pitch (Ry) | −40° | +40° | Pollen `dof_table.png` |
| Head yaw (Rz, relative to body) | −60° | +60° | Pollen `dof_table.png` (tighter), `max_relative_yaw` IK-internal +65° |
| Body yaw (Rz) | −155° | +155° | Pollen `dof_table.png` (tighter), URDF +160° |
| Right antenna (R) | −180° | +180° | URDF |
| Left antenna (R) | −180° | +180° | URDF |
| Head translation x | ≈ −20 … +20 mm | ⚠ TBD: measure on the real device | IK polytope |
| Head translation y | ≈ −20 … +20 mm | ⚠ TBD: measure on the real device | IK polytope |
| Head translation z | ≈ −45 … +45 mm relative to head_z_offset | ⚠ TBD | IK polytope |

The translation ranges are extrapolated from the SLEEP-pose values (see Layer 3) and are **not** verified; a follow-up audit on hardware can pin them.

#### IK bisection — measured polytope boundaries (Phase A)

Bisection sweep against `AnalyticalKinematics.ik(...)` run locally with `reachy_mini==1.7.2` (2026-05-12). Bisection ε = 1e-4, stop criterion: the IK rejects **or** at least one Stewart joint crosses the URDF limit.

| Axis | Δ_max (IK + URDF-conformant) | limit-triggering joint | s1 (°) | s2 (°) | s3 (°) | s4 (°) | s5 (°) | s6 (°) | body (°) |
|---|---|---|---|---|---|---|---|---|---|
| `tx_pos` | +50.88 mm | s3, s4 (±80°) | +25.85 | −77.09 | +79.95 | −79.95 | +77.09 | −25.85 | 0 |
| `tx_neg` | −46.78 mm | s1, s6 (±80°) | +79.91 | −32.10 | +50.05 | −50.05 | +32.10 | −79.91 | 0 |
| `ty_pos` | +47.56 mm | s5 (+80°) | +64.25 | −35.50 | +23.69 | −78.49 | +79.99 | −50.45 | 0 |
| `ty_neg` | −47.56 mm | s2 (−80°) | +50.45 | −79.99 | +78.49 | −23.69 | +35.50 | −64.25 | 0 |
| **`tz_pos`** (head fully up) | **+23.05 mm** | all six simultaneously | +79.70 | −79.70 | +79.70 | −79.70 | +79.70 | −79.70 | 0 |
| `tz_neg` (head fully down) | −50.78 mm | s1, s3 (−48°) + s4, s6 (+48°) | −47.83 | +47.83 | −47.83 | +47.83 | −47.83 | +47.83 | 0 |
| `roll_pos` | +47.71° | s2 (−80°) | +60.44 | −80.00 | +44.34 | −30.67 | +14.01 | −13.90 | 0 |
| `roll_neg` | −47.71° | s5 (+80°) | +13.90 | −14.01 | +30.67 | −44.34 | +80.00 | −60.44 | 0 |
| `pitch_pos` (head tilted up) | **+48.01°** | s3, s4 (±80°) | +21.26 | −26.33 | +80.00 | −80.00 | +26.33 | −21.26 | 0 |
| `pitch_neg` (head tilted forward/down) | **−72.43°** | s1 (+80°), s6 (−80°) | +80.00 | −45.01 | +6.55 | −6.55 | +45.01 | −80.00 | 0 |
| `yaw_pos` | +90° (probe cap) | none — IK accepts more | +62.57 | −33.89 | +62.57 | −33.89 | +62.57 | −33.89 | +25.00 |
| `yaw_neg` | −90° (probe cap) | none | +33.89 | −62.57 | +33.89 | −62.57 | +33.89 | −62.57 | −25.00 |

Observations:

- **Pitch is markedly asymmetric**: +48° ("head up") versus −72° ("head forward"). That reflects the mirrored-pair Stewart actuator arrangement.
- **Max heave ("head fully up") = +23.05 mm** with all six Stewart joints at their own ±80° limit, pair-wise anti-symmetric. Heave downward reaches significantly further (−50.78 mm) thanks to the opposite limit asymmetry.
- **Yaw stays unconstrained by URDF**: the IK accepts head-yaw past the probe cap of 90°; the theoretical maximum is `max_relative_yaw + max_body_yaw` = 65° + 160° = **225°**.

> **⚠ WARNING — these values are NOT mechanically safe.** They are the mathematical polytope boundary of the analytical IK *while respecting the URDF limits*. A real motion to the IK boundary can trigger self-collision — see Layer 4 §"Phase-B live incident 2026-05-12". The binding range for composition is the **Pollen nominal operations range** (±40° pitch/roll), not this table. This table documents *what the IK would accept*, not *what the hardware can take*.

#### Three layers of validity — order of checking

A behavior composition **MUST** check in this order, from outside in:

1. **IK polytope** (software default, mathematical). Accepts even poses outside the URDF limits because the solver only knows the `kinematics_data.json` limits (±π). **Not** enough as a safety gate on its own.
2. **URDF mechanical limits** (Layer 1). These describe the real motor travel. A pose that drives a Stewart joint past this limit gets clipped by the motor — or, as the `set_mode/enabled` path showed, silently ignored by the `goto` API.
3. **Pollen nominal operations range** (Pollen datasheet, ±40° pitch/roll, ±60° head-yaw, ±155° body-yaw). The *recommended* range in which the hardware operates reliably, without self-collision and without mechanical stress. **This is binding for every motion composition.**

Layer 3 ⊊ Layer 2 ⊊ Layer 1. Stopping at Layer 1 leaves no safety statement. Stopping at Layer 2 leaves no self-collision guarantee. **Only Layer 3 is binding.**

#### T1–T8 live verification 2026-05-13

Eight target poses inside the Pollen nominal range with a 5–15° / 8–15 mm safety margin (test set from Layer 6 §"Test set"), driven on a Reachy Wireless v1.7.1 under IMU bypass (see Layer 5 §"Stage-3 path 2"). `duration = 6 s` per motion, with a recenter to INIT between every test (also 6 s). Local IK prediction via `analytical_kinematics.ik(target_pose)` from the `reachy_mini==1.7.2` package; live `head_joints` via `GET /api/state/full?with_head_joints=true`.

| Test | Target | Real pose component | Pose diff (norm) | IK-pred Stewart (°) | Real Stewart (°) |
|---|---|---|---|---|---|
| **T1** pitch +30° | `pitch = +0.5236` rad | pitch **+31.1°** (+1.1° over) | 0.049 | s1 +24.5, s2 −29.8, s3 +58.6, s4 −58.6, s5 +29.8, s6 −24.5 | s1 +24.3, s2 −30.1, s3 +58.4, s4 −58.6, s5 **+26.9**, s6 −24.3 |
| **T2** pitch −30° | `pitch = −0.5236` rad | pitch **−29.1°** (+0.9° short) | 0.037 | s1 +52.2, s2 −41.2, s3 +17.8, s4 −17.8, s5 +41.2, s6 −52.2 | s1 +52.1, s2 −39.9, s3 +16.9, s4 −16.8, s5 **+37.8**, s6 −52.0 |
| **T3** roll +25° | `roll = +0.4363` rad | roll **+26.8°**, pitch-bleed **−3.8°** | 0.074 | s1 +48.6, s2 −54.8, s3 +40.3, s4 −32.4, s5 +21.6, s6 −23.6 | s1 +48.7, s2 −55.0, s3 +35.7, s4 −28.8, s5 +21.3, s6 −21.2 |
| **T4** roll −25° | `roll = −0.4363` rad | roll **−26.9°**, pitch-bleed −0.9° | 0.040 | s1 +23.6, s2 −21.6, s3 +32.4, s4 −40.3, s5 +54.8, s6 −48.6 | s1 +21.5, s2 −19.9, s3 +32.3, s4 −35.0, s5 +51.7, s6 −48.8 |
| **T5** heave +15 mm | `z = +0.015` m | z **+12.8 mm** (−2.2 mm), pitch-bleed **+2.4°** | 0.043 | all ±58.6° (antisymmetric) | s1 +58.2, s2 −51.2, s3 +56.1, s4 −56.3, s5 +51.6, s6 −58.4 |
| **T6** heave −35 mm | `z = −0.035` m | z **−35.0 mm** (≈ exact) | **0.006** | all ±11.6° (antisymmetric) | s1 −11.9, s2 +11.3, s3 −11.7, s4 +12.0, s5 −11.3, s6 +11.7 |
| **T7** head-yaw +45° | `yaw = +0.7854` rad | yaw **+44.3°** | 0.029 | s1 +53.5, s2 −30.3, s3 +53.5, s4 −30.3, s5 +53.5, s6 −30.3 | s1 +53.5, s2 −29.1, s3 +52.9, s4 −28.7, s5 **+49.5**, s6 −28.7 |
| **T8** body-yaw +90° | `body_yaw = +1.5708` rad | body **+64.3°** (IK clips!) | 0.030 | body **+65.0** (max_relative_yaw clip already in the IK), s1 +33.9, s2 −62.6, s3 +33.9, s4 −62.6, s5 +33.9, s6 −62.6 | body +64.3, s1 +31.6, s2 −60.8, s3 +29.4, s4 −57.7, s5 +29.9, s6 −60.4 |

Findings:

- **IK ↔ real agrees very closely.** T6 (heave −35 mm) has a pose-diff norm of **0.006** — practically byte-exact. The Stewart joint values also agree to within < 0.6° per joint.
- **T1–T7 all stay within a 0.05–0.08 rad pose-diff norm.** The Pollen nominal range with a 5–15° margin is reliably reachable in hardware — no self-collision, no daemon anomaly.
- **T8 surfaces `max_relative_yaw=65°` as a binding constraint:** an API request `head.yaw=0, body_yaw=+90°` would mean a relative yaw of 90° > 65°. Both the local `analytical_kinematics.ik(...)` and the daemon clip at 65°. The discrepancy (target 90° → real 64.3°) is therefore **not a hardware weakness but a spec-conformant IK guard**.
- **Hardware sweet spots:** T6 (heave down) is exact; T7 (head yaw) and T2 (pitch down) come very close to the target. Motions where the Stewart platform works symmetrically are more precise.
- **Pitch bleed on roll and heave-up:** T3 (roll +25°) pulls pitch by −3.8°; T5 (heave +15 mm) pulls pitch by +2.4°. That is a **systematic coupling of the Stewart geometry**, not a calibration drift. Behaviors that need isolated roll or heave motion must compensate pitch explicitly or budget for the bleed size.
- **Init-pose joint values from the real device:** after recovering to INIT, the device reports `[body 0°, s1 +35.5°, s2 −32.1°, s3 +34.5°, s4 −35.2°, s5 +31.8°, s6 −35.2°]`. These are **more symmetric** than the hard-coded `init_positions` from the SDK source `[~0°, +30.1°, −38.3°, +34.8°, −34.8°, +38.3°, −30.1°]` (Layer 3 §"Init pose") — the current IK solution likely differs from the frozen hardcoding. The hard-coded values stay spec-relevant because `goto_sleep` uses them for its distance check, but the actual target joint solution for `INIT_HEAD_POSE` is the one measured here.

#### Yaw split body / head

A rotation "Reachy looks 70° to the left" is automatically decomposed by `inverse_kinematics_safe` into `head_yaw=65°` + `body_yaw=5°`, because `max_relative_yaw=65°` would be exceeded. In `automatic_body_yaw=False` mode, the caller has to provide the split — a head-yaw-only request of 70° fails instead of being compensated by the body.

### Layer 3 — canonical poses from the SDK source

Values taken verbatim from [`src/reachy_mini/reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py) and [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py), state `main` 2026-05-12.

#### Init pose (`wake_up` end-state, neutral reference)

`INIT_HEAD_POSE`:

```
np.eye(4)          # 4×4 identity matrix
# Rotation: none (roll = pitch = yaw = 0)
# Translation (head frame): (0, 0, 0) m
# Effective world translation: (0, 0, head_z_offset) = (0, 0, 0.177) m
```

`INIT_ANTENNAS_JOINT_POSITIONS`:

| Antenna | rad | deg |
|---|---|---|
| `right_antenna` | −0.1745 | **−10°** |
| `left_antenna` | +0.1745 | **+10°** |

Reason from the SDK source: *"~10° offset to reduce shaking at vertical"* — both antennas are deliberately leaned slightly outward to avoid vertical micro-tremor from motor backlash. **A true 0/0 antenna pose is explicitly not the default rest state.**

Hard-coded IK solution for `INIT_HEAD_POSE` (used in the SDK source as fallback when the daemon cannot yet supply a pose):

```
[body_yaw, stewart_1, stewart_2, stewart_3, stewart_4, stewart_5, stewart_6] =
[6.96e-07, +0.5252, −0.6687, +0.6067, −0.6067, +0.6687, −0.5252]    # rad
[~0°,      +30.09°, −38.31°, +34.76°, −34.76°, +38.31°, −30.09°]    # deg
```

Observation: the Stewart vector is **pair-wise anti-symmetric** (s1 vs. s6, s2 vs. s5, s3 vs. s4) — confirming the mirror-pair arrangement of the actuators.

#### Sleep pose (`goto_sleep` end-state)

`SLEEP_HEAD_POSE`:

```
[[ 0.911,  0.004,  0.413, -0.021],
 [-0.004,  1.0,   -0.001,  0.001],
 [-0.413, -0.001,  0.911, -0.044],
 [ 0.0,    0.0,    0.0,    1.0  ]]
```

| Component | Value | Meaning |
|---|---|---|
| Rotation R (extracted) | Pitch ≈ **+24.4°** (xyz-Euler: `arcsin(−R[2,0]) = arcsin(+0.413)`) | Head tipped forward-down (positive pitch in the scipy xyz convention; live-verified 2026-05-13 with `pitch ≈ +26°` real) |
| Translation (head frame) | x = **−21 mm**, y = **+1 mm**, z = **−44 mm** | Head retracted and lowered |
| Effective world z | head_z_offset + z = 0.177 − 0.044 = **0.133 m** | Final sleep platform height |

`SLEEP_ANTENNAS_JOINT_POSITIONS`:

| Antenna | rad | deg | Distance to limit |
|---|---|---|---|
| `right_antenna` | **−3.05** | **−174.7°** | 5.3° short of the ±180° stop |
| `left_antenna` | **+3.05** | **+174.7°** | 5.3° short of the ±180° stop |

So the antennas fold back almost entirely — they sit practically against their mechanical stops. The 5.3° margin is intentional: if encoder drift accumulates during sleep, the antennas do not end up wedged against the stop.

Hard-coded IK solution for `SLEEP_HEAD_POSE`:

```
[body_yaw, stewart_1, stewart_2, stewart_3, stewart_4, stewart_5, stewart_6] =
[0.0, −0.9848, +1.2625, −0.2439, +0.2056, −1.2364, +1.0032]    # rad
[0°,  −56.43°, +72.32°, −13.97°, +11.78°, −70.84°, +57.48°]    # deg
```

Observation: `stewart_2` reaches **+72.32°**, which is only 2.32° **above** its URDF maximum of +70° — **this is a discrepancy** worth attention. ⚠ TBD: confirm whether the SDK constant or the URDF limit needs to be corrected; if the sleep pose sits beyond a kinematic limit, it may fail on some units with slightly different calibration. `stewart_5` shows the same pattern at −70.84°, just past its −70° URDF limit — also TBD.

#### `wake_up` trajectory

From `ReachyMini.wake_up()` (`reachy_mini.py:575`):

1. `goto_target(INIT_HEAD_POSE, antennas=INIT_ANTENNAS_JOINT_POSITIONS, duration=2.0 s)` — from any pose to neutral
2. `time.sleep(0.1)`
3. Sound `wake_up.wav` (a toudoum chime)
4. `goto_target(pose_roll20, duration=0.2 s)` — where `pose_roll20` = `INIT_HEAD_POSE` with roll +20° (xyz euler) — head leans briefly to the left
5. `goto_target(INIT_HEAD_POSE, duration=0.2 s)` — back to the neutral pose

End-state of the `wake_up` sequence is `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS`.

#### `goto_sleep` trajectory

From `ReachyMini.goto_sleep()` (`reachy_mini.py:591`):

1. `get_current_joint_positions()` → distance check against the hard-coded `init_positions` (see above); if `np.linalg.norm > 0.2 rad`, then
   `goto_target(INIT_HEAD_POSE, antennas=INIT_ANTENNAS_JOINT_POSITIONS, duration=1.0 s)` + `time.sleep(0.2)`
2. Sound `go_sleep.wav` (a "pfiou" sigh)
3. `goto_target(SLEEP_HEAD_POSE, antennas=SLEEP_ANTENNAS_JOINT_POSITIONS, duration=2.0 s)`
4. `time.sleep(2)`

Observation: step 1 makes the sleep path **state-dependent**. An app cannot just call `goto_sleep` and rely on a predictable 2-second motion — when the distance is large, the call takes 3.2 s or more.

### Layer 4 — known conflict shapes

The SDK performs **no** collision check (`collision check: false`). The following shapes are risky as best-current-knowledge; they are either accepted by the IK or rejected with a hard-to-read error, but a behavior composition is better off avoiding them up front.

#### IK-error classes

- **Pose outside the Stewart polytope**: extreme pitch + extreme roll simultaneously — e.g. roll +35° combined with pitch +35°. Each within the nominal ±40°, the combination often not IK-solvable. **Consequence**: `goto_target` raises a kinematics exception; on the REST path, `POST /api/move/goto` returns a 4xx status.
- **Translation outside the reachable volume**: head-frame translation z = +50 mm combined with a pitch deflection — the effective Stewart leg length exceeds `motor_arm_length + rod_length`. Same error class.
- **Head yaw without body compensation beyond ±65°**: only in the `automatic_body_yaw=False` path. In the default path, silently compensated by the IK.

#### Mechanical conflicts (not caught by the IK)

- **Antenna crossing**: both antenna joints driven into directions where their tips overlap — for example `right_antenna = +90°` and `left_antenna = −90°`. The antennas can physically touch or scrape the head shell. ⚠ TBD: validate on the device whether there is a safe "crossing-forbidden zone".
- **Antenna stop damage**: keeping an antenna at ±180° against the stop for an extended period, with continued torque demand. The URDF effort limit (10 N·m) does not fully prevent it — the motor keeps trying. **Recommendation**: switch the antenna motors into `disabled` or `gravity_compensation` once a sleep-near pose is reached (mode names see Layer 5); `goto_sleep` currently does not.
- **Sleep pose near a URDF limit**: `SLEEP_HEAD_JOINT_POSITIONS` contains `stewart_2 = +72.32°` and `stewart_5 = −70.84°` — both past their URDF limits (`+70°` and `−70°`). On a nominally calibrated unit the IK is likely to clip or reject. ⚠ TBD: whether the SDK constant is stale (the URDF was tightened after `SLEEP_HEAD_JOINT_POSITIONS` was set) or whether the constant is intentional and the URDF is too conservative.
- **Camera cable in extreme pitch**: ⚠ TBD — the Pollen doc does not call out a cable path explicitly, but camera modules with a USB cable can exceed their travel under aggressive pitch.

#### State conflicts

- **Motor mode change during a running app**: a `POST /api/motors/set_mode/{mode}` from `enabled` to `disabled` or `gravity_compensation` mid-`set_target` stream can interrupt the current pose non-deterministically. Convention: mode switches are rare, deliberate operations — see `reachy-mini/daemon-rest-api` §"Motors". Correct mode names are documented in Layer 5.

#### Phase-B live incident 2026-05-12 (self-collision recorded)

While attempting to verify the polytope boundaries found in Layer 2 §"IK bisection" on the real Wireless, the Reachy ran into self-collisions at the IK-valid pitch poses: the head "slammed hard into its body" (operator observation). Sequence in order:

1. `tz_pos` (target z = +23.05 mm) — Reachy actually reached only ≈ +10 mm; head_pose stayed at pitch +0.005 rad, z ≈ −145 mm (baseline −155 mm)
2. `tz_neg` (target z = −50.78 mm) — Reachy reached ≈ −53 mm; head_pose pitch +0.093 rad, z = −208 mm
3. `pitch_pos` (target pitch = +48°) — read pitch = +17.8°, **30° discrepancy**; the mechanical collision likely started here
4. `pitch_neg` (target pitch = −72°) — read pitch = −15.6°, **56° discrepancy**; further collision

Aftermath:
- `head_pose` stays byte-identical at pitch ≈ +0.75 rad, z ≈ −187 mm despite `POST /api/move/goto INIT` with `duration=6.0` and `motors/set_mode/enabled`
- `head_joints: null` in `/api/state/full` — daemon can no longer read Stewart positions
- `backend_status.ready: false`, `backend_status.last_alive: null` — the motor backend (USB bus to the Dynamixel motors) is offline
- `POST /api/motors/set_mode/gravity_compensation` → **500 Internal Server Error**
- `POST /api/move/goto` → 200 OK with UUID, but **no** motion occurs

Interpretation: the Dynamixel motors most likely self-protected via an overload guard or position error and dropped off the bus. Recovery probably requires a daemon restart (`POST /api/daemon/restart`) or a power-cycle of the Reachy. **A pure REST recovery from this state was not possible on 2026-05-12.**

Lessons (binding):

1. **Never** use IK-polytope boundary values from Layer 2 §"IK bisection" as live targets. Live motions stay within the Pollen nominal operations range (±40° pitch/roll, ±60° head-yaw).
2. For any live sweep, pick `duration` ≥ 5.0 s so the daemon has time to react and the operator has time to abort.
3. Pre-flight: before any motion sequence, check the app-lock state and `backend_status.ready` — if `ready != true`, **do not** start with `goto`.
4. With `backend_status.ready: false` and `head_joints: null` the Reachy is **not** software-recoverable; a physical check plus power-cycle is needed.

### Layer 5 — motor modes and backend health

Correction to the `reachy-mini/control-surface` spec, which named the modes `stiff` / `compliant`: the actual daemon mode names (as of `reachy_mini==1.7.1`, verified live 2026-05-12) are:

| Mode name | API behaviour | Effect |
|---|---|---|
| `enabled` | Default; `set_target` and `goto` are executed | Motors actively hold the setpoint pose ("stiff") |
| `disabled` | Motors released; `goto` is accepted but does not move | Robot is freely movable by hand; gravity pulls the head down |
| `gravity_compensation` | Holds the current pose against gravity without an active setpoint | **Only available with `kinematics_engine=Placo`.** With the default engine `AnalyticalKinematics`, `set_mode/gravity_compensation` returns `500 Internal Server Error` with `RuntimeError: Gravity compensation mode is only supported with the Placo kinematics engine.` (verified via SSH daemon log 2026-05-13, source: `reachy_mini/daemon/backend/robot/backend.py:563`) |

API: `POST /api/motors/set_mode/{mode}` — the path parameter is an enum accepting only those three values; invalid names return **422 Unprocessable Entity**. `GET /api/motors/status` returns `{"mode": "<current_mode>"}`. Check which engine is active with `GET /api/kinematics/info`; with `AnalyticalKinematics`, `gravity_compensation` is practically unusable — recovery triage (see below) **MUST NOT** rely on this mode.

#### Backend status — when the Reachy stops responding

`GET /api/daemon/status` returns a `backend_status` object that describes the motor controller. The most important fields:

| Field | Meaning | Safe value |
|---|---|---|
| `backend_status.ready` | `true` when the motor controller maintains the Dynamixel bus connection | `true` |
| `backend_status.last_alive` | Last heartbeat timestamp | non-null |
| `backend_status.motor_control_mode` | Mirrors `GET /api/motors/status`-mode | `enabled` / `disabled` / `gravity_compensation` |
| `backend_status.control_loop_stats.mean_control_loop_frequency` | Nominal ≈ 50 Hz (measured 49.7 Hz) | > 40 Hz |
| `backend_status.control_loop_stats.nb_error` | Motor-controller error counter | 0 |
| `backend_status.error` | Last daemon-side error string | `null` |

A live `goto` motion **MUST** first check `backend_status.ready == true`. If `false`, `head_joints` in `/api/state/full` is typically `null` and the `goto` API accepts motions silently (200 with UUID) without executing them. In this state, stop live tests immediately and triage via daemon restart or power-cycle.

#### When a daemon restart is enough, when a power-cycle is, and when neither is

Verified on 2026-05-13 after the Phase-B incident, in three stages with negative findings at each step:

**Stage 1 — daemon restart via REST.** `POST /api/daemon/restart` cleanly restarts the daemon service (REST response within < 3 s, fresh `job_id`), but does **not** wake a Dynamixel motor backend locked by overload-protect or position-error. After the restart, `backend_status.ready: false` and `head_joints: null` stayed unchanged for 60 s+; only `motor_control_mode` was reset to `disabled`, and the `head_pose` read returned a different, also stale value.

**Stage 2 — power-cycle of the Reachy.** Physically disconnect power, wait ≥ 30 s, reconnect, allow ~1 minute boot time. The daemon comes back cleanly (`state: running`, a fresh `version` read is possible) — **`backend.ready` can still remain `false`** if the Dynamixel motors or the U2D2 USB interface are in a state that a boot does not clear.

**Stage 3 — hardware recovery outside REST reach.** When the pre-flight gates G1+G2 stay red even after a power-cycle, REST-only recovery is exhausted. Required steps:

- SSH into the Reachy: read `journalctl -u reachy-mini-daemon -n 200` — the daemon logs name explicit bus errors that never reach `backend_status.error` when the bus is "silent dead"
- Dynamixel EEPROM reset via the Pollen CLI / `reachy_mini` Python scripts on the device — a position-error stored in motor EEPROM survives a power-cycle and has to be cleared explicitly with a Dynamixel protocol command
- Mechanical inspection by the operator: check cables and connectors (U2D2 adapter, antenna harnesses) for collision damage
- Physical battery-status check (LED indicator) — a separately tripped battery protection on the motor rail can produce the same symptom
- If symptom persists: contact Pollen support; a motor backend that stays locked after self-collision is a plausible warranty case

**Diagnostic pattern "silent dead" (identifiable from REST alone):**

The hallmark of a hardware lock at the bus interface is the **combination** of the following values in `GET /api/daemon/status`:

- `backend_status.ready: false`
- `backend_status.last_alive: null`
- `backend_status.error: null`
- `backend_status.control_loop_stats.nb_error: 0`
- `backend_status.control_loop_stats.mean_control_loop_frequency ≈ 50 Hz` (the daemon keeps polling)

Plus: three back-to-back `GET /api/state/full` reads return **byte-identical** `head_pose`, `body_yaw`, and `antennas_position` fields, **or** they show micro-drift in the range of ≈ 0.001 rad (≈ 0.06°). Micro-drift combined with `last_alive: null` and `head_joints: null` is a **partial live read**: the daemon reads the forward kinematics of the joints in a separate path, but the main loop that sets `last_alive` and populates `head_joints` never completed a successful iteration. **Both sub-patterns confirm Stage-3 recovery.**

**Diagnostic explanation:** the `mean_control_loop_frequency ≈ 50 Hz` value is **not** proof that the polling loop is running. The field is computed from `_stats["timestamps"]` diffs and can be a stale cache from previous sessions, or the frequency reflects the outer daemon tick without a successful bus read per tick. **`last_alive` is authoritative**: while it stays `null`, the loop is **not** alive at the bus.

**Triage rules (binding):**

- Stage 1 (daemon restart via `POST /api/daemon/restart`) **MUST** be attempted first — it fixes pure software hangs after a connection drop or USB reconnect glitch
- If `backend.ready` is still `false` 60 s after Stage 1, Stage 2 (power-cycle) **MUST** follow; a second restart iteration **MUST NOT** happen automatically
- If the "silent dead" pattern is present after a power-cycle and boot, escalation to Stage 3 (hardware recovery outside REST) **MUST** happen
- **`POST /api/motors/set_mode/enabled` does NOT reactivate the backend polling loop** when the loop is blocked at its pre-loop init step (FK/IK with `no_iterations=20` against a post-collision-shifted platform pose, see `reachy_mini/daemon/backend/robot/backend.py` `run()`). Even after `enabled`, `last_alive` stays `null` and `head_joints` stays `null`. **An additional `systemctl restart reachy-mini-daemon.service` over SSH is equally ineffective in this situation** — verified 2026-05-13.
- In neither Stage 2 nor Stage 3 **MAY** a `POST /api/move/goto` be issued while pre-flight gates G1+G2 are red — that is a direct violation of Layer 6 §"Hard pre-flight"

#### Stage-3 recovery paths (outside REST/SSH reach)

When the "silent dead" pattern persists after a power-cycle plus service restart, walk these paths in order — all require physical access to the Reachy:

1. **Thread wait points via `/proc/$PID/task/*/wchan`** — the first-rank Stage-3 diagnostic. `pgrep -f reachy_mini.daemon.app.main` yields the daemon PID; then:

   ```bash
   ps -L -p <PID> -o tid,pcpu,stat,wchan:30,comm
   for t in /proc/<PID>/task/*/; do
     echo "tid=$(basename $t) wchan=$(cat $t/wchan) comm=$(cat $t/comm)"
   done
   ```

   The `wchan` field names the kernel function the thread is waiting in. The diagnosis has two clearly distinguishable main cases:

   - **`wchan=bcm2835_i2c_xfer` with the thread in `D` state and non-zero CPU%** → **IMU hang on I²C bus 4**. The BMI088 chip still responds to `i2cdetect -y 4` (addresses `0x18` accel + `0x69` gyro), but every sensor read (`read_accelerometer` / `read_gyroscope` / `get_quat` / `read_temperature` in `backend.py:486–496`) blocks the loop. Captured on 2026-05-13 after the Phase-B incident; suspected cause: collision-induced clock-stretching hang or solder-joint damage on the IMU. Recovery options below.
   - **`wchan` with UART / serial driver names** (e.g. `serial8250_tx_chars`, `uart_*`, `tty_*`) → **motor bus hang on `/dev/ttyAMA3`**. Recovery path is Dynamixel Wizard.

   Both sub-paths produce the same REST symptom (`silent dead`); only the wait point separates them. **Without this diagnostic, Stage-3 treatment is guesswork.**

2. **IMU-path workaround — start the daemon without the IMU** when `wchan=bcm2835_i2c_xfer` is the finding. The `bmi088` initialisation in `backend.py:128–136` is gated on `wireless_version=True`; with `wireless_version=False`, `self.bmi088 = None` and the loop block at line 258 (`if self.imu_publisher is not None and self.bmi088 is not None:`) is skipped. **Live-verified 2026-05-13:** with the override config below the Reachy ran all T1–T8 tests from Layer 6 §"Test set" cleanly (see Layer 2 §"T1–T8 live verification"). Concrete override config (Reachy Wireless v1.7.1):

   ```
   /etc/systemd/system/reachy-mini-daemon.service.d/no-imu.conf
   ---
   [Service]
   ExecStart=
   ExecStart=/venvs/mini_daemon/bin/python -u -m reachy_mini.daemon.app.main --serialport /dev/ttyAMA3 --no-wake-up-on-start
   ```

   Then `sudo systemctl daemon-reload && sudo systemctl restart reachy-mini-daemon`. **Important:** the `--serialport /dev/ttyAMA3` must be explicit because `serialport=auto` detection is wired to `wireless_version=True` (boot-log error on the 2026-05-13 attempt: *"No Reachy Mini serial port found"*). Side effect of the override: no Wifi / update / battery telemetry through the wireless routers; reversible by deleting the drop-in file.

3. **BMI088 soft-reset via I²C** before (2), if the chip still reacts:

   ```bash
   sudo systemctl stop reachy-mini-daemon
   sudo i2cset -y 4 0x18 0x7E 0xB6   # BMI088 accel soft reset
   sudo i2cset -y 4 0x69 0x14 0xB6   # BMI088 gyro soft reset
   sudo systemctl start reachy-mini-daemon
   ```

   ⚠ These calls can themselves hang on the stuck bus — `i2cset` uses the same driver. Wrap with `timeout` (e.g. `timeout 5 sudo i2cset ...`).

4. **Daemon stack dump via `py-spy`** as a fallback to the `wchan` diagnosis (if installed on the Reachy): `sudo py-spy dump --pid $(pgrep -f reachy_mini.daemon.app.main)` yields the Python stack of the hanging thread. On a 2026-05-13 Reachy Wireless v1.7.1, `py-spy` was **not** pre-installed; pull it into the daemon venv via `pip` when needed.

5. **Search the daemon logs for non-RuntimeError exceptions**: `journalctl -u reachy-mini-daemon -b | grep -iE "traceback|exception|panic|fatal"` — exceptions outside the loop's try/except are logged but only kill the backend thread; they never reach `backend_status.error`.

6. **Dynamixel Wizard (Robotis)** over a UART/USB adapter to the motor bus — only useful with a UART wait point, **not** with the IMU hang. Ping every motor (IDs 10–18) individually and read the hardware-error register; a residual position error in EEPROM is **not** always cleared by a hardware reboot — it needs an explicit `Reboot` command per motor, or an EEPROM reset.

7. **Contact Pollen Robotics support** — a backend lock that persists through power-cycle and service restart after self-collision is a plausible warranty case. Include in the ticket: `reachy_mini==1.7.1`, `kinematics_engine=AnalyticalKinematics`, `backend_status.ready=false / last_alive=null / nb_error=0 / mean_freq≈50Hz / head_joints=null`, the exact `wchan` from (1), plus the collision sequence from Layer 4 §"Phase-B live incident 2026-05-12".

These seven paths are **diagnostic or reversibly workable**. Only (6) and (7) require hardware service outside one's own reach.

### Layer 6 — live verification methodology

Every live verification of the values pinned in earlier layers follows the methodology documented here. It is the direct consequence of the 2026-05-12 self-collision incident (Layer 4) and binds every future live session.

#### Hard pre-flight (four conditions, re-checked before EVERY single motion)

All four must be `true`, otherwise abort the session:

1. `daemon/status.backend_status.ready == true`
2. `state/full.head_joints` is **not** `null` (Stewart reads work)
3. `robot-app-lock-status.state == "free"`
4. `motors/status.mode == "enabled"`

A single red condition is a stop — no live motion, no "let's just try".

#### Test set — Pollen nominal range with safety margin

Binding test set, all values inside the Pollen nominal operations range (Layer 2) with a 5–15° / 8–15 mm margin to the nominal limit. **Never** use IK-polytope boundary values from Layer 2 §"IK bisection".

| Test | Axis | Target | Pollen nominal limit | Margin | Vs. Phase-A IK boundary |
|---|---|---|---|---|---|
| T1 | Pitch up | **+30°** (+0.524 rad) | +40° | 10° | well below +48° |
| T2 | Pitch down | **−30°** (−0.524 rad) | −40° | 10° | well below −72° |
| T3 | Roll left | **+25°** (+0.436 rad) | +40° | 15° | well below +48° |
| T4 | Roll right | **−25°** (−0.436 rad) | −40° | 15° | well below −48° |
| T5 | Heave up | **+15 mm** (+0.015 m) | IK max +23 mm | 8 mm | safely under +23 mm |
| T6 | Heave down | **−35 mm** (−0.035 m) | IK max −51 mm | 15 mm | safely under −51 mm |
| T7 | Head yaw | **+45°** (+0.785 rad) | +60° relative | 15° | well below ±65° relative |
| T8 | Body yaw | **+90°** (+1.571 rad) | ±155° | 65° | well below ±160° |

Extending this set **MUST NOT** happen without explicit operator approval, and not past the innermost layer defined in Layer 2 §"Three layers of validity" (Pollen nominal).

#### Motion profile

- `duration = 6.0 s` per individual motion (slow, abortable at any time via `POST /api/motors/set_mode/disabled`)
- Between every test: return to `INIT_HEAD_POSE` with `duration = 6.0 s`, then a **2 s standstill**
- Antennas stay throughout at `INIT_ANTENNAS_JOINT_POSITIONS = [-10°, +10°]`; **no** antenna sweeps in this phase
- `body_yaw` is touched only in T8; in T1–T7 it is 0
- Before every motion, re-read the four-point pre-flight — on any deviation, **stop immediately**
- Per motion: read `state/full` after 6.5 s; before recentering, read a second time (steady-state)

#### Verification logic — three values per test

Per test a tuple is recorded:

1. **Target** — the `head_pose` component passed into the API
2. **IK prediction** — the local `analytical_kinematics.ik(target)` joint solution
3. **Real** — the `state/full.head_pose` read after 6.5 s, plus the joint solution obtained by feeding that real pose back through the local IK

Two comparisons:

- **Pose discrepancy** = target − real (norm in 6D pose space); small means the hardware follows the target
- **Joint discrepancy** = IK prediction − IK-from-real; small means the IK is internally consistent

#### Abort criteria during the session

The session is aborted **immediately** when any of the following happens:

- Operator observation "the Reachy is touching itself" or any unusual mechanical noise
- `backend_status.ready` flips to `false`
- `head_joints` becomes `null`
- `backend.error` or `daemon.error` go non-null
- `goto` returns non-200 or the pose discrepancy exceeds 0.2 rad / 30 mm between target and real

On abort: `POST /api/motors/set_mode/disabled` (releases torque immediately), inform the operator, **do not** automatically retry.

#### Recording the results into the spec

Once T1–T8 have run successfully and the discrepancies are measured, Layer 2 §"IK bisection" gets a "live real value" column added per axis, and Layer 3 §"Init pose" gets a note if the device-side identity IK solution differs from the local `analytical_kinematics.ik(np.eye(4))` prediction. The discrepancy magnitude is recorded as a table with the ISO date of the measurement.

## Acceptance Criteria

- [ ] Spec exists at `spec/reachy-mini/motor-positions/de.md` (canonical) and `spec/reachy-mini/motor-positions/en.md` (translation)
- [ ] Every active joint from the live URDF is listed in Layer 1 with exact values (rad + deg)
- [ ] Stewart asymmetry is presented as a pair pattern (A, B, C-1, C-2)
- [ ] The discrepancy between the URDF limit and the `kinematics_data.json` limit is explicitly named; URDF is authoritative
- [ ] IK parameters (motor_arm_length, rod_length, head_z_offset) and safety thresholds (`max_relative_yaw`, `max_body_yaw`) are listed
- [ ] Canonical poses `INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS`, `SLEEP_HEAD_POSE`, `SLEEP_ANTENNAS_JOINT_POSITIONS` are mirrored verbatim from the SDK source as matrices or vectors
- [ ] Hard-coded Stewart joint vectors for init and sleep poses are listed in rad and deg, anti-symmetry is named
- [ ] `wake_up` and `goto_sleep` trajectories are documented step by step with duration and sound asset
- [ ] `collision check: false` is flagged as a central SDK statement in multiple places
- [ ] The discrepancy between `SLEEP_HEAD_JOINT_POSITIONS` and the URDF limits (stewart_2, stewart_5) is flagged as ⚠ TBD, not glossed over
- [ ] Known conflict shapes (antenna crossing, antenna stop, IK polytope violation) are listed, with a clear split into "caught by the IK" vs. "ignored by the IK"
- [ ] The IK-bisection table (Phase A) lists all 12 extreme poses + URDF-limit-triggering joints, clearly marked "not mechanically safe"
- [ ] "Three layers of validity" (IK polytope, URDF limits, Pollen nominal range) are documented as a binding check order
- [ ] The Phase-B incident from 2026-05-12 is documented as a self-collision warning with sequence and aftermath (`backend_status.ready: false`, `head_joints: null`)
- [ ] Motor mode names are correctly named: `enabled` / `disabled` / `gravity_compensation`, not `stiff` / `compliant`
- [ ] The `backend_status.ready` check is named as a mandatory pre-flight before any live motion
- [ ] Three-stage recovery triage is described: daemon restart → power-cycle → hardware recovery outside REST (py-spy, daemon log grep, Dynamixel Wizard, Pollen support)
- [ ] The diagnostic "silent dead" pattern is explicitly named (ready=false, last_alive=null, error=null, nb_error=0, freq≈50 Hz, three byte-identical **or** micro-drifting state reads in a row) as the indicator that recovery is at Stage 3
- [ ] `last_alive` is named as the authoritative truth field for "loop is live at the bus" — `mean_control_loop_frequency` alone is not enough
- [ ] `set_mode/enabled` is documented as NOT sufficient to reactivate the backend when the pre-loop FK/IK init is hanging (verified 2026-05-13)
- [ ] `gravity_compensation` is marked with the `kinematics_engine=Placo` constraint; with `AnalyticalKinematics` it is unusable (source: backend.py:563)
- [ ] Layer 2 §"T1–T8 live verification 2026-05-13" lists per test the target pose, IK prediction, measured pose, and measured Stewart joints from the live run
- [ ] Sleep-pose pitch is documented with a positive sign (+24.4°); the xyz-Euler convention is explicit; live-verified with pitch ≈ +26° on the real device
- [ ] Pitch bleed on roll and heave-up is named as a systematic Stewart geometry coupling, not a calibration drift
- [ ] T8 `body_yaw=+90°` is documented as the evidence for the `max_relative_yaw=65°` clip (target ≠ real, but IK-consistent)
- [ ] Stage-3 path 2 (IMU bypass via systemd drop-in) is marked as **live-verified 2026-05-13**
- [ ] Layer 6 §"Live verification methodology" contains the binding test set T1–T8 with safety margins to the Pollen nominal range
- [ ] Layer 6 names four pre-flight gates (`backend.ready`, `head_joints`, `app-lock`, `motors.mode`) and abort criteria
- [ ] Layer 6 explicitly forbids IK-polytope boundary values as live targets
- [ ] DE and EN versions are structurally in sync
- [ ] Every concrete number carries a source attribution (URDF, `analytical_kinematics.py`, `reachy_mini.py`, `kinematics_data.json`, Pollen datasheet, Phase-A IK sweep)

## References

- Live source for joint limits: `http://<daemon-host>:8000/api/kinematics/urdf` (see [`spec/reachy-mini/daemon-rest-api/`](../daemon-rest-api/en.md))
- URDF source: [`src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)
- Kinematics constants and solver: [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)
- Kinematics geometry: [`src/reachy_mini/assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json)
- Canonical poses + `wake_up`/`goto_sleep`: [`src/reachy_mini/reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)
- Pollen hardware datasheet (nominal operations range, DOF table): <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware>
- Related specs in this repo: [`reachy-mini/control-surface`](../control-surface/en.md) (inventory + motion design), [`reachy-mini/daemon-rest-api`](../daemon-rest-api/en.md) (REST endpoints), [`reachy-mini/motions/`](../motions/) (concrete motion sequences)
- Consuming skills: [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md), [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/en.md), [`dance-choreography`](../../claude/dance-choreography/en.md), [`app-scaffold`](../../claude/app-scaffold/en.md)

## Open Questions

- Are the Stewart joint values from `SLEEP_HEAD_JOINT_POSITIONS` (`stewart_2 = +72.32°`, `stewart_5 = −70.84°`) consistent with the tighter URDF limits (`+70°` and `−70°`)? If not, is the SDK constant stale, is the URDF limit too conservative, or is the sleep pose intentionally placed at the polytope edge? Clarify on the real device and possibly with Pollen
- How large is the real workspace for head translations x, y, z within the IK-reachable polytope? A short sweep script on the real device can pin the ⚠ TBD values in Layer 2
- Is there a geometric "crossing-forbidden zone" for the antennas where the left and right antenna touch mechanically? Measure
- Should `goto_sleep` switch the antenna motors into `compliant` at the end to avoid drift at the end-stop? Suggestion for Pollen, or a workaround in our `reachy-mini-inspect`/`reachy-mini-start` path? Document here, do not decide
- Are the `max_relative_yaw=65°` and `max_body_yaw=160°` constants stable in future SDK versions, or will they become configurable? If they bump, the spec needs to be re-verified
- What velocity and acceleration profiles does `goto_target` use by default? The URDF only names velocity = 8 rad/s as a hard cap, but the real profile generator likely smooths. Document in a follow-up spec if needed
- Should this spec point at a drift-check script (e.g. `scripts/check-motor-positions-spec.py`), analogous to the `daemon-rest-api` spec? Worth it once the URDF is changed by an SDK version bump
