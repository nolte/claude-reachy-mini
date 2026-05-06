# Deploy Agent for Reachy Mini Apps

Status: draft

## Context
Once a Reachy Mini app is structurally complete (scaffolded via `app-scaffold`, behavior code reviewed against `reachy-mini-sdk` idioms), it must reach the Pollen-daemon environment of a real Reachy Mini device before any further activity — live trial, integration test, demo. By hand that means: SSH to the device (or to the Lite host PC), confirm no other app is currently running, rsync the repo into a deploy target, install the app into the daemon's Python environment, verify the `reachy_mini_apps` entry-point is discoverable. These steps are sequential, latency-bound, and produce hundreds of lines of installer / rsync output that would clog the main thread of a Claude Code conversation. The `reachy-mini-deploy` agent encapsulates that lifecycle in its own tool session and hands the main thread a tight structured summary. It deploys, it does not run — running the app is `reachy-mini-start` (skill, with user prompts) or `reachy-mini-on-device` (agent, full trial lifecycle).

## Goals
- An app reaches the real Pollen-daemon environment in a single agent invocation
- Pollen contract violations are caught locally before any device side-effect
- Install output and dependency-resolution noise are isolated to a log artifact, never the main context
- The agent stays narrow: deploy and verify, no run, no behavior change
- The result returns as a tight PASS / FAIL / ABORTED summary with a pointer to a full-text log artifact
- Multiple consumers (the user directly, the `reachy-mini-on-device` trial agent's caller, an automation hook) can call this agent with the same input shape

## Non-Goals
- Running the deployed app — owned by `reachy-mini-start` (skill) and `reachy-mini-on-device` (agent)
- Hardware bring-up (separate skill planned)
- Firmware flashing (separate skill planned)
- Behavior / motion code editing (`reachy-mini-sdk`, `app-scaffold`)
- Hugging Face publishing (`reachy-mini-app-assistant publish`, or a future `behavior-publish-hf`)
- Persistent watchdog / auto-redeploy operation — the agent runs a single-shot deploy lifecycle, not a daemon
- Pollen daemon restart, reload, or reconfiguration — out of scope, never performed by this agent
- Simulation: there is nothing to deploy in `use_sim=True`; simulation runs are the on-device agent's territory

## Skill-vs-Agent rationale
This concern is modelled as an **agent** rather than a skill because several rationales from `nolte-shared/spec/claude/skill-vs-agent/` apply at once:

- **Multi-stage orchestration with own failure modes** — pre-flight, connect, busy-check, discover venv, sync, install, verify, disconnect. Each phase has distinct error signatures (auth error, lock contention, missing venv, dependency conflict, broken entry-point) that need own recovery paths.
- **Latency-bound tool session** — `rsync`, `pip install`, and `ssh` round-trips dominate; running them inline would block the main conversation for tens of seconds per phase.
- **Context-window protection** — installer logs and dependency-resolution traces routinely run hundreds of lines; the agent reduces them to a structured summary and writes the full text to `.audits/deploy/`.
- **Narrow tool surface** — Bash for `ssh` / `rsync` / `curl`, plus `Read` / `Glob` / `Grep` on the local repo. No edit access to the app under deployment.
- **Distribution: `plugin`** — the agent ships with the plugin alongside `reachy-mini-on-device`.
- **Counter-dimension** — interactive mid-flow confirmations (e.g. "another app is running, stop it?") are deliberately given up; the agent's `if_busy: abort` default never force-takes the daemon. When an interactive decision is needed, callers reach for the `reachy-mini-start` skill instead.

## Requirements

### Inputs
- **MUST** accept `app_path` — a local directory of the app repo containing `pyproject.toml` with a `reachy_mini_apps` entry point and an HF-conformant `index.html` per Pollen contract
- **MUST** accept `device` — an SSH host. For `wireless` this is the robot itself (`pollen@reachy-mini.local` style); for `lite` it is the **host PC** that holds the Reachy via USB-C
- **MUST** accept `platform` — exactly one of `wireless` or `lite`. `simulation` is out of scope and **MUST** be rejected with a clear error
- **MAY** accept `mode` — `editable` (default, fast for dev iteration) or `release` (snapshot wheel install)
- **MAY** accept `verify` — boolean, default `true`; when false skip the post-install entry-point and import checks
- **MAY** accept `if_busy` — `abort` (default, never force-stops a third-party app) or `wait` (poll for up to 30 s)
- **MAY** accept `dry_run` — boolean, default `false`; when true sync only, no install, no verify
- **MUST NOT** accept plaintext credentials in any input — credentials come exclusively from `ssh_config` / environment

### Lifecycle
- **MUST** run the lifecycle in this order: local pre-flight → connect → robot-busy check → discover deploy target → sync code → install → verify → disconnect
- **MUST** run `reachy-mini-app-assistant check <app_path>` from the local environment in the pre-flight phase; a contract violation aborts the lifecycle before any device side-effect
- **MUST** run a robot-busy check via the Pollen daemon (`/api/apps/current-app-status` and `/api/daemon/robot-app-lock-status`) before install. When another app holds the lock, the action depends on `if_busy`; `abort` is the default and the agent **MUST NOT** auto-stop a third-party app
- **MUST** discover the Pollen-daemon Python interpreter on the device rather than hard-coding it; record the resolved path in the report. Acceptable resolution order: documented venv path → `which reachy-mini-app-assistant` shebang → fail with a clear pointer to Pollen's daemon-installation docs
- **MUST NOT** install into the device's system Python under any circumstance
- **MUST** run `rsync` with safe excludes (`.git`, `.venv`, `__pycache__`, `.audits`, `*.egg-info`, `.pytest_cache`, `.ruff_cache`, `node_modules`) and **MUST** scope `--delete` to the deploy target only
- **MUST** verify, when `verify=true` and `dry_run=false`, that the deployed app appears in `entry_points(group='reachy_mini_apps')` queried from the Pollen-daemon Python interpreter, and that the entry-point's package imports without raising
- **MUST** record per-phase outcomes structurally (phase, status, duration, error class if any)
- **MUST** terminate cleanly on disconnect — no hanging SSH sessions, no orphaned processes on the device
- **MUST NOT** restart, reload, or reconfigure the Pollen daemon as part of this lifecycle
- **MAY** prefer `uv pip` over `pip` when `uv` is on the device PATH; both are accepted

### Output / output format
- **MUST** return only a structured summary into the main thread: overall status (`PASS` / `FAIL` / `ABORTED`), per-phase outcomes, app name, entry-point, deploy path, log-artifact pointer
- **MUST** distinguish `ABORTED` (could not proceed for an external reason — busy lock, SSH auth, missing venv — no install attempted) from `FAIL` (an install or verify step ran and failed)
- **MUST** drop the full-text log into `.audits/deploy/<ISO-timestamp>-<app-name>.log` and name the path in the summary
- **MUST NOT** return raw logs, dependency-resolution traces, or credentials to the main thread
- **SHOULD** include a `follow_ups` field listing the natural next steps (typically: "dispatch `reachy-mini-on-device` for a live trial" or "invoke `reachy-mini-start` to bring the app online")

### Security and secrets
- **MUST** read SSH / device credentials from environment or `ssh_config`, never from a plaintext argument
- **MUST NOT** write credentials into the output log or summary; mask any identifier mentioned
- **SHOULD** keep SSH host-key verification on; on first connect, surface the fingerprint in the output rather than auto-accept
- **MUST** apply platform-aware auth models:
  - **Wireless**: SSH directly to the Reachy
  - **Lite**: SSH to the host PC; the Pollen daemon there receives the install through that session
- **MUST** ensure `.audits/` is in the consuming repo's `.gitignore` before writing artifacts there; the audit folder is generated, never committed

### Boundaries
- **SHOULD** point at `app-scaffold` when the local pre-flight reveals the app skeleton itself is broken or missing
- **SHOULD** point at `reachy-mini-sdk` when the local pre-flight reveals SDK-pin or import issues that belong to the behavior side
- **SHOULD** point at `reachy-mini-start` (skill) as the natural follow-up once `verify` passes and the user wants the app to actually run
- **SHOULD** point at `reachy-mini-on-device` (agent) as the natural follow-up when a live trial with telemetry is wanted
- **MUST NOT** duplicate content from those artifacts — this agent is a deployment orchestrator, not a knowledge base
- **MUST NOT** dispatch sibling agents or call other Skills (forbidden by `spec/claude/skill-vs-agent/`)

## Acceptance Criteria
- [ ] The agent rejects `platform=simulation` with a clear error pointing at the on-device agent
- [ ] The agent runs `reachy-mini-app-assistant check <app_path>` locally as the first step and aborts on contract violation before connecting to the device
- [ ] The agent never auto-stops a third-party app on the daemon; `if_busy=abort` is the default behaviour
- [ ] The agent never installs into the device's system Python; resolution to a Pollen-daemon venv is mandatory
- [ ] The agent never restarts the Pollen daemon
- [ ] The agent verifies, on `verify=true`, that the deployed app appears in `entry_points(group='reachy_mini_apps')`
- [ ] The output report names the deploy path explicitly (`via_ssh_direct` / `via_host_usb`)
- [ ] The output report distinguishes `ABORTED` from `FAIL` per the rule above
- [ ] The full-text log lives at `.audits/deploy/<timestamp>-<app-name>.log` and `.audits/` is in the consuming repo's `.gitignore`
- [ ] The agent exists at `agents/reachy-mini-deploy.md` with valid frontmatter — `name: reachy-mini-deploy`, `description`, `distribution: plugin`, optional tags
- [ ] The `description` activates on phrasings like "deploy to the reachy", "rollout to reachy-mini.local", "ship the app to the device", and the equivalent German variants
- [ ] A skill-vs-agent rationale is visible in the agent body (at least multi-stage orchestration, context-window protection, narrow tool surface)
- [ ] References to `reachy-mini-start`, `reachy-mini-on-device`, `app-scaffold`, `reachy-mini-sdk` are visible in the body
- [ ] Statements without hardware verification carry a `⚠ TBD: validate against real hardware` marker

## References
- Pollen contract validator (used in pre-flight): `reachy-mini-app-assistant check <path>` (shipped with the `reachy-mini` package)
- App lifecycle contract (`stop_event`, `wrapped_run`, app manager): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Daemon REST surface (busy-check endpoints, installed-apps listing): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollen `AGENTS.md` (entry-point group, app conventions): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Sibling agent for live trial runs: `agents/reachy-mini-on-device.md`
- Sibling skill for starting the deployed app: `skills/reachy-mini-start/SKILL.md`

## Open Questions
- Which exact REST endpoint lists the apps the daemon currently sees registered? Confirm `/api/apps/installed` (or the actual path) on first hardware contact and pin in the agent body
- What is the canonical deploy target on Wireless — `~/apps/<name>/` under the `pollen` user, or a daemon-managed location? Confirm and pin
- Does the Pollen daemon refresh its `entry_points(group='reachy_mini_apps')` view automatically after a `pip install`, or is a daemon SIGHUP / restart needed for the new entry-point to surface? If a daemon restart is required, this agent must not perform it; the user (or `reachy-mini-start`) does
- Does Pollen's daemon-installation expose a stable env var or path file for the venv, or must we always probe? Probing is the safe default; an env var would simplify the report
- Should `dry_run=true` still run the verify step against the previously-installed version of the app (regression check), or strictly skip verify? Leaning: skip verify, since dry-run is for fast diff inspection
- For Lite: does the host-PC daemon expose the same `/api/apps/...` endpoints as the Wireless daemon, or is the API surface different? Confirm on first Lite hardware contact
