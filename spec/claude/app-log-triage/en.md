# App Log Triage Skill

Status: draft

## Context

While developing a Reachy Mini app, the developer needs to extract a failure class from the distributed log sources, classify it, and decide on the next diagnostic action. The [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) spec captures the knowledge required (three logger trees, two capture paths, platform sinks, Common Issues triage catalog, verify-basics-first heuristic). What is missing is the operational counterpart: applying that knowledge as a repeatable workflow. This `app-log-triage` skill is exactly that counterpart.

It is the reusable answer to the question "why isn't the app running, why isn't it doing what I expect". It **reads** logs, classifies them against the canonical Common Issues catalog of the knowledge spec, and returns a recovery recommendation with a pointer to the responsible skill / agent. It does **not** write logs, **not** fix code, **not** stop another app, and **not** flood the main context with raw logs — behavior fixes stay with the developer, large volumes belong to the [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) agent.

Term clarification: "triage" here means **fast classification of an observed failure class plus recovery recommendation**, not deep performance analysis, not a production incident postmortem, not hardware recovery.

## Goals

- A precise activation `description` that fires on typical triage occasions during app development
- Consistent multi-source log capture per platform and run mode, with default filters that suppress HTTP noise
- Deterministic classification of an observed failure against the Common Issues triage catalog of the knowledge spec
- A binding verify-basics-first fallback whenever the classification is uncertain
- A structured, compact report — classification + confidence + recovery proposal with skill / agent cross-refs
- Deliberately narrow: no behavior fixes, no hardware recovery, no autonomous remediation, no production logging

## Non-Goals

- Behavior code fixes (app logic stays with the developer; SDK knowledge lives in [`reachy-mini-sdk`](../reachy-mini-sdk/en.md))
- Hardware recovery (Pollen's hardware troubleshooting docs at <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting>)
- Production log analysis on provisioned hosts ([`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md))
- On-device test lifecycle including bulk log sampling — the [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) agent owns that, with its own shape for large volumes
- Performance profiling, tracing, flame graphs
- Log persistence / search / indexing back-ends — the skill consumes logs, it does not retain them
- A generic Python `logging` tutorial (knowledge lives in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md))

## Requirements

### Trigger and activation

- **MUST** carry a `description` that activates Claude Code on phrasings like "triage app logs", "why isn't the app running", "classify this failure", "diagnose this crash", "check what went wrong with the app", "log triage for Reachy Mini app"
- **MUST** include the keywords in the `description`: log, triage, debug, app, Reachy Mini, classify, diagnose, failure
- **SHOULD** explicitly call out when _not_ to activate: when the user wants a pure on-device test (that belongs to [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md)), when it concerns production-host triage (that belongs to [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md)), when it is pure behavior / move composition (that belongs to [`reachy-mini-sdk`](../reachy-mini-sdk/en.md)), when it is hardware recovery (Pollen docs)

### Input parameters

- **MUST** accept a platform hint (`wireless` / `lite` / `simulation`); when missing, the skill **MUST** derive it from the environment (e.g. mDNS lookup on `reachy-mini.local`, local daemon port, or via a clarifying question to the user)
- **MUST** accept a run-mode hint (`daemon-hosting` / `direct` / `pytest`); without a hint, the default is `direct` — rationale: per [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md), this is the recommended development path
- **SHOULD** accept a time anchor (`since`) in `journalctl --since` format; without a hint, the default is `5 min ago`
- **SHOULD** on Wireless accept a hostname hint (default `reachy-mini.local`, per Pollen's mDNS convention)
- **SHOULD** accept an observation symptom in words (e.g. "robot doesn't move", "connection refused", "audio fails") — sharpens the classification but is optional

### Pre-flight (every run, before any log action)

- **MUST** verify that the expected log source is reachable:
  - **Wireless**: SSH reachability for `pollen@<host>` plus `systemctl status reachy-mini-daemon.service` as the daemon heartbeat
  - **Lite**: local daemon-process status (e.g. `pgrep -f reachy-mini-daemon` or a daemon port probe `lsof -i :8000`)
  - **Simulation**: no pre-flight — same process
- **MUST** on a failed reachability check, return failure class `connection-refused` (see Classification) **directly**, **without** attempting a log capture — rationale: with no reachable daemon there are no meaningful logs to classify
- **MUST NOT** the pre-flight step (re)start the daemon on its own — daemon restart is a recommendation, never an action (see out-of-scope)

### Log capture

- **MUST** select the canonical log source per platform and run mode from [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) § Platform profiles — do not invent sources, do not deviate from the spec
- **MUST** apply the default filter `grep -v "uvicorn\|GET \|POST "` to suppress REST-surface HTTP noise — same convention as [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) §67–71
- **SHOULD** in `daemon-hosting` mode read all three logger trees (`reachy_mini.*`, `reachy_mini.daemon.*`, `reachy_mini.apps.manager.runner`) jointly, because an app action typically leaves traces across multiple trees
- **MUST NOT** the skill emit raw logs into the main context — the capture is delivered as a structured aggregate (see Output / Report)
- **MUST NOT** attempt log capture when the pre-flight reachability check has failed
- **MUST NOT** pull more than the last two hours of logs without confirming the time anchor with the user first (volume guard)

### Classification

- **MUST** match against every class in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) § Common Issues triage catalog: `connection-refused`, `app-lock-held`, `no-motion`, `jerky-motion`, `import-error`, `audio-fail`, `motors-different-states`
- **MUST** use the log pattern named in the knowledge spec for each class, **not** a free heuristic (e.g. for `app-lock-held` exactly the pattern `RobotAppLock: rejected — held by <app_name>` via the `reachy_mini.daemon.robot_app_lock` logger)
- **SHOULD** when several classes match, return the most specific one with the highest confidence and list the alternatives as `candidates` with lower confidence
- **MUST** on no match, return the pseudo class `unclassified`, **not** a newly invented class
- **SHOULD** on `unclassified` add a hint that the symptom is an open question for the `app-logging` spec and should be reconciled against Pollen's sources ([`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md))

### Verify-basics-first heuristic

- **MUST** on class `unclassified` propose the `examples/minimal_demo.py` sanity check as the first recovery step — per the MUST clause in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) ("before diagnosing any app-specific failure class, run minimal_demo.py first")
- **SHOULD** on class `no-motion` additionally recommend the sanity check as the first step — when `minimal_demo.py` itself produces no motion, the problem is connectivity / hardware, not app logic
- **MUST NOT** the skill execute the sanity check on its own — that is a developer action; the skill recommends, lists the exact command, and reports the outcome on the next run

### Report format

- **MUST** return a compact, structured report with the sections: (1) detected platform / mode, (2) sources sampled (which logger trees / sinks), (3) primary class with confidence, (4) candidate classes with confidence, (5) expected log pattern (citation from the knowledge spec), (6) match lines with ±2 context lines, (7) recovery recommendation with skill / agent cross-ref
- **MUST** carry per-class Pollen source references over from the knowledge spec, **not** cite them anew
- **SHOULD** on class `motors-different-states` point directly to the safe-torque pattern in Pollen's [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md)
- **SHOULD** on class `app-lock-held` extract the holding app name from the match pattern and surface it in the report
- **MUST NOT** include raw logs beyond the match lines plus context — the report is a summary, not a log dump
- **MUST NOT** include tokens, HF auth keys, WiFi credentials, or sensor data with personally identifiable context (consistent with [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) MUST NOT on PII)

### Out-of-scope clarification

- **MUST NOT** modify behavior code — that is the developer's job, possibly with knowledge from [`reachy-mini-sdk`](../reachy-mini-sdk/en.md)
- **MUST NOT** (re)start the daemon on its own — the recommendation lands in the report; the user executes
- **MUST NOT** stop a second app to resolve an app-lock conflict — recommended, not executed
- **MUST NOT** propose hardware recovery beyond the safe-torque pattern — mic FPC cable, motor replacement, spherical-joint maintenance belong in Pollen's hardware docs
- **SHOULD** on large log volumes (e.g. multi-hour sessions, bulk triage across multiple failures) point the user at the [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) agent — that is the shape for device-side bulk processing with artifact persistence

## Acceptance Criteria

- [ ] The skill lives at `skills/app-log-triage/SKILL.md` with valid frontmatter (`name: app-log-triage`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The `description` carries the keywords (log, triage, debug, app, Reachy Mini, classify, diagnose, failure) and explicitly calls out at least three anti-triggers
- [ ] The pre-flight reachability check runs before every log capture and on failure returns class `connection-refused` directly, without further log action
- [ ] The default filter `grep -v "uvicorn\|GET \|POST "` is applied to every capture that actually runs (`connection-refused` skips the capture, and the filter with it)
- [ ] The Common Issues classification covers all seven classes from [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) and uses the log patterns from there verbatim
- [ ] On a match, the skill reports class + confidence + expected log pattern + match lines with context + recovery proposal
- [ ] On no match, `unclassified` is returned with the `minimal_demo.py` sanity check as the first proposal
- [ ] The report contains no raw logs beyond match lines ±2 context lines
- [ ] PII clause is honored: no tokens / auth keys / WiFi credentials in the report
- [ ] Cross-refs to [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md), [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md), [`claude/reachy-mini-sdk`](../reachy-mini-sdk/en.md), Pollen's [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md), and [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) are visible
- [ ] The skill aborts or cleanly redirects when the request falls into a neighbouring skill / agent's scope (on-device test, production, hardware recovery)
- [ ] `pre-commit run --all-files` passes on the skill file

## References

> Cross-refs to internal knowledge specs are linked; Pollen code-source anchors, when used, are verified against `pollen-robotics/reachy_mini@main` — convention from [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) § References.

- Knowledge spec (canonical source for the platform table, Common Issues catalog, verify-basics-first, PII clause): [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md)
- On-device test agent (responsible for device-side bulk triage and the test lifecycle): [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md)
- SDK knowledge base (idiomatic SDK use, `deep-dive-docs` MUST): [`claude/reachy-mini-sdk`](../reachy-mini-sdk/en.md)
- Production logging on provisioned hosts (clear demarcation): [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md)
- Pollen skill `debugging` (Common Issues source, verify-basics-first heuristic): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- Pollen skill `safe-torque` (recovery pattern for motor state mismatches): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Pollen's `AGENTS.md` (entry point, general conventions): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Pollen example `minimal_demo.py` (canonical sanity check): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>

## Open Questions

- Skill vs. agent split: today [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) handles device-side test lifecycles including log tail. For long-lived triage sessions (multi-hour, multi-failure) — should that move into a dedicated `reachy-mini-log-tail` agent, or is the skill-level pointer to `reachy-mini-on-device` enough?
- Stderr heuristic from [`apps/manager.py:206–209`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py): should the skill replicate the exact marker strings to make stderr classification deterministic, or is the coarse Pollen pointer enough?
- Report format: Markdown with a table section (chat-readable) or a structured JSON block (parseable for follow-up skills)? Proposal: Markdown as default, JSON mode behind an optional parameter when a follow-up skill consumes it.
- Auto platform detection: is there a reliable indicator (e.g. does `nc -z reachy-mini.local 8000` answer? local daemon port?), or does the platform hint stay mandatory?
- Sample accumulation for long sessions: does that live in this skill (stream window, last n minutes) or in the proposed bulk agent?
- Should the skill optionally propose a `mockup-sim` variant (no MuJoCo / GStreamer) as a sanity-check fallback when sim dependencies are missing?
- Match-line context: is "±2 context lines" the right default, or is ±5 with a tighter line cap per line better?
- Permissions: does the skill need `sudo` rights for `journalctl` on Wireless, or is user read access sufficient given Pollen's default configuration? Verify against Pollen's docs.
