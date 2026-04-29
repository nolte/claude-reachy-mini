# On-Device-Test-Agent für Reachy Mini

Status: draft

## Kontext
Sobald die Hardware da ist, müssen Behaviors auf dem echten Reachy Mini überprüft werden, bevor sie in eine App oder eine Hugging-Face-Veröffentlichung wandern. Manuell heißt das: SSH oder USB anbinden, Code synchronisieren, Abhängigkeiten installieren, Behavior starten, Logs und Telemetrie sammeln, bei Fehlverhalten stoppen, ein Protokoll schreiben. Diese Schritte sind sequenziell, latenz-lastig, fehleranfällig und produzieren viel Rohausgabe — genau die Art von Aufgabe, die den Hauptthread eines Claude-Code-Gesprächs zumüllt, wenn sie inline läuft. Der Agent `reachy-mini-on-device` kapselt diesen Lifecycle in einer eigenen Tool-Session und liefert dem Hauptthread nur eine knappe, strukturierte Zusammenfassung zurück. Er testet, er entwickelt nicht — Code-Anpassungen bleiben Aufgabe des Hauptthreads, der die Skills `reachy-mini-sdk`, `behavior-scaffold` und `home-assistant-bridge` nutzt.

## Ziele
- Ein Behavior wird mit einem einzigen Aufruf auf das echte Gerät gebracht und live ausgeführt
- Während des Laufs werden Telemetrie und Logs strukturiert eingesammelt, ohne den Hauptkontext zu fluten
- Fehlverhalten führt zu kontrolliertem Notstopp und sauberer Trennung vom Gerät, nicht zu hängenden Verbindungen
- Das Ergebnis kommt als knappe PASS/FAIL-Zusammenfassung mit Verweis auf einen Volltext-Log-Artefakt
- Der Agent bleibt narrow: Test- und Beobachtungs-Aufgabe, keine Bewegungs-Logik, keine Code-Änderung am Behavior

## Nicht-Ziele
- Hardware-Bringup (eigener Skill geplant)
- Firmware-Flash (eigener Skill geplant)
- Entwicklung des Behaviors oder der App (eigene Repos, eigene Skills)
- Veröffentlichung des Behaviors auf Hugging Face (`behavior-publish-hf`, geplant)
- Audio-/Beat-Tracking (`audio-beat-tracking`, geplant)
- Dauerhafter Betrieb / Watchdog im Produktiv-Setup — der Agent ist ein Test-Lifecycle, kein Daemon

## Skill-vs-Agent-Begründung
Diese Aufgabe wird als **Agent** und nicht als Skill modelliert, weil mehrere Begründungen aus `nolte-shared/spec/claude/skill-vs-agent/` simultan zutreffen:

- **Lange, latenz-lastige Tool-Session** — SSH/USB-Connect, scp/rsync-Deploy, abwartendes Behavior-Step-Watching mit Sekunden- bis Minuten-Latenzen pro Phase. Skills sind für interaktive Inline-Workflows optimiert; ein langer, sequenzieller Lifecycle gehört in eine eigene Tool-Session.
- **Kontext-Volumen** — Rohe Behavior-Logs und Sensor-Telemetrie können tausende Zeilen pro Lauf erzeugen. Im Hauptthread würden sie den Kontext verschlingen, ohne der nachgelagerten Entscheidung zu nützen. Der Agent reduziert das auf eine Zusammenfassung und einen Datei-Artefakt-Pfad.
- **Mehrstufige Orchestrierung mit Fehler-Recovery** — Connect → Deploy → Install Deps → Start → Watch → Stop → Disconnect. Jede Stufe hat eigene Failure-Modes (Auth-Fehler, Disk-Full, Behavior-Crash, USB-Disconnect), die eigene Recovery brauchen, ohne den Hauptthread zu unterbrechen.
- **Eigenständiges Tool-Set** — Der Agent braucht Bash für `ssh`/`scp`/`rsync` plus ggf. ein gerätespezifisches CLI. Diese Tools gehören nicht in den Hauptthread, der Code-Editing dominiert.
- **Spezialisiertes Verhalten** — Sicherheits-Notstopp, gracefuler Cleanup bei Disconnect und striktes Output-Format sind Anforderungen, die ein dedizierter Agent kanonisiert.
- **Distribution: `plugin`** — der Agent gehört zum Plugin und wird mit ihm verteilt.

## Anforderungen

### Eingaben
- **MUSS [MUST]** den Ziel-Behavior-Pfad annehmen (lokales Verzeichnis im konsumierenden App-Repo)
- **MUSS [MUST]** die Geräte-Adresse annehmen — entweder einen SSH-Host (`user@host` plus optionalem Identitäts-Pfad) oder ein USB-Device-Identifier; der konkrete Adress-Typ ist `> ⚠ TBD: validate against real hardware`
- **MUSS [MUST]** ein hartes Timeout pro Lauf annehmen (Default `> ⚠ TBD: validate against real hardware`); nach Timeout wird der Notstopp-Pfad ausgelöst
- **SOLLTE [SHOULD]** einen Trigger-Modus annehmen: `autonomous` (Behavior läuft ohne externe Trigger) vs. `interactive` (HA-Event löst Step aus, optional über `home-assistant-bridge`-Patterns)
- **KANN [MAY]** zusätzliche Optionen annehmen: Trockenlauf (Deploy ohne Run), nur-Watch (kein Deploy), Verbosity der Zusammenfassung

### Lifecycle
- **MUSS [MUST]** den Lifecycle in dieser Reihenfolge ausführen: connect → sync code → install deps → start behavior → watch & sample → stop → disconnect
- **MUSS [MUST]** in jeder Phase das beobachtete Ergebnis strukturiert protokollieren (Phase, Status, Dauer, Fehler-Klasse falls vorhanden)
- **MUSS [MUST]** bei Disconnect oder unerwartetem Behavior-Exit kontrolliert enden — keine hängenden SSH-Sessions, keine offen gelassenen Behavior-Prozesse
- **SOLLTE [SHOULD]** zwischen den Phasen einen Health-Check einschieben (CPU-/Spannungs-/Temperatur-Werte, falls das SDK sie bereitstellt — `> ⚠ TBD: validate against real hardware`)

### Notstopp
- **MUSS [MUST]** einen Notstopp-Pfad bereitstellen, der den Behavior-Prozess sicher beendet und das Gerät in eine definierte Ruhepose bringt — Pose-Definition `> ⚠ TBD: validate against real hardware`
- **MUSS [MUST]** den Notstopp ohne Nutzer-Bestätigung auslösen, wenn ein konfigurierter Sicherheits-Threshold reißt (z. B. ungewöhnlich hohe Stromaufnahme, Bewegungs-Limits überschritten)
- **MUSS [MUST]** den Notstopp im Output-Protokoll als gesondertes Ereignis ausweisen
- **DARF NICHT [MUST NOT]** den Notstopp auf eine reine Log-Notiz reduzieren — die physische Konsequenz hat Vorrang

### Ausgabe / Output-Format
- **MUSS [MUST]** in den Hauptthread nur eine strukturierte Zusammenfassung zurückgeben: Gesamt-Status (`PASS` / `FAIL` / `ABORTED`), Hooks-Statistik (welche Hooks aufgerufen, wie oft, mit welcher mittleren Latenz), Anomalien-Liste, Dauer
- **MUSS [MUST]** den Volltext-Log als Datei-Artefakt unter `.audits/on-device/<ISO-timestamp>-<behavior-name>.log` ablegen und den Pfad in der Zusammenfassung nennen
- **DARF NICHT [MUST NOT]** rohe Logs oder Sensor-Streams in den Hauptthread zurückgeben
- **SOLLTE [SHOULD]** ein Maschinen-lesbares Beiseite-Artefakt mitliefern (z. B. JSON mit denselben Daten), wenn Folge-Schritte automatisiert werden

### Sicherheit und Geheimnisse
- **MUSS [MUST]** SSH-/Geräte-Credentials aus der Umgebung oder aus einer ssh-config lesen, niemals als Argument im Klartext
- **DARF NICHT [MUST NOT]** Credentials in den Output-Log oder in die Zusammenfassung schreiben — Maskierung Pflicht, falls eine Identifier-Notation nötig ist
- **SOLLTE [SHOULD]** SSH-Host-Key-Verifikation aktivieren; bei einer Erst-Verbindung den Fingerprint im Output ausweisen, statt automatisch zu akzeptieren

### Boundaries
- **SOLLTE [SHOULD]** auf `reachy-mini-sdk` verweisen, sobald der Hauptthread Bewegungs-Idiomatik nach dem Test anpassen soll
- **SOLLTE [SHOULD]** auf `behavior-scaffold` verweisen, wenn der Test zeigt, dass das Behavior strukturell unvollständig ist
- **SOLLTE [SHOULD]** auf `home-assistant-bridge` verweisen, wenn der `interactive`-Modus mit echten HA-Events laufen soll
- **DARF NICHT [MUST NOT]** Inhalte aus diesen Skills duplizieren — der Agent ist Beobachter und Orchestrator, nicht Wissensbasis

## Akzeptanzkriterien
- [ ] Der Agent ist unter `agents/reachy-mini-on-device.md` mit gültiger Frontmatter angelegt — `name: reachy-mini-on-device`, `description`, `distribution: plugin`, optional Tags
- [ ] Die `description` aktiviert auf Phrasings wie „test the behavior on the device", „deploy and run X on Reachy Mini", „live-trial behavior <name>"
- [ ] Eine Skill-vs-Agent-Begründung ist im Agent-Body sichtbar (mindestens Tool-Session-Länge, Kontext-Volumen, Orchestrierung)
- [ ] Lifecycle-Phasen (connect / deploy / install / start / watch / stop / disconnect) sind im Body dokumentiert
- [ ] Notstopp-Verhalten und Default-Sicherheits-Thresholds sind dokumentiert (mit TBD-Markern, wo Hardware-verifiziert werden muss)
- [ ] Eingabe-Parameter (Behavior-Pfad, Geräte-Adresse, Timeout, Trigger-Modus) sind dokumentiert
- [ ] Ausgabe-Format ist als striktes Schema dokumentiert; rohe Logs landen in `.audits/on-device/<timestamp>-<name>.log`
- [ ] `.audits/` ist in `.gitignore` enthalten, sodass Logs niemals committet werden
- [ ] Verweise auf `reachy-mini-sdk`, `behavior-scaffold`, `home-assistant-bridge` sind im Body sichtbar
- [ ] Der Agent wird vom Skill-Agent-Katalog-Generator akzeptiert (Frontmatter validiert, `name` matcht Dateinamen, `distribution` ist gesetzt)
- [ ] Aussagen ohne Hardware-Verifikation tragen einen `⚠ TBD: validate against real hardware`-Hinweis

## Offene Fragen
- Welches Deploy-Protokoll ist kanonisch — `rsync` über SSH, `scp`, ein gerätespezifisches Tool, oder unterstützt das SDK Remote-Run direkt?
- Welches Telemetrie-Format liefert das SDK (Events, Sample-Streams, Log-Lines)? Davon hängt das Output-Schema ab.
- Welche minimale Ruhepose ist für den Notstopp sicher? Vorschlag: alle Joints in Mittellage, Antennen neutral. Endgültig vor erster Hardware-Inbetriebnahme bestätigen.
- Welche Sicherheits-Thresholds (Strom, Temperatur, Bewegungs-Limits) sind ab Werk verfügbar, welche müssen wir im Agent selbst messen?
- Soll der Agent Multi-Run-Vergleiche unterstützen (zwei Läufe vergleichen, um Regressionen zu erkennen), oder strikt einen Lauf pro Aufruf?
- Wie integriert sich der Agent mit CI? Vorschlag: Auf Hardware-Lauf nur lokal/auf einem Hardware-Runner, in der CI nur Trockenlauf-Modus.
- Wie verhält sich der Agent, wenn das SDK selbst eine Test-/Mock-Schicht bietet — fällt er darauf zurück, statt echte Hardware zu erwarten? Tendenz: nein, Mock ist Sache eines separaten Skills.
- Welche Maximal-Größe darf das Log-Artefakt haben, bevor rotation oder Trimm-Strategien greifen?
