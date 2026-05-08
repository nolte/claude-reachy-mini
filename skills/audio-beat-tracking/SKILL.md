---
name: audio-beat-tracking
description: Extract BPM and beat onsets from a Reachy Mini audio source — file or live stream — and return values that drop directly into a `dance-choreography` section table or feed app-runtime motion sync. Activate on phrasings like "BPM einer Audio-Datei", "extract beats for a dance app", "BPM detect", "beat tracking", "onset detection für Reachy-Tanz", "tempo estimation from audio". Do not activate for audio-capture tasks (Pollen's `reachy_mini.media.*` does that), pure audio-format conversion (FFmpeg without beat logic), stem separation or vocals detection, or BPM guessing from the gut — this skill always returns a measured value with a confidence level.
tags: [reachy-mini, audio, beat, tempo, bpm, dance]
---

# Audio Beat Tracking

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/audio-beat-tracking/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/audio-beat-tracking/en.md).

## When this skill activates

Use this skill when the developer wants to:

- get a **BPM** with a confidence level for a song the dance app should follow
- get a **beat-onset list** (seconds-floats) that maps onto the section structure of a `dance-choreography` artefact
- run a rolling-window live BPM estimate on a Pollen `AudioFrame` stream from the mic array

## When NOT to activate

- audio capture from the Reachy Mini mic array → Pollen's `reachy_mini.media.audio_*`
- pure audio-format conversion (e.g. WAV → MP3) → FFmpeg directly
- stem separation, vocals detection, genre or mood classification → external tooling, separate spec if ever needed
- BPM guessing from the gut ("a pop song is roughly 120 BPM") → this skill always measures
- composing dance moves around the beat → developer's job, supported by `dance-choreography` and `reachy-mini-sdk`

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `source` | yes | — | `{"mode": "file", "path": "<abs_path>"}` or `{"mode": "stream", "frames": <iterator>}`; `stream` mode expects Pollen `AudioFrame` iterators or NumPy arrays |
| `expected_bpm_range` | no | `(60, 200)` | Hardening hint against BPM halving / doubling. Out-of-range candidates are re-evaluated with doubled or halved BPM |
| `start_offset_s` | no | `0.0` | File-mode only: skip an intro before analysis |
| `duration_s` | no | full length | File-mode only: bound the analysis window |
| `window_s` | no | `8.0` | Stream-mode only: rolling-window size; a fresh BPM estimate is yielded every 1 s |

The skill picks the canonical library — **`librosa`** by default (PyPI-stable, MIT/BSD-permissive) — and pins it. Alternatives `aubio` (real-time) and `madmom` (DNN, GPL) are documented in the spec but not the default.

## Hard rules

1. **Pollen's audio stack is not replaced.** The skill consumes audio (file path or AudioFrame iterator); it never opens the mic array on its own. Capture, playback, and the GStreamer pipeline stay in `reachy_mini.media.*`.
2. **No raw audio in the output.** PCM samples, spectrogram images, and decoded frames stay inside the analysis pass. The output carries derived values only — analogous PII clause to `reachy-mini/app-logging`.
3. **Library availability is verified before analysis.** Missing `librosa`, missing FFmpeg, ARM64 incompatibility — all explicit errors with a remediation hint, never a silent fallback.
4. **Confidence levels are honest.** A `low` confidence triggers a warning in the output ("verify against a metronome / listen back") instead of a silent assertion.
5. **Stream mode is rolling, not buffered.** The skill never accumulates audio beyond the active window; no recording, no on-disk persistence.

## Pre-flight (every run, in order — abort on first failure)

1. **Library available** — `python -c "import librosa, ffmpeg"` (or whichever pinned version) resolves; on failure abort with `uv pip install librosa ffmpeg-python`.
2. **File-mode**: target file exists and is readable; FFmpeg can decode the format.
3. **Stream-mode**: the supplied sample format (rate, channels, dtype) matches the analyser; mismatch is an error, not silent resampling.
4. **Platform**: ARM64 vs. x86_64 wheel availability for the pinned library; `librosa` and `aubio` work on both, `madmom` only when TF / PyTorch is installable.

## Workflow — file mode

1. Resolve the source file to an absolute path; resample internally to **F32LE / 48 kHz / mono** before analysis.
2. Run `librosa.beat.beat_track` (or the pinned library's equivalent) on the prepared signal, scoped by `start_offset_s` and `duration_s` if supplied.
3. **BPM hardening**: if the returned BPM falls outside `expected_bpm_range`, re-evaluate with halved / doubled BPM and pick the candidate inside the range; record the original-vs-corrected value in `warnings`.
4. Compute the **confidence level**: `high` when the library exposes a stable score (e.g. `librosa`'s default ≥ 0.5 plus tight inter-beat spacing); `medium` when the score is borderline; `low` when the inter-beat-interval standard deviation is high.
5. Emit the file-mode output schema (see below).

## Workflow — stream mode

1. Open a rolling buffer of `window_s` seconds (default 8.0).
2. Every 1 s, run beat tracking on the current window and emit the per-window output schema with the absolute `window_t0_s` / `window_t1_s` and a `lookahead_ms` hint (typically 100–300 ms for `librosa`).
3. Continue until the consumer stops the iterator. **Never persist** the stream beyond the live window.

## Output schemas

**File mode:**

```python
{
  "mode": "file",
  "source_path": "<resolved abs path>",
  "duration_s": <float>,
  "bpm": <float>,
  "bpm_confidence": "high" | "medium" | "low",
  "beats_s": [<float>, <float>, ...],   # monotonically increasing seconds
  "library": "librosa",
  "library_version": "<x.y.z>",
  "warnings": [<str>, ...]              # e.g. "expected_bpm_range hardened from 240 to 120"
}
```

**Stream mode (per window):**

```python
{
  "mode": "stream",
  "window_t0_s": <float>,
  "window_t1_s": <float>,
  "bpm": <float>,
  "bpm_confidence": "high" | "medium" | "low",
  "beats_s": [<float>, ...],            # absolute seconds, NOT relative to the window
  "lookahead_ms": <int>,
  "library": "librosa",
  "library_version": "<x.y.z>"
}
```

## Consumption hints

- **`dance-choreography` authoring**: drop `bpm` straight into the choreography frontmatter `bpm: <float>` and `beats_s` length into the section table per section. Add a `source: audio-beat-tracking, analyzed_at: <ISO date>, library: librosa==<ver>` provenance comment alongside.
- **App runtime**: in stream mode, hand `beats_s` to a scheduler that respects `lookahead_ms`. The skill does not own the scheduler — that lives in the app, supported by `reachy-mini-sdk` for SDK idioms.
- **Low confidence**: surface the warning to the caller; do not silently write a `low`-confidence BPM into a choreography file.

## Boundaries to neighbouring skills

- choreography artefact authoring → `dance-choreography`
- SDK idioms (single-owner loop, method choice between `goto_target` / `set_target`, safe-torque) → `reachy-mini-sdk`
- audio capture / playback / GStreamer pipeline → Pollen's `reachy_mini.media.*` (out of skill scope)
- live test on the device → agent `reachy-mini-on-device`

External canonical sources (cited via the spec, not duplicated here): `librosa.beat.beat_track`, `aubio` (real-time alternative), `madmom` (DNN alternative, GPL), Pollen's audio stack under `src/reachy_mini/media/`.
