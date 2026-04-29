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
| [`reachy-mini/ha-integration`](reachy-mini/ha-integration/de.md) | Home-Assistant-Integration: Architektur | Home Assistant Integration: Architecture | draft | unversioned |
| [`reachy-mini/motions/agreeing-nod`](reachy-mini/motions/agreeing-nod/de.md) | Bewegungsablauf: Zustimmen / Nicken | Motion Sequence: Agreeing / Nod | draft | unversioned |
| [`reachy-mini/motions/alarm`](reachy-mini/motions/alarm/de.md) | Bewegungsablauf: Alarm | Motion Sequence: Alarm | draft | unversioned |
| [`reachy-mini/motions/alert-listening`](reachy-mini/motions/alert-listening/de.md) | Bewegungsablauf: Aufmerksam Hörend | Motion Sequence: Alert Listening | draft | unversioned |
| [`reachy-mini/motions/angry`](reachy-mini/motions/angry/de.md) | Bewegungsablauf: Wütend | Motion Sequence: Angry | draft | unversioned |
| [`reachy-mini/motions/bow`](reachy-mini/motions/bow/de.md) | Bewegungsablauf: Verbeugung | Motion Sequence: Bow | draft | unversioned |
| [`reachy-mini/motions/confused`](reachy-mini/motions/confused/de.md) | Bewegungsablauf: Verwirrt | Motion Sequence: Confused | draft | unversioned |
| [`reachy-mini/motions/curious`](reachy-mini/motions/curious/de.md) | Bewegungsablauf: Neugierig | Motion Sequence: Curious | draft | unversioned |
| [`reachy-mini/motions/disagreeing-shake`](reachy-mini/motions/disagreeing-shake/de.md) | Bewegungsablauf: Ablehnen / Kopfschütteln | Motion Sequence: Disagreeing / Head Shake | draft | unversioned |
| [`reachy-mini/motions/disappointed`](reachy-mini/motions/disappointed/de.md) | Bewegungsablauf: Enttäuscht | Motion Sequence: Disappointed | draft | unversioned |
| [`reachy-mini/motions/disgust`](reachy-mini/motions/disgust/de.md) | Bewegungsablauf: Ekel | Motion Sequence: Disgust | draft | unversioned |
| [`reachy-mini/motions/excited`](reachy-mini/motions/excited/de.md) | Bewegungsablauf: Aufgeregt | Motion Sequence: Excited | draft | unversioned |
| [`reachy-mini/motions/farewell-wave`](reachy-mini/motions/farewell-wave/de.md) | Bewegungsablauf: Abschieds-Welle | Motion Sequence: Farewell Wave | draft | unversioned |
| [`reachy-mini/motions/flinch`](reachy-mini/motions/flinch/de.md) | Bewegungsablauf: Zurückzucken | Motion Sequence: Flinch | draft | unversioned |
| [`reachy-mini/motions/greeting-wave`](reachy-mini/motions/greeting-wave/de.md) | Bewegungsablauf: Begrüßungs-Welle | Motion Sequence: Greeting Wave | draft | unversioned |
| [`reachy-mini/motions/groove-bob`](reachy-mini/motions/groove-bob/de.md) | Bewegungsablauf: Groove-Bob | Motion Sequence: Groove Bob | draft | unversioned |
| [`reachy-mini/motions/happy`](reachy-mini/motions/happy/de.md) | Bewegungsablauf: Glücklich | Motion Sequence: Happy | draft | unversioned |
| [`reachy-mini/motions/headbang-soft`](reachy-mini/motions/headbang-soft/de.md) | Bewegungsablauf: Sanftes Headbangen | Motion Sequence: Soft Headbang | draft | unversioned |
| [`reachy-mini/motions/peek`](reachy-mini/motions/peek/de.md) | Bewegungsablauf: Hervorlugen | Motion Sequence: Peek | draft | unversioned |
| [`reachy-mini/motions/proud`](reachy-mini/motions/proud/de.md) | Bewegungsablauf: Stolz | Motion Sequence: Proud | draft | unversioned |
| [`reachy-mini/motions/recognition`](reachy-mini/motions/recognition/de.md) | Bewegungsablauf: Aha-Erkenntnis | Motion Sequence: Recognition / "Aha!" | draft | unversioned |
| [`reachy-mini/motions/sad`](reachy-mini/motions/sad/de.md) | Bewegungsablauf: Traurig | Motion Sequence: Sad | draft | unversioned |
| [`reachy-mini/motions/scanning`](reachy-mini/motions/scanning/de.md) | Bewegungsablauf: Raum scannen | Motion Sequence: Scanning the Room | draft | unversioned |
| [`reachy-mini/motions/shy`](reachy-mini/motions/shy/de.md) | Bewegungsablauf: Schüchtern | Motion Sequence: Shy | draft | unversioned |
| [`reachy-mini/motions/sleepy`](reachy-mini/motions/sleepy/de.md) | Bewegungsablauf: Schläfrig | Motion Sequence: Sleepy | draft | unversioned |
| [`reachy-mini/motions/spin-look-around`](reachy-mini/motions/spin-look-around/de.md) | Bewegungsablauf: Pseudo-Rundumblick | Motion Sequence: Pseudo Look-Around Spin | draft | unversioned |
| [`reachy-mini/motions/surprised`](reachy-mini/motions/surprised/de.md) | Bewegungsablauf: Überrascht | Motion Sequence: Surprised | draft | unversioned |
| [`reachy-mini/motions/sway-side`](reachy-mini/motions/sway-side/de.md) | Bewegungsablauf: Seitliches Wiegen | Motion Sequence: Side Sway | draft | unversioned |
| [`reachy-mini/motions/thinking`](reachy-mini/motions/thinking/de.md) | Bewegungsablauf: Nachdenken | Motion Sequence: Thinking | draft | unversioned |
| [`reachy-mini/motions/waiting-idle`](reachy-mini/motions/waiting-idle/de.md) | Bewegungsablauf: Warten | Motion Sequence: Waiting Idle | draft | unversioned |

## Konventionen

- Slugs sind ASCII-kebab-case, abgeleitet aus dem kanonischen DE-Titel.
- Jede Spec lebt in genau einem Ordner mit einer Datei pro konfigurierter Sprache.
- Strukturelle Drift zwischen DE und EN wird per `nolte-shared:spec`-Skill (Operation `drift-check`) gefangen.
- RFC-2119-Schlüsselworte stehen in der DE-Fassung als `MUSS [MUST]`, `SOLLTE [SHOULD]`, `KANN [MAY]` und in der EN-Fassung als `MUST`, `SHOULD`, `MAY`.
