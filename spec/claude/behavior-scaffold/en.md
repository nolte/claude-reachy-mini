# Behavior Scaffold Skill

Status: draft

## Context
Reachy Mini behaviors (e.g. "dances to music", "nods on a Home Assistant call") are the primary deliverable of app development built around this plugin. Pollen Robotics / Hugging Face define a canonical repository and module shape that keeps behaviors loadable and publishable. Hand-rolled scaffolds tend to drift on detail conventions — manifest fields, hook signatures, test layout — and that drift only surfaces on the first load or publish attempt. The `behavior-scaffold` skill produces the complete, valid skeleton of a new behavior so the developer only has to fill in the motion logic. It complements the `reachy-mini-sdk` skill (knowledge base) on the writing path and delegates everything beyond the skeleton to specialised skills.

## Goals
- A new behavior is structurally complete after a single skill invocation — manifest, module, hooks, test stub, docs stub
- The skeleton follows the official Pollen Robotics behavior convention and is Hugging Face compatible if a future publish is wanted
- Name collisions with existing behaviors are caught before any file is written
- Generated files are syntactically valid out of the box and pass the repo's lint / pre-commit standards
- The skill stays narrow: it ships the skeleton, not the logic, and delegates neighbouring concerns to the relevant skills

## Non-Goals
- Concrete motion or dance logic (the developer's job; SDK knowledge lives in `reachy-mini-sdk`)
- Publishing the behavior to Hugging Face Spaces / Hub (separate skill `behavior-publish-hf` planned)
- Audio analysis, beat / tempo detection (separate skill `audio-beat-tracking` planned)
- Home Assistant wiring of the behavior (separate skill `home-assistant-bridge`)
- Live deployment / on-device testing (separate agent `reachy-mini-on-device`)
- Behavior refactoring or migration to a new SDK major version

## Requirements

### Triggering and activation
- **MUST** ship a `description` tight enough for Claude Code to activate on phrasings like "scaffold a new Reachy behavior", "create reachy mini behavior", "start a new dance behavior"
- **MUST** include the key terms in the `description`: behavior, scaffold, Reachy Mini, new
- **SHOULD** state explicitly when _not_ to activate (e.g. when an existing behavior is only being edited or published)

### Input parameters
- **MUST** require at least the behavior name, normalised to ASCII kebab-case
- **MUST** accept a short description (1–3 sentences) for the behavior manifest and the docs stub
- **SHOULD** optionally accept author (default from `git config user.name`/`user.email`) and tags (kebab-case, ≤5)
- **SHOULD** make the target path configurable (default: the consuming repo's standard behaviors directory, discovered by convention or configuration)

### Generated artifacts
- **MUST** create a behavior folder whose layout matches the official Pollen Robotics convention — the exact layout is `> ⚠ TBD: validate against pollen-robotics/reachy_mini` and is confirmed before the skill is implemented
- **MUST** emit the behavior manifest with all required fields (name, description, author, optional version, optional SDK-compat range); for unknown detail fields, prefer a TBD stub over guessing
- **MUST** emit the behavior module (Python) with the lifecycle hooks the SDK contract requires (e.g. `setup`, `step`, `stop` — exact signatures TBD until verified)
- **MUST** emit a README / docstring stub containing description, intended hardware preconditions, and a quickstart block
- **MUST** emit a test stub that at minimum imports the behavior and instantiates the hook signatures; motion-specific tests may be `> ⚠ TBD: validate against real hardware`
- **SHOULD** leave a `.gitignore`-friendly footprint (no caches, egg-info, IDE files committed)
- **MAY** also emit an optional Hugging Face Spaces manifest when publishing is foreseeable; otherwise omit rather than ship an empty stub

### Pre-write validation
- **MUST** check whether a behavior with the same name already exists; on collision, abort and name the conflicting path rather than overwrite
- **MUST** validate the behavior name against the SDK and Hugging Face naming rules (kebab-case, ASCII, ≤<TBD> chars) — exact limits confirmed before the skill is implemented
- **SHOULD** read the `reachy_mini` SDK pin to use from the consuming repo's configuration (not guessed in the skill body); if the consuming repo lacks an SDK pin, abort with a clear error

### Consistency with repo standards
- **MUST** emit files such that `pre-commit run --all-files` passes without auto-fix changes (correct newlines, no trailing whitespace, valid YAML / JSON)
- **MUST** mark every hardware-dependent assumption with `> ⚠ TBD: validate against real hardware` rather than stating it as fact
- **SHOULD** return a short next-steps checklist after the scaffold (e.g. "set the SDK pin in the manifest, fill the hooks, run the on-device agent X")

### Out-of-scope clarification
- **MUST NOT** pre-implement motion logic in the behavior module beyond a clearly marked example stub; that is the developer's job
- **MUST NOT** auto-publish the behavior to Hugging Face or expect credentials for it; publishing belongs to `behavior-publish-hf`
- **SHOULD** point at the neighbouring skills (`reachy-mini-sdk`, `home-assistant-bridge`, `audio-beat-tracking`, agent `reachy-mini-on-device`) instead of duplicating their concerns

## Acceptance Criteria
- [ ] The skill exists at `skills/behavior-scaffold/SKILL.md` with valid frontmatter (`name: behavior-scaffold`, `description`, optional tags) and is accepted by the catalog generator
- [ ] A test invocation with name, description, and (optional) author/tags produces a behavior folder containing manifest, module, README / docstring, and test stub
- [ ] The manifest contains the required fields; unknown detail fields are TBD-marked
- [ ] The behavior module contains the lifecycle hooks (signatures TBD-marked where unverified) and is syntactically valid
- [ ] The test stub imports the behavior and asserts the presence of the hooks; motion-specific tests are TBD-marked
- [ ] On a name collision the skill aborts and names the existing path
- [ ] The behavior name is validated for kebab-case and length; violations abort with a clear error
- [ ] `pre-commit run --all-files` passes on the generated files without auto-fix modifications
- [ ] Generated files follow the official Pollen Robotics behavior layout (once verified) or a clearly TBD-marked best-effort layout when the layout in the source tree is not yet finally confirmed
- [ ] References to `reachy-mini-sdk`, `behavior-publish-hf`, `home-assistant-bridge`, `audio-beat-tracking`, and `reachy-mini-on-device` are visible in the skill body
- [ ] The post-scaffold next-steps checklist is documented as a convention in the skill

## Open Questions
- What does the official behavior layout in the current `pollen-robotics/reachy_mini` repo look like concretely (folder structure, manifest filename, manifest schema)? Verify before implementing the skill.
- Which manifest schema does Hugging Face Spaces require for publishable behaviors? Which fields are required, which optional?
- Which length and character-set rules apply exactly to behavior names on Hugging Face and in the SDK?
- Where does the behaviors folder usually live in the consuming app repo? Configuration, convention, or auto-discovery?
- Which test-framework convention applies (pytest, unittest, a Pollen-specific harness)?
- Should the skill optionally generate an example behavior that _additionally_ to the bare skeleton shows a tiny Move sequence, or strictly only the skeleton?
- Should the skill take the SDK pin from the consuming repo or own the pin itself? Proposal: from the repo, with a clear error when missing.
- How does the skill react when the official behavior layout changes (e.g. new required hooks)? Proposal: hook the drift audit into `reachy-mini-sdk`'s drift check.
