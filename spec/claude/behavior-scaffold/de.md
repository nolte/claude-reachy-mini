# Behavior-Scaffold-Skill

Status: draft

## Kontext
Reachy-Mini-Behaviors (z. B. „tanzt zur Musik", „nickt auf Anruf von Home Assistant") sind das primäre Lieferobjekt der App-Entwicklung mit diesem Plugin. Pollen Robotics / Hugging Face geben für Behaviors eine kanonische Repository- und Modul-Form vor, die Behaviors lade- und veröffentlichungsfähig hält. Wer manuell scaffolded, weicht regelmäßig in Detail-Konventionen ab — Manifest-Felder, Hook-Signaturen, Test-Layout — und das wird beim ersten Lade- oder Publish-Versuch sichtbar. Dieser Skill `behavior-scaffold` erzeugt das vollständige, valide Skelett für ein neues Behavior, sodass der Entwickler nur noch Bewegungs-Logik einfüllt. Er ergänzt den Skill `reachy-mini-sdk` (Wissensbasis) um den schreibenden Pfad und delegiert alles, was über das Skelett hinausgeht, an spezialisierte Skills.

## Ziele
- Ein neues Behavior ist mit einem einzigen Skill-Aufruf strukturell vollständig vorhanden — Manifest, Modul, Hooks, Test-Stub, Doku-Stub
- Das Skelett folgt der offiziellen Pollen-Robotics-Behavior-Konvention und ist Hugging-Face-kompatibel, falls später eine Veröffentlichung gewünscht ist
- Kollisionen mit bestehenden Behavior-Namen werden vor dem Schreiben erkannt
- Generierte Dateien sind sofort syntaktisch valide und passen den Lint-/Pre-commit-Standards des Repos
- Der Skill bleibt schmal: er liefert das Skelett, nicht die Logik, und delegiert benachbarte Anliegen an die zuständigen Skills

## Nicht-Ziele
- Konkrete Bewegungs- oder Tanz-Logik (Aufgabe des Entwicklers; SDK-Wissen liefert `reachy-mini-sdk`)
- Veröffentlichung des Behaviors auf Hugging Face Spaces / Hub (eigener Skill `behavior-publish-hf` geplant)
- Audio-Analyse, Beat- und Tempo-Erkennung (eigener Skill `audio-beat-tracking` geplant)
- Home-Assistant-Anbindung des Behaviors (eigener Skill `home-assistant-bridge`)
- Live-Deployment / On-Device-Test (eigener Agent `reachy-mini-on-device`)
- Behavior-Refactoring oder -Migration auf eine neue SDK-Major-Version

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine treffsichere `description` liefern, die Claude Code aktiviert auf Formulierungen wie „neues Behavior für Reachy anlegen", „Behavior scaffolden", „create reachy mini behavior", „start a new dance behavior"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: behavior, scaffold, Reachy Mini, new
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist (z. B. wenn ein bestehendes Behavior nur editiert oder veröffentlicht werden soll)

### Eingabe-Parameter
- **MUSS [MUST]** mindestens den Behavior-Namen als Pflicht-Parameter erwarten, normalisiert zu ASCII-Kebab-Case
- **MUSS [MUST]** eine kurze Beschreibung (1–3 Sätze) für das Behavior-Manifest und den Doku-Stub annehmen
- **SOLLTE [SHOULD]** optional Autor (Default: aus `git config user.name`/`user.email`) und Tags (kebab-case, ≤5) entgegennehmen
- **SOLLTE [SHOULD]** den Ziel-Pfad parametrisierbar machen (Default: das Standard-Behaviors-Verzeichnis des konsumierenden Repos, ermittelt durch Konvention oder Konfiguration)

### Erzeugte Artefakte
- **MUSS [MUST]** einen Behavior-Ordner anlegen, dessen Layout der offiziellen Pollen-Robotics-Konvention entspricht — exakter Aufbau ist `> ⚠ TBD: validate against pollen-robotics/reachy_mini` und wird vor Skill-Implementierung bestätigt
- **MUSS [MUST]** das Behavior-Manifest mit allen Pflicht-Feldern erzeugen (Name, Description, Author, ggf. Version, ggf. SDK-Compat-Range); bei unbekannten Feldern lieber TBD-Stub als raten
- **MUSS [MUST]** das Behavior-Modul (Python) mit den Lifecycle-Hooks anlegen, die der SDK-Vertrag fordert (z. B. `setup`, `step`, `stop` — exakte Signaturen TBD bis verifiziert)
- **MUSS [MUST]** ein README- bzw. Docstring-Stub erzeugen, der Beschreibung, beabsichtigte Hardware-Voraussetzungen und einen Quickstart-Block enthält
- **MUSS [MUST]** einen Test-Stub anlegen, der mindestens das Behavior importiert und die Hook-Signaturen instanziiert; eigentliche Bewegungs-Tests dürfen `> ⚠ TBD: validate against real hardware` sein
- **SOLLTE [SHOULD]** einen `.gitignore`-konformen Ordner-Footprint hinterlassen (keine Cache-, Egg-Info-, IDE-Dateien einchecken)
- **KANN [MAY]** ein optionales Hugging-Face-Spaces-Manifest mit erzeugen, wenn Publishing absehbar ist; sonst auslassen, statt einen leeren Stub mit zu schreiben

### Validierung vor dem Schreiben
- **MUSS [MUST]** prüfen, ob ein Behavior mit demselben Namen bereits existiert; bei Kollision abbrechen und den Konflikt-Pfad benennen, statt zu überschreiben
- **MUSS [MUST]** den Behavior-Namen gegen die SDK- und Hugging-Face-Namens-Regeln validieren (kebab-case, ASCII, ≤<TBD> Zeichen) — Detail-Limits werden vor Skill-Implementierung bestätigt
- **SOLLTE [SHOULD]** die zu pinnende `reachy_mini`-SDK-Version aus der Repository-Konfiguration übernehmen (nicht aus dem Skill-Body raten); wenn der SDK-Pin im konsumierenden Repo fehlt, mit klarer Fehlermeldung abbrechen

### Konsistenz mit Repo-Standards
- **MUSS [MUST]** generierte Dateien so erzeugen, dass `pre-commit run --all-files` ohne Modifikation grün durchläuft (passende Newlines, keine Trailing-Whitespace, valide YAML/JSON)
- **MUSS [MUST]** alle hardware-abhängigen Annahmen mit `> ⚠ TBD: validate against real hardware` markieren, statt als Fakt zu formulieren
- **SOLLTE [SHOULD]** dem Entwickler nach dem Scaffold die nächsten erwarteten Schritte als kurze Checkliste zurückgeben (z. B. „SDK-Pin in Manifest setzen, Hooks füllen, On-Device-Test mit Agent X")

### Out-of-Scope-Klarstellung
- **DARF NICHT [MUST NOT]** Bewegungs-Logik im Behavior-Modul vor-implementieren, die über einen klar markierten Beispiel-Stub hinausgeht; das ist Aufgabe des Entwicklers
- **DARF NICHT [MUST NOT]** das Behavior automatisch auf Hugging Face veröffentlichen oder dafür Credentials erwarten; Veröffentlichung gehört zu `behavior-publish-hf`
- **SOLLTE [SHOULD]** auf Nachbar-Skills verweisen (`reachy-mini-sdk`, `home-assistant-bridge`, `audio-beat-tracking`, Agent `reachy-mini-on-device`) statt deren Inhalte zu duplizieren

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/behavior-scaffold/SKILL.md` mit gültiger Frontmatter (`name: behavior-scaffold`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Ein Test-Aufruf mit Namen, Beschreibung und (optional) Autor/Tags erzeugt einen Behavior-Ordner mit Manifest, Modul, README/Docstring und Test-Stub
- [ ] Manifest enthält Pflicht-Felder; unbekannte Detail-Felder sind TBD-markiert
- [ ] Behavior-Modul enthält die Lifecycle-Hooks (Signaturen TBD-markiert wo unverifiziert) und ist syntaktisch valide
- [ ] Test-Stub importiert das Behavior und prüft das Vorhandensein der Hooks; bewegungs-spezifische Tests sind TBD-markiert
- [ ] Bei Namens-Kollision bricht der Skill ab und benennt den existierenden Pfad
- [ ] Behavior-Name wird auf kebab-case und Längen-Grenze validiert; Verstöße brechen mit klarer Fehlermeldung ab
- [ ] `pre-commit run --all-files` läuft auf den generierten Dateien grün, ohne Auto-Fix-Modifikationen
- [ ] Generierte Dateien folgen dem offiziellen Pollen-Robotics-Behavior-Layout (sobald verifiziert) bzw. einem klar TBD-markierten Best-Effort-Layout, wenn das Layout im Quellbaum noch nicht final bestätigt ist
- [ ] Verweise auf `reachy-mini-sdk`, `behavior-publish-hf`, `home-assistant-bridge`, `audio-beat-tracking` und `reachy-mini-on-device` sind im Skill-Body sichtbar
- [ ] Die nach dem Scaffold ausgegebene Next-Steps-Checkliste ist im Skill als Konvention dokumentiert

## Offene Fragen
- Wie sieht das offizielle Behavior-Layout im aktuellen `pollen-robotics/reachy_mini`-Repository konkret aus (Ordner-Struktur, Manifest-Datei-Name, Manifest-Schema)? Vor Skill-Implementierung verifizieren.
- Welches Manifest-Schema verlangt Hugging Face Spaces für veröffentlichungs-fähige Behaviors? Welche Pflicht-Felder, welche Optionalfelder?
- Welche Längen- und Zeichensatz-Regeln gelten exakt für Behavior-Namen auf Hugging Face und im SDK?
- Wo lebt der Behaviors-Ordner üblicherweise im konsumierenden App-Repo? Konfiguration, Konvention oder Auto-Discovery?
- Welche Test-Framework-Konvention gilt (pytest, unittest, eigenes Pollen-Test-Harness)?
- Soll der Skill optional ein Beispiel-Behavior generieren, das _zusätzlich_ zum reinen Skelett eine Mini-Move-Sequenz zeigt, oder strikt nur das Skelett?
- Soll der Skill den SDK-Pin aus dem konsumierenden Repo übernehmen oder den Pin selbst halten? Vorschlag: aus dem Repo, mit klarer Fehlermeldung wenn er fehlt.
- Wie reagiert der Skill, wenn das offizielle Behavior-Layout sich ändert (z. B. neue Pflicht-Hooks)? Vorschlag: Drift-Audit-Ankopplung an `reachy-mini-sdk`-Drift-Check.
