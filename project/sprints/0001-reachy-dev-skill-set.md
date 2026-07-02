---
number: 1
status: closed
started: 2026-07-02
ended: 2026-07-02
value_statement: The plugin author invokes a bundled skill or agent and builds a Reachy Mini app artefact (a behaviour or a Home Assistant integration piece) with the plugin's spec-grounded tooling.
artifact_ref: develop (shipped capability, pre-planning-suite)
roadmap_items: [R-1]
features: [F-1]
---

## Goal

The plugin author uses the bundled Claude Code skills and agents to build Reachy
Mini apps with spec-grounded tooling. Success is verified by F-1 `acceptance-1`:
invoking a bundled skill or agent builds a Reachy Mini app artefact.

## Features

- [F-1](../features/built-reachy-mini-app-artefact.md) — Built Reachy Mini app artefact — status: done

## Out of scope

- claude-shared's portfolio-wide skills and agents (a separate repository).
- The Reachy Mini hardware and the reachy_mini SDK themselves (upstream).

## Review notes

Retroactive reconciliation (2026-07-02): the
`claude-code-skills-and-agents-for-reachy-mini` capability was already
`status: active` before this repository adopted the planning suite (issue
nolte/claude-shared#262 mission-authoring backfill). This sprint records roadmap
item R-1 and feature F-1 as `done`, and itself as `closed`, to document the
delivered MVP rather than to plan new work.
