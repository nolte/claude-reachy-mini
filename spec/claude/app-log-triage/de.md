# App-Log-Triage-Skill

Status: draft

## Kontext

Während der Reachy-Mini-App-Entwicklung sucht der Entwickler eine Failure-Klasse aus den verteilten Log-Quellen heraus, klassifiziert sie und entscheidet über die nächste Diagnose-Aktion. Die [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md)-Spec hat das Wissen darüber kodifiziert (drei Logger-Bäume, zwei Capture-Pfade, Plattform-Senken, Common-Issues-Triage-Katalog, Verify-Basics-First-Heuristik). Bisher fehlt aber ein operatives Gegenstück: die Anwendung dieses Wissens als wiederholbarer Workflow. Dieser Skill `app-log-triage` ist genau dieses Gegenstück.

Er ist die wiederverwendbare Antwort auf die Frage „warum läuft die App nicht / warum tut sie nicht, was ich erwarte". Er **liest** Logs, klassifiziert sie gegen den kanonischen Common-Issues-Katalog der Wissens-Spec und liefert eine Recovery-Empfehlung mit Verweis auf den zuständigen Skill / Agent. Er **schreibt** keine Logs, **fixt** keinen Code, **stoppt** keine andere App und **flutet** den Main-Context nicht mit Roh-Logs — Behavior-Fixes bleiben Sache des Entwicklers, große Volumes gehören zum [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)-Agent.

Begriffsklärung: „Triage" hier = **Schnell-Klassifikation einer aktuell beobachteten Failure-Klasse plus Recovery-Empfehlung**, nicht tiefe Performance-Analyse, kein Production-Incident-Postmortem, kein Hardware-Recovery.

## Ziele

- Eine treffsichere Aktivierungs-`description`, die auf typischen Triage-Anlässen während der App-Entwicklung zündet
- Konsistente Multi-Source-Log-Erfassung pro Plattform und Lauf-Modus, mit Default-Filtern, die HTTP-Rauschen ausblenden
- Deterministische Klassifikation einer Failure-Beobachtung gegen den Common-Issues-Triage-Katalog der Wissens-Spec
- Verbindlicher Verify-Basics-First-Fall-back, wenn die Klassifikation unsicher ist
- Strukturierter, kompakter Report — Klassifikation + Vertrauenseinschätzung + Recovery-Vorschlag mit Skill-/Agent-Querverweisen
- Bewusst schmal: keine Behavior-Fixes, kein Hardware-Recovery, keine eigenen Aufräumaktionen, kein Production-Logging

## Nicht-Ziele

- Behavior-Code-Fixes (App-Logik bleibt Sache des Entwicklers; SDK-Wissen liefert [`reachy-mini-sdk`](../reachy-mini-sdk/de.md))
- Hardware-Recovery (Pollen-Hardware-Troubleshooting-Doku unter <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting>)
- Production-Log-Analyse auf provisionierten Hosts ([`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md))
- On-Device-Test-Lifecycle inklusive Bulk-Log-Sampling — der [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)-Agent macht das mit eigener Schale für große Volumes
- Performance-Profiling, Tracing, Flame-Graphs
- Log-Persistenz / -Search / -Indexing-Backends — der Skill konsumiert Logs, er bewahrt sie nicht auf
- Eine generische Python-`logging`-Tutorial-Sammlung (Wissen liegt in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md))

## Anforderungen

### Trigger und Aktivierung

- **MUSS [MUST]** eine `description` liefern, die Claude Code aktiviert auf Formulierungen wie „triage app logs", „warum läuft die App nicht", „classify this failure", „diagnose this crash", „check what went wrong with the app", „log triage for Reachy Mini app"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: log, triage, debug, app, Reachy Mini, classify, diagnose, failure
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist: bei reiner On-Device-Test-Anforderung (gehört zu [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)), bei Production-Host-Triage (gehört zu [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md)), bei reiner Behavior-/Move-Komposition (gehört zu [`reachy-mini-sdk`](../reachy-mini-sdk/de.md)), bei Hardware-Recovery (Pollen-Doku)

### Eingabe-Parameter

- **MUSS [MUST]** einen Plattform-Hint annehmen (`wireless` / `lite` / `simulation`); fehlt der Hint, **MUSS [MUST]** der Skill ihn aus der Umgebung erschließen (z. B. mDNS-Lookup auf `reachy-mini.local`, lokaler Daemon-Port, oder via User-Rückfrage)
- **MUSS [MUST]** einen Lauf-Modus-Hint annehmen (`daemon-hosting` / `direct` / `pytest`); ohne Hint ist der Default `direct` — Begründung: das ist laut [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) der empfohlene Entwicklungs-Pfad
- **SOLLTE [SHOULD]** einen Zeitanker (`since`) annehmen, der dem `journalctl --since`-Format entspricht; ohne Hint Default `5 min ago`
- **SOLLTE [SHOULD]** auf Wireless einen Hostnamen-Hint annehmen (Default `reachy-mini.local` per Pollen-mDNS-Konvention)
- **SOLLTE [SHOULD]** ein Beobachtungs-Symptom in Worten annehmen (z. B. „robot doesn't move", „connection refused", „audio fails") — schärft die Klassifikation, ist aber optional

### Pre-Flight-Pflichten (vor jeder Log-Aktion)

- **MUSS [MUST]** prüfen, ob die erwartete Log-Quelle erreichbar ist:
  - **Wireless**: SSH-Reachability auf `pollen@<host>` plus `systemctl status reachy-mini-daemon.service` als Daemon-Heartbeat
  - **Lite**: Daemon-Prozess-Lokal-Status (z. B. `pgrep -f reachy-mini-daemon` oder Daemon-Port-Probe `lsof -i :8000`)
  - **Simulation**: kein Pre-Flight, weil im selben Prozess
- **MUSS [MUST]** bei fehlender Reachability die Failure-Klasse `connection-refused` (siehe Klassifikation) **direkt** zurückmelden, **ohne** Log-Erfassung zu versuchen — Begründung: ohne erreichbaren Daemon gibt es keine sinnvollen Logs zu klassifizieren
- **DARF NICHT [MUST NOT]** der Pre-Flight-Schritt eigenständig den Daemon (re)starten — Daemon-Restart ist Empfehlung, nie Aktion (siehe Out-of-Scope)

### Log-Erfassung

- **MUSS [MUST]** pro Plattform und Lauf-Modus die kanonische Log-Quelle aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) § Plattform-Profile auswählen — nicht eigene Quellen erfinden, nicht von der Spec abweichen
- **MUSS [MUST]** als Default-Filter `grep -v "uvicorn\|GET \|POST "` anwenden, um HTTP-Rauschen aus dem REST-Surface auszublenden — gleiche Konvention wie [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md) Z. 67–71
- **SOLLTE [SHOULD]** in `daemon-hosting`-Modus die drei Logger-Bäume (`reachy_mini.*`, `reachy_mini.daemon.*`, `reachy_mini.apps.manager.runner`) gemeinsam mitlesen, weil eine App-Aktion typischerweise Spuren in mehreren Bäumen hinterlässt
- **DARF NICHT [MUST NOT]** der Skill Roh-Logs in den Main-Context schreiben — die Erfassung wird als strukturiertes Aggregat geliefert (siehe Output / Report)
- **DARF NICHT [MUST NOT]** Log-Erfassung versuchen, wenn der Pre-Flight-Reachability-Check fehlgeschlagen ist
- **DARF NICHT [MUST NOT]** mehr als die letzten zwei Stunden Logs ziehen, ohne dem User vorher den Zeit-Anker zu bestätigen (Volumen-Schutz)

### Klassifikation

- **MUSS [MUST]** gegen jede Klasse aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) § Common-Issues-Triage-Katalog matchen: `connection-refused`, `app-lock-held`, `no-motion`, `jerky-motion`, `import-error`, `audio-fail`, `motors-different-states`, `daemon-stale-state`, `webrtc-plugin-missing`
- **MUSS [MUST]** bei Klasse `daemon-stale-state` (App startet mit `ConnectionError: Could not connect to daemon on localhost`, obwohl `systemctl is-active reachy-mini-daemon.service` `active` meldet) als Pre-Flight zusätzlich `ss -tln | grep ":6053"` ausführen (Wireless) — schweigt der Port, ist die Klasse bestätigt; Recovery-Empfehlung in den Report aufnehmen, **ohne** den Restart selbst auszuführen
- **MUSS [MUST]** bei Klasse `webrtc-plugin-missing` (`RuntimeError: Failed to create webrtcsrc element. Is the GStreamer webrtc rust plugin installed?` aus dem `ReachyMini`-Konstruktor im direct-mode) im Report **explizit** vermerken, dass die Verify-Basics-First-Sanity-Probe nicht durchführbar ist und das Problem **nicht** App-bezogen ist, sondern eine Setup-Lücke der direct-mode-Umgebung; das verhindert, dass nachgelagerte Skill-Aufrufe (`reachy-mini-sdk`, `app-scaffold`) aus dem Triage-Bericht App-Code-Hypothesen ableiten
- **MUSS [MUST]** pro Klasse das in der Wissens-Spec genannte Log-Muster verwenden, **nicht** freie Heuristik (z. B. für `app-lock-held` exakt das Pattern `RobotAppLock: rejected — held by <app_name>` über den Logger `reachy_mini.daemon.robot_app_lock`)
- **SOLLTE [SHOULD]** bei mehreren Match-Klassen die spezifischste mit höchstem Vertrauen melden, die Alternativen mit niedrigerem Vertrauen als „candidates" auflisten
- **MUSS [MUST]** bei No-Match die Pseudo-Klasse `unclassified` zurückgeben, **nicht** eine neue Klasse erfinden
- **SOLLTE [SHOULD]** bei `unclassified` einen Hinweis liefern, dass das Symptom eine Open Question für die `app-logging`-Spec ist und gegen Pollens Quellen ([`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md)) abgeglichen werden sollte

### Verify-Basics-First-Heuristik

- **MUSS [MUST]** bei Klasse `unclassified` als ersten Recovery-Vorschlag den `examples/minimal_demo.py`-Sanity-Check melden — gemäß der MUST-Klausel in [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) („vor jeder Diagnose einer App-spezifischen Failure-Klasse zuerst minimal_demo.py")
- **SOLLTE [SHOULD]** bei Klasse `no-motion` zusätzlich den Sanity-Check als ersten Schritt empfehlen — wenn `minimal_demo.py` selbst nichts bewegt, ist das Problem nicht App-, sondern Connectivity-/Hardware-bezogen
- **DARF NICHT [MUST NOT]** der Skill den Sanity-Check eigenständig ausführen — er ist Entwickler-Aktion; der Skill empfiehlt, listet den exakten Befehl, und meldet im nächsten Lauf das Ergebnis
- **MUSS [MUST]** bei Empfehlung des direct-mode-Sanity-Checks die mögliche `webrtc-plugin-missing`-Edge-Case mitkommunizieren (siehe [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) § Verify-Basics-First-Heuristik): wenn die produktive App im daemon-hosting läuft, ist eine **daemon-hosted Sanity-App** aussagekräftiger als ein direct-mode-Skript — der Skill empfiehlt im Zweifel den daemon-hosting-Pfad und vermerkt, dass `gst-plugin-webrtc-rust` keine App-Voraussetzung ist
- **SOLLTE [SHOULD]** bei aufeinanderfolgenden App-Stop/Start-Cycles (Hot-Patch-Diagnose) den Hinweis aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) § Recovery-Aktionen mitgeben: ~30 s Cooldown zwischen Cycles oder den Daemon-Stale-State-Recovery-Befehl bereit halten — das ist Vorbeuge-Empfehlung, kein Diagnose-Trigger

### Report-Format

- **MUSS [MUST]** einen kompakten, strukturierten Report zurückgeben mit den Sektionen: (1) detected platform / mode, (2) sources sampled (welche Logger-Bäume / Senken), (3) primäre Klasse mit Vertrauen, (4) Kandidaten-Klassen mit Vertrauen, (5) erwartetes Log-Muster (Zitat aus Wissens-Spec), (6) Match-Zeilen mit ±2 Kontextzeilen, (7) Recovery-Empfehlung mit Skill-/Agent-Cross-Ref
- **MUSS [MUST]** Pollen-Quell-Verweise pro Klasse aus der Wissens-Spec übernehmen, **nicht** neu zitieren
- **SOLLTE [SHOULD]** bei Klasse `motors-different-states` direkt auf das Safe-Torque-Pattern aus Pollens [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md) verweisen
- **SOLLTE [SHOULD]** bei Klasse `app-lock-held` den haltenden App-Namen aus dem Match-Pattern extrahieren und im Report nennen
- **DARF NICHT [MUST NOT]** Roh-Logs außerhalb der Match-Zeilen + Kontext einfügen — der Report ist eine Zusammenfassung, kein Log-Dump
- **DARF NICHT [MUST NOT]** Tokens, HF-Auth-Keys, WiFi-Credentials oder Sensor-Daten mit personenbezogenem Bezug in den Report aufnehmen (Konsistenz mit [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) MUST-NOT zu PII)

### Out-of-Scope-Klarstellung

- **DARF NICHT [MUST NOT]** Behavior-Code modifizieren — das ist Aufgabe des Entwicklers, ggf. mit Wissen aus [`reachy-mini-sdk`](../reachy-mini-sdk/de.md)
- **DARF NICHT [MUST NOT]** den Daemon eigenständig (re)starten — die Empfehlung wird im Report formuliert, ausgeführt vom User
- **DARF NICHT [MUST NOT]** eine zweite App stoppen, um App-Lock-Konflikte aufzulösen — auch das wird empfohlen, nicht ausgeführt
- **DARF NICHT [MUST NOT]** Hardware-Recovery vorschlagen, die über das Safe-Torque-Pattern hinausgeht — Mic-FPC-Cable, Motor-Replacement, Spherical-Joint-Maintenance gehören in Pollens Hardware-Doku
- **SOLLTE [SHOULD]** bei großen Log-Volumen (z. B. mehrstündige Sessions, Bulk-Triage über mehrere Failures) den User auf den [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)-Agent verweisen — der hat die Schale für device-side Bulk-Verarbeitung mit Artefakt-Persistenz

## Akzeptanzkriterien

- [ ] Skill ist unter `skills/app-log-triage/SKILL.md` mit gültiger Frontmatter (`name: app-log-triage`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description` enthält die Schlüsselbegriffe (log, triage, debug, app, Reachy Mini, classify, diagnose, failure) und benennt mindestens drei Anti-Trigger explizit
- [ ] Pre-Flight-Reachability-Check läuft vor jeder Log-Erfassung und meldet bei Fehlschlag direkt Klasse `connection-refused`, ohne weitere Log-Aktion
- [ ] Default-Filter `grep -v "uvicorn\|GET \|POST "` ist auf jede tatsächlich durchgeführte Erfassung angewandt (bei `connection-refused` entfällt die Erfassung, dann auch der Filter)
- [ ] Common-Issues-Klassifikation deckt alle sieben Klassen aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) ab und nutzt die dort genannten Log-Muster wörtlich
- [ ] Bei Match meldet der Skill Klasse + Vertrauen + erwartetes Log-Muster + Match-Zeilen mit Kontext + Recovery-Vorschlag
- [ ] Bei No-Match wird `unclassified` zurückgegeben, mit `minimal_demo.py`-Sanity-Check als ersten Vorschlag
- [ ] Der Report enthält keine Roh-Logs außer Match-Zeilen ±2 Kontextzeilen
- [ ] PII-Klausel ist erfüllt: keine Tokens / Auth-Keys / WiFi-Credentials im Report
- [ ] Cross-Refs auf [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md), [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md), [`claude/reachy-mini-sdk`](../reachy-mini-sdk/de.md), Pollens [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md) und [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) sind sichtbar
- [ ] Der Skill bricht ab oder verweist sauber, wenn die Anfrage in den Scope eines benachbarten Skills / Agents fällt (On-Device-Test, Production, Hardware-Recovery)
- [ ] `pre-commit run --all-files` läuft auf der Skill-Datei grün

## Quellen

> Cross-Refs auf interne Wissens-Specs sind verlinkt; Pollen-Code-Source-Anker, falls aufgenommen, sind gegen `pollen-robotics/reachy_mini@main` zu verifizieren — Konvention aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) § Quellen.

- Wissens-Spec (kanonische Quelle für Plattform-Tabelle, Common-Issues-Katalog, Verify-Basics-First, PII-Klausel): [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md)
- On-Device-Test-Agent (zuständig für device-side Bulk-Triage und Test-Lifecycle): [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)
- SDK-Wissensbasis (Idiomatic SDK-Use, `deep-dive-docs`-MUST): [`claude/reachy-mini-sdk`](../reachy-mini-sdk/de.md)
- Production-Logging auf provisionierten Hosts (klare Abgrenzung): [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md)
- Pollen-Skill `debugging` (Common-Issues-Quelle, Verify-Basics-First-Heuristik): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- Pollen-Skill `safe-torque` (Recovery-Pattern bei Motor-State-Mismatch): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Pollens `AGENTS.md` (Einstiegspunkt, allgemeine Konventionen): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Pollen-Beispiel `minimal_demo.py` (kanonischer Sanity-Check): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>

## Offene Fragen

- Skill vs. Agent-Trennung: aktuell macht [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md) device-side Test-Lifecycles inklusive Log-Tail. Bei langlebigen Triage-Sessions (mehrere Stunden, mehrere Failures) — gehört das in einen eigenen `reachy-mini-log-tail`-Agent, oder reicht der Skill-Verweis auf `reachy-mini-on-device`?
- Stderr-Heuristik aus [`apps/manager.py:206–209`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py): soll der Skill die exakten Marker-Strings replizieren, um stderr-Klassifikation deterministisch zu machen, oder reicht der grobe Pollen-Hinweis?
- Format des Reports: Markdown mit Tabellen-Sektion (lesbar im Chat) oder strukturiertes JSON-Block (parseable für Folge-Skills)? Vorschlag: Markdown als Default, JSON-Modus per optionalem Parameter, falls ein Folge-Skill konsumiert.
- Auto-Plattform-Erkennung: gibt es einen zuverlässigen Indikator (z. B. Antwortet `nc -z reachy-mini.local 8000`?, lokaler Daemon-Port?), oder bleibt der Plattform-Hint Pflicht?
- Sample-Akkumulation für lange Sessions: gehört das zur Skill (Stream-Window, n letzte Minuten) oder zum vorgeschlagenen Bulk-Agent?
- Soll der Skill optional eine `mockup-sim`-Variante (ohne MuJoCo / GStreamer) als Sanity-Check-Fallback empfehlen, wenn die Sim-Dependencies fehlen?
- Match-Zeilen-Kontext: ist „±2 Kontextzeilen" das richtige Default, oder besser ±5 mit kürzerem Linecap pro Zeile?
- Permissions: braucht der Skill auf Wireless `sudo`-Rechte für `journalctl`, oder reicht User-Read-Access durch die Pollen-Default-Konfiguration? Gegen Pollen-Doku verifizieren.
