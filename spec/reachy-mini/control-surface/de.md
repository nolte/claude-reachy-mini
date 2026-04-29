# Steuerungs-Oberfläche und Bewegungs-Design des Reachy Mini

Status: draft

## Kontext
Wer in diesem Plugin Skills, Behaviors oder Agents schreibt, braucht eine kanonische Referenz dafür, *welche* Elemente am Reachy Mini überhaupt steuerbar sind, *wie* sie angesprochen werden und *unter welchen Grenzen* daraus natürlich wirkende Bewegungen komponiert werden. Ohne diese Spezifikation rekonstruiert jede Implementierung die Hardware-Realität neu — meistens aus halbgaren Trainings-Stichproben, was beim ersten echten Gerätekontakt entweder Hardware gefährdet oder zu mechanisch wirkenden Bewegungen führt. Diese Spec konsolidiert die öffentlich bekannten Hardware- und SDK-Eigenschaften des Reachy Mini, beschreibt jede Steuerungs-Schicht des SDKs, und gibt für jede mechanische und elektrische Limitation an, wie sie die Bewegungs-Komposition formt. Sie ist die normative Wissensbasis, gegen die der `reachy-mini-sdk`-Skill seine Snippets validiert und gegen die der `behavior-scaffold`-Skill seine Schablonen anpasst.

## Ziele
- Jedes steuerbare Element des Reachy Mini ist katalogisiert mit Bezeichner, Achsen, Wertebereich und Einheit
- Steuerungs-Schichten (Pose-Goto, Move-Primitive, Behavior-Lifecycle, Streaming) sind voneinander abgegrenzt; ein Behavior weiß, welche Schicht für welche Aufgabe taugt
- Abhängigkeiten zwischen Hardware-Variante, SDK-Version und Firmware sind sichtbar gemacht, sodass eine Implementierung sie früh prüfen kann
- Mechanische, elektrische und latenz-bezogene Limitationen sind dokumentiert in einer Form, die im Code als Vorbedingung formuliert werden kann
- Patterns für natürliche, flüssige Bewegung (Easing, Komposition, Anticipation, Follow-Through, Synchronisation, Idle-Atemzug) sind ausformuliert und mit Anti-Patterns kontrastiert
- Alle hardware-spezifischen Zahlen sind als `> ⚠ TBD` markiert, bis sie gegen die echte SDK-Doku und das Gerät verifiziert sind

## Nicht-Ziele
- Konkrete Bewegungs-Choreografien (Aufgabe einzelner Behaviors / Apps)
- Audio-Beat-Tracking-Pipelines (`audio-beat-tracking`, geplant)
- Home-Assistant-API-Vertrag (`home-assistant-bridge`)
- Hardware-Bringup, Kalibrierung, Firmware-Flash (eigene Skills, geplant)
- Simulations- oder URDF-Modell (separate Spec)
- Vision-Pipeline, Sprach-Erkennung, On-Device-ML
- Mechanische CAD-Daten oder Elektrik-Schaltpläne — diese Spec bleibt auf der Software-Steuerungs-Ebene

## Anforderungen

### Architektur-Überblick

Reachy Mini ist ein Desktop-Roboter von Pollen Robotics / Hugging Face, der in zwei Varianten erscheint: **Wired** (kabelgebunden, hängt an einem Host-Rechner) und **Wireless** (mit eingebautem Raspberry Pi 5 und Akku). Die softwareseitige Steuerung läuft über das Python-SDK `reachy_mini`, das eine `ReachyMini`-Instanz als zentralen Einstiegspunkt zur Verfügung stellt. Diese Instanz aggregiert die einzelnen steuerbaren Subsysteme — Kopf, Antennen, Body-Rotation, Audio-Output, Display, Sensorik. Bewegungen werden in mehreren Schichten ausgedrückt; höhere Schichten verwenden niedrigere als Implementierungs-Detail.

> ⚠ TBD: validate against pollen-robotics/reachy_mini — die exakte Aggregation der Subsysteme an der `ReachyMini`-Instanz (Attribut-Pfade, Methoden-Namen) wird beim ersten Gerätekontakt gegen das offizielle SDK abgeglichen.

### Hardware-Inventar — Aktuatoren

Steuerbare mechanische Achsen:

| Subsystem | DoF | Achsen | Wertebereich | Einheit | Hardware-Variante |
|---|---|---|---|---|---|
| Kopf (Stewart-Plattform) | bis zu 6 | Pan (Yaw), Tilt (Pitch), Roll, optional Translation x/y/z | `> ⚠ TBD` pro Achse | rad bzw. mm `> ⚠ TBD` | Wired + Wireless |
| Antenne links | 1 | Rotation um Basis-Achse | `> ⚠ TBD` | rad | Wired + Wireless |
| Antenne rechts | 1 | Rotation um Basis-Achse | `> ⚠ TBD` | rad | Wired + Wireless |
| Körper-Rotation (Body-Yaw) | 1 | Rotation um vertikale Z-Achse | `> ⚠ TBD` | rad | nur Wireless `> ⚠ TBD` |

Anforderungen an die Inventar-Pflege:

- **MUSS [MUST]** jede Achse mit Bezeichner, Wertebereich, Einheit und Default-Pose dokumentieren, sobald die SDK-Doku verfügbar ist
- **MUSS [MUST]** je Achse ausweisen, ob die SDK-API absolute Targets, inkrementelle Targets, oder beides akzeptiert
- **MUSS [MUST]** Achsen markieren, die nur in einer Hardware-Variante existieren (z. B. Body-Yaw nur Wireless `> ⚠ TBD`)
- **SOLLTE [SHOULD]** je Achse die Geschwindigkeits- und Beschleunigungs-Grenzen ausweisen, die sich aus dem mechanischen Aufbau ergeben (`> ⚠ TBD`)

### Hardware-Inventar — Outputs (nicht-mechanisch)

| Subsystem | Eigenschaft | Steuerbarkeit |
|---|---|---|
| Display („Augen") | Mini-LCD/AMOLED, Auflösung `> ⚠ TBD`, Inhalt programmierbar | Bilder, einfache Animationen, Augen-Ausdruck-Layer |
| Lautsprecher | 1 Stück, Leistung `> ⚠ TBD W` | Audio-Wiedergabe (WAV/PCM, weitere Codecs `> ⚠ TBD`) |
| Status-LEDs (optional) | Anzahl `> ⚠ TBD` | `> ⚠ TBD` ob über SDK steuerbar oder fest |

Anforderungen:

- **MUSS [MUST]** das Display-Subsystem mindestens als ausdrucksgebende Output-Schicht modellieren — Bewegungs-Code darf gleichzeitig Augen-Ausdruck und Pose triggern
- **MUSS [MUST]** dokumentieren, ob die Audio-Wiedergabe synchron zum Behavior-Tick blockiert oder asynchron läuft (`> ⚠ TBD`)
- **SOLLTE [SHOULD]** Empfehlungen für Audio-Latenz-Mess- und Kompensations-Patterns enthalten, sobald die SDK-Verhaltensweise verifiziert ist

### Hardware-Inventar — Sensoren / Inputs

| Subsystem | Eigenschaft | Lese-API |
|---|---|---|
| Mikrofon-Array | 4 Mikrofone `> ⚠ TBD`, Richtungs-Capture möglich | Stream / Frame-Pulls `> ⚠ TBD` |
| Kamera | 1 Stück, Weitwinkel, Auflösung `> ⚠ TBD`, Framerate `> ⚠ TBD` | Frame-Pull / Stream `> ⚠ TBD` |
| IMU (falls vorhanden) | `> ⚠ TBD` ob vorhanden und ob über SDK exponiert | `> ⚠ TBD` |
| Position-Feedback der Aktuatoren | Geometrie-Lese-Endpoints für jede Achse | `reachy.head.pose`, `reachy.antennas.left.angle` etc. — exakte API `> ⚠ TBD` |

Anforderungen:

- **MUSS [MUST]** für jeden Aktuator dokumentieren, welche Methode den aktuellen Zustand zurückliefert; Behaviors müssen Position lesen können, ohne den Aktuator zu blockieren
- **MUSS [MUST]** klarstellen, ob Sensor-Streams blockierend oder als Async-Iteratoren bereitgestellt werden
- **DARF NICHT [MUST NOT]** annehmen, dass eine Sensor-Lesung kostenlos ist — die Lese-Frequenz unterliegt einer Limitation (siehe Latenz-Abschnitt)

### Hardware-Varianten und ihre Konsequenzen

- **Wired** — kabelgebunden, hängt an einem Host-PC; rechenstarker Host, niedrige Latenz auf der USB-Verbindung; ggf. kein Body-Yaw `> ⚠ TBD`
- **Wireless** — mit eingebautem Raspberry Pi 5 und Akku; CPU-/Speicher-Budget begrenzt, Bewegungs-Code muss schlanker laufen; Body-Yaw verfügbar `> ⚠ TBD`

Anforderungen:

- **MUSS [MUST]** im Code sichtbar machen, gegen welche Variante eine Bewegung geschrieben ist; eine Wired-Bewegung mit hoher Update-Frequenz darf nicht stillschweigend auf Wireless laufen, wenn dort die CPU nicht reicht
- **SOLLTE [SHOULD]** einen Weg vorsehen, die Variante zur Laufzeit zu erkennen (Capability-Discovery, `> ⚠ TBD` ob das SDK das anbietet)
- **DARF NICHT [MUST NOT]** Body-Yaw-Bewegungen ohne Variant-Check auf Wired-Geräten ausführen

### Steuerungs-Schichten

Vier Schichten, von feingranular (low-level) zu narrativ (high-level):

1. **Pose-Goto** — direktes Anfahren einer Ziel-Pose mit Dauer; SDK übernimmt Interpolation und Rampe
   - Beispiel-API: `reachy.head.goto(pan, tilt, roll, duration)` (`> ⚠ TBD`)
   - Granularität: ein Aufruf = eine Bewegung
   - Eigenes Interrupt-Modell: ein neuer Goto während eines laufenden Goto verhält sich `> ⚠ TBD` (Override / Queue / Reject)

2. **Move-Primitive** — komponierbare Bausteine wie Nicken, Schütteln, Look-At, Antennen-Flapping
   - Liefert Eigenschaften wie Easing-Profil, Dauer, Wiederholung
   - Hat eine `play()`/`compose()`-Form, die mit anderen Primitives parallel oder sequentiell verkettet wird (`> ⚠ TBD`)

3. **Behavior** — stateful, mit `setup`/`step`/`stop`-Hooks und Tick-Loop
   - Lese-Schreib-Kreislauf: jeder Tick liest Sensoren, entscheidet Move-Primitives, schreibt Targets
   - Tick-Frequenz: `> ⚠ TBD` Hz typisch; obergrenze durch CPU/Variante limitiert
   - Owns Lifecycle, kann Notstopp triggern

4. **Streaming** — kontinuierliche Pose-Targets bei niedriger Latenz, ohne Move-Primitive-Wrapping
   - Anwendungsfall: Tanz auf Audio-Stream, externe Mocap-Treiber
   - Benötigt explizit Backpressure-Handling, sonst Aktuator-Saturation
   - Verfügbarkeit `> ⚠ TBD`

Anforderungen:

- **MUSS [MUST]** in jedem geschriebenen Behavior klar entscheiden, welche Schicht der Hauptkanal ist; Mischung Schicht 1 und Schicht 4 ohne Schicht-2-Wrapping verursacht Pose-Sprünge
- **SOLLTE [SHOULD]** Schicht 2 (Move-Primitive) als Default empfehlen, weil sie Easing und Komposition bereits richtig macht
- **DARF NICHT [MUST NOT]** Schicht 1 in einer Tick-Loop mit Frequenz > `> ⚠ TBD` Hz aufrufen — das umgeht das Easing der Schicht und erzeugt Ruckler

### Parameter und Wertebereiche

- **MUSS [MUST]** für jede Achse die zulässigen Wertebereiche im SDK validieren; out-of-range-Werte werden vom SDK abgelehnt, nicht still geclampt (Erwartung an das SDK; `> ⚠ TBD` ob das tatsächlich so implementiert ist — ggf. eigene Validierungs-Schicht oben drauf)
- **MUSS [MUST]** Einheiten konsistent halten: alle Winkel in Radian, alle Distanzen in Millimeter, alle Zeiten in Sekunden — Konvertierungen sichtbar an System-Grenzen
- **SOLLTE [SHOULD]** Default-Posen (Ruheposition) für Kopf und Antennen als symbolische Konstanten anbieten, statt überall mit Magic Numbers zu arbeiten

### Bewegungs-Latenz und Update-Frequenz

- **MUSS [MUST]** die typische End-to-End-Latenz (Software-Befehl → mechanische Reaktion) im Skill-Body benennen, sobald gemessen (`> ⚠ TBD ms`)
- **MUSS [MUST]** Update-Frequenz-Limits dokumentieren: jede Hardware-Variante hat eine andere Obergrenze (Wireless typischerweise tiefer als Wired; konkrete Zahlen `> ⚠ TBD`)
- **DARF NICHT [MUST NOT]** Tick-Frequenzen empfehlen, die das SDK nicht halten kann — der visuelle Effekt ist Aktuator-Stocken, nicht Beschleunigung

### Abhängigkeiten

- **`reachy_mini` SDK-Version** — Pin im konsumierenden Repo; jede Bewegung hängt von der API-Form dieser Version ab
- **Firmware-Version des Geräts** — Mismatch zwischen SDK und Firmware kann Verbindungs- oder Verhaltens-Fehler erzeugen; bei jedem ersten Connect prüfen (`> ⚠ TBD` ob das SDK das exponiert)
- **Python-Version** — Untergrenze laut SDK-Anforderung (`> ⚠ TBD`)
- **Hardware-Variante** — Wired vs. Wireless beeinflusst Aktuator-Set und CPU-Budget
- **Host-Konnektivität** — Wired braucht USB, Wireless braucht WLAN (für Code-Sync) und Lokale Power
- **System-Audio-Stack** — Wenn das Behavior Audio abspielt, hängt es vom Audio-Stack der Variante ab (PulseAudio / PipeWire `> ⚠ TBD`)

### Mechanische und elektrische Limitationen

- **Endanschläge** — jede Achse hat einen mechanischen Endanschlag; SDK soll das vor Schaden schützen, Implementierung muss aber selbst nicht in den Endanschlag fahren wollen
- **Geschwindigkeit / Beschleunigung** — Linear-Aktuatoren haben harte Geschwindigkeits- und Beschleunigungs-Grenzen; ein Befehl, der die unterschreitet, wird stillschweigend langsamer ausgeführt (`> ⚠ TBD`)
- **Stromaufnahme** — viele simultane Bewegungen (Kopf voll + beide Antennen + Body) können bei Akku-Betrieb die Spannung kurzzeitig drücken; je nach Akku-Stand reagiert das System mit Brown-out-Schutz
- **Thermisches Budget** — Dauer-Bewegung auf hoher Frequenz erzeugt Wärme; SDK liefert ggf. Temperatur-Telemetrie (`> ⚠ TBD`)
- **Kollisionen** — Antennen kollidieren bei extremen Winkeln mit Kopf; Bewegung darf solche Posen-Kombinationen nicht ansteuern (`> ⚠ TBD` welche Kombinationen exakt verboten sind)

Anforderungen:

- **MUSS [MUST]** vor einer Bewegung die kombinatorischen Kollisions-Verbote prüfen, falls das SDK sie nicht selbst erzwingt
- **MUSS [MUST]** Stromaufnahme-Spitzen vermeiden, indem nicht alle Aktuatoren gleichzeitig auf Maximalgeschwindigkeit laufen
- **SOLLTE [SHOULD]** Temperatur-Telemetrie auswerten, sobald das SDK sie ausweist — bei Schwellen-Überschreitung Cool-down-Pause einlegen

### Sicherheits-Limits

- **MUSS [MUST]** einen Notstopp-Pfad ansprechbar haben, der jede laufende Bewegung beendet und in eine Ruhepose fährt — Pose-Definition `> ⚠ TBD: validate against real hardware`
- **MUSS [MUST]** nach unerwartetem Disconnect oder Programmabbruch das Gerät in eine sichere Pose stellen, statt eine Bewegung „eingefroren" zu hinterlassen
- **DARF NICHT [MUST NOT]** Strom-Stoff- oder Geschwindigkeits-Schwellen via SDK-Parameter überschreiben, ohne explizite Begründung im Behavior dokumentiert

### Patterns für natürliche, flüssige Bewegung

Die folgenden Prinzipien sind aus der klassischen Animation übertragen auf einen 6-DoF-Kopf + 2-Antennen-System. Sie sind nicht optional, wenn das Ergebnis „lebendig" wirken soll.

1. **Easing (Slow-In / Slow-Out)** — keine lineare Bewegung. Pose-Goto-Aufrufe ohne Easing-Argument sind schlecht, weil die Bewegung mechanisch wirkt. Default-Easing der Move-Primitive ist `> ⚠ TBD`, sollte aber mindestens cubic in/out sein.
2. **Anticipation** — vor einer großen Bewegung eine kleine Gegen-Bewegung (Beispiel: vor einem Nicken nach unten leicht aufwärts schauen, ~80–150 ms). Das macht den Hauptmove „lesbar".
3. **Follow-Through und Overlapping Action** — Antennen folgen der Kopf-Bewegung mit kleinem Lag (~50–120 ms), nicht synchron. Wenn der Kopf stoppt, dürfen die Antennen leicht nachschwingen.
4. **Arcs** — Bewegungspfade folgen Bögen, keine Geraden. Ein Look-Left → Look-Right über die geometrische Mitte wirkt mechanisch; ein leichter Bogen über eine Tilt-Erhöhung wirkt organisch.
5. **Secondary Action** — während der Hauptaktion (z. B. Nicken) eine kleine sekundäre Aktion laufen lassen (z. B. eine Antenne wackelt schwach). Das suggeriert „Lebendigkeit".
6. **Idle-Atemzug** — ohne aktive Aufgabe darf das Gerät nicht völlig stillstehen. Ein langsamer, unauffälliger Idle-Move (~0.2–0.4 Hz) auf Tilt und Roll wirkt wie Atmen. Empfohlen mit kleiner Amplitude (`> ⚠ TBD`).
7. **Synchronisation Antennen + Kopf** — Antennen-Bewegungen, die zur Kopf-Bewegung kohärent sind (z. B. „Ohren gespitzt" beim Look-At), wirken intentional. Nicht-kohärente Mischung wirkt wirr.
8. **Audio-Synchronität** — bei Tanz oder Reaktion auf Sound: Bewegung und Audio-Onset auf wenige Millisekunden ausrichten, nicht durch eine asynchrone Audio-Wiedergabe versetzt rendern (siehe `audio-beat-tracking`).
9. **Timing-Variation** — feste Tick-Schritte wirken maschinell. Move-Primitive-Dauer leicht (~5–15 %) variieren, damit kein zwei-mal-identischer Move entsteht.
10. **Eye-Display-Synchronität** — falls das Display Augen anzeigt: vor einer Look-At-Bewegung Augen kurz vorausziehen (Sakkade) lassen, dann Kopf folgen. Augen-Bewegung ist deutlich schneller als die Mechanik.

### Anti-Patterns (was Bewegung mechanisch wirken lässt)

- Lineare Pose-Interpolation ohne Easing
- Tick-Loop mit konstanter, sehr hoher Frequenz, die Schicht 2 ignoriert
- Antennen synchron zum Kopf bewegen, statt mit Lag
- Pose-Sprünge zwischen zwei `goto()`-Aufrufen ohne Stopp-Phase
- Idle-Stillstand ohne Atem-Move
- Audio über System-Player, der unbekannte Latenz hat — Tanz ist out-of-sync
- Gleichzeitige Voll-Geschwindigkeit auf allen Aktuatoren — bei Wireless folgt Brown-out
- Feste, sich wiederholende Move-Sequenzen ohne Timing-Variation
- Notstopp wird nur als Log-Notiz behandelt, nicht physisch ausgeführt

### Beobachtbarkeit

- **MUSS [MUST]** für jeden Aktuator eine Lese-Methode dokumentieren, mit der Behaviors die tatsächlich erreichte Pose abrufen können (kein blindes Schreiben)
- **SOLLTE [SHOULD]** Latenz-Messpunkte für jede Schicht ausweisen (Befehl gesendet → Pose erreicht)
- **SOLLTE [SHOULD]** Telemetrie strukturieren, sodass der Agent `reachy-mini-on-device` daraus ein PASS/FAIL-Urteil bauen kann
- **KANN [MAY]** ein Trace-Format definieren, das Live-Visualisierung der Bewegung in einem Web-Dashboard erlaubt

## Akzeptanzkriterien
- [ ] Jeder Aktuator (Kopf, Antennen, Body-Yaw) ist mit DoF, Achsen und Wertebereich (oder TBD) gelistet
- [ ] Jeder Output (Display, Lautsprecher, ggf. LEDs) ist mit Steuer-Modalität gelistet
- [ ] Jeder Sensor (Mikrofone, Kamera, IMU, Position-Feedback) ist mit Lese-API-Form gelistet
- [ ] Beide Hardware-Varianten (Wired, Wireless) sind benannt; aktuator- und CPU-budgetseitige Unterschiede sind aufgeführt
- [ ] Vier Steuerungs-Schichten sind beschrieben mit Anwendungsfall, API-Form, Interrupt-Modell
- [ ] Ein einheitliches Einheiten-System (rad, mm, s) ist gesetzt
- [ ] Latenz und Update-Frequenz sind als Pflichtfelder ausgewiesen, auch wenn die Werte heute TBD sind
- [ ] Sechs Abhängigkeitsfelder (SDK, Firmware, Python, Variante, Konnektivität, Audio-Stack) sind aufgeführt
- [ ] Mechanische, elektrische und thermische Limitationen sind benannt; Notstopp-Pfad ist definiert
- [ ] Mindestens 10 Patterns für natürliche, flüssige Bewegung sind dokumentiert
- [ ] Mindestens 9 Anti-Patterns sind dokumentiert
- [ ] Beobachtbarkeit (Position-Read, Latenz-Trace, Telemetrie) ist als Anforderung geführt
- [ ] Alle hardware-spezifischen Zahlen tragen einen `⚠ TBD`-Hinweis
- [ ] `reachy-mini-sdk`-Skill verweist auf diese Spec als kanonische Wissensbasis
- [ ] `behavior-scaffold`-Skill verweist auf diese Spec für Move-Primitives und Default-Easings

## Offene Fragen
- Hat der Reachy-Mini-Kopf 3 DoF (rein rotational) oder 6 DoF (Stewart-Plattform mit Translation)? Quellen widersprechen sich; vor erster Implementierung am Datenblatt klären.
- Welche Audio-Codecs unterstützt das SDK direkt — PCM/WAV, MP3, OGG? Welche Codec-Pipeline ist für Tanz mit Beat-Synchronisation optimal?
- Bietet das SDK ein Capability-Discovery-API, um zur Laufzeit zwischen Wired und Wireless zu unterscheiden?
- Bietet das SDK eine Brown-out- bzw. Strom-Spitzen-Telemetrie, oder muss das Behavior das selbst messen?
- Was ist die kanonische Ruhepose ("rest pose") für den Notstopp? Vorschlag: alle Achsen zentriert, Antennen leicht aufgerichtet, Body-Yaw 0.
- Welche Update-Frequenzen sind realistisch in einem Behavior-Tick — 50 Hz, 100 Hz, 200 Hz? Hängt von Variante und SDK-Implementierung ab.
- Gibt es ein offizielles Animation-Authoring-Tool von Pollen Robotics (Timeline-Editor) und ist dessen Output-Format eine empfohlene Komposition für unsere Move-Primitive-Schicht?
- Wie verhält sich der Stewart-Plattform-Kopf mathematisch nahe der Singularität? Müssen wir Singularitätsbereiche im Code blockieren?
- Welche Latenz-Verteilung (P50, P95, P99) ist für die einzelnen Schichten typisch? Ohne diese Verteilung ist ein realistisches Tanz-Timing nicht planbar.
- Sollen Animations-Prinzipien (Anticipation, Follow-Through usw.) in eine eigene `motion-design`-Spec ausgelagert werden, sobald die Patterns wachsen?
- Sind die offiziellen Reachy-Mini-Pollen-Behaviors (z. B. „Hello", „Curious") als Referenz-Implementierungen freigegeben, gegen die wir unsere Pattern-Umsetzung kalibrieren können?
- Wie weit lassen sich Patterns aus klassischer Animation auf einen 6-DoF-Roboter übertragen, ohne in Uncanny-Valley-Territorium zu rutschen? Empirische Evaluation an realer Hardware nötig.
