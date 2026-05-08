---
name: reachy-app-publish-hf
description: Publish a Reachy Mini app to Hugging Face Spaces by calling Pollen's `reachy-mini-app-assistant publish` CLI, but only after walking ten pre-publish gates from `reachy-mini/app-development-workflow` § Phase 10 in fixed order. Activate on phrasings like "publish a Reachy app to Hugging Face", "release the app", "custom HF publish workflow", "publish reachy-mini-app to a Hugging Face Space", "release the app on Hugging Face". Do not activate for app creation (`app-scaffold`), deployment to one's own device (agent `reachy-mini-deploy`), plugin releases (`nolte-shared:release-publish-trigger`), or Hugging Face Hub authentication problems (Pollen / HF docs).
tags: [reachy-mini, app, publish, hugging-face, release]
---

# Reachy App Publish HF

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-app-publish-hf/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-app-publish-hf/en.md).

The skill is a narrow wrapper around Pollen's `reachy-mini-app-assistant publish`. It does not reimplement the Hugging Face push, does not perform HF auth, and does not modify app code. It walks ten pre-publish gates from [`reachy-mini/app-development-workflow`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-development-workflow/de.md) § Phase 10 in fixed order, then invokes the CLI only when every gate is green.

## When this skill activates

Use this skill when the developer wants to:

- publish a finished, locally-tested app to a Hugging Face Space via Pollen's official CLI path
- run the full pre-publish gate set (plan approval, security-review report, version pin, provenance, clean repo, branch policy) before the push
- format the publish call with the right visibility / official / nocheck flags

## When NOT to activate

- app creation or re-scaffold → `app-scaffold`
- deployment to one's own Reachy → agent `reachy-mini-deploy` (different distribution path)
- live trial / on-device validation → agent `reachy-mini-on-device`
- plugin releases of `claude-reachy-mini` itself → `nolte-shared:release-publish-trigger`
- Hugging Face Hub authentication problems (`hf auth login` failing) → Pollen / HF docs, not this skill
- rollback / unpublish — Hugging Face's own UI / API; this skill is forward-only
- SDK-pin bumps, dependency renovation, changelog generation → repo-internal business of the app

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `app_path` | yes | — | absolute or repo-relative path to the app to publish; the skill never publishes the current working directory implicitly |
| `commit_message` | yes | — | message for the Hugging Face Space commit; no auto-generated default — empty input aborts |
| `visibility` | no | `public` | `public` or `private`; Pollen's convention is `public` for the open app catalogue |
| `official` | no | `false` | `true` adds Pollen's `--official` flag (request to be accepted as an official Reachy Mini app, Pollen-side review) |
| `skip_gates` | no | `false` | `true` reports the pre-publish gates as warnings and adds `--nocheck` to the CLI call; **must** be requested explicitly |

## Hard rules

1. **Pollen CLI is the only execution path.** `reachy-mini-app-assistant publish` does the actual Hugging Face push. The skill does not call the `huggingface_hub` API directly, does not write Space metadata itself, does not configure git remotes manually.
2. **`skip_gates=false` is the default and the safe path.** Skipping gates routes through Pollen's `--nocheck` and is reported as warnings, not silenced.
3. **No retry on a failed publish.** The CLI's error path is for the user to interpret; the skill never retries with adjusted parameters.
4. **Forward-only.** The skill never deletes a Hugging Face Space, never `gh api -X DELETE`, never pushes a force-revert.
5. **PII clause inherited from `reachy-mini/app-logging`.** No HF tokens, no `hf auth` cookies, no vault values, no secret env vars in the report.

## Pre-publish gates (in this order — abort on first failure with a concrete remediation)

1. **Pollen CLI on PATH** — `reachy-mini-app-assistant --help` resolves; if missing, recommend `uv tool install reachy-mini`.
2. **Structural contract** — `reachy-mini-app-assistant check <app_path>` runs without findings; on red findings abort (Pollen convention: never publish a structurally broken app).
3. **Hugging Face auth** — `hf auth whoami` returns a valid identity; without login recommend `uv pip install --upgrade huggingface_hub && hf auth login` with **Write** permission.
4. **`plan.md` signed off** — `<app_path>/plan.md` exists and the § Approval-Gate block carries reviewer + date + commit SHA; empty or missing block aborts (source: `app-development-workflow` § Phase 3).
5. **Security-review report present and recent** — `<app_path>/.audits/security-review/<YYYY-MM-DD>.md` exists and the date is within the last 30 days; older or missing aborts (source: `app-development-workflow` § Phase 7).
6. **SDK version pin visible** — `<app_path>/pyproject.toml` carries a concrete `reachy-mini` pin (`==X.Y.Z` or `~=X.Y`); `>=X.Y` without an upper bound aborts.
7. **Provenance markers present** — `<app_path>/CLAUDE.md` and `<app_path>/README.md` carry the plugin reference block as defined in [`app-scaffold`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/app-scaffold/SKILL.md) § Generated artefacts; either missing aborts.
8. **Clean repository** — `git status --porcelain` inside `<app_path>` is empty; uncommitted changes abort.
9. **Local CI green** (optional, when `task lint` / `pytest tests/` exist) — on red checks abort, do not publish a known-broken state.
10. **Branch is `main` or `develop`** — feature / experiment branches should not land on Hugging Face directly; on other branch names warn and require user confirmation.

On `skip_gates=true`, the skill still **runs** the gates and **reports** every failure, but proceeds to the CLI call with `--nocheck`. Default is "all or nothing".

## Workflow

```bash
# After all gates are green:
reachy-mini-app-assistant publish <app_path> "<commit_message>" \
  $( [ "$visibility" = "private" ] && echo --private || echo --public ) \
  $( [ "$official" = "true" ]     && echo --official ) \
  $( [ "$skip_gates" = "true" ]   && echo --nocheck )
```

The CLI invocation is the only place where the actual publish happens. The skill captures the CLI's output verbatim and surfaces it in the report.

## Post-publish verification

- Extract the Hugging Face Space URL from the CLI output and surface it in the report.
- Probe the URL once (HTTP without auth, expect 200 / 302) to confirm reachability — never as the success criterion (the CLI is the source of truth), only as a sanity check.
- Capture `git remote -v` inside `<app_path>` so the user sees which HF remote was added by the CLI.
- **Never** trigger the first app run on Hugging Face or collect telemetry; the skill stops at "published".

## Report format

Five sections, in order:

1. **Pre-publish gate results** — ✓/✗ per gate, with a one-line failure reason for each ✗
2. **CLI invocation** — the exact `reachy-mini-app-assistant publish` command (visibility / official / nocheck flags filled in)
3. **Hugging Face Space URL** — from the CLI output; reachability probe result
4. **`git remote -v` excerpt** — so the user sees the new HF remote
5. **Next-steps hint** — pointer to `reachy-mini-on-device` for hardware validation when the app has not yet been hardware-tested

## Boundaries to neighbouring skills / agents

- new-app scaffolding → [`app-scaffold`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/app-scaffold/SKILL.md)
- deployment to one's own Reachy → agent [`reachy-mini-deploy`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-deploy.md)
- live trial / hardware validation before publish → agent [`reachy-mini-on-device`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-on-device.md)
- the canonical phase-by-phase development workflow → [`reachy-mini/app-development-workflow`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-development-workflow/de.md)
- SDK idioms inside the app → [`reachy-mini-sdk`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-mini-sdk/SKILL.md)

External canonical sources (cited via the spec):

- Pollen `reachy-mini-app-assistant publish` — the only path to a Hugging Face Space
- Pollen's `AGENTS.md` — publish expectations
- Hugging Face Spaces docs — distribution backend
