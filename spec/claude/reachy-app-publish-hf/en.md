# Reachy App Publish HF Skill

Status: draft

## Context

Publishing a finished Reachy Mini app to Hugging Face Spaces looks trivial today: `reachy-mini-app-assistant publish <path> "<commit-message>"` is enough. That is precisely the problem — the command bypasses every pre-publish gate that [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/en.md) § Phase 10 marks as binding: signed-off plan, green security-review audit, clean repository, pinned SDK release, correct provenance markers. Anyone running the CLI directly can unintentionally publish an app that still carries an open `plan.md` review or a red security finding — and that is then public on Hugging Face.

This `reachy-app-publish-hf` skill is the plugin's own shell around Pollen's publish CLI: a **narrow wrapper** that walks every mandatory gate before the call and only publishes when all gates are green. It does not replace the CLI (which stays the executing layer for HF Space creation, git remote configuration, asset upload); it **gates** it.

Term clarification: "publish" here means the Hugging Face Space upload (code, provenance, README with HF frontmatter), not the internal plugin releases (`release-publish-trigger` is `nolte-shared` and concerns the plugin repo, not a consumed app).

## Goals

- A Reachy Mini app gets published to Hugging Face with a single skill invocation — provided every pre-publish gate from [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/en.md) § Phase 10 is green
- Pre-publish gates are explicitly documented and walked in fixed order, with a clear error message per failed gate
- Pollen's `reachy-mini-app-assistant publish` is the only executing layer — the skill calls it; it does not modify the Hugging Face push itself
- The skill stays narrow: it consumes `app-development-workflow` as the contract, delegates app creation to `app-scaffold`, deployment to one's own device to the `reachy-mini-deploy` agent, and live validation to the `reachy-mini-on-device` agent

## Non-Goals

- Replace Pollen's publish CLI — `reachy-mini-app-assistant publish` stays the only path to the actual Hugging Face publish
- Perform Hugging Face Hub authentication on its own — `hf auth login` is a precondition, not a task of this skill
- App creation or re-scaffold (`app-scaffold` does that)
- Deployment of an app to the local Reachy Mini (`reachy-mini-deploy` agent does that)
- Live trial after publish (`reachy-mini-on-device` agent does that)
- Plugin release of `claude-reachy-mini` plugin versions (`nolte-shared:release-publish-trigger`)
- Rollback / unpublish — Hugging Face has its own UI / API for deletion; the skill is forward-only
- Hugging Face quotas, plans, storage limits (Hugging Face's own docs)
- CI / CD integration in a pipeline — the skill is single-shot per call, not a workflow step (`release-automation` is plugin business, not app)
- SDK pin bumps, dependency renovation, changelog generation — repo-internal business of the app, possibly with Renovate / Release-Drafter

## Requirements

### Trigger and activation

- **MUST** carry a `description` that activates Claude Code on phrasings like "publish a Reachy app to Hugging Face", "release the app", "custom HF publish workflow", "publish reachy-mini-app to a Hugging Face Space", "release the app on Hugging Face"
- **MUST** include the keywords in the `description`: publish, Hugging Face, Reachy Mini, app, release, Space
- **SHOULD** explicitly call out when _not_ to activate: pure app creation (`app-scaffold`), deployment to one's own device (`reachy-mini-deploy` agent), plugin releases (`nolte-shared:release-publish-trigger`), Hugging Face Hub auth issues (Pollen / HF docs)

### Input parameters

- **MUST** accept the app path as a mandatory parameter (`app_path`); the skill does not publish the current working directory without an explicit path input
- **MUST** accept a commit message (`commit_message`) for the Hugging Face Space commit; absent the message, the skill aborts — no auto-generated default
- **SHOULD** accept a visibility parameter (`visibility`: `public` | `private`); default is `public` (Pollen's convention for the open app catalogue), `private` as opt-in for internal apps
- **SHOULD** accept an `official` flag (default `false`) — `true` triggers Pollen's `--official` flag (request to be accepted as an official Reachy Mini app, Pollen-side review)
- **SHOULD** optionally accept a `skip_gates` flag (default `false`); when `true`, pre-publish gates are **reported** and skipped, and the CLI call receives the `--nocheck` flag — must be explicitly requested by the user, never default
- **MUST NOT** the skill silently skip mandatory gates without `skip_gates=true` set explicitly — default behaviour is "all or nothing"

### Pre-publish gates (in this order)

The skill **MUST** check the following gates before the CLI call, in order, and abort on the first failure:

1. **Pollen CLI available** — `reachy-mini-app-assistant --help` resolves on PATH; if missing, abort with instructions (`uv tool install reachy-mini`)
2. **App structural contract** — `reachy-mini-app-assistant check <app_path>` runs without findings; abort on red findings (Pollen convention: never publish a structurally broken app)
3. **Hugging Face auth** — `hf auth whoami` returns a valid identity; without login, abort with instructions (`uv pip install --upgrade huggingface_hub && hf auth login`, token with **Write** permission)
4. **`plan.md` signed off** — the file `<app_path>/plan.md` exists and has the § Approval-Gate block filled in with reviewer + date + SHA; on empty or missing approval block, abort (source: `app-development-workflow` § Phase 3)
5. **Security-review report present** — the file `<app_path>/.audits/security-review/<YYYY-MM-DD>.md` exists with an entry whose date falls within the last 30 days; older or missing → abort (source: `app-development-workflow` § Phase 7)
6. **SDK version pin visible** — `<app_path>/pyproject.toml` carries a concrete `reachy-mini` pin (`==X.Y.Z` or `~=X.Y`); on `>=X.Y` without an upper bound, abort (otherwise Hugging Face Spaces could pull a future incompatible SDK release)
7. **Provenance markers present** — `<app_path>/CLAUDE.md` and `<app_path>/README.md` carry the plugin reference block as defined in [`claude/app-scaffold`](../app-scaffold/en.md) § Generated artefacts; on either missing, abort
8. **Clean repository (no dirty git tree)** — `git status --porcelain` inside `<app_path>` is empty; uncommitted changes → abort (consumers should know what is actually being published, no surprises from uncommitted files)
9. **Local CI green** (optional, when `task lint` / `pytest tests/` is defined in the app repo) — on red checks abort, rather than publish a broken state
10. **Branch is `main` or `develop`** — feature / experiment branches should not land on Hugging Face directly; on other branch names, warn and require confirmation

- **SHOULD** the skill attach a concrete remediation hint to every failed gate (e.g. "plan approval missing → fill in § Approval gate in plan.md")
- **MUST NOT** silently skip a pre-publish gate when `skip_gates=false`

### CLI invocation

- **MUST** the skill, after a green pre-publish pass, call `reachy-mini-app-assistant publish <app_path> "<commit_message>"` with the matching visibility / official / nocheck flags
- **MUST** return the CLI output as-is to the user; the skill does not modify it
- **MUST NOT** the skill implement the Hugging Face push itself (use `huggingface_hub` API directly) — the CLI is the only supported interface
- **MUST NOT** the skill retry on a failed `publish` invocation — failure triage is the user's job

### Post-publish verification

- **MUST** extract the Hugging Face Space URL from the CLI output and surface it in the report
- **SHOULD** verify the URL is reachable (HTTP probe without auth, expect 200 / 302)
- **SHOULD** name the `git remote -v` output of the app repo so the user sees which HF remote was added
- **MUST NOT** the skill trigger the first app run on Hugging Face or collect telemetry — out of scope

### Report format

- **MUST** the skill return a compact report with these sections: (1) pre-publish gate results (✓/✗ per gate), (2) CLI call command (with visibility / official flags), (3) Hugging Face Space URL, (4) `git remote` excerpt, (5) next-steps hint
- **SHOULD** point at the `reachy-mini-on-device` agent in next-steps when the app has not yet been hardware-tested
- **MUST NOT** the report contain the HF token, vault values, or other secrets (PII clause analogous to [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md))

### Out-of-scope clarification

- **MUST NOT** the skill modify app code, perform a version bump in `pyproject.toml`, or generate a changelog — repo-internal business
- **MUST NOT** Pollen daemon restart, app-lock force release, or hardware safety limits — no hardware touchpoint
- **SHOULD** point at [`reachy-mini-deploy`](../reachy-mini-deploy/en.md) as the sister operation for deployment to one's own device (HF ↔ own device are different distribution paths)
- **SHOULD** point at [`reachy-mini-on-device`](../reachy-mini-on-device/en.md) as the validation step **before** publish (`app-development-workflow` Phase 8 before Phase 10)

## Acceptance Criteria

- [ ] The skill lives at `skills/reachy-app-publish-hf/SKILL.md` with valid frontmatter (`name: reachy-app-publish-hf`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The `description` carries the keywords (publish, Hugging Face, Reachy Mini, app, release, Space) and explicitly calls out at least three anti-triggers
- [ ] The ten pre-publish gates run in the specified order, each with a clear error message
- [ ] `skip_gates=false` is the default; `skip_gates=true` activates Pollen's `--nocheck` and reports gates as warnings
- [ ] CLI invocation uses `reachy-mini-app-assistant publish` exclusively; no direct `huggingface_hub` API usage
- [ ] Default visibility is `public`, `private` as opt-in
- [ ] `official=true` results in Pollen's `--official` flag in the CLI call
- [ ] Post-publish report contains the Hugging Face Space URL, `git remote` excerpt, next-steps hint
- [ ] On a `low` status of a gate, the concrete fix proposal is named in the report
- [ ] Cross-refs to [`app-scaffold`](../app-scaffold/en.md), [`reachy-mini-deploy`](../reachy-mini-deploy/en.md), [`reachy-mini-on-device`](../reachy-mini-on-device/en.md), [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/en.md) are visible
- [ ] PII clause is honoured: no tokens, vault values, secrets in the report
- [ ] `pre-commit run --all-files` passes on the skill file

## References

> Source references to Pollen code files point to file plus line number (once implemented); references to Pollen Markdown sources are cited at file level.

- Pollen's app-assistant CLI (publish subcommand): <https://github.com/pollen-robotics/reachy-mini-app-assistant>
- Pollen's app concept docs (publish convention): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- Hugging Face Spaces docs (distribution backend): <https://huggingface.co/docs/hub/spaces>
- Pollen's `AGENTS.md` (publish expectations, plan convention): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Internal cross-refs:
  - [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/en.md) — Phase 10 contract
  - [`claude/app-scaffold`](../app-scaffold/en.md) — provenance markers, plan.md schema
  - [`claude/reachy-mini-deploy`](../reachy-mini-deploy/en.md) — sister distribution (own device)
  - [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) — pre-publish validation
  - [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) — PII clause role model

## Open Questions

- Retention of the security-review report: is "last 30 days" the right threshold, or should it be app-configurable? Pragmatic first, sharpen later.
- Branch restriction to `main` / `develop`: too strict, or too lax? Should a tag pin also be allowed? Wait for consumer feedback.
- Pollen's `--official` flag: what are the Pollen-side preconditions for an official app request? Verify against Pollen docs once the flag is actually used.
- Default visibility `public`: is this the right Pollen convention, or is there a security request for `private` as default? Consumer consensus needed.
- Multi-space publish (e.g. "one app, two HF orgs"): not currently in scope; if a consumer needs it, separate spec.
- Update workflow for already-published apps: is a separate sub-operation needed, or is republish via the same CLI invocation fine? `reachy-mini-app-assistant publish` is idempotent against the HF Space, so probably no separate update path is needed.
- Audit trail: should the skill log every publish to `<app_path>/.audits/publish/<timestamp>.md`, analogous to `.audits/security-review/`? Proposal: yes, low cost, useful for compliance questions.
