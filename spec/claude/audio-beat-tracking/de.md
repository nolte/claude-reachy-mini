# Audio-Beat-Tracking-Skill

Status: draft

## Kontext

Eine Reihe von Reachy-Mini-Apps (insbesondere alle Tanz-Apps) braucht zwei Größen aus einer Musik-Quelle: die **BPM** (zur Sektions-Parametrisierung in [`dance-choreography`](../dance-choreography/de.md)) und eine **Beat-Onset-Liste** (für taktgenaue Move-Trigger zur Laufzeit). Heute wird beides in jedem Konsumenten neu erschlossen — entweder aus dem Bauch geschätzt, manuell mit einem Metronom abgemessen, oder per ad-hoc Python-Snippet aus einer Audio-Datei extrahiert. Das ist fehleranfällig (BPM-Halbierungs-/Verdopplungs-Fehler, Onset-Drift gegenüber dem subjektiven Hörbild) und nicht wiederverwendbar.

Dieser Skill `audio-beat-tracking` füllt die Lücke. Er nimmt eine Audio-Quelle entgegen, liefert eine BPM-Schätzung mit Vertrauensgrad und eine Liste von Beat-Onsets als Sekunden-Offsets. Der Skill ist **schmal** — er macht weder Pollens Audio-Stack neu (`reachy_mini.media.*` bleibt für Capture/Playback zuständig), noch liefert er Move-Code (das ist Sache der App). Er ist die kanonische Stelle, an der Konsumenten wie `dance-choreography` BPM und Beat-Onsets nachfragen, statt sie selbst zu rekonstruieren.

Begriffsklärung: „Beat" hier = wahrgenommener Schlag im Sinne der musikalischen Pulsation, nicht der DSP-Begriff „beat frequency"; „BPM" = Beats Per Minute auf den wahrgenommenen Puls bezogen, nicht auf Tempo-Halbierungs-/Verdopplungs-Artefakte.

## Ziele

- Aus einer Audio-Datei eine **BPM-Schätzung** (Float, mit Vertrauensgrad `high` / `medium` / `low`) und eine **Beat-Onset-Liste** (Sekunden-Offsets ab Datei-Anfang) liefern, die für die `dance-choreography`-Sektions-Tabelle direkt nutzbar ist
- Optional auf einem **Live-Audio-Stream** (Pollens Mic-Array über `reachy_mini.media`) BPM und rolling-window-Onsets liefern, für Apps, die zur tatsächlich gespielten Musik tanzen
- Eine kanonische Konvention für die Beat-Onset-Repräsentation festlegen: Liste von Sekunden-Floats ab `start_offset_s` der Quelle, monotonic increasing
- Plattform-Profile sauber abgrenzen: was geht in Simulation (Datei-basiert), was geht auf Lite/Wireless (Datei oder Live)
- Eine reproduzierbare Library-Wahl mit klarem Begründungs-Pfad — Konsumenten sollen wissen, wogegen sie pinnen

## Nicht-Ziele

- Pollens Audio-Stack ersetzen — Capture, Playback, GStreamer-Pipeline, Mic-Array-DoA bleiben in [`reachy_mini.media`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media); dieser Skill ist **konsumierender** Layer, nicht Ersatz
- Move-Code, Tanz-Logik, Move-Sequenz-Generierung — das ist Sache der App und von [`dance-choreography`](../dance-choreography/de.md)
- Audio-Capture vom Mikrofon-Array selbst — der Skill nimmt einen Audio-Eingang entgegen (Datei, AudioFrame, NumPy-Array), holt sich Audio nicht selber
- Genre-/Mood-/Stimmungs-Klassifikation, Stem-Separation, Vocals-Erkennung — andere Tools, andere Specs falls überhaupt benötigt
- Verlustfreie Stream-Kopie, Encoding, Format-Konvertierung — der Skill erwartet eine vorbereitete Audio-Quelle in Pollens Standard-Format (F32LE / 48 kHz / 2 ch oder mono) oder einer per FFmpeg konvertierbaren Datei
- Beat-genaue Move-Trigger-Übergabe an die App-Loop — der Skill liefert Onsets als Daten; die Konsumption (Scheduler, Look-Ahead, Lead-Time-Korrektur) ist Sache der App
- Real-Time-Onset-Garantien unter Latenz-Bound — der Skill ist beste-effort und liefert mit kleiner Lookahead-Latenz; harte Echtzeit ist außerhalb des Scopes

## Anforderungen

### Trigger und Aktivierung

- **MUSS [MUST]** eine `description` liefern, die Claude Code aktiviert auf Formulierungen wie „BPM einer Audio-Datei", „Beats für eine Tanz-App extrahieren", „BPM detect", „beat tracking", „onset detection für Reachy-Tanz", „Tempo-Schätzung aus Audio"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: BPM, Beat, Tempo, audio, tracking, detection, Reachy Mini
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist: bei Audio-Capture-Tasks (das macht Pollens SDK selbst), bei reiner Audio-Format-Konvertierung (FFmpeg ohne Beat-Logik), bei Stem-Separation oder Vocals-Detection, bei BPM-Schätzungen aus dem Bauch („ungefähr 120 BPM ist ein Pop-Song")

### Eingabe-Parameter

- **MUSS [MUST]** eine `source`-Variante annehmen, mit zwei Modi:
  - `file` — Pfad zu einer Audio-Datei (WAV / MP3 / FLAC / OGG; FFmpeg-decodierbar). Default-Modus für Authoring (`dance-choreography`)
  - `stream` — Live-Audio-Stream als NumPy-Array oder als Pollen-`AudioFrame`-Iterator über `reachy_mini.media.audio_*`. Modus für App-Runtime
- **MUSS [MUST]** im `file`-Modus die Eingangs-Datei akzeptieren in jedem von FFmpeg geöffneten Format und auf `F32LE / 48 kHz / mono` resamplen, bevor sie analysiert wird — keine Annahme über das Quell-Format
- **SOLLTE [SHOULD]** einen `expected_bpm_range`-Parameter annehmen (z. B. `(60, 200)`) als Hilfe gegen das BPM-Halbierungs-/Verdopplungs-Problem
- **SOLLTE [SHOULD]** einen `start_offset_s` und `duration_s` Parameter annehmen, um nur ein Audio-Segment zu analysieren (z. B. „die ersten 30 s sind Intro, analysiere ab 30 s")
- **SOLLTE [SHOULD]** im `stream`-Modus einen `window_s`-Parameter annehmen (Default `8.0`) für die rolling-window-Größe der Live-BPM-Schätzung
- **DARF NICHT [MUST NOT]** Stream-Audio aus dem Mikrofon eigenständig öffnen — der Audio-Eingang wird vom Konsumenten geliefert

### Pre-Flight-Pflichten (vor jeder Analyse)

- **MUSS [MUST]** vor der ersten Analyse prüfen, dass die gewählte Beat-Tracking-Library und FFmpeg im aktiven Python-Umfeld geladen werden können; bei Fehlschlag mit klarer Anleitung abbrechen, statt eine schweigende Heuristik zu nutzen
- **MUSS [MUST]** im `file`-Modus die Datei-Existenz und Lese-Permissions vor dem Decodieren prüfen
- **MUSS [MUST]** im `stream`-Modus prüfen, dass das gelieferte Sample-Format mit dem analyzer-erwarteten Format kompatibel ist; Mismatch ist Fehler, kein automatisches Resampling
- **DARF NICHT [MUST NOT]** ohne Pre-Flight-OK in die Analyse-Phase eintreten

### Library-Wahl

- **MUSS [MUST]** der Skill **eine konkrete Beat-Tracking-Library** als Default benennen und gegen sie pinnen, statt die Wahl an den Konsumenten zu schieben — Begründung: Reproduzierbarkeit über mehrere Apps hinweg
- **SOLLTE [SHOULD]** Default-Vorschlag **`librosa`** sein (PyPI-stabil, MIT/BSD-permissive Lizenz, gut für File-basierte Analyse; `librosa.beat.beat_track` ist die kanonische Funktion); Alternative `aubio` (C-basiert, leichter für Real-Time-Streams), Alternative `madmom` (DNN-basiert, höchste Genauigkeit, GPL-Lizenz — daher Vorsicht bei Distribution)
- **MUSS [MUST]** die Lizenz der gewählten Library im Skill-Body benannt sein; GPL-Libraries dürfen nur dann gewählt werden, wenn die Konsumenten-App selbst GPL-kompatibel veröffentlicht
- **MUSS [MUST]** die Library-Version pinnen, damit BPM-Resultate reproduzierbar sind; Drift-Update folgt der `reachy-mini-sdk`-Drift-Konvention (gegen ein Major-Release der Library prüfen)

### Analyse — `file`-Modus

- **MUSS [MUST]** Datei → resample → Mono-Mix → BPM-Schätzung + Beat-Onset-Liste in dieser Reihenfolge ausführen, ohne Zwischenschritte zu überspringen
- **MUSS [MUST]** das BPM-Halbierungs-/Verdopplungs-Phänomen mit dem `expected_bpm_range` (oder einem Default `(60, 200)`) härten — Resultate außerhalb des Range werden mit verdoppeltem oder halbiertem BPM nochmal evaluiert
- **MUSS [MUST]** einen Vertrauensgrad-Indikator (`high` / `medium` / `low`) liefern, abgeleitet aus der Library-internen Konfidenz oder, falls die Library keine liefert, aus der Standard-Abweichung der Inter-Beat-Intervalle
- **SOLLTE [SHOULD]** bei `low`-Vertrauen im Report-Hinweis auf manuelles Nachprüfen (Audio anhören, mit Metronom vergleichen) verweisen — keine versteckte Korrektur
- **DARF NICHT [MUST NOT]** im File-Modus Stream-Logik aktivieren oder umgekehrt

### Analyse — `stream`-Modus

- **MUSS [MUST]** mit einem rolling-window arbeiten (Default `8.0 s`), das alle 1 s ein neues BPM-Schätzwert liefert; Onsets innerhalb des Windows werden zurückgegeben
- **MUSS [MUST]** Look-Ahead-Latenz (Beat-Erkennung passiert nach dem Beat-Auftreten) im Report explizit nennen — typisch 100–300 ms je nach Library; Konsumenten brauchen das, um Move-Trigger korrekt zu schedulen
- **SOLLTE [SHOULD]** einen Confidence-Verlauf liefern, damit Konsumenten Glitches (BPM springt von 120 auf 240 für eine Window) erkennen und ignorieren können
- **DARF NICHT [MUST NOT]** Stream-Audio puffern und persistieren — der Skill ist stateful nur innerhalb des aktuellen rolling-window, kein Recording

### Output-Format

- **MUSS [MUST]** im `file`-Modus folgendes Schema zurückgeben:

  ```python
  {
    "mode": "file",
    "source_path": "<resolved abs path>",
    "duration_s": <float>,
    "bpm": <float>,
    "bpm_confidence": "high" | "medium" | "low",
    "beats_s": [<float>, <float>, ...],   # monotonic increasing seconds
    "library": "<name>",
    "library_version": "<x.y.z>",
    "warnings": [<str>, ...]              # e.g. "expected_bpm_range hardened from 240 to 120"
  }
  ```

- **MUSS [MUST]** im `stream`-Modus folgendes Schema pro Window zurückgeben:

  ```python
  {
    "mode": "stream",
    "window_t0_s": <float>,
    "window_t1_s": <float>,
    "bpm": <float>,
    "bpm_confidence": "high" | "medium" | "low",
    "beats_s": [<float>, ...],            # absolute seconds, NOT relative to window
    "lookahead_ms": <int>,
    "library": "<name>",
    "library_version": "<x.y.z>"
  }
  ```

- **MUSS [MUST]** Audio-Roh-Daten **niemals** in den Output aufnehmen — nur abgeleitete Werte
- **DARF NICHT [MUST NOT]** PCM-Samples, Spektrogramm-Bilder oder Logs in den Output schreiben (analoge Klausel zur PII-Klausel in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md))

### Plattform-Profile

| Plattform | `file`-Modus | `stream`-Modus | Bemerkung |
|---|---|---|---|
| Reachy Mini Wireless | ✓ | ✓ (Mic-Array) | Compute auf RPi 4 CM4 — Library-Wahl muss ARM64-kompatibel sein (librosa ✓, aubio ✓, madmom ✓ falls TF/PyTorch installierbar) |
| Reachy Mini Lite | ✓ | ✓ (Mic-Array via Host) | Volle Host-Compute, alle Libraries unproblematisch |
| Simulation | ✓ | ✗ (kein Mic) | Nur Datei-basiert testbar |

- **MUSS [MUST]** der Skill plattform-spezifisch auf Library-Verfügbarkeit prüfen (insbesondere ARM64 vs. x86_64); fehlende Library auf einer Plattform → klare Fehlermeldung, kein silent-skip
- **SOLLTE [SHOULD]** im Report die Plattform benennen, gegen die der Lauf ausgeführt wurde

### Konsumption durch `dance-choreography`

- **MUSS [MUST]** das `file`-Modus-Output direkt in das Frontmatter-Feld `bpm: <float>` einer `dance-choreography`-Datei einsetzbar sein
- **SOLLTE [SHOULD]** das Output mit einem Notiz-Feld (`source: audio-beat-tracking`, `analyzed_at: <ISO-Datum>`) ergänzt werden, damit BPM-Drift später nachvollziehbar ist
- **MUSS [MUST]** der Skill auf `low`-Vertrauen den Konsumenten warnen, statt einen `low`-BPM stillschweigend in eine Choreographie zu schreiben

### Out-of-Scope-Klarstellung

- **DARF NICHT [MUST NOT]** der Skill Move-Code, Move-Trigger oder Move-Scheduling-Logik enthalten — das ist Aufgabe der App, ggf. mit `reachy-mini-sdk`-Wissen
- **DARF NICHT [MUST NOT]** der Skill Audio aus Pollens Mic-Array selbst lesen — der Konsument liefert den Audio-Stream
- **DARF NICHT [MUST NOT]** Genre-, Mood- oder Stimmungs-Schätzungen liefern — eine künftige separate Spec, falls überhaupt benötigt
- **SOLLTE [SHOULD]** auf [`dance-choreography`](../dance-choreography/de.md) als Haupt-Konsument verweisen
- **SOLLTE [SHOULD]** auf [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/de.md) als Audio-Pipeline-Vertrag-Quelle verweisen, falls Stream-Format-Fragen aufkommen

## Akzeptanzkriterien

- [ ] Skill ist unter `skills/audio-beat-tracking/SKILL.md` mit gültiger Frontmatter (`name: audio-beat-tracking`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description` enthält die Schlüsselbegriffe (BPM, Beat, Tempo, audio, tracking, detection, Reachy Mini) und benennt mindestens drei Anti-Trigger explizit
- [ ] `file`-Modus akzeptiert WAV / MP3 / FLAC / OGG und resamplet intern auf F32LE / 48 kHz / mono
- [ ] `stream`-Modus akzeptiert NumPy-Arrays oder Pollen-`AudioFrame`-Iteratoren und liefert pro 1 s ein neues BPM-Schätzwert mit rolling-window-Default 8.0 s
- [ ] BPM-Halbierungs-/Verdopplungs-Härtung über `expected_bpm_range` (Default `(60, 200)`) ist implementiert
- [ ] Output-Schemata für `file`- und `stream`-Modus sind dokumentiert und werden eingehalten
- [ ] Vertrauensgrad-Indikator (`high` / `medium` / `low`) ist im Output enthalten und bei `low` gibt es einen Warn-Hinweis
- [ ] Default-Library, Lizenz und Pin-Version sind im Skill-Body genannt
- [ ] Plattform-Verfügbarkeit (ARM64 / x86_64 / Mic-Array) wird vor der Analyse geprüft, fehlende Library bricht mit klarer Meldung ab
- [ ] Pollens Audio-Stack (`reachy_mini.media.*`) wird **nicht** ersetzt — der Skill konsumiert nur, kein eigenes Audio-Capture
- [ ] Keine Roh-Audio-Daten, keine Spektrogramme, keine PCM-Samples im Output
- [ ] Cross-Refs auf [`dance-choreography`](../dance-choreography/de.md), [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/de.md), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) sind sichtbar
- [ ] `pre-commit run --all-files` läuft auf der Skill-Datei grün

## Quellen

> Quell-Verweise auf Pollen-Code-Dateien zeigen auf Datei + Zeilen-Nummer; Verweise auf Pollen-Markdown-Quellen sind Datei-Level zitiert. Library-Verweise zeigen auf die offiziellen Doku-/Repo-Seiten.

- Pollens Audio-Stack (Capture, GStreamer-Pipeline, Mic-Array-DoA): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- Pollens Audio-Beispiele (Verwendung von `mini.media`): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/sound_record.py>
- librosa Beat-Tracking-Doku (Default-Library-Vorschlag): <https://librosa.org/doc/latest/generated/librosa.beat.beat_track.html>
- aubio Beat-Tracking-Doku (Real-Time-Alternative): <https://aubio.org/doc/latest/group__tempo.html>
- madmom DNN-basierte Beat-Detection (Genauigkeits-Alternative, GPL): <https://madmom.readthedocs.io/en/latest/modules/features/beats.html>
- Interne Cross-Refs:
  - [`claude/dance-choreography`](../dance-choreography/de.md) — Haupt-Konsument
  - [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/de.md) — Audio-Pipeline-Vertrag
  - [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) — PII-Klausel-Vorbild
  - [`claude/reachy-mini-sdk`](../reachy-mini-sdk/de.md) — SDK-Idiome, falls Stream-Format-Fragen aufkommen

## Offene Fragen

- Library-Endwahl: ist `librosa` für File-Authoring + Real-Time auf RPi-4-CM4 schnell genug, oder braucht der Stream-Modus eine separate Library (z. B. `aubio`)? Vorschlag: erstmal `librosa` für beide; Wechsel zu `aubio` für Stream-Modus, falls Latenz-Probleme auftauchen.
- BPM-Halbierungs-/Verdopplungs-Härtung: reicht ein einfaches Range-Mapping, oder braucht es eine smartere Logik (z. B. autocorrelation-Peak-Matching)? Erst pragmatisch, später schärfen.
- Live-Beat-Trigger an die App-Loop: gehört eine Skeleton-Klasse `BeatTracker` mit `on_beat`-Callback in den Skill, oder ist das schon Move-Logik und damit App-Aufgabe? Aktuell als App-Aufgabe markiert.
- Mic-Array vs. Lautsprecher-Ausgang: bei Live-Tanz-Apps ist die Frage relevant, ob der Skill den Mic-Eingang oder den Audio-Ausgang (was tatsächlich gespielt wird) analysiert. Mic ist anfällig für Raum-Hall, Audio-Out ist sauber. Konsumenten-Entscheidung.
- Latenz-Lookahead: 100–300 ms Schätzung — sollte gegen reale Library-Messungen verifiziert werden, sobald Hardware da ist.
- Format-Erweiterung: gibt es Use-Cases für `Opus` oder `AAC` jenseits der WAV/MP3/FLAC/OGG-Default-Liste?
