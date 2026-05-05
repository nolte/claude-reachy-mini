# App Scaffold Skill

Status: draft

## Context
A **Reachy Mini app** in Pollen's sense (a Hugging-Face-Space-publishable Python package with the `reachy_mini_python_app` tag, an entry point in the `reachy_mini_apps` group, and a `ReachyMiniApp` subclass with `run(self, reachy_mini, stop_event)`) is the primary deliverable of app development built around this plugin. Pollen ships an official CLI tool — `reachy-mini-app-assistant create` — that scaffolds the skeleton 1:1 to the Pollen convention (`pyproject.toml` with the entry point, `README.md` with the HF frontmatter tag, `index.html` / `style.css` for the HF Space landing page, `<pkg>/main.py` with the `ReachyMiniApp` class, optional `<pkg>/static/` for a web UI). Pollen's own docs are explicit: "**Never create app folders manually.** Manual creation leads to subtle issues that are hard to debug." This `app-scaffold` skill is therefore a **thin wrapper** around `reachy-mini-app-assistant create`, augments the result with provenance markers (a pointer back to this plugin), and creates a `plan.md` stub with a user-approval gate so Claude Code sessions align on requirements with the user before the first code commit. The skill complements the `reachy-mini-sdk` skill (knowledge base) on the writing path and delegates everything beyond the skeleton to specialised skills.

Terminology: "app" and "behavior" are sometimes used interchangeably in this plugin. Strictly speaking, this skill scaffolds a **Pollen app** (the Hugging-Face-publishable package); inside it, one or more **behaviors** (Move subclasses, motion logic) can live. The neighbouring spec [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/en.md) describes the relationship in detail.

## Goals
- A new Reachy Mini app is structurally complete after a single skill invocation — through Pollen's official `reachy-mini-app-assistant create`, augmented with provenance markers and a `plan.md` stub
- The skeleton follows the official Pollen Robotics app convention and is Hugging Face compatible, with `--publish` as the default
- Name collisions with existing app names are caught before anything is written
- Before the first code commit the skill enforces a user gate via `plan.md` (Pollen's AGENTS.md convention)
- The skill stays narrow: it calls the official CLI, augments provenance, and delegates neighbouring concerns to the relevant skills

## Non-Goals
- Concrete motion or dance logic (the developer's job; SDK knowledge lives in `reachy-mini-sdk`)
- Re-implementing the Pollen CLI layout — the skill **must** call `reachy-mini-app-assistant create`, never write the manifest, `pyproject.toml`, `main.py`, or `README.md` itself
- JS-only / web-only apps (Pollen docs: "JS-only apps are not yet supported for discovery/sharing"); this skill scaffolds Python apps with an optional `static/` web UI only
- Publishing the app to Hugging Face beyond what the CLI does (`reachy-mini-app-assistant publish` is a separate step; a dedicated `behavior-publish-hf` skill is planned for custom workflows)
- Audio analysis, beat / tempo detection (separate skill `audio-beat-tracking` planned)
- Home Assistant wiring of the app (separate skill `home-assistant-bridge`)
- Live deployment / on-device testing (separate agent `reachy-mini-on-device`)
- App refactoring or migration to a new SDK major version

## Requirements

### Triggering and activation
- **MUST** ship a `description` tight enough for Claude Code to activate on phrasings like "scaffold a new Reachy Mini app", "scaffold a new Reachy behavior", "create reachy mini app", "start a new dance app", "new behavior skeleton for Reachy Mini"
- **MUST** include the key terms in the `description`: app, scaffold, Reachy Mini, new (plus `behavior` as a synonym)
- **SHOULD** state explicitly when _not_ to activate (e.g. when an existing app is only being edited or published)

### Input parameters
- **MUST** require the app name as a mandatory parameter, ASCII kebab-case (`reachy-mini-show`); the CLI normalises internally to snake_case (`reachy_mini_show`) for the Python package name
- **MUST** require the target path (the parent directory under which the CLI creates the app folder)
- **MUST** accept a short description (1–3 sentences) for `pyproject.toml` / `README.md`
- **SHOULD** accept a **`template` parameter** (`default` | `conversation`) — for LLM / speech apps mandatory `conversation` (forked from the official conversation app template, including audio pipeline and FastAPI settings UI), otherwise `default`. Source: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/create-app.md> § Choose a Template
- **SHOULD** accept a **`publish` flag** (`true` | `false`) — default `true` (Pollen convention: "**Always use `--publish` unless the user explicitly requests a local-only app**"); on `true` the Hugging Face Space is created with a git remote
- **SHOULD** optionally accept author (default from `git config user.name`/`user.email`) and tags if the Pollen CLI takes them as flags; otherwise post-process the generated `pyproject.toml`

### Pre-flight obligations (before any write)
- **MUST** verify before the CLI call that `reachy-mini-app-assistant` is on PATH; if absent, abort with clear instructions (`uv tool install reachy-mini` or `uv pip install reachy-mini` in the active venv)
- **MUST** on `publish=true` verify `hf auth whoami` upfront; without a valid login abort with instructions (`uv pip install --upgrade huggingface_hub && hf auth login`, token with **Write** permission). Never silently fall back to `publish=false`.
- **MUST** check the target directory `<target_dir>/<name>/` for existence; on collision abort and name the conflicting path rather than overwrite

### Generated artifacts (via the official CLI)
- **MUST** create the skeleton via `reachy-mini-app-assistant create <name> <target_dir> [--publish] [--template default|conversation]`; **MUST NOT** write the manifest, `pyproject.toml`, `main.py`, `README.md`, `index.html`, `style.css`, or the entry-point declaration itself — Pollen's docs are explicit: "**Never create app folders manually**. Manual creation leads to subtle issues that are hard to debug." If the CLI is missing or fails, abort, do not reconstruct the skeleton by hand.
- **MUST** verify the result after the CLI run via `reachy-mini-app-assistant check <path>`; a failing check aborts the skill
- **MUST** post-process provenance markers afterwards:
  - in `pyproject.toml` add a `[project.urls]` block with `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
  - create a `CLAUDE.md` at the app repo root that names the recommended plugin skills (`reachy-mini-sdk`, `app-scaffold`, agent `reachy-mini-on-device`) and links the plugin repo
  - inject a provenance block into `README.md` directly after the HF frontmatter
- **MUST** create a `plan.md` stub in the app directory with the four required sections: (1) Understanding (what the app should do, in own words), (2) Technical approach (which SDK methods, which platform profiles), (3) Open questions (with empty answer fields), (4) User-approval gate. Source: Pollen's AGENTS.md, <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- **MUST** add a smoke-test stub (`tests/test_smoke.py`) that runs against `ReachyMini(spawn_daemon=True, use_sim=True)` and at minimum verifies import + class instantiation + one `set_target` tick; guard the runtime test on GStreamer availability via a skip condition
- **MUST NOT** scaffold a JS-only / web-only skeleton — Pollen docs: "JS-only apps are not yet supported for discovery/sharing." Discovery via Hugging Face requires a Python app; web UI lives (optionally) as a `static/` subfolder inside the Python package.

### User-approval gate before first code commit (AGENTS.md convention)
- **MUST** after creating `plan.md` and before any first code commit, explicitly wait for user approval; the next-steps checklist must list this gate as the first step
- **MUST NOT** make the first code commit (behavior logic, Move subclasses, etc.) without user confirmation of `plan.md` — the plan is Pollen's mechanism for catching approach drift early

### Platform profiles
- **MUST** run the test stub against `ReachyMini(spawn_daemon=True, use_sim=True)` as the default path — Simulation is the only profile that works without hardware and belongs in every CI run
- **SHOULD** annotate clearly in the test stub which aspects can **not** be checked in simulation (audio playback, IMU telemetry, LED sync, real pose reach) — pointer to the `reachy-mini-on-device` agent for on-hardware validation against Wireless or Lite
- **MUST** include in the app's README stub a platform table that names Wireless / Lite / Simulation and the applicability per platform
- **MUST NOT** hard-code platform-specific assumptions in the test stub (e.g. an IMU read that fails on Lite) — those checks are the `reachy-mini-on-device` agent's territory

### Consistency with repo standards
- **MUST** ensure that the CLI-generated files plus the provenance post-processing edits together pass `pre-commit run --all-files` without auto-fix changes
- **SHOULD** return a short next-steps checklist after the scaffold (e.g. "1. fill plan.md and get user approval, 2. confirm SDK pin, 3. fill `ReachyMiniApp.run()`, 4. on-device test with agent X")

### Out-of-scope clarification
- **MUST NOT** pre-implement motion logic in `main.py` beyond what the CLI ships as a demo; that is the developer's job
- **MUST NOT** silently override `reachy-mini-app-assistant` defaults the user did not explicitly waive (e.g. silently disable `--publish` because HF auth is missing — instead abort with instructions)
- **SHOULD** point at the neighbouring skills (`reachy-mini-sdk`, `home-assistant-bridge`, `audio-beat-tracking`, agent `reachy-mini-on-device`) instead of duplicating their concerns

## Acceptance Criteria
- [ ] The skill exists at `skills/app-scaffold/SKILL.md` with valid frontmatter (`name: app-scaffold`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The skill calls `reachy-mini-app-assistant create` internally — not a single manifest or layout file is written by the skill itself (only provenance post-processing)
- [ ] Default is `--publish=true`; without `hf auth whoami` the skill aborts with clear instructions and **does not** silently fall back to a local-only run
- [ ] On `template=conversation` the Pollen conversation template is selected; default is `default`
- [ ] After the CLI run, `reachy-mini-app-assistant check <path>` passes without findings
- [ ] Provenance post-processing adds `pyproject.toml` `[project.urls]`, a `CLAUDE.md` at the app repo root, and a provenance block in `README.md`
- [ ] A `plan.md` is created in the app directory with the four required sections (Understanding, Approach, Open questions, Approval gate)
- [ ] The next-steps checklist names the user approval on `plan.md` as the first step, before any code commit
- [ ] Test stub `tests/test_smoke.py` runs against `ReachyMini(spawn_daemon=True, use_sim=True)` without hardware and skips gracefully when GStreamer is missing
- [ ] Test stub annotates which aspects simulation cannot check
- [ ] README stub carries a platform table (Wireless / Lite / Simulation)
- [ ] The app name is validated for kebab-case; violations abort with a clear error; `reachy-mini-app-assistant` normalises internally to snake_case for the Python package name — that normalisation is surfaced in the skill output
- [ ] On a name collision (target path exists), the skill aborts and names the existing path
- [ ] `pre-commit run --all-files` passes on the generated files
- [ ] References to `reachy-mini-sdk`, `behavior-publish-hf`, `home-assistant-bridge`, `audio-beat-tracking`, and `reachy-mini-on-device` are visible in the skill body
- [ ] The post-scaffold next-steps checklist is documented as a convention in the skill

## References
- Upstream SDK repo (source of truth for manifest schema and hook signatures): <https://github.com/pollen-robotics/reachy_mini>
- App templates of the SDK (skeleton precedent, including `pyproject.toml.j2`, `main.py.j2`, `README.md.j2`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates>
- App manager implementation (canonical lifecycle expectations for `main(reachy, stop_event)`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Upstream Claude skill `create-app` (parallel authoring source against which drift is reconciled): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/create-app.md>
- SDK concept docs (Apps, Core Concept, Quickstart): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>

## Open Questions
- What does the official behavior layout in the current [`pollen-robotics/reachy_mini`](https://github.com/pollen-robotics/reachy_mini) repo look like concretely (folder structure, manifest filename, manifest schema)? Verify before implementing the skill — see [`src/reachy_mini/apps/templates`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates).
- Which manifest schema does Hugging Face Spaces require for publishable behaviors? Which fields are required, which optional?
- Which length and character-set rules apply exactly to behavior names on Hugging Face and in the SDK?
- Where does the behaviors folder usually live in the consuming app repo? Configuration, convention, or auto-discovery?
- Which test-framework convention applies (pytest, unittest, a Pollen-specific harness)?
- Should the skill optionally generate an example behavior that _additionally_ to the bare skeleton shows a tiny Move sequence, or strictly only the skeleton?
- Should the skill take the SDK pin from the consuming repo or own the pin itself? Proposal: from the repo, with a clear error when missing.
- How does the skill react when the official behavior layout changes (e.g. new required hooks)? Proposal: hook the drift audit into `reachy-mini-sdk`'s drift check.
