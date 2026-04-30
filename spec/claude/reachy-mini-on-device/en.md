# On-Device Test Agent for Reachy Mini

Status: draft

## Context
Once the hardware is on hand, behaviors must be exercised on the real Reachy Mini before they end up in an app or a Hugging Face publish. By hand that means: SSH or USB attach, sync code, install dependencies, start the behavior, collect logs and telemetry, stop on misbehavior, write a report. These steps are sequential, latency-bound, error-prone, and they produce a lot of raw output — exactly the kind of work that clogs the main thread of a Claude Code conversation when run inline. The `reachy-mini-on-device` agent encapsulates that lifecycle in its own tool session and hands the main thread a tight structured summary. It tests, it does not develop — code changes stay with the main thread, which uses the `reachy-mini-sdk`, `behavior-scaffold`, and `home-assistant-bridge` skills.

## Goals
- A behavior reaches the real device and runs live in a single agent invocation
- Telemetry and logs are gathered structurally during the run, without flooding the main context
- Misbehavior leads to a controlled emergency stop and a clean disconnect, not hanging connections
- The result returns as a tight PASS/FAIL summary with a pointer to a full-text log artifact
- The agent stays narrow: testing and observation, no motion logic, no code change to the behavior

## Non-Goals
- Hardware bring-up (separate skill planned)
- Firmware flashing (separate skill planned)
- Developing the behavior or the app (separate repos, separate skills)
- Publishing the behavior to Hugging Face (`behavior-publish-hf`, planned)
- Audio / beat tracking (`audio-beat-tracking`, planned)
- Persistent operation / watchdog in production — the agent runs a test lifecycle, not a daemon

## Skill-vs-Agent rationale
This concern is modelled as an **agent** rather than a skill because several rationales from `nolte-shared/spec/claude/skill-vs-agent/` apply at once:

- **Long, latency-bound tool session** — SSH/USB connect, scp/rsync deploy, behavior-step watching with seconds-to-minutes latency per phase. Skills are tuned for inline interactive workflows; a long sequential lifecycle belongs in its own tool session.
- **Context volume** — raw behavior logs and sensor telemetry can produce thousands of lines per run. Inlining them into the main thread would consume context without informing the next decision. The agent reduces this to a summary plus an artifact path.
- **Multi-stage orchestration with error recovery** — connect → deploy → install deps → start → watch → stop → disconnect. Each stage has its own failure modes (auth error, disk full, behavior crash, USB disconnect) that need own recovery paths without interrupting the main thread.
- **Standalone tool set** — the agent needs Bash for `ssh`/`scp`/`rsync` plus possibly a device-specific CLI. Those tools do not belong in the main thread, which is dominated by code editing.
- **Specialised behavior** — emergency-stop, graceful disconnect cleanup, and a strict output format are requirements that a dedicated agent canonicalises.
- **Distribution: `plugin`** — the agent ships with the plugin.

## Requirements

### Inputs
- **MUST** accept a target behavior path (a local directory in the consuming app repo)
- **MUST** accept the platform — `wireless` / `lite` / `simulation` — because deploy path, telemetry availability, and emergency-stop mechanics differ materially across platforms (see Platform profiles)
- **MUST** accept a platform-aware connection address: for `wireless` an SSH host (`user@host` plus optional identity path) to the robot's IP; for `lite` an SSH host to the **host PC** that holds the Reachy via USB-C; for `simulation` no host (the behavior runs in the same Python process)
- **MUST** accept a hard timeout per run; on timeout the emergency-stop path triggers
- **SHOULD** accept a trigger mode: `autonomous` (behavior runs without external triggers) vs. `interactive` (HA event drives steps, optionally via `home-assistant-bridge` patterns)
- **MAY** accept additional options: dry-run (deploy without run), watch-only (no deploy), verbosity of the summary

### Platform profiles

The agent distinguishes the three Reachy Mini platforms because deploy path, telemetry availability, and safety thresholds differ materially:

- **Reachy Mini** (Wireless) — self-contained with RPi 4 CM4 + LiFePO4 battery. Deploy directly to the robot's IP via SSH or via the daemon REST API. Full telemetry: IMU (accelerometer, gyroscope, quaternion, temperature), battery polling, daemon-published joint positions / head pose at 50 Hz. Emergency-stop triggers may include current spike, IMU temperature threshold, and battery brown-out.
- **Reachy Mini Lite** — tethered to a host PC via USB-C, external 6.8–7.6 V supply. Deploy primarily through the host PC: SSH to the host, talk to the Pollen daemon there — **not** SSH directly to the Reachy. The actuator set is identical to Wireless, but **no IMU telemetry** and **no battery sensor** — safety thresholds come from Stewart joint limits (URDF) and from daemon-published effort / current data when available. Missing IMU data is not a FAIL criterion.
- **Simulation** — `ReachyMini(use_sim=True)` in the same Python process. **No deploy needed**, no SSH, no USB. Fully deterministic. No real sensor telemetry (apart from pose read), no audio playback, no LED effects. PASS/FAIL focuses on pose reachability, move lifecycle, and logic — not on physical-world effects.

Requirements:

- **MUST** read the platform from the input on behavior start and verify it against `DaemonStatus` — a mismatch between input and observed platform is a FAIL
- **MUST** apply platform-specific PASS/FAIL criteria: "IMU telemetry missing" is **not** a FAIL on Lite or Simulation; on Wireless it is
- **MUST** state the deploy path explicitly in the output summary: `via_ssh_direct` (Wireless), `via_host_usb` (Lite), `in_process` (Simulation)
- **MUST** mark on simulation runs which acceptance criteria could **not** be checked (e.g. real pose reach, audio sync, servo heat) and emit a `not_applicable_in_simulation` list in the output protocol
- **MUST NOT** treat Lite telemetry gaps (missing IMU / battery data) as "sensors offline" — that is the norm, not a defect

### Lifecycle
- **MUST** run the lifecycle in this order: connect → sync code → install deps → start behavior → watch & sample → stop → disconnect
- **MUST** execute the `connect` step platform-specifically: `wireless` SSH to the robot's IP, `lite` SSH to the host PC plus a daemon API probe, `simulation` is a no-op (the `use_sim=True` constructor provides the connection in-process)
- **MUST** skip the `sync code` and `install deps` steps in simulation (no deploy needed)
- **MUST** record per-phase outcomes structurally (phase, status, duration, error class if any)
- **MUST** terminate cleanly on disconnect or unexpected behavior exit — no hanging SSH sessions, no orphaned behavior processes
- **SHOULD** insert a health check between phases — Wireless with IMU temperature and battery state; Lite with daemon effort / current data when available; in simulation the check is omitted

### Emergency stop
- **MUST** offer an emergency-stop path that cleanly terminates the behavior process and brings the device into `INIT_HEAD_POSE` (4×4 identity matrix, head centred) plus `INIT_ANTENNAS_JOINT_POSITIONS` — pose constants verified in `reachy_mini.py`
- **MUST** allow platform-specific trigger sources for emergency stop:
  - **Wireless**: IMU temperature threshold (`mini.imu["temperature"]`), battery brown-out, current spike (daemon-published), motion-limit violation
  - **Lite**: daemon-published effort / current data when available, motion-limit violation; **no** IMU or battery triggers
  - **Simulation**: only logic triggers (pose outside URDF range, hook exception, timeout); no physical emergency stop needed, but execute the pose reset anyway for consistency
- **MUST** trigger emergency stop without user confirmation when a platform-appropriate safety threshold is hit
- **MUST** call out the emergency stop as a distinct event in the output protocol, including trigger source and platform
- **MUST NOT** reduce the emergency stop to a log note on Wireless or Lite — physical consequence trumps logging

### Output / output format
- **MUST** return only a structured summary into the main thread: overall status (`PASS` / `FAIL` / `ABORTED`), hook statistics (which hooks ran, how often, mean latency), anomaly list, duration
- **MUST** drop the full-text log into `.audits/on-device/<ISO-timestamp>-<behavior-name>.log` and name the path in the summary
- **MUST NOT** return raw logs or sensor streams to the main thread
- **SHOULD** also ship a machine-readable side artifact (e.g. JSON with the same data) when downstream automation is foreseen

### Security and secrets
- **MUST** read SSH / device credentials from the environment or from `ssh_config`, never from a plaintext argument
- **MUST NOT** write credentials into the output log or summary — masking is mandatory if an identifier notation is needed
- **SHOULD** keep SSH host-key verification on; on first connect, surface the fingerprint in the output rather than auto-accept
- **MUST** apply platform-aware auth models:
  - **Wireless**: SSH directly to the Reachy (Pi OS user); daemon token over the same path
  - **Lite**: SSH to the host PC (operator user); plus the Pollen daemon token, forwarded from the host to the daemon
  - **Simulation**: no SSH auth, no daemon token — everything runs in the same Python process

### Boundaries
- **SHOULD** point at `reachy-mini-sdk` whenever the main thread needs to adjust motion idioms after the test
- **SHOULD** point at `behavior-scaffold` when the test reveals the behavior is structurally incomplete
- **SHOULD** point at `home-assistant-bridge` when `interactive` mode is to run against real HA events
- **MUST NOT** duplicate content from those skills — the agent is observer and orchestrator, not knowledge base

## Acceptance Criteria
- [ ] The agent reads the platform (`wireless` / `lite` / `simulation`) from input and verifies it against `DaemonStatus`
- [ ] The output report names the deploy path explicitly (`via_ssh_direct` / `via_host_usb` / `in_process`)
- [ ] Simulation runs include a `not_applicable_in_simulation` list of skipped criteria
- [ ] Emergency-stop triggers are configured platform-specifically (IMU/battery only Wireless, effort data on Lite, logic triggers everywhere)
- [ ] Lite telemetry gaps (missing IMU / battery) do not produce a FAIL
- [ ] The agent exists at `agents/reachy-mini-on-device.md` with valid frontmatter — `name: reachy-mini-on-device`, `description`, `distribution: plugin`, optional tags
- [ ] The `description` activates on phrasings like "test the behavior on the device", "deploy and run X on Reachy Mini", "live-trial behavior <name>"
- [ ] A skill-vs-agent rationale is visible in the agent body (at least tool-session length, context volume, orchestration)
- [ ] Lifecycle phases (connect / deploy / install / start / watch / stop / disconnect) are documented in the body
- [ ] Emergency-stop behavior and default safety thresholds are documented (with TBD markers where hardware verification is needed)
- [ ] Input parameters (behavior path, device address, timeout, trigger mode) are documented
- [ ] The output format is documented as a strict schema; raw logs land in `.audits/on-device/<timestamp>-<name>.log`
- [ ] `.audits/` is in `.gitignore` so logs never get committed
- [ ] References to `reachy-mini-sdk`, `behavior-scaffold`, `home-assistant-bridge` are visible in the body
- [ ] The agent is accepted by the skill/agent catalog generator (frontmatter valid, `name` matches filename, `distribution` set)
- [ ] Statements without hardware verification carry a `⚠ TBD: validate against real hardware` marker

## Open Questions
- Which deploy protocol is canonical — `rsync` over SSH, `scp`, a device-specific tool, or does the SDK support remote-run directly?
- Which telemetry format does the SDK emit (events, sample streams, log lines)? The output schema depends on it.
- Which minimal rest pose is safe for the emergency stop? Proposal: all joints centred, antennas neutral. Confirm before first hardware run.
- Which safety thresholds (current, temperature, motion limits) are available out of the box, and which must we measure inside the agent?
- Should the agent support multi-run comparisons (two runs compared to detect regressions), or strictly one run per invocation?
- How does the agent integrate with CI? Proposal: real-hardware runs only locally or on a hardware-attached runner; CI runs only in dry-run mode.
- How does the agent behave when the SDK itself ships a test / mock layer — does it fall back to that instead of expecting real hardware? Leaning: no, mock belongs to a separate skill.
- What maximum size may the log artifact reach before rotation / trim strategies kick in?
