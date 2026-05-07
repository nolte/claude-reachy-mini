# Spezifikationen

Quelle der Wahrheit hinter Skills und Agents. Specs liegen unter [`spec/`](https://github.com/nolte/claude-reachy-mini/tree/develop/spec) im Repository, jeweils zweisprachig (DE kanonisch, EN Übersetzung). Die kanonische Spec-Konvention ist in [`spec/.spec-config.yml`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/.spec-config.yml) deklariert.

Eine vollständige, automatisch gepflegte Tabelle aller Specs mit Status und Datum lebt in [`spec/README.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/README.md).

## Spec-Gruppen

### `spec/claude/` — Skill- und Agent-Specs

Eine Spec pro Skill / Agent, beschreibt Trigger, Eingabe-Parameter, Pre-Flight-Pflichten, Hard rules und Akzeptanzkriterien. Heutige Pairs:

- [`app-scaffold`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-scaffold/de.md) — neue Reachy-Mini-App über `reachy-mini-app-assistant create` scaffolden
- [`app-log-triage`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/app-log-triage/de.md) — Failure-Klassifikation aus Log-Output (Skill geplant)
- [`dance-choreography`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/dance-choreography/de.md) — Tanz-Sektions-Tabelle mit BPM und Slug-Verweisen erzeugen
- [`home-assistant-bridge`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/home-assistant-bridge/de.md) — bidirektionale HA-Integration
- [`reachy-mini-deploy`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-deploy/de.md) — Deploy-Agent für eine fertige App
- [`reachy-mini-on-device`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-on-device/de.md) — On-Device-Test-Agent
- [`reachy-mini-sdk`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-sdk/de.md) — SDK-Wissensbasis
- [`reachy-mini-start`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-start/de.md) — App im Daemon starten

### `spec/reachy-mini/` — Wissens- und Architektur-Specs

Plattform- und SDK-übergreifendes Wissen, gegen das Skills und Agents arbeiten:

- [`app-architecture`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-architecture/de.md) — App-Layout, Daemon-Lifecycle, Distributions-Pfad
- [`app-development-workflow`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-development-workflow/de.md) — Methodik-Spec mit zehn Phasen, Plan-First-Gate und Security-Review-Gate
- [`app-logging`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/app-logging/de.md) — Logging-Topologie, Common-Issues-Triage-Katalog, Verify-Basics-First-Heuristik
- [`control-surface`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/control-surface/de.md) — was am Reachy Mini steuerbar ist und unter welchen Grenzen
- [`ha-integration`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/ha-integration/de.md) — Architektur der Home-Assistant-Integration
- [`host-provisioning`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/host-provisioning/de.md) — WiFi-Onboarding, FS-Layout, App-Distribution via Ansible / Pull-Service

### `spec/reachy-mini/motions/` — Motion-Catalog

Eine Spec pro Bewegungs-Slug (Tanz, Emotion, soziale Geste, State, Defensive). Slugs werden von `dance-choreography` und Move-Subklassen-Implementierungen konsumiert.

## Konventionen

- **Slugs** sind ASCII-kebab-case, abgeleitet aus dem kanonischen DE-Titel.
- **Strukturelle Drift zwischen DE und EN** wird per `nolte-shared:spec`-Skill (Operation `drift-check`) gefangen.
- **RFC-2119-Schlüsselworte**: DE-Fassung als `MUSS [MUST]`, `SOLLTE [SHOULD]`, `KANN [MAY]` und `DARF NICHT [MUST NOT]`; EN-Fassung als `MUST`, `SHOULD`, `MAY`, `MUST NOT`.
- **Quell-Verweise**: Code-Verweise auf `pollen-robotics/reachy_mini` zeigen auf konkrete Datei + Zeile; Markdown-Verweise auf Datei-Ebene.
