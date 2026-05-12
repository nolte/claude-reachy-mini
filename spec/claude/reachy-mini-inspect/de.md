# Read-Only-Inspect-Skill für den Reachy-Mini-Daemon

Status: draft

## Kontext

Wer einen Reachy-Mini-Daemon im Alltag bedient, will oft nur mal eben sehen, wie es gerade aussieht: Läuft der Daemon? Hält eine App das App-Lock? Sind die Motoren stiff oder compliant? Wo steht der Kopf? Wie laut ist der Speaker? Genau dieser Lesezugriff ist heute spürbar friction-belastet. Drei Pfade stehen zur Verfügung — und keiner passt zu einem schnellen „zeig mal":

1. **Manuelles `curl` gegen einzelne `/api/...`-Endpunkte** — der Nutzer muss die exakten Pfade kennen, JSON parsen und sich aus mehreren Antworten ein Bild zusammenbauen.
2. **Python-REPL mit dem `reachy_mini`-SDK** — funktioniert, aber lädt eine Bibliothek, hält evtl. eine SDK-Verbindung und ist für eine einzelne Frage überdimensioniert.
3. **Geplanter MCP-Server** (`spec/reachy-mini/mcp-server/`) — bedient LLM-Frontends außerhalb von Claude Code (Claude Desktop, Cursor) per MCP-Protokoll; ist als eigener Server-Prozess gedacht und braucht Setup. Der MCP-Server-Code lebt außerhalb dieses Plugins.

Was fehlt, ist eine **Skill-Distribution innerhalb von Claude Code** desselben Lesezugriffs: direkt im Hauptkontext, ohne Server-Setup, ohne SDK-Instanziierung. `reachy-mini-inspect` schließt diese Lücke. Er ist die komplementäre Skill-Distribution dessen, was der geplante MCP-Server unter Tier-1-Read abbildet — beide dürfen nebeneinander existieren und nutzen identisch die REST-Endpoints aus [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/de.md).

## Ziele

- Drei Operations-Modi mit klar abgegrenztem Output-Volumen: `quick` (Default, Minimal-Status), `full` (alle Read-Endpoints), `raw <endpoint>` (einzelner GET, durchgereicht)
- Antwort als kompakte Markdown-Tabelle / strukturierter Bericht im Hauptkontext — geeignet als Zwischenschritt in einer Konversation, nicht als Volldokument
- Read-Only-Garantie: der Skill ruft **nur** GET-Endpunkte auf, niemals einen mutierenden Endpunkt (POST / PUT / DELETE)
- Klare Trennung zur MCP-Server-Distribution: der Skill verlinkt die mcp-server-Spec, dupliziert aber keine Tools — beide Distributionen referenzieren dieselbe REST-API als Quelle der Wahrheit
- Geringe Hürde: keine Installation eines zusätzlichen Prozesses; der Skill ist sofort über Claude Code aufrufbar, sobald das Plugin geladen ist
- Verständliche Fehlerausgabe bei nicht erreichbarem Daemon — eine einzige klare Zeile, kein Python-Traceback, kein silent ignore

## Nicht-Ziele

- Schreibzugriff auf den Daemon (Set-Operations bleiben `reachy-mini-sdk` für SDK-Idiome bzw. dem MCP-Server für LLM-Konsumenten)
- Eigener Server-Prozess — das ist die mcp-server-Spec
- App-Lifecycle (install / start / stop / remove / update) — gehört zu `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`
- Behavior- oder Motion-Entwicklung — `reachy-mini-sdk`, `app-scaffold`
- Hardware-Diagnose unterhalb der REST-Surface (USB-Erkennung, Firmware-Versionen, Treiber-Logs) — Sache der Hardware-Troubleshooting-Doku von Pollen
- Streaming / High-Frequency-Polling — `quick` und `full` sind Snapshots; wer kontinuierlich Pose lesen will, gehört in eine SDK-App mit `set_target`-Loop
- Logging- oder Crash-Triage — dafür existiert `app-log-triage`
- Eigene Caching-Schicht — der Skill ist stateless; wer Antworten cachen will, tut das auf der Aufrufer-Seite

## Anforderungen

### Konfiguration und Daemon-Erreichbarkeit

- **MUSS [MUST]** der Daemon-Host konfigurierbar sein (Default `http://127.0.0.1:8000`, Wireless-mDNS-Override `http://reachy-mini.local:8000`, Lite localhost) — niemals als Default auf eine Nicht-localhost-Adresse zeigen
- **MUSS [MUST]** der Skill als erste Operation jedes Modus einen Erreichbarkeits-Check ausführen (`GET /api/daemon/status`) und bei Connection-Refused mit einer einzigen klaren Fehlermeldung abbrechen — kein Stacktrace, kein silent fallback
- **MUSS [MUST]** der Skill bei jeder Antwort den HTTP-Status validieren; HTTP `5xx` und Timeouts werden im Output **explizit** markiert, nicht als leere Zelle ausgegeben
- **SOLLTE [SHOULD]** der Skill einen Default-Request-Timeout von 5 s pro GET nutzen und in den Modi `quick` und `full` einen Gesamt-Timeout von 15 s nicht überschreiten

### Operations-Modi

- **MUSS [MUST]** der Skill drei Modi unterstützen — `quick` (Default), `full`, `raw <endpoint>`
- **MUSS [MUST]** der `quick`-Modus genau diese drei GETs ausführen und in einer Markdown-Tabelle zurückliefern: `GET /api/daemon/status`, `GET /api/daemon/robot-app-lock-status`, `GET /api/motors/status`
- **MUSS [MUST]** der `full`-Modus zusätzlich zu den drei `quick`-GETs den aggregierten Zustand und die Audio-/Volume-Lage abdecken — Quellen: `GET /api/state/full` (Pose, Body-Yaw, Antennen, DoA), `GET /api/media/status`, `GET /api/volume/current`, `GET /api/volume/microphone/current`, `GET /api/apps/current-app-status`
- **MUSS [MUST]** der `raw`-Modus den Pfad gegen das Endpoint-Inventar in [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/de.md) validieren — ein Pfad, der dort nicht gelistet ist, wird mit klarem Hinweis abgelehnt
- **MUSS [MUST]** der `raw`-Modus auf GET-Endpunkte beschränkt sein — auch wenn das Daemon-Inventar einen Endpunkt für andere Methoden kennt, lehnt dieser Skill jede Methode außer `GET` ab
- **SOLLTE [SHOULD]** der `full`-Modus die GETs parallelisieren, um die Snapshot-Latenz gering zu halten

### Read-Only-Garantie

- **DARF NICHT [MUST NOT]** der Skill jemals einen POST-, PUT-, DELETE-, PATCH-Endpunkt aufrufen — auch nicht als „harmlosen" Probe-Call (z. B. `POST /health-check`); Health-Liveness wird über den GET-Status-Endpunkt abgedeckt
- **DARF NICHT [MUST NOT]** der Skill das App-Lock berühren, ändern oder force-stoppen — der Skill liest den Lock-Zustand, mehr nicht
- **DARF NICHT [MUST NOT]** der Skill bei `quick` oder `full` Antwortzeiten in eine Schleife stecken — ein Snapshot pro Aufruf, keine Polling-Logik
- **DARF NICHT [MUST NOT]** der Skill eine eigene Caching-Schicht halten — jeder Aufruf trifft den Daemon neu; das hält den Skill stateless und vermeidet stale-data-Bugs

### Output-Format

- **MUSS [MUST]** der Output eine Markdown-Tabelle pro Sektion enthalten (z. B. „Daemon", „App-Lock", „Motoren", „Pose", „Audio", „Aktuelle App") — Spaltennamen in der Sprache der laufenden Konversation
- **MUSS [MUST]** der Output bei einer fehlenden Antwort die Zelle explizit als `unreachable`, `timeout` oder `http <code>` markieren — niemals leer lassen
- **MUSS [MUST]** der Skill in der Schlusszeile den autoritativen Daemon-Host und den Snapshot-Zeitstempel (ISO-8601 UTC) ausgeben, damit eine zweite Inspektion vergleichbar ist
- **DARF NICHT [MUST NOT]** der Skill rohe JSON-Bodies in den Hauptkontext spülen — der `raw`-Modus liefert die Antwort als formatierten Code-Block, alles andere wird auf die Felder reduziert, die für den Snapshot relevant sind

### Beziehung zu anderen Skills / Specs

- **MUSS [MUST]** der Skill-Body explizit auf [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/de.md) als autoritative Endpoint-Quelle verweisen
- **MUSS [MUST]** der Skill-Body in der Abgrenzungs-Sektion auf [`spec/reachy-mini/mcp-server/`](../../reachy-mini/mcp-server/de.md) verweisen und die komplementäre Distribution kurz erklären — warum beide existieren, was sie unterscheidet
- **SOLLTE [SHOULD]** der Skill bei Inspect-Wünschen, die über Read hinausgehen, auf den passenden Skill weiterleiten — Move-Operationen → `reachy-mini-sdk`, App-Start → `reachy-mini-start`, Live-Trial → `reachy-mini-on-device`, Log-Analyse → `app-log-triage`
- **DARF NICHT [MUST NOT]** der Skill Tools oder Operations aus der mcp-server-Spec duplizieren — die mcp-server-Spec beschreibt eine eigene Distribution; dieser Skill ist die Plugin-Skill-Distribution derselben Read-Surface

## Akzeptanzkriterien

- [ ] Skill existiert unter `skills/reachy-mini-inspect/SKILL.md` mit gültigem Frontmatter (`name: reachy-mini-inspect`, `description`, `distribution: plugin`)
- [ ] Die `description` aktiviert auf Phrasen wie „zeig mir den Zustand", „Daemon-Status abfragen", „Motorposition lesen", „wie steht der Reachy gerade", „show robot state", „check daemon health", „where is the head pointing"
- [ ] Quick-Mode ist Default — ein Aufruf ohne Argumente liefert genau die drei spezifizierten GETs
- [ ] Full-Mode deckt die in der Anforderung genannten Endpunkte ab und ist parallelisiert
- [ ] Raw-Mode validiert den Pfad gegen `spec/reachy-mini/daemon-rest-api/` und lehnt Nicht-GET-Anfragen ab
- [ ] Der Skill ruft im gesamten Code-Pfad keinen POST-, PUT-, DELETE-, PATCH-Endpunkt auf — verifizierbar durch Code-Review oder ein dediziertes Test-Snippet im Skill-Body
- [ ] Connection-Refused führt zu einer einzigen klaren Fehlermeldung mit dem Daemon-Host, ohne Stacktrace
- [ ] Output ist eine Markdown-Tabelle in der Sprache der laufenden Konversation; jede fehlende Antwort ist explizit markiert
- [ ] Querverweis auf `spec/reachy-mini/daemon-rest-api/` und `spec/reachy-mini/mcp-server/` ist im Skill-Body sichtbar
- [ ] DE- und EN-Spec sind strukturell synchron; Anforderungs- und Akzeptanzkriterien-Reihenfolge ist identisch

## Referenzen

- Autoritatives Endpoint-Inventar: [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/de.md) (wird mit PR #23 auf develop verfügbar)
- Komplementäre MCP-Distribution: [`spec/reachy-mini/mcp-server/`](../../reachy-mini/mcp-server/de.md)
- MCP-Server-Bootstrap-Skill (für die andere Distribution): [`spec/claude/mcp-server-bootstrap/`](../mcp-server-bootstrap/de.md)
- Live-Quelle der Endpoint-Wahrheit: `http://<daemon-host>:8000/openapi.json` — typische Hosts: `http://reachy-mini.local:8000` (Wireless), `http://127.0.0.1:8000` (Lite oder Sim)
- Skill-vs-Agent-Heuristik: [`spec/claude/skill-vs-agent/`](https://github.com/nolte/claude-shared/blob/develop/spec/claude/skill-vs-agent/) (im claude-shared-Plugin)
- Plattform-Limits und Hardware-Inventar als Lese-Kontext: [`spec/reachy-mini/control-surface/`](../../reachy-mini/control-surface/de.md)

## Offene Fragen

- Default-Timeout pro GET ist mit 5 s vorgeschlagen — verifizieren beim ersten Hardware-Kontakt; auf Wireless mit kalter mDNS-Auflösung kann der erste GET deutlich länger dauern
- Soll der `full`-Modus den aggregierten `GET /api/state/full` nutzen ODER die Einzeln-Endpunkte (`present_head_pose`, `present_body_yaw`, `present_antenna_joint_positions`, `doa`) abrufen und kombinieren? Trade-off: `/api/state/full` ist ein einziger Roundtrip aber potenziell mit weniger Detail; einzelne Endpunkte parallel sind mehr Requests aber granularer
- Wie behandelt der Skill den Edge Case „Daemon up, aber Antwort 200 mit leerem Body"? Vorschlag: als `unknown` markieren mit deutlichem Hinweis im Output, nicht als Erfolg werten
- Soll der `raw`-Modus die JSON-Antwort schon kanonisch formatiert ausgeben (sortierte Keys, 2-Space-Indent), oder den Body 1:1 durchreichen? Tendenz: kanonisch — vergleichbar zwischen Aufrufen
- Bei `quick` oder `full` mit gleichzeitig laufender App: soll der Snapshot kenntlich machen, dass die Werte unter App-Last entstanden sind? Vorschlag: ja, eine Zusatzzeile „observed while app `<name>` is running"
- Soll es einen `--json`-Schalter geben, der die Tabelle durch ein JSON-Objekt ersetzt für maschinelle Konsumenten? Halten wir bewusst offen — der Skill ist primär für Konversations-Konsum, JSON-Konsumenten greifen direkt auf die REST-API zu
