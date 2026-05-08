# MCP-Server-Bootstrap-Skill

Status: draft

## Kontext

Die Wissens-Spec [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/de.md) definiert die kanonische Form eines MCP-Servers, der Reachy Mini über die Daemon-REST-API einem MCP-Client (Claude Desktop, Claude Code, Cursor, eigener Client) verfügbar macht. Die Implementation lebt in einem separaten Repo (`reachy-mini-mcp-server`), das als PyPI-Paket distributiert wird. Was bisher fehlt, ist die **operative Schale**, die einen Entwickler in fünf Minuten von „Paket nicht installiert" zu „MCP-Client redet mit dem Reachy" bringt — installieren, konfigurieren, hochfahren, gegen den lokalen Daemon health-checken, dem MCP-Client-Frontend eine fertig formatierte Config-Snippet liefern.

Dieser Skill `mcp-server-bootstrap` ist genau diese operative Schale. Er ist **schmal**: er installiert kein Paket, das schon da ist, er startet den Server höchstens als bewusst kurzlebigen Health-Check-Subprozess, und er schreibt keine Tools selbst. Die Server-Implementation, das Tool-Inventar, die Sicherheits-Gates — das ist alles Sache der Wissens-Spec und des Server-Repos. Dieser Skill ist Operations: Vorbedingungen prüfen, Server hochfahren oder als Daemon-Unit registrieren, MCP-Client-Konfiguration auswerfen, Audit-Log-Schreibbarkeit verifizieren.

Begriffsklärung: „Bootstrap" hier = Erst-Inbetriebnahme oder Neustart eines bereits einmal installierten Servers, plus die Generierung der MCP-Client-Konfiguration; **nicht** Server-Code-Schreiben, **nicht** Tool-Implementation, **nicht** Distribution-Build.

## Ziele

- Eine Reachy-Mini-MCP-Server-Sitzung läuft mit einem einzigen Skill-Aufruf, vorausgesetzt der Pollen-Daemon und das Server-Paket sind verfügbar
- Die Pre-Flight-Checks aus der Wissens-Spec ([`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/de.md)) werden konsistent in derselben Reihenfolge durchgeführt — Daemon-Erreichbarkeit, Plattform-Detect, Audit-Log-Schreibbarkeit, MCP-Client-Konfiguration sichtbar
- Drei Lauf-Modi sauber abgegrenzt: **`stdio`** (Server als Subprozess des MCP-Clients, kurzlebig), **`http`** (eigenständiger lokaler Daemon-Prozess für mehrere Clients), **`systemd`** (System-Unit für Server auf provisioniertem Host)
- MCP-Client-Konfigurations-Snippets werden für die gängigen Frontends ausgegeben (Claude Desktop, Claude Code, Cursor) — copy-paste-fähig, nicht generisch hingewinkt
- Skill bleibt schmal: keine Server-Code-Edits, keine Daemon-Konfiguration, keine Plugin-/App-Lifecycle-Eingriffe; alles delegiert an Nachbarn

## Nicht-Ziele

- Die Server-Implementation selbst — der Code lebt in `reachy-mini-mcp-server` (separates Repo), dieser Skill scaffoldet kein Server-Repo
- Tool-Inventar-Definitionen oder Tool-Code — Spec-/Implementations-Sache, nicht Operations
- Daemon-Restart, App-Lock-Force-Release, Hardware-Recovery — keine Tier-3-Operationen aus der Wissens-Spec werden hier gespiegelt
- Auth-Konfiguration für Remote-Modus — Wissens-Spec sagt v1 ist localhost-only; Bootstrap deshalb nur localhost
- Plugin-/App-Distribution (`app-scaffold`, `reachy-app-publish-hf`)
- Live-Trial-Validierung gegen Hardware (`reachy-mini-on-device`-Agent)
- CI/CD-Integration, GitHub-Actions-Workflows, Cloud-Deployment
- Multi-User / Mandantentrennung — Wissens-Spec sagt Single-User-localhost; Bootstrap respektiert das

## Anforderungen

### Trigger und Aktivierung

- **MUSS [MUST]** eine `description` liefern, die Claude Code aktiviert auf Formulierungen wie „MCP-Server für Reachy Mini starten", „MCP server hochfahren", „start the Reachy MCP server", „bootstrap the reachy-mini MCP", „configure Claude Desktop for Reachy Mini"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: MCP, server, Reachy Mini, bootstrap, start, configure
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist: bei Server-Code-Edits (eigenes Repo), bei Tool-Implementation (eigenes Repo), bei Daemon-Restart (Hardware-Sache, nicht Bootstrap), bei Live-Trial-/On-Device-Tests ([`reachy-mini-on-device`](../reachy-mini-on-device/de.md)-Agent), bei Plugin-Releases (`nolte-shared:release-publish-trigger`)

### Eingabe-Parameter

- **MUSS [MUST]** den Lauf-Modus annehmen (`mode`: `stdio` | `http` | `systemd`); Default ist `stdio` (kürzeste Latenz, einfachste Einrichtung, kein Lifecycle-Daemon nötig)
- **MUSS [MUST]** das gewünschte MCP-Client-Frontend annehmen (`frontend`: `claude-desktop` | `claude-code` | `cursor` | `generic`); aus dieser Wahl wird das Konfigurations-Snippet generiert
- **SOLLTE [SHOULD]** die Daemon-Adresse als optionalen Override-Parameter (`daemon_url`, Default `http://127.0.0.1:8000`) annehmen — Wireless mit mDNS-Default `http://reachy-mini.local:8000` separat unterstützen
- **SOLLTE [SHOULD]** den Bind-Port für `http`-Modus annehmen (`mcp_port`, Default `47600`) und vor dem Start auf Konflikt prüfen
- **SOLLTE [SHOULD]** ein optionales `log_level` annehmen (`info` | `debug`, Default `info`)
- **DARF NICHT [MUST NOT]** der Skill `0.0.0.0` als Default-Bind annehmen — localhost-only ist verbindlich aus der Wissens-Spec übernommen

### Pre-Flight-Pflichten (vor jeder Lauf-Aktion)

Der Skill **MUSS [MUST]** vor dem Server-Start diese Gates der Reihe nach prüfen und auf erstem fehlgeschlagenen Gate abbrechen mit einer konkreten Anleitung:

1. **Server-Paket installiert** — `reachy-mini-mcp-server --version` läuft im aktiven Python-Umfeld; fehlt es, abbrechen mit `uv tool install reachy-mini-mcp-server` (oder `uv pip install reachy-mini-mcp-server` im aktiven venv) als Empfehlung
2. **Daemon erreichbar** — HTTP-Probe gegen `<daemon_url>/api/daemon/status` (oder gleichwertigen Status-Endpunkt aus [`apps.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers) / [`daemon.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers)); auf Wireless mit mDNS-Lookup auf `reachy-mini.local`; Fehlschlag → Anleitung zum Daemon-Start (Lite: `reachy-mini-daemon`; Wireless: `systemctl status reachy-mini-daemon.service` per SSH)
3. **Plattform-Detect** — Server-Paket läuft eine kurze Plattform-Detect-Routine (`wireless` / `lite` / `simulation`); das erkannte Profil wird im Report ausgewiesen, damit der Operator weiß, welche Tier-1-Tools (IMU, Battery) verfügbar sind
4. **Audit-Log-Pfad schreibbar** — `~/.cache/reachy-mini-mcp/<YYYY-MM-DD>.log` muss erzeugbar sein; fehlende Permission abbrechen, niemals stillschweigend in `/tmp/` ausweichen
5. **Port frei (`http`-Modus)** — bei `mode=http` prüfen, dass `mcp_port` frei ist; auf Konflikt mit klarer Empfehlung abbrechen, nicht zufällig einen anderen Port wählen
6. **Existierender Daemon-MCP-Server-Prozess** — bei `stdio`/`http` prüfen, ob schon eine Instanz läuft (pid-File / Socket); bei Konflikt abbrechen, nicht zwei parallele Server starten

- **DARF NICHT [MUST NOT]** der Skill ohne grünen Pre-Flight in den Server-Start gehen
- **SOLLTE [SHOULD]** der Skill bei jedem fehlgeschlagenen Gate die nächste konkrete Aktion benennen (z. B. „Daemon nicht erreichbar → starte ihn mit `reachy-mini-daemon`")

### Lauf-Modi

| Modus | Lauf-Form | Lifecycle | Use-Case |
|---|---|---|---|
| **`stdio`** (Default) | Server als Subprozess des MCP-Clients, stdin/stdout-Pipe | Lebt mit der MCP-Client-Sitzung | Claude Desktop / Claude Code / Cursor — die meisten Frontends erwarten genau das |
| **`http`** | Server als eigenständiger lokaler Prozess, HTTP-Listener auf `127.0.0.1:<port>` | Eigener Prozess, vom User gestartet/gestoppt | Mehrere MCP-Clients gleichzeitig (z. B. Claude Code + lokales Web-UI), oder Sitzungen über mehrere Backends |
| **`systemd`** | Server als systemd-User-Unit, automatischer Restart | Daemonized, optional auf einem provisionierten Host laufend | Wireless-Reachy als „Always-on-Endpunkt" oder Lite-Host als persistenter MCP-Service |

- **MUSS [MUST]** der Skill für jeden Modus dokumentieren, wie der Server gestoppt wird (Strg-C / `pkill` / `systemctl --user stop`); kein Modus ohne klaren Stop-Pfad
- **SOLLTE [SHOULD]** der Skill für `systemd`-Modus die User-Unit-Datei (`~/.config/systemd/user/reachy-mini-mcp.service`) idempotent rendern, statt sie bei jedem Lauf zu überschreiben — der User darf manuelle Anpassungen behalten
- **DARF NICHT [MUST NOT]** der Skill systemd-System-Units anlegen (`/etc/systemd/system/`) — der MCP-Server ist Single-User, keine system-weite Service

### Workflow

1. **Pre-Flight** durchziehen (siehe oben); auf erstem Fehlschlag mit konkreter Aktion abbrechen
2. **Server starten** im gewählten `mode`:
   - `stdio`: Skill berichtet das Start-Kommando und überlässt das tatsächliche Spawning dem MCP-Client (Konfig-Snippet, siehe nächster Schritt)
   - `http`: Skill startet den Server als Background-Prozess, wartet 2 s, dann Health-Check
   - `systemd`: Skill rendert / aktualisiert die User-Unit, lädt systemd neu (`systemctl --user daemon-reload`), startet die Unit (`systemctl --user start reachy-mini-mcp.service`)
3. **Health-Check** ausführen — `tools/list`-Probe gegen den Server, erwarte mindestens das Tier-1-Inventar plus `health-check` (siehe Wissens-Spec)
4. **MCP-Client-Konfigurations-Snippet** generieren — Frontend-spezifisch, copy-paste-fähig, mit dem aktuellen `mode` und `daemon_url` befüllt
5. **Report** zurückgeben: Pre-Flight-Status, Server-Lauf-Status, Konfig-Snippet-Pfad oder -Inhalt, Stop-Anleitung

- **MUSS [MUST]** der Skill den Workflow strikt sequentiell ausführen — keine Parallelität, keine Background-Wartezeit ohne Health-Check
- **DARF NICHT [MUST NOT]** der Skill den Server-Code modifizieren, eine `pyproject.toml` schreiben oder Tools hinzufügen — das ist Implementations-Sache des Server-Repos

### MCP-Client-Konfigurations-Snippets

Pro Frontend ein **kanonisches** Snippet:

- **`claude-desktop`**: JSON-Eintrag in `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) bzw. `%APPDATA%/Claude/claude_desktop_config.json` (Windows); `mcpServers`-Block mit `command`, `args`, `env`-Mapping
- **`claude-code`**: Eintrag in `~/.claude/settings.json` `mcpServers`-Block (gleiches Schema)
- **`cursor`**: `mcpServers`-Eintrag in der Cursor-MCP-Konfig (Pfad versionsabhängig — Skill muss das aus der Cursor-Doku ziehen)
- **`generic`**: Allgemeines JSON, `mode=stdio`, mit Hinweis, dass jeder MCP-fähige Client das so konsumieren kann

- **MUSS [MUST]** der Skill für jeden Frontend-Wert den Pfad zur Konfig-Datei und das Schema des Eintrags ausweisen — niemals einen Snippet ohne Erklärung, wo er hingeht
- **SOLLTE [SHOULD]** der Skill anbieten, das Snippet **nur auszugeben** (Default), oder es in die Datei zu schreiben (Opt-in, mit Bestätigung) — nie ohne Bestätigung
- **DARF NICHT [MUST NOT]** der Skill bestehende `mcpServers`-Einträge des Users überschreiben ohne Diff-Anzeige und Bestätigung

### Out-of-Scope-Klarstellung

- **DARF NICHT [MUST NOT]** der Skill Server-Code, Tool-Code, oder Server-Konfiguration jenseits des Lauf-Modus schreiben — das gehört ins `reachy-mini-mcp-server`-Repo
- **DARF NICHT [MUST NOT]** der Skill den Pollen-Daemon starten, stoppen, oder neu konfigurieren — das ist Hardware-Bedienung, kein MCP-Bootstrap
- **DARF NICHT [MUST NOT]** der Skill den Reachy bewegen, Pose-Tools triggern oder Audit-Logs anderer Server-Instanzen lesen — der Skill ist Operations, nicht Tool-Aufruf
- **SOLLTE [SHOULD]** der Skill für ein dedicated Live-Trial auf den [`reachy-mini-on-device`](../reachy-mini-on-device/de.md)-Agent verweisen — der MCP-Server ist nicht für Bulk-Test-Lifecycles gedacht
- **SOLLTE [SHOULD]** der Skill auf [`reachy-mini-sdk`](../reachy-mini-sdk/de.md) für SDK-Idiome verweisen, wenn eine MCP-Client-Sitzung Folge-Fragen zu Method-Choice oder Safe-Torque hat

## Akzeptanzkriterien

- [ ] Skill ist unter `skills/mcp-server-bootstrap/SKILL.md` mit gültiger Frontmatter (`name: mcp-server-bootstrap`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description` enthält die Schlüsselbegriffe (MCP, server, Reachy Mini, bootstrap, start, configure) und benennt mindestens drei Anti-Trigger explizit
- [ ] Drei Lauf-Modi (`stdio`, `http`, `systemd`) sind dokumentiert, jeder mit klarem Lifecycle und Stop-Anleitung
- [ ] Sechs Pre-Flight-Gates werden in der spezifizierten Reihenfolge geprüft, jeder mit konkreter Fehler-Aktion
- [ ] Server wird **niemals** ohne grünen Pre-Flight gestartet
- [ ] Health-Check erfolgt nach Server-Start und vor Konfig-Snippet-Ausgabe
- [ ] MCP-Client-Konfigurations-Snippets sind für `claude-desktop`, `claude-code`, `cursor`, `generic` ausformuliert, mit dem korrekten Konfig-Datei-Pfad pro Frontend
- [ ] Snippet-Schreiben in Konfig-Datei ist Opt-in, mit Diff-Anzeige bei Konflikt mit existierenden `mcpServers`-Einträgen
- [ ] systemd-Modus rendert ausschließlich User-Units (`~/.config/systemd/user/`), niemals System-Units
- [ ] Server-Code wird **nicht** vom Skill modifiziert; Tool-Code wird **nicht** vom Skill geschrieben
- [ ] Cross-Refs auf [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/de.md), [`reachy-mini-sdk`](../reachy-mini-sdk/de.md), [`reachy-mini-on-device`](../reachy-mini-on-device/de.md), [`reachy-mini-start`](../reachy-mini-start/de.md), [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md) sind sichtbar
- [ ] Bei Daemon-Connection-Refused gibt der Skill eine konkrete Anleitung zur Daemon-Inbetriebnahme, niemals einen schweigenden Fehlschlag
- [ ] `pre-commit run --all-files` läuft auf der Skill-Datei grün

## Quellen

> Quell-Verweise auf Pollen-Code-Dateien zeigen auf das jeweilige Verzeichnis im Daemon-Tree; Markdown-Quellen sind Datei-Level zitiert. Konfigurations-Pfade pro MCP-Client folgen den jeweiligen Frontend-Doku-Stand.

- Wissens-Spec (kanonische Quelle für Tool-Inventar und Sicherheits-Modell): [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/de.md)
- Pollen-Daemon-Status-Endpunkt (Pre-Flight-Quelle): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers>
- Pollen-Daemon-Reachability-Pattern (Wireless mDNS, Lite localhost): [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md) und [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md) Z. 78
- Model Context Protocol — Spezifikation: <https://modelcontextprotocol.io>
- Model Context Protocol — Python-Server-SDK: <https://github.com/modelcontextprotocol/python-sdk>
- Claude Desktop MCP-Konfiguration: <https://docs.anthropic.com/en/docs/claude-code/mcp>
- systemd User-Units (für `systemd`-Modus): <https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html>
- Interne Cross-Refs:
  - [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/de.md) — Wissen über Tool-Inventar und Sicherheit
  - [`claude/reachy-mini-sdk`](../reachy-mini-sdk/de.md) — SDK-Idiome (Method-Choice, Safe-Torque), gegen die der Server-Operator Folge-Fragen stellt
  - [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md) — Test-Lifecycle als Schwester-Surface
  - [`claude/reachy-mini-start`](../reachy-mini-start/de.md) — App-Start-Pattern als Vorbild für saubere Pre-Flight-Sequenzen
  - [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/de.md) — systemd-Unit-Konventionen und mDNS-Default

## Offene Fragen

- Server-Paket-Name: ist `reachy-mini-mcp-server` der richtige Name auf PyPI? Pollen-Koordination wäre sinnvoll, falls sie selbst etwas Ähnliches publishen wollen.
- Health-Check-Tool: ist `health-check` als dedicated MCP-Tool im Server vorhanden, oder reicht ein `tools/list` als implizite Erreichbarkeits-Probe? Vorschlag: explizites `health-check`-Tool, weil ein leeres Tool-Inventar ein silent-bug wäre.
- `mcp_port` Default-Wert: `47600` ist ein plausibler unbelegt-Bereich, aber gibt es eine Pollen-/MCP-Konvention, gegen die wir uns abstimmen sollten?
- Konfigurations-Datei-Pfade pro Frontend: Cursor's MCP-Pfad ist versions-abhängig — Skill muss das pflegen oder dem User die Doku-Suche überlassen?
- Auto-Restart-Backoff im `systemd`-Modus: was sind sinnvolle Defaults (`Restart=on-failure`, `RestartSec=5s`)? Konsumenten-Konsens nötig.
- Multi-Reachy-Bootstrap: ein Entwickler hat zwei Geräte (z. B. Wireless + Lite) — soll der Skill mehrere Server-Instanzen parallel hochfahren, oder ist das App-Sache?
- Server-Update-Pfad: wie wird ein Versions-Bump des `reachy-mini-mcp-server`-Pakets im Bootstrap-Skill spürbar? Soll der Skill prüfen, ob das installierte Paket veraltet ist?
- MCP-Client-Konfig-Datei-Sicherung: vor dem Schreiben einen Backup erzeugen (`<file>.bak.<timestamp>`)? Vorschlag: ja, niedriger Aufwand, hoher Schutz.
