---
id: F-1
title: Built Reachy Mini app artefact
status: done
roadmap_item: R-1
sprint: 1
created: 2026-07-02
ended: 2026-07-02
verifies_sprint_value: acceptance-1
consistency_check:
  performed_at: 2026-07-02
  agent_version: manual-fallback (retroactive; feature-consistency-reviewer not run cross-repo)
  findings:
    - kind: clean
      target: project/features/
      resolution: proceed
      evidence: "project/features/ empty (first decomposition); no feature-to-feature overlap possible."
    - kind: prior-art
      target: the shipped skills and agents
      resolution: proceed
      evidence: "The Reachy Mini skill and agent set already exists; F-1 documents the build contract, it does not build new skills."
---

## Description

F-1 is the mission-verifying feature for the shipped
`claude-code-skills-and-agents-for-reachy-mini` capability. The bundled skills and
agents build Reachy Mini apps with spec-grounded tooling. The contract holds when
the plugin author invokes a bundled skill or agent and builds a Reachy Mini app
artefact. This holds against the shipped set, so the retroactive reconciliation
records this feature as `done` (issue nolte/claude-shared#262).

## Acceptance criteria

- [x] **acceptance-1** The plugin author invokes a bundled skill or agent that
  builds a Reachy Mini app artefact (a behaviour or a Home Assistant integration
  piece) with the plugin's spec-grounded tooling. _(This is the sprint value
  verifier.)_
- [x] **acceptance-2** The skills and agents cover the app classes named in the
  capability (dance-to-music behaviours and Home Assistant integration).
- [x] **acceptance-3** The plugin is distributable through the Claude Code plugin
  marketplace.

## Test hooks

- **acceptance-1** — invoke a bundled skill or agent and inspect the built
  artefact — passing.
- **acceptance-2** — inspect the shipped skills and agents — passing.
- **acceptance-3** — the plugin manifest and marketplace listing — passing.

## Consistency notes

Retroactive documentation feature: the skill and agent set predates the planning
suite. No new implementation is introduced; the feature exists so the mission's
`verifies_via: F-1:acceptance-1` and sprint 1's `value_statement` resolve to a
real acceptance criterion.

## References

- `project/portfolio.yml` capability `claude-code-skills-and-agents-for-reachy-mini`
- `AUDIENCES.md` audiences (dogfooding author, later public community)
- `README.md` (the bundled skills and agents)
