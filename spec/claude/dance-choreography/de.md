# Dance-Choreography-Skill

Status: draft

## Kontext
Die App `reachy-mini-show` (siehe `reachy-mini/app-architecture`) liefert vier BPM-parametrisierte Tanz-Bausteine — `groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around` — sowie Emotion- und State-Bausteine (z. B. `excited`, `proud`, `happy`, `surprised`, `shy`, `waiting-idle`, `alert-listening`). Wer eine Tanz-Sequenz zu einem konkreten Lied oder einer Stimmung bauen will, steht heute vor zwei Lücken: (1) es gibt keinen kanonischen, maschinen- und menschenlesbaren Plan, der die Bausteine in eine Sektions-Struktur (Intro / Verse / Chorus / Bridge / Outro) zu einem Musikstück bringt, und (2) die hardware-bedingten Grenzen aus `reachy-mini/control-surface` (Pitch-Velocity, Servo-Wärme, `max_body_yaw`, Brown-out-Risiko) müssen pro Sektion neu hergeleitet werden, was bei manueller Komposition regelmäßig schiefgeht. Der Skill `dance-choreography` schließt diese Lücken: er nimmt eine Lied-/Stimmungs-Beschreibung an und erzeugt eine **Choreographie-Datei** als Authoring-Artefakt — eine Sektions-Tabelle mit Slug-Verweisen auf den Motion-Catalog, BPM- und Beat-Parametern, Übergangs-Regeln und einer Übersetzungs-Checkliste. Der Entwickler übersetzt diese Datei anschließend manuell in `Move`-Subklassen oder Move-Sequenzen für `reachy-mini-show`. Der Skill schreibt **keinen** Roboter-Code, sendet **keine** Befehle an die App und führt **keine** Beat-Detection durch.

## Ziele
- Aus einer Lied- / Stimmungs-Beschreibung entsteht in einem Skill-Aufruf eine vollständige, validierte Choreographie-Datei
- Jede Sektion verweist ausschließlich auf existierende Slugs aus `spec/reachy-mini/motions/` — keine erfundenen Bewegungen
- Hardware-Limits aus `spec/reachy-mini/control-surface/de.md` werden vor dem Schreiben pro Sektion durchgeprüft (BPM-Range je Baustein, Cool-down nach `headbang-soft`-Bursts, `max_body_yaw` bei `spin-look-around`, Brown-out-Risiko bei voller Aktuator-Last)
- Plattform-Konsequenzen (Wireless / Lite / Simulation) sind je Sektion explizit ausgewiesen — eine `headbang-soft`-Sequenz, die auf Wireless mit IMU-Cool-down sicher ist, trägt auf Lite ein hartes Bangs-Limit
- Output ist sowohl menschenlesbar (für den übersetzenden Entwickler) als auch maschinenlesbar (für spätere Tools, die Choreographien diff-, lint- oder render-bar machen wollen)
- Der Skill bleibt schmal: er liefert den Plan, nicht den Code, und delegiert SDK-Idiome an `reachy-mini-sdk`, Behavior-Scaffolding an `behavior-scaffold`, Audio-Beat-Erkennung an `audio-beat-tracking` (geplant)

## Nicht-Ziele
- Konkrete `Move`-Subklassen, Move-Sequenz-Code oder App-Patches im `reachy-mini-show`-Repo (Aufgabe des Entwicklers; SDK-Idiome via `reachy-mini-sdk`)
- Beat-Detection oder BPM-Schätzung aus einer Audio-Datei (Aufgabe von `audio-beat-tracking`, geplant)
- Direkter WebSocket-Push an die App (`set_dance`, `play_behavior`); Choreographien werden nicht zur Laufzeit eingespielt
- Komposition neuer Tanz-Bausteine — der Skill nutzt den bestehenden Motion-Catalog und schlägt höchstens neue Slugs als „Open Question" vor, statt sie selbst zu erfinden
- Audio-Datei-Mitgabe; Audio-Asset-Pflege liegt im Konsumenten-Repo (siehe `reachy-mini/app-architecture` § Audio-Asset-Management)
- Choreographie-Wiedergabe, -Visualisierung oder -Live-Editing (eigene Tools, geplant)
- Hugging-Face-Publishing der entstehenden Choreographien (kein Distributionspfad in dieser Spec)

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine treffsichere `description` liefern, die Claude Code aktiviert auf Formulierungen wie „erzeuge eine Tanz-Choreographie für Reachy", „plane einen Tanz zu diesem Lied", „compose a dance choreography for the Reachy Mini", „lay out a dance for BPM 110", „Choreographie für Reachy zu Genre X"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: dance, choreography, Reachy Mini, plan, sections
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist: für reine Beat-Detection, für die Implementierung der `Move`-Subklassen, für direkte WebSocket-Befehle an die App, für Hardware-Bring-up

### Eingabe-Parameter
- **MUSS [MUST]** mindestens einen **Choreographie-Namen** als Pflicht-Parameter erwarten, normalisiert zu ASCII-Kebab-Case (z. B. `summer-pop-90s`, `metal-energy-burst`)
- **MUSS [MUST]** **mindestens eines** der folgenden Felder akzeptieren, damit Tempo und Charakter der Choreographie ableitbar sind: `bpm` (Zahl oder Bereich), `genre` (Freitext), `mood` (Freitext), `tempo_class` (`slow` / `medium` / `fast` / `variable`)
- **MUSS [MUST]** eine **Gesamtdauer** annehmen (`duration_s`, ganzzahlig oder Float; oder alternativ `beats_total`, woraus mit BPM die Dauer rekonstruiert wird)
- **SOLLTE [SHOULD]** ein **Plattform-Profil** akzeptieren (`platform: wireless | lite | simulation | any`, Default: `any`); das Profil entscheidet, welche Plattform-Konsequenzen prominent in den Plan geschrieben werden
- **SOLLTE [SHOULD]** eine **Sektions-Struktur** akzeptieren (z. B. `["intro", "verse", "chorus", "verse", "chorus", "bridge", "chorus", "outro"]`); Default: aus `tempo_class` und `duration_s` ableiten (siehe Sektions-Heuristik unten)
- **SOLLTE [SHOULD]** einen optionalen **Stimmungs-Bogen** akzeptieren (`mood_arc`, z. B. `["calm", "rising", "peak", "release"]`), der die Wahl der Bausteine pro Sektion zusätzlich biasen darf
- **SOLLTE [SHOULD]** den Ziel-Pfad parametrisierbar machen (Default: `choreographies/<name>.md` im aktuellen Arbeitsverzeichnis); bei Verwendung im `reachy-mini-show`-Repo bietet sich `reachy_mini_show/choreographies/` an
- **DARF NICHT [MUST NOT]** stillschweigend Default-Werte für `bpm` raten, wenn weder `bpm` noch `tempo_class` noch `genre` angegeben sind — in diesem Fall mit klarer Fehlermeldung abbrechen

### Motion-Catalog als einzige Slug-Quelle
- **MUSS [MUST]** ausschließlich Slugs verwenden, die in `spec/reachy-mini/motions/<slug>/` als Spec-Ordner existieren
- **MUSS [MUST]** vor dem Schreiben jeden geplanten Slug gegen das Verzeichnis verifizieren; ein fehlender Slug bricht den Skill ab und nennt den Konflikt
- **MUSS [MUST]** die Tanz-Bausteine als **Primärkanal** behandeln: `groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around`
- **SOLLTE [SHOULD]** Emotion-Bausteine (`happy`, `excited`, `proud`, `surprised`, `shy`, `confused`, `curious`, `sad`, `angry`, `disappointed`, `disgust`, `sleepy`) als **Akzent-Inserts** zwischen Tanz-Sektionen oder am Sektions-Übergang vorsehen, nicht als rhythmischen Hauptkanal
- **SOLLTE [SHOULD]** State-Bausteine (`waiting-idle`, `alert-listening`, `thinking`) für die Outro-Idle-Phase nach dem Lied bzw. für eine ruhige Bridge nutzen
- **DARF NICHT [MUST NOT]** Defensive-Bausteine (`flinch`, `alarm`, `scanning`) in einer Choreographie führen — sie haben semantisch nichts mit Tanz zu tun
- **KANN [MAY]** in einer „Open Questions"-Sektion am Ende der Choreographie einen Hinweis auf einen fehlenden Baustein als zukünftige Motion-Spec aufwerfen — niemals selbst einen Slug erfinden und einsetzen

### Sektions-Struktur und -Heuristik
- **MUSS [MUST]** die Choreographie in benannte Sektionen gliedern; gültige Sektions-Typen: `intro`, `verse`, `pre-chorus`, `chorus`, `bridge`, `instrumental`, `breakdown`, `outro`, `outro-idle`
- **MUSS [MUST]** je Sektion mindestens diese Felder festhalten: `section`, `slug` (Tanz-Baustein), `bpm`, `beats`, `lead_time_s`, `expected_duration_s`, optional `accent_slug` (Emotion-Insert), optional `notes`
- **SOLLTE [SHOULD]** Sektions-Längen aus den BPM-Range-Empfehlungen der Bausteine ableiten:
  - `intro` 1–2 Bausteine, niedrig-energetisch (`waiting-idle` → `sway-side` low BPM)
  - `verse` und `pre-chorus` mit `groove-bob` oder `sway-side`
  - `chorus` mit `groove-bob` höhere BPM oder `headbang-soft` bei energiereicher Musik
  - `bridge` ggf. mit `spin-look-around` oder einem Emotion-Akzent
  - `outro` weicher Auslauf zurück zu `groove-bob` oder `sway-side` low BPM
  - `outro-idle` `waiting-idle`, mit `loop_count: null` für unbegrenztes Loopen bis Stop-Signal
- **MUSS [MUST]** die Summe der `expected_duration_s` plus aller Bausteine-Eintritte/Austritte (gemäß den Motion-Specs) in einem Toleranzfenster von ±10 % der eingegebenen `duration_s` halten; bei größerer Abweichung Sektionen umverteilen oder Beats anpassen, statt das Lied stillschweigend zu kürzen oder zu verlängern
- **SOLLTE [SHOULD]** den `mood_arc` so auf Sektionen mappen, dass der Energie-Höhepunkt mit einer Chorus-Sektion zusammenfällt (z. B. `mood_arc=["calm","rising","peak","release"]` mit 4 Sektionen → letzte Sektion bekommt das energiereichste Tanz-Pattern)
- **DARF NICHT [MUST NOT]** zwei aufeinanderfolgende Sektionen mit `headbang-soft` ohne dazwischenliegende Cool-down-Sektion (`groove-bob`, `sway-side` oder `waiting-idle`) ansetzen, wenn das Plattform-Profil `wireless` oder `lite` ist

### Hardware- und Plattform-Validierung
- **MUSS [MUST]** je Sektion die BPM-Range des verwendeten Bausteins gegen die Werte in der jeweiligen Motion-Spec prüfen (z. B. `groove-bob` 60–180, `sway-side` 50–140, `headbang-soft` 60–130 auf Hardware, 60–180 in Simulation)
- **MUSS [MUST]** bei `headbang-soft` die Anzahl aufeinanderfolgender Bangs pro Sektion und Plattform begrenzen:
  - `wireless`: maximal 16 Bangs pro Sektion, Cool-down-Sektion (≥ 2 s) verpflichtend nach 8 Bangs am Stück; auf IMU-Temperatur-Polling im Code hinweisen
  - `lite`: hartes Limit von 8 Bangs pro Sektion ohne Cool-down dazwischen
  - `simulation`: kein Hardware-Limit, nur eine Notiz, dass die On-Hardware-Validierung den Agenten `reachy-mini-on-device` braucht
- **MUSS [MUST]** bei `spin-look-around` darauf hinweisen, dass `automatic_body_yaw=False` während der Sequenz gesetzt sein muss (gemäß Motion-Spec) und dass `max_body_yaw` ≤ ±150° bleibt
- **MUSS [MUST]** Brown-out-Risiko ausweisen, wenn eine Sektion mehrere voll-aktive Aktuatoren simultan auf hoher BPM nutzt (z. B. `sway-side` ≥ 120 BPM mit Antennen-Asymmetrie und `automatic_body_yaw=True`); auf Wireless explizit als Risiko notieren, auf Lite als reduziertes Risiko (externe Spannungsversorgung)
- **MUSS [MUST]** bei aktiviertem Plattform-Profil `simulation` notieren, welche Aspekte (Audio-Wiedergabe, IMU-Telemetrie, echte Pose-Erreichung, Servo-Wärme) in der Simulation **nicht** geprüft werden können und daher On-Hardware-Validierung erfordern
- **MUSS [MUST]** alle hardware-spezifischen Werte (Cool-down-Schwellen, Servo-Temperatur-Schwellen, exakte BPM-Obergrenzen der Hardware), die in der `control-surface`-Spec als `> ⚠ TBD: validate against real hardware` markiert sind, ebenfalls als TBD übernehmen statt sie zu raten

### Output-Format
- **MUSS [MUST]** den Plan als **Markdown-Datei mit YAML-Frontmatter** schreiben — Frontmatter ist die maschinenlesbare Schicht, der Markdown-Body ist die menschenlesbare Schicht für den Entwickler
- **MUSS [MUST]** die Frontmatter mindestens diese Felder tragen:

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
  protocol_version: "1.0"   # spiegelt das WebSocket-Protokoll der App
  sections:
    - section: intro
      slug: waiting-idle
      bpm: null
      beats: null
      duration_s: 4.0
      lead_time_s: 0.0
      accent_slug: null
      notes: "weicher Einstieg"
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
    - "headbang-soft chorus überschreitet 8-Bang-Limit auf Lite — Cool-down einlegen"
  ---
  ```

- **MUSS [MUST]** im Markdown-Body folgende Abschnitte tragen, in dieser Reihenfolge: `# <Titel>`, `## Kontext` (1–3 Sätze), `## Sektions-Tabelle` (eine Tabelle mit allen Sektionen), `## Plattform-Konsequenzen`, `## Übersetzungs-Checkliste für den Entwickler`, `## Offene Fragen`
- **MUSS [MUST]** die `Übersetzungs-Checkliste` mindestens diese Punkte enthalten:
  1. Pro Sektions-Slug die `Move`-Subklasse aus `reachy_mini_show/behaviors/` instanziieren oder die Slug-Registry erweitern
  2. BPM, Beats und Lead-Time aus der Frontmatter konstruktor-parametrisiert übergeben
  3. Idle-/Outro-Sektion mit `loop_count=None` als Hintergrund-Behavior einplanen
  4. Audio-Trigger gegen das Lied taktgenau ausrichten (Hinweis auf `audio-beat-tracking`, geplant)
  5. Test gegen `ReachyMini(use_sim=True)` schreiben, dann On-Hardware-Validierung via `reachy-mini-on-device`-Agent
  6. Plattform-spezifische Fallbacks (z. B. `headbang-soft` → `groove-bob` bei Servo-Wärme) implementieren
- **MUSS [MUST]** die `Offene Fragen`-Sektion explizit Open-Question-Marker enthalten, wenn der Skill eine Lücke im Motion-Catalog erkannt hat (z. B. „Für eine glaubhafte Reggae-Bridge fehlt ein `head-tilt-side`-Baustein — als Motion-Spec vorschlagen?")
- **SOLLTE [SHOULD]** im Markdown-Body Quer-Verweise auf die zugrunde liegenden Motion-Specs setzen (`spec/reachy-mini/motions/<slug>/de.md`) und auf die `app-architecture`-Spec, wenn protokoll-relevante Punkte berührt sind

### Validierung vor dem Schreiben
- **MUSS [MUST]** prüfen, ob unter `<target_dir>/<name>.md` bereits eine Datei existiert; bei Kollision abbrechen und den Konflikt-Pfad benennen, statt zu überschreiben
- **MUSS [MUST]** den Choreographie-Namen gegen ASCII-Kebab-Case validieren; Verstöße brechen mit klarer Fehlermeldung ab (kein stilles Umschreiben)
- **MUSS [MUST]** jeden geplanten Slug gegen `spec/reachy-mini/motions/<slug>/` verifizieren; fehlende Slugs brechen ab und nennen den fehlenden Slug
- **MUSS [MUST]** die Plausibilität der Sektions-Summe gegen `duration_s` prüfen (siehe Toleranzfenster oben); Verstoß bricht ab und schlägt eine Umverteilung vor
- **SOLLTE [SHOULD]** die Choreographie zusätzlich gegen die Spec `reachy-mini/app-architecture` prüfen — `protocol_version`-Feld und WebSocket-`set_dance`-Konvention müssen kompatibel bleiben

### Konsistenz mit Repo-Standards
- **MUSS [MUST]** die generierte Datei so erzeugen, dass `pre-commit run --all-files` ohne Auto-Fix-Änderung grün durchläuft (LF-Newlines, kein Trailing-Whitespace, valide YAML-Frontmatter)
- **MUSS [MUST]** alle hardware-abhängigen Annahmen, die nicht aus einer verifizierten Quelle stammen, mit `> ⚠ TBD: validate against real hardware` markieren
- **SOLLTE [SHOULD]** dem Entwickler nach dem Schreiben eine kurze Zusammenfassung zurückgeben: Pfad zur erzeugten Datei, Anzahl Sektionen, gefundene Warnings, Verweis auf die Übersetzungs-Checkliste

### Out-of-Scope-Klarstellung
- **DARF NICHT [MUST NOT]** `Move`-Subklassen-Code, App-Patches oder WebSocket-Befehle erzeugen — der Skill produziert ausschließlich die Choreographie-Datei
- **DARF NICHT [MUST NOT]** eine Audio-Datei verarbeiten oder BPM aus einer Audio-Datei schätzen; das ist Aufgabe von `audio-beat-tracking` (geplant)
- **DARF NICHT [MUST NOT]** den `reachy_mini`-Daemon, eine `ReachyMini`-Instanz oder das Hugging-Face-Spaces-Repo der App ansprechen
- **SOLLTE [SHOULD]** auf Nachbar-Skills verweisen (`reachy-mini-sdk`, `behavior-scaffold`, `audio-beat-tracking`, Agent `reachy-mini-on-device`) statt deren Inhalte zu duplizieren

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/dance-choreography/SKILL.md` mit gültiger Frontmatter (`name: dance-choreography`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Ein Test-Aufruf mit Name, BPM und Dauer erzeugt eine Choreographie-Datei mit YAML-Frontmatter und Markdown-Body
- [ ] Alle Sektions-Slugs in der erzeugten Datei existieren als Ordner unter `spec/reachy-mini/motions/`
- [ ] Tanz-Bausteine (`groove-bob`, `sway-side`, `headbang-soft`, `spin-look-around`) sind die Primär-Slugs in den Tanz-Sektionen; Emotion-Bausteine erscheinen höchstens als `accent_slug`
- [ ] BPM-Werte je Sektion liegen innerhalb der in der Motion-Spec definierten Range
- [ ] Bei Plattform `wireless` oder `lite` setzt eine Choreographie mit `headbang-soft` eine Cool-down-Sektion zwischen ≥ 8-Bang-Bursts
- [ ] Bei Plattform `simulation` ist im Markdown-Body explizit benannt, welche Aspekte nicht geprüft sind (Audio, IMU, Servo-Wärme)
- [ ] Sektions-Summe der Dauern liegt innerhalb ±10 % der eingegebenen `duration_s`
- [ ] Die Frontmatter trägt `protocol_version: "1.0"` analog zur App-Architektur-Spec
- [ ] Bei Namens-Kollision bricht der Skill ab und benennt den existierenden Pfad
- [ ] Bei einem nicht existierenden Slug bricht der Skill ab und nennt den fehlenden Slug
- [ ] Defensive-Bausteine (`flinch`, `alarm`, `scanning`) tauchen in keiner Choreographie auf
- [ ] Übersetzungs-Checkliste ist in jeder Choreographie-Datei vorhanden und nennt mindestens die sechs Pflicht-Schritte
- [ ] `pre-commit run --all-files` läuft auf der erzeugten Datei grün, ohne Auto-Fix-Modifikationen
- [ ] Verweise auf `reachy-mini-sdk`, `behavior-scaffold`, `audio-beat-tracking` und Agent `reachy-mini-on-device` sind im Skill-Body sichtbar
- [ ] Hardware-spezifische Werte, die in `control-surface` TBD sind, sind auch in der Choreographie als `> ⚠ TBD: validate against real hardware` markiert

## Offene Fragen
- Soll der Skill optional eine zweite Output-Variante (rein YAML, ohne Markdown-Body) anbieten, sobald ein Choreographie-Renderer / -Linter existiert?
- Wie weit darf der Skill `mood`-Texte (Freiform-Englisch oder -Deutsch) interpretieren, bevor er eine LLM-Heuristik wird, die schwer zu prüfen ist? Vorschlag: striktes Mapping von vordefinierten `mood`-Tokens (`calm`, `happy`, `melancholic`, `aggressive`, `playful`, `solemn`) auf Baustein-Präferenzen, plus Freitext-Notiz für Sonderfälle.
- Sollen mehrere Choreographien für dasselbe Lied (z. B. `simmer-pop-90s.wireless.md` und `summer-pop-90s.lite.md`) als getrennte Dateien geführt werden, oder als eine Datei mit Plattform-Varianten in der Frontmatter?
- Wie wird die Choreographie an die App ausgespielt — bleibt der Entwickler manueller Übersetzer, oder soll eine spätere Iteration einen automatischen Übersetzer in die Slug-Registry der App liefern? Vorschlag: erst manueller Pfad, automatischer Übersetzer als eigenständiger Skill `dance-choreography-compile`.
- Sollte die Cool-down-Schwelle bei `headbang-soft` aus einer eigenen `motion-thermal-budget`-Spec kommen, sobald die echten Servo-Temperatur-Daten am Gerät erhoben sind?
- Welche `mood_arc`-Token-Sätze sind sinnvoll? Vorschlag: max. acht Tokens (`calm`, `rising`, `peak`, `release`, `melancholic`, `playful`, `aggressive`, `solemn`), erweiterbar via Open-Question-Marker.
- Soll der Skill eine Choreographie auch für nicht-musikalische Anwendungs-Szenarien (z. B. „Begrüßungs-Tanz beim Heimkommen") erlauben? Tendenz: ja, solange `bpm`/`tempo_class` mitgegeben wird; sonst Abbruch wie oben spezifiziert.
- Wie integriert sich der Skill mit dem Pollen-CLI-Layout für Apps, falls Pollen einen Choreographie-Loader im SDK bekommt? Aktuell nicht relevant, langfristig zu beobachten.
