---
name: app-log-triage
description: Triage logs from a Reachy Mini app session and classify the observed failure against the canonical Common Issues catalogue from `reachy-mini/app-logging`. Activate on phrasings like "triage app logs", "warum läuft die App nicht", "classify this failure", "diagnose this crash", "check what went wrong with the app", "log triage for Reachy Mini app". Do not activate for on-device test lifecycles (use the `reachy-mini-on-device` agent), production-host triage on provisioned hosts (use `host-provisioning`), behavior or motion development (use `reachy-mini-sdk` and `app-scaffold`), or hardware recovery (use Pollen's hardware-troubleshooting docs).
tags: [reachy-mini, log, triage, debug, diagnose, failure]
---

# App Log Triage

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-log-triage/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-log-triage/en.md).

Knowledge base: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-logging/de.md> — every classification, log pattern, and recovery hint in this skill mirrors that spec; this skill operationalises the knowledge.

## When this skill activates

Use this skill when the developer wants to:

- understand a single observed failure of an in-development Reachy Mini app
- get a structured classification of an "app doesn't run / doesn't behave" symptom against the canonical Common Issues catalogue
- bring multi-source log output (app code, SDK, daemon) into one short, actionable report

## When NOT to activate

- on-device test lifecycle (deploy, run, sample telemetry, return PASS/FAIL) → agent `reachy-mini-on-device`
- production-host triage (`reachy-app@<slug>.service` on a provisioned device) → `reachy-mini/host-provisioning`
- writing or editing behavior / motion code → developer's job, supported by `reachy-mini-sdk` and `app-scaffold`
- hardware recovery (mic FPC cable, motors, spherical joints) → Pollen's hardware-troubleshooting docs
- bulk log triage spanning hours or multiple failures → agent `reachy-mini-on-device` (handles big volumes with artefact persistence)

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `platform` | yes (auto-derive when missing) | — | `wireless` / `lite` / `simulation`. Auto-derive via mDNS lookup on `reachy-mini.local`, local daemon-port probe, or a clarifying question to the user. |
| `run_mode` | no | `direct` | `daemon-hosting` / `direct` / `pytest`. The default reflects `reachy-mini/app-logging`'s recommended development path (direct mode keeps app and SDK loggers in the same terminal). |
| `since` | no | `5 min ago` | `journalctl --since`-format time anchor for the log window. |
| `host` | no | `reachy-mini.local` | Hostname for `wireless`; per Pollen's mDNS convention. |
| `symptom` | no | — | Free-text description of the observed failure (e.g. "robot doesn't move", "connection refused", "audio fails"). Sharpens classification. |

## Hard rules

1. **Pre-flight reachability before any log capture.** Never tail logs from a host that may not exist or whose daemon is dead. On a failed reachability check, return failure class `connection-refused` directly — the lack of a daemon **is** the diagnosis.
2. **Never restart the daemon, stop another app, or modify behavior code.** All those are recommendations in the report, never actions taken by this skill.
3. **No raw log dumps.** The report carries match lines plus ±2 context lines, never streams. Bulk triage belongs to the `reachy-mini-on-device` agent.
4. **Classifications come from `reachy-mini/app-logging` § Common Issues, never invented.** On no match, return the pseudo class `unclassified` and propose the verify-basics-first sanity check.
5. **PII clause inherited.** No tokens, HF auth keys, WiFi credentials, or sensor data with identifying context in the report.

## Pre-flight (every run, in order — abort on first failure)

1. **Reachability** —
   - `wireless`: SSH probe (`ssh pollen@<host> true`) plus `systemctl status reachy-mini-daemon.service` for the daemon heartbeat
   - `lite`: local daemon presence (`pgrep -f reachy-mini-daemon`) or daemon port probe (`lsof -i :8000`)
   - `simulation`: no pre-flight needed (same process as the developer)
2. On failed reachability → emit class `connection-refused` and stop; do not attempt log capture.
3. **Volume guard** — if `since` would pull more than two hours of logs, confirm with the user before tailing.

## Log sources by platform and run mode

Source-of-truth tables live in [`reachy-mini/app-logging` § Plattform-Profile](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-logging/de.md). Apply the row that matches the inputs.

The default HTTP-noise filter is applied to every capture that actually runs:

```
... | grep -v "uvicorn\|GET \|POST "
```

Same convention as the `reachy-mini-on-device` agent's watch-and-sample step.

In `daemon-hosting` mode, read the three logger trees jointly (`reachy_mini.*`, `reachy_mini.daemon.*`, `reachy_mini.apps.manager.runner`) — an app action typically leaves traces across more than one tree.

## Classification

Match against the canonical Common Issues classes from [`reachy-mini/app-logging` § Common-Issues-Triage-Katalog](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-logging/de.md). The expected log pattern per class is the spec's text verbatim — do not paraphrase.

| Class | Expected log pattern (anchor) |
|---|---|
| `connection-refused` | `ConnectionRefusedError` in the app logger tree, or no daemon heartbeat in journald |
| `app-lock-held` | `RobotAppLock: rejected — held by <app_name>` via the `reachy_mini.daemon.robot_app_lock` logger |
| `no-motion` | `set_target` calls in the app logger without errors, but no motion; SDK logger raises no warning |
| `jerky-motion` | usually no clean log trace — almost always a code-path issue (single-owner loop violation; `goto_target` mixed with `set_target`) |
| `import-error` | `ModuleNotFoundError: reachy_mini` in the app subprocess stderr → surfaces as `runner.error` in the daemon log |
| `audio-fail` | `Failed to initialize media server` in the daemon logger; possibly a GStreamer warning |
| `motors-different-states` | sporadic effort / position out-of-range warnings in the daemon logger |

When several classes match, return the most specific one with the highest confidence and list the alternatives as `candidates` with lower confidence. On no match, return the pseudo class `unclassified` and propose the sanity check (next section).

**Motion-anomaly cross-classification.** When the triage class points at motion-side root causes (`jerky-motion`, `motors-different-states`, or a `connection-refused` after a previous `start-app`), the report **SHOULD** also map the symptom to one of the four canonical motion-anomaly classes from [`reachy-mini/motion-anomaly-detection`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motion-anomaly-detection/de.md) — this skill is the **post-hoc consumer** of that spec. Concretely: `connection-refused` after `start-app` plus a `ConnectionError: Could not connect to daemon on localhost` traceback in the app log is the post-hoc fingerprint of **Class A** (head-against-body self-collision); a `kinematics` exception traceback is **Class D** (Stewart-limit / IK-unsolvable). The skill emits the anomaly-event-record in the binding shape (`class`, `phase=post-hoc`, `severity`, `detected_at`, `verification_basis`) alongside the standard triage classification.

## Verify-basics-first heuristic

On class `unclassified`, the first recovery step in the report is always Pollen's verify-basics-first heuristic — run `examples/minimal_demo.py` against the same daemon, in the same run mode. The skill **recommends**; the developer **executes**. If the sanity check fails too, the failure belongs to a different class than the original symptom suggested.

On class `no-motion`, additionally recommend the sanity check as the first step — when `minimal_demo.py` itself produces no motion, the problem is connectivity / hardware, not app logic.

## Report format

Markdown with these sections, in order:

1. **Detected platform and run mode** (one line each).
2. **Sources sampled** (which logger trees / sinks were tailed).
3. **Primary class** with confidence (`high` / `medium` / `low`).
4. **Candidates** (zero or more, with confidence).
5. **Expected log pattern** (citation from the knowledge spec, verbatim).
6. **Match lines** with ±2 context lines per match. No raw streams.
7. **Recovery proposal** with cross-ref to the responsible skill / agent.

For class `motors-different-states`, point at Pollen's [`safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md) directly. For class `app-lock-held`, extract the holding app name from the match pattern and surface it in the report. For class `unclassified`, include the `examples/minimal_demo.py` command and a one-liner asking the developer to report the result back.

## Boundaries to neighbouring skills / agents

- knowledge layer (where logs live, which loggers exist, how levels work) → [`reachy-mini/app-logging`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-logging/de.md)
- on-device test lifecycle, including bulk telemetry sampling → agent [`reachy-mini-on-device`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-on-device.md)
- SDK idioms, method choice, safe-torque pattern → [`reachy-mini-sdk`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-mini-sdk/SKILL.md)
- production-host triage on provisioned devices → [`reachy-mini/host-provisioning`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/host-provisioning/de.md)
- new-app scaffolding → [`app-scaffold`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/app-scaffold/SKILL.md)

External canonical sources (cited via the knowledge spec, not duplicated here):

- Pollen `debugging.md` — Common Issues inventory, verify-basics-first heuristic
- Pollen `safe-torque.md` — recovery pattern for motor state mismatches
- Pollen `examples/minimal_demo.py` — canonical sanity check
