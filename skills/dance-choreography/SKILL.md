---
name: dance-choreography
description: >-
  Compose a Reachy Mini dance choreography as an authoring artifact — a
  Markdown file with YAML frontmatter that arranges the dance blocks from
  the motion catalog (`groove-bob`, `sway-side`, `headbang-soft`,
  `spin-look-around`) plus emotion accents into named song sections
  (intro, verse, chorus, bridge, outro). The developer then translates the
  file by hand into `Move` subclasses inside the `reachy-mini-show` app.
  Activate on phrasings like "compose a dance choreography for the Reachy
  Mini", "plan a dance for this song", "lay out a choreography for BPM
  110", "design a Reachy dance for genre X", "erzeuge eine Tanz-
  Choreographie für Reachy", "plane einen Tanz zu diesem Lied". Do not
  activate on pure beat / tempo detection from audio (that is
  `audio-beat-tracking`), on writing the actual `Move` subclasses (that is
  `app-scaffold` plus the developer using `reachy-mini-sdk`), on
  sending direct WebSocket commands to the app, or on hardware bring-up.
tags: [reachy-mini, dance, choreography, authoring]
---

# Dance Choreography

This skill produces an **authoring artifact**, not robot code. The output is a Markdown file with YAML frontmatter that a developer translates by hand into `Move` subclasses or move sequences inside the `reachy-mini-show` app (see `spec/reachy-mini/app-architecture/de.md`).

## Source of truth (in order, on conflict)

1. **Motion catalog** — `spec/reachy-mini/motions/<slug>/de.md` (canonical) and `en.md` (translation). Every slug used in a choreography MUST exist here.
2. **Motor positions** — `spec/reachy-mini/motor-positions/de.md` for per-joint URDF limits, T1–T8 live-verified targets, three-layer validity model, and **pitch-bleed coupling on roll / heave-up**.
3. **Motion-anomaly detection** — `spec/reachy-mini/motion-anomaly-detection/de.md` for the four anomaly classes with binding rules and the unified anomaly-event-record schema. This skill is the **composition-time pre-flight consumer**: enforce Class A (pose target within Pollen nominal range), Class B (pose-delta/dt below `0.16 rad/sample` warning, `0.30 rad/sample` hard reject), Class C (antenna rest-setpoint `|x| ≥ 5°`) before a choreography section is allowed to lock.
4. **Control surface** — `spec/reachy-mini/control-surface/de.md` for hardware limits, BPM ranges, easing modes, brown-out and servo-heat constraints.
5. **App architecture** — `spec/reachy-mini/app-architecture/de.md` for the WebSocket protocol version, slug registry conventions, and platform profiles.
6. **This skill** — curated workflow; loses to the sources above on conflict.

Before writing a choreography, **open the relevant motion specs** and confirm BPM range, beat structure, and platform profile.

### Pose-composition constraints (live-verified)

When composing dance blocks, three platform constraints from the 2026-05-13 live verification of motor-positions apply directly. Carry them as comments / `mood_notes` on every section that triggers them.

- **Pitch bleed.** A pure-roll command bleeds **−3.8°** of pitch for every +25° of roll (T3 measurement), and a pure heave-up command bleeds **+2.4°** of pitch for every +15 mm of z (T5). Choreographies that stack roll-driven sway (`sway-side`, `headbang-soft` with roll) with intentional pitch motion (`groove-bob`, `agreeing-nod`) **MUST** account for the bleed: either reduce the pitch amplitude, or accept the visible coupling and label it in the choreography document. Pure-pitch, pure-yaw, and pure heave-down sections show no measurable bleed.
- **Pollen nominal operations range is the only binding layer.** The IK accepts pose targets up to ±π per Stewart joint (kinematics-data.json software default), but only the Pollen nominal range (±40° pitch/roll, ±60° head-yaw, ±155° body-yaw) is mechanically safe. The 2026-05-12 Phase-B self-collision incident is the canonical precedent. A choreography **MUST NOT** quote pose values outside the Pollen nominal range, even if a motion spec lists them as IK-feasible.
- **Body-yaw clip at ±65° relative to head.** A `head_yaw=0, body_yaw=+90°` request is silently clipped to `body_yaw ≈ +65°` by `max_relative_yaw=65°` in the IK solver. Spin moves like `spin-look-around` that exceed ±65° relative yaw will not deliver the value they request — split the rotation across `head_yaw` + `body_yaw` explicitly, or accept the soft clip.

## When this skill activates

- the user wants to plan a dance for a concrete song, genre, or mood
- the user asks to compose a Reachy choreography around a BPM, a tempo class, or a section structure (intro / verse / chorus / bridge / outro)
- the user wants a choreography file that another developer can pick up and translate into `Move` subclasses

## When NOT to activate

- pure beat / tempo detection from an audio file → `audio-beat-tracking` (planned)
- writing the actual `Move` subclasses → `app-scaffold` plus developer with `reachy-mini-sdk`
- sending live WebSocket commands to the app → app's own surface (see `spec/reachy-mini/app-architecture`)
- composing a brand-new dance block (a new motion spec) → `nolte-shared:spec` plus the motion-catalog convention
- on-device test or live deployment → agent `reachy-mini-on-device`

## Inputs

Collect from the user before generating anything:

| Field | Required | Default |
|---|---|---|
| `name` | yes — ASCII kebab-case, validated | — |
| `bpm` *or* `tempo_class` *or* `genre` | at least one of the three | — |
| `duration_s` *or* `beats_total` | yes (one of the two) | — |
| `platform` | no — `wireless` / `lite` / `simulation` / `any` | `any` |
| `mood` | no — short free-text label | — |
| `mood_arc` | no — list of mood tokens (`calm`, `rising`, `peak`, `release`, `melancholic`, `playful`, `aggressive`, `solemn`) | derived from `mood` and section count |
| `sections` | no — explicit section structure | derived from `tempo_class` and `duration_s` |
| `target_dir` | no | `choreographies/` in the current working directory |

If `name` is missing or violates kebab-case, stop and report — do not silently rewrite the input. If none of `bpm` / `tempo_class` / `genre` is supplied, stop and report — never default the BPM silently.

## Pre-write validation

Run all of these **before** creating any file. Stop and report on the first failure:

1. **Name collision** — if `<target_dir>/<name>.md` already exists, abort and quote the existing path. Never overwrite.
2. **Name shape** — ASCII kebab-case, length ≤ 60 characters.
3. **Slug existence** — for every block planned, confirm a folder under `spec/reachy-mini/motions/<slug>/`. A missing slug aborts and names the conflict.
4. **BPM range per block** — read the block's motion spec (e.g. `groove-bob` 60–180, `sway-side` 50–140, `headbang-soft` 60–130 on hardware / 60–180 in simulation). A BPM outside the range aborts.
5. **Duration tolerance** — sum of `expected_duration_s` plus block entry / exit segments stays within ±10 % of the input `duration_s`. Out of tolerance aborts and proposes a redistribution; do not silently shrink or stretch the song.
6. **Cool-down rule** — on `platform: wireless` or `lite`, two consecutive `headbang-soft` sections without an interleaved cool-down section (`groove-bob`, `sway-side`, `waiting-idle`) abort.
7. **Defensive blocks** — `flinch`, `alarm`, `scanning` in any section abort (they are not dance blocks).

## What gets generated

A single file: `<target_dir>/<name>.md`. The shape is fixed:

```markdown
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
protocol_version: "1.0"
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

# <Titel>

## Kontext
<1–3 sentences: song / mood / target platform>

## Sektions-Tabelle
| # | Section | Slug | BPM | Beats | Duration (s) | Lead time (s) | Accent | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | intro | waiting-idle | — | — | 4.0 | 0.0 | — | soft entry |
| 2 | verse | groove-bob | 100 | 16 | 9.6 | 0.05 | — | |
| ... |

## Plattform-Konsequenzen
- Wireless: <e.g. servo heat polling needed for headbang-soft>
- Lite: <e.g. hard 8-bang limit, no IMU read>
- Simulation: <e.g. no audio, no servo heat — use reachy-mini-on-device for hardware proof>

## Übersetzungs-Checkliste für den Entwickler
1. For each section slug, instantiate the matching `Move` subclass from `reachy_mini_show/behaviors/` or extend the slug registry.
2. Pass BPM, beats, and lead time as constructor parameters (e.g. `GrooveBob(bpm=100, beats=16, lead_time_s=0.05)`); inside `Move.evaluate(t)`, compute against **`t_beats = elapsed_s * bpm / 60.0`** rather than seconds — Pollen's convention for BPM-synchronised moves (source: Pollen `symbolic-motion.md`).
3. Plan the idle / outro section with `loop_count=None` as background behavior.
4. Align audio triggers with the song on the beat — see `audio-beat-tracking` (planned) for the BPM extraction path.
5. Write a test against `ReachyMini(use_sim=True)`, then on-hardware validation via the `reachy-mini-on-device` agent.
6. Implement platform-specific fallbacks (e.g. `headbang-soft` → `groove-bob` on servo heat).
7. For emotion `accent_slug`s, consume the official emotions library `pollen-robotics/reachy-mini-emotions-library` via `RecordedMoves(...).get(<slug>)` rather than re-composing the emotion (source: Pollen `AGENTS.md` § Emotions Library).

## Offene Fragen
- <only when the skill detected a real gap — e.g. a missing block in the motion catalog>
```

The exact block selection per section follows the section heuristic from the spec:

- `intro`: `waiting-idle` or `sway-side` at low BPM
- `verse` / `pre-chorus`: `groove-bob` or `sway-side`
- `chorus`: `groove-bob` at higher BPM, or `headbang-soft` for energetic music
- `bridge`: `spin-look-around` (with the `automatic_body_yaw=False` and `max_body_yaw ≤ ±150°` constraint surfaced in the section's `notes`), or an emotion accent
- `outro`: soft fade back to `groove-bob` or `sway-side` at low BPM
- `outro-idle`: `waiting-idle` with `loop_count: null` (unbounded, until external stop)

Emotion blocks (`happy`, `excited`, `proud`, `surprised`, `shy`, `confused`, `curious`, `sad`, `angry`, `disappointed`, `disgust`, `sleepy`) are allowed only as `accent_slug` between dance sections — never as the rhythmic main slug. Social blocks (`bow`, `agreeing-nod`, `disagreeing-shake`, `recognition`, `peek`, `greeting-wave`, `farewell-wave`) are also allowed as `accent_slug` when the song's semantics call for a *gesture* rather than a *mood* — `bow` for dignified / historical / political tracks, `agreeing-nod` for affirmative hooks, `recognition` for an "aha" moment in a bridge, `greeting-wave` / `farewell-wave` as opening or closing gesture. Per section, at most one `accent_slug` is set — emotion and social accents never mix in the same section. State blocks (`waiting-idle`, `alert-listening`, `thinking`) are allowed for `intro` / `bridge` / `outro-idle`. Defensive blocks (`flinch`, `alarm`, `scanning`) are forbidden in any choreography.

## Hard rules / out of scope

- **MUST NOT** write `Move` subclass code, app patches, or WebSocket commands. The skill's only artifact is the choreography Markdown file.
- **MUST NOT** invent slugs. Every slug in the choreography refers to an existing folder under `spec/reachy-mini/motions/<slug>/`. If a missing block would clearly improve the choreography, raise it in the `Open questions` section as a future motion-spec proposal — never use the slug as if it existed.
- **MUST NOT** process audio files or estimate BPM from audio. That is the job of `audio-beat-tracking` (planned).
- **MUST NOT** silently default `bpm`. If neither `bpm` nor `tempo_class` nor `genre` is supplied, abort with a clear error.
- **MUST NOT** overwrite an existing choreography file. On collision, abort and report the path.
- **MUST NOT** include defensive blocks (`flinch`, `alarm`, `scanning`) in any choreography.
- **MUST** carry every hardware-specific value that is `> ⚠ TBD: validate against real hardware` in `control-surface` (e.g. exact servo temperature thresholds, exact hardware BPM ceilings) as TBD in the choreography too — never guess.
- **MUST** schedule an interleaved cool-down section between `headbang-soft` bursts of ≥ 8 bangs on `wireless` or `lite`.
- **MUST** emit a file that passes `pre-commit run --all-files` without auto-fix changes (LF newlines, no trailing whitespace, valid YAML frontmatter).
- **MUST** carry `protocol_version: "1.0"` in the frontmatter, mirroring the WebSocket protocol of the app (see `spec/reachy-mini/app-architecture/de.md`).
- **MUST** delegate concerns owned by neighbouring skills (see below) instead of growing this skill into them.

## Boundaries to neighbouring skills

- SDK knowledge / idiomatic API use → `reachy-mini-sdk`
- Scaffolding the app or a new behavior → `app-scaffold`
- Audio beat / tempo / BPM extraction → `audio-beat-tracking` (planned)
- New motion specs → `nolte-shared:spec` plus the motion-catalog convention under `spec/reachy-mini/motions/`
- Live deployment / on-device verification of the resulting dance → agent `reachy-mini-on-device`
- Home Assistant triggers around the dance → `home-assistant-bridge`

## Next-steps checklist (returned to the developer after writing)

1. Open `<target_dir>/<name>.md` and read the section table plus warnings.
2. For each section slug, look up the corresponding motion spec under `spec/reachy-mini/motions/<slug>/de.md` (or `en.md`) for pose tables, easing, and acceptance criteria.
3. Translate sections into `Move` subclasses or move sequences inside `reachy-mini-show/reachy_mini_show/behaviors/dance/` — see `spec/reachy-mini/app-architecture/de.md` § Behavior-Implementierung.
4. Wire BPM, beats, lead time, and any `accent_slug` into the constructor calls; keep the slug registry in `behaviors/__init__.py` aligned.
5. Pick the right idle behavior for the outro-idle section (`waiting-idle` is the default).
6. Run the test stub against `ReachyMini(use_sim=True)`; then dispatch the `reachy-mini-on-device` agent for hardware validation against the platform profile in the frontmatter.
7. If the choreography needs a block that does not exist yet, open a new motion spec under `spec/reachy-mini/motions/<slug>/` with `nolte-shared:spec` before extending this choreography.
