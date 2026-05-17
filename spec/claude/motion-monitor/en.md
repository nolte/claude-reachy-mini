# Motion Monitor Agent for Reachy Mini

Status: draft

## Context

When a Reachy Mini runs a behavior session, any of the four anomaly classes from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md) can appear at any time — head-against-body self-collision (Class A), jerky motion (Class B), antenna wobble (Class C), Stewart-limit knock (Class D). The spec defines *what* to detect, but not *who* watches it at runtime. The `motion-monitor` agent fills that gap: it polls the daemon REST API at 10 Hz, correlates the reads with `journalctl --user` output from the Pollen daemon and the currently running app, classifies every anomaly per the four canonical classes, and writes a structured anomaly-event-record log under `~/.cache/reachy-mini-monitor/`. At the end of a bounded 5–15-minute session it returns a tight summary to the main thread — findings per class, with a pointer to the full-text log artifact.

This task is modelled as an **agent** because it runs long, processes thousands of telemetry points, and the main thread should see neither the polling detail nor the raw output. The agent is read-only over the REST API (only `GET`), it never triggers a recovery action (no daemon restarts, no `stop-current-app` calls), and it never moves the robot — the strict separation of detection from recovery from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md) is preserved.

Consumed knowledge specs: [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md) (classification, detect signals, event-record schema), [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/en.md) (endpoint inventory), [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md) (URDF limits, three-layer validity, `_status.ready` bug from Layer 5), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) (triage classes for post-hoc patterns).

## Goals

- A bounded live-monitoring session runs with a single agent invocation and returns a structured summary
- The four anomaly classes from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md) are detected at runtime and reported in the binding event-record format
- Three log sources are evaluated in parallel: Pollen daemon journal (`journalctl --user -u reachy-mini-daemon.service`), app log (`/api/apps/current-app-status.error` plus daemon-forwarded app stdout), and the agent's own audit log under `~/.cache/reachy-mini-monitor/<session-id>.jsonl`
- The agent stays strictly read-only and mutates neither the daemon nor the app — detection stays cleanly separated from recovery
- A session runs bounded between 5 and 15 minutes; on timeout it ends in a controlled manner and the audit log is closed
- One audit log file is written per session under `~/.cache/reachy-mini-monitor/<YYYY-MM-DDTHH-MM-SS>.jsonl`, one event per line, JSON Lines format

## Non-Goals

- Continuous / production watchdog — the agent is a bounded test lifecycle, not a systemd daemon. An "always-on" monitor belongs in a separate tool (e.g. a variant of `reachy-mini-mcp-server` with a streaming endpoint)
- Auto-recovery or auto-mitigation — the agent reports, the agent does not restart. Restart logic belongs in a dedicated recovery skill, not here
- Daemon mutations: no `POST /api/daemon/start`, no `POST /api/apps/stop-current-app`, no `set_mode/*`, no app-lock force-release. Not even on failure
- Motion composition or pose targeting — the agent observes, it does not trigger motion
- App-lifecycle management — app start belongs to [`reachy-mini-start`](../reachy-mini-start/en.md), app deploy to [`reachy-mini-deploy`](../reachy-mini-deploy/en.md), live-trial test to [`reachy-mini-on-device`](../reachy-mini-on-device/en.md)
- Hardware bring-up, calibration, firmware flash — separate skills (planned)
- Audio, vision, or LED anomalies — other subsystems; this spec is motor / motion focused
- Definition of the anomaly classes themselves — those come from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md), not from here

## Skill-vs-Agent Rationale

This task is modelled as an **agent** rather than a skill because several arguments from the skill-vs-agent trade-off apply simultaneously:

- **Long runtime** — A session runs bounded 5–15 minutes, with polling every 100 ms (10 Hz). Skills are optimised for interactive inline workflows; a long observe-then-report lifecycle belongs in its own tool session.
- **High telemetry volume** — 10 Hz × 900 s = up to 9,000 state reads per 15-minute session, plus parallel `journalctl --follow` streams from two sources. In the main thread that would consume context; the agent reduces it to a summary plus a file artifact.
- **Fire-and-forget lifecycle** — the caller sets the parameters once (duration, platform, class filter), the agent runs, the agent reports. Mid-flow tweaks during the run are not foreseen.
- **Dedicated tool set** — the agent needs Bash for `journalctl`, HTTP calls (via `curl` or `httpx`), and JSON-Lines writes to disk. These tools don't belong in the main thread, which is typically code-editing dominated.
- **Specialised behaviour** — anomaly classification across four classes, three-way liveness cross-check (see [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md) Layer 5), structured event-record schema. A dedicated agent canonicalises that.
- **Distribution: `plugin`** — the agent belongs to the plugin and is distributed with it.

## Requirements

### Inputs

- **MUST** accept the platform (`platform`: `wireless` / `lite` / `simulation`); on `simulation` the agent runs against a daemon in the same Python process and reads the app log via app-process stdout instead of `systemd journalctl`
- **MUST** accept the daemon address (`daemon_url`, default `http://127.0.0.1:8000`); on Wireless with the mDNS default `http://reachy-mini.local:8000`
- **MUST** accept the session duration (`duration_seconds`, default `300` = 5 min, max `900` = 15 min); values outside [60, 900] are rejected
- **MUST** accept the polling cadence (`poll_interval_ms`, default `100` = 10 Hz); values outside [50, 1000] are rejected — faster than 20 Hz puts unnecessary load on the daemon, slower than 1 Hz misses Class A detection
- **SHOULD** accept a class filter list (`watch_classes`, default `[A, B, C, D]`); empty list is rejected
- **SHOULD** accept an optional `audit_log_dir` (default `~/.cache/reachy-mini-monitor/`); missing write permission aborts, never silently substitutes `/tmp/`
- **SHOULD** accept an optional `severity_floor` (`hard` | `warn` | `info`, default `info`); events below the floor are written but not surfaced in the summary
- **MAY** accept the daemon journal service name (`daemon_service`, default `reachy-mini-daemon.service`) — Wireless default; on Lite the name can differ depending on host setup
- **MUST NOT** allow unbounded runtime — `duration_seconds=0` or values > 900 are rejected inputs

### Platform Profiles

| Platform | Daemon log source | App log source | Polling endpoint | Constraints |
|---|---|---|---|---|
| **`wireless`** | `journalctl --user -u <daemon_service> --follow` via SSH or locally on the Reachy | `/api/apps/current-app-status` plus app stdout via daemon forward | `http://reachy-mini.local:8000/api/state/full?with_head_joints=true` | Full Class A detection via `backend.ready` flip |
| **`lite`** | `journalctl --user -u <daemon_service>` on the host PC | `/api/apps/current-app-status` plus app stdout via daemon | `http://127.0.0.1:8000/api/state/full?with_head_joints=true` | Classes A–D verifiable; **not yet hardware-validated** (see Open Questions) |
| **`simulation`** | App-process stdout directly (no systemd) | App-process stdout (same source) | `http://127.0.0.1:8000/api/state/full?with_head_joints=true` | Classes A and C are **not** detectable in sim (no real Stewart mechanics, no servo hardware); only Classes B and D run meaningfully |

Requirements:

- **MUST** validate the platform input against `GET /api/daemon/status.wireless_version` at session start — a mismatch is a FAIL and ends the session before the first poll
- **MUST** apply platform-specific filters to class detection: Classes A and C are automatically skipped on `simulation` and marked `not_applicable_in_simulation` in the report
- **MUST NOT** treat missing IMU telemetry on Lite as a defect — that's expected (see [`reachy-mini-on-device`](../reachy-mini-on-device/en.md) §Platform Profiles)

### Lifecycle

- **MUST** execute the lifecycle in this order: pre-flight → session-start → poll-loop → log-tail → graceful-stop → report → cleanup
- **MUST** in the **pre-flight** phase check five gates and abort on the first failure:
  1. **Daemon reachable** — `GET <daemon_url>/api/daemon/status` HTTP 200 within 3 s; one mDNS cold-start retry allowed
  2. **Platform match** — as above
  3. **Audit log path writable** — `audit_log_dir/<session-id>.jsonl` must be creatable
  4. **Daemon journal reachable** — `systemctl --user is-active <daemon_service>` must return `active`; skipped on `simulation`
  5. **Current app session visible** — `GET /api/apps/current-app-status` must not return HTTP 5xx; `null` (no app running) is OK and noted in the report
- **MUST** in the **session-start** phase generate a unique session ID (`YYYY-MM-DDTHH-MM-SS_<random>`), open the audit log with a session-header event (platform, daemon URL, configuration, verification date from `daemon/status.version`), and start an async tail on the daemon journal
- **MUST** in the **poll-loop** phase read a state snapshot every `poll_interval_ms`, check it against the four classes, and write detected anomalies to the audit log
- **MUST** in the **log-tail** phase, in parallel with polling, check the log streams (`journalctl --follow` and app stdout) for known patterns (Class A: `ConnectionError: Could not connect to daemon on localhost`; Class D: `kinematics` exception traceback; see [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md))
- **MUST** in the **graceful-stop** phase cleanly terminate the poll loop and log tails after `duration_seconds` has elapsed (or on explicit caller cancellation), flush all buffered events to the audit log, and write a session-footer event (total duration, event counts per class)
- **MUST** in the **report** phase return a tight summary to the caller (schema in "Output Schema")
- **MUST** in the **cleanup** phase ensure all subprocesses are terminated and no file handles remain open

### Live Polling and Class Detection

Each poll reads the following fields and checks them against the four classes (class details and thresholds from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md)):

- **Class A** — `GET /api/daemon/status.backend_status.ready` flips to `false` AND `GET /api/state/full?with_head_joints=true.head_joints == null` AND no pose micro-drift between two consecutive reads ⇒ three-way cross-check positive ⇒ event `class=A, severity=hard`
- **Class B** — `‖Δjoints‖₂` between two consecutive `head_joints` reads per `Δt` larger than the URDF velocity threshold (`0.16 rad/sample` at 10 Hz → ~1.6 rad/s, so not every sample can exhaust the URDF velocity; at `poll_interval_ms=100` the threshold is `0.16 rad`) ⇒ `severity=warn`; > `0.30 rad/sample` ⇒ `severity=hard`
- **Class C** — standard deviation of antenna joint reads over a 2-second rolling window (= 20 samples at 10 Hz) `> 0.2°` despite a nominally stable setpoint ⇒ event `class=C, severity=warn`; Class C is skipped on `simulation`
- **Class D** — `backend_status.nb_error` spike (`> 0` and rising) after the last sent pose command, or `head_joints` diverges from `target_head_joints` by more than URDF-limit tolerance ⇒ event `class=D, severity=warn`

Requirements:

- **MUST** use the binding event-record schema from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md) for every detected anomaly (fields `class`, `phase`, `severity`, `detected_at`, `verification_basis` are mandatory)
- **MUST** set `phase` in the record to `live` (for polling detections) or `post-hoc` (for log-pattern matches)
- **MUST** populate `verification_basis` from `daemon/status.version` and the platform input — never invented
- **MUST NOT** classify a single `backend_status.ready=false` read on its own as Class A — the three-way cross-check rule from [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md) Layer 5 is mandatory
- **SHOULD** flag the Pollen daemon bug (`_status.ready` desync, see [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md) Layer 5) in the audit log once `backend_status.ready=false` persists longer than 10 s without the other two cross-checks (mean_freq, head_joints) confirming the lack of liveness

### Log Sources and Tail Mechanics

Three parallel log sources are evaluated during the session:

1. **Pollen daemon journal** — `journalctl --user -u <daemon_service> --follow --since <session-start>` (on `wireless` via SSH, on `lite` locally, omitted on `simulation`). Pattern-match for Class A indicators (`backend.ready: false` directly after `start-app`) and Class D indicators (kinematics exception tracebacks)
2. **App log** — `/api/apps/current-app-status.error` (status polling) plus the daemon-forwarded app stdout; pattern-match for `ConnectionError: Could not connect to daemon on localhost` (= Class A post-hoc per [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) triage class `daemon-stale-state`) and kinematics tracebacks (= Class D)
3. **Own audit log** — `audit_log_dir/<session-id>.jsonl`, JSON Lines format, one event per line

Requirements:

- **MUST** write every pattern match from sources 1 and 2 as an event into the audit log (source 3); never produce separate log files per source
- **MUST** use JSON Lines format for the audit log — one complete event line per `\n`, never buffer partial writes
- **MUST** write a session-header event as the first line and a session-footer event as the last line — header carries configuration and platform, footer carries per-class event counts and total duration
- **SHOULD NOT** automatically delete audit log files older than 30 days — cleanup is the caller's responsibility
- **MUST NOT** write PII or auth tokens into the audit log (PII clause from [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md))

### Emergency Stop and Bounds

- **MUST** on an internal failure (e.g. HTTP timeout persistent, `journalctl` subprocess crashes) write the session footer with `result=aborted` and populate `aborted_reason` in the report
- **MUST** on caller cancellation (e.g. SIGTERM, context cancel) execute graceful-stop rather than hard abort
- **MUST NOT** issue mutating daemon calls even on failure — no `stop-current-app`, no `restart`, no `set_mode/*`
- **MUST NOT** move the Reachy — the agent is detection, not correction

### Output Schema

The agent returns a tight summary (Markdown) plus the path to the audit log artifact:

```text
# Motion Monitor — Session <session-id>

Platform: <wireless|lite|simulation>
Daemon: <daemon_url>  (firmware <version>)
Duration: <actual_seconds> s of <duration_seconds> s budget
Result: <pass|warn|fail|aborted>

## Findings per class

- Class A (head-against-body): <count> events (<severities>)
- Class B (jerky motion):      <count> events
- Class C (antenna wobble):    <count> events  [skipped on simulation]
- Class D (Stewart-limit):     <count> events

## Notable patterns

- <one-line summary per pattern, e.g. "Class A live + Class A post-hoc match — daemon hung after start-app at 17:42:11">

## Audit log

Path: ~/.cache/reachy-mini-monitor/<session-id>.jsonl
Events: <total_event_count>
```

Requirements:

- **MUST** set `result` to exactly one of `pass` (no hard events), `warn` (warn events only), `fail` (at least one hard event), `aborted` (session ended prematurely)
- **MUST** include platform, daemon URL, firmware version, session duration, and audit-log path in the output — never omitted
- **MUST** explicitly mark non-applicable classes on `simulation` (A, C) as `[skipped on simulation]`
- **SHOULD** keep "Notable patterns" to at most 5 entries — on more findings point to the audit log

### Hard Rules

1. **Read-only.** The agent calls **only** `GET` endpoints on the Pollen daemon. No `POST`, no `PUT`, no `DELETE`. Not even on failure.
2. **No motion.** The agent never triggers a pose target, a move, or a mode change.
3. **No recovery.** A detected Class A anomaly is reported, **not** "fixed" by `stop-current-app` + restart. Recovery belongs in a dedicated skill.
4. **Bounded lifecycle.** Maximum `duration_seconds=900` (15 min). An infinite loop is not possible.
5. **Three-way cross-check for Class A.** A single `backend_status.ready=false` read does not suffice — the rule from [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md) Layer 5 is binding.
6. **Verification date.** Every event record carries `verification_basis` with platform + firmware + date.
7. **JSON Lines audit.** The audit log is JSON Lines, not a JSON array, so that an abrupt abort does not corrupt the file.

## Acceptance Criteria

- [ ] The agent exists at `agents/motion-monitor.md` with valid frontmatter (`name: motion-monitor`, `description`, `distribution: plugin`, `tools`, optional tags)
- [ ] The `description` names the three log sources (daemon journal, app log, own audit log) and all four anomaly classes
- [ ] Pre-flight checks five gates in the specified order, with a concrete failure action per gate
- [ ] Polling cadence default is 10 Hz (`poll_interval_ms=100`), maximum 20 Hz, minimum 1 Hz
- [ ] Bounded session: `duration_seconds` default 300, max 900, values outside [60, 900] are rejected
- [ ] Three-way cross-check for Class A is implemented: `backend.ready` + `head_joints` + pose micro-drift must all three indicate "no liveness" before Class A is reported
- [ ] Class C is skipped on `simulation` and marked `[skipped on simulation]` in the report
- [ ] Audit log is written in JSON Lines format under `audit_log_dir/<session-id>.jsonl`, framed by session-header and session-footer
- [ ] Every detected anomaly uses the binding event-record schema from [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md), all five mandatory fields present
- [ ] The agent issues **no** mutating daemon calls — not even on failure (Hard Rule 1+3)
- [ ] The `result` field in the report is exactly one of `pass` / `warn` / `fail` / `aborted`
- [ ] Cross-refs to [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md), [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/en.md), [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) are visible
- [ ] `pre-commit run --all-files` passes green on the agent file

## Sources

- Anomaly classification, detect signals, event-record schema: [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/en.md)
- REST endpoint inventory: [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/en.md)
- URDF limits, three-layer validity, `_status.ready` bug: [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/en.md)
- Triage classes for log pattern matches: [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md)
- Platform-profiles (Wireless / Lite / Simulation) template: [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md)
- Pollen SDK source (daemon status loop, joint reads): <https://github.com/pollen-robotics/reachy_mini>
- systemd `journalctl --user --follow` reference: <https://www.freedesktop.org/software/systemd/man/latest/journalctl.html>

## Open Questions

- Should the agent on `lite` use SSH to the host PC, or is local `journalctl` sufficient (when the host is the MCP client)? Recommendation: local `journalctl`, because the typical Lite setup has the host as the developer machine; SSH variant as optional in a later spec revision
- What is the exact upper bound for Class A latency between pose command and `backend.ready=false` flip in the failure path? The `motion-anomaly-detection` open question is load-bearing here; without a measured value the agent can only "wait long enough", which is an ergonomic weakness
- Should the agent produce an optional heatmap output per class over the session duration (e.g. Class B events aggregated every 10 s)? Proposal: no for v1, optional later
- How should the agent behave when the daemon restarts during the session? The `journalctl --follow` should survive that; the REST polls briefly get `connection refused`. Should the agent log the restart as `info` or as `warn`?
- Multi-Reachy setup: a developer has two devices (Wireless + Lite); should the agent manage a session group or stay one-Reachy-per-session? Proposal: one Reachy per session, multi-Reachy is the caller's job (two agent invocations)
- Audit log rotation: the agent currently writes one file per session under `~/.cache/reachy-mini-monitor/`. Should there be a maximum retained file count, or does cleanup stay caller-side? Recommendation: caller-side (also documented in the "Non-Goals" section)
- Lite platform: all four classes are hardware-verified only on Wireless (`motion-anomaly-detection` open question). Once Lite is on hand, the agent must be validated against the actual daemon-response shapes and servo-hardware idiosyncrasies
