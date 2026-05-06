# Entwicklungs-Workflow für Reachy-Mini-Apps

Status: draft

## Kontext

**Leser:** erfahrene Reachy-Entwickler, Claude-Sessions beim Bau einer Reachy-Mini-App, zukünftige Skill-/Agent-Autoren, die diese Spec als Bedarfs-Karte nutzen.

Dieses Plugin (`claude-reachy-mini`) liefert Skills, Agents und Domain-Specs als Werkzeugkasten für die Entwicklung von Reachy-Mini-Apps. Was bisher fehlt, ist die **Methodik-Spec**, die beschreibt, *wie* ein erfahrener Reachy-Entwickler aus einer Anforderung eine fertige, lauffähige App entstehen lässt — welche Phasen er durchläuft, welche Quellen er konsultiert, welche Gates er nicht überspringen darf, und an welcher Stelle der Code auf sicherheitsrelevante Fehlimplementierungen geprüft wird.

Ohne diese Spec rekonstruiert jede Claude-Session den Prozess neu, was zu zwei wiederkehrenden Problemen führt: (1) einzelne Phasen werden übersprungen — typisch der Plan-First-Schritt und der Security-Review-Schritt, (2) das Skill- und Agent-Inventar wächst unkoordiniert, weil es nicht klar ist, welche Phase überhaupt durch welches Tool abgedeckt sein sollte.

Diese Spec ist deshalb zwei Dinge gleichzeitig: die kanonische Workflow-Beschreibung *und* die Zuordnung der Workflow-Phasen zu Skills/Agents. Damit dient sie als Bedarfs-Karte (welche Skills/Agents existieren, welche fehlen) und als Qualitäts-Gate (welche Phase darf nicht übersprungen werden).

Sie ist die methodische Schwester der Artefakt-Spec [`reachy-mini/app-architecture`](../app-architecture/de.md): jene legt fest, *was* die fertige App ist, diese legt fest, *wie* sie entsteht.

## Ziele

- Durchgehender Workflow von Anforderungs-Aufnahme bis lauffähiger, deployter App, in klar abgegrenzten Phasen mit Inputs, Outputs und Verantwortlichem (Skill/Agent oder Mensch)
- Verbindlicher **Plan-First-Gate**: kein Code ohne schriftlich festgehaltenen, vom Anforderer abgenommenen Plan
- Verbindlicher **Security-Review-Gate** vor dem Deploy: Source-Code wird gegen einen Reachy-spezifischen Sicherheits-Katalog geprüft
- **Eindeutige Skill/Agent-Zuordnung pro Phase**: jede Phase nennt entweder den existierenden Skill/Agent als Owner oder markiert die Lücke als `GAP`
- **Eindeutige Referenz-Liste**: jede Phase verweist auf die relevanten SDKs, Pollen-Dokumentationen und internen Specs, gegen die gearbeitet wird
- Zweck des Plans und Form des Plans sind so präzise definiert, dass ein Reviewer den Plan als Artefakt finden, lesen und gegen die Anforderung abgleichen kann

## Nicht-Ziele

- Diese Spec **ersetzt nicht** die einzelnen Skill- und Agent-Specs (`app-scaffold`, `reachy-mini-sdk`, `reachy-mini-on-device`, `reachy-mini-deploy`, `reachy-mini-start`, `dance-choreography`, `home-assistant-bridge`); sie verlinkt und ordnet sie ein
- Diese Spec **ersetzt nicht** die Artefakt-Spec `reachy-mini/app-architecture`; sie bezieht sich auf deren Output, beschreibt aber den Weg dorthin
- Kein Python-/Allgemein-Software-Tutorial — die Spec setzt einen erfahrenen Python-Entwickler voraus, der mit `pip`, `pyproject.toml`, virtuellen Umgebungen und git vertraut ist
- Kein vollständiges Threat-Model des Reachy-Mini-Stacks — der Security-Review-Gate ist Code-Review-orientiert; ein eigenes Threat-Model ist eigene Spec, falls überhaupt benötigt
- Keine Release-/Versions-Strategie — `release-automation` und `release-publish-trigger` haben eigene Specs in `nolte-shared`
- Keine CI-/CD-Pipeline-Spezifikation — die Workflow-Definition ist Ablauf-orientiert, nicht Pipeline-orientiert
- Keine generische Anforderungs-Engineering-Methodik — die Anforderungs-Phase nimmt an, dass der Anforderer eine User-Story / ein Ziel mitbringt; Stakeholder-Discovery ist außerhalb des Scopes

## Anforderungen

### Workflow-Phasen — Übersicht

Eine Reachy-Mini-App-Entwicklung **MUSS [MUST]** in genau folgenden Phasen ablaufen, in dieser Reihenfolge:

1. **Anforderungs-Aufnahme** (Requirement Intake)
2. **Domain-Discovery** (Spec- und SDK-Sichtung)
3. **Plan-First-Gate** (Plan erstellen, abnehmen, einfrieren)
4. **Scaffold** (Projekt-Skeleton erzeugen)
5. **Implementierung** (iterativ, gegen den Plan)
6. **Lokaler Selbst-Test** (Simulation oder lokaler Daemon)
7. **Security-Review-Gate** (Code gegen Sicherheits-Katalog prüfen)
8. **On-Device-Test** (Live-Trial auf realer Hardware)
9. **Deploy & Start** (Installation und Start auf dem Reachy)
10. **Publikation** (optional — Hugging-Face-Spaces oder andere Distribution)

Jede Phase hat Inputs, Outputs, Owner (Skill/Agent oder Mensch) und Referenzen. Phasen-Skipping ist nur an explizit gekennzeichneten Stellen erlaubt (siehe „Skip-Regeln" unten).

### Phase 1 — Anforderungs-Aufnahme

- **MUSS [MUST]** mindestens festhalten: was die App tut (Verhaltens-Beschreibung), für wen sie es tut (Anforderer / Nutzer), wann sie als fertig gilt (Erfolgs-Kriterien)
- **MUSS [MUST]** klären, ob das Verhalten reine Hardware-Aktion ist, eine HA-Integration trägt, oder ob es Audio- / Vision-Input verarbeitet — diese Achsen entscheiden später, welche Specs in der Discovery-Phase relevant sind
- **SOLLTE [SHOULD]** in einem persistenten Artefakt landen (Issue, ADR, Plan-Vorform), nicht nur als Chat-Verlauf
- **KANN [MAY]** schon konkrete Motion-Slugs aus dem Inventar `reachy-mini/motions/` referenzieren, wenn der Anforderer sie kennt

**Inputs:** User-Wunsch in beliebiger Form
**Outputs:** Anforderungs-Beschreibung mit Erfolgs-Kriterien
**Owner:** Mensch + Anforderer (kein Skill/Agent — `GAP-OK`: Anforderungen sind Mensch-zu-Mensch-Kontext, kein Tooling-Bedarf)
**Referenzen:** —

### Phase 2 — Domain-Discovery

Bevor irgendein Plan entsteht, **MUSS [MUST]** der Entwickler die folgenden internen Specs gelesen oder gegengelesen haben (die Liste ist abhängig von der Anforderung; mindestens die ersten drei sind immer Pflicht):

- [`reachy-mini/app-architecture`](../app-architecture/de.md) — App-Layout, Daemon-Lifecycle, Distributions-Pfad
- [`reachy-mini/control-surface`](../control-surface/de.md) — was am Reachy steuerbar ist und in welchen Grenzen
- [`reachy-mini/app-logging`](../app-logging/de.md) — Logging-Topologie, Konventionen, Triage-Katalog
- [`reachy-mini/ha-integration`](../ha-integration/de.md) — wenn Home-Assistant-Touchpoint vorgesehen
- [`reachy-mini/motions/<slug>`](../motions/) — wenn konkrete Bewegungs-Primitive verwendet werden
- Skill-Specs der Skills, die später aufgerufen werden: [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md), [`claude/app-scaffold`](../../claude/app-scaffold/de.md)

Zusätzlich **MUSS [MUST]** mindestens ein Blick auf die folgenden externen Quellen erfolgen, um die SDK-Realität gegen die Spec-Annahmen abzugleichen:

- Pollen-Robotics SDK-Repository: https://github.com/pollen-robotics/reachy_mini
- Pollen-Robotics App-Assistant-CLI (Scaffold-Tool): https://github.com/pollen-robotics/reachy-mini-app-assistant
- Hugging-Face-Spaces-Distribution für Reachy-Mini-Apps: https://huggingface.co/collections/pollen-robotics/reachy-mini
- Pollen `AGENTS.md` und `skills/`-Verzeichnisse im SDK-Repository (kanonische Pollen-Konventionen)

**Inputs:** Anforderungs-Beschreibung aus Phase 1
**Outputs:** Liste relevanter Specs und externer Doku-Stellen, kurze Notiz pro Quelle, was sie für die Anforderung beiträgt
**Owner:** [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md)-Skill (Wissens-Aktivierung) + Mensch (Kuratierung)
**Referenzen:** alle oben genannten

### Phase 3 — Plan-First-Gate

Diese Phase ist **das zentrale Gate** des Workflows. Vor diesem Gate gibt es nur Lesen und Notizen; nach diesem Gate beginnt die Code-Erzeugung.

- **MUSS [MUST]** als Datei `plan.md` im Wurzelverzeichnis des Ziel-App-Repos entstehen und mitversioniert sein — `plan.md` ist die einzige kanonische Plan-Senke; Issues und PR-Beschreibungen dürfen darauf verlinken, ersetzen es aber nicht
- **MUSS [MUST]** vom Anforderer (oder vom Entwickler in der Rolle des Anforderers) explizit abgenommen werden, bevor Phase 4 beginnt; die Abnahme wird im Plan selbst protokolliert (siehe Template-Stub unten)
- **MUSS [MUST]** dem in dieser Spec verbindlich vorgegebenen Plan-Template-Stub folgen (siehe nächster Unterabschnitt) — Abschnitts-Namen und -Reihenfolge sind fix, Inhalt darf je App-Charakter wachsen
- **MUSS [MUST]** mindestens folgende Plan-Abschnitte ausgefüllt mitbringen:
  - Scope: was die App tut, was sie explizit nicht tut
  - Motion-Inventar: welche Motion-Specs / Move-Klassen verwendet werden
  - IPC-Oberfläche: WebSocket-Befehle, eingehend wie ausgehend (per `reachy-mini/app-architecture`)
  - Externe Touchpoints: HA-Services, Webhooks, Audio-Eingang — wenn vorgesehen
  - Sicherheits-Erwägungen: welche Secrets, welche Netzwerk-Surfaces, welche Eingabe-Validierung (siehe Phase 7)
  - Test-Strategie: was wird in Simulation getestet, was muss aufs Gerät, was bleibt manuell
  - Skill/Agent-Plan: welcher Skill/Agent wird in welcher späteren Phase aufgerufen
- **SOLLTE [SHOULD]** Open Questions explizit listen, statt sie zu erfinden — offene Punkte sind Plan-Inhalt, keine Zukunfts-Annahmen
- **KANN [MAY]** durch frühe Skill-Konsultation entstehen (z. B. `dance-choreography` produziert ein Plan-Vorprodukt für eine Tanz-App)
- **MUSS NICHT [MUST NOT]** Code enthalten — Pseudocode oder Schnittstellen-Skizzen sind erlaubt, ausführbarer Code ist nicht erlaubt

#### Plan-Template-Stub (verbindlich)

Jeder Plan **MUSS [MUST]** mit dem folgenden Markdown-Skelett beginnen. Abschnitts-Überschriften und -Reihenfolge sind fix; je App-Charakter dürfen Tabellen-Zeilen, Stichpunkte und Unter-Bullets wachsen, aber keine Top-Level-Abschnitte fehlen oder umbenannt werden.

```markdown
# Plan: <App-Name>

Status: draft | abgenommen
Anforderer: <Name>
Entwickler: <Name>
Datum: <YYYY-MM-DD>

## Scope
- **Tut:**
- **Tut explizit nicht:**
- **Erfolgs-Kriterien:**

## Motion-Inventar
| Slug | Move-Klasse | Quelle (Motion-Spec / SDK) |
|---|---|---|

## IPC-Oberfläche
- **Eingehend (WebSocket):**
- **Ausgehend (WebSocket):**

## Externe Touchpoints
- **HA-Services:**
- **Webhooks:**
- **Audio / Vision:**

## Sicherheits-Erwägungen
- **Secrets (Quelle, Speicher-Form):**
- **Netzwerk-Surfaces (Bind-Adresse, TLS):**
- **Eingabe-Validierung (Schema-Form):**
- **Whitelist / Limits (Motion-Slugs, Wertebereiche):**

## Test-Strategie
- **Simulation:**
- **On-Device:**
- **Manuell:**

## Skill / Agent-Plan
| Phase | Skill / Agent |
|---|---|
| 4 — Scaffold | claude/app-scaffold |
| 5 — Implementierung | claude/reachy-mini-sdk |
| 7 — Security-Review | reachy-app-security-review (geplant) + nolte-shared:security-review |
| 8 — On-Device-Test | claude/reachy-mini-on-device |
| 9 — Deploy & Start | claude/reachy-mini-deploy + claude/reachy-mini-start |

## Open Questions
-

## Abnahme
- [ ] Anforderer (`<Name>`): abgenommen am <YYYY-MM-DD>
- [ ] Entwickler (`<Name>`): bestätigt am <YYYY-MM-DD>
```

**Inputs:** Anforderungs-Beschreibung, Discovery-Notizen
**Outputs:** Abgenommener Plan-Artefakt
**Owner:** Mensch — `GAP`: derzeit kein dedizierter Plan-Skill; Kandidat ist ein zukünftiger `app-plan-author`-Skill, der aus Anforderung + Discovery-Output einen Plan-Entwurf erzeugt; bis dahin ist die Phase mensch-getragen mit punktueller Skill-Unterstützung (`reachy-mini-sdk`, `dance-choreography`)
**Referenzen:** [`reachy-mini/app-architecture`](../app-architecture/de.md); der `app-scaffold`-Skill legt die `plan.md`-Datei an und übernimmt das hier definierte Template-Schema wörtlich

### Phase 4 — Scaffold

- **MUSS [MUST]** über den [`claude/app-scaffold`](../../claude/app-scaffold/de.md)-Skill und damit über die Pollen-CLI `reachy-mini-app-assistant create` erfolgen — kein hand-gerolltes Skeleton
- **MUSS [MUST]** Provenienz-Marker setzen (CLAUDE.md, Verweis auf dieses Plugin) wie in `app-scaffold` definiert
- **MUSS [MUST]** den abgenommenen Plan aus Phase 3 als `plan.md` im Repo verankern
- **SOLLTE [SHOULD]** den Reachy-SDK-Pin auf eine konkrete Minor-Version setzen (per `reachy-mini/app-architecture`)

**Inputs:** Plan-Artefakt
**Outputs:** Initialisiertes App-Repository mit Pollen-konformer Struktur, eingefrorenem Plan, Provenienz
**Owner:** [`claude/app-scaffold`](../../claude/app-scaffold/de.md)-Skill
**Referenzen:** [`reachy-mini/app-architecture`](../app-architecture/de.md), Pollen-CLI

### Phase 5 — Implementierung

- **MUSS [MUST]** dem Plan folgen; Plan-Abweichungen werden im Plan dokumentiert (Plan-Update + erneute Abnahme), bevor sie umgesetzt werden
- **MUSS [MUST]** den [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md)-Skill als Wissensbasis für SDK-Idiome nutzen
- **MUSS [MUST]** Logging gemäß [`reachy-mini/app-logging`](../app-logging/de.md) Konvention verwenden (`logging.getLogger(__name__)`, kein `print(...)` außer bei expliziter Debug-Sitzung)
- **MUSS [MUST]** den `reachy_mini`-SDK auf einen konkreten Minor-Pin festlegen
- **SOLLTE [SHOULD]** in kleinen Schritten committen, mit aussagekräftigen Conventional-Commit-Messages
- **MUSS NICHT [MUST NOT]** Funktionalität jenseits des Plans hinzufügen ohne Plan-Update — Scope-Creep ist eine Plan-Verletzung

**Inputs:** Plan, Scaffold
**Outputs:** App-Code mit allen Plan-Punkten umgesetzt
**Owner:** Mensch + [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md)-Skill
**Referenzen:** [`reachy-mini/app-architecture`](../app-architecture/de.md), [`reachy-mini/control-surface`](../control-surface/de.md), [`reachy-mini/app-logging`](../app-logging/de.md), Pollen-SDK-Quellen

### Phase 6 — Lokaler Selbst-Test

- **MUSS [MUST]** mindestens den `ReachyMini(spawn_daemon=True, use_sim=True)`-Modus durchlaufen, sofern keine reine Hardware-Aktion ohne SDK-Test-Pfad vorliegt
- **SOLLTE [SHOULD]** Pollen-Heuristik „verify basics first" anwenden: `examples/minimal_demo.py` oder ein App-spezifischer Smoke-Test als erste Sanity-Check-Stufe (per [`reachy-mini/app-logging`](../app-logging/de.md))
- **KANN [MAY]** über `pytest` automatisierte Tests fahren, soweit der App-Charakter das hergibt

**Inputs:** App-Code
**Outputs:** Lokaler Lauf-Nachweis (Log-Ausschnitt, Test-Ergebnis)
**Owner:** Mensch — `GAP-OK`: lokales Testen ist Standard-Python-Workflow, kein eigener Skill-Bedarf
**Referenzen:** [`reachy-mini/app-logging`](../app-logging/de.md)

### Phase 7 — Security-Review-Gate

Diese Phase ist **das zweite zentrale Gate** des Workflows. Sie **MUSS [MUST]** vor jedem On-Device-Test gegen reale Hardware in einer Mehrnutzer-Umgebung und vor jedem Deploy bestanden werden.

Der Source-Code **MUSS [MUST]** gegen den folgenden Reachy-spezifischen Sicherheits-Katalog geprüft werden. Jeder Punkt **MUSS [MUST]** entweder als „nicht zutreffend" begründet oder als bestanden markiert sein:

#### Secrets und Credentials

- Keine Klartext-Secrets im Repository (HA-Long-Lived-Tokens, Wyoming-Keys, MQTT-Credentials, HF-Tokens) — `git grep`-Negativ-Check auf typische Patterns (`token`, `password`, `api_key`, `Bearer `, JWT-Strukturen)
- Secrets werden ausschließlich über Umgebungsvariablen oder externe Secret-Stores geladen
- Logging redact: Secrets erscheinen niemals in Log-Ausgaben, weder INFO noch DEBUG noch in Tracebacks (Pollen-Daemon captured stderr — siehe [`reachy-mini/app-logging`](../app-logging/de.md))

#### Netzwerk-Surfaces

- Pollen-konformer App-Lokal-WebSocket bleibt `localhost`-only, keine Bind-Erweiterung auf `0.0.0.0` (per [`reachy-mini/app-architecture`](../app-architecture/de.md) explizites Non-Goal)
- HA-Integrationen nutzen TLS, sofern HA-Endpoint nicht ausdrücklich als plain `http://` deklariert ist; selbst dann SOLLTE ein Hinweis im Plan stehen
- Externe Webhooks validieren ihre Signaturen / shared Secrets — kein „accept all"-Endpoint

#### Eingabe-Validierung

- Alle WebSocket-/Webhook-Eingaben werden gegen ein erwartetes Schema validiert (Pydantic, dataclass + Validierung, jsonschema o. Ä.) — keine ungeprüfte Übergabe an SDK-Methoden
- Motion-Slug-Aufrufe werden gegen die Whitelist der bekannten Motion-Slugs aus [`reachy-mini/motions/`](../motions/) abgeglichen — keine String-Konkatenation, die zu Reflection auf beliebige Klassen führen kann
- Numerische Werte (Winkel, Geschwindigkeiten) werden gegen die Limits aus [`reachy-mini/control-surface`](../control-surface/de.md) geclampt, **bevor** sie ans SDK gehen

#### Abhängigkeiten

- Vor dem Deploy ein `pip-audit` (oder gleichwertig) gegen den Lockfile — siehe [`nolte-shared:dependency-audit`](https://github.com/nolte/claude-shared/tree/main/skills/dependency-audit) als Tooling-Anker
- `reachy_mini`-Pin folgt dem Architektur-Soll (konkrete Minor-Version)

#### Privilege & Side-Effects

- Keine `subprocess.Popen(..., shell=True)` mit ungeprüfter Eingabe
- Keine Datei-Schreibzugriffe außerhalb des erwarteten App-Datenverzeichnisses
- Keine Cloud-AI-Aufrufe aus dem App-Prozess (per [`reachy-mini/app-architecture`](../app-architecture/de.md) explizites Non-Goal — wenn der Plan das doch verlangt, ist es eine Plan-Änderung mit erneuter Abnahme)

#### Erwartete Tooling-Anbindung

- **MUSS [MUST]** dieser Gate so geführt werden, dass der Output reproduzierbar ist (Markdown-Report, Issue-Kommentar oder PR-Review)
- **SOLLTE [SHOULD]** der `nolte-shared`-Skill `security-review` (allgemeine Code-Security-Review-Skill, Plugin-extern) zur Hilfe genommen werden — er liefert einen Security-Review-Lauf gegen den Diff
- **KANN [MAY]** der `nolte-shared:dependency-audit`-Skill den CVE-Teil automatisieren

- **MUSS [MUST]** der Report im Ziel-App-Repo unter `.audits/security-review/<YYYY-MM-DD>.md` abgelegt und mitversioniert werden; im Gegensatz zu den Agenten-eigenen Unterverzeichnissen (`.audits/deploy/`, `.audits/on-device/` — beide gitignored, weil Maschinen-Logs) ist `.audits/security-review/` ein menschen-erzeugter Auditierungs-Beweis und gehört in git
- **MUSS [MUST]** das App-Repo eine `.gitignore`-Regel führen, die dies explizit macht — Vorlage: `.audits/` ignorieren, `!.audits/security-review/` als Ausnahme; PR-/Issue-Verlinkung ist zusätzlich erlaubt, ersetzt aber nicht die Datei

**Inputs:** App-Code, Plan (für „nicht zutreffend"-Begründungen)
**Outputs:** Security-Review-Report unter `.audits/security-review/<YYYY-MM-DD>.md` (jeder Katalog-Punkt: bestanden / begründet nicht zutreffend / Befund + Fix-Beschreibung)
**Owner:** Mensch + [`nolte-shared:security-review`](https://github.com/nolte/claude-shared) Skill — `GAP`: dedizierter Plugin-Skill `reachy-app-security-review` ist als eigener Skill **innerhalb dieses Plugins** geplant, weil der Reachy-Katalog (Motion-Whitelist gegen `reachy-mini/motions/`, Wertebereiche aus `reachy-mini/control-surface`, Pollen-Daemon-Non-Goals aus `reachy-mini/app-architecture`) plugin-lokales Domain-Wissen ist und nicht in `nolte-shared` gehört; bis dieser Skill existiert, ist die Phase mensch-getragen mit dem allgemeinen `nolte-shared:security-review`-Skill als Hilfe
**Referenzen:** [`reachy-mini/app-architecture`](../app-architecture/de.md), [`reachy-mini/control-surface`](../control-surface/de.md), [`reachy-mini/app-logging`](../app-logging/de.md), [`reachy-mini/motions/`](../motions/), Pollen-SDK-Quellen

### Phase 8 — On-Device-Test

- **MUSS [MUST]** über den [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md)-Agent erfolgen, der einen begrenzten Test-Lifecycle mit Telemetrie-Beobachtung führt
- **MUSS [MUST]** das Ergebnis als Audit-Report unter `.audits/on-device/` ablegen (Agent-Vertrag)
- **SOLLTE [SHOULD]** mindestens den Plan-„Erfolgs-Kriterium"-Pfad einmal erfolgreich durchlaufen lassen

**Inputs:** App-Code mit bestandenem Security-Review
**Outputs:** Strukturierter PASS/FAIL-Report
**Owner:** [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md)-Agent
**Referenzen:** [`reachy-mini/app-logging`](../app-logging/de.md)

### Phase 9 — Deploy & Start

- **Deploy** **MUSS [MUST]** über den [`claude/reachy-mini-deploy`](../../claude/reachy-mini-deploy/de.md)-Agent erfolgen — er prüft Pollen-Vertrag, synct den Code, verankert die Installation in der Daemon-Umgebung
- **Start** **MUSS [MUST]** über den [`claude/reachy-mini-start`](../../claude/reachy-mini-start/de.md)-Skill erfolgen — er respektiert App-Locks und prüft den Entry-Point-Katalog
- **MUSS NICHT [MUST NOT]** der Deploy-Agent zum reinen Run-Zweck eingesetzt werden — Live-Trial ist Phase 8, Run-Start ist `reachy-mini-start`

**Inputs:** App-Repo mit bestandenem On-Device-Test
**Outputs:** Auf dem Gerät installierte und gestartete App
**Owner:** [`claude/reachy-mini-deploy`](../../claude/reachy-mini-deploy/de.md)-Agent + [`claude/reachy-mini-start`](../../claude/reachy-mini-start/de.md)-Skill
**Referenzen:** [`reachy-mini/app-architecture`](../app-architecture/de.md)

### Phase 10 — Publikation

- **KANN [MAY]** als Hugging-Face-Space veröffentlicht werden (`reachy-mini-app-assistant publish` direkt — derzeit kein dedizierter Plugin-Skill)
- **SOLLTE [SHOULD]** den finalen Security-Review-Report als Teil der Release-Notes referenzieren

**Inputs:** Lauffähige App-Version
**Outputs:** Veröffentlichte App
**Owner:** Pollen-CLI direkt — `GAP-OK`: derzeit ausreichend ohne Plugin-Skill; bei Bedarf Kandidat `reachy-app-publish-hf` (siehe Skill-/Agent-Hooks)
**Referenzen:** Hugging-Face-Spaces-Doku

### Skip-Regeln

- Phase 6 (Lokaler Selbst-Test) **KANN [MAY]** übersprungen werden, wenn die App ausschließlich Hardware-Effekte hat, die in Simulation nicht prüfbar sind — Begründung wandert in den Plan
- Phase 10 (Publikation) ist optional und immer ein Phasen-Skip-Kandidat
- Alle anderen Phasen **MUSS NICHT [MUST NOT]** übersprungen werden
- Insbesondere die Gates **Plan-First** (Phase 3) und **Security-Review** (Phase 7) sind nicht überspringbar — auch nicht „nur dieses eine Mal"

### Update-Workflow (für Behavior-Updates einer existierenden App)

Wird ein Behavior in einer bereits existierenden, gescaffoldeten App **erweitert oder geändert**, gilt eine reduzierte Phasen-Schleife. Sie ersetzt nicht den vollen Workflow für neue Apps, sondern beschreibt den Sonderfall „inkrementelle Änderung".

- **MUSS [MUST]** mindestens die Phasen **1, 3, 5, 7, 8, 9** durchlaufen — Anforderung aufnehmen, Plan aktualisieren und neu abnehmen, implementieren, Security-Review erneut bestehen, On-Device testen, deployen und starten
- **KANN [MAY]** Phase **2** (Domain-Discovery) auf eine Delta-Discovery reduzieren: nur die Specs werden konsultiert, deren Bereich vom Update berührt ist
- **MUSS NICHT [MUST NOT]** Phase **4** (Scaffold) erneut durchlaufen — die App existiert bereits; ein erneutes Scaffold würde Provenienz-Marker und Plan-Historie zerstören
- **KANN [MAY]** Phase **6** (Lokaler Selbst-Test) und Phase **10** (Publikation) wie im Vollworkflow überspringen, mit denselben Begründungs-Pflichten
- **MUSS [MUST]** das `plan.md` im App-Repo **aktualisiert** und neu abgenommen werden — kein zweites `plan.md`, keine stillen Plan-Änderungen; Historie ergibt sich aus git
- **MUSS [MUST]** der Security-Review-Report einen neuen `.audits/security-review/<YYYY-MM-DD>.md` als eigenen Eintrag erzeugen (nicht den alten überschreiben), damit Update-Reports historisch nachvollziehbar bleiben

### Skill-/Agent-Hooks (Bedarfs-Karte)

Diese Tabelle ist die zentrale Ableitung dieser Spec: pro Phase ist sichtbar, welcher Skill/Agent existiert und wo Lücken sind. Lücken sind Auftrags-Vorrat für `nolte-shared:claude-plugin-developer` bzw. `nolte-shared:skill-management`.

| Phase | Owner heute | Status | Bedarfs-Vorschlag |
|---|---|---|---|
| 1 — Anforderungs-Aufnahme | Mensch | `GAP-OK` | kein Tooling-Bedarf |
| 2 — Domain-Discovery | `reachy-mini-sdk`-Skill (partial) | `OK` | — |
| 3 — Plan-First-Gate | Mensch | `GAP` | zukünftig `app-plan-author`-Skill; Eingabe = Anforderung + Discovery, Ausgabe = Plan-Entwurf |
| 4 — Scaffold | `app-scaffold`-Skill | `OK` | — |
| 5 — Implementierung | Mensch + `reachy-mini-sdk`-Skill | `OK` | — |
| 6 — Lokaler Selbst-Test | Mensch | `GAP-OK` | kein Tooling-Bedarf |
| 7 — Security-Review-Gate | Mensch + `nolte-shared:security-review` | `GAP` | dedizierter Plugin-Skill `reachy-app-security-review` in **diesem** Plugin (kennt Motion-Whitelist, control-surface-Limits, Pollen-Non-Goals); diese Spec § Phase 7 ist die Vorlage |
| 8 — On-Device-Test | `reachy-mini-on-device`-Agent | `OK` | — |
| 9 — Deploy & Start | `reachy-mini-deploy`-Agent + `reachy-mini-start`-Skill | `OK` | — |
| 10 — Publikation | Pollen-CLI direkt | `GAP-OK` | optional zukünftig `reachy-app-publish-hf`-Skill |

## Akzeptanzkriterien

- [ ] Jede der zehn Phasen hat in dieser Spec einen eigenen Abschnitt mit Inputs, Outputs, Owner und Referenzen
- [ ] Jede Phase außer Phase 1 (Anforderungs-Aufnahme) führt mindestens einen Verweis auf eine interne Spec, einen Skill, einen Agent oder eine externe Doku-URL
- [ ] Plan-First-Gate ist durch ein konkretes Plan-Artefakt prüfbar (Reviewer kann die Datei `plan.md` im App-Repo öffnen) und der Plan folgt dem in dieser Spec verbindlich vorgegebenen Template-Stub (Abschnitts-Namen und -Reihenfolge)
- [ ] Security-Review-Katalog ist in mindestens fünf konkrete, prüfbare Kategorien gegliedert (Secrets, Netzwerk, Eingabe, Abhängigkeiten, Privilege)
- [ ] Security-Review-Report aus Phase 7 liegt unter `.audits/security-review/<YYYY-MM-DD>.md` im App-Repo und ist mitversioniert; das App-Repo führt eine `.gitignore`-Ausnahme `!.audits/security-review/`
- [ ] Skill-/Agent-Hooks-Tabelle bildet jede Phase auf einen Owner ab und nennt für jede `GAP`-Phase einen Skill-/Agent-Kandidaten *mit* Begründung der Plugin-Grenze (intern in diesem Plugin vs. `nolte-shared` vs. extern)
- [ ] Alle Verweise auf existierende interne Specs zeigen auf real vorhandene Pfade unter `spec/`
- [ ] Mindestens drei externe SDK-/Doku-URLs sind in Phase 2 verlinkt und derefenzierbar
- [ ] Skip-Regeln markieren explizit, welche Phasen unter welcher Bedingung übersprungen werden dürfen, und welche niemals
- [ ] Bei einem Update-Workflow-Lauf trägt `plan.md` einen aktualisierten Abnahme-Eintrag mit Datum nach dem letzten Commit vor dem Update; `.audits/security-review/` enthält einen neuen Datums-Eintrag, der den vorherigen nicht überschreibt; Phase 4 (Scaffold) wurde nicht erneut durchlaufen

## Offene Fragen

Alle initialen offenen Fragen wurden in Iteration 1 entschieden:

- **Plan-Artefakt-Ort:** `plan.md` im App-Repo ist die einzige kanonische Plan-Senke (siehe § Phase 3)
- **Plan-Template:** verbindlicher Markdown-Stub in dieser Spec (siehe § Phase 3, „Plan-Template-Stub")
- **Security-Review-Skill-Grenze:** dedizierter Plugin-interner Skill `reachy-app-security-review` (siehe § Phase 7 und Skill-/Agent-Hooks)
- **Security-Report-Archiv:** `.audits/security-review/<YYYY-MM-DD>.md` im App-Repo (siehe § Phase 7)
- **Update-Workflow:** reduzierte Phasen-Schleife 1, 3, 5, 7, 8, 9 (siehe § Update-Workflow)

Neue offene Fragen, die sich aus der Anwendung ergeben, werden in einer zweiten Iteration hier ergänzt.
