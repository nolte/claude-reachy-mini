# Spezifikationen — `claude-reachy-mini`

Quelle der Wahrheit hinter den Skills und Agents dieses Plugins. Specs sind zweisprachig: Deutsch ist kanonisch (`de.md`), Englisch ist Übersetzung (`en.md`). Konfiguration siehe `.spec-config.yml`.

## Index

| Slug | Titel (DE) | Titel (EN) | Status | Zuletzt aktualisiert |
|---|---|---|---|---|
| [`claude/behavior-scaffold`](claude/behavior-scaffold/de.md) | Behavior-Scaffold-Skill | Behavior Scaffold Skill | draft | unversioned |
| [`claude/home-assistant-bridge`](claude/home-assistant-bridge/de.md) | Home-Assistant-Bridge-Skill | Home Assistant Bridge Skill | draft | unversioned |
| [`claude/reachy-mini-on-device`](claude/reachy-mini-on-device/de.md) | On-Device-Test-Agent für Reachy Mini | On-Device Test Agent for Reachy Mini | draft | unversioned |
| [`claude/reachy-mini-sdk`](claude/reachy-mini-sdk/de.md) | Reachy-Mini-SDK-Skill | Reachy Mini SDK Skill | draft | unversioned |
| [`reachy-mini/control-surface`](reachy-mini/control-surface/de.md) | Steuerungs-Oberfläche und Bewegungs-Design des Reachy Mini | Reachy Mini Control Surface and Motion Design | draft | unversioned |

## Konventionen

- Slugs sind ASCII-kebab-case, abgeleitet aus dem kanonischen DE-Titel.
- Jede Spec lebt in genau einem Ordner mit einer Datei pro konfigurierter Sprache.
- Strukturelle Drift zwischen DE und EN wird per `nolte-shared:spec`-Skill (Operation `drift-check`) gefangen.
- RFC-2119-Schlüsselworte stehen in der DE-Fassung als `MUSS [MUST]`, `SOLLTE [SHOULD]`, `KANN [MAY]` und in der EN-Fassung als `MUST`, `SHOULD`, `MAY`.
