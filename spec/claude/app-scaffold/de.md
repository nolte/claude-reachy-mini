# App-Scaffold-Skill

Status: draft

## Kontext
Eine **Reachy-Mini-App** im Pollen-Sinn (Hugging-Face-Space-publishable Python-Paket mit `reachy_mini_python_app`-Tag, Entry-Point in der Group `reachy_mini_apps`, `ReachyMiniApp`-Subklasse mit `run(self, reachy_mini, stop_event)`) ist das primäre Lieferobjekt der App-Entwicklung mit diesem Plugin. Pollen liefert ein offizielles CLI-Tool — `reachy-mini-app-assistant create` —, das das Skelett 1:1 nach Pollen-Konvention erzeugt (`pyproject.toml` mit Entry-Point, `README.md` mit HF-Frontmatter-Tag, `index.html`/`style.css` für die HF-Space-Landing-Page, `<pkg>/main.py` mit `ReachyMiniApp`-Klasse, optional `<pkg>/static/` für Web-UI). Pollens eigene Doku ist explizit: „**Never create app folders manually.** Manual creation leads to subtle issues that are hard to debug." Dieser Skill `app-scaffold` ist daher ein **dünner Wrapper** um `reachy-mini-app-assistant create`, ergänzt das Ergebnis um Provenienz-Marker (Verweis auf dieses Plugin) und legt einen `plan.md`-Stub mit User-Wartepunkt an, damit Claude Code-Sessions vor dem ersten Code-Commit die Anforderungen mit dem User abstimmen. Der Skill ergänzt den Skill `reachy-mini-sdk` (Wissensbasis) um den schreibenden Pfad und delegiert alles, was über das Skelett hinausgeht, an spezialisierte Skills.

Begriffsklärung: „App" und „Behavior" werden in diesem Plugin teilweise synonym verwendet. Streng genommen scaffolded dieser Skill eine **Pollen-App** (das Hugging-Face-publishable Paket), in der dann ein oder mehrere **Behaviors** (Move-Subklassen, Bewegungslogik) leben können. Nachbar-Spec [`reachy-mini/app-architecture`](../../reachy-mini/app-architecture/de.md) beschreibt das Verhältnis im Detail.

## Ziele
- Eine neue Reachy-Mini-App ist mit einem einzigen Skill-Aufruf strukturell vollständig vorhanden — über Pollens offizielles `reachy-mini-app-assistant create`, ergänzt um Provenienz-Marker und `plan.md`-Stub
- Das Skelett folgt der offiziellen Pollen-Robotics-App-Konvention und ist Hugging-Face-kompatibel, mit `--publish` als Default
- Kollisionen mit bestehenden App-Namen werden vor dem Schreiben erkannt
- Vor dem ersten Code-Commit erzwingt der Skill einen User-Wartepunkt über `plan.md` (AGENTS.md-Konvention von Pollen)
- Der Skill bleibt schmal: er ruft das offizielle CLI auf, ergänzt Provenienz, und delegiert benachbarte Anliegen an die zuständigen Skills

## Nicht-Ziele
- Konkrete Bewegungs- oder Tanz-Logik (Aufgabe des Entwicklers; SDK-Wissen liefert `reachy-mini-sdk`)
- Re-Implementierung des Pollen-CLI-Layouts — der Skill **muss** `reachy-mini-app-assistant create` aufrufen, niemals selbst Manifest, `pyproject.toml`, `main.py` oder `README.md` schreiben
- Reine JS-/Web-Apps (Pollen-Doku: „JS-only apps are not yet supported for discovery/sharing"); diese Skill scaffolded nur Python-Apps mit optionalem `static/`-Web-UI
- Veröffentlichung der App auf Hugging Face über das CLI hinaus (`reachy-mini-app-assistant publish` als separater Schritt; eigener Skill `reachy-app-publish-hf` geplant für Custom-Workflows)
- Audio-Analyse, Beat- und Tempo-Erkennung (eigener Skill `audio-beat-tracking` geplant)
- Home-Assistant-Anbindung der App (eigener Skill `home-assistant-bridge`)
- Live-Deployment / On-Device-Test (eigener Agent `reachy-mini-on-device`)
- App-Refactoring oder -Migration auf eine neue SDK-Major-Version

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine treffsichere `description` liefern, die Claude Code aktiviert auf Formulierungen wie „neue Reachy-Mini-App anlegen", „App scaffolden", „create reachy mini app", „start a new dance app", „neues Behavior-App-Skelett für Reachy"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: app, scaffold, Reachy Mini, new (zusätzlich `behavior` als Synonym)
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist (z. B. wenn eine bestehende App nur editiert oder veröffentlicht werden soll)

### Eingabe-Parameter
- **MUSS [MUST]** den App-Namen als Pflicht-Parameter erwarten, ASCII-Kebab-Case (`reachy-mini-show`); das CLI normalisiert intern auf snake_case (`reachy_mini_show`) für den Python-Package-Namen
- **MUSS [MUST]** den Ziel-Pfad annehmen (das übergeordnete Verzeichnis, in dem das CLI das App-Verzeichnis anlegt)
- **MUSS [MUST]** eine kurze Beschreibung (1–3 Sätze) für `pyproject.toml`/`README.md` annehmen
- **SOLLTE [SHOULD]** ein **`template`-Parameter** (`default` | `conversation`) annehmen — bei LLM-/Speech-Anforderungen zwingend `conversation` (forked vom offiziellen Conversation-App-Template, inkl. Audio-Pipeline und FastAPI-Settings-UI), sonst `default`. Quelle: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/create-app.md> § Choose a Template
- **SOLLTE [SHOULD]** ein **`publish`-Flag** (`true` | `false`) annehmen — Default `true` (Pollen-Konvention: „**Always use `--publish` unless the user explicitly requests a local-only app**"); bei `true` wird der Hugging-Face-Space mit Git-Remote angelegt
- **SOLLTE [SHOULD]** optional Autor (Default: aus `git config user.name`/`user.email`) und Tags annehmen, falls das Pollen-CLI sie per Flag akzeptiert; sonst Post-Process im erzeugten `pyproject.toml`

### Pre-Flight-Pflichten (vor jeder schreibenden Aktion)
- **MUSS [MUST]** vor dem CLI-Aufruf prüfen, dass `reachy-mini-app-assistant` im PATH liegt; fehlt es, mit klarer Anleitung abbrechen (`uv tool install reachy-mini` oder `uv pip install reachy-mini` im aktiven venv)
- **MUSS [MUST]** bei `publish=true` vorab `hf auth whoami` prüfen; ohne gültigen Login mit Anleitung abbrechen (`uv pip install --upgrade huggingface_hub && hf auth login`, Token mit **Write**-Permission). Niemals stillschweigend `publish=false` fallback.
- **MUSS [MUST]** das Ziel-Verzeichnis `<target_dir>/<name>/` auf Existenz prüfen; bei Kollision abbrechen und den Konflikt-Pfad nennen, statt zu überschreiben

### Erzeugte Artefakte (über das offizielle CLI)
- **MUSS [MUST]** das Skelett über `reachy-mini-app-assistant create <name> <target_dir> [--publish] [--template default|conversation]` anlegen; **DARF NICHT [MUST NOT]** Manifest, `pyproject.toml`, `main.py`, `README.md`, `index.html`, `style.css` oder den Entry-Point-Eintrag selbst schreiben — Pollens Doku ist explizit: „**Never create app folders manually**. Manual creation leads to subtle issues that are hard to debug." Wenn das CLI fehlt oder fehlschlägt, abbrechen, statt das Skelett von Hand zu rekonstruieren.
- **MUSS [MUST]** nach dem CLI-Lauf das Ergebnis verifizieren über `reachy-mini-app-assistant check <pfad>`; ein Failing Check bricht den Skill ab
- **MUSS [MUST]** anschließend Provenienz-Marker einarbeiten (post-process):
  - in `pyproject.toml` einen `[project.urls]`-Block mit `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
  - eine `CLAUDE.md` im App-Repo-Root anlegen, die die empfohlenen Plugin-Skills (`reachy-mini-sdk`, `app-scaffold`, Agent `reachy-mini-on-device`) namentlich auflistet und auf das Plugin-Repo verlinkt
  - im `README.md` direkt nach dem HF-Frontmatter einen Provenienz-Block einfügen
- **MUSS [MUST]** im App-Verzeichnis-Root die Datei `plan.md` anlegen, deren Inhalt **wörtlich** dem verbindlichen Plan-Template-Stub aus [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/de.md) § Phase 3 folgt (Abschnitts-Namen und -Reihenfolge sind dort fix). Dieser Stub ist die kanonische Spec-Quelle für das `plan.md`-Schema und ist eine Obermenge der vier in Pollens AGENTS.md (<https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>) genannten Pflicht-Abschnitte (Verständnis ⊂ Scope, Technischer Approach ⊂ Motion-Inventar + IPC + Test-Strategie, Klärfragen ⊂ Open Questions, User-Approval ⊂ Abnahme); Pollens AGENTS.md bleibt damit erfüllt
- **MUSS [MUST]** einen Test-Stub anlegen (`tests/test_smoke.py`), der gegen `ReachyMini(spawn_daemon=True, use_sim=True)` läuft und mindestens Import + Klassen-Instanziierung + ein `set_target`-Tick prüft; in der Skip-Bedingung GStreamer-Verfügbarkeit guarden
- **DARF NICHT [MUST NOT]** ein reines JS-/Web-App-Skelett scaffolden — Pollen-Doku: „JS-only apps are not yet supported for discovery/sharing." Discovery via Hugging Face setzt eine Python-App voraus; Web-UI gehört (optional) als `static/`-Subordner ins Python-Paket.

### User-Wartepunkt vor erstem Code-Commit (AGENTS.md-Konvention)
- **MUSS [MUST]** nach Anlage von `plan.md` und vor jedem ersten Code-Commit explizit auf User-Approval warten; die Next-Steps-Checkliste muss diesen Wartepunkt als ersten Schritt benennen
- **DARF NICHT [MUST NOT]** den ersten Code-Commit (Behavior-Logik, Move-Subklassen, etc.) ohne User-Bestätigung des `plan.md` durchführen — der Plan ist Pollens Mechanismus, um Approach-Drift früh zu fangen

### Plattform-Profile
- **MUSS [MUST]** den Test-Stub gegen `ReachyMini(spawn_daemon=True, use_sim=True)` als Default-Pfad laufen lassen — Simulation ist das einzige Profil, das auch ohne Hardware funktioniert und gehört in jeden CI-Lauf
- **SOLLTE [SHOULD]** im Test-Stub deutlich kommentieren, welche Aspekte in Simulation _nicht_ geprüft werden können (Audio-Wiedergabe, IMU-Telemetrie, LED-Sync, echte Pose-Erreichung) — Verweis auf `reachy-mini-on-device`-Agent für die On-Hardware-Validierung gegen Wireless oder Lite
- **MUSS [MUST]** im README-Stub der App eine Plattform-Tabelle vorgeben, die Wireless / Lite / Simulation und je Plattform die Anwendbarkeit nennt
- **DARF NICHT [MUST NOT]** im Test-Stub Plattform-spezifische Annahmen hart-codieren (z. B. ein IMU-Lesezugriff, der auf Lite scheitert) — solche Tests sind dem `reachy-mini-on-device`-Agent vorbehalten

### Konsistenz mit Repo-Standards
- **MUSS [MUST]** sicherstellen, dass die vom CLI erzeugten Dateien plus die Provenienz-Post-Processing-Edits zusammen `pre-commit run --all-files` ohne Auto-Fix-Modifikationen passen
- **SOLLTE [SHOULD]** dem Entwickler nach dem Scaffold die nächsten erwarteten Schritte als kurze Checkliste zurückgeben (z. B. „1. plan.md ausfüllen + User-Approval einholen, 2. SDK-Pin prüfen, 3. ReachyMiniApp.run() füllen, 4. On-Device-Test mit Agent X")

### Out-of-Scope-Klarstellung
- **DARF NICHT [MUST NOT]** Bewegungs-Logik in `main.py` vor-implementieren, die über das vom CLI gelieferte Demo hinausgeht; das ist Aufgabe des Entwicklers
- **DARF NICHT [MUST NOT]** an `reachy-mini-app-assistant`-Aufruf-Defaults schrauben, die der User nicht explizit überstimmt hat (z. B. heimlich `--publish` ausschalten, weil HF-Auth fehlt — stattdessen abbrechen mit Anleitung)
- **SOLLTE [SHOULD]** auf Nachbar-Skills verweisen (`reachy-mini-sdk`, `home-assistant-bridge`, `audio-beat-tracking`, Agent `reachy-mini-on-device`) statt deren Inhalte zu duplizieren

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/app-scaffold/SKILL.md` mit gültiger Frontmatter (`name: app-scaffold`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Der Skill ruft intern `reachy-mini-app-assistant create` auf — kein einziges Manifest- oder Layout-File wird vom Skill selbst geschrieben (nur Provenienz-Post-Processing)
- [ ] Default ist `--publish=true`; ohne `hf auth whoami` bricht der Skill mit klarer Anleitung ab und führt **nicht** stillschweigend einen Local-only-Lauf durch
- [ ] Bei `template=conversation` wird das Pollen-Conversation-Template gewählt; Default ist `default`
- [ ] Nach dem CLI-Lauf passiert `reachy-mini-app-assistant check <pfad>` ohne Findings
- [ ] Provenienz-Post-Processing ergänzt `pyproject.toml` `[project.urls]`, eine `CLAUDE.md` im App-Repo und einen Provenienz-Block im `README.md`
- [ ] Es wird ein `plan.md` im App-Verzeichnis erzeugt, dessen Inhalt dem Plan-Template-Stub aus `reachy-mini/app-development-workflow` § Phase 3 entspricht (Abschnitts-Namen und -Reihenfolge unverändert übernommen)
- [ ] Die Next-Steps-Checkliste benennt das User-Approval auf `plan.md` als ersten Schritt, vor jedem Code-Commit
- [ ] Test-Stub `tests/test_smoke.py` läuft gegen `ReachyMini(spawn_daemon=True, use_sim=True)` ohne Hardware durch und gated auf GStreamer-Verfügbarkeit
- [ ] Test-Stub kommentiert, welche Aspekte Simulation nicht prüfen kann
- [ ] README-Stub trägt eine Plattform-Tabelle (Wireless / Lite / Simulation)
- [ ] App-Name wird auf kebab-case validiert; Verstöße brechen mit klarer Fehlermeldung ab; `reachy-mini-app-assistant` normalisiert intern auf snake_case für den Python-Package-Namen — diese Normalisierung wird im Skill-Output ausgewiesen
- [ ] Bei Namens-Kollision (Ziel-Pfad existiert) bricht der Skill ab und benennt den existierenden Pfad
- [ ] `pre-commit run --all-files` läuft auf den generierten Dateien grün
- [ ] Verweise auf `reachy-mini-sdk`, `reachy-app-publish-hf`, `home-assistant-bridge`, `audio-beat-tracking` und `reachy-mini-on-device` sind im Skill-Body sichtbar
- [ ] Die nach dem Scaffold ausgegebene Next-Steps-Checkliste ist im Skill als Konvention dokumentiert

## Quellen
- Upstream-SDK-Repo (Quelle der Wahrheit für Manifest-Schema und Hook-Signaturen): <https://github.com/pollen-robotics/reachy_mini>
- Pollens `AGENTS.md` (Quelle für `plan.md`-Konvention, „Never create app folders manually", Plattform-Tabelle): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- App-Templates des SDKs (was das CLI tatsächlich erzeugt, inkl. `pyproject.toml.j2`, `main.py.j2`, `README.md.j2`, `index.html.j2`, `style.css.j2`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates>
- App-Manager-Implementierung (kanonische Lifecycle-Erwartungen an `ReachyMiniApp.run(reachy_mini, stop_event)`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Upstream-Claude-Skill `create-app` (parallele Authoring-Quelle für CLI-Wrapping, `--publish`-Default, Template-Wahl): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/create-app.md>
- Upstream-Claude-Skill `setup-environment` (Pre-Flight-Heuristik: SDK installiert, Daemon erreichbar, `agents.local.md`-Konvention): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/setup-environment.md>
- Upstream-Claude-Skill `testing-apps` (Sim-Pfad-Konventionen für den Test-Stub): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/testing-apps.md>
- Upstream-Claude-Skill `debugging` (Erste-Smoke-Test-Heuristik, Daemon-Health-Check): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- SDK-Konzept-Doku (Apps, Core-Concept, Quickstart): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Konkretes Wrapper-Beispiel: das `reachy-mini-show`-Repo (`~/repos/github/reachy_mini_show`) ist der erste Konsument dieses Skills und dient als End-to-End-Referenz

## Offene Fragen
- Wie sieht das offizielle Behavior-Layout im aktuellen [`pollen-robotics/reachy_mini`](https://github.com/pollen-robotics/reachy_mini)-Repository konkret aus (Ordner-Struktur, Manifest-Datei-Name, Manifest-Schema)? Vor Skill-Implementierung verifizieren — siehe [`src/reachy_mini/apps/templates`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates).
- Welches Manifest-Schema verlangt Hugging Face Spaces für veröffentlichungs-fähige Behaviors? Welche Pflicht-Felder, welche Optionalfelder?
- Welche Längen- und Zeichensatz-Regeln gelten exakt für Behavior-Namen auf Hugging Face und im SDK?
- Wo lebt der Behaviors-Ordner üblicherweise im konsumierenden App-Repo? Konfiguration, Konvention oder Auto-Discovery?
- Welche Test-Framework-Konvention gilt (pytest, unittest, eigenes Pollen-Test-Harness)?
- Soll der Skill optional ein Beispiel-Behavior generieren, das _zusätzlich_ zum reinen Skelett eine Mini-Move-Sequenz zeigt, oder strikt nur das Skelett?
- Soll der Skill den SDK-Pin aus dem konsumierenden Repo übernehmen oder den Pin selbst halten? Vorschlag: aus dem Repo, mit klarer Fehlermeldung wenn er fehlt.
- Wie reagiert der Skill, wenn das offizielle Behavior-Layout sich ändert (z. B. neue Pflicht-Hooks)? Vorschlag: Drift-Audit-Ankopplung an `reachy-mini-sdk`-Drift-Check.
