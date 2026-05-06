# Deploy-Agent für Reachy-Mini-Apps

Status: draft

## Kontext
Sobald eine Reachy-Mini-App strukturell fertig ist (per `app-scaffold` aufgesetzt, Behavior-Code an den `reachy-mini-sdk`-Idiomen ausgerichtet), muss sie in die Pollen-Daemon-Umgebung eines echten Reachy-Mini-Geräts gelangen, bevor irgendeine weitere Aktivität — Live-Trial, Integrationstest, Demo — möglich ist. Per Hand bedeutet das: SSH zum Gerät (oder zum Host-PC bei Lite), prüfen dass keine andere App läuft, Repo per rsync in ein Deploy-Ziel synchronisieren, App in das Daemon-Python-Environment installieren, verifizieren dass der `reachy_mini_apps`-Entry-Point sichtbar ist. Die Schritte sind sequenziell, latenzbeschränkt, und produzieren Hunderte Zeilen Installer-/rsync-Output, die den Hauptthread einer Claude-Code-Konversation zumüllen würden. Der `reachy-mini-deploy`-Agent kapselt diesen Lifecycle in einer eigenen Tool-Session und liefert dem Hauptthread eine knappe strukturierte Zusammenfassung. Er deployt, er startet die App nicht — Starten ist Sache von `reachy-mini-start` (Skill, mit User-Prompts) oder `reachy-mini-on-device` (Agent, voller Trial-Lifecycle).

## Ziele
- Eine App erreicht die echte Pollen-Daemon-Umgebung in einem einzigen Agent-Aufruf
- Pollen-Vertragsverletzungen werden lokal erkannt, bevor irgendeine Aktion auf dem Gerät stattfindet
- Installer-Output und Dependency-Resolution-Rauschen landen in einem Log-Artefakt, niemals im Hauptkontext
- Der Agent bleibt schmal: deploy + verify, kein Run, keine Behavior-Änderung
- Das Ergebnis kommt als knappe `PASS` / `FAIL` / `ABORTED`-Zusammenfassung mit Verweis auf das Volltext-Log zurück
- Mehrere Aufrufer (Nutzer direkt, der Caller des `reachy-mini-on-device`-Trial-Agents, ein Automations-Hook) können den Agent mit dem gleichen Input-Schema nutzen

## Nicht-Ziele
- Starten der deployten App — gehört zu `reachy-mini-start` (Skill) und `reachy-mini-on-device` (Agent)
- Hardware-Inbetriebnahme (separater Skill geplant)
- Firmware-Flashen (separater Skill geplant)
- Behavior-/Motion-Code editieren (`reachy-mini-sdk`, `app-scaffold`)
- Hugging-Face-Publish (`reachy-mini-app-assistant publish`, oder ein zukünftiger `behavior-publish-hf`)
- Persistenter Watchdog / Auto-Redeploy — der Agent fährt einen einmaligen Deploy-Lifecycle, kein Daemon
- Pollen-Daemon-Restart, -Reload oder -Reconfiguration — out of scope, niemals durch diesen Agent
- Simulation: in `use_sim=True` gibt es nichts zu deployen; Simulation ist Revier des `reachy-mini-on-device`-Agents

## Skill-vs-Agent-Begründung
Diese Aufgabe ist als **Agent** modelliert, weil mehrere Begründungen aus `nolte-shared/spec/claude/skill-vs-agent/` zugleich greifen:

- **Mehrstufige Orchestrierung mit eigenen Fehlerklassen** — Pre-flight, Connect, Busy-Check, Venv-Discovery, Sync, Install, Verify, Disconnect. Jede Phase hat eigene Fehlersignaturen (Auth-Fehler, Lock-Konflikt, fehlendes Venv, Dependency-Konflikt, kaputter Entry-Point) mit eigenen Recovery-Pfaden.
- **Latenzbeschränkte Tool-Session** — `rsync`, `pip install` und `ssh`-Roundtrips dominieren; inline würde das den Hauptthread pro Phase um zig Sekunden blockieren.
- **Kontextfenster-Schutz** — Installer-Logs und Dependency-Resolution-Traces laufen routinemäßig auf Hunderte Zeilen pro Aufruf; der Agent reduziert das auf eine strukturierte Zusammenfassung und schreibt den Volltext nach `.audits/deploy/`.
- **Schmale Tool-Oberfläche** — Bash für `ssh`/`rsync`/`curl`, plus `Read`/`Glob`/`Grep` auf das lokale Repo. Kein Edit-Zugriff auf die App unter Deployment.
- **Distribution: `plugin`** — der Agent wird mit dem Plugin neben `reachy-mini-on-device` ausgeliefert.
- **Gegen-Dimension** — interaktive Mid-Flow-Bestätigungen (z. B. „Eine andere App läuft, anhalten?") sind bewusst aufgegeben; der Default `if_busy: abort` übernimmt den Daemon nie gewaltsam. Wenn eine interaktive Entscheidung nötig ist, greifen Aufrufer zum `reachy-mini-start`-Skill.

## Anforderungen

### Inputs
- **MUSS** `app_path` akzeptieren — ein lokales App-Repo-Verzeichnis, das eine `pyproject.toml` mit `reachy_mini_apps`-Entry-Point sowie eine HF-konforme `index.html` (Pollen-Vertrag) enthält
- **MUSS** `device` akzeptieren — einen SSH-Host. Bei `wireless` der Roboter selbst (`pollen@reachy-mini.local`-Stil); bei `lite` der **Host-PC**, der den Reachy via USB-C hält
- **MUSS** `platform` akzeptieren — exakt einen der Werte `wireless` oder `lite`. `simulation` ist out of scope und **MUSS** mit klarem Fehler abgelehnt werden
- **DARF** `mode` akzeptieren — `editable` (Default, schnell für Dev-Iteration) oder `release` (Snapshot-Wheel-Install)
- **DARF** `verify` akzeptieren — Boolean, Default `true`; bei `false` werden Entry-Point- und Import-Checks nach dem Install übersprungen
- **DARF** `if_busy` akzeptieren — `abort` (Default, stoppt nie eine fremde App) oder `wait` (bis zu 30 s pollen)
- **DARF** `dry_run` akzeptieren — Boolean, Default `false`; bei `true` nur Sync, kein Install, kein Verify
- **DARF KEINE** Klartext-Credentials in irgendeinem Input akzeptieren — Credentials kommen ausschließlich aus `ssh_config`/Environment

### Lifecycle
- **MUSS** den Lifecycle in dieser Reihenfolge durchlaufen: lokales Pre-Flight → Connect → Robot-Busy-Check → Deploy-Ziel ermitteln → Code-Sync → Install → Verify → Disconnect
- **MUSS** im Pre-Flight `reachy-mini-app-assistant check <app_path>` lokal ausführen; eine Vertragsverletzung bricht den Lifecycle ab, bevor irgendeine Aktion auf dem Gerät passiert
- **MUSS** vor dem Install einen Robot-Busy-Check über die Pollen-Daemon-API durchführen (`/api/apps/current-app-status` und `/api/daemon/robot-app-lock-status`). Wenn eine andere App den Lock hält, hängt das Verhalten von `if_busy` ab; `abort` ist der Default und der Agent **DARF NICHT** automatisch fremde Apps stoppen
- **MUSS** den Pollen-Daemon-Python-Interpreter auf dem Gerät dynamisch ermitteln, statt ihn fest einzucodieren; den ermittelten Pfad im Report festhalten. Akzeptable Reihenfolge: dokumentierter Venv-Pfad → `which reachy-mini-app-assistant`-Shebang → mit klarem Fehler auf Pollens Daemon-Installations-Doku verweisen
- **DARF** unter keinen Umständen ins System-Python des Geräts installieren
- **MUSS** `rsync` mit sicheren Excludes ausführen (`.git`, `.venv`, `__pycache__`, `.audits`, `*.egg-info`, `.pytest_cache`, `.ruff_cache`, `node_modules`) und **MUSS** `--delete` ausschließlich auf das Deploy-Ziel begrenzen
- **MUSS** bei `verify=true` und `dry_run=false` verifizieren, dass die deployte App in `entry_points(group='reachy_mini_apps')` (abgefragt im Pollen-Daemon-Python) auftaucht und das Entry-Point-Package fehlerfrei importiert
- **MUSS** Phasenergebnisse strukturiert festhalten (Phase, Status, Dauer, ggf. Fehlerklasse)
- **MUSS** beim Disconnect sauber abschließen — keine offenen SSH-Sessions, keine verwaisten Prozesse auf dem Gerät
- **DARF** den Pollen-Daemon nicht restarten, neuladen oder rekonfigurieren
- **KANN** `uv pip` gegenüber `pip` bevorzugen, wenn `uv` auf dem Geräte-PATH liegt; beide sind erlaubt

### Output / Output-Format
- **MUSS** in den Hauptthread ausschließlich eine strukturierte Zusammenfassung zurückgeben: Gesamtstatus (`PASS` / `FAIL` / `ABORTED`), Phasenergebnisse, App-Name, Entry-Point, Deploy-Pfad, Log-Artefakt-Pointer
- **MUSS** zwischen `ABORTED` (konnte aus externem Grund — Lock-Konflikt, SSH-Auth, fehlendes Venv — nicht weiterlaufen, kein Install-Versuch) und `FAIL` (ein Install- oder Verify-Schritt lief und scheiterte) unterscheiden
- **MUSS** das Volltext-Log nach `.audits/deploy/<ISO-timestamp>-<app-name>.log` ablegen und den Pfad in der Zusammenfassung nennen
- **DARF KEINE** Roh-Logs, Dependency-Resolution-Traces oder Credentials in den Hauptthread zurückgeben
- **SOLLTE** ein Feld `follow_ups` enthalten, das die natürlichen nächsten Schritte nennt (typischerweise: „dispatch `reachy-mini-on-device` for a live trial" oder „invoke `reachy-mini-start` to bring the app online")

### Sicherheit und Geheimnisse
- **MUSS** SSH-/Geräte-Credentials aus Environment oder `ssh_config` lesen, niemals aus Klartext-Argumenten
- **DARF KEINE** Credentials in Output-Log oder Zusammenfassung schreiben; jeden genannten Identifier maskieren
- **SOLLTE** SSH-Host-Key-Verification aktiv lassen; beim ersten Connect den Fingerprint im Output ausgeben statt automatisch zu akzeptieren
- **MUSS** plattformspezifische Auth-Modelle anwenden:
  - **Wireless**: SSH direkt zum Reachy
  - **Lite**: SSH zum Host-PC; der dortige Pollen-Daemon empfängt den Install über diese Session
- **MUSS** sicherstellen, dass `.audits/` in der `.gitignore` des Consumer-Repos steht, bevor dort Artefakte geschrieben werden; das Audit-Verzeichnis ist generiert, niemals committed

### Grenzen
- **SOLLTE** auf `app-scaffold` zeigen, wenn das lokale Pre-Flight zeigt, dass das App-Skelett selbst kaputt oder unvollständig ist
- **SOLLTE** auf `reachy-mini-sdk` zeigen, wenn das Pre-Flight SDK-Pin- oder Import-Probleme aufdeckt, die zur Behavior-Seite gehören
- **SOLLTE** auf `reachy-mini-start` (Skill) als natürlichen Folge-Schritt zeigen, sobald `verify` durchgelaufen ist und der Nutzer die App tatsächlich starten möchte
- **SOLLTE** auf `reachy-mini-on-device` (Agent) als natürlichen Folge-Schritt zeigen, wenn ein Live-Trial mit Telemetrie gewünscht ist
- **DARF** Inhalte aus diesen Artefakten nicht duplizieren — der Agent ist Deploy-Orchestrator, keine Wissensbasis
- **DARF** keine Geschwister-Agents dispatchen oder andere Skills aufrufen (verboten durch `spec/claude/skill-vs-agent/`)

## Akzeptanzkriterien
- [ ] Der Agent lehnt `platform=simulation` mit klarem Fehler ab und verweist auf den On-Device-Agent
- [ ] Der Agent führt `reachy-mini-app-assistant check <app_path>` lokal als ersten Schritt aus und bricht bei Vertragsverletzung ab, bevor er das Gerät kontaktiert
- [ ] Der Agent stoppt niemals automatisch eine fremde App auf dem Daemon; `if_busy=abort` ist der Default
- [ ] Der Agent installiert niemals ins System-Python des Geräts; die Auflösung in ein Pollen-Daemon-Venv ist verpflichtend
- [ ] Der Agent restartet niemals den Pollen-Daemon
- [ ] Der Agent verifiziert bei `verify=true`, dass die deployte App in `entry_points(group='reachy_mini_apps')` erscheint
- [ ] Der Output-Report nennt den Deploy-Pfad explizit (`via_ssh_direct` / `via_host_usb`)
- [ ] Der Output-Report unterscheidet `ABORTED` von `FAIL` gemäß obiger Regel
- [ ] Das Volltext-Log liegt unter `.audits/deploy/<timestamp>-<app-name>.log` und `.audits/` steht in der `.gitignore` des Consumer-Repos
- [ ] Der Agent existiert als `agents/reachy-mini-deploy.md` mit gültigem Frontmatter — `name: reachy-mini-deploy`, `description`, `distribution: plugin`, optionale Tags
- [ ] Die `description` aktiviert auf Phrasings wie „App auf den Reachy ausrollen", „auf das Gerät deployen", „aktuellen Stand auf den Reachy bringen", sowie englische Varianten
- [ ] Eine Skill-vs-Agent-Begründung ist im Agent-Body sichtbar (mindestens mehrstufige Orchestrierung, Kontextfenster-Schutz, schmale Tool-Oberfläche)
- [ ] Verweise auf `reachy-mini-start`, `reachy-mini-on-device`, `app-scaffold`, `reachy-mini-sdk` sind im Body sichtbar
- [ ] Aussagen ohne Hardware-Verifikation tragen einen `⚠ TBD: validate against real hardware`-Marker

## Referenzen
- Pollen-Vertragsvalidator (im Pre-Flight verwendet): `reachy-mini-app-assistant check <path>` (Teil des `reachy-mini`-Pakets)
- App-Lifecycle-Vertrag (`stop_event`, `wrapped_run`, App-Manager): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Daemon-REST-Surface (Busy-Check-Endpoints, Installed-Apps-Listing): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollens `AGENTS.md` (Entry-Point-Group, App-Konventionen): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Geschwister-Agent für Live-Trials: `agents/reachy-mini-on-device.md`
- Geschwister-Skill zum Starten der deployten App: `skills/reachy-mini-start/SKILL.md`

## Offene Fragen
- Welcher REST-Endpoint listet die Apps, die der Daemon aktuell registriert hat? `/api/apps/installed` (oder den tatsächlichen Pfad) beim ersten Hardware-Kontakt verifizieren und im Agent-Body fixieren
- Was ist das kanonische Deploy-Ziel auf Wireless — `~/apps/<name>/` unter dem `pollen`-User oder ein vom Daemon verwalteter Ort? Verifizieren und fixieren
- Aktualisiert der Pollen-Daemon seine `entry_points(group='reachy_mini_apps')`-Sicht automatisch nach einem `pip install`, oder ist ein Daemon-SIGHUP / Restart nötig, damit der neue Entry-Point sichtbar wird? Falls ein Daemon-Restart nötig ist, darf dieser Agent ihn nicht ausführen; der Nutzer (oder `reachy-mini-start`) macht das
- Stellt Pollens Daemon-Installation eine stabile ENV-Variable oder Pfad-Datei für das Venv bereit, oder müssen wir immer probieren? Probieren ist der sichere Default; eine ENV-Variable würde den Report vereinfachen
- Soll `dry_run=true` den Verify-Schritt gegen die zuvor installierte Version laufen lassen (Regressions-Check) oder strikt skippen? Tendenz: skippen, da Dry-Run zur schnellen Diff-Inspektion gedacht ist
- Bei Lite: stellt der Host-PC-Daemon dieselben `/api/apps/...`-Endpoints bereit wie der Wireless-Daemon, oder ist die API-Surface anders? Beim ersten Lite-Hardware-Kontakt verifizieren
