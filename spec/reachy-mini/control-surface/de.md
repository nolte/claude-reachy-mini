# Steuerungs-Oberfläche und Bewegungs-Design des Reachy Mini

Status: draft

## Kontext
Wer in diesem Plugin Skills, Behaviors oder Agents schreibt, braucht eine kanonische Referenz dafür, *welche* Elemente am Reachy Mini überhaupt steuerbar sind, *wie* sie angesprochen werden und *unter welchen Grenzen* daraus natürlich wirkende Bewegungen komponiert werden. Ohne diese Spezifikation rekonstruiert jede Implementierung die Hardware-Realität neu — meistens aus halbgaren Trainings-Stichproben, was beim ersten echten Gerätekontakt entweder Hardware gefährdet oder zu mechanisch wirkenden Bewegungen führt. Diese Spec konsolidiert die öffentlich bekannten Hardware- und SDK-Eigenschaften des Reachy Mini, beschreibt jede Steuerungs-Schicht des SDKs, und gibt für jede mechanische und elektrische Limitation an, wie sie die Bewegungs-Komposition formt. Sie ist die normative Wissensbasis, gegen die der `reachy-mini-sdk`-Skill seine Snippets validiert und gegen die der `app-scaffold`-Skill seine Schablonen anpasst.

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

Reachy Mini ist ein Desktop-Roboter von Pollen Robotics / Hugging Face, der in drei Plattformen ausgeliefert wird: **Reachy Mini** (Wireless-Variante mit eingebautem Raspberry Pi 4 Compute Module und LiFePO4-Akku), **Reachy Mini Lite** (an einen Host-Rechner per USB-C gebunden, externe Stromversorgung) und **Simulation** (rein softwareseitig, gleiche `ReachyMini`-API ohne reale Motoren). Die softwareseitige Steuerung folgt einem **Client-Server-Modell**: ein lokaler Daemon hält die Hardware-Verbindung und die Sicherheitsschicht, der SDK-Client (das Python-Paket `reachy_mini`) spricht den Daemon über REST/WebSocket an. Eine `ReachyMini`-Instanz ist der zentrale Einstiegspunkt und wird typischerweise als Context Manager (`with ReachyMini() as mini:`) genutzt. Sie aggregiert Bewegungs-, Sensor- und Media-Funktionen direkt als eigene Methoden und Properties (`mini.goto_target(...)`, `mini.imu`, `mini.media`); es gibt **keine** separaten `head` / `antennas` / `eyes` / `display` Sub-Objekte.

Bewegungen werden in zwei Koordinatensystemen ausgedrückt — **Head Frame** (lokal an der Stewart-Plattform) und **World Frame** (Welt-relativ, z. B. für `look_at_world(...)`). Das SDK bringt eingebaute **Safety-Limits** mit, die Selbstkollision und Hardware-Schaden verhindern.

Kanonische Quellen: gehostete Doku <https://huggingface.co/docs/reachy_mini/>, Core-Concepts <https://huggingface.co/docs/reachy_mini/SDK/core-concept>, Klassen-Source <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>.

### Hardware-Inventar — Aktuatoren

Steuerbare mechanische Achsen (verifiziert anhand `ReachyMini`-Klassen-API <https://huggingface.co/docs/reachy_mini/API/reachymini>):

| Subsystem | DoF | Motoren | Repräsentation | Schreibe-API | Lese-API | Pose-/Joint-Limits (verifiziert) | Plattformen |
|---|---|---|---|---|---|---|---|
| Kopf | 6 (Stewart-Plattform: 3 Rotation + 3 Translation) | 6× Dynamixel XL330-M288-T | 4×4-Transform-Matrix; Builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees, mm)` | `goto_target(head=...)`, `set_target(head=...)`, `set_target_head_pose(pose)` | `get_current_head_pose() -> np.ndarray` (4×4) | Pitch und Roll: ±90° (Upright-Constraint, [`analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)); Yaw relativ zum Body: max ±65° (`max_relative_yaw`); Translation x/y/z innerhalb des IK-erreichbaren Volumens (`head_z_offset` aus `kinematics_data.json`) | alle |
| Antennen (Paar) | 2 (1 DoF je Antenne) | 2× Dynamixel XL330-M077-T | `List[float]` mit zwei Joint-Winkeln (links, rechts) | `goto_target(antennas=...)`, `set_target(antennas=...)`, `set_target_antenna_joint_positions(antennas)` | `get_present_antenna_joint_positions() -> List[float]` | je Antenne: -π bis +π rad (volle Rotation), Geschwindigkeits-Limit 8 rad/s, Effort-Limit 10 N·m ([URDF](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)) | alle |
| Body-Yaw | 1 (Basis-Rotation) | 1× custom Dynamixel XC330-M288-PG | `float` | `goto_target(body_yaw=...)`, `set_target(body_yaw=...)`, `set_target_body_yaw(value)`, `set_automatic_body_yaw(enabled)` | als Teil von `get_current_joint_positions()` | ±160° (`max_body_yaw=np.deg2rad(160)`, [`analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)) | alle (Wireless **und** Lite, in Simulation als Soft-State) |

Stewart-Plattform-Joint-Limits (low-level, vom IK abstrahiert; aus [`robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)): jeder der sechs Aktuatoren `stewart_1`..`stewart_6` hat einen Wertebereich zwischen ungefähr -1,396 rad (-80°) und +1,396 rad (+80°) — asymmetrisch je Joint —, Velocity-Limit 8 rad/s, Effort-Limit 10 N·m. Diese Werte sind die **Hardware-Grenze**; die effektive Head-Pose-Erreichbarkeit ist enger und wird vom IK durchgesetzt.

**Nominal-Operations-Range** (offizielles Hardware-Datasheet, Bild `dof_table.png` auf [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)): die hier genannten Bereiche sind die *empfohlene* Operations-Range, gegen die Behaviors komponiert werden sollten. Sie sind enger als die obigen kinematischen Maxima und stellen sicher, dass das Gerät die Pose zuverlässig erreicht.

| Achse | Min | Max |
|---|---|---|
| Tx (Kopf) | −1,5 cm | +2,5 cm |
| Ty (Kopf) | −4 cm | +4 cm |
| Tz (Kopf) | −4 cm | +2,5 cm |
| Rx (Head-Roll) | −40° | +40° |
| Ry (Head-Pitch) | −40° | +40° |
| Rz (Head-Yaw) | −60° | +60° |
| Rz (Body-Yaw) | −155° | +155° |
| R (Antenne rechts) | −180° | +180° |
| R (Antenne links) | −180° | +180° |

**Motor-IDs am Dynamixel-Bus** (aus `motors_detail.png` derselben Hardware-Seite) — direkt verwendbar mit `enable_motors(ids)` / `disable_motors(ids)`:

| Subsystem | Bezeichner | ID |
|---|---|---|
| Body-Yaw | Motor F | 10 |
| Stewart 1 | Motor 1 | 11 |
| Stewart 2 | Motor 2 | 12 |
| Stewart 3 | Motor 3 | 13 |
| Stewart 4 | Motor 4 | 14 |
| Stewart 5 | Motor 5 | 15 |
| Stewart 6 | Motor 6 | 16 |
| Antenne rechts | — | 17 |
| Antenne links | — | 18 |

> **Hinweis zur Quellen-Konsistenz**: Die offizielle Hardware-Seite enthält zwei textuell-bildliche Inkonsistenzen, bei denen wir dem Begleittext folgen — nicht den Bild-Beschriftungen:
> 1. `motors_detail.png` beschriftet die Antennen-Motoren mit „XL330 M288-T" und den Body-Motor mit „XL330 M288PG-T". Der Begleittext nennt korrekt **XL330-M077-T** (Antennen) bzw. **custom XC330-M288-PG** (Body) und verlinkt jeweils das Robotis-Datasheet.
> 2. Im Bild `electronics.png` heißt die Wireless-Steuerelektronik „Wireless Control Board"; im Begleittext heißt sie „CM4 Controller Board" — beide bezeichnen dasselbe Bauteil.

Konkrete Schreibe-API-Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>. Pose-Builder: <https://huggingface.co/docs/reachy_mini/API/tools>.

Anforderungen an die Inventar-Pflege:

- **MUSS [MUST]** jede Achse mit Bezeichner, Wertebereich, Einheit und Default-Pose dokumentieren, sobald die SDK-Doku verfügbar ist
- **MUSS [MUST]** je Achse ausweisen, ob die SDK-API absolute Targets, inkrementelle Targets, oder beides akzeptiert
- **MUSS [MUST]** Achsen markieren, die nur in einer Plattform existieren (Body-Yaw existiert auf Wireless und Lite — Stewart und Antennen ebenfalls; einzig die IMU-Telemetrie ist Wireless-only)
- **SOLLTE [SHOULD]** je Achse die Geschwindigkeits- und Beschleunigungs-Grenzen ausweisen, die sich aus dem mechanischen Aufbau ergeben (Velocity-Limit 8 rad/s pro aktivem Joint laut URDF; Beschleunigungs-Grenze nicht direkt im URDF — durch Effort 10 N·m und Trägheit beschränkt)

### Hardware-Inventar — Outputs (nicht-mechanisch)

| Subsystem | Eigenschaft | Steuerbarkeit |
|---|---|---|
| Lautsprecher | 5 W @ 4 Ω, ein Stück | Audio-Wiedergabe über `mini.media.audio.*` (asynchrone GStreamer-Pipeline); Push-API erwartet `F32LE`-Samples bei 48 kHz, 2 Channels (Konstanten: [`AudioBase.SAMPLE_RATE`, `AudioBase.CHANNELS`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_base.py)). `play_sound(file=...)` decodiert beliebige Formate via GStreamer `playbin` — WAV, MP3, OGG, FLAC sind dadurch implizit erreichbar. Lautstärke 0–100 ([`SetVolumeCmd`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)) |
| LED-Ring am Mikrofon-Modul | LEDs am ReSpeaker-/Mic-Array-Board; Register `LED_EFFECT`, `LED_BRIGHTNESS`, `LED_GAMMIFY`, `LED_SPEED` | über `audio_control_utils` ([Source](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_control_utils.py)); programmatisch beschreibbar |
| ~~Display / Augen-Bildschirm~~ | **Reachy Mini hat keinen programmatischen Augen-Display.** Die „Augen" sind mechanische 3D-Druckteile (`pp01079_back_big_eye`, `pp01080_back_small_eye`) am Kopfgehäuse — nicht steuerbar. „Expressions" der Pollen-Control-App sind Bewegungs-Kompositionen aus Kopf-Pose + Antennen-Stellung, kein Display-Inhalt. | nicht steuerbar |

Anforderungen:

- Audio-Wiedergabe läuft **asynchron** über eine GStreamer-`appsrc`-Pipeline (verifiziert via [`audio_gstreamer.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_gstreamer.py)); der Behavior-Tick wird nicht blockiert
- **SOLLTE [SHOULD]** Empfehlungen für Audio-Latenz-Mess- und Kompensations-Patterns enthalten, sobald die SDK-Verhaltensweise verifiziert ist
- **MUSS [MUST]** die LED-Ring-Steuerung als reine Begleit-Anzeige modellieren (Status, Audio-Rückmeldung) — nicht als „Augen-Ausdruck", weil sie nicht im Augen-Bereich des Roboters liegt
- **DARF NICHT [MUST NOT]** ein „Augen-Display" als steuerbares Element annehmen oder simulieren — das gibt es im Reachy Mini nicht

### Hardware-Inventar — Sensoren / Inputs

| Subsystem | Eigenschaft | Lese-API | Doku |
|---|---|---|---|
| Mikrofon-Array | 4× PDM MEMS digital, 16 kHz Samplerate (Hardware), -26 dB FS Empfindlichkeit, 64 dBA SNR, Direction-of-Arrival; basiert auf dem Seeed-Studio reSpeaker XMOS XVF3800; Mic-Lautstärke 0–100 ([`SetMicrophoneVolumeCmd`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)) | über `mini.media` (`MediaManager`) | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), [`SDK/media-architecture`](https://huggingface.co/docs/reachy_mini/SDK/media-architecture), Beispiel [`sound_doa`](https://huggingface.co/docs/reachy_mini/examples/sound_doa) |
| Kamera | Raspberry Pi v3 Wide Angle (Sony IMX708, 12 MP, Autofokus, 120° Sichtfeld); montiert in der zentralen Brückenlinse zwischen den (rein dekorativen) Augen — das u/v-Frame in `look_at_image(u, v, …)` bezieht sich auf diese Position, nicht auf die Augen; konkrete Stream-Parameter `> ⚠ TBD: validate against current backend` | über `mini.media.camera` | [`API/media`](https://huggingface.co/docs/reachy_mini/API/media), Beispiel [`take_picture`](https://huggingface.co/docs/reachy_mini/examples/take_picture) |
| IMU (**nur Wireless**) | Accelerometer (`accelerometer: list[float]`), Gyroscope (`gyroscope: list[float]`), Quaternion (`quaternion: list[float]`), Temperatur (`temperature: float`); Daten werden vom Daemon mit 50 Hz publiziert ([`ImuDataMsg`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)). Auf Lite und Simulation gibt `mini.imu` `None` zurück. | `mini.imu` (Property → `Dict \| None`) | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini), Beispiel [`imu`](https://huggingface.co/docs/reachy_mini/examples/imu) |
| Position-Feedback Kopf | aktuelle 4×4-Pose | `mini.get_current_head_pose() -> np.ndarray` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Position-Feedback Antennen + Joints | Joint-Winkel | `mini.get_current_joint_positions()`, `mini.get_present_antenna_joint_positions()` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |
| Media-Release-/-Acquire | Kamera/Mic an externen Code abgeben | `mini.release_media()`, `mini.acquire_media()`, Property `mini.media_released` | [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) |

Anforderungen:

- **MUSS [MUST]** für jeden Aktuator dokumentieren, welche Methode den aktuellen Zustand zurückliefert; Behaviors müssen Position lesen können, ohne den Aktuator zu blockieren
- **MUSS [MUST]** klarstellen, ob Sensor-Streams blockierend oder als Async-Iteratoren bereitgestellt werden
- **DARF NICHT [MUST NOT]** annehmen, dass eine Sensor-Lesung kostenlos ist — die Lese-Frequenz unterliegt einer Limitation (siehe Latenz-Abschnitt)

### Hardware-Plattformen und ihre Konsequenzen

Pollen Robotics liefert die `ReachyMini`-API auf drei Plattformen aus, mit denselben Methoden, aber unterschiedlichem Compute- und Aktuator-Profil:

- **Reachy Mini** (Wireless) — eigene Wireless Control Board (Raspberry Pi 4 Compute Module CM4104016, 4 GB RAM, 16 GB Flash) und Wireless Power Board, LiFePO4-Akku 2000 mAh / 6,4 V / 12,8 Wh mit Schutzfunktionen (Over-Charge, Over-Discharge, Over-Current, Short, Temperatur-Sensor), 2,4–5 GHz Dual-Band-Patch-Antenne (2,79 dBi, omnidirektional); vollständiger Feature-Satz, autark. Rückseiten-Bedienelemente siehe Abschnitt „Rückseiten-Interface". Doku: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/get_started>
- **Reachy Mini Lite** — eigene Lite Control Board und Lite Power Board (kein CM4, kein Akku), USB-C zum Host-Rechner als Daten- und High-Level-Compute-Verbindung, externe 6,8–7,6 V Spannungsversorgung (separater Versorgungs-Anschluss, **nicht** über USB-C); gleiche Aktuator-Ausstattung wie Wireless (Stewart, Antennen, Body-Yaw, Mic-Array, Kamera, Lautsprecher) — nur die IMU-Telemetrie ist Wireless-only. Doku: <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini_lite/get_started>
- **Simulation** — softwareseitig, gleiche `ReachyMini`-API ohne reale Motoren; nutzbar für CI, Tests, Code-Erprobung ohne Gerät. Zwei Pfade: (a) `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:` bootet einen Sim-Daemon im selben Python-Prozess, oder (b) externer Daemon via `reachy-mini-daemon --sim` (oder ohne System-Dependencies `reachy-mini-daemon --mockup-sim --no-media --headless`) und `with ReachyMini() as mini:` aus einem zweiten Prozess. Reines `use_sim=True` ohne `spawn_daemon=True` ist **kein** gültiger Pfad — der Client sucht dann einen externen Daemon und scheitert. Voller `--sim`-Pfad braucht GStreamer-System-Pakete (`gir1.2-gst-plugins-base-1.0` u. a.) und MuJoCo. Doku: <https://huggingface.co/docs/reachy_mini/platforms/simulation/get_started>

Anforderungen:

- **MUSS [MUST]** im Code sichtbar machen, gegen welche Plattform eine Bewegung geschrieben ist; eine Bewegung mit hoher Update-Frequenz, die für Wireless ausgelegt ist, darf nicht ungeprüft auf Lite oder Simulation laufen
- **SOLLTE [SHOULD]** einen Weg vorsehen, die Plattform zur Laufzeit zu erkennen (Capability-Discovery, `> ⚠ TBD` ob das SDK eine direkte Variant-Property exponiert; Konstruktor-Argument `use_sim` ist verifiziert verfügbar)
- **MUSS [MUST]** Bewegungen, die Hardware-Eigenschaften voraussetzen (z. B. echte Audio-Wiedergabe, IMU-Werte), in der Simulation explizit gegenüber dem Caller signalisieren statt stillschweigend No-Op zu fahren

### Mechanisches Profil

Aus dem offiziellen Hardware-Datasheet (Bild `reachy_mini_dimensions.png` auf [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)):

- Maße (extended, Antennen aufrecht): **30 cm Höhe × 20 cm Tiefe × 15,5 cm Breite**
- Body-Höhe in Nominal Position (ohne Antennen): 27,5 cm
- Antennen-Höhe (allein): 14 cm
- Sleeping pos: kompakte Silhouette, Höhe ~15,5 cm (Antennen klappen seitlich an)
- Masse: **1,475 kg**
- Materialien: ABS, PC, Aluminium, Stahl

Behavior-Implikationen:

- **MUSS [MUST]** Look-Targets in `look_at_world(x, y, z, …)` so wählen, dass das Sichtfeld der Brücken-Kamera (120°, in Nominal-Höhe ca. 27,5 cm über Standfläche) das Ziel umfasst — die Augen sind dekorativ, die Kamera liegt zwischen ihnen
- **MUSS [MUST]** Stell-Vorbedingungen auf 20 × 15,5 cm Footprint plus zusätzlichen Bewegungs-Freiraum für die Antennen berechnen
- **SOLLTE [SHOULD]** `goto_sleep()` für Transport- und Aufbewahrungs-Befehle nutzen — das Gerät nimmt dabei die kompakte Sleeping pos an

### Rückseiten-Interface (Wireless)

An der Rückseite des Gehäuses sitzen vier Bedienelemente (Bild `back_interface.png` auf [`platforms/reachy_mini/hardware`](https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware)):

| Element | Funktion | Behavior-Relevanz |
|---|---|---|
| **USB-C** | Peripherie-**Output** (z. B. USB-Stick, USB-Audio-Device); CM4 als USB-Host | **lädt das Gerät nicht** — separater Anschluss für Versorgung |
| **Power supply** | Eigener Lade- / Versorgungs-Anschluss, 6,8–7,6 V | hier kommt der Strom rein, nicht über USB-C |
| **On/Off-Schalter** | Physischer Hardware-Schalter | Hard-Off umgeht den SDK-Notstopp-Pfad — Gerät kann in einer ungesicherten Pose stehen bleiben, wenn vor `goto_sleep()` ausgeschaltet wird |
| **LED Indicator** | Status-LED (Power / Boot / Ready) | einziger Power-/Boot-Status ohne API-Lese-Zugriff — nicht programmatisch abfragbar |

Anforderungen:

- **MUSS [MUST]** Lade-/Versorgungs-Logik vom USB-C-Output trennen — beides sind separate physische Anschlüsse mit unterschiedlicher Funktion
- **MUSS [MUST]** vor einem geplanten Hard-Off `goto_sleep()` aufrufen, um Aktuatoren in eine sichere Pose zu bringen
- **SOLLTE [SHOULD]** im Behavior-Tutorial den On/Off-Schalter und die Status-LED als nicht-API-Pfad benennen, damit konsumierende Skills sie nicht als Lese-Quelle modellieren

### Steuerungs-Schichten

Vier Schichten, von direkt-schreibend (low-level) zu narrativ (high-level), mit verifizierten Methoden-Namen:

1. **Direct-Set** — Target wird sofort geschrieben, keine Interpolation
   - Methoden: `set_target(head, antennas, body_yaw)`, `set_target_head_pose(pose)`, `set_target_antenna_joint_positions(antennas)`, `set_target_body_yaw(value)`
   - Doku: <https://huggingface.co/docs/reachy_mini/API/reachymini>
   - Anwendung: nur wenn Easing aus einer höheren Schicht oder einer eigenen Trajektorie kommt — sonst mechanisch wirkende Bewegung

2. **Goto-Target** — smooth-goto über benannte Dauer, SDK übernimmt Interpolation
   - Methode: `goto_target(head, antennas, duration, method, body_yaw)` mit `method: InterpolationTechnique`
   - Pose-Builder: `create_head_pose(...)` aus `reachy_mini.utils`
   - Verfügbare Interpolations-Modi (verifiziert in [`utils/interpolation.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/utils/interpolation.py)): `LINEAR` (linear), `MIN_JERK` (minimum-jerk-Trajektorie, **Default**), `EASE_IN_OUT` (quadratisches Ease-In/Out), `CARTOON` (elastisches Overshoot)
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

- **MUSS [MUST]** die typische End-to-End-Latenz (Software-Befehl → mechanische Reaktion) gegen die echte Hardware messen, sobald verfügbar (`> ⚠ TBD: validate against real hardware`)
- Daemon publiziert `JointPositionsMsg`, `HeadPoseMsg` und (auf Wireless) `ImuDataMsg` bei **50 Hz** ([`io/protocol.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py)). Behavior-Tick-Loops orientieren sich daran — höhere Tick-Frequenzen geben keinen Mehrwert, weil State-Reads sowieso erst alle 20 ms aktualisiert werden
- Audio-Pipeline-Latenz (GStreamer-Konstanten aus [`audio_gstreamer.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/media/audio_gstreamer.py)): Sink-Buffer 50 ms (`PLAYBACK_SINK_BUFFER_TIME_US = 50000`), Sink-Latenz 5 ms (`PLAYBACK_SINK_LATENCY_TIME_US = 5000`), Gap-Reset 200 ms (`PLAYBACK_GAP_RESET_NS = 200_000_000`)
- **DARF NICHT [MUST NOT]** Tick-Frequenzen über 50 Hz empfehlen — der Daemon hält keinen schnelleren State-Refresh; höhere Frequenzen sind Verschwendung oder erzeugen Aktuator-Stocken
- **SOLLTE [SHOULD]** Audio-und-Bewegungs-Synchronität auf das ~50 ms Audio-Buffer abgleichen, statt auf eine angenommene Null-Latenz

### Abhängigkeiten

- **`reachy_mini` SDK-Version** — Pin im konsumierenden Repo; jede Bewegung hängt von der API-Form dieser Version ab. Aktuell: `reachy_mini==1.7.0` (verifiziert via [`pyproject.toml`](https://github.com/pollen-robotics/reachy_mini/blob/main/pyproject.toml))
- **Firmware-Version des Geräts** — Mismatch zwischen SDK und Firmware kann Verbindungs- oder Verhaltens-Fehler erzeugen; bei jedem ersten Connect prüfen (`DaemonStatus.version` exponiert die Daemon-Version)
- **Python-Version** — `>=3.10` laut SDK-Anforderung (verifiziert via `requires-python` in [`pyproject.toml`](https://github.com/pollen-robotics/reachy_mini/blob/main/pyproject.toml))
- **Hardware-Plattform** — Wireless / Lite / Simulation beeinflusst CPU-Budget und Stromversorgung; Aktuator-Set ist auf Wireless und Lite identisch
- **Host-Konnektivität** — Lite braucht USB-C zum Host plus externe 6,8–7,6 V; Wireless braucht WLAN (für Code-Sync) und arbeitet aus dem Akku; Simulation braucht keinen Host außerhalb des Python-Prozesses
- **System-Audio-Stack** — Audio-Wiedergabe läuft über GStreamer mit OS-spezifischem Backend (PulseAudio / ALSA auf Linux, WASAPI auf Windows, CoreAudio auf macOS); kein Wireless-vs-Lite-Unterschied auf SDK-Ebene

### Mechanische und elektrische Limitationen

- **Endanschläge** — jede Achse hat einen mechanischen Endanschlag; SDK soll das vor Schaden schützen, Implementierung muss aber selbst nicht in den Endanschlag fahren wollen
- **Geschwindigkeit** — pro aktivem Joint 8 rad/s laut [URDF](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf); ein Befehl, der den Joint schneller bewegen würde, wird vom Daemon auf das Limit zurückgeklemmt. **Beschleunigung** — kein direktes URDF-Limit; effektiv durch Effort 10 N·m, Trägheit der Stewart-Plattform und Servo-Charakteristik beschränkt (`> ⚠ TBD: messen wenn Hardware da`)
- **Stromaufnahme** — viele simultane Bewegungen (Kopf voll + beide Antennen + Body) können bei Akku-Betrieb die Spannung kurzzeitig drücken; je nach Akku-Stand reagiert das System mit Brown-out-Schutz
- **Thermisches Budget** — Dauer-Bewegung auf hoher Frequenz erzeugt Wärme; SDK liefert ggf. Temperatur-Telemetrie (`> ⚠ TBD`)
- **Kollisionen** — Antennen kollidieren bei extremen Winkeln mit Kopf; Bewegung darf solche Posen-Kombinationen nicht ansteuern (`> ⚠ TBD` welche Kombinationen exakt verboten sind)

Anforderungen:

- **MUSS [MUST]** vor einer Bewegung die kombinatorischen Kollisions-Verbote prüfen, falls das SDK sie nicht selbst erzwingt
- **MUSS [MUST]** Stromaufnahme-Spitzen vermeiden, indem nicht alle Aktuatoren gleichzeitig auf Maximalgeschwindigkeit laufen
- **SOLLTE [SHOULD]** Temperatur-Telemetrie auswerten, sobald das SDK sie ausweist — bei Schwellen-Überschreitung Cool-down-Pause einlegen

### Sicherheits-Limits

- **MUSS [MUST]** einen Notstopp-Pfad ansprechbar haben, der jede laufende Bewegung beendet und in eine Ruhepose fährt. Kanonische Ruhepose: `INIT_HEAD_POSE = np.eye(4)` (4×4-Identitäts-Matrix, Kopf zentriert) und `INIT_ANTENNAS_JOINT_POSITIONS` (~10° Offset je Antenne, verifiziert in [`reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)). Alternativ steht die `SLEEP_HEAD_POSE` zur Verfügung, die `goto_sleep()` ansteuert
- **MUSS [MUST]** nach unerwartetem Disconnect oder Programmabbruch das Gerät in eine sichere Pose stellen, statt eine Bewegung „eingefroren" zu hinterlassen
- **DARF NICHT [MUST NOT]** Strom-Stoff- oder Geschwindigkeits-Schwellen via SDK-Parameter überschreiben, ohne explizite Begründung im Behavior dokumentiert

### Patterns für natürliche, flüssige Bewegung

Die folgenden Prinzipien sind aus der klassischen Animation übertragen auf einen 6-DoF-Kopf + 2-Antennen-System. Sie sind nicht optional, wenn das Ergebnis „lebendig" wirken soll.

1. **Easing (Slow-In / Slow-Out)** — keine lineare Bewegung. `goto_target` akzeptiert ein `method`-Argument; Default ist `InterpolationTechnique.MIN_JERK` (Minimum-Jerk-Trajektorie — bereits weich-organisch). Verfügbare Modi: `LINEAR`, `MIN_JERK` (empfohlener Default), `EASE_IN_OUT`, `CARTOON`. `LINEAR` nur dann nutzen, wenn die Bewegung explizit mechanisch wirken soll.
2. **Anticipation** — vor einer großen Bewegung eine kleine Gegen-Bewegung (Beispiel: vor einem Nicken nach unten leicht aufwärts schauen, ~80–150 ms). Das macht den Hauptmove „lesbar".
3. **Follow-Through und Overlapping Action** — Antennen folgen der Kopf-Bewegung mit kleinem Lag (~50–120 ms), nicht synchron. Wenn der Kopf stoppt, dürfen die Antennen leicht nachschwingen.
4. **Arcs** — Bewegungspfade folgen Bögen, keine Geraden. Ein Look-Left → Look-Right über die geometrische Mitte wirkt mechanisch; ein leichter Bogen über eine Tilt-Erhöhung wirkt organisch.
5. **Secondary Action** — während der Hauptaktion (z. B. Nicken) eine kleine sekundäre Aktion laufen lassen (z. B. eine Antenne wackelt schwach). Das suggeriert „Lebendigkeit".
6. **Idle-Atemzug** — ohne aktive Aufgabe darf das Gerät nicht völlig stillstehen. Ein langsamer, unauffälliger Idle-Move (~0.2–0.4 Hz) auf Tilt und Roll wirkt wie Atmen. Empfohlen mit kleiner Amplitude (`> ⚠ TBD`).
7. **Synchronisation Antennen + Kopf** — Antennen-Bewegungen, die zur Kopf-Bewegung kohärent sind (z. B. „Ohren gespitzt" beim Look-At), wirken intentional. Nicht-kohärente Mischung wirkt wirr.
8. **Audio-Synchronität** — bei Tanz oder Reaktion auf Sound: Bewegung und Audio-Onset auf wenige Millisekunden ausrichten, nicht durch eine asynchrone Audio-Wiedergabe versetzt rendern (siehe `audio-beat-tracking`).
9. **Timing-Variation** — feste Tick-Schritte wirken maschinell. Move-Primitive-Dauer leicht (~5–15 %) variieren, damit kein zwei-mal-identischer Move entsteht.
10. **Body-Yaw + Kopf-Pose-Synchronität** — wenn `automatic_body_yaw=True` (Default-empfohlen), folgt der Body via IK der Kopf-Pose und nimmt Last vom Stewart-Plattform-Yaw weg. Bei manueller Steuerung beider Achsen Body-Yaw und Kopf-Yaw nicht gegenläufig fahren — die resultierende Verdrehung wirkt unnatürlich. Doku: [`API/reachymini`](https://huggingface.co/docs/reachy_mini/API/reachymini) (`set_automatic_body_yaw`).

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
- [ ] Nominal-Operations-Range aus dem offiziellen Hardware-Datasheet ist zusätzlich zur kinematischen Maximalrange ausgewiesen
- [ ] Motor-IDs (Body, Stewart 1–6, Antennen) sind tabelliert und der API `enable_motors` / `disable_motors` zugeordnet
- [ ] Mechanisches Profil (Maße extended + sleeping pos, Masse, Materialien) ist gelistet
- [ ] Rückseiten-Interface (USB-C, Power supply, On/Off-Schalter, Status-LED) ist als eigene Tabelle dokumentiert
- [ ] Jeder Output (Lautsprecher, LED-Ring am Mic-Modul) ist mit Steuer-Modalität gelistet; das Fehlen eines programmierbaren Augen-Displays ist explizit benannt
- [ ] Jeder Sensor (Mikrofone mit SNR + Chip-Bezeichnung, Kamera mit FoV + Position, IMU, Position-Feedback) ist mit Lese-API-Form gelistet
- [ ] Drei Plattformen (Reachy Mini, Reachy Mini Lite, Simulation) sind benannt; aktuator- und CPU-budget-seitige Unterschiede sind aufgeführt; Lite hat eigene Lite Control + Lite Power Board (kein „dummes Kabel zum Host")
- [ ] Drift-Risiko zwischen offiziellem Begleittext und Bildern ist als Quellen-Hinweis festgehalten
- [ ] API-Referenz-Übersicht enthält für jeden steuerbaren Bereich einen direkten Link auf die hosted Doku oder das Source-Modul
- [ ] Vier Steuerungs-Schichten sind beschrieben mit Anwendungsfall, API-Form, Interrupt-Modell
- [ ] Ein einheitliches Einheiten-System (rad, mm, s) ist gesetzt
- [ ] Latenz und Update-Frequenz sind als Pflichtfelder ausgewiesen, auch wenn die Werte heute TBD sind
- [ ] Sechs Abhängigkeitsfelder (SDK, Firmware, Python, Variante, Konnektivität, Audio-Stack) sind aufgeführt
- [ ] Mechanische, elektrische und thermische Limitationen sind benannt; Notstopp-Pfad ist definiert
- [ ] Mindestens 10 Patterns für natürliche, flüssige Bewegung sind dokumentiert
- [ ] Mindestens 9 Anti-Patterns sind dokumentiert
- [ ] Beobachtbarkeit (Position-Read, Latenz-Trace, Telemetrie) ist als Anforderung geführt
- [ ] Hardware-Zahlen, die nicht aus offizieller Doku oder SDK-Source verifiziert sind, tragen einen `⚠ TBD`-Hinweis
- [ ] `reachy-mini-sdk`-Skill verweist auf diese Spec als kanonische Wissensbasis
- [ ] `app-scaffold`-Skill verweist auf diese Spec für Move-Primitives und Default-Easings

## Quellen
- Upstream-SDK-Repo (kanonische Quelle für alle in den Tabellen referenzierten Konstanten und Klassen): <https://github.com/pollen-robotics/reachy_mini>
- SDK-Source-Tree (`ReachyMini`, IO, Media, Motion, Daemon, Apps, Tools): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Motion-Modul (`Move`-ABC, Easing-Modi `MIN_JERK` / `CARTOON`, `goto`, `recorded_move`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- IO-Protokoll (sämtliche `*Cmd`/`*Msg`-Typen, die in den Anforderungen verwendet werden): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- Media-Stack (Kamera, Audio, GStreamer-Pipelines, Mic-DOA): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- Daemon (Status, App-Lock, REST-API): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- API-Doku (MDX-Quellen für `reachymini`, `media`, `motion`, `daemon`, `apps`, `tools`, `utils`, REST-API, OpenAPI-Schema): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/API>
- SDK-Konzept-Doku (Quickstart, Core-Concept, Apps, Python-/JavaScript-SDK, Media-Architektur): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Plattform-Profile-Doku (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- ~~Hat der Reachy-Mini-Kopf 3 DoF oder 6 DoF?~~ **Beantwortet**: 6 DoF (Stewart-Plattform), Pose ist 4×4-Transform-Matrix; Builder `create_head_pose(x, y, z, roll, pitch, yaw, …)` aus `reachy_mini.utils`.
- ~~Welche Audio-Codecs?~~ **Beantwortet**: Push-API erwartet `F32LE` bei 48 kHz, 2 Channels. `play_sound(file=...)` decodiert beliebige Formate über GStreamer `playbin`.
- ~~Update-Frequenz?~~ **Beantwortet**: Daemon publiziert mit 50 Hz; höhere Tick-Frequenzen sind nicht sinnvoll.
- ~~Hat Reachy Mini eine IMU?~~ **Beantwortet**: ja, **nur auf Wireless**; Felder: accelerometer, gyroscope, quaternion, temperature.
- Bietet das SDK ein Capability-Discovery-API, um zur Laufzeit zwischen Reachy Mini, Reachy Mini Lite und Simulation zu unterscheiden? Konstruktor-Argument `use_sim` ist verifiziert; eine explizite Variant-Property auf der Instanz `> ⚠ TBD: read DaemonStatus.no_media / camera_specs_name as proxy`.
- Bietet das SDK eine Brown-out- bzw. Strom-Spitzen-Telemetrie, oder muss das Behavior das selbst messen?
- ~~Was ist die kanonische Ruhepose ("rest pose") für den Notstopp?~~ **Beantwortet**: `INIT_HEAD_POSE = np.eye(4)` plus `INIT_ANTENNAS_JOINT_POSITIONS` (~10° Offset). `SLEEP_HEAD_POSE` ist eine Alternative, die `goto_sleep()` ansteuert.
- ~~Welche Update-Frequenzen sind realistisch in einem Behavior-Tick — 50 / 100 / 200 Hz?~~ **Beantwortet**: Der Daemon publiziert mit 50 Hz; höhere Tick-Frequenzen sind verschwendet.
- Gibt es ein offizielles Animation-Authoring-Tool von Pollen Robotics (Timeline-Editor) und ist dessen Output-Format eine empfohlene Komposition für unsere Move-Primitive-Schicht?
- Wie verhält sich der Stewart-Plattform-Kopf mathematisch nahe der Singularität? Müssen wir Singularitätsbereiche im Code blockieren?
- Welche Latenz-Verteilung (P50, P95, P99) ist für die einzelnen Schichten typisch? Ohne diese Verteilung ist ein realistisches Tanz-Timing nicht planbar.
- Sollen Animations-Prinzipien (Anticipation, Follow-Through usw.) in eine eigene `motion-design`-Spec ausgelagert werden, sobald die Patterns wachsen?
- Sind die offiziellen Reachy-Mini-Pollen-Behaviors (z. B. „Hello", „Curious") als Referenz-Implementierungen freigegeben, gegen die wir unsere Pattern-Umsetzung kalibrieren können?
- Wie weit lassen sich Patterns aus klassischer Animation auf einen 6-DoF-Roboter übertragen, ohne in Uncanny-Valley-Territorium zu rutschen? Empirische Evaluation an realer Hardware nötig.
