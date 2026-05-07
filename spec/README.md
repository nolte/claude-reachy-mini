# Spezifikationen — `claude-reachy-mini`

Quelle der Wahrheit hinter den Skills und Agents dieses Plugins. Specs sind zweisprachig: Deutsch ist kanonisch (`de.md`), Englisch ist Übersetzung (`en.md`). Konfiguration siehe `.spec-config.yml`.

## Index

| Slug | Titel (DE) | Titel (EN) | Status | Zuletzt aktualisiert |
|---|---|---|---|---|
| [`claude/app-log-triage`](claude/app-log-triage/de.md) | App-Log-Triage-Skill | App Log Triage Skill | draft | unversioned |
| [`claude/app-scaffold`](claude/app-scaffold/de.md) | App-Scaffold-Skill | App Scaffold Skill | draft | 2026-05-06 |
| [`claude/audio-beat-tracking`](claude/audio-beat-tracking/de.md) | Audio-Beat-Tracking-Skill | Audio Beat Tracking Skill | draft | unversioned |
| [`claude/dance-choreography`](claude/dance-choreography/de.md) | Dance-Choreography-Skill | Dance Choreography Skill | draft | 2026-05-06 |
| [`claude/home-assistant-bridge`](claude/home-assistant-bridge/de.md) | Home-Assistant-Bridge-Skill | Home Assistant Bridge Skill | draft | 2026-05-05 |
| [`claude/reachy-mini-deploy`](claude/reachy-mini-deploy/de.md) | Deploy-Agent für Reachy-Mini-Apps | Deploy Agent for Reachy Mini Apps | draft | 2026-05-06 |
| [`claude/reachy-mini-on-device`](claude/reachy-mini-on-device/de.md) | On-Device-Test-Agent für Reachy Mini | On-Device Test Agent for Reachy Mini | draft | 2026-05-06 |
| [`claude/reachy-mini-sdk`](claude/reachy-mini-sdk/de.md) | Reachy-Mini-SDK-Skill | Reachy Mini SDK Skill | draft | 2026-05-06 |
| [`claude/reachy-mini-start`](claude/reachy-mini-start/de.md) | Start-Skill für Reachy-Mini-Apps | Start Skill for Reachy Mini Apps | draft | 2026-05-06 |
| [`reachy-mini/app-architecture`](reachy-mini/app-architecture/de.md) | App-Architektur: Reachy-Mini-Show | App Architecture: Reachy Mini Show | draft | 2026-05-05 |
| [`reachy-mini/app-development-workflow`](reachy-mini/app-development-workflow/de.md) | Entwicklungs-Workflow für Reachy-Mini-Apps | Development Workflow for Reachy Mini Apps | draft | unversioned |
| [`reachy-mini/app-logging`](reachy-mini/app-logging/de.md) | Logging und Fehleranalyse während der App-Entwicklung | Logging and Failure Analysis During App Development | draft | unversioned |
| [`reachy-mini/control-surface`](reachy-mini/control-surface/de.md) | Steuerungs-Oberfläche und Bewegungs-Design des Reachy Mini | Reachy Mini Control Surface and Motion Design | draft | 2026-05-05 |
| [`reachy-mini/ha-integration`](reachy-mini/ha-integration/de.md) | Home-Assistant-Integration: Architektur | Home Assistant Integration: Architecture | draft | 2026-05-05 |
| [`reachy-mini/host-provisioning`](reachy-mini/host-provisioning/de.md) | Host-Provisioning: WiFi, Filesystem-Layout und App-Distribution | Host Provisioning: WiFi, Filesystem Layout, and App Distribution | draft | unversioned |
| [`reachy-mini/motions/agreeing-nod`](reachy-mini/motions/agreeing-nod/de.md) | Bewegungsablauf: Zustimmen / Nicken (`agreeing-nod`) | Motion Sequence: Agreeing / Nod (`agreeing-nod`) | draft | 2026-05-05 |
| [`reachy-mini/motions/alarm`](reachy-mini/motions/alarm/de.md) | Bewegungsablauf: Alarm (`alarm`) | Motion Sequence: Alarm (`alarm`) | draft | 2026-05-05 |
| [`reachy-mini/motions/alert-listening`](reachy-mini/motions/alert-listening/de.md) | Bewegungsablauf: Aufmerksam Hörend (`alert-listening`) | Motion Sequence: Alert Listening (`alert-listening`) | draft | 2026-05-05 |
| [`reachy-mini/motions/angry`](reachy-mini/motions/angry/de.md) | Bewegungsablauf: Wütend (`angry`) | Motion Sequence: Angry (`angry`) | draft | 2026-05-05 |
| [`reachy-mini/motions/bow`](reachy-mini/motions/bow/de.md) | Bewegungsablauf: Verbeugung (`bow`) | Motion Sequence: Bow (`bow`) | draft | 2026-05-05 |
| [`reachy-mini/motions/confused`](reachy-mini/motions/confused/de.md) | Bewegungsablauf: Verwirrt (`confused`) | Motion Sequence: Confused (`confused`) | draft | 2026-05-05 |
| [`reachy-mini/motions/curious`](reachy-mini/motions/curious/de.md) | Bewegungsablauf: Neugierig (`curious`) | Motion Sequence: Curious (`curious`) | draft | 2026-05-05 |
| [`reachy-mini/motions/disagreeing-shake`](reachy-mini/motions/disagreeing-shake/de.md) | Bewegungsablauf: Ablehnen / Kopfschütteln (`disagreeing-shake`) | Motion Sequence: Disagreeing / Head Shake (`disagreeing-shake`) | draft | 2026-05-05 |
| [`reachy-mini/motions/disappointed`](reachy-mini/motions/disappointed/de.md) | Bewegungsablauf: Enttäuscht (`disappointed`) | Motion Sequence: Disappointed (`disappointed`) | draft | 2026-05-05 |
| [`reachy-mini/motions/disgust`](reachy-mini/motions/disgust/de.md) | Bewegungsablauf: Ekel (`disgust`) | Motion Sequence: Disgust (`disgust`) | draft | 2026-05-05 |
| [`reachy-mini/motions/excited`](reachy-mini/motions/excited/de.md) | Bewegungsablauf: Aufgeregt (`excited`) | Motion Sequence: Excited (`excited`) | draft | 2026-05-05 |
| [`reachy-mini/motions/farewell-wave`](reachy-mini/motions/farewell-wave/de.md) | Bewegungsablauf: Abschieds-Welle (`farewell-wave`) | Motion Sequence: Farewell Wave (`farewell-wave`) | draft | 2026-05-05 |
| [`reachy-mini/motions/flinch`](reachy-mini/motions/flinch/de.md) | Bewegungsablauf: Zurückzucken (`flinch`) | Motion Sequence: Flinch (`flinch`) | draft | 2026-05-05 |
| [`reachy-mini/motions/greeting-wave`](reachy-mini/motions/greeting-wave/de.md) | Bewegungsablauf: Begrüßungs-Welle (`greeting-wave`) | Motion Sequence: Greeting Wave (`greeting-wave`) | draft | 2026-05-05 |
| [`reachy-mini/motions/groove-bob`](reachy-mini/motions/groove-bob/de.md) | Bewegungsablauf: Groove-Bob (`groove-bob`) | Motion Sequence: Groove Bob (`groove-bob`) | draft | 2026-05-05 |
| [`reachy-mini/motions/happy`](reachy-mini/motions/happy/de.md) | Bewegungsablauf: Glücklich (`happy`) | Motion Sequence: Happy (`happy`) | draft | 2026-05-05 |
| [`reachy-mini/motions/headbang-soft`](reachy-mini/motions/headbang-soft/de.md) | Bewegungsablauf: Sanftes Headbangen (`headbang-soft`) | Motion Sequence: Soft Headbang (`headbang-soft`) | draft | 2026-05-05 |
| [`reachy-mini/motions/peek`](reachy-mini/motions/peek/de.md) | Bewegungsablauf: Hervorlugen (`peek`) | Motion Sequence: Peek (`peek`) | draft | 2026-05-05 |
| [`reachy-mini/motions/proud`](reachy-mini/motions/proud/de.md) | Bewegungsablauf: Stolz (`proud`) | Motion Sequence: Proud (`proud`) | draft | 2026-05-05 |
| [`reachy-mini/motions/recognition`](reachy-mini/motions/recognition/de.md) | Bewegungsablauf: Aha-Erkenntnis (`recognition`) | Motion Sequence: Recognition / "Aha!" (`recognition`) | draft | 2026-05-05 |
| [`reachy-mini/motions/sad`](reachy-mini/motions/sad/de.md) | Bewegungsablauf: Traurig (`sad`) | Motion Sequence: Sad (`sad`) | draft | 2026-05-05 |
| [`reachy-mini/motions/scanning`](reachy-mini/motions/scanning/de.md) | Bewegungsablauf: Raum scannen (`scanning`) | Motion Sequence: Scanning the Room (`scanning`) | draft | 2026-05-05 |
| [`reachy-mini/motions/shy`](reachy-mini/motions/shy/de.md) | Bewegungsablauf: Schüchtern (`shy`) | Motion Sequence: Shy (`shy`) | draft | 2026-05-05 |
| [`reachy-mini/motions/sleepy`](reachy-mini/motions/sleepy/de.md) | Bewegungsablauf: Schläfrig (`sleepy`) | Motion Sequence: Sleepy (`sleepy`) | draft | 2026-05-05 |
| [`reachy-mini/motions/spin-look-around`](reachy-mini/motions/spin-look-around/de.md) | Bewegungsablauf: Pseudo-Rundumblick (`spin-look-around`) | Motion Sequence: Pseudo Look-Around Spin (`spin-look-around`) | draft | 2026-05-05 |
| [`reachy-mini/motions/surprised`](reachy-mini/motions/surprised/de.md) | Bewegungsablauf: Überrascht (`surprised`) | Motion Sequence: Surprised (`surprised`) | draft | 2026-05-05 |
| [`reachy-mini/motions/sway-side`](reachy-mini/motions/sway-side/de.md) | Bewegungsablauf: Seitliches Wiegen (`sway-side`) | Motion Sequence: Side Sway (`sway-side`) | draft | 2026-05-05 |
| [`reachy-mini/motions/thinking`](reachy-mini/motions/thinking/de.md) | Bewegungsablauf: Nachdenken (`thinking`) | Motion Sequence: Thinking (`thinking`) | draft | 2026-05-05 |
| [`reachy-mini/motions/waiting-idle`](reachy-mini/motions/waiting-idle/de.md) | Bewegungsablauf: Warten (`waiting-idle`) | Motion Sequence: Waiting Idle (`waiting-idle`) | draft | 2026-05-05 |

## Konventionen

- Slugs sind ASCII-kebab-case, abgeleitet aus dem kanonischen DE-Titel.
- Jede Spec lebt in genau einem Ordner mit einer Datei pro konfigurierter Sprache.
- Strukturelle Drift zwischen DE und EN wird per `nolte-shared:spec`-Skill (Operation `drift-check`) gefangen.
- RFC-2119-Schlüsselworte stehen in der DE-Fassung als `MUSS [MUST]`, `SOLLTE [SHOULD]`, `KANN [MAY]` und in der EN-Fassung als `MUST`, `SHOULD`, `MAY`.
