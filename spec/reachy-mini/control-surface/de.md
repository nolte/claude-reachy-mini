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

Reachy Mini ist ein Desktop-Roboter von Pollen Robotics / Hugging Face, der in drei Plattformen ausgeliefert wird: **Reachy Mini** (Wireless-Variante mit eingebautem Raspberry Pi 5 und Akku), **Reachy Mini Lite** (an einen Host-Rechner gebunden, weniger On-Robot-Compute) und **Simulation** (rein softwareseitig, gleiche `ReachyMini`-API ohne reale Motoren). Die softwareseitige Steuerung läuft über das Python-SDK `reachy_mini`; eine `ReachyMini`-Instanz ist der zentrale Einstiegspunkt und wird typischerweise als Context Manager (`with ReachyMini() as mini:`) genutzt. Sie aggregiert Bewegungs-, Sensor- und Media-Funktionen direkt als eigene Methoden und Properties (`mini.goto_target(...)`, `mini.imu`, `mini.media`); es gibt keine separaten `head` / `antennas` Sub-Objekte.

Kanonische Quellen: gehostete Doku <https://huggingface.co/docs/reachy_mini/>, Klassen-Source <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>.

### Hardware-Inventar — Aktuatoren

Steuerbare mechanische Achsen (verifiziert anhand `ReachyMini`-Klassen-API <https://huggingface.co/docs/reachy_mini/API/reachymini>):

| Subsystem | DoF | Repräsentation | Schreibe-API | Lese-API | Wertebereich | Einheit |
|---|---|---|---|---|---|---|
| Kopf | 6 (Stewart-Plattform mit Translation + Rotation) | 4×4-Transform-Matrix; Builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees, mm)` | `goto_target(head=...)`, `set_target(head=...)`, `set_target_head_pose(pose)` | `get_current_head_pose() -> np.ndarray` (4×4) | `> ⚠ TBD` pro Achse | rad / mm |
| Antennen (Paar) | 2 (1 DoF je Antenne) | `List[float]` mit zwei Joint-Winkeln (links, rechts) | `goto_target(antennas=...)`, `set_target(antennas=...)`, `set_target_antenna_joint_positions(antennas)` | `get_present_antenna_joint_positions() -> List[float]` | `> ⚠ TBD` | rad |
| Body-Yaw | 1 | `float` | `goto_target(body_yaw=...)`, `set_target(body_yaw=...)`, `set_target_body_yaw(value)`, `set_automatic_body_yaw(enabled)` | als Teil von `get_current_joint_positions()` | `> ⚠ TBD` | rad |

Konkrete Schreibe-API-Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>. Pose-Builder: <https://huggingface.co/docs/reachy_mini/API/tools>.

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

| Subsystem | Eigenschaft | Lese-API | Doku |
|---|---|---|---|
| Mikrofon-Array | mehrere Mikrofone, Direction-of-Arrival möglich | über `mini.media` (`MediaManager`) | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), Beispiel [`sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Kamera | Weitwinkel, Auflösung `> ⚠ TBD`, Framerate `> ⚠ TBD` | über `mini.media.camera` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), Beispiel [`take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| IMU (vorhanden, bestätigt) | Accelerometer, Gyroscope, Quaternion, Temperatur | `mini.imu` (Property → `Dict \| None`) | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini), Beispiel [`imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| Position-Feedback Kopf | aktuelle 4×4-Pose | `mini.get_current_head_pose() -> np.ndarray` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Position-Feedback Antennen + Joints | Joint-Winkel | `mini.get_current_joint_positions()`, `mini.get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Media-Release-/-Acquire | Kamera/Mic an externen Code abgeben | `mini.release_media()`, `mini.acquire_media()`, Property `mini.media_released` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |

Anforderungen:

- **MUSS [MUST]** für jeden Aktuator dokumentieren, welche Methode den aktuellen Zustand zurückliefert; Behaviors müssen Position lesen können, ohne den Aktuator zu blockieren
- **MUSS [MUST]** klarstellen, ob Sensor-Streams blockierend oder als Async-Iteratoren bereitgestellt werden
- **DARF NICHT [MUST NOT]** annehmen, dass eine Sensor-Lesung kostenlos ist — die Lese-Frequenz unterliegt einer Limitation (siehe Latenz-Abschnitt)

### Hardware-Plattformen und ihre Konsequenzen

Pollen Robotics liefert die `ReachyMini`-API auf drei Plattformen aus, mit denselben Methoden, aber unterschiedlichem Compute- und Aktuator-Profil:

- **Reachy Mini** (Wireless) — eingebauter Raspberry Pi 5 und Akku; vollständiger Feature-Satz; CPU- und Energie-Budget des Roboters sind das Limit. Doku: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/get_started>
- **Reachy Mini Lite** — an einen Host-Rechner gebunden; reduziertes On-Robot-Compute, der Host übernimmt heavy-lift; ideal für Entwicklung und energieintensive Workloads. Doku: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini_lite/get_started>
- **Simulation** — softwareseitig, gleiche `ReachyMini`-API ohne reale Motoren; nutzbar für CI, Tests, Code-Erprobung ohne Gerät. Doku: <https://huggingface.co/docs/reachy_mini/platforms/simulation/get_started>

Anforderungen:

- **MUSS [MUST]** im Code sichtbar machen, gegen welche Plattform eine Bewegung geschrieben ist; eine Bewegung mit hoher Update-Frequenz, die für Wireless ausgelegt ist, darf nicht ungeprüft auf Lite oder Simulation laufen
- **SOLLTE [SHOULD]** einen Weg vorsehen, die Plattform zur Laufzeit zu erkennen (Capability-Discovery, `> ⚠ TBD` ob das SDK eine direkte Variant-Property exponiert; Konstruktor-Argument `use_sim` ist verifiziert verfügbar)
- **MUSS [MUST]** Bewegungen, die Hardware-Eigenschaften voraussetzen (z. B. echte Audio-Wiedergabe, IMU-Werte), in der Simulation explizit gegenüber dem Caller signalisieren statt stillschweigend No-Op zu fahren

### Steuerungs-Schichten

Vier Schichten, von direkt-schreibend (low-level) zu narrativ (high-level), mit verifizierten Methoden-Namen:

1. **Direct-Set** — Target wird sofort geschrieben, keine Interpolation
   - Methoden: `set_target(head, antennas, body_yaw)`, `set_target_head_pose(pose)`, `set_target_antenna_joint_positions(antennas)`, `set_target_body_yaw(value)`
   - Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>
   - Anwendung: nur wenn Easing aus einer höheren Schicht oder einer eigenen Trajektorie kommt — sonst mechanisch wirkende Bewegung

2. **Goto-Target** — smooth-goto über benannte Dauer, SDK übernimmt Interpolation
   - Methode: `goto_target(head, antennas, duration, method, body_yaw)` mit `method: InterpolationTechnique`
   - Pose-Builder: `create_head_pose(...)` aus `reachy_mini.utils`
   - Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>, Source: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/goto.py>
   - Spielfeld: <https://huggingface.co/docs/reachy_mini/examples/goto_interpolation_playground>
   - Granularität: ein Aufruf = eine Bewegung. Interrupt-Modell `> ⚠ TBD: validate against real hardware`.

3. **Move-Komposition** — wiederverwendbare Bewegungen als Subklasse der ABC `Move` mit `duration` und `evaluate(t) -> (head_pose | None, antennas | None, body_yaw | None)`
   - Source: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py>
   - Ausführung: `mini.async_play_move(move, play_frequency, initial_goto_duration, sound)` (non-blocking) oder `mini.play_move(...)` (blocking); `mini.cancel_move()` stoppt laufende Wiedergabe
   - Beispiele: <https://huggingface.co/docs/reachy_mini/examples/recorded_moves>, <https://huggingface.co/docs/reachy_mini/examples/sequence>
   - Easing / Komposition: keine eingebauten Easing-Primitives in `Move` selbst — Subklassen liefern eigene Trajektorien; sequentielle/parallele Komposition wird durch Konsumenten gebaut

4. **High-Level-Behaviors** — fertige, vom SDK gelieferte Methoden, die Pose + Sound + Animation komponieren
   - Methoden: `wake_up()`, `goto_sleep()`, `look_at_image(u, v, duration, perform_movement)`, `look_at_world(x, y, z, duration, perform_movement)`
   - Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>, Beispiel: <https://huggingface.co/docs/reachy_mini/examples/look_at>
   - Bevorzugen, wenn die Aufgabe semantisch dem Methoden-Namen entspricht — keine eigene Re-Implementierung

Anforderungen:

- **MUSS [MUST]** in jedem Behavior klar dokumentieren, welche Schicht den Hauptkanal stellt
- **SOLLTE [SHOULD]** Schicht 4 (High-Level) bevorzugen, wenn die Aufgabe einer der gelieferten Methoden entspricht — keine eigene `look_at`-Implementierung neben der offiziellen
- **SOLLTE [SHOULD]** für wiederverwendbare Bewegungen Schicht 3 (Move-Subklasse) wählen — sie liefert klare Lifecycle-Semantik (`duration`, `evaluate`) und ist mit Audio-Synchronisation kompatibel
- **DARF NICHT [MUST NOT]** Schicht 1 (Direct-Set) in einem Tick-Loop ohne eigene Trajektorien-Glättung aufrufen — das erzeugt Ruckler

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

### API-Referenz-Übersicht (kanonische Doku-Links)

| Bereich | API-Element | Doku-Link |
|---|---|---|
| Konstruktion | `ReachyMini(robot_name, host, port, connection_mode, spawn_daemon, use_sim, timeout, automatic_body_yaw, log_level, media_backend, localhost_only)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Pose-Builder | `create_head_pose(...)` aus `reachy_mini.utils` | [`API/tools`](https://huggingface.co/docs/reachy_mini/API/tools) |
| Smooth-Goto | `goto_target(head, antennas, duration, method, body_yaw)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Direct-Set | `set_target(...)`, `set_target_head_pose(...)`, `set_target_antenna_joint_positions(...)`, `set_target_body_yaw(...)`, `set_automatic_body_yaw(...)` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| State-Read | `get_current_head_pose()`, `get_current_joint_positions()`, `get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| IMU | Property `mini.imu` | [`examples/imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| High-Level | `wake_up()`, `goto_sleep()`, `look_at_image(u, v, duration)`, `look_at_world(x, y, z, duration)` | [`examples/look_at`](https://huggingface.co/docs/reachy_mini/examples/look_at) |
| Move-ABC | `Move` (Source: [`motion/move.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/move.py)), `GotoMove` (Source: [`motion/goto.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/motion/goto.py)), `InterpolationTechnique` | [`API/motion`](https://huggingface.co/docs/reachy_mini/API/motion) |
| Move-Wiedergabe | `play_move(move, play_frequency, initial_goto_duration, sound)`, `async_play_move(...)`, `cancel_move()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Recording | `start_recording()`, `stop_recording() -> Optional[List[Dict]]` | [`examples/recorded_moves`](https://huggingface.co/docs/reachy_mini/examples/recorded_moves) |
| Motoren | `enable_motors(ids)`, `disable_motors(ids)`, `enable_gravity_compensation()`, `disable_gravity_compensation()` | [`examples/reachy_compliant_demo`](https://huggingface.co/docs/reachy_mini/examples/reachy_compliant_demo) |
| Media-Manager | Property `mini.media`, `release_media()`, `acquire_media()`, Property `media_released` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture) |
| REST / WebSocket | Daemon-API für Nicht-Python-Clients | [`API/rest-api`](https://huggingface.co/docs/reachy_mini/API/rest-api), [`API/daemon`](https://huggingface.co/docs/reachy_mini/API/daemon), [OpenAPI](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/openapi.json) |
| Audio | `mini.media.audio.*` | [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), [`examples/sound_play`](https://huggingface.co/docs/reachy_mini/examples/sound_play), [`examples/sound_record`](https://huggingface.co/docs/reachy_mini/examples/sound_record), [`examples/sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Kamera | `mini.media.camera.*` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`examples/take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| Quickstart | offizieller Einstieg | [`SDK/quickstart`](https://huggingface.co/docs/reachy_mini/SDK/quickstart) |
| Core-Concepts | übergeordnete Konzepte | [`SDK/core-concept`](https://huggingface.co/docs/reachy_mini/SDK/core-concept) |

## Akzeptanzkriterien
- [ ] Jeder Aktuator (Kopf, Antennen, Body-Yaw) ist mit DoF, Achsen und Wertebereich (oder TBD) gelistet
- [ ] Jeder Output (Display, Lautsprecher, ggf. LEDs) ist mit Steuer-Modalität gelistet
- [ ] Jeder Sensor (Mikrofone, Kamera, IMU, Position-Feedback) ist mit Lese-API-Form gelistet
- [ ] Drei Plattformen (Reachy Mini, Reachy Mini Lite, Simulation) sind benannt; aktuator- und CPU-budget-seitige Unterschiede sind aufgeführt
- [ ] API-Referenz-Übersicht enthält für jeden steuerbaren Bereich einen direkten Link auf die hosted Doku oder das Source-Modul
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
- ~~Hat der Reachy-Mini-Kopf 3 DoF oder 6 DoF?~~ **Beantwortet**: 6 DoF (Stewart-Plattform), Pose ist 4×4-Transform-Matrix; Builder `create_head_pose(x, y, z, roll, pitch, yaw, …)` aus `reachy_mini.utils`.
- Welche Audio-Codecs unterstützt das SDK direkt — PCM/WAV, MP3, OGG? Welche Codec-Pipeline ist für Tanz mit Beat-Synchronisation optimal? Klären über [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture).
- Bietet das SDK ein Capability-Discovery-API, um zur Laufzeit zwischen Reachy Mini, Reachy Mini Lite und Simulation zu unterscheiden? Konstruktor-Argument `use_sim` ist verifiziert; explizite Variant-Property auf der Instanz `> ⚠ TBD`.
- Bietet das SDK eine Brown-out- bzw. Strom-Spitzen-Telemetrie, oder muss das Behavior das selbst messen?
- Was ist die kanonische Ruhepose ("rest pose") für den Notstopp? Vorschlag: alle Achsen zentriert, Antennen leicht aufgerichtet, Body-Yaw 0.
- Welche Update-Frequenzen sind realistisch in einem Behavior-Tick — 50 Hz, 100 Hz, 200 Hz? Hängt von Variante und SDK-Implementierung ab.
- Gibt es ein offizielles Animation-Authoring-Tool von Pollen Robotics (Timeline-Editor) und ist dessen Output-Format eine empfohlene Komposition für unsere Move-Primitive-Schicht?
- Wie verhält sich der Stewart-Plattform-Kopf mathematisch nahe der Singularität? Müssen wir Singularitätsbereiche im Code blockieren?
- Welche Latenz-Verteilung (P50, P95, P99) ist für die einzelnen Schichten typisch? Ohne diese Verteilung ist ein realistisches Tanz-Timing nicht planbar.
- Sollen Animations-Prinzipien (Anticipation, Follow-Through usw.) in eine eigene `motion-design`-Spec ausgelagert werden, sobald die Patterns wachsen?
- Sind die offiziellen Reachy-Mini-Pollen-Behaviors (z. B. „Hello", „Curious") als Referenz-Implementierungen freigegeben, gegen die wir unsere Pattern-Umsetzung kalibrieren können?
- Wie weit lassen sich Patterns aus klassischer Animation auf einen 6-DoF-Roboter übertragen, ohne in Uncanny-Valley-Territorium zu rutschen? Empirische Evaluation an realer Hardware nötig.
