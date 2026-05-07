# Audio Beat Tracking Skill

Status: draft

## Context

A class of Reachy Mini apps — particularly every dance app — needs two values out of a music source: the **BPM** (for the section parametrisation in [`dance-choreography`](../dance-choreography/en.md)) and a **beat-onset list** (for beat-accurate move triggers at runtime). Today both are reconstructed in every consumer — guessed by ear, measured manually with a metronome, or extracted via an ad-hoc Python snippet against an audio file. That is error-prone (BPM halving / doubling artefacts, onset drift against the perceived beat) and not reusable.

This `audio-beat-tracking` skill fills the gap. It takes an audio source and returns a BPM estimate with a confidence level plus a list of beat onsets as second offsets. The skill is **narrow** — it does not reimplement Pollen's audio stack (`reachy_mini.media.*` stays the owner of capture / playback) and it does not produce move code (that stays with the app). It is the canonical place where consumers like `dance-choreography` ask for BPM and beat onsets instead of reconstructing them.

Term clarification: "beat" here = perceived musical pulse, not the DSP "beat frequency"; "BPM" = beats per minute on the perceived pulse, not on tempo halving / doubling artefacts.

## Goals

- From an audio file, deliver a **BPM estimate** (float, with confidence level `high` / `medium` / `low`) and a **beat-onset list** (second offsets from the file start) that can be dropped directly into the `dance-choreography` section table
- Optionally, on a **live audio stream** (Pollen's mic array via `reachy_mini.media`), deliver BPM and rolling-window onsets for apps that dance to whatever music is actually playing
- Define a canonical convention for the beat-onset representation: a list of seconds-floats from `start_offset_s` of the source, monotonically increasing
- Demarcate platform profiles cleanly: which mode works on simulation (file-only), which works on Lite / Wireless (file or live)
- Make the library choice reproducible and motivated, so consumers know what to pin against

## Non-Goals

- Replace Pollen's audio stack — capture, playback, GStreamer pipeline, mic-array DoA stay in [`reachy_mini.media`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media); this skill is a **consuming** layer, not a replacement
- Move code, dance logic, move-sequence generation — that belongs to the app and to [`dance-choreography`](../dance-choreography/en.md)
- Audio capture from the microphone array itself — the skill takes an audio input (file, AudioFrame, NumPy array); it does not fetch audio
- Genre / mood / vocals classification, stem separation — different tools, separate specs if ever needed
- Lossless stream copy, encoding, format conversion — the skill expects a prepared audio source in Pollen's standard format (F32LE / 48 kHz / 2 ch or mono) or a file decodable by FFmpeg
- Beat-accurate handover of move triggers to the app loop — the skill returns onsets as data; consumption (scheduler, look-ahead, lead-time correction) is the app's job
- Hard real-time onset guarantees under a latency bound — the skill is best-effort with a small look-ahead latency; hard real-time is out of scope

## Requirements

### Trigger and activation

- **MUST** carry a `description` that activates Claude Code on phrasings like "BPM of an audio file", "extract beats for a dance app", "BPM detect", "beat tracking", "onset detection for a Reachy dance", "tempo estimation from audio"
- **MUST** include the keywords in the `description`: BPM, beat, tempo, audio, tracking, detection, Reachy Mini
- **SHOULD** explicitly call out when _not_ to activate: audio-capture tasks (Pollen's SDK does that), pure audio-format conversion (FFmpeg without beat logic), stem separation or vocals detection, BPM guessing from the gut ("a pop song is roughly 120 BPM")

### Input parameters

- **MUST** accept a `source` variant with two modes:
  - `file` — path to an audio file (WAV / MP3 / FLAC / OGG; FFmpeg-decodable). Default mode for authoring (`dance-choreography`).
  - `stream` — a live audio stream as a NumPy array or a Pollen `AudioFrame` iterator over `reachy_mini.media.audio_*`. Mode for app runtime.
- **MUST** in `file` mode accept the input file in any FFmpeg-readable format and resample to `F32LE / 48 kHz / mono` before analysis — no assumption about the source format
- **SHOULD** accept an `expected_bpm_range` parameter (e.g. `(60, 200)`) as a hardening hint against the BPM halving / doubling problem
- **SHOULD** accept `start_offset_s` and `duration_s` parameters to analyse only an audio segment (e.g. "the first 30 s are intro, analyse from 30 s")
- **SHOULD** in `stream` mode accept a `window_s` parameter (default `8.0`) for the rolling-window size of the live BPM estimate
- **MUST NOT** open stream audio from the microphone on its own — the audio input is supplied by the consumer

### Pre-flight (every run, before any analysis)

- **MUST** verify before the first analysis that the chosen beat-tracking library and FFmpeg can be imported in the active Python environment; on failure abort with a clear instruction rather than fall back to silent heuristics
- **MUST** in `file` mode verify file existence and read permissions before decoding
- **MUST** in `stream` mode verify that the supplied sample format matches the analyser's expected format; mismatch is an error, not silent resampling
- **MUST NOT** enter the analysis phase without a green pre-flight

### Library choice

- **MUST** the skill name **one specific beat-tracking library** as default and pin against it, instead of pushing the choice onto consumers — rationale: reproducibility across multiple apps
- **SHOULD** the default suggestion be **`librosa`** (PyPI-stable, MIT/BSD-permissive license, good fit for file-based analysis; `librosa.beat.beat_track` is the canonical entry point); alternatives include `aubio` (C-based, lighter for real-time streams) and `madmom` (DNN-based, highest accuracy, GPL license — caution required for distribution)
- **MUST** the license of the chosen library be named in the skill body; GPL libraries may be picked only when the consuming app itself ships under a GPL-compatible license
- **MUST** pin the library version, so BPM results are reproducible; drift updates follow the `reachy-mini-sdk` drift convention (verify against a major release of the library)

### Analysis — `file` mode

- **MUST** execute file → resample → mono mix → BPM estimate + beat-onset list in this order, without skipping intermediate steps
- **MUST** harden against the BPM halving / doubling phenomenon using `expected_bpm_range` (or a default `(60, 200)`) — results outside the range are re-evaluated with doubled or halved BPM
- **MUST** return a confidence level (`high` / `medium` / `low`), derived from the library's internal confidence value or, when the library does not publish one, from the standard deviation of the inter-beat intervals
- **SHOULD** point at manual cross-checking (listen to the audio, compare with a metronome) in the report when the confidence level is `low` — no silent correction
- **MUST NOT** activate stream logic in file mode or vice versa

### Analysis — `stream` mode

- **MUST** operate on a rolling window (default `8.0 s`) that yields a fresh BPM estimate every 1 s; onsets within the window are returned
- **MUST** name the look-ahead latency (beat detection happens after the beat occurred) explicitly in the report — typically 100–300 ms depending on the library; consumers need this to schedule move triggers correctly
- **SHOULD** return a confidence trace so consumers can spot glitches (BPM jumping from 120 to 240 for one window) and ignore them
- **MUST NOT** buffer or persist stream audio — the skill is stateful only inside the current rolling window, no recording

### Output format

- **MUST** in `file` mode return the following schema:

  ```python
  {
    "mode": "file",
    "source_path": "<resolved abs path>",
    "duration_s": <float>,
    "bpm": <float>,
    "bpm_confidence": "high" | "medium" | "low",
    "beats_s": [<float>, <float>, ...],   # monotonically increasing seconds
    "library": "<name>",
    "library_version": "<x.y.z>",
    "warnings": [<str>, ...]              # e.g. "expected_bpm_range hardened from 240 to 120"
  }
  ```

- **MUST** in `stream` mode return the following schema per window:

  ```python
  {
    "mode": "stream",
    "window_t0_s": <float>,
    "window_t1_s": <float>,
    "bpm": <float>,
    "bpm_confidence": "high" | "medium" | "low",
    "beats_s": [<float>, ...],            # absolute seconds, NOT relative to the window
    "lookahead_ms": <int>,
    "library": "<name>",
    "library_version": "<x.y.z>"
  }
  ```

- **MUST** never include raw audio data in the output — only derived values
- **MUST NOT** include PCM samples, spectrogram images, or logs in the output (analogous to the PII clause in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md))

### Platform profiles

| Platform | `file` mode | `stream` mode | Note |
|---|---|---|---|
| Reachy Mini Wireless | ✓ | ✓ (mic array) | Compute on RPi 4 CM4 — the library choice has to be ARM64-compatible (librosa ✓, aubio ✓, madmom ✓ if TF / PyTorch installs cleanly) |
| Reachy Mini Lite | ✓ | ✓ (mic array via host) | Full host compute, every library is unproblematic |
| Simulation | ✓ | ✗ (no mic) | File-based testing only |

- **MUST** the skill check library availability per platform (in particular ARM64 vs. x86_64); a missing library on a platform is a clear error, no silent skip
- **SHOULD** name the platform the run was executed on in the report

### Consumption by `dance-choreography`

- **MUST** make the `file` mode output drop-in usable for the `bpm: <float>` frontmatter field in a `dance-choreography` file
- **SHOULD** add a note field (`source: audio-beat-tracking`, `analyzed_at: <ISO date>`) to the output, so BPM drift remains traceable later
- **MUST** warn the consumer on a `low` confidence rather than silently writing a `low`-confidence BPM into a choreography

### Out-of-scope clarification

- **MUST NOT** the skill ship move code, move triggers, or move scheduling logic — that is the app's job, possibly with `reachy-mini-sdk` knowledge
- **MUST NOT** the skill read audio from Pollen's mic array on its own — the consumer supplies the audio stream
- **MUST NOT** ship genre / mood classifications — a future separate spec, if ever needed
- **SHOULD** point at [`dance-choreography`](../dance-choreography/en.md) as the primary consumer
- **SHOULD** point at [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/en.md) as the audio-pipeline contract source if stream-format questions arise

## Acceptance Criteria

- [ ] The skill lives at `skills/audio-beat-tracking/SKILL.md` with valid frontmatter (`name: audio-beat-tracking`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The `description` carries the keywords (BPM, beat, tempo, audio, tracking, detection, Reachy Mini) and explicitly calls out at least three anti-triggers
- [ ] `file` mode accepts WAV / MP3 / FLAC / OGG and resamples internally to F32LE / 48 kHz / mono
- [ ] `stream` mode accepts NumPy arrays or Pollen `AudioFrame` iterators and yields a fresh BPM estimate every 1 s with a default rolling-window of 8.0 s
- [ ] BPM halving / doubling hardening via `expected_bpm_range` (default `(60, 200)`) is implemented
- [ ] Output schemas for `file` and `stream` mode are documented and respected
- [ ] Confidence level (`high` / `medium` / `low`) is included in the output, with a warning hint on `low`
- [ ] Default library, license, and pin version are named in the skill body
- [ ] Platform availability (ARM64 / x86_64 / mic array) is verified before analysis; a missing library aborts with a clear message
- [ ] Pollen's audio stack (`reachy_mini.media.*`) is **not** replaced — the skill consumes only, no own audio capture
- [ ] No raw audio data, no spectrograms, no PCM samples in the output
- [ ] Cross-refs to [`dance-choreography`](../dance-choreography/en.md), [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/en.md), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) are visible
- [ ] `pre-commit run --all-files` passes on the skill file

## References

> Source references to Pollen code files point to file plus line number; references to Pollen Markdown sources are cited at file level. Library references point at the official documentation / repo pages.

- Pollen's audio stack (capture, GStreamer pipeline, mic-array DoA): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- Pollen's audio examples (use of `mini.media`): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/sound_record.py>
- librosa beat-tracking docs (default library proposal): <https://librosa.org/doc/latest/generated/librosa.beat.beat_track.html>
- aubio beat-tracking docs (real-time alternative): <https://aubio.org/doc/latest/group__tempo.html>
- madmom DNN-based beat detection (accuracy alternative, GPL): <https://madmom.readthedocs.io/en/latest/modules/features/beats.html>
- Internal cross-refs:
  - [`claude/dance-choreography`](../dance-choreography/en.md) — primary consumer
  - [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/en.md) — audio-pipeline contract
  - [`reachy-mini/app-logging`](../../reachy-mini/app-logging/en.md) — PII clause role model
  - [`claude/reachy-mini-sdk`](../reachy-mini-sdk/en.md) — SDK idioms, in case stream-format questions arise

## Open Questions

- Final library choice: is `librosa` fast enough for both file authoring and real-time on RPi 4 CM4, or does the stream mode need a separate library (e.g. `aubio`)? Proposal: start with `librosa` for both; switch to `aubio` for stream mode if latency issues show up.
- BPM halving / doubling hardening: is a simple range mapping enough, or do we need smarter logic (e.g. autocorrelation peak matching)? Pragmatic first, sharpen later.
- Live beat triggers into the app loop: does a `BeatTracker` skeleton class with an `on_beat` callback belong inside the skill, or is that already move logic and therefore an app concern? Currently flagged as an app concern.
- Mic array vs. line-out: in live dance apps the question is whether the skill analyses the mic input or the audio output (what is actually playing). Mic is sensitive to room reverb, line-out is clean. Consumer decision.
- Look-ahead latency: 100–300 ms is an estimate — verify against real library measurements once hardware is available.
- Format extension: are there use cases for `Opus` or `AAC` beyond the WAV / MP3 / FLAC / OGG default list?
