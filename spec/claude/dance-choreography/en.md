# Dance Choreography Skill

Status: draft

## Context
The `reachy-mini-show` app (see `reachy-mini/app-architecture`) ships four BPM-parameterized dance building blocks — `groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around` — plus emotion and state blocks (e.g. `excited`, `proud`, `happy`, `surprised`, `shy`, `waiting-idle`, `alert-listening`). Anyone composing a dance sequence for a concrete song or mood today faces two gaps: (1) there is no canonical, machine- and human-readable plan that arranges the blocks into a section structure (intro / verse / chorus / bridge / outro) for a piece of music, and (2) the hardware constraints from `reachy-mini/control-surface` (pitch velocity, servo heat, `max_body_yaw`, brown-out risk) have to be re-derived per section, which goes wrong regularly when composed by hand. The `dance-choreography` skill closes both gaps: it accepts a song / mood description and produces a **choreography file** as an authoring artifact — a section table with slug references into the motion catalog, BPM and beat parameters, transition rules, and a translation checklist. The developer then translates that file by hand into `Move` subclasses or move sequences inside `reachy-mini-show`. The skill writes **no** robot code, sends **no** commands to the app, and performs **no** beat detection.

## Goals
- A single skill invocation produces a complete, validated choreography file from a song or mood description
- Every section refers exclusively to slugs that exist under `spec/reachy-mini/motions/` — no invented motions
- Hardware limits from `spec/reachy-mini/control-surface/de.md` are checked per section before writing (BPM range per block, cool-down after `headbang-soft` bursts, `max_body_yaw` for `spin-look-around`, brown-out risk under full actuator load)
- Platform consequences (Wireless / Lite / Simulation) are explicit per section — a `headbang-soft` sequence safe on Wireless via IMU cool-down carries a hard bang limit on Lite
- Output is both human-readable (for the translating developer) and machine-readable (for later tools that want to diff, lint, or render choreographies)
- The skill stays narrow: it delivers the plan, not the code, and delegates SDK idioms to `reachy-mini-sdk`, behavior scaffolding to `behavior-scaffold`, and audio beat detection to `audio-beat-tracking` (planned)

## Non-goals
- Concrete `Move` subclasses, move sequence code, or app patches inside the `reachy-mini-show` repo (developer's job; SDK idioms via `reachy-mini-sdk`)
- Beat detection or BPM estimation from an audio file (job of `audio-beat-tracking`, planned)
- Direct WebSocket push to the app (`set_dance`, `play_behavior`); choreographies are not played back at runtime by this skill
- Composing new dance blocks — the skill consumes the existing motion catalog and at most flags missing slugs as an open question, never invents one
- Audio file shipping; audio asset management lives in the consumer repo (see `reachy-mini/app-architecture` § Audio Asset Management)
- Choreography playback, visualization, or live editing (separate tools, planned)
- Hugging Face publishing of the resulting choreographies (no distribution path in this spec)

## Requirements

### Trigger and activation
- **MUST** ship a precise `description` that activates Claude Code on phrasings like "compose a dance choreography for the Reachy Mini", "lay out a dance for BPM 110", "plan a dance for this song", "Choreographie für Reachy zu Genre X", "erzeuge eine Tanz-Choreographie für Reachy"
- **MUST** include the keywords in the `description`: dance, choreography, Reachy Mini, plan, sections
- **SHOULD** explicitly name when **not** to activate: pure beat detection, implementing the `Move` subclasses, sending direct WebSocket commands to the app, hardware bring-up

### Input parameters
- **MUST** require a **choreography name** as a mandatory parameter, normalized to ASCII kebab-case (e.g. `summer-pop-90s`, `metal-energy-burst`)
- **MUST** accept **at least one** of these fields so tempo and character are derivable: `bpm` (number or range), `genre` (free text), `mood` (free text), `tempo_class` (`slow` / `medium` / `fast` / `variable`)
- **MUST** accept a **total duration** (`duration_s`, integer or float; or alternatively `beats_total`, from which duration is reconstructed using BPM)
- **SHOULD** accept a **platform profile** (`platform: wireless | lite | simulation | any`, default: `any`); the profile decides which platform consequences are surfaced prominently in the plan
- **SHOULD** accept a **section structure** (e.g. `["intro", "verse", "chorus", "verse", "chorus", "bridge", "chorus", "outro"]`); default: derived from `tempo_class` and `duration_s` (see section heuristic below)
- **SHOULD** accept an optional **mood arc** (`mood_arc`, e.g. `["calm", "rising", "peak", "release"]`) that may additionally bias block selection per section
- **SHOULD** make the target path parameterizable (default: `choreographies/<name>.md` relative to the current working directory); when used inside the `reachy-mini-show` repo, `reachy_mini_show/choreographies/` is the natural home
- **MUST NOT** silently default `bpm` when neither `bpm` nor `tempo_class` nor `genre` was provided — abort with a clear error in that case

### Motion catalog as the only slug source
- **MUST** use only slugs that exist as a folder under `spec/reachy-mini/motions/<slug>/`
- **MUST** verify every planned slug against that directory before writing; a missing slug aborts the skill and names the conflict
- **MUST** treat the dance blocks as the **primary channel**: `groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around`
- **SHOULD** use emotion blocks (`happy`, `excited`, `proud`, `surprised`, `shy`, `confused`, `curious`, `sad`, `angry`, `disappointed`, `disgust`, `sleepy`) as **accent inserts** between dance sections or at section transitions, not as the rhythmic main channel
- **SHOULD** also allow social blocks (`bow`, `agreeing-nod`, `disagreeing-shake`, `recognition`, `peek`, `greeting-wave`, `farewell-wave`) as **accent inserts** when the semantics of the song call for it — `bow` for dignified / historical / political tracks, `agreeing-nod` for affirmative hooks, `recognition` for an "aha" moment in a bridge, `greeting-wave` / `farewell-wave` as opening or closing gesture. Selection heuristic: emotion accents carry the *mood*, social accents carry the *gesture* — both may appear in the same choreography, but per section at most one `accent_slug` still holds
- **SHOULD** use state blocks (`waiting-idle`, `alert-listening`, `thinking`) for the outro idle phase after the song or for a calm bridge
- **MUST NOT** include defensive blocks (`flinch`, `alarm`, `scanning`) in any choreography — they are semantically unrelated to dancing
- **MAY** raise an "Open Questions" hint at the end of the choreography about a missing block as a future motion spec — never invent a slug and use it

### Section structure and heuristic
- **MUST** organize the choreography into named sections; valid section types: `intro`, `verse`, `pre-chorus`, `chorus`, `bridge`, `instrumental`, `breakdown`, `outro`, `outro-idle`
- **MUST** record at least these fields per section: `section`, `slug` (dance block), `bpm`, `beats`, `lead_time_s`, `expected_duration_s`, optional `accent_slug` (emotion insert), optional `notes`
- **SHOULD** derive section lengths from the BPM range recommendations of the blocks:
  - `intro` 1–2 blocks, low energy (`waiting-idle` → `sway-side` low BPM)
  - `verse` and `pre-chorus` with `groove-bob` or `sway-side`
  - `chorus` with `groove-bob` higher BPM or `headbang-soft` for energetic music
  - `bridge` optionally with `spin-look-around` or an emotion accent
  - `outro` soft fade back to `groove-bob` or `sway-side` low BPM
  - `outro-idle` `waiting-idle`, with `loop_count: null` for unbounded looping until stop signal
- **MUST** keep the sum of `expected_duration_s` plus all block entry / exit segments (per the motion specs) within ±10 % of the input `duration_s`; on a larger deviation, redistribute sections or adjust beats instead of silently shortening or stretching the song
- **SHOULD** map `mood_arc` onto sections such that the energy peak coincides with a chorus section (e.g. `mood_arc=["calm","rising","peak","release"]` over four sections → the last section receives the most energetic dance pattern)
- **MUST NOT** schedule two consecutive sections of `headbang-soft` without an interleaved cool-down section (`groove-bob`, `sway-side`, or `waiting-idle`) when the platform profile is `wireless` or `lite`

### Hardware and platform validation
- **MUST** check the BPM range of the chosen block against the values in the corresponding motion spec (e.g. `groove-bob` 60–180, `sway-side` 50–140, `headbang-soft` 60–130 on hardware, 60–180 in simulation)
- **MUST** for `headbang-soft` cap the number of consecutive bangs per section and platform:
  - `wireless`: at most 16 bangs per section, mandatory cool-down section (≥ 2 s) after 8 consecutive bangs; flag IMU temperature polling in code
  - `lite`: hard limit of 8 bangs per section without an interleaved cool-down
  - `simulation`: no hardware limit, only a note that on-hardware validation requires the `reachy-mini-on-device` agent
- **MUST** for `spin-look-around` flag that `automatic_body_yaw=False` MUST be set during the sequence (per the motion spec) and that `max_body_yaw` ≤ ±150° must hold
- **MUST** flag brown-out risk when a section drives multiple fully active actuators simultaneously at high BPM (e.g. `sway-side` ≥ 120 BPM with antenna asymmetry plus `automatic_body_yaw=True`); call this out explicitly on Wireless and as reduced risk on Lite (external power supply)
- **MUST** when the platform profile is `simulation`, note which aspects (audio playback, IMU telemetry, real pose reach, servo heat) cannot be validated in simulation and therefore require on-hardware validation
- **MUST** carry every hardware-specific value (cool-down thresholds, servo temperature thresholds, exact hardware BPM caps) that is marked `> ⚠ TBD: validate against real hardware` in `control-surface` as TBD here too — never guess

### Output format
- **MUST** write the plan as a **Markdown file with YAML frontmatter** — frontmatter is the machine-readable layer, the Markdown body is the human-readable layer for the developer
- **MUST** carry at least these fields in the frontmatter:

  ```yaml
  ---
  name: <choreography-slug>
  description: <one-line summary>
  bpm: <number | range>
  genre: <string>
  mood: <string>
  duration_s: <number>
  platform: wireless | lite | simulation | any
  motion_catalog_ref: spec/reachy-mini/motions/
  app_target: reachy-mini-show
  protocol_version: "1.0"   # mirrors the WebSocket protocol of the app
  sections:
    - section: intro
      slug: waiting-idle
      bpm: null
      beats: null
      duration_s: 4.0
      lead_time_s: 0.0
      accent_slug: null
      notes: "soft entry"
    - section: verse
      slug: groove-bob
      bpm: 100
      beats: 16
      duration_s: 9.6
      lead_time_s: 0.05
      accent_slug: null
      notes: ""
    # ...
  warnings:
    - "headbang-soft chorus exceeds the 8-bang limit on Lite — insert a cool-down"
  ---
  ```

- **MUST** carry these sections in the Markdown body, in this order: `# <Title>`, `## Context` (1–3 sentences), `## Section table` (one table containing every section), `## Platform consequences`, `## Translation checklist for the developer`, `## Open questions`
- **MUST** make the `Translation checklist` carry at least these items:
  1. For each section slug, instantiate the `Move` subclass from `reachy_mini_show/behaviors/` or extend the slug registry
  2. Pass BPM, beats, and lead time as constructor parameters
  3. Plan the idle / outro section with `loop_count=None` as background behavior
  4. Align audio triggers with the song on the beat (pointer to `audio-beat-tracking`, planned)
  5. Write a test against `ReachyMini(use_sim=True)`, then on-hardware validation via the `reachy-mini-on-device` agent
  6. Implement platform-specific fallbacks (e.g. `headbang-soft` → `groove-bob` on servo heat)
- **MUST** make the `Open questions` section explicit when the skill detected a gap in the motion catalog (e.g. "A credible reggae bridge would need a `head-tilt-side` block — propose as a motion spec?")
- **SHOULD** cross-link the Markdown body to the underlying motion specs (`spec/reachy-mini/motions/<slug>/de.md`) and to the `app-architecture` spec when protocol-relevant points are touched

### Pre-write validation
- **MUST** check whether a file already exists at `<target_dir>/<name>.md`; on collision, abort and name the conflicting path instead of overwriting
- **MUST** validate the choreography name against ASCII kebab-case; violations abort with a clear error (no silent rewriting)
- **MUST** verify each planned slug against `spec/reachy-mini/motions/<slug>/`; missing slugs abort the skill and name the missing slug
- **MUST** check section duration sum against `duration_s` (see tolerance window above); a violation aborts and proposes a redistribution
- **SHOULD** additionally validate the choreography against the `reachy-mini/app-architecture` spec — `protocol_version` field and the WebSocket `set_dance` convention must stay compatible

### Repository standards
- **MUST** emit the file such that `pre-commit run --all-files` passes without auto-fix changes (LF newlines, no trailing whitespace, valid YAML frontmatter)
- **MUST** mark every hardware-dependent assumption that does not come from a verified source with `> ⚠ TBD: validate against real hardware`
- **SHOULD** return a short summary to the developer after writing: path of the generated file, number of sections, surfaced warnings, pointer to the translation checklist

### Out-of-scope clarification
- **MUST NOT** generate `Move` subclass code, app patches, or WebSocket commands — the skill produces only the choreography file
- **MUST NOT** process an audio file or estimate BPM from an audio file; that is the job of `audio-beat-tracking` (planned)
- **MUST NOT** talk to the `reachy_mini` daemon, a `ReachyMini` instance, or the Hugging Face Spaces repo of the app
- **SHOULD** point at neighbouring skills (`reachy-mini-sdk`, `behavior-scaffold`, `audio-beat-tracking`, agent `reachy-mini-on-device`) instead of duplicating their content

## Acceptance Criteria
- [ ] The skill lives under `skills/dance-choreography/SKILL.md` with valid frontmatter (`name: dance-choreography`, `description`, optional tags) and is accepted by the catalog generator
- [ ] A test invocation with name, BPM, and duration produces a choreography file with YAML frontmatter and Markdown body
- [ ] Every section slug in the generated file exists as a folder under `spec/reachy-mini/motions/`
- [ ] Dance blocks (`groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around`) are the primary slugs in dance sections; emotion and social blocks appear at most as `accent_slug`, never as the rhythmic main channel
- [ ] Per section, at most one `accent_slug` is set — no mixing of emotion and social accents in the same section
- [ ] BPM values per section sit inside the range defined by the motion spec
- [ ] On platform `wireless` or `lite`, a choreography using `headbang-soft` schedules a cool-down section between bursts of ≥ 8 bangs
- [ ] On platform `simulation`, the Markdown body explicitly names which aspects are not validated (audio, IMU, servo heat)
- [ ] The section duration sum sits within ±10 % of the input `duration_s`
- [ ] The frontmatter carries `protocol_version: "1.0"` aligned with the app architecture spec
- [ ] On a name collision the skill aborts and names the existing path
- [ ] On a non-existent slug the skill aborts and names the missing slug
- [ ] Defensive blocks (`flinch`, `alarm`, `scanning`) appear in no choreography
- [ ] The translation checklist is present in every choreography file and lists at least the six mandatory steps
- [ ] `pre-commit run --all-files` passes on the generated file without auto-fix modifications
- [ ] References to `reachy-mini-sdk`, `behavior-scaffold`, `audio-beat-tracking`, and agent `reachy-mini-on-device` are visible in the skill body
- [ ] Hardware-specific values that are TBD in `control-surface` are also marked `> ⚠ TBD: validate against real hardware` in the choreography

## References
- Upstream SDK repo (source of the `Move` subclass, easing modes, pose constants the choreography is translated against): <https://github.com/pollen-robotics/reachy_mini>
- `Move` ABC, `goto` path, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Example move sequences (canonical template for translating the frontmatter into code): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/sequence.py> and <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/recorded_moves.py>
- Upstream Claude skill `motion-philosophy` (Pollen view on motion character): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/motion-philosophy.md>
- App manager and app templates (lifecycle context that `set_dance` runs inside): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps>

## Open questions
- Should the skill optionally offer a second output variant (pure YAML, no Markdown body) once a choreography renderer / linter exists?
- How far may the skill interpret free-form `mood` text (English or German) before becoming an LLM heuristic that is hard to test? Proposal: strict mapping from a fixed set of `mood` tokens (`calm`, `happy`, `melancholic`, `aggressive`, `playful`, `solemn`) onto block preferences, plus a free-text note for special cases.
- Should multiple choreographies for the same song (e.g. `summer-pop-90s.wireless.md` and `summer-pop-90s.lite.md`) live as separate files, or as one file with platform variants in the frontmatter?
- How does the choreography flow into the app — does the developer remain a manual translator, or should a later iteration ship an automatic translator into the slug registry of the app? Proposal: manual path first, automatic translator as a separate skill `dance-choreography-compile`.
- Should the cool-down threshold for `headbang-soft` come from a dedicated `motion-thermal-budget` spec once real servo temperature data is collected on the device?
- Which `mood_arc` token sets are sensible? Proposal: at most eight tokens (`calm`, `rising`, `peak`, `release`, `melancholic`, `playful`, `aggressive`, `solemn`), extensible via an open-question marker.
- Should the skill also allow choreographies for non-musical scenarios (e.g. "a welcome dance when arriving home")? Tendency: yes, as long as `bpm` / `tempo_class` is supplied; otherwise abort as specified above.
- How does the skill integrate with the Pollen CLI app layout if Pollen ever ships a choreography loader in the SDK? Not relevant today, watch over time.
