# Reachy-Mini-SDK-Skill

Status: draft

## Kontext
Das `reachy_mini`-Python-SDK von Pollen Robotics / Hugging Face ist die primäre Schnittstelle, um den Reachy-Mini-Roboter programmatisch zu steuern: Kopf-Bewegungen (Pan/Tilt/Roll), Antennen, Behaviors, optional Audio- und Vision-Streams. Claude Code soll bei jeder Berührung dieses SDKs idiomatischen, lauffähigen Code produzieren — dafür braucht es eine treffsichere Wissensbasis, die genau dann aktiviert wird, wenn die Aufgabe das SDK berührt, und sich beim Drift offenbart, statt veraltete Patterns zu wiederholen. Diese Spezifikation regelt, was der Skill `reachy-mini-sdk` liefert und welche Themen explizit anderen Skills überlassen bleiben.

## Ziele
- Claude Code erkennt zuverlässig, wenn eine Aufgabe das `reachy_mini`-SDK berührt, und aktiviert genau dann diesen Skill
- Claude Code erzeugt idiomatischen, gegen die offizielle API verifizierten Code
- Jeder Code-Vorschlag bezieht sich auf eine namentlich genannte SDK-Version
- Das Plugin macht Versions-Drift sichtbar, statt ihn zu kaschieren
- Nicht-Anliegen werden klar an spezialisierte Skills delegiert, statt diesen Skill aufzublähen

## Nicht-Ziele
- Hardware-Bringup, Kalibrierung, Firmware-Flash (eigener Skill geplant)
- Simulation / MuJoCo / URDF des Reachy-Modells (separater Skill möglich)
- Veröffentlichung von Behaviors auf Hugging Face Spaces / Hub (eigener Skill `behavior-publish-hf` geplant)
- Beat- und Tempo-Erkennung für Tanz-Anwendungen (eigener Skill `audio-beat-tracking` geplant)
- Home-Assistant-Integration (eigener Skill `home-assistant-bridge`)
- Scaffolding eines neuen Behaviors (eigener Skill `behavior-scaffold`)
- Live-Deployment / Test auf dem Gerät (eigener Agent `reachy-mini-on-device`)

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine treffsichere Skill-Description liefern, die Claude Code bei jeder Aufgabe aktiviert, die das `reachy_mini`-SDK berührt — erkennbar an Imports von `reachy_mini`, der Klasse `ReachyMini`, Behavior-Definitionen oder API-Aufrufen für Kopf-/Antennen-Bewegung
- **MUSS [MUST]** Schlüssel-Trigger-Begriffe in der Description enthalten: `reachy_mini`, `ReachyMini`, Behavior, Antennen, Pan/Tilt/Roll, Move
- **SOLLTE [SHOULD]** explizit benennen, _wann der Skill nicht_ aktiviert werden soll (z. B. reine Hardware-Bringup-Aufgaben oder reine Simulation ohne SDK-Kontakt)

### Wissensbasis-Inhalt
- **MUSS [MUST]** die folgenden API-Bausteine des SDKs dokumentieren:
  - Konstruktion und Connection-Management der `ReachyMini`-Instanz inklusive Lifecycle (Open/Close, Context-Manager, Sync- vs. Async-Variante)
  - Kopf-Bewegung: Pan/Tilt/Roll, absolute vs. inkrementelle Ziele, Wertebereiche, Default-Pose
  - Antennen-Steuerung: links/rechts, Winkel, Geschwindigkeit, Synchronisation mit Kopf-Moves
  - Behavior-Lifecycle: Setup, Update-Loop, Tick-Frequenz, sauberes Stoppen, Exception-Handling
  - Standard-Move-Primitives: Easing, Dauer, Interpolation, Komposition mehrerer Moves
- **MUSS [MUST]** mindestens ein lauffähiges, minimales Code-Beispiel pro dokumentiertem Bereich enthalten
- **MUSS [MUST]** für jedes Beispiel die SDK-Version benennen, gegen die es verifiziert wurde, und auf die offizielle Quelle verweisen (Pollen-Robotics-Doku oder offizielles GitHub-Repo)
- **SOLLTE [SHOULD]** Async-Patterns abdecken (Tasks, Cancellation, Cleanup bei Exceptions), wenn das SDK eine Async-Oberfläche bietet
- **SOLLTE [SHOULD]** typische Fehlerbedingungen benennen (Hardware nicht angeschlossen, USB-/Serial-Fehler, Protokoll-Mismatch zwischen SDK und Firmware)
- **KANN [MAY]** Hinweise zu Update-Frequenz, Latenz und Performance von Behaviors aufnehmen

### Code-Beispiel-Konventionen
- **MUSS [MUST]** alle Beispiele auf eine Python-Untergrenze zielen, die der offiziellen SDK-Anforderung entspricht; bei Drift wird die Untergrenze per Spec-Update angepasst
- **MUSS [MUST]** Beispiele im Stil zeigen, den das offizielle SDK selbst vorgibt (z. B. `with`-Statement, falls das SDK ein Context-Manager-Modell vorlebt)
- **MUSS [MUST]** jeden Snippet mit einem Quell-Verweis auf die offizielle Pollen-Robotics-Doku oder das offizielle GitHub-Repo versehen
- **DARF NICHT [MUST NOT]** Code-Beispiele enthalten, die ungeprüft aus älteren Reachy-SDKs (Reachy 2, Reachy Pro) übernommen wurden — Wiederverwendungen müssen markiert werden, wenn die API für Reachy Mini abweicht

### Versions-Pinning und Drift-Erkennung
- **MUSS [MUST]** im Skill-Body die SDK-Version benennen, gegen die der Skill aktuell verifiziert ist (z. B. `reachy_mini==0.x.y`)
- **SOLLTE [SHOULD]** einen wiederholbaren Drift-Check vorsehen: bei jedem neuen `reachy_mini`-Release wird der Skill gegen die aktuelle API geprüft, entweder manuell beim nächsten Touchpoint oder über einen geplanten Audit-Skill
- **MUSS [MUST]** Aussagen, die mangels Hardware oder mangels Verifikation nicht belegt sind, durch eine sichtbare Markierung kennzeichnen (z. B. `> ⚠ TBD: zu validieren mit echter Hardware`)

### Schnittstellen zu benachbarten Skills
- **SOLLTE [SHOULD]** auf den Skill `behavior-scaffold` verweisen, sobald die Aufgabe ein _neues_ Behavior anlegt — statt Scaffolding-Logik zu duplizieren
- **SOLLTE [SHOULD]** auf den Skill `home-assistant-bridge` verweisen, sobald die Aufgabe Reachy mit Home Assistant verbindet
- **SOLLTE [SHOULD]** auf den geplanten Skill `audio-beat-tracking` verweisen, sobald die Aufgabe Audio analysiert (z. B. für Tanz-Synchronisation)
- **SOLLTE [SHOULD]** auf den geplanten Agent `reachy-mini-on-device` verweisen, sobald die Aufgabe ein Behavior live auf dem Gerät testen will

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/reachy-mini-sdk/SKILL.md` mit gültiger Frontmatter (`name: reachy-mini-sdk`, `description`, optional `tags`) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description`-Frontmatter ist so geschrieben, dass Claude Code den Skill bei einer Test-Aufgabe aktiviert, die `from reachy_mini import ReachyMini` enthält
- [ ] Die Wissensbasis dokumentiert mindestens: Konstruktion / Lifecycle, Kopf-Bewegung, Antennen, Behavior-Loop, Cleanup
- [ ] Mindestens ein lauffähiges Code-Beispiel pro dokumentiertem Bereich existiert
- [ ] Jedes Beispiel trägt eine Quell-Referenz und benennt die SDK-Version
- [ ] Die geprüfte SDK-Version ist im Skill-Body explizit ausgewiesen
- [ ] Out-of-Scope-Themen (Bringup, Simulation, HF-Publishing, Beat-Tracking, HA-Bridge, Behavior-Scaffolding, On-Device-Testing) sind als „dafür gibt es Skill / Agent X" markiert
- [ ] Aussagen ohne Hardware-Verifikation tragen einen sichtbaren TBD-Hinweis
- [ ] Der Skill wird im MkDocs-Katalog korrekt gerendert (Build läuft `task docs --strict` ohne Fehler)

## Offene Fragen
- Welche genaue `reachy_mini`-Version pinnen wir initial? Vorschlag: die letzte stabile vor Hardware-Eintreffen, dokumentiert im Skill-Body.
- Hat das SDK eine offizielle Compatibility-Matrix mit Python-Versionen, die wir verlinken sollten?
- Sollen Code-Beispiele die Async- oder die synchrone Variante des SDKs favorisieren? Hängt davon ab, was das SDK tatsächlich primär anbietet.
- Wie tief sollen Behaviors-Konventionen für die Hugging-Face-Veröffentlichung in diesem Skill auftauchen, _bevor_ ein eigener `behavior-publish-hf`-Skill existiert?
- Welches Tag-Set ist sinnvoll? Vorschlag: `[reachy-mini, sdk, python, robotics]`. Endgültig im Frontmatter klären.
- Soll der Skill auch auf reine Simulations-Aufgaben (MuJoCo / URDF ohne echte Hardware) reagieren? Tendenz: nein — das gehört in einen separaten Simulation-Skill.
- Wie häufig wird der Drift-Check ausgeführt? Vorschlag: vierteljährlich oder bei jedem `reachy_mini`-Major-Release.
- Wer ist die autoritative Quelle bei Konflikten zwischen Pollen-Robotics-Doku und SDK-Source-Code? Vorschlag: Source wins, Doku als Sekundärquelle.
