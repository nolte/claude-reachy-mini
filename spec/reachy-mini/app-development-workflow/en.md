# Development Workflow for Reachy Mini Apps

Status: draft

## Context

**Audience:** experienced Reachy developers, Claude sessions building a Reachy Mini app, future skill / agent authors who use this spec as a needs map.

This plugin (`claude-reachy-mini`) ships skills, agents, and domain specs as a toolbox for Reachy Mini app development. What was missing so far is the **methodology spec** that describes *how* an experienced Reachy developer turns a requirement into a finished, runnable app — which phases they go through, which sources they consult, which gates they may not skip, and at which point the code is checked for security-relevant misimplementations.

Without this spec, every Claude session reconstructs the process from scratch, which causes two recurring problems: (1) individual phases get skipped — typically the plan-first step and the security-review step, (2) the skill and agent inventory grows uncoordinated because it is unclear which phase should even be covered by which tool.

This spec is therefore two things at once: the canonical workflow description *and* the mapping of workflow phases to skills/agents. It serves as a needs map (which skills/agents exist, which are missing) and as a quality gate (which phase may not be skipped).

It is the methodological sister of the artefact spec [`reachy-mini/app-architecture`](../app-architecture/en.md): that one defines *what* the finished app is, this one defines *how* it comes into being.

## Goals

- End-to-end workflow from requirement intake to a running, deployed app, in clearly delimited phases with inputs, outputs, and an owner (skill/agent or human)
- Mandatory **plan-first gate**: no code without a written, requester-approved plan
- Mandatory **security-review gate** before deploy: source code is checked against a Reachy-specific security catalogue
- **Unambiguous skill/agent mapping per phase**: each phase either names the existing skill/agent as owner or marks the gap as `GAP`
- **Unambiguous reference list**: each phase points to the relevant SDKs, Pollen documentation, and internal specs it works against
- Purpose and form of the plan are defined precisely enough that a reviewer can locate the plan as an artefact, read it, and reconcile it against the requirement

## Non-Goals

- This spec **does not replace** the individual skill and agent specs (`app-scaffold`, `reachy-mini-sdk`, `reachy-mini-on-device`, `reachy-mini-deploy`, `reachy-mini-start`, `dance-choreography`, `home-assistant-bridge`); it links and orders them
- This spec **does not replace** the artefact spec `reachy-mini/app-architecture`; it points at its output but describes the path that leads there
- No Python or general-software tutorial — the spec assumes an experienced Python developer comfortable with `pip`, `pyproject.toml`, virtual environments, and git
- No full threat model of the Reachy Mini stack — the security-review gate is code-review-oriented; a dedicated threat model is its own spec, if needed at all
- No release/versioning strategy — `release-automation` and `release-publish-trigger` have their own specs in `nolte-shared`
- No CI/CD pipeline specification — the workflow definition is sequence-oriented, not pipeline-oriented
- No generic requirements-engineering methodology — the requirements phase assumes the requester brings a user story or goal; stakeholder discovery is out of scope

## Requirements

### Workflow phases — overview

A Reachy Mini app development **MUST** run through exactly the following phases, in this order:

1. **Requirement intake**
2. **Domain discovery** (spec and SDK review)
3. **Plan-first gate** (draft plan, sign off, freeze)
4. **Scaffold** (generate project skeleton)
5. **Implementation** (iterative, against the plan)
6. **Local self-test** (simulation or local daemon)
7. **Security-review gate** (review code against the security catalogue)
8. **On-device test** (live trial on real hardware)
9. **Deploy & start** (install and start on the Reachy)
10. **Publication** (optional — Hugging Face Spaces or other distribution)

Each phase has inputs, outputs, an owner (skill/agent or human), and references. Skipping a phase is allowed only at explicitly marked points (see "Skip rules" below).

### Phase 1 — Requirement intake

- **MUST** capture at minimum: what the app does (behavioural description), for whom (requester / user), when it is considered done (success criteria)
- **MUST** clarify whether the behaviour is pure hardware action, carries a Home Assistant integration, or processes audio / vision input — these axes later decide which specs are relevant in the discovery phase
- **SHOULD** land in a persistent artefact (issue, ADR, plan precursor), not only as chat history
- **MAY** already reference concrete motion slugs from the inventory under `reachy-mini/motions/` if the requester knows them

**Inputs:** user wish in any form
**Outputs:** requirement description with success criteria
**Owner:** human + requester (no skill/agent — `GAP-OK`: requirements are human-to-human context, no tooling need)
**References:** —

### Phase 2 — Domain discovery

Before any plan is written, the developer **MUST** have read or re-read the following internal specs (the list depends on the requirement; the first three are always mandatory):

- [`reachy-mini/app-architecture`](../app-architecture/en.md) — app layout, daemon lifecycle, distribution path
- [`reachy-mini/control-surface`](../control-surface/en.md) — what is controllable on the Reachy, and within which limits
- [`reachy-mini/app-logging`](../app-logging/en.md) — logging topology, conventions, triage catalogue
- [`reachy-mini/ha-integration`](../ha-integration/en.md) — when a Home Assistant touchpoint is planned
- [`reachy-mini/motions/<slug>`](../motions/) — when concrete motion primitives are used
- Skill specs for the skills that will later be invoked: [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md), [`claude/app-scaffold`](../../claude/app-scaffold/en.md)

In addition, the developer **MUST** at least glance at the following external sources to reconcile the SDK reality against the spec assumptions:

- Pollen Robotics SDK repository: <https://github.com/pollen-robotics/reachy_mini>
- Pollen Robotics app-assistant CLI (scaffold tool): <https://github.com/pollen-robotics/reachy-mini-app-assistant>
- Hugging Face Spaces distribution for Reachy Mini apps: <https://huggingface.co/collections/pollen-robotics/reachy-mini>
- Pollen `AGENTS.md` and `skills/` directories inside the SDK repository (canonical Pollen conventions)

**Inputs:** requirement description from phase 1
**Outputs:** list of relevant specs and external doc anchors, short note per source on what it contributes to the requirement
**Owner:** [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) skill (knowledge activation) + human (curation)
**References:** all of the above

### Phase 3 — Plan-first gate

This phase is **the central gate** of the workflow. Before this gate there is only reading and note-taking; after this gate, code generation begins.

- **MUST** be created as the file `plan.md` in the target app repo's root directory and version-controlled — `plan.md` is the only canonical plan sink; issues and PR descriptions may link to it but never replace it
- **MUST** be explicitly signed off by the requester (or by the developer in the requester role) before phase 4 starts; the sign-off is recorded in the plan itself (see template stub below)
- **MUST** follow the plan-template stub mandated by this spec (see next subsection) — section names and order are fixed, content may grow per app character
- **MUST** at least carry the following filled-in plan sections:
  - Scope: what the app does, what it explicitly does not
  - Motion inventory: which motion specs / `Move` classes are used
  - IPC surface: WebSocket commands, inbound and outbound (per `reachy-mini/app-architecture`)
  - External touchpoints: HA services, webhooks, audio input — when planned
  - Security considerations: which secrets, which network surfaces, which input validation (see phase 7)
  - Test strategy: what is tested in simulation, what must reach the device, what stays manual
  - Skill/agent plan: which skill/agent will be invoked in which later phase
- **SHOULD** list open questions explicitly instead of inventing answers — open points are plan content, not future assumptions
- **MAY** be produced via early skill consultation (for example `dance-choreography` produces a plan precursor for a dance app)
- **MUST NOT** contain code — pseudocode or interface sketches are allowed, executable code is not

#### Plan-template stub (mandatory)

Every plan **MUST** start from the following Markdown skeleton. Section headings and ordering are fixed; per app character, table rows, bullets, and sub-bullets may grow, but no top-level section may be missing or renamed.

```markdown
# Plan: <App name>

Status: draft | signed-off
Requester: <name>
Developer: <name>
Date: <YYYY-MM-DD>

## Scope
- **Does:**
- **Explicitly does not:**
- **Success criteria:**

## Motion inventory
| Slug | Move class | Source (motion spec / SDK) |
|---|---|---|

## IPC surface
- **Inbound (WebSocket):**
- **Outbound (WebSocket):**

## External touchpoints
- **HA services:**
- **Webhooks:**
- **Audio / vision:**

## Security considerations
- **Secrets (source, storage form):**
- **Network surfaces (bind address, TLS):**
- **Input validation (schema form):**
- **Whitelist / limits (motion slugs, value ranges):**

## Test strategy
- **Simulation:**
- **On-device:**
- **Manual:**

## Skill / agent plan
| Phase | Skill / agent |
|---|---|
| 4 — Scaffold | claude/app-scaffold |
| 5 — Implementation | claude/reachy-mini-sdk |
| 7 — Security review | reachy-app-security-review (planned) + nolte-shared:security-review |
| 8 — On-device test | claude/reachy-mini-on-device |
| 9 — Deploy & start | claude/reachy-mini-deploy + claude/reachy-mini-start |

## Open Questions
-

## Sign-off
- [ ] Requester (`<name>`): signed off on <YYYY-MM-DD>
- [ ] Developer (`<name>`): confirmed on <YYYY-MM-DD>
```

**Inputs:** requirement description, discovery notes
**Outputs:** signed-off plan artefact
**Owner:** human — `GAP`: no dedicated plan skill yet; candidate is a future `app-plan-author` skill that turns requirement + discovery output into a plan draft; until then the phase is human-driven with point-wise skill support (`reachy-mini-sdk`, `dance-choreography`)
**References:** [`reachy-mini/app-architecture`](../app-architecture/en.md); the `app-scaffold` skill creates the `plan.md` file and adopts the template schema defined here verbatim

### Phase 4 — Scaffold

- **MUST** go through the [`claude/app-scaffold`](../../claude/app-scaffold/en.md) skill and therefore through the Pollen CLI `reachy-mini-app-assistant create` — no hand-rolled skeleton
- **MUST** add provenance markers (CLAUDE.md, pointer to this plugin) as defined in `app-scaffold`
- **MUST** anchor the signed-off plan from phase 3 as `plan.md` in the repo
- **SHOULD** pin the Reachy SDK to a concrete minor version (per `reachy-mini/app-architecture`)

**Inputs:** plan artefact
**Outputs:** initialised app repository with Pollen-conformant structure, frozen plan, provenance
**Owner:** [`claude/app-scaffold`](../../claude/app-scaffold/en.md) skill
**References:** [`reachy-mini/app-architecture`](../app-architecture/en.md), Pollen CLI

### Phase 5 — Implementation

- **MUST** follow the plan; plan deviations are documented in the plan (plan update + re-sign-off) before they are implemented
- **MUST** use the [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) skill as the knowledge base for SDK idioms
- **MUST** use logging per the [`reachy-mini/app-logging`](../app-logging/en.md) convention (`logging.getLogger(__name__)`, no `print(...)` outside an explicit debug session)
- **MUST** pin the `reachy_mini` SDK to a concrete minor version
- **SHOULD** commit in small steps with meaningful Conventional Commit messages
- **MUST NOT** add functionality beyond the plan without a plan update — scope creep is a plan violation

**Inputs:** plan, scaffold
**Outputs:** app code with all plan items implemented
**Owner:** human + [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) skill
**References:** [`reachy-mini/app-architecture`](../app-architecture/en.md), [`reachy-mini/control-surface`](../control-surface/en.md), [`reachy-mini/app-logging`](../app-logging/en.md), Pollen SDK sources

### Phase 6 — Local self-test

- **MUST** at least exercise the `ReachyMini(spawn_daemon=True, use_sim=True)` mode, unless the app is a pure hardware action with no SDK test path
- **SHOULD** apply the Pollen heuristic "verify basics first": `examples/minimal_demo.py` or an app-specific smoke test as the first sanity-check stage (per [`reachy-mini/app-logging`](../app-logging/en.md))
- **MAY** drive automated tests via `pytest` to the extent the app character allows

**Inputs:** app code
**Outputs:** local-run evidence (log excerpt, test result)
**Owner:** human — `GAP-OK`: local testing is standard Python workflow, no dedicated skill needed
**References:** [`reachy-mini/app-logging`](../app-logging/en.md)

### Phase 7 — Security-review gate

This phase is **the second central gate** of the workflow. It **MUST** be passed before any on-device test against real hardware in a multi-user environment, and before any deploy.

The source code **MUST** be reviewed against the following Reachy-specific security catalogue. Each item **MUST** be either justified as "not applicable" or marked as passed:

#### Secrets and credentials

- No plaintext secrets in the repository (HA long-lived tokens, Wyoming keys, MQTT credentials, HF tokens) — `git grep` negative check for typical patterns (`token`, `password`, `api_key`, `Bearer` followed by a space, JWT structures)
- Secrets are loaded exclusively via environment variables or external secret stores
- Logging redact: secrets never appear in log output, neither at INFO nor DEBUG nor in tracebacks (the Pollen daemon captures stderr — see [`reachy-mini/app-logging`](../app-logging/en.md))

#### Network surfaces

- The Pollen-conformant app-local WebSocket stays `localhost`-only, no bind expansion to `0.0.0.0` (per [`reachy-mini/app-architecture`](../app-architecture/en.md) explicit non-goal)
- HA integrations use TLS unless the HA endpoint is explicitly declared as plain `http://`; even then a note SHOULD live in the plan
- External webhooks validate their signatures / shared secrets — no "accept all" endpoint

#### Input validation

- All WebSocket / webhook inputs are validated against an expected schema (Pydantic, dataclass + validation, jsonschema, or similar) — no unchecked passthrough into SDK methods
- Motion-slug calls are matched against the whitelist of known motion slugs from [`reachy-mini/motions/`](../motions/) — no string concatenation that could lead to reflection on arbitrary classes
- Numeric values (angles, velocities) are clamped against the limits from [`reachy-mini/control-surface`](../control-surface/en.md) **before** being passed into the SDK

#### Dependencies

- Before deploy, run `pip-audit` (or equivalent) against the lockfile — see [`nolte-shared:dependency-audit`](https://github.com/nolte/claude-shared/tree/main/skills/dependency-audit) as the tooling anchor
- The `reachy_mini` pin follows the architecture target (concrete minor version)

#### Privilege & side-effects

- No `subprocess.Popen(..., shell=True)` with unchecked input
- No file writes outside the expected app data directory
- No cloud-AI calls from the app process (per [`reachy-mini/app-architecture`](../app-architecture/en.md) explicit non-goal — if the plan requires it anyway, that is a plan change with re-sign-off)

#### Expected tooling integration

- This gate **MUST** be run such that the output is reproducible (Markdown report, issue comment, or PR review)
- The `nolte-shared` skill `security-review` (general code-security review skill, plugin-external) **SHOULD** be used for support — it produces a security-review run against the diff
- The `nolte-shared:dependency-audit` skill **MAY** automate the CVE part

- The report **MUST** be stored in the target app repo under `.audits/security-review/<YYYY-MM-DD>.md` and version-controlled; unlike the agent-owned subfolders (`.audits/deploy/`, `.audits/on-device/` — both gitignored, since they are machine logs), `.audits/security-review/` is a human-produced auditing artefact and belongs in git
- The app repo **MUST** carry a `.gitignore` rule that makes this explicit — template: ignore `.audits/`, then `!.audits/security-review/` as the exception; a PR / issue link is additionally allowed but does not replace the file

**Inputs:** app code, plan (for "not applicable" justifications)
**Outputs:** security-review report under `.audits/security-review/<YYYY-MM-DD>.md` (each catalogue item: passed / justified not applicable / finding + fix description)
**Owner:** human + [`nolte-shared:security-review`](https://github.com/nolte/claude-shared) skill — `GAP`: a dedicated plugin-internal skill `reachy-app-security-review` is planned **inside this plugin**, because the Reachy catalogue (motion whitelist against `reachy-mini/motions/`, value ranges from `reachy-mini/control-surface`, Pollen-daemon non-goals from `reachy-mini/app-architecture`) is plugin-local domain knowledge and does not belong in `nolte-shared`; until that skill exists, the phase is human-driven with the general `nolte-shared:security-review` skill as support
**References:** [`reachy-mini/app-architecture`](../app-architecture/en.md), [`reachy-mini/control-surface`](../control-surface/en.md), [`reachy-mini/app-logging`](../app-logging/en.md), [`reachy-mini/motions/`](../motions/), Pollen SDK sources

### Phase 8 — On-device test

- **MUST** be performed via the [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) agent, which runs a bounded test lifecycle with telemetry observation
- **MUST** persist the result as an audit report under `.audits/on-device/` (agent contract)
- **SHOULD** at least let the plan's "success criterion" path run through successfully once

**Inputs:** app code with passed security review
**Outputs:** structured PASS/FAIL report
**Owner:** [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) agent
**References:** [`reachy-mini/app-logging`](../app-logging/en.md)

### Phase 9 — Deploy & start

- **Deploy** **MUST** go through the [`claude/reachy-mini-deploy`](../../claude/reachy-mini-deploy/en.md) agent — it checks the Pollen contract, syncs the code, and anchors the install in the daemon environment
- **Start** **MUST** go through the [`claude/reachy-mini-start`](../../claude/reachy-mini-start/en.md) skill — it respects app locks and verifies the entry-point catalogue
- The deploy agent **MUST NOT** be used for pure-run purposes — live trial is phase 8, run-start is `reachy-mini-start`

**Inputs:** app repo with passed on-device test
**Outputs:** app installed and started on the device
**Owner:** [`claude/reachy-mini-deploy`](../../claude/reachy-mini-deploy/en.md) agent + [`claude/reachy-mini-start`](../../claude/reachy-mini-start/en.md) skill
**References:** [`reachy-mini/app-architecture`](../app-architecture/en.md)

### Phase 10 — Publication

- **MAY** be released as a Hugging Face Space (`reachy-mini-app-assistant publish` directly — no dedicated plugin skill at present)
- **SHOULD** reference the final security-review report as part of the release notes

**Inputs:** runnable app version
**Outputs:** published app
**Owner:** Pollen CLI directly — `GAP-OK`: currently sufficient without a plugin skill; candidate `reachy-app-publish-hf` if needed (see Skill / Agent hooks)
**References:** Hugging Face Spaces docs

### Skip rules

- Phase 6 (local self-test) **MAY** be skipped when the app has only hardware effects that cannot be checked in simulation — the justification goes into the plan
- Phase 10 (publication) is optional and always a phase-skip candidate
- Every other phase **MUST NOT** be skipped
- In particular, the gates **plan-first** (phase 3) and **security-review** (phase 7) are not skippable — not even "just this once"

### Update workflow (for behaviour updates of an existing app)

When a behaviour is **extended or changed** in an already existing, scaffolded app, a reduced phase loop applies. It does not replace the full workflow for new apps but describes the special case "incremental change".

- **MUST** at least run through phases **1, 3, 5, 7, 8, 9** — capture the requirement, update and re-sign the plan, implement, pass the security review again, run the on-device test, deploy and start
- Phase **2** (domain discovery) **MAY** be reduced to delta discovery: only the specs whose area is touched by the update are consulted
- Phase **4** (scaffold) **MUST NOT** be re-run — the app already exists; a re-scaffold would destroy provenance markers and plan history
- Phase **6** (local self-test) and phase **10** (publication) **MAY** be skipped under the same rules and justification duties as in the full workflow
- The `plan.md` in the app repo **MUST** be **updated** and re-signed off — no second `plan.md`, no silent plan changes; history follows from git
- The security-review report **MUST** create a new `.audits/security-review/<YYYY-MM-DD>.md` entry of its own (not overwrite the previous one) so update reports remain historically traceable

### Skill / Agent hooks (needs map)

This table is the central derivation of this spec: per phase it makes visible which skill/agent exists and where the gaps are. Gaps are work-order stock for `nolte-shared:claude-plugin-developer` or `nolte-shared:skill-management`.

| Phase | Owner today | Status | Suggested need |
|---|---|---|---|
| 1 — Requirement intake | human | `GAP-OK` | no tooling need |
| 2 — Domain discovery | `reachy-mini-sdk` skill (partial) | `OK` | — |
| 3 — Plan-first gate | human | `GAP` | future `app-plan-author` skill; input = requirement + discovery, output = plan draft |
| 4 — Scaffold | `app-scaffold` skill | `OK` | — |
| 5 — Implementation | human + `reachy-mini-sdk` skill | `OK` | — |
| 6 — Local self-test | human | `GAP-OK` | no tooling need |
| 7 — Security-review gate | human + `nolte-shared:security-review` | `GAP` | dedicated plugin skill `reachy-app-security-review` inside **this** plugin (knows the motion whitelist, control-surface limits, Pollen non-goals); this spec § phase 7 is the blueprint |
| 8 — On-device test | `reachy-mini-on-device` agent | `OK` | — |
| 9 — Deploy & start | `reachy-mini-deploy` agent + `reachy-mini-start` skill | `OK` | — |
| 10 — Publication | Pollen CLI directly | `GAP-OK` | optional future `reachy-app-publish-hf` skill |

## Acceptance Criteria

- [ ] Each of the ten phases has its own section in this spec with inputs, outputs, owner, and references
- [ ] Every phase except phase 1 (requirement intake) carries at least one reference to an internal spec, a skill, an agent, or an external doc URL
- [ ] The plan-first gate is verifiable through a concrete plan artefact (a reviewer can open the `plan.md` file in the app repo) and the plan follows the mandatory template stub from this spec (section names and ordering)
- [ ] The security-review catalogue is split into at least five concrete, checkable categories (secrets, network, input, dependencies, privilege)
- [ ] The phase-7 security-review report lives under `.audits/security-review/<YYYY-MM-DD>.md` in the app repo and is version-controlled; the app repo carries a `.gitignore` exception `!.audits/security-review/`
- [ ] The skill / agent hooks table maps every phase to an owner and names a skill / agent candidate for every `GAP` phase *with* a justification of the plugin boundary (internal to this plugin vs. `nolte-shared` vs. external)
- [ ] All references to existing internal specs point at paths that actually exist under `spec/`
- [ ] At least three external SDK / doc URLs are linked in phase 2 and dereferenceable
- [ ] Skip rules explicitly mark which phases may be skipped under which condition, and which never may
- [ ] On an update-workflow run, `plan.md` carries an updated sign-off entry with a date later than the last commit before the update; `.audits/security-review/` contains a new date-named entry that does not overwrite the previous one; phase 4 (scaffold) was not re-run

## Open Questions

All initial open questions were resolved in iteration 1:

- **Plan artefact location:** `plan.md` in the app repo is the only canonical plan sink (see § phase 3)
- **Plan template:** mandatory Markdown stub in this spec (see § phase 3, "Plan-template stub")
- **Security-review skill boundary:** dedicated plugin-internal skill `reachy-app-security-review` (see § phase 7 and skill / agent hooks)
- **Security-report archive:** `.audits/security-review/<YYYY-MM-DD>.md` in the app repo (see § phase 7)
- **Update workflow:** reduced phase loop 1, 3, 5, 7, 8, 9 (see § update workflow)

New open questions that arise from applying the spec will be added here in a second iteration.
