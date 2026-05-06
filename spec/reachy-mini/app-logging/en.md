# Logging and Failure Analysis During App Development

Status: draft

## Context

While developing a Reachy Mini app, log records spread across **at least three separate process and logger trees**: the app's own code (Python `logging`), the SDK package `reachy_mini` (logger tree `reachy_mini.*`), and the Pollen daemon (separate process, separate logger tree). Anyone unaware of this topology routinely spends hours searching in the wrong place, because almost every app action leaves log traces in several of these trees at once — and depending on the target platform (Wireless / Lite / Simulation) and the hosting mode (Pollen daemon vs. direct `pytest` run), those traces land in different sinks.

This spec consolidates the logging topology that is currently scattered across the Pollen sources — `skills/debugging.md`, `src/reachy_mini/reachy_mini.py`, `src/reachy_mini/daemon/daemon.py`, `src/reachy_mini/apps/manager.py`, `src/reachy_mini/daemon/robot_app_lock.py` — and makes it available as a canonical knowledge base for app development. It is the foundation that the `reachy-mini-sdk` skill (knowledge base), the `app-scaffold` skill (test stub), and the `reachy-mini-on-device` agent (log tailing step) build on, so they don't have to reassemble that knowledge each time.

Term clarification: "logs" in this spec means structurally captured diagnostic output of the standard Python `logging` framework plus the stdout / stderr streams the daemon captures from the app subprocess. "Failure analysis" covers both reading those logs during development (local `pytest`, local daemon, ad-hoc run) and the first triage steps against Pollen's canonical Common Issues inventory.

## Goals

- A complete map of every log source, grouped by logger tree and by platform, so each entry can be traced back to its sink and originating module
- A single, unambiguous convention for app-side logging (`logging.getLogger(__name__)`, RFC 2119 wording) that stays compatible with Pollen's SDK convention
- A clearly documented hosting behavior: what happens to `print(...)` and to exceptions in an app running under Pollen daemon hosting, vs. an app run directly via `pytest` or `python -m`
- A triage catalog of the most common failure modes mapped to concrete log patterns, reconciled against Pollen's `skills/debugging.md`
- The Pollen heuristic "verify basics first" (`examples/minimal_demo.py` as a sanity check before any app-specific diagnosis) as the mandatory first step on every new failure class

## Non-Goals

- Production logging on provisioned hosts (`reachy-app@<slug>.service`, journald vacuum, app pull-service logs) — covered in [`reachy-mini/host-provisioning`](../host-provisioning/en.md)
- Log-tailing script logic of the `reachy-mini-on-device` agent (which filters, which time anchor) — covered in [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md)
- Hardware failure recovery (mic FPC cable, motor diagnosis, spherical joints) — covered under [`pollen-robotics/reachy_mini/docs/source/troubleshooting/`](https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting)
- WebRTC / JS side logging of a browser app (separate surface, separate tooling world) — Pollen's `AGENTS.md` § JS apps covers it
- A generic Python `logging` tutorial — this spec assumes familiarity with `logging.getLogger`, `setLevel`, `StreamHandler`
- Structured logging in the JSON Lines / OpenTelemetry sense — not currently a Pollen standard; would be a separate spec if it ever became one
- Performance profiling, tracing, flame graphs — different tooling world

## Requirements

### Log source inventory

Every Reachy Mini app session emits log records across **three orthogonal logger trees** plus two stdout / stderr capture paths. The following table is the canonical map:

| Source | Logger name (Python) | Default level | Emitted by | Configuration source |
|---|---|---|---|---|
| App code | arbitrary (convention: `logging.getLogger(__name__)`) | set by the app itself; without setting, Python falls back to `WARNING` | every `logger.info(...)` call site in `reachy_mini_app/main.py` and submodules | `logging.basicConfig(...)` or explicit handlers in the app code |
| SDK main class | `reachy_mini` (via `getLogger(__name__)` in [`reachy_mini.py:133`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)) | `INFO` | `ReachyMini` methods (connection-mode selection, media re-acquire, move cancellation) | constructor parameter `ReachyMini(log_level: str = "INFO")` |
| Daemon main loop | `reachy_mini.daemon.daemon` (via `getLogger(__name__)` in [`daemon.py:50`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py)) | `INFO` | daemon lifecycle, media-server bring-up, central signaling relay | CLI flag `reachy-mini-daemon --verbose` (sets `DEBUG`); programmatically via the `log_level` parameter |
| App lock | `reachy_mini.daemon.robot_app_lock` ([`robot_app_lock.py:42`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/robot_app_lock.py)) | inherits from the daemon | lock acquire / lock release / conflict paths (the exact points at which a second app is rejected) | inherits; not separately configurable |
| App manager | `reachy_mini.apps.manager` plus the child `reachy_mini.apps.manager.runner` ([`manager.py:68`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py)) | inherits from the daemon | app subprocess spawn, lifecycle transitions, capture of app `stdout` / `stderr` | inherits |
| App `stdout` (under daemon hosting) | piped into `reachy_mini.apps.manager.runner.info` | inherits | every `print(...)` and anything the app subprocess writes to stdout; the subprocess is spawned with `python -u` (unbuffered) | not directly configurable; should flow through the app logger to obtain clean levels |
| App `stderr` (under daemon hosting) | piped into `reachy_mini.apps.manager.runner.error` (with heuristic) or `.warning` | inherits | tracebacks, uncaught exceptions, anything on stderr; the heuristic in `manager.py:206–209` classifies lines that look like errors as `error`, otherwise as `warning` | not directly configurable |

- **MUST** every new app use `logging.getLogger(__name__)` as its logger source in its own code base, never `logging.getLogger("root")` or `print()` for diagnosis; this produces a logger name that matches the Python module name and therefore enables fine-grained per-module filters and level switches
- **MUST NOT** the app code globally reconfigure the root logger (no unconditional `logging.basicConfig(level=DEBUG)`); doing so would unleash the SDK logger tree without control

### Platform profiles — where logs land

Where the logger trees above actually become visible depends on the target platform and on the run mode:

| Platform / mode | App logger sink | SDK logger sink | Daemon logger sink | Tool to read them |
|---|---|---|---|---|
| Wireless, app under daemon hosting | via the app-manager runner into the daemon log | inside the daemon process | systemd journald | `ssh pollen@reachy-mini.local "sudo journalctl -u reachy-mini-daemon.service -f"` |
| Wireless, app started manually via SSH | app `stdout` directly in the SSH terminal | inside the same process as the app | separate daemon service, still journald | app output in the SSH terminal; daemon in parallel via `journalctl` |
| Lite, app under daemon hosting | via the app-manager runner into the daemon log | inside the daemon process | daemon `stdout` in the terminal of the `reachy-mini-daemon` invocation | terminal of the daemon process; optionally with `--verbose` |
| Lite, app via `python -m` or editor run | app `stdout` in the terminal of the app invocation | inside the same process as the app | separate daemon process, separate terminal | two terminals side by side |
| Lite, `pytest` locally | pytest capture (`-s` disables it) | inside the pytest process | only when the test starts a daemon via `ReachyMini(spawn_daemon=True)` — then inside the test subprocess | `pytest -s` shows live output; without `-s`, the test failure output surfaces it |
| Simulation (`use_sim=True`) | app `stdout` in the terminal | inside the same process as the app | when the daemon-sim starts via `spawn_daemon=True`, in the same process; with `reachy-mini-daemon --sim` as an external process in its own terminal | as on Lite |

- **MUST** every newly written log path in an app be verified against its sink per platform before it is marked "working"; an entry that is visible locally in `pytest` is not automatically visible in journald
- **SHOULD** the default development setup run the app in **direct mode** (`pytest` or `python -m reachy_mini_app.main`) so that the app logger and the SDK logger both land in the terminal directly; daemon hosting is for the lifecycle validation pass, not for daily iteration
- **SHOULD** on Wireless under daemon hosting, the filter `journalctl -u reachy-mini-daemon.service -f --since '<run-start>'` plus `grep -v "uvicorn\|GET \|POST "` be used to suppress the HTTP noise from the REST surface — source: [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) §67–71

### Producing log entries in app code

Pollen's SDK fully relies on the standard Python `logging` framework — no custom wrappers, no Loguru, no structlog. Apps follow that.

- **MUST** the app code declare a module-global logger as a constant at the top of every module:

  ```python
  import logging

  logger = logging.getLogger(__name__)
  ```

- **MUST** diagnostic output flow through that logger, never through `print(...)`. Rationale: only logger calls respect levels, handler configuration, and multi-sink routing; `print(...)` bypasses all of it and turns into `info`-level noise under daemon hosting.
- **SHOULD** the app code use `logger.info(...)` for structural diagnostic points (lifecycle transitions, choreography section transitions, audio triggers); `logger.debug(...)` for fine-grained tracing; `logger.warning(...)` or `logger.error(...)` (with `exc_info=True`) for its own failure paths when an exception context is in scope
- **SHOULD** non-fatal exceptions handled by the app code be logged via `logger.exception("...")` (or `logger.error(..., exc_info=True)`) — that puts the traceback into the log line
- **MUST NOT** the app code log tokens, Hugging Face auth keys, WiFi credentials, or sensor values with personally identifiable context (consistent with [`reachy-mini/host-provisioning`](../host-provisioning/en.md) § Logging and Diagnostics)

### Daemon hosting: what happens to `print()` and exceptions

Pollen's app manager spawns the app subprocess in [`apps/manager.py:154`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py) with the flag `python -u` (unbuffered stdout / stderr) and reads both streams asynchronously:

- **stdout** → every line becomes `reachy_mini.apps.manager.runner.info(line)` (`manager.py:188–192`)
- **stderr** → every line is classified by a heuristic: lines containing typical error markers (Traceback, Exception, etc.) become `runner.error(line)`, otherwise `runner.warning(line)` (`manager.py:196–209`)

Consequences for the app developer:

- **MUST** every `print(...)` call in the app code be understood as "lands in the daemon log at level `info`"; that is not "lost", but it is also not the desired diagnosis path
- **MUST** an **uncaught exception** in `ReachyMiniApp.run(...)` always appear in the daemon log at level `error` — that is the primary anchor for crash triage; if it does not appear, the app is not running in the expected hosting mode
- **SHOULD** the app code format tracebacks itself and emit them via `logger.exception(...)`, rather than letting them propagate uncaught — that gives more context (app logger name, own message) and lets the app run into recovery paths instead of being killed by the daemon
- **MUST NOT** the app code reconfigure stdout / stderr directly (`sys.stdout = ...`); the daemon expects standard pipe behavior

### Log level configuration

Three knobs:

| Knob | Scope | Invocation |
|---|---|---|
| `ReachyMini(log_level=...)` | SDK logger tree (`reachy_mini.*`) | in app code at `ReachyMini` instance construction: `ReachyMini(log_level="DEBUG")` |
| `reachy-mini-daemon --verbose` | daemon logger tree, plus all child loggers (app manager, robot app lock) | when starting the daemon in the Lite setup |
| App-side `logger.setLevel(...)` or `logging.basicConfig(level=...)` | only the app's own logger tree | in the `main()` entry point; **do not** set the root logger globally |

- **SHOULD** the SDK level stay at the `INFO` default for daily development; switch to `DEBUG` **only** during active triage of a concrete failure class
- **MUST** the chosen level be documented in the app code base whenever it deviates from the SDK default (e.g. as a constant with rationale)
- **MUST NOT** the app code remotely reconfigure the daemon's level at runtime — the daemon is its own process; level changes require a daemon restart with the adjusted flag

### Common Issues triage catalog

Pollen's [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) lists the canonical failure classes. The following table binds each class to its expected log patterns and to the first triage action:

| Symptom (per Pollen) | Expected log pattern | First action |
|---|---|---|
| "Connection refused" / timeout while building `ReachyMini()` | `ConnectionRefusedError` in the app logger tree, or no daemon heartbeat in journald | check daemon status: `systemctl status reachy-mini-daemon.service` (Wireless) or daemon terminal output (Lite); another app holding the lock? see next row |
| Another app holds the lock | `RobotAppLock: rejected — held by <app_name>` in the daemon log (via the `reachy_mini.daemon.robot_app_lock` logger, `manager.py` / `robot_app_lock.py`) | stop the running app via the Pollen REST API or wait for the daemon to auto-release |
| Robot doesn't move | the app logger shows `set_target` calls without errors, but no motion; the SDK logger raises no warning | call `mini.get_motor_status()` from a diagnosis snippet; possibly `mini.enable_motors()`; see Pollen's debugging.md § "Robot doesn't move" |
| Jerky / choppy motion | no clean log trace — usually not a logging problem but a code-path problem | Pollen's debugging.md § "Jerky motion": single-owner loop at 50–100 Hz, no mix of `goto_target` and `set_target` |
| Import error at app start | `ModuleNotFoundError: reachy_mini` in the app subprocess stderr → surfaces as `runner.error` in the daemon log | is `reachy-mini` pinned as a dependency in the app's `pyproject.toml`? venv active? `uv pip install -e .` in the app directory |
| Audio playback failed | `Failed to initialize media server` in the daemon logger; possibly a GStreamer warning | check the platform (Lite has an audio backend, sim often does not); Pollen's debugging.md § "Sim vs Physical"; SDK parameter `media_backend="gstreamer_no_video"` as a fallback |
| "Motors in different states" | sporadic effort / position out-of-range warnings in the daemon logger | recovery pattern from Pollen's [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md): goto SLEEP_HEAD_POSE → `disable_motors()` |

- **MUST** every newly observed failure class be reconciled against this table before any spec update; if nothing fits, it is an Open Question, not a silent addition
- **SHOULD** the app developer first check the column "Expected log pattern" for any new failure class before testing on new code paths — when the expected pattern is missing, the problem belongs to a different class than initially assumed

### Verify-basics-first heuristic

Pollen's [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) § "First: Verify Basic Connectivity" formulates a concrete heuristic that this spec adopts as binding:

- **MUST** before diagnosing any app-specific failure class, [`examples/minimal_demo.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py) be run against the **same daemon, in the same run mode** — if it fails, the failure class is connectivity / daemon / hardware-related, not app-related
- **MUST** the result of this sanity check be documented in the triage report (failure reproduced? green run?) so downstream reviewers can follow the classification
- **MUST NOT** any app-code change be started while the sanity check is still red — that would be symptom fighting in the wrong place

### Recovery actions

| Action | Platform | Command | Effect |
|---|---|---|---|
| Daemon restart | Wireless | `ssh pollen@reachy-mini.local "sudo systemctl restart reachy-mini-daemon.service"` | terminates all app locks, kills active apps, starts the daemon process clean |
| Daemon restart | Lite | Ctrl-C in the daemon terminal, then `reachy-mini-daemon` (or `--verbose` / `--sim`) again | as above, manual |
| Motor recovery | Wireless / Lite | Pollen's safe-torque pattern: `mini.goto_target(head=SLEEP_HEAD_POSE); mini.disable_motors()` (source: [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md)) | anti-jerk before disable, then mechanically safe |
| App-lock force-release | Wireless / Lite | not officially supported — daemon restart is the supported path | see daemon restart |

- **MUST NOT** a daemon restart be wired into a CI / test run as a routine step — it would mask failure classes that should be solved by clean lock / lifecycle logic instead

## Acceptance Criteria

- [ ] The spec lists all three logger trees (app code, SDK, daemon) plus the two stdout / stderr capture paths under daemon hosting in a table
- [ ] Per platform (Wireless / Lite / Simulation) and per run mode (daemon hosting / direct), the sink of each logger tree is named
- [ ] The app-code convention `logging.getLogger(__name__)` is formulated as a MUST, with a clear rationale against `print(...)` and against root-logger reconfiguration
- [ ] The daemon-hosting behavior of `print(...)` and stderr is anchored to source-code line references in `apps/manager.py`
- [ ] The three log-level knobs (SDK constructor, daemon CLI flag, app-side logger) are named and demarcated against each other
- [ ] The Common Issues triage catalog covers every class listed in Pollen's `skills/debugging.md` and names the expected log pattern and first action for each
- [ ] The verify-basics-first heuristic (`examples/minimal_demo.py` before app diagnosis) is anchored as a MUST
- [ ] Recovery actions (daemon restart per platform, motor recovery via safe-torque) are documented as a table
- [ ] Cross-refs to [`host-provisioning`](../host-provisioning/en.md) (production logging), [`reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) (test-agent tailing), and [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) (idiomatic SDK use) are visible
- [ ] Source references to Pollen **code** files point to file plus line number; references to Pollen **Markdown** sources (`AGENTS.md`, `skills/*.md`) are cited at file level
- [ ] No MUST clause requires an API function whose existence is not anchored in `src/reachy_mini/` (consistent with the `deep-dive-docs` MUST in [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md))

## References

> Code source references are verified against `pollen-robotics/reachy_mini@main` as of 2026-05-06; when Pollen refactors, the line numbers will drift silently and are reconciled in a later drift audit. Markdown sources are cited at file level because they do not carry stable line anchors.

- Pollen skill `debugging` (canonical triage heuristic, Common Issues inventory, verify-basics-first): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- SDK main class (`log_level` constructor parameter, `self.logger = logging.getLogger(__name__)`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>
- Daemon implementation (daemon logger setup, default level, media-server bring-up): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py>
- App manager (app subprocess spawn with `-u`, stdout / stderr capture, runner child logger, error heuristic): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Robot-app-lock logger (lock acquire / release / conflict paths): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/robot_app_lock.py>
- Pollen's `AGENTS.md` (entry point, JS vs. Python surface, log-convention pointers): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Pollen skill `safe-torque` (recovery pattern for motor state mismatches): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Pollen example `minimal_demo.py` (canonical sanity check): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>
- Pollen hardware troubleshooting docs (hardware recovery, demarcated): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting>
- Internal cross-refs: [`reachy-mini/host-provisioning`](../host-provisioning/en.md) (production journald), [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) (test-agent log-tailing), [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) (idiomatic SDK)

## Open Questions

- Should the app stub in the `app-scaffold` skill set up a default `logging.basicConfig(...)` block at the `main()` entry point with a default format string, or is that left to the app developer?
- Structured logging (JSON Lines / OpenTelemetry bridge): does it have a use case on Reachy Mini that justifies the additional complexity? Proposal: only when a consumer actually needs it — until then, standard format strings.
- Log rotation on Wireless devices: journald vacuum is covered in [`host-provisioning`](../host-provisioning/en.md) — are there development scenarios where a local daemon run produces enough log volume to warrant its own measures?
- Cross-platform identification of the active logger inventory: does `logging.Logger.manager.loggerDict` give a reliable runtime self-report on every active `reachy_mini.*` logger, or is a separate diagnostic snippet needed?
- Pollen's `apps/manager.py:206–209` stderr heuristic: which exact marker strings classify a line as `error` vs. `warning`? The spec currently refers to it generically — should the classification list be tracked once Pollen's code changes?
- Jupyter / IPython sessions as a development mode: does the same `logging.basicConfig` default apply as for `pytest`, or does IPython capture the streams differently?
