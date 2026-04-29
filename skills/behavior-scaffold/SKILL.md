---
name: behavior-scaffold
description: Scaffold a new Reachy Mini behavior with the official Pollen Robotics / Hugging Face folder shape — manifest, behavior module with lifecycle hooks, test stub, and docs stub. Activate on phrasings like "scaffold a new Reachy behavior", "create a Reachy Mini behavior named X", "start a new dance behavior for Reachy", "new behavior skeleton for Reachy Mini". Do not activate when the user only edits an existing behavior, only publishes one to Hugging Face, or asks about motion logic itself — those have their own skills/agents.
tags: [reachy-mini, behavior, scaffolding]
---

# Behavior Scaffold

> ⚠ TBD: validate against pollen-robotics/reachy_mini — the exact Pollen Robotics behavior layout (manifest filename, manifest schema, hook signatures, naming length limits) is not confirmed against the live SDK yet. Read the canonical layout under <https://github.com/pollen-robotics/reachy_mini> before generating files, and update this skill (and the spec) when the layout is verified.

## When this skill activates

Use this skill when the user wants to:

- create a brand-new behavior skeleton for the Reachy Mini robot
- start a new dance / expression / interaction behavior
- get a Hugging-Face-publish-ready behavior folder seeded so they only fill in motion logic

## When NOT to activate

- editing an existing behavior → no scaffold needed; use `reachy-mini-sdk` knowledge
- publishing a finished behavior to Hugging Face → `behavior-publish-hf` (planned)
- writing the actual motion / dance logic → developer's job, supported by `reachy-mini-sdk`
- testing a behavior live on the device → agent `reachy-mini-on-device` (planned)

## Inputs

Collect from the user before writing anything:

| Field | Required | Default |
|---|---|---|
| `name` | yes — ASCII kebab-case, validated | — |
| `description` | yes — 1–3 sentences | — |
| `author` | no | `git config user.name <email>` |
| `tags` | no — kebab-case, ≤5 entries, ≤30 chars each | — |
| `target_dir` | no | the consuming repo's behaviors directory (`> ⚠ TBD: confirm convention`) |

If `name` is missing or violates kebab-case / length rules, stop and report — do not silently rewrite the input.

## Pre-write validation

Run all of these **before** creating any file. Stop and report on the first failure:

1. **Name collision** — if `<target_dir>/<name>/` already exists, abort and quote the existing path. Never overwrite.
2. **Name shape** — ASCII kebab-case, length within Pollen / Hugging Face limits (`> ⚠ TBD: confirm exact limits`).
3. **SDK pin** — read the `reachy_mini` version pin from the consuming repo's manifest (e.g. `pyproject.toml`, `requirements.txt`, or a documented config). If no pin is found, abort with a clear error pointing the user at where to declare it. Do not guess a version inside the skill.

## What gets generated

All paths relative to `<target_dir>/<name>/`. The exact file names below are best-effort and **must be reconciled** against the live Pollen Robotics layout — every file therefore carries a TBD marker until verification.

```
<target_dir>/<name>/
├── manifest.<ext>            # > ⚠ TBD: confirm filename & schema vs. pollen-robotics/reachy_mini
├── <name>.py                 # behavior module with lifecycle hooks
├── README.md                 # description, hardware preconditions, quickstart
└── tests/
    └── test_<name>.py        # imports behavior, asserts hooks exist
```

- **Manifest** — required fields: `name`, `description`, `author`, `version`, optional `tags`, optional SDK-compat range. Unknown detail fields are emitted as TBD-marked stubs, not invented.
- **Behavior module** — for a reusable motion, prefer a subclass of the SDK's `Move` ABC with `duration` and `evaluate(t)` (source: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py>, docs: <https://huggingface.co/docs/reachy_mini/API/motion>). For a stateful long-running app, the SDK's apps surface (<https://huggingface.co/docs/reachy_mini/SDK/apps>, [`API/apps`](https://huggingface.co/docs/reachy_mini/API/apps)) is the right base; pull the actual hook signatures from there and **do not invent** `setup` / `step` / `stop` shapes that do not match the SDK. Each generated hook body is a single `pass` plus a pointer comment to `reachy-mini-sdk` and to the canonical control-surface reference at <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md>.
- **README / docstring** — quotes the description, lists hardware preconditions, shows a quickstart that imports and instantiates the behavior. Motion examples are out of scope.
- **Test stub** — imports the behavior, asserts the three hooks exist and accept the documented signatures. Motion-specific assertions are TBD-marked.
- **Optional Hugging Face Spaces manifest** — emit only when the user explicitly opts in; otherwise omit rather than ship an empty stub.

Authoritative source for the exact shape: <https://github.com/pollen-robotics/reachy_mini>.

## Hard rules / out of scope

- **MUST NOT** pre-implement motion logic beyond a `pass`-bodied stub plus pointer comment. Filling the hooks is the developer's job.
- **MUST NOT** auto-publish anything to Hugging Face or expect HF credentials at scaffold time.
- **MUST NOT** overwrite an existing behavior folder. On collision, abort and report the path.
- **MUST** mark every unverified layout / signature / limit with `> ⚠ TBD: validate against pollen-robotics/reachy_mini`.
- **MUST** emit files that pass `pre-commit run --all-files` without auto-fix changes (LF newlines, no trailing whitespace, valid YAML / JSON).
- **MUST** delegate concerns owned by neighbouring skills (see below) instead of growing this skill into them.

## Boundaries to neighbouring skills

- SDK knowledge / idiomatic API use → `reachy-mini-sdk`
- Home Assistant integration of the behavior → `home-assistant-bridge`
- Audio / beat / tempo detection for dance behaviors → `audio-beat-tracking` (planned)
- Publishing the finished behavior to Hugging Face → `behavior-publish-hf` (planned)
- Live deployment / on-device test → agent `reachy-mini-on-device` (planned)

## Next-steps checklist (returned to the developer after scaffold)

1. Confirm the `reachy_mini` SDK pin in the manifest matches the consuming repo's pin.
2. Replace each `> ⚠ TBD` marker once the live layout / signature is verified against the Pollen Robotics source.
3. Fill the `setup` / `step` / `stop` hook bodies — the `reachy-mini-sdk` skill is the canonical source for idiomatic patterns.
4. Run the test stub to confirm the behavior imports cleanly.
5. When the behavior is ready for hardware, dispatch the `reachy-mini-on-device` agent for a live test.
6. When you intend to publish, hand off to `behavior-publish-hf` — do not push files to Hugging Face from this skill.
