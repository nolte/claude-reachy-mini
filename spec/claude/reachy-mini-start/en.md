# Start Skill for Reachy Mini Apps

Status: draft

## Context
A Reachy Mini app that has been deployed onto the Pollen daemon's environment (typically via the `reachy-mini-deploy` agent) is registered as an entry point but is not yet running. Bringing the app online — i.e. instructing the Pollen daemon to actually invoke its `ReachyMiniApp.run()` — is a short, latency-light operation that is dominated by an interactive concern: the daemon enforces a single-app invariant, so any app already running has to be stopped first, and that stop is a state-changing decision the user must approve. The `reachy-mini-start` skill encapsulates this short start lifecycle in the main thread, with explicit user prompts at the only ambiguity (lock contention). It starts, it does not deploy and it does not run a full trial — those are the `reachy-mini-deploy` agent and the `reachy-mini-on-device` agent.

## Goals
- Bring a deployed app online on a real Reachy in a single skill invocation
- Make the single-app-invariant trade-off explicit at the user surface — never silently stop a running third-party app
- Confirm the daemon actually transitioned the app to `running` after the start call, not just that the start request was accepted
- Stay narrow: start and verify-running, no deploy, no trial, no daemon lifecycle work
- Compose cleanly with `reachy-mini-deploy`: a typical end-to-end is "deploy → start" or "deploy → on-device trial"; the skill is one half of that pair

## Non-Goals
- Deploying / installing code on the device — owned by `reachy-mini-deploy` (agent)
- Running a full live trial with telemetry — owned by `reachy-mini-on-device` (agent)
- Behavior / motion code editing — `reachy-mini-sdk`, `app-scaffold`
- Pollen daemon restart, reload, or reconfiguration — out of scope, never performed by this skill
- Hugging Face publishing — `reachy-mini-app-assistant publish`, or a future `reachy-app-publish-hf`
- Persistent supervisor / auto-restart — the skill is single-shot

## Skill-vs-Agent rationale
This concern is modelled as a **skill** rather than an agent because the skill-bias dimensions from `nolte-shared/spec/claude/skill-vs-agent/` apply directly:

- **Mid-flow user approval is required** — when another app holds the daemon lock, the only safe path is to ask the user. An agent (fire-and-forget) cannot route this question back to the parent thread.
- **Output flows naturally back to the conversation** — a one-line "started `<app>` on `<device>`" plus a state confirmation; no need for the structured-report boundary that an agent provides.
- **Short, low-volume operation** — a handful of REST calls; no installer logs, no rsync transfers, no telemetry samples. Context-window protection is not a load-bearing concern here.
- **Persists across the conversation** — the user may invoke `reachy-mini-start` more than once in a session (start app A, swap to app B, swap back), each invocation is a fresh decision point.
- **Counter-dimension** — tool restriction (the agent-side argument) does not apply meaningfully; the skill needs Bash for `curl` / `ssh`, which is the same surface the main thread already has.

## Requirements

### Inputs
- **MUST** accept `app_name` — the installed name as registered in `entry_points(group='reachy_mini_apps')`
- **MUST** accept `device` — an SSH host / daemon endpoint. For `wireless` this is the robot itself; for `lite` it is the host PC
- **MUST** accept `platform` — exactly one of `wireless` or `lite`. `simulation` **MUST** be rejected with a clear error
- **MAY** accept `if_busy` — `prompt` (default), `abort`, or `replace`. `prompt` asks the user; `abort` refuses silently when busy; `replace` stops the held app and starts the requested one **only after** the caller has explicitly confirmed at the invocation site
- **MUST NOT** accept plaintext credentials in any input — credentials come exclusively from `ssh_config` / environment

### Lifecycle
- **MUST** run pre-flight checks before any state change: daemon reachability, installed-apps listing, current app-lock state
- **MUST** abort cleanly when `app_name` is not visible in the daemon's entry-point catalog and surface `reachy-mini-deploy` as the natural follow-up; **MUST NOT** attempt to install
- **MUST** branch on the lock state explicitly:
  - `free` → start
  - already held by `app_name` → no-op success (the app is already running)
  - held by a different app → branch on `if_busy`
- **MUST**, when `if_busy=prompt`, ask the user once with the holder name and the proposed action, and wait for an explicit confirmation; **MUST NOT** carry a "yes" from a previous invocation forward
- **MUST**, when `if_busy=replace`, accept that the caller has already confirmed at the invocation site; the skill does not double-prompt
- **MUST**, after starting, verify via `GET /api/apps/current-app-status` (or the equivalent) that the daemon reports the requested app as `running`; a 200 OK on the start call alone is **not** sufficient confirmation
- **MUST** surface a one-line confirmation to the user with the started app name and the device; if the app declares a `custom_app_url`, mention it as a hint without blocking on its reachability
- **MUST NOT** restart, reload, or reconfigure the Pollen daemon under any circumstance
- **MUST NOT** force-stop a running third-party app without explicit user confirmation
- **MUST NOT** install, modify, or copy code as part of this lifecycle

### Output
- **MUST** return a one-line user-visible confirmation in the natural conversation flow
- **SHOULD** include the daemon-reported app state and any actionable hint (e.g. "FastAPI settings UI at `http://<device>:<port>`")
- **MUST NOT** return raw daemon logs, dependency traces, or credentials

### Security and secrets
- **MUST** read SSH / device credentials from environment or `ssh_config`
- **MUST NOT** write credentials into any user-visible response
- **SHOULD** keep SSH host-key verification on; surface the fingerprint rather than auto-accepting on first connect
- **MUST** apply platform-aware endpoint resolution: on Wireless the daemon endpoint is the robot, on Lite it is the host PC

### Boundaries
- **SHOULD** point at `reachy-mini-deploy` whenever the requested app is not yet in the daemon's entry-point catalog
- **SHOULD** point at `reachy-mini-on-device` whenever the user wants to test, observe, or assert behavior — not just bring the app online
- **SHOULD** point at `reachy-mini-sdk` and `app-scaffold` when the issue is upstream of the start call (broken behavior code, broken app skeleton)
- **MUST NOT** duplicate content from those artifacts — this skill is a daemon-side start/verify flow, not a knowledge base
- **MUST NOT** invoke other skills as a chain inside this skill's logic; it is a leaf operation

## Acceptance Criteria
- [ ] The skill rejects `platform=simulation` with a clear error pointing at the on-device agent
- [ ] The skill aborts when `app_name` is not in `entry_points(group='reachy_mini_apps')` and surfaces `reachy-mini-deploy` as the next step
- [ ] The skill never auto-stops a running third-party app; `if_busy=prompt` always asks first, `if_busy=replace` requires upstream confirmation, `if_busy=abort` exits without state change
- [ ] The skill never restarts the Pollen daemon
- [ ] The skill never installs, modifies, or copies code
- [ ] After a successful start, the skill verifies via the daemon's status endpoint that the requested app is reported as `running` — a 200 OK on `start-app` alone is not sufficient
- [ ] The skill surfaces a one-line confirmation that includes the started app, the device, and any FastAPI hint the app declares
- [ ] The skill exists as `skills/reachy-mini-start/SKILL.md` with valid frontmatter — `name: reachy-mini-start`, `description`, optional `tags`
- [ ] The `description` activates on phrasings like "start the app on the reachy", "bring this app online", "let reachy run the app now", "auf dem Reachy starten", "App auf dem Roboter starten"
- [ ] A skill-vs-agent rationale is visible in the skill body (at least mid-flow user approval, output flows naturally, short low-volume operation)
- [ ] References to `reachy-mini-deploy`, `reachy-mini-on-device`, `app-scaffold`, `reachy-mini-sdk` are visible in the body
- [ ] Statements without hardware verification carry a `⚠ TBD: validate against real hardware` marker

## References
- App lifecycle contract: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Daemon REST surface: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollen `AGENTS.md`: <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Sibling agent for code deployment: `agents/reachy-mini-deploy.md`
- Sibling agent for live trial runs: `agents/reachy-mini-on-device.md`

## Open Questions
- Which exact REST endpoint starts an installed app — `POST /api/apps/start-app/<name>` with the entry-point name, or a body-form variant? Confirm on first hardware contact and pin in the skill body
- Does the daemon's `current-app-status` endpoint return `running` synchronously after the start call, or is there a settling window during which the field is empty? If a settling window exists, define a default poll budget (proposal: 5 s, every 0.5 s) in the skill
- For Lite: is the daemon's REST surface identical to Wireless's, or does the host-PC daemon expose a different path prefix? Confirm on first Lite hardware contact
- Should the skill expose a "start-and-wait-for-app-ready" flag that polls the app's own `custom_app_url` until it returns 200, or is the daemon-side `running` confirmation enough? Leaning: enough; app-side reachability is the consumer's concern, not the daemon's contract
- Should `if_busy=replace` carry through to a future composite skill ("deploy then start") so the user only confirms once for both halves? Defer until the composite skill exists
