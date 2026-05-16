# Detecting atypical motion patterns on the Reachy Mini

Status: draft

## Context

Anyone who triggers a motion on the Reachy Mini must avoid four classes of atypical behaviour before they reach the device, detect them while they run, and trace them back when they have happened: **head-against-body self-collision** (observed live on 2026-05-12), **jerky motion** that mechanically slams the Stewart actuators, the antenna-servo **"wacken"** (wobble) inside the deadband around `0°`, and **Stewart-limit knock** with an unsolvable inverse-kinematics call. The detection logic for these lives scattered today: motor-positions Layer 4 names the conflict shapes, control-surface the mechanical limits and the antenna deadband, app-logging the triage classes, and individual skills carry their own ad-hoc checks. This spec centralises the detection methodology so every consumer (skills, agents, later a dedicated validator) shares the same rules.

**Readers.** Authors of the four named consumer skills (`reachy-mini-sdk`, `reachy-mini-inspect`, `app-log-triage`, `dance-choreography`), and implementers of a future dedicated `motion-validator` skill.

The spec is consumed by the [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) skill for pre-flight validation of code snippets, by the [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/en.md) skill for live telemetry reads, by the [`app-log-triage`](../../claude/app-log-triage/en.md) skill for post-hoc analysis of app logs, and by the [`dance-choreography`](../../claude/dance-choreography/en.md) skill when composing extreme pose sequences.

Important finding up front: the Pollen SDK performs **no self-collision check** (`engine: AnalyticalKinematics, collision check: false` from `GET /api/kinematics/info`, see [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2). A pose accepted by the IK polytope is *kinematically reachable* — not automatically *mechanically safe*. This spec's detection methodology fills exactly that gap until Pollen closes it SDK-side.

Verification basis: Reachy Mini Wireless, firmware 1.7.1, live-verified on **2026-05-12** (Phase-B self-collision incident) and **2026-05-13** (T1–T8 motion verification, antenna-deadband measurements). The Lite platform is explicitly **not yet** verified — see Open Questions.

## Goals

- Separate three orthogonal detection phases cleanly: **pre-flight** (before the command reaches the daemon), **live** (telemetry monitor during the motion), **post-hoc** (log and telemetry replay)
- Catalogue four binding anomaly classes — head-against-body self-collision, jerky motion, antenna wobble, Stewart-limit knock — each with severity grade, binding rule, and at least one concrete detect signal per phase
- Name the recovery strategy per class when the anomaly could not be prevented (power-cycle, pose correction, telemetry capture)
- Keep cross-links to the canonical value sources (motor-positions, control-surface, daemon-rest-api, app-logging) instead of duplicating values
- Define a binding unified anomaly-event-record schema so consuming skills can report structured findings and aggregation works across platforms
- Keep hardware-verification status per class explicit (Wireless 1.7.1 verified, Lite not yet) so a later Lite verification pass can be targeted

## Non-Goals

- Implementing the detection inside a concrete skill or agent — this spec is normative knowledge base, not an operations script. The consuming logic lives in the named consumer skills or in a future `motion-validator` skill
- Self-collision algorithm with URDF-mesh check or pose-sampling solver — explicitly not in the Pollen SDK and not in this spec. Until such a solver is available, the spec stays on pose-range bounds (Pollen nominal) as the binding layer
- General safety architecture: emergency-stop paths, brown-out protection, thermal budget, cool-down — belongs to [`reachy-mini/control-surface`](../control-surface/en.md) §"Safety limits"
- Concrete motion composition (easing, anticipation, beat sync) — belongs to [`reachy-mini/control-surface`](../control-surface/en.md) §"Patterns for natural, fluid motion" and to the [`dance-choreography`](../../claude/dance-choreography/en.md) skill
- Values for the simulation variant when they diverge from hardware — the sim shares IK and URDF limits, but phenomena such as the antenna deadband and Stewart effort knock are sim-irrelevant
- Hardware bring-up, calibration, firmware flash, IMU recovery — separate skills (planned), not part of this spec
- Audio, vision, or LED anomalies — other subsystems

## Requirements

### Anomaly class overview

| Class | Name | Severity | Binding rule | Primary detect path | Verified |
|---|---|---|---|---|---|
| **A** | Head-against-body self-collision | hard | **MUST NOT** occur | Pre-flight pose-range check | Wireless 1.7.1, 2026-05-12 |
| **B** | Jerky motion (excessive joint velocity) | soft/medium | **SHOULD NOT** (>0.16 rad/sample) / **MUST NOT** (>0.30 rad/sample) | Pre-flight pose-delta / dt | Wireless 1.7.1, partial (see OQ2) |
| **C** | Antenna "wacken" (micro-oscillation in deadband) | soft | **SHOULD** be avoided | Pre-flight antenna setpoint | Wireless 1.7.1, 2026-05-13 |
| **D** | Stewart-limit knock + IK-unsolvable | medium | **MUST NOT** be attempted | Pre-flight IK bisection | Wireless 1.7.1, 2026-05-13 |

### Class A — Head-against-body self-collision

**What happens.** A pose target outside the Pollen nominal operations range (±40° pitch/roll, ±60° head-yaw) is *IK-mathematically solvable* but *not mechanically safe*. The daemon dispatches the motion; the Stewart mechanism presses the head against the body shell. After the incident the daemon reports `backend_status.ready: false`, `head_joints: null`, and `POST /api/motors/set_mode/gravity_compensation` answers with HTTP 500. Canonical precedent: the Phase-B live incident on **2026-05-12** on a Reachy Wireless 1.7.1 ([`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 4).

**Binding rule.** A live motion **MUST** stay inside the **Pollen nominal range** (±40° pitch/roll, ±60° head-yaw, ±155° body-yaw). The IK polytope and the URDF mechanical limits are **not enough** as binding layer. This rule is the innermost of the three validity layers from [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2 §"Three layers of validity".

**Detect signals.**

| Phase | Signal | Cost disposition |
|---|---|---|
| Pre-flight | Target pose outside (±40°, ±40°, ±60°, ±155°) ⇒ **MUST** be rejected | trivial, static |
| Live | `GET /api/daemon/status.backend_status.ready` flips to `false` after a pose command, simultaneously `head_joints: null` and `state/full` returns no current head-pose snapshot | polling cost; recovery-relevant |
| Post-hoc | App log contains `ConnectionError: Could not connect to daemon on localhost` (see [`reachy-mini/app-logging`](../app-logging/en.md) triage class `daemon-stale-state`) or daemon log shows `backend.ready: false` right after `start-app` | replay-capable |

**Recovery.** Power-cycle the Reachy. The daemon has no soft-recovery path for this class because the sensor backend status hangs. Before the next live attempt the offending pose range in the consuming script should be corrected.

### Class B — Jerky motion (excessive joint velocity)

**What happens.** The daemon clamps a joint-velocity request at the URDF limit (8 rad/s per Stewart joint), but that clamp is *too coarse* — it only prevents the absolute peak, not a sequence of pose targets that combine to a mechanically straining jerk. Symptoms: audible servo clack, visible platform vibration, Stewart-arm resonance in extreme cases. Mechanical consequence: shortened servo life, platform calibration may drift.

**Binding rule.** A trajectory **SHOULD NOT** produce a joint-vector delta between two consecutive pose commands that exceeds the URDF velocity limit at the command frequency. Concretely: for the Pollen daemon's typical 50 Hz loop that means a joint delta per sample of at most `8 rad/s ÷ 50 Hz = 0.16 rad ≈ 9.2°`.

**Detect signals.**

| Phase | Signal | Cost disposition |
|---|---|---|
| Pre-flight | Pose-to-pose joint diff per command interval > 0.16 rad (=9.2°) on an active Stewart joint ⇒ warning; > 0.30 rad ⇒ rejection | trivial, static when the trajectory is known beforehand |
| Live | `GET /api/state/full?with_head_joints=true` two consecutive reads, compute ‖Δjoints‖₂ per Δt; or `backend_status.nb_error` spike | polling cost; nb_error is more authoritative |
| Post-hoc | Telemetry replay: per Stewart joint the maximum sample-to-sample difference; smoothness score (mean of ‖Δjoints‖₂ per second) | replay-capable, static |

**Recovery.** A pure Class B anomaly rarely produces a daemon hang; servo stress is the primary consequence. Correct via gentler easing or lower command frequency in the script.

### Class C — Antenna "wacken" (micro-oscillation in deadband)

**What happens.** Each antenna servo (XL330-M077-T) exhibits a sign-asymmetric deadband around `0°` in which the setpoint is not held steadily: the servo oscillates ~±0.5° peak-to-peak. Verified empirically on a Wireless 1.7.1 (**2026-05-13**): setpoints at `|x| ≥ 15°` held perfectly steady (stdev = 0.000° over multi-second observation); within `|x| < 5°` at least one of the two antennas wobbled visibly. Source: [`reachy-mini/control-surface`](../control-surface/en.md) §"Mechanical and electrical limitations". The SDK's own `INIT_ANTENNAS_JOINT_POSITIONS = [-10°, +10°]` is an implicit acknowledgement of the same property.

**Binding rule.** Antenna rest setpoints **SHOULD** be kept at `|setpoint| ≥ 5°`, ideally `≥ 10°` to match `INIT_ANTENNAS_JOINT_POSITIONS`. A motion **SHOULD NOT** ease into an all-zero antenna pose at the end when the robot will then sit idle — at least one antenna will visibly wobble.

**Detect signals.**

| Phase | Signal | Cost disposition |
|---|---|---|
| Pre-flight | Antenna setpoint in the rest frame with `\|x\| < 5°` ⇒ warning; `\|x\| < 2°` ⇒ rejection | trivial, static |
| Live | Standard deviation of antenna joint reads over a 2-second rolling window > 0.2° despite a stable setpoint ⇒ antenna in deadband | polling cost; needs telemetry buffer |
| Post-hoc | Telemetry replay: per antenna the fraction of samples with `\|setpoint\| < 5°`; stdev per setpoint bin | replay-capable |

**Recovery.** Class C damages nothing and blocks no follow-up motion. It is primarily a quality finding ("looks nervous even though the robot is still"). Correction: set antenna rest poses to `≥ 10°`.

### Class D — Stewart-limit knock + IK-unsolvable

**What happens.** A pose target requires a Stewart joint outside its URDF limit (for example `stewart_1 > +80°` or `stewart_4 < −80°`, see [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 1). The `AnalyticalKinematics.ik` solver throws an exception, the daemon replies with HTTP 4xx or 5xx depending on the path. In live telemetry `backend_status.nb_error` rises and `head_joints` either shows the last valid value or `null` (see the `_status.ready` desync from [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 5). The class is medium severity because it does not produce a mechanical self-collision but can leave the IK solver in an unclear follow-on state.

**Binding rule.** A pose target whose IK solution lies outside the URDF joint limits **MUST NOT** be sent to the daemon. Pre-flight here is markedly cheaper than live recovery.

**Detect signals.**

| Phase | Signal | Cost disposition |
|---|---|---|
| Pre-flight | Local IK call (for example `reachy_mini==1.7.2` in a helper venv) with the URDF-limit tuple as stop criterion; result outside ⇒ rejection | module-call cost; pins the local SDK version |
| Live | `backend_status.nb_error` spike (`> 0` and rising) after a pose command; `head_joints` vs. target-joints diff > URDF-limit tolerance | polling cost; nb_error is more authoritative |
| Post-hoc | App log shows a `kinematics`-exception traceback or daemon log shows an `ik_failed` marker; in app logs this is a class `kinematics-unsolvable` (see [`reachy-mini/app-logging`](../app-logging/en.md), possibly to be added) | replay-capable |

**Recovery.** Bring the pose target back to a Layer-2 / Layer-3 validity (respect URDF limits) and resend. Class D needs no power-cycle as long as the IK solver aborts cleanly.

### Phase 1 — Pre-flight (detection before the command)

Mandatory checks before every pose or trajectory command to the daemon:

- **MUST** validate the target-pose range against the Pollen nominal operations range (Class A); outside ⇒ rejection with a Class A marker
- **MUST** for trajectories (multiple consecutive pose targets) compute the joint-vector diff per command interval and check against `0.16 rad / sample` (Class B); exceeded ⇒ warning or rejection
- **SHOULD** check antenna setpoints for rest poses against `|x| ≥ 5°` (Class C)
- **MUST** for poses that are not obviously inside the Pollen nominal range, perform a local IK call and check the Stewart-joint solution against URDF limits (Class D)
- **SHOULD** report the result structured — class, severity, affected joints, suggested correction

Note: pre-flight does not replace the live and post-hoc layers. A pose range may be fine pre-flight, but Stewart-platform coupling (pitch bleed from [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2 §"T1–T8 live verification") can produce a different runtime pose than intended.

### Phase 2 — Live (detection during the motion)

Mandatory checks while motions are in flight, on top of the daemon REST API (see [`reachy-mini/daemon-rest-api`](../daemon-rest-api/en.md)):

- **MUST** poll `GET /api/daemon/status` immediately after a pose command, using the three liveness cross-checks from [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 5: (a) `mean_control_loop_frequency > 40 Hz` + `nb_error == 0`, (b) `head_joints` populated (after `?with_head_joints=true`), (c) pose micro-drift between two consecutive `state/full` reads
- **MUST** treat `head_joints: null` plus `backend_status.ready: false` after a pose command as a Class A anomaly and escalate the recovery path (power-cycle recommended, no auto-restart from inside the skill)
- **SHOULD** report an `nb_error` spike (`> 0` and rising) as a Class D indicator, together with the last sent pose target
- **SHOULD** keep antenna joint reads in a 2-second rolling window and compute standard deviation as a Class C indicator when the setpoint is nominally stable
- **MUST NOT** auto-restart or auto-`stop-current-app` after detecting Class A or D — such mutations belong in a recovery skill, not in the detection layer; instead, report the result structured back to the user

Note: the Pollen daemon shows a known desync bug (`_status.ready` and `_status.last_alive` are not synchronised, see [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 5). The three cross-checks are therefore more authoritative than a single bool read.

### Phase 3 — Post-hoc (detection after the incident)

Detection from recorded app logs and telemetry replays:

- **MUST** evaluate app logs against the triage classes from [`reachy-mini/app-logging`](../app-logging/en.md) — in particular `daemon-stale-state` is the post-hoc fingerprint of Class A
- **SHOULD** evaluate telemetry replays for per-joint maximum-sample-difference and smoothness score (Class B)
- **SHOULD** examine antenna joint reads per setpoint bin for stdev (Class C)
- **SHOULD** mark kinematics-exception tracebacks in app logs as Class D indicators
- **MUST** carry the verification-date marker per detected anomaly (Wireless 1.7.1 / Lite / Sim) so a later class refinement remains traceable

### Unified anomaly-event-record

Consuming skills (pre-flight from `reachy-mini-sdk`, live from `reachy-mini-inspect`, post-hoc from `app-log-triage`) **MUST** emit structured anomaly records in the following JSON format so that aggregation and replay remain consistent across platforms and skills:

```jsonc
{
  "class": "A" | "B" | "C" | "D",
  "phase": "pre-flight" | "live" | "post-hoc",
  "severity": "hard" | "warn" | "info",
  "detected_at": "2026-05-13T17:42:11Z",
  "summary": "head pose pitch = +48° outside Pollen nominal range ±40°",
  "telemetry_snapshot": {
    "head_pose": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]],
    "head_joints": [0.0, 0.5, -0.3, ...],
    "backend_status": { "ready": false, "nb_error": 0, "mean_control_loop_frequency": 49.8 }
  },
  "suggested_correction": "clamp pitch to +40°",
  "verification_basis": "Reachy Mini Wireless firmware 1.7.1, 2026-05-13"
}
```

- **MUST** the `class` field carry one of the four class identifiers (`A` / `B` / `C` / `D`)
- **MUST** the `phase` field carry one of the three detection-phase identifiers (`pre-flight` / `live` / `post-hoc`)
- **MUST** the `severity` field carry one of `hard` / `warn` / `info`
- **MUST** the `detected_at` field carry an ISO-8601 timestamp with a timezone suffix
- **MUST** the `verification_basis` field carry platform + firmware + date
- **SHOULD** `telemetry_snapshot` contain the relevant fields without PII or auth tokens (PII clause from [`reachy-mini/app-logging`](../app-logging/en.md) applies)
- **MAY** `suggested_correction` stay empty when no deterministic correction is derivable

## Acceptance Criteria

- [ ] The spec separates three detection phases (pre-flight / live / post-hoc) and lists, per class, at least one concrete detect-signal source for each phase
- [ ] Every class carries a binding rule (MUST / MUST NOT / SHOULD / SHOULD NOT) and a recovery path
- [ ] Class A names the Pollen nominal range as the binding layer and links to [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2 §"Three layers of validity"
- [ ] Class C names `|setpoint| ≥ 5°` (ideal `≥ 10°`) as the antenna rest setpoint and links to [`reachy-mini/control-surface`](../control-surface/en.md) §"Mechanical and electrical limitations" (bullet "Antenna deadband around 0°")
- [ ] Class B names `0.16 rad / sample` (8 rad/s at a 50 Hz loop) as the pre-flight threshold for joint-vector diff
- [ ] Class D requires a local IK call against URDF limits as the pre-flight check, instead of relying on the daemon
- [ ] Per detection phase at least one concrete detect signal is named that can be implemented today against the verified REST contract from [`reachy-mini/daemon-rest-api`](../daemon-rest-api/en.md) (no TBD-only entries)
- [ ] Hardware-verification status is explicit per class (Wireless 1.7.1 verified 2026-05-12/13; Lite explicitly not yet)
- [ ] Cross-links to consumer skills (`reachy-mini-sdk`, `reachy-mini-inspect`, `app-log-triage`, `dance-choreography`) are visible in the body and resolvable
- [ ] The binding anomaly-event-record is anchored with the five mandatory fields (`class`, `phase`, `severity`, `detected_at`, `verification_basis`) as MUST rules in the Requirements section
- [ ] Live consumers poll `backend_status` immediately after every pose command, not only on visible anomaly
- [ ] Per detected anomaly the verification-date marker (platform + firmware + date) is carried in the anomaly-event-record
- [ ] The spec contains no auto-restart or auto-mutation instructions — detection stays strictly separated from recovery

## Sources

- Pollen SDK source (IK solver, pose constants, daemon status loop): <https://github.com/pollen-robotics/reachy_mini>
- Phase-B live incident (head-against-body self-collision, 2026-05-12): [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 4
- Three validity layers + IK polytope: [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 2
- URDF joint limits and Stewart asymmetry: [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 1
- Antenna deadband + pitch bleed: [`reachy-mini/control-surface`](../control-surface/en.md) §"Mechanical and electrical limitations"
- Daemon liveness cross-checks + `_status.ready` bug: [`reachy-mini/motor-positions`](../motor-positions/en.md) Layer 5
- REST endpoint inventory: [`reachy-mini/daemon-rest-api`](../daemon-rest-api/en.md)
- Triage classes for post-hoc analysis: [`reachy-mini/app-logging`](../app-logging/en.md)
- Consumer skills: [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md), [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/en.md), [`app-log-triage`](../../claude/app-log-triage/en.md), [`dance-choreography`](../../claude/dance-choreography/en.md)

## Open Questions

- Should pre-flight pose validation live in the consuming plugin (today) or directly inside the Pollen SDK (upstream PR)? Recommendation: in this plugin as skill logic first, until Pollen ships the check SDK-side. The spec keeps the detection rules so that both routes share the same contract
- Which exact jerk threshold for Class B (rad/s² or rad/sample) is binding? The `0.16 rad/sample`-SHOULD-NOT and `0.30 rad/sample`-MUST-NOT thresholds are pose-velocity bounds derived from the URDF velocity limit, not acceleration bounds. An empirical measurement campaign (trajectory smoothness vs. servo acoustics / platform vibration) is needed to validate or recalibrate them
- For Class A live detection: what is the maximum latency between a pose command and the `backend.ready=false` flip in the failure path? Needs a targeted measurement on a second Reachy Wireless because repeating the Phase-B incident on the same device is not advisable while the device is still on the recovery path
- Lite platform: all four classes are only verified on Wireless. In particular the daemon REST responses may follow a different schema path on Lite (USB-host-driven daemon vs. on-board), and the antenna deadband is servo-hardware-specific and should be confirmed on Lite with the same methodology (`|x| < 5°` setpoint sweep, stdev over second-window)
- Should the wobble/smoothness thresholds (Class B `0.16 rad/sample` and `0.30 rad/sample`, Class C `stdev > 0.2°`) be treated as configuration tunables or as fixed constants of this spec? Recommendation: fixed constants in the spec; consumers may be stricter but not laxer
- Should `app-logging` be extended with the triage class `kinematics-unsolvable` for Class-D post-hoc detection? The present spec references that class in the Class-D detect table, but it does not yet exist in the `app-logging` spec. Follow-up PR recommended
