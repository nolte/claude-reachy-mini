# Zielgruppen — `claude-reachy-mini`-Plugin (dieses Repository)

<!--
Erzeugt mit dem `audience-identify`-Skill, gemäß
spec/project/audience-identification/ (im nolte-shared-Plugin).
Keine Zielgruppen ergänzen, ohne zuerst den Bounded Context unten anzupassen.
-->

## Bounded Context

**Was dieser Kontext *ist***:

- Das Repository `nolte/claude-reachy-mini`, veröffentlicht als Claude-Code-Plugin `claude-reachy-mini` (Version 0.1.0) über den Plugin-Marketplace.
- Es bündelt wiederverwendbare **Skills** (`skills/<name>/SKILL.md`), **Agents** (`agents/<name>.md`) und **Specs** (`spec/`, DE-kanonisch, später ggf. mit EN-Übersetzungen) für die Entwicklung mit dem [Reachy Mini](https://www.pollen-robotics.com/reachy-mini/) (Pollen Robotics / Hugging Face).
- Enthält außerdem das MkDocs-Doku-Setup (`docs/`, derzeit DE-only) und die Taskfile-basierte Automatisierung.
- Erste konkrete Anwendungs-Domäne der Skills: Reachy tanzt zur Musik, bidirektionale Kopplung mit Home Assistant.

**Wo die Grenzen verlaufen**:

- Externe Oberflächen: das Plugin-Manifest + der Marketplace-Eintrag (Install-Pfad), die Slash-Commands (z. B. `/claude-reachy-mini:reachy-mini-sdk`), die Agent-Definitionen sowie die veröffentlichte MkDocs-Seite.
- Das Repo selbst, die Branches `develop`/`main` und die CI-Workflows gehören zum Kontext.

**Was explizit *außerhalb* liegt**:

- Die konkrete Tanz-/HA-App, die mit Hilfe dieser Skills gebaut wird — sie lebt in einem separaten Repository und ist nur Konsument der Plugin-Outputs.
- Die Reachy-Mini-Hardware sowie das `reachy_mini`-SDK von Pollen Robotics — Abhängigkeiten und Wissensgrundlage, aber nicht Teil dieses Kontexts.
- Home Assistant selbst — externes System, mit dem die generierten Apps reden, das aber nicht hier verändert wird.
- Claude Code selbst (CLI/IDE-Integration) — das Plugin baut darauf auf, betreibt es aber nicht.
- Inhaltliche Themen einzelner Skills — jeder Skill hat seinen eigenen, schmaleren Kontext, falls separat auditiert.

## Zielgruppen

Jeder Eintrag: Label, Beziehungs-Kategorie, Interaktions-Oberfläche, Erwartung, offene Fragen, `confirmed` oder `assumed`, Kritikalität (primary / secondary / peripheral). Eine ganze Kategorie wird mit `none — <Grund>` markiert, wenn sie nicht zutrifft.

### Direkte Konsumenten

- **Plugin-Autor beim Dogfooding in diesem Repo (nolte)** — _Kategorie_: direct-consumer · _Oberfläche_: `claude --plugin-dir .`, `/reload-plugins`, lokales Skill-Invokation während der Entwicklung an Skills/Agents/Specs · _erwartet_: Änderungen an Skills/Agents/Specs werden ohne Reinstall sofort aufrufbar; Skills funktionieren auch gegen dieses Repo selbst (z. B. `/nolte-shared:project-structure-apply`) · _Status_: `assumed` · _Kritikalität_: primary
  - Offene Fragen: keine

- **Plugin-Autor beim Bau der konkreten Tanz-/HA-App (nolte)** — _Kategorie_: direct-consumer · _Oberfläche_: Slash-Commands wie `/claude-reachy-mini:reachy-mini-sdk`, `/claude-reachy-mini:app-scaffold`, `/claude-reachy-mini:home-assistant-bridge` und der Agent `reachy-mini-on-device`, eingesetzt im separaten App-Repo · _erwartet_: aktuelle, zur SDK-Version passende Wissensbausteine; reproduzierbare App-Scaffolds; HA-API-Patterns die out-of-the-box funktionieren · _Status_: `assumed` · _Kritikalität_: primary
  - Offene Fragen: An welche `reachy_mini`-Version pinnen wir die SDK-Skills? Wie reagieren wir auf Breaking-Changes upstream?

- **Spätere öffentliche Nutzer (Reachy-Mini-Hobby- und Maker-Community)** — _Kategorie_: direct-consumer · _Oberfläche_: Plugin-Installation aus dem Marketplace, dieselben Slash-Commands wie der Autor · _erwartet_: belastbare Skills auch ohne Insider-Wissen über das nolte-Portfolio; klar dokumentiert was das Plugin _nicht_ leistet (z. B. keine fertige Tanz-App); Versionierung kommuniziert Breaking-Changes · _Status_: `assumed` · _Kritikalität_: peripheral (heute hypothetisch — erst relevant, wenn das Plugin geteilt wird)
  - Offene Fragen: Wann wird das Plugin tatsächlich veröffentlicht? Welche Mindest-Reife (Anzahl Skills, Test-Coverage, Doku) muss vorher erreicht sein?

### Betreiber

- **GitHub Actions CI für dieses Repo** — _Kategorie_: operator · _Oberfläche_: Workflows unter `.github/workflows/` (insbesondere `ci.yml` mit `lint`/`test`/`docs` als Required-Checks auf `develop`), plus Release-/Automerge-/`main`-Fast-Forward-Infrastruktur über `nolte/gh-plumbing` · _erwartet_: reproduzierbare Läufe; stabile Task-Targets (`task lint`/`test`/`docs`); keine flaky Checks, die `develop` blockieren · _Status_: `assumed` · _Kritikalität_: primary
  - Offene Fragen: keine

- **GitHub Pages als Hosting der Plugin-Doku** — _Kategorie_: operator · _Oberfläche_: `release-cd-deliver-docs.yml`, MkDocs-Site veröffentlicht unter `nolte.github.io/claude-reachy-mini` · _erwartet_: jeder Release liefert eine reproduzierbare Doku aus; Generator-Hook (`scripts/docs/gen_catalog.py`) bricht bei kaputter Frontmatter ab statt stillen Lücken · _Status_: `assumed` · _Kritikalität_: secondary
  - Offene Fragen: keine

### Beitragende / Maintainer

- **Repo-Maintainer (nolte)** — _Kategorie_: contributor · _Oberfläche_: direkter Commit-Zugriff auf alle Branches, Review-Autorität, Release-Autorität, Spec-Evolutions-Autorität · _erwartet_: Specs, Skills und Plugin-Manifest bleiben konsistent; `CLAUDE.md` reflektiert den Repo-Stand; Konventionen (DE-kanonische Specs, Conventional Commits, PR-Workflow) werden eingehalten · _Status_: `assumed` · _Kritikalität_: primary
  - Offene Fragen: keine

- **Claude Code als Co-Autor** — _Kategorie_: contributor · _Oberfläche_: Skills aus `nolte-shared` (`/nolte-shared:skill-management`, `/nolte-shared:spec`, `/nolte-shared:pull-request-create`) — Claude scaffolded und editiert Files unter `skills/`, `agents/`, `spec/` und erzeugt Commits/PRs · _erwartet_: Skills folgen ihren eigenen Specs (Meta-Konsistenz); Änderungen bleiben review-fähig; Hard Rules werden respektiert (z. B. keine Plugin-Skills nach `.claude/skills/` kopieren) · _Status_: `assumed` · _Kritikalität_: primary
  - Offene Fragen: keine

- **Externe Contributors via Pull Request** — _Kategorie_: contributor · _Oberfläche_: GitHub-Forks, PRs gegen `develop`, Issue-Tracker · _erwartet_: klare Einstiegspunkte (README, `CLAUDE.md`, Spec-Layout); PR-Workflow via `/nolte-shared:pull-request-create` ist ohne Insider-Wissen befolgbar; Skill-vs-Plugin-Architektur ist nachvollziehbar dokumentiert · _Status_: `assumed` · _Kritikalität_: peripheral (heute kein offener Beitragspfad)
  - Offene Fragen: Soll dieses Repo aktiv für externe Beiträge geöffnet werden? Aktuell existiert keine `CONTRIBUTING.md`. Ab welchem Reifegrad ist das sinnvoll?

### Steuernde Parteien

- **Portfolio-Konsistenz-Anker (`nolte/gh-plumbing`, `nolte/taskfiles`, `nolte/claude-shared` als Spec-Quelle)** — _Kategorie_: governing-party · _Oberfläche_: `_extends`-Pointer in `.github/settings.yml` / `release-drafter.yml` / `boring-cyborg.yml` / `stale.yml`, gepinnter `gh-plumbing`-Tag, `TASK_COLLECTION_BASE`-Referenzen, Specs aus `nolte-shared` · _erwartet_: dieses Repo divergiert nicht von den Portfolio-Standards; Upstream-Änderungen werden nachgezogen (z. B. neue gh-plumbing-Releases via Renovate, Spec-Drift via `project-structure-apply`) · _Status_: `assumed` · _Kritikalität_: secondary
  - Offene Fragen: keine

- **Pollen Robotics / Hugging Face als SDK- und Hardware-Stewards** — _Kategorie_: governing-party · _Oberfläche_: `reachy_mini`-Python-SDK, Reachy-Mini-Firmware, Hugging-Face-Spaces-Konventionen für Behaviors · _erwartet_: dass Wissens-Skills die offiziellen API-Konventionen widerspiegeln und nicht eine private Fork-Realität dokumentieren; dass Skills explizit benennen, gegen welche SDK-Version sie verifiziert wurden · _Status_: `assumed` · _Kritikalität_: secondary
  - Offene Fragen: Hat Pollen Robotics eine offizielle Position zu Drittanbieter-Tooling oder Plugins, die ihre API beschreiben? Gibt es eine offizielle Version-Compatibility-Matrix, an die wir uns halten sollten?

- **Home-Assistant-Projekt als API-Steward** — _Kategorie_: governing-party · _Oberfläche_: HA REST-API / WebSocket-API, Long-Lived-Access-Token-Modell, Service-Definitionen · _erwartet_: dass HA-Bridge-Skills die offiziellen API-Verträge spiegeln; dass Auth-Patterns nicht riskante Workarounds dokumentieren · _Status_: `assumed` · _Kritikalität_: secondary
  - Offene Fragen: Welche minimal unterstützte HA-Version wollen wir setzen?

### Indirekte Zielgruppen

- **End-User der späteren Tanz-/HA-App (Bewohner des Haushalts mit Reachy Mini auf dem Schreibtisch)** — _Kategorie_: indirect · _Oberfläche_: keine direkt — sie sehen nur den fertigen Roboter tanzen oder lösen über HA-Sprachbefehle Aktionen aus. Einfluss läuft mediated, weil die Skills die Qualität der App formen, die diese End-User dann erleben · _erwartet_: nichts direkt von diesem Plugin. Das Plugin übernimmt explizit keine Verantwortung für End-User-Outcomes; Skills sind Tooling, keine Garantien — Verantwortlich ist die App, die mit ihnen gebaut wird · _Status_: `assumed` · _Kritikalität_: peripheral
  - Offene Fragen: keine

- **Andere Reachy-Mini-Plugins / Portfolio-Plugins, die `claude-reachy-mini` als Vorbild für hardware-/SDK-spezifische Plugins betrachten** — _Kategorie_: indirect · _Oberfläche_: keine direkt — sie installieren das Plugin nicht, schauen aber auf die hier kodifizierten Patterns (wie ein domänenspezifisches Plugin gegen das nolte-shared-Skelett aussieht) · _erwartet_: dass das Plugin als sauberes Beispiel taugt — also Specs, Skill-Layout und Konventionen reproduzierbar sind · _Status_: `assumed` · _Kritikalität_: peripheral
  - Offene Fragen: keine

## Offene Fragen (übergreifend)

- Keine Zielgruppe ist heute `confirmed` — keine wurde gegen einen echten Repräsentanten oder eine autoritative Quelle validiert. Alle Einträge bleiben `assumed`, bis eine solche Validierung passiert (z. B. erste tatsächliche Nutzung des Plugins gegen die echte Hardware, erste öffentliche Veröffentlichung).
- Die SDK- und HA-Versions-Politik ist ungeklärt: gegen welche `reachy_mini`- und HA-Versionen pinnen wir Skills, und wie kommunizieren wir Breaking-Changes upstream an Plugin-Nutzer?
- Eine Veröffentlichungs-Schwelle ist nicht definiert: Ab welchem Skill-/Reifegrad gibt das Plugin den Nicht-Owner-Konsumenten überhaupt einen Mehrwert, der eine Marketplace-Listung rechtfertigt?

## Anlässe für Re-Identifikation

- Hardware ist eingetroffen und das erste Behavior wurde tatsächlich auf dem Gerät ausgeführt — das hebt Status mehrerer `assumed`-Einträge potenziell auf `confirmed`.
- Das Plugin wird erstmals öffentlich gelistet — dann werden „Spätere öffentliche Nutzer" und „Externe Contributors" von peripheral zu mindestens secondary.
- Eine zweite konkrete Reachy-Mini-App entsteht (über Tanz/HA hinaus) — das verändert die direct-consumer-Landschaft.
- Pollen Robotics oder Hugging Face veröffentlichen Breaking-Changes am SDK oder verschieben das Plugin-Modell für Behaviors — die governing-party-Erwartungen ändern sich.
- Das `nolte-shared`-Plugin verschärft die Spec für `audience-identification` (z. B. neue Pflicht-Kategorien) — dann muss diese Datei nachgezogen werden.
- Ein zweiter Reachy-orientierter Skill-/Agent-Katalog entsteht in einem anderen Repo, der dieses hier ablöst oder mit ihm konkurriert — Anlass, die indirect-Kategorie ernsthaft zu prüfen.
