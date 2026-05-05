# Reachy Mini SDK Skill

Status: draft

## Context
The `reachy_mini` Python SDK from Pollen Robotics / Hugging Face is the primary interface for programmatically controlling the Reachy Mini robot: head motion (pan/tilt/roll), antennas, behaviors, optionally audio and vision streams. Claude Code should produce idiomatic, runnable code whenever a task touches this SDK — which requires a focused knowledge base that activates exactly when the task touches the SDK, and that surfaces drift instead of repeating outdated patterns. This specification defines what the `reachy-mini-sdk` skill ships and which concerns it deliberately delegates to other skills.

## Goals
- Claude Code reliably detects tasks that touch the `reachy_mini` SDK and activates this skill at exactly those moments
- Claude Code emits idiomatic code verified against the official API
- Every code suggestion ties back to a named SDK version
- The plugin makes version drift visible instead of hiding it
- Non-concerns are clearly delegated to specialised skills rather than bloating this one

## Non-Goals
- Hardware bring-up, calibration, firmware flashing (separate skill planned)
- Simulation / MuJoCo / URDF of the Reachy model (separate skill possible)
- Publishing behaviors to Hugging Face Spaces / Hub (separate skill `behavior-publish-hf` planned)
- Audio beat / tempo detection for dance applications (separate skill `audio-beat-tracking` planned)
- Home Assistant integration (separate skill `home-assistant-bridge`)
- Scaffolding a new behavior (separate skill `app-scaffold`)
- Live deployment / on-device testing (separate agent `reachy-mini-on-device`)

## Requirements

### Triggering and activation
- **MUST** ship a description tight enough for Claude Code to activate the skill on every task that touches the `reachy_mini` SDK — recognisable through imports of `reachy_mini`, the `ReachyMini` class, behavior definitions, or API calls for head/antenna motion
- **MUST** name the key trigger terms inside the description: `reachy_mini`, `ReachyMini`, behavior, antennas, pan/tilt/roll, Move
- **SHOULD** explicitly state _when not_ to activate (e.g. pure hardware-bringup tasks or pure simulation tasks without SDK contact)

### Knowledge-base content
- **MUST** document the following SDK building blocks:
  - Construction and connection management of the `ReachyMini` instance, including lifecycle (open/close, context manager, sync vs. async variant, `spawn_daemon=True, use_sim=True` for sim)
  - **Wake/sleep lifecycle**: `wake_up()` before any motion — otherwise pose commands are silently ignored; `goto_sleep()` for idle / shutdown; pose constants `SLEEP_HEAD_POSE`, `INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS` from `src/reachy_mini/reachy_mini.py`
  - **Motion API with method choice**: explicitly document both paths —
    - `goto_target(head=<4×4>, antennas=[r, l] in rad, body_yaw, duration, method)` as the **default for choreographed motions ≥ 0.5 s**; `method` from the `InterpolationTechnique` enum: `MIN_JERK` (default), `LINEAR`, `EASE_IN_OUT`, `CARTOON`
    - `set_target(head, antennas, body_yaw)` as the **real-time path for high-frequency loops** (50–100 Hz tick rate, **single-owner loop**); do not mix with `goto_target`, they will overwrite each other
    - Source: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/motion-philosophy.md>
  - Head motion: 6 DoF Stewart platform, 4×4 pose matrix, builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees=True)`; value ranges per [`reachy-mini/control-surface`](../../reachy-mini/control-surface/en.md) (head pitch/roll ±40°, head yaw ±60°, body yaw ±155°, yaw delta ≤ 65°)
  - Antenna control: 2× XL330-M077-T, **order `[right, left]` in radians** (not degrees!), angle, speed, synchronisation with head moves
  - **`Move` ABC and `play_move()` / `async_play_move()`**: for reusable motion, define a `Move` subclass with a `duration` property and an `evaluate(t) → (head, antennas, body_yaw)` method (source: `src/reachy_mini/motion/move.py`)
  - **App lifecycle**: `ReachyMiniApp` subclass with `run(self, reachy_mini, stop_event)`; `wrapped_run()` in `__main__`; tick frequency, clean stop via `stop_event`, exception handling
  - **Safe-torque anti-jerk pattern** (mandatory on motor toggle, otherwise the head jerks):
    1. Before `disable_motors()`: drive to `SLEEP_HEAD_POSE`
    2. Before `enable_motors()`: set the goal to the current pose (short `goto_target` with `duration ≈ 0.05`), only then enable
    3. On a mixed-motor state (some on, some off): first `disable_motors()` on all IDs, then enable sequentially
    Source: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- **MUST** include at least one runnable, minimal code example per documented area
- **MUST** name the SDK version each example was verified against and link the official source (Pollen Robotics docs or the official GitHub repo)
- **SHOULD** cover async patterns (tasks, cancellation, cleanup on exceptions) when the SDK exposes an async surface
- **SHOULD** name typical failure conditions (hardware unplugged, USB / serial errors, protocol mismatch between SDK and firmware, missing daemon)
- **MAY** include hints on update frequency, latency, and behavior performance

### Code-example conventions
- **MUST** target whichever Python lower bound the official SDK requires; on drift the lower bound is bumped through a spec update
- **MUST** show examples in the style the SDK itself prescribes (e.g. `with`-statements when the SDK exposes a context-manager model)
- **MUST** decorate every snippet with a source reference to the official Pollen Robotics docs or the official GitHub repo
- **MUST NOT** include code examples copy-pasted unchecked from older Reachy SDKs (Reachy 2, Reachy Pro) — any reuse must be marked when the API differs for Reachy Mini

### Version pinning and drift detection
- **MUST** name in the skill body the SDK version the skill is currently verified against (e.g. `reachy_mini==0.x.y`)
- **SHOULD** define a repeatable drift check: each new `reachy_mini` release re-validates the skill against the current API — manual at the next touchpoint or via a planned audit skill
- **MUST** mark statements that are not verified for lack of hardware or for lack of verification with a visible marker (e.g. `> ⚠ TBD: validate against real hardware`)

### Boundaries to neighbouring skills
- **SHOULD** point at the skill `app-scaffold` whenever the task creates a _new_ behavior — instead of duplicating scaffolding logic
- **SHOULD** point at the skill `home-assistant-bridge` whenever the task connects Reachy with Home Assistant
- **SHOULD** point at the planned skill `audio-beat-tracking` whenever the task analyses audio (e.g. for dance synchronisation)
- **SHOULD** point at the planned agent `reachy-mini-on-device` whenever the task wants to test a behavior live on the device

## Acceptance Criteria
- [ ] The skill exists at `skills/reachy-mini-sdk/SKILL.md` with valid frontmatter (`name: reachy-mini-sdk`, `description`, optional `tags`) and is accepted by the catalog generator
- [ ] The `description` frontmatter is written so that Claude Code activates the skill on a test prompt containing `from reachy_mini import ReachyMini`
- [ ] The knowledge base documents at least: construction / lifecycle, head motion, antennas, behavior loop, cleanup
- [ ] At least one runnable code example exists per documented area
- [ ] Every example carries a source reference and names the SDK version
- [ ] The verified SDK version is explicitly stated in the skill body
- [ ] Out-of-scope concerns (bring-up, simulation, HF publishing, beat tracking, HA bridge, behavior scaffolding, on-device testing) are marked as "use skill / agent X for this"
- [ ] Statements without hardware verification carry a visible TBD marker
- [ ] The skill renders in the MkDocs catalog (build runs `task docs --strict` without error)

## References
- Upstream SDK repo (canonical source for API, versioning, license): <https://github.com/pollen-robotics/reachy_mini>
- SDK source tree (`ReachyMini`, IO, Media, Motion, Daemon, Apps, Tools): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Motion module (`Move` ABC, easing modes `MIN_JERK` / `CARTOON`, `goto`, `recorded_move`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- API docs (MDX sources for `reachymini`, `media`, `motion`, `daemon`, `apps`, `tools`, `utils`, REST API, OpenAPI schema): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/API>
- SDK concept docs (Quickstart, Core Concept, Apps, Python / JavaScript SDK, Media architecture, Installation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Runnable examples (canonical templates for code snippets): <https://github.com/pollen-robotics/reachy_mini/tree/main/examples>
- Upstream Claude skills (Pollen's parallel authoring source; drift check reconciles against them): <https://github.com/pollen-robotics/reachy_mini/tree/main/skills>
- Platform profile docs (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Open Questions
- Which exact `reachy_mini` version do we pin initially? Proposal: the last stable release before the hardware arrives, documented in the skill body.
- Does the SDK ship an official Python compatibility matrix we should link?
- Should code examples favour the async or the synchronous variant of the SDK? Depends on what the SDK actually offers as the primary surface.
- How deeply should Behaviors-publishing conventions for Hugging Face appear in this skill _before_ a dedicated `behavior-publish-hf` skill exists?
- Which tag set is appropriate? Proposal: `[reachy-mini, sdk, python, robotics]`. Final decision in the frontmatter.
- Should the skill also activate on pure simulation tasks (MuJoCo / URDF without real hardware)? Leaning: no — that belongs in a separate simulation skill.
- How often is the drift check executed? Proposal: quarterly, or on every `reachy_mini` major release.
- Who is the authoritative source on conflicts between the Pollen Robotics docs and the SDK source code? Proposal: source wins, docs as secondary.
