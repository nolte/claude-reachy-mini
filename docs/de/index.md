# claude-reachy-mini

Claude-Code-Plugin mit Skills und Agents für die effiziente Entwicklung mit dem [Reachy Mini](https://www.pollen-robotics.com/reachy-mini/) — dem Desktop-Roboter von Pollen Robotics / Hugging Face.

## Worum es geht

Dieses Plugin liefert wiederverwendbare Bausteine, mit denen Claude Code Reachy-Mini-Projekte mit weniger Reibung umsetzt. Konkretes Anwendungsziel: ein Reachy Mini, der zur Musik tanzt und über Home Assistant gesteuert werden kann (bidirektional — HA triggert Bewegungen, Reachy ruft HA-Services auf).

## Inhalt

- **Skills** — Wissens- und Workflow-Bausteine für SDK-Nutzung (`reachy-mini-sdk`), App-Scaffolding (`app-scaffold`), App-Start im Daemon (`reachy-mini-start`), Tanz-Choreographie (`dance-choreography`) und Home-Assistant-Anbindung (`home-assistant-bridge`).
- **Agents** — größere, eigenständige Aufgaben mit eigener Schale: Deploy einer App auf das Gerät (`reachy-mini-deploy`) und Live-Test einer Behavior unter beobachteter Telemetrie (`reachy-mini-on-device`).
- **Spezifikationen** — die Quelle der Wahrheit hinter jedem Skill und Agent, plus die Reachy-Mini-Wissens-Specs (Steuerungs-Oberfläche, App-Architektur, App-Logging, Entwicklungs-Workflow, Host-Provisioning, Motion-Catalog).

## Status

Aktive Entwicklung. Spec-Layer und Skill-/Agent-Inventar sind etabliert; die ersten Konsumenten-Apps (zum Beispiel `reachy-mini-app`) bauen darauf auf. Die Validierung gegen reale Hardware steht noch aus — bis dahin laufen Tests gegen `ReachyMini(spawn_daemon=True, use_sim=True)`, und der `reachy-mini-on-device`-Agent verwaltet die spätere On-Hardware-Verifikation.
