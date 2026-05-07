# Reachy-App-Publish-HF-Skill

Status: draft

## Kontext

Eine fertige Reachy-Mini-App auf Hugging Face Spaces zu veröffentlichen ist heute scheinbar trivial: `reachy-mini-app-assistant publish <pfad> "<commit-message>"` reicht. Genau das ist das Problem — der Befehl umgeht alle Vor-Publikations-Gates, die [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/de.md) § Phase 10 als verbindlich markiert: abgenommener Plan, gegrünter Security-Review-Audit, sauberes Repo, gepinnter SDK-Stand, korrekte Provenienz-Marker. Wer das CLI direkt aufruft, kann unbeabsichtigt eine App publishen, die einen offenen `plan.md`-Review oder einen roten Security-Review-Befund mit sich trägt — und das ist auf Hugging Face dann öffentlich.

Dieser Skill `reachy-app-publish-hf` ist die Plugin-eigene Schale um Pollens Publish-CLI: ein **schmaler Wrapper**, der vor dem Aufruf alle Pflicht-Gates durchgeht und nur dann publisht, wenn alle Gates grün sind. Er ersetzt nicht das CLI (das bleibt der ausführende Layer für Hugging-Face-Space-Erzeugung, Git-Remote-Konfiguration, Asset-Upload), er **gateet** es.

Begriffsklärung: „Publish" meint hier den Hugging-Face-Space-Upload (Code, Provenienz, README mit HF-Frontmatter), nicht die internen Plugin-Releases (`release-publish-trigger` ist `nolte-shared` und betrifft das Plugin-Repo, nicht eine konsumierte App).

## Ziele

- Eine Reachy-Mini-App wird mit einem einzigen Skill-Aufruf nach Hugging Face publisht — vorausgesetzt alle Pre-Publish-Gates aus [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/de.md) § Phase 10 sind grün
- Die Pre-Publish-Gates sind explizit dokumentiert und werden in fixer Reihenfolge durchlaufen, mit klarer Fehlermeldung pro fehlgeschlagenem Gate
- Pollens `reachy-mini-app-assistant publish` ist der einzige ausführende Layer — der Skill ruft es auf, modifiziert nicht den Hugging-Face-Push selbst
- Der Skill bleibt schmal: er konsumiert das `app-development-workflow` als Vertrag, delegiert App-Erzeugung an `app-scaffold`, Deployment auf das Gerät an den `reachy-mini-deploy`-Agent, und Live-Validierung an den `reachy-mini-on-device`-Agent

## Nicht-Ziele

- Pollens Publish-CLI ersetzen — `reachy-mini-app-assistant publish` bleibt der einzige Pfad zur tatsächlichen Hugging-Face-Veröffentlichung
- Hugging-Face-Hub-Authentifizierung selbst durchführen — `hf auth login` ist Voraussetzung, nicht Aufgabe dieses Skills
- App-Erzeugung oder Re-Scaffold (`app-scaffold` macht das)
- Deployment einer App auf den lokalen Reachy Mini (`reachy-mini-deploy`-Agent macht das)
- Live-Trial nach Publish (`reachy-mini-on-device`-Agent macht das)
- Plugin-Release der `claude-reachy-mini`-Plugin-Versionen (`nolte-shared:release-publish-trigger`)
- Rollback / Unpublish — Hugging Face hat eigene UI / API für Löschen; der Skill ist Forward-Only
- Hugging-Face-Quotas, -Tarif-Pläne, -Storage-Limits (Hugging-Face-eigene Doku)
- CI/CD-Integration in eine Pipeline — der Skill ist Single-Shot per Aufruf, nicht ein Workflow-Step (`release-automation` ist Plugin-Sache, nicht App)
- Versions-Bump des SDK-Pins, Dependency-Renovate, Changelog-Generierung — Repo-eigene Sache der App, ggf. mit Renovate / Release-Drafter

## Anforderungen

### Trigger und Aktivierung

- **MUSS [MUST]** eine `description` liefern, die Claude Code aktiviert auf Formulierungen wie „Reachy-App auf Hugging Face publishen", „App veröffentlichen", „custom HF publish workflow", „publish reachy-mini-app to Hugging Face Space", „release the app on Hugging Face"
- **MUSS [MUST]** in der `description` die Schlüsselbegriffe enthalten: publish, Hugging Face, Reachy Mini, app, release, Space
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist: bei reiner App-Erzeugung (`app-scaffold`), bei Deploy auf das eigene Gerät (`reachy-mini-deploy`-Agent), bei Plugin-Releases (`nolte-shared:release-publish-trigger`), bei Hugging-Face-Hub-Auth-Problemen (Pollen-/HF-Doku)

### Eingabe-Parameter

- **MUSS [MUST]** den App-Pfad als Pflicht-Parameter annehmen (`app_path`); der Skill veröffentlicht nicht das aktuelle Working-Directory ohne expliziten Pfad-Input
- **MUSS [MUST]** eine Commit-Message annehmen (`commit_message`), die für den Hugging-Face-Space-Commit verwendet wird; ohne Message bricht der Skill ab — keine generierte Default-Message
- **SOLLTE [SHOULD]** einen Visibility-Parameter (`visibility`: `public` | `private`) annehmen; Default ist `public` (Pollen-Konvention für die offene App-Sammlung), `private` als Opt-in für interne Apps
- **SOLLTE [SHOULD]** einen `official`-Flag annehmen (Default `false`) — `true` triggert Pollens `--official`-Flag (Anfrage zur Aufnahme als offizielle Reachy-Mini-App, Pollen-Side-Review)
- **SOLLTE [SHOULD]** optional einen `skip_gates`-Flag annehmen (Default `false`); bei `true` werden die Pre-Publish-Gates **gemeldet** und übersprungen, der CLI-Aufruf erhält dann das `--nocheck`-Flag — **muss** ausdrücklich vom User angefordert werden, niemals Default
- **DARF NICHT [MUST NOT]** der Skill stillschweigend Pflicht-Gates überspringen, ohne dass `skip_gates=true` explizit gesetzt ist — Default-Verhalten ist „alles oder nichts"

### Pre-Publish-Gates (in dieser Reihenfolge)

Der Skill **MUSS [MUST]** vor dem CLI-Aufruf diese Gates der Reihe nach prüfen und auf erstem fehlgeschlagenen Gate abbrechen:

1. **Pollen-CLI verfügbar** — `reachy-mini-app-assistant --help` läuft im PATH; fehlt das CLI, abbrechen mit Anleitung (`uv tool install reachy-mini`)
2. **App-Strukturvertrag** — `reachy-mini-app-assistant check <app_path>` läuft ohne Findings; bricht ab bei red Findings (Pollen-Konvention: keine strukturell defekte App publishen)
3. **Hugging-Face-Auth** — `hf auth whoami` liefert eine gültige Identität; ohne Login abbrechen mit Anleitung (`uv pip install --upgrade huggingface_hub && hf auth login`, Token mit **Write**-Permission)
4. **`plan.md` abgenommen** — die Datei `<app_path>/plan.md` existiert und enthält im § Approval-Gate-Block einen ausgefüllten Reviewer + Datum + SHA; bei leerem oder fehlendem Approval-Block abbrechen (Quelle: `app-development-workflow` § Phase 3)
5. **Security-Review-Bericht vorhanden** — die Datei `<app_path>/.audits/security-review/<YYYY-MM-DD>.md` existiert mit einem Eintrag, dessen Datum innerhalb der letzten 30 Tage liegt; älter oder fehlend → abbrechen (Quelle: `app-development-workflow` § Phase 7)
6. **SDK-Versions-Pin sichtbar** — `<app_path>/pyproject.toml` enthält einen konkreten `reachy-mini`-Pin (`==X.Y.Z` oder `~=X.Y`); bei `>=X.Y` ohne Upper-Bound abbrechen (sonst können Hugging-Face-Spaces eine zukünftige inkompatible SDK-Version ziehen)
7. **Provenienz-Marker vorhanden** — `<app_path>/CLAUDE.md` und `<app_path>/README.md` enthalten den Plugin-Verweis-Block, wie in [`claude/app-scaffold`](../app-scaffold/de.md) § Erzeugte Artefakte definiert; fehlt einer, abbrechen
8. **Repo sauber (kein dirty git-tree)** — `git status --porcelain` im `<app_path>` ist leer; uncommittete Änderungen → abbrechen (Konsumenten sollen wissen, was tatsächlich publisht wird, ohne Überraschungen aus uncommitteten Files)
9. **Lokale CI grün** (optional, wenn `task lint` / `pytest tests/` im App-Repo definiert) — bei roten Checks abbrechen, statt einen kaputten Stand zu publishen
10. **Branch ist `main` oder `develop`** — Feature-/Experiment-Branches sollten nicht direkt auf Hugging Face landen; bei anderen Branch-Namen warnen und Bestätigung verlangen

- **SOLLTE [SHOULD]** der Skill jedem fehlgeschlagenen Gate eine konkrete Aktion zuordnen (z. B. „Plan-Approval fehlt → fülle § Approval gate in plan.md aus")
- **DARF NICHT [MUST NOT]** ein Pre-Publish-Gate stillschweigend überspringen, wenn `skip_gates=false`

### CLI-Aufruf

- **MUSS [MUST]** der Skill nach erfolgreichem Pre-Publish-Pass `reachy-mini-app-assistant publish <app_path> "<commit_message>"` aufrufen, mit den entsprechenden Visibility-/Official-/Nocheck-Flags
- **MUSS [MUST]** der CLI-Output als-ist an den User zurückgegeben werden; der Skill modifiziert ihn nicht
- **DARF NICHT [MUST NOT]** der Skill den Hugging-Face-Push selbst implementieren (`huggingface_hub`-API direkt nutzen) — das CLI ist die einzige unterstützte Schnittstelle
- **DARF NICHT [MUST NOT]** der Skill bei einem fehlgeschlagenen `publish`-Aufruf erneut versuchen — Failure-Triage gehört zum User

### Post-Publish-Verifikation

- **MUSS [MUST]** nach erfolgreichem Publish die Hugging-Face-Space-URL aus dem CLI-Output extrahieren und im Report nennen
- **SOLLTE [SHOULD]** verifizieren, dass die URL erreichbar ist (HTTP-Probe ohne Auth, 200/302 erwartet)
- **SOLLTE [SHOULD]** den `git remote -v`-Output im App-Repo nennen, damit der User sieht, welcher HF-Remote angelegt wurde
- **DARF NICHT [MUST NOT]** der Skill den ersten App-Lauf auf Hugging Face triggern oder Telemetry sammeln — das ist außerhalb des Scopes

### Report-Format

- **MUSS [MUST]** der Skill einen kompakten Report zurückgeben mit den Sektionen: (1) Pre-Publish-Gate-Ergebnisse (✓/✗ pro Gate), (2) CLI-Aufruf-Befehl (mit Visibility-/Official-Flags), (3) Hugging-Face-Space-URL, (4) `git remote`-Auszug, (5) Next-Steps-Hinweis
- **SOLLTE [SHOULD]** in den Next-Steps auf den `reachy-mini-on-device`-Agent verweisen, falls die App noch nicht auf Hardware getestet wurde
- **DARF NICHT [MUST NOT]** der Report den HF-Token, Vault-Werte, oder andere Secrets enthalten (PII-Klausel analog zu [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md))

### Out-of-Scope-Klarstellung

- **DARF NICHT [MUST NOT]** der Skill App-Code modifizieren, einen Versions-Bump in `pyproject.toml` durchführen oder einen Changelog generieren — Repo-eigene Sache
- **DARF NICHT [MUST NOT]** Pollen-Daemon-Restart, App-Lock-Force-Release oder Sicherheits-Limits auf der Hardware — kein Hardware-Touchpoint
- **SOLLTE [SHOULD]** auf [`reachy-mini-deploy`](../reachy-mini-deploy/de.md) als Schwester-Operation für Deploy auf das eigene Gerät verweisen (HF ↔ Eigenes-Gerät sind verschiedene Distributionspfade)
- **SOLLTE [SHOULD]** auf [`reachy-mini-on-device`](../reachy-mini-on-device/de.md) als Validierungs-Operation **vor** dem Publish hinweisen (`app-development-workflow` Phase 8 vor Phase 10)

## Akzeptanzkriterien

- [ ] Skill ist unter `skills/reachy-app-publish-hf/SKILL.md` mit gültiger Frontmatter (`name: reachy-app-publish-hf`, `description`, optionale Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description` enthält die Schlüsselbegriffe (publish, Hugging Face, Reachy Mini, app, release, Space) und benennt mindestens drei Anti-Trigger explizit
- [ ] Die zehn Pre-Publish-Gates werden in der spezifizierten Reihenfolge geprüft, jeder mit klarer Fehlermeldung
- [ ] `skip_gates=false` ist Default; `skip_gates=true` aktiviert Pollens `--nocheck` und übergibt Gates als Warnung
- [ ] CLI-Aufruf nutzt ausschließlich `reachy-mini-app-assistant publish`; keine direkte `huggingface_hub`-API-Nutzung
- [ ] Visibility-Default ist `public`, `private` als Opt-in
- [ ] `official=true` ergibt Pollens `--official`-Flag im CLI-Aufruf
- [ ] Post-Publish-Report enthält Hugging-Face-Space-URL, `git remote`-Auszug, Next-Steps-Hinweis
- [ ] Bei `low`-Status eines Gates wird der konkrete Fix-Vorschlag im Report genannt
- [ ] Cross-Refs auf [`app-scaffold`](../app-scaffold/de.md), [`reachy-mini-deploy`](../reachy-mini-deploy/de.md), [`reachy-mini-on-device`](../reachy-mini-on-device/de.md), [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/de.md) sind sichtbar
- [ ] PII-Klausel ist erfüllt: keine Tokens, Vault-Werte, Secrets im Report
- [ ] `pre-commit run --all-files` läuft auf der Skill-Datei grün

## Quellen

> Quell-Verweise auf Pollen-Code-Dateien zeigen auf Datei + Zeilen-Nummer (sobald implementiert); Verweise auf Pollen-Markdown-Quellen sind Datei-Level zitiert.

- Pollens App-Assistant-CLI (Publish-Subcommand): <https://github.com/pollen-robotics/reachy-mini-app-assistant>
- Pollens App-Konzept-Doku (Publish-Konvention): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- Hugging-Face-Spaces-Doku (Distributions-Backend): <https://huggingface.co/docs/hub/spaces>
- Pollens `AGENTS.md` (Publish-Erwartungen, Plan-Konvention): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Interne Cross-Refs:
  - [`reachy-mini/app-development-workflow`](../../reachy-mini/app-development-workflow/de.md) — Phase 10 Vertrag
  - [`claude/app-scaffold`](../app-scaffold/de.md) — Provenienz-Marker, plan.md-Schema
  - [`claude/reachy-mini-deploy`](../reachy-mini-deploy/de.md) — Schwester-Distribution (Eigenes Gerät)
  - [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md) — Pre-Publish-Validierung
  - [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) — PII-Klausel-Vorbild

## Offene Fragen

- Aufbewahrungsdauer des Security-Review-Berichts: ist „letzte 30 Tage" der richtige Schwellwert, oder soll das App-spezifisch konfigurierbar sein? Erst pragmatisch, später schärfen.
- Branch-Restriktion auf `main` / `develop`: zu strikt, oder zu lax? Soll auch ein Tag-Pin erlaubt sein? Konsumenten-Feedback abwarten.
- Pollens `--official`-Flag: was sind die Pollen-seitigen Voraussetzungen für eine offizielle App-Anfrage? Pollen-Doku verifizieren, sobald der Flag tatsächlich verwendet wird.
- Visibility-Default `public`: ist das die richtige Pollen-Konvention, oder gibt es einen Sicherheits-Wunsch nach `private` als Default? Konsumenten-Konsens nötig.
- Multi-Space-Publish (z. B. „eine App, zwei HF-Org"): aktuell nicht im Scope; wenn ein Konsument das braucht, eigene Spec.
- Update-Workflow für bereits publishte Apps: ist eine separate Sub-Operation nötig, oder ist Re-Publish über das gleiche CLI-Verfahren OK? `reachy-mini-app-assistant publish` ist idempotent gegenüber dem HF-Space — also vermutlich kein eigener Update-Pfad nötig.
- Audit-Trail: soll der Skill jeden Publish-Vorgang in `<app_path>/.audits/publish/<timestamp>.md` ablegen, analog zu `.audits/security-review/`? Vorschlag: ja, niedriger Aufwand, hilft bei Compliance-Fragen.
