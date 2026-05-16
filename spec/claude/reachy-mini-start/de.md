# Start-Skill für Reachy-Mini-Apps

Status: draft

## Kontext
Eine Reachy-Mini-App, die in die Pollen-Daemon-Umgebung deployt wurde (typischerweise via `reachy-mini-deploy`-Agent), ist als Entry-Point registriert, aber noch nicht aktiv. Die App online zu bringen — also den Pollen-Daemon anzuweisen, ihren `ReachyMiniApp.run()` tatsächlich auszuführen — ist eine kurze, latenzarme Operation, deren entscheidendes Merkmal die Interaktion ist: der Daemon erzwingt die Single-App-Invariante, eine bereits laufende App muss also vorher gestoppt werden, und dieser Stopp ist eine zustandsändernde Entscheidung, die vom Nutzer bestätigt werden muss. Der `reachy-mini-start`-Skill kapselt diesen kurzen Start-Lifecycle im Hauptthread, mit expliziten User-Prompts an der einzigen Mehrdeutigkeit (Lock-Konflikt). Er startet, er deployt nicht und führt keinen vollen Trial — das sind der `reachy-mini-deploy`-Agent und der `reachy-mini-on-device`-Agent.

## Ziele
- Eine deployte App in einem einzigen Skill-Aufruf auf einem echten Reachy online bringen
- Die Single-App-Invariante an der Nutzeroberfläche explizit machen — niemals heimlich eine laufende fremde App stoppen
- Bestätigen, dass der Daemon nach dem Start-Call tatsächlich auf `running` umgeschwenkt hat — nicht nur dass der Start-Request angenommen wurde
- Schmal bleiben: starten und Running-State verifizieren, kein Deploy, kein Trial, keine Daemon-Lifecycle-Arbeit
- Sauber mit `reachy-mini-deploy` zusammenspielen: ein typisches End-to-End ist „Deploy → Start" oder „Deploy → On-Device-Trial"; der Skill ist eine Hälfte dieses Paars

## Nicht-Ziele
- Code auf dem Gerät deployen / installieren — gehört zu `reachy-mini-deploy` (Agent)
- Vollen Live-Trial mit Telemetrie laufen lassen — gehört zu `reachy-mini-on-device` (Agent)
- Behavior-/Motion-Code editieren — `reachy-mini-sdk`, `app-scaffold`
- Pollen-Daemon-Restart, -Reload oder -Reconfiguration — out of scope, niemals durch diesen Skill
- Hugging-Face-Publish — `reachy-mini-app-assistant publish`, oder ein zukünftiger `reachy-app-publish-hf`
- Persistenter Supervisor / Auto-Restart — der Skill ist Single-Shot

## Skill-vs-Agent-Begründung
Die Aufgabe ist als **Skill** modelliert, nicht als Agent, weil die Skill-Bias-Dimensionen aus `nolte-shared/spec/claude/skill-vs-agent/` direkt greifen:

- **Mid-Flow-Nutzerbestätigung ist erforderlich** — wenn eine andere App den Daemon-Lock hält, ist der einzig sichere Pfad, den Nutzer zu fragen. Ein Agent (Fire-and-forget) kann diese Frage nicht in den Hauptthread zurückrouten.
- **Output fließt natürlich in die Konversation zurück** — eine Zeile „App `<name>` auf `<device>` gestartet" plus State-Bestätigung; kein Bedarf für die Strukturreport-Grenze, die ein Agent bereitstellt.
- **Kurze Operation mit kleinem Volumen** — eine Handvoll REST-Calls; keine Installer-Logs, keine rsync-Transfers, keine Telemetrie-Samples. Kontextfenster-Schutz ist hier nicht ausschlaggebend.
- **Über die Konversation hinweg wiederholbar** — der Nutzer kann `reachy-mini-start` in einer Session mehrfach aufrufen (App A starten, zu B wechseln, zurück zu A), jede Invocation ist ein frischer Entscheidungspunkt.
- **Gegen-Dimension** — Tool-Restriktion (das Agent-Argument) greift hier nicht relevant; der Skill braucht Bash für `curl`/`ssh`, das hat der Hauptthread ohnehin.

## Anforderungen

### Inputs
- **MUSS** `app_name` akzeptieren — den installierten Namen, wie in `entry_points(group='reachy_mini_apps')` registriert
- **MUSS** `device` akzeptieren — einen SSH-Host / Daemon-Endpoint. Bei `wireless` der Roboter selbst; bei `lite` der Host-PC
- **MUSS** `platform` akzeptieren — exakt einen der Werte `wireless` oder `lite`. `simulation` **MUSS** mit klarem Fehler abgelehnt werden
- **DARF** `if_busy` akzeptieren — `prompt` (Default), `abort` oder `replace`. `prompt` fragt den Nutzer; `abort` lehnt im Busy-Fall stumm ab; `replace` stoppt die haltende App und startet die gewünschte **erst nachdem** der Aufrufer am Aufrufpunkt explizit bestätigt hat
- **DARF KEINE** Klartext-Credentials in irgendeinem Input akzeptieren — Credentials kommen ausschließlich aus `ssh_config`/Environment

### Lifecycle
- **MUSS** Pre-Flight-Checks vor jeder Zustandsänderung ausführen: HTTP-API-Erreichbarkeit, Hardware-Daemon-State, Installed-Apps-Listing, aktueller App-Lock-State
- **MUSS** beachten, dass der Pollen-Daemon zwei voneinander unabhängige Schichten betreibt: die **HTTP-API-Schicht** (FastAPI auf `:8000`, beantwortet `/api/apps/*` und `/api/daemon/*`) und einen **Hardware-Subprozess**, mit dem die App per IPC redet, sobald sie `ReachyMini()` instanziiert. Apps benötigen beide gleichzeitig
- **MUSS** vor dem Start-Call `GET /api/daemon/status` lesen und sauber abbrechen, wenn `state != "running"`. Dann mit klarem Diagnostic (gemeldeter State, vorgeschlagene Daemon-Bring-up-Aktion `POST /api/daemon/start` außerhalb dieses Skills) beenden; **DARF** den Hardware-Daemon nicht selbst starten — gehört zum Daemon-Lifecycle-Scope
- **MUSS** sauber abbrechen, wenn `app_name` nicht im Daemon-Entry-Point-Katalog sichtbar ist, und `reachy-mini-deploy` als natürlichen Folge-Schritt benennen; **DARF** keinen Install-Versuch unternehmen
- **MUSS** auf den Lock-State explizit verzweigen:
  - `free` → starten
  - bereits von `app_name` gehalten → No-op-Erfolg (App läuft schon)
  - von einer anderen App gehalten → auf `if_busy` verzweigen
- **MUSS** bei `if_busy=prompt` den Nutzer einmal mit Holder-Namen und vorgeschlagener Aktion fragen und auf explizite Bestätigung warten; **DARF** kein „Ja" aus einer früheren Invocation übernehmen
- **MUSS** bei `if_busy=replace` akzeptieren, dass der Aufrufer am Aufrufpunkt bereits bestätigt hat; der Skill prompted nicht doppelt
- **MUSS** nach dem Start via `GET /api/apps/current-app-status` verifizieren, dass der Daemon die angeforderte App als `running` meldet; ein 200 OK auf den Start-Call allein ist **nicht** ausreichend. Das Poll-Budget ist 5–15 s mit 1-s-Intervallen
- **MUSS** den Post-Start-State `error` als eigenständigen Failure-Branch behandeln: der App-Prozess kann gespawnt werden, in der `ReachyMini()`-Initialisierung scheitern und vom Daemon mit `state=error` plus Traceback in `current-app-status.error` gemeldet werden. Häufigste Wurzel: Hardware-Daemon-Layer war nicht `running`. Der Skill **MUSS** in diesem Fall das `error`-Feld an den Nutzer durchreichen, auf den Daemon-Lifecycle als nächsten Schritt zeigen und nicht stumm nochmal starten
- **MUSS** beachten, dass `current-app-status.state="error"` **sticky** ist: der Daemon räumt das App-Slot-Memorial nicht selbst auf. Solange das Memorial steht, lehnt der `POST /api/apps/start-app/<name>`-Endpunkt jeden neuen Start mit **`HTTP 400 "An app is already running"`** ab — auch wenn `robot-app-lock-status.state == "free"` ist. Der Daemon liest also `current-app-status` als Conflict-Source, nicht `lock-status`. Der Skill **MUSS** beide Endpunkte zusammen interpretieren und darf nicht aus dem Lock-Status allein „kein Konflikt" schließen
- **MUSS** bei einem sticky `state=error`-Memorial eine Recovery-Sequenz anbieten: User-Bestätigung einholen → `POST /api/apps/stop-current-app` aufrufen → verifizieren, dass `GET /api/apps/current-app-status` jetzt `null` zurückgibt → `POST /api/apps/start-app/<name>` erneut feuern. Die Recovery ist **genau einmal** pro Skill-Invocation erlaubt; nach einem zweiten `state=error` muss der Skill aussteigen und die Wurzel an den Nutzer durchreichen, statt in einer Retry-Schleife zu hängen
- **MUSS** dem Nutzer eine einzeilige Bestätigung mit gestartetem App-Namen und Device ausgeben; falls die App ein `custom_app_url` deklariert, dies als Hinweis erwähnen, ohne auf seine Erreichbarkeit zu warten
- **DARF** den Pollen-Daemon unter keinen Umständen restarten, neuladen oder rekonfigurieren — auch nicht den Hardware-Subprozess via `POST /api/daemon/start` oder `POST /api/daemon/stop`
- **DARF** keine laufende fremde App ohne explizite Nutzerbestätigung force-stoppen
- **DARF** als Teil dieses Lifecycles nichts installieren, modifizieren oder kopieren

### Output
- **MUSS** eine einzeilige, nutzerseitig sichtbare Bestätigung im natürlichen Konversations-Flow zurückgeben
- **SOLLTE** den vom Daemon gemeldeten App-State und einen umsetzbaren Hinweis enthalten (z. B. „FastAPI-Settings-UI auf `http://<device>:<port>`")
- **DARF KEINE** Roh-Daemon-Logs, Dependency-Traces oder Credentials zurückgeben

### Sicherheit und Geheimnisse
- **MUSS** SSH-/Geräte-Credentials aus Environment oder `ssh_config` lesen
- **DARF KEINE** Credentials in irgendeiner nutzerseitig sichtbaren Antwort schreiben
- **SOLLTE** SSH-Host-Key-Verification aktiv lassen; den Fingerprint beim ersten Connect ausgeben statt automatisch zu akzeptieren
- **MUSS** plattformbewusste Endpoint-Auflösung anwenden: bei Wireless ist der Daemon-Endpoint der Roboter, bei Lite der Host-PC

### Grenzen
- **SOLLTE** auf `reachy-mini-deploy` zeigen, sobald die gewünschte App noch nicht im Daemon-Entry-Point-Katalog steht
- **SOLLTE** auf `reachy-mini-on-device` zeigen, sobald der Nutzer testen, beobachten oder Behavior-Eigenschaften verifizieren möchte — nicht nur die App online bringen
- **SOLLTE** auf `reachy-mini-sdk` und `app-scaffold` zeigen, wenn das Problem upstream des Start-Calls liegt (kaputter Behavior-Code, kaputtes App-Skelett)
- **DARF** Inhalte aus diesen Artefakten nicht duplizieren — dieser Skill ist ein Daemon-seitiger Start-/Verify-Flow, keine Wissensbasis
- **DARF** in seiner eigenen Logik keine anderen Skills aufrufen; es ist eine Blattoperation

## Akzeptanzkriterien
- [ ] Der Skill lehnt `platform=simulation` mit klarem Fehler ab und verweist auf den On-Device-Agent
- [ ] Der Skill prüft im Pre-Flight `GET /api/daemon/status` und bricht ab, wenn `state != "running"`, mit Diagnostic und Verweis auf den Daemon-Lifecycle (`POST /api/daemon/start`, **nicht** in diesem Skill ausgeführt)
- [ ] Der Skill bricht ab, wenn `app_name` nicht in `entry_points(group='reachy_mini_apps')` steht, und benennt `reachy-mini-deploy` als nächsten Schritt
- [ ] Der Skill stoppt niemals automatisch eine laufende fremde App; `if_busy=prompt` fragt immer, `if_busy=replace` verlangt Upstream-Bestätigung, `if_busy=abort` beendet ohne State-Änderung
- [ ] Der Skill restartet niemals den Pollen-Daemon — weder HTTP-API-Layer noch Hardware-Subprozess
- [ ] Der Skill installiert, modifiziert oder kopiert niemals Code
- [ ] Nach erfolgreichem Start verifiziert der Skill über den Status-Endpoint des Daemons, dass die angeforderte App als `running` gemeldet ist — ein 200 OK auf `start-app` allein reicht nicht
- [ ] Wenn `current-app-status.state == "error"` nach dem Start, gibt der Skill das `error`-Feld inklusive Traceback-Snippet an den Nutzer durch, identifiziert das `ConnectionError: Could not connect to daemon on localhost`-Pattern als Hardware-Layer-Diagnose und versucht **nicht** stumm einen Re-Start
- [ ] Wenn vor dem Start ein sticky `current-app-status.state == "error"` aus einem früheren Lauf steht und der `start-app`-Call deshalb mit `HTTP 400 "An app is already running"` abgelehnt wird, bietet der Skill nach Nutzerbestätigung eine einmalige Recovery-Sequenz `stop-current-app → start-app` an; ein zweiter Fehler beendet den Skill ohne weiteren Retry
- [ ] Der Skill interpretiert `robot-app-lock-status` und `current-app-status` zusammen und schließt nicht aus `lock-status.state == "free"` allein „kein Konflikt"
- [ ] Der Skill gibt eine einzeilige Bestätigung aus, die App, Device und einen etwaigen FastAPI-Hinweis aus den App-Deklarationen enthält
- [ ] Der Skill existiert als `skills/reachy-mini-start/SKILL.md` mit gültigem Frontmatter — `name: reachy-mini-start`, `description`, optionale `tags`
- [ ] Die `description` aktiviert auf Phrasings wie „start the app on the reachy", „bring this app online", „let reachy run the app now", „auf dem Reachy starten", „App auf dem Roboter starten"
- [ ] Eine Skill-vs-Agent-Begründung ist im Skill-Body sichtbar (mindestens Mid-Flow-Nutzerbestätigung, Output fließt natürlich, kurze Operation mit kleinem Volumen)
- [ ] Verweise auf `reachy-mini-deploy`, `reachy-mini-on-device`, `app-scaffold`, `reachy-mini-sdk` sind im Body sichtbar
- [ ] Aussagen ohne Hardware-Verifikation tragen einen `⚠ TBD: validate against real hardware`-Marker; der verifizierte REST-Vertrag (Abschnitt unten) trägt diesen Marker **nicht**

## Verifizierter REST-Vertrag

Verifikationsbasis: **Reachy Mini Wireless, Firmware 1.7.1, `reachy-mini.local`, 2026-05-12.** Die folgenden Endpunkt-Pfade und Response-Formen sind als beobachtet bestätigt und ersetzen die früheren TBD-Annahmen. Für Lite gilt der Vertrag **noch nicht** als verifiziert (siehe Offene Fragen).

| Methode + Pfad | Response (Beispiel) | Verwendung im Skill |
|---|---|---|
| `GET /api/daemon/status` | `{"type":"daemon_status","state":"running"\|"stopped","wireless_version":true,"backend_status":{"ready":true\|false,"motor_control_mode":"enabled","control_loop_stats":{...},"error":null},"wlan_ip":"...","version":"1.7.1",...}` | Pre-Flight Hardware-Layer-Check (MUSS-Anforderung). Nur `state` ist Pre-Flight-relevant; `backend_status.ready=false` ist **kein** Show-Stopper (beobachtet: Start war erfolgreich bei `state="running" + backend_status.ready=false`) |
| `GET /api/daemon/robot-app-lock-status` | `{"state":"free","holder_name":null}` oder `{"state":"local_app","holder_name":"<app>"}` | Pre-Flight Lock-Branching. Beobachtete States: `free`, `local_app`. Weitere Werte (vermutlich `hf_space` o. ä. für Marketplace-Apps) noch nicht beobachtet |
| `GET /api/apps/list-available/installed` | Array von `{"name":"<entry_point>","source_kind":"installed","description":"","url":null,...}` | Pre-Flight Entry-Point-Lookup |
| `GET /api/apps/current-app-status` | `null` (nichts gestartet) **oder** `{"info":{"name":"...","source_kind":"installed",...},"state":"starting"\|"running"\|"error","error":null\|"<traceback>"}`. **State `error` ist sticky** — bleibt bis zu einem `stop-current-app`-Call stehen | Post-Start Verifikation und Failure-Diagnose; primäre Conflict-Source für `start-app` |
| `POST /api/apps/start-app/{app_name}` | Erfolg: `{"info":{...},"state":"starting","error":null}`. Konflikt: `HTTP 400 {"detail":"An app is already running"}` (siehe Failure-Signature unten) | Start-Trigger |
| `POST /api/apps/stop-current-app` | `null` (HTTP 200). Räumt zusätzlich `current-app-status` auf `null` | Soft-Stop einer haltenden App **und** Cleanup eines sticky `state=error`-Memorials |

### Beobachtete Failure-Signatures

**Signature 1 — Hardware-Daemon `stopped` beim Start-Versuch.** App wird gespawnt, scheitert in `ReachyMini()._initialize_client`:

```text
current-app-status.state == "error"
current-app-status.error  == "Process exited with code 1\n
                                ...
                                File \".../reachy_mini/reachy_mini.py\", line 441, in _initialize_client
                                  raise ConnectionError(
                              ConnectionError: Could not connect to daemon on localhost.
                              Is the Reachy Mini daemon running?"
```

Nach diesem Crash ist `robot-app-lock-status` wieder `free` (Daemon räumt den Lock auf), aber `current-app-status` behält den Error-State bis zum nächsten `stop-current-app`-Call.

**Signature 2 — Sticky `current-app-status.state="error"` blockiert nachfolgende Start-Versuche.** Auch wenn `robot-app-lock-status.state == "free"` ist, antwortet der `start-app`-Endpunkt:

```text
POST /api/apps/start-app/<name>
→ HTTP 400 {"detail":"An app is already running"}
```

Der Daemon liest also `current-app-status` als Conflict-Source, nicht `lock-status`. Recovery: `POST /api/apps/stop-current-app` aufrufen (räumt das Memorial), dann erneut starten. Diese Sequenz ist im Lifecycle als einmaliger Auto-Recovery-Pfad spezifiziert.

Der Skill **MUSS** daher den Lock-State und den Current-App-State zusammen interpretieren, nicht jeden für sich.

## Referenzen
- App-Lifecycle-Vertrag: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Daemon-REST-Surface: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollens `AGENTS.md`: <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Geschwister-Agent für Code-Deployment: `agents/reachy-mini-deploy.md`
- Geschwister-Agent für Live-Trials: `agents/reachy-mini-on-device.md`
- Hardware-Verifikationslauf (Erst-Live-Trial des Skills): Reachy Mini Wireless, Firmware 1.7.1, `reachy-mini.local`, 2026-05-12 — Endpunkt-Pfade und Failure-Signatures aus diesem Lauf sind in „Verifizierter REST-Vertrag" eingelaufen

## Offene Fragen
- Bei Lite: ist die REST-Surface des Daemons identisch zu Wireless, oder hat der Host-PC-Daemon einen anderen Pfad-Prefix? Beim ersten Lite-Hardware-Kontakt verifizieren — der Wireless-Vertrag ist nun verifiziert (siehe Abschnitt „Verifizierter REST-Vertrag")
- Soll der Skill ein Flag „start-and-wait-for-app-ready" anbieten, das die `custom_app_url` der App pollt, bis sie 200 zurückliefert, oder reicht die Daemon-seitige `running`-Bestätigung? Tendenz: reicht; App-seitige Erreichbarkeit ist Sache des Konsumenten, nicht des Daemon-Vertrags
- Soll `if_busy=replace` in einen späteren zusammengesetzten Skill („deploy then start") durchgereicht werden, sodass der Nutzer nur einmal für beide Hälften bestätigt? Vertagen, bis der zusammengesetzte Skill existiert
- Soll der Skill bei `state="stopped"` des Hardware-Daemons einen separaten `reachy-mini-bring-up`-Skill aufrufen dürfen, oder bleibt das eine reine Diagnose-Ausgabe? Aktuell: reine Diagnose; eskalieren, sobald der Bring-up-Skill existiert
- Gibt es weitere `current-app-status.state`-Werte jenseits von `null`/`starting`/`running`/`error`? Pollen-Quellcode-Pfad `src/reachy_mini/apps/manager.py` für die Aufzählung lesen und in den verifizierten REST-Vertrag aufnehmen
- Gibt es weitere `robot-app-lock-status.state`-Werte jenseits von `free`/`local_app`? Speziell vermutet: ein eigener Wert für Marketplace-Apps (`hf_space` o. ä.). Beim ersten HF-Space-App-Start verifizieren und in den verifizierten REST-Vertrag nachtragen
- Welche Bedingungen flippen `backend_status.ready` von `false` auf `true`? In unserer Verifikation startete eine App erfolgreich bei `state="running" + backend_status.ready=false`. Falls `ready=true` eine Voraussetzung für bestimmte Sensor-API-Zugriffe (Kamera, IMU) ist, sollte dies im Skill als optionaler "extended pre-flight"-Check angeboten werden
