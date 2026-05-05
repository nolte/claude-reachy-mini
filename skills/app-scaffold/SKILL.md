---
name: app-scaffold
description: Scaffold a new Reachy Mini app via the official Pollen CLI (`reachy-mini-app-assistant create`), then add provenance markers and a `plan.md` user-approval gate. Activate on phrasings like "scaffold a new Reachy Mini app", "create reachy mini app", "scaffold a new Reachy behavior", "start a new dance app for Reachy", "new app skeleton for Reachy Mini". Do not activate when the user only edits an existing app, only publishes one to Hugging Face, or asks about motion logic itself — those have their own skills/agents.
tags: [reachy-mini, app, scaffolding]
---

# App Scaffold

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-scaffold/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-scaffold/en.md).

## When this skill activates

Use this skill when the user wants to:

- create a brand-new Reachy Mini app skeleton
- start a new dance / expression / interaction app
- get a Hugging-Face-publish-ready app folder seeded so they only fill in motion logic

## When NOT to activate

- editing an existing app → no scaffold needed; use `reachy-mini-sdk` knowledge
- publishing a finished app to Hugging Face → `behavior-publish-hf` (planned), or `reachy-mini-app-assistant publish` directly
- writing the actual motion / dance logic → developer's job, supported by `reachy-mini-sdk`
- testing an app live on the device → agent `reachy-mini-on-device`

## Hard rules

1. **Never create app folders manually.** Always wrap `reachy-mini-app-assistant create`. Pollen's docs are explicit about this — manual creation drifts on entry-points, HF tags, and package structure in subtle ways. If the CLI is missing or fails, abort with instructions; do not reconstruct the skeleton by hand.
2. **`--publish` is the default.** Pollen's convention: always publish unless the user explicitly requests local-only. On `publish=true`, verify `hf auth whoami` upfront — never silently fall back to local-only when HF auth is missing.
3. **`plan.md` gate before any code.** After scaffolding, write a `plan.md` stub (Understanding / Approach / Open questions / Approval gate) and **wait for explicit user approval** before any further code commit. This is Pollen's AGENTS.md convention.
4. **No JS-only apps.** Pollen Hugging Face discovery requires a Python app. Web UI lives optionally as `<pkg>/static/`.

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `name` | yes | — | ASCII kebab-case (`reachy-mini-show`); CLI normalises to snake_case for the Python package |
| `target_dir` | yes | — | parent directory; CLI creates `<target_dir>/<name>/` |
| `description` | yes | — | 1–3 sentences for `pyproject.toml` / `README.md` |
| `template` | no | `default` | `default` for plain apps; `conversation` for LLM / speech / audio apps |
| `publish` | no | `true` | when `true`, creates HF Space + git remote; requires `hf auth login` |

If the user is silent on `template` or `publish`, use the defaults but state them in the response.

## Pre-flight (every run, in order — abort on first failure)

1. `reachy-mini-app-assistant --help` available on PATH? If not: `uv tool install reachy-mini` (or in the active venv: `uv pip install reachy-mini`).
2. If `publish=true`: `hf auth whoami` returns a valid identity? If not: `uv pip install --upgrade huggingface_hub && hf auth login` (token with **Write** permission).
3. Target path `<target_dir>/<name>/` does not yet exist? On collision, abort and quote the path. Never overwrite.

## Workflow

```bash
# 1) scaffold via the official CLI
reachy-mini-app-assistant create <name> <target_dir> \
  --template <default|conversation> \
  $( [ "$publish" = "true" ] && echo --publish )

# 2) verify Pollen's structural contract
reachy-mini-app-assistant check <target_dir>/<name>/
```

After step 2, post-process (these are the only files the skill itself writes):

- **`pyproject.toml`** — append a `[project.urls]` block:
  - `Plugin = "https://github.com/nolte/claude-reachy-mini"`
  - `SDK = "https://github.com/pollen-robotics/reachy_mini"`
  - `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
- **`CLAUDE.md`** — at the app repo root, point to this plugin and name the relevant skills (`reachy-mini-sdk`, `app-scaffold`, agent `reachy-mini-on-device`).
- **`README.md`** — inject a provenance block immediately after the HF frontmatter, linking back to the plugin and the motion catalogue.
- **`plan.md`** — at the app repo root, with four sections: Understanding, Approach, Open questions, Approval gate.
- **`tests/test_smoke.py`** — runs against `ReachyMini(spawn_daemon=True, use_sim=True)`, gated by GStreamer availability (skip if missing).

## Next-steps checklist (returned to the developer after scaffold)

1. **Fill `plan.md` and get user approval — before any code commit.** This is the gate.
2. Confirm the `reachy_mini` SDK pin in `pyproject.toml` matches the consuming app's expectations.
3. Replace the demo body inside `ReachyMiniShowApp.run()` with the real logic. Idiomatic SDK use: defer to the `reachy-mini-sdk` skill.
4. Run `pytest tests/` to confirm the smoke test stays green.
5. When ready for hardware, dispatch the `reachy-mini-on-device` agent for a live test.

## Boundaries to neighbouring skills

- SDK knowledge / idiomatic API use → `reachy-mini-sdk`
- Home Assistant integration of the app → `home-assistant-bridge`
- Audio / beat / tempo detection for dance apps → `audio-beat-tracking` (planned)
- Custom Hugging Face publishing workflows beyond what `reachy-mini-app-assistant publish` does → `behavior-publish-hf` (planned)
- Live deployment / on-device test → agent `reachy-mini-on-device`
