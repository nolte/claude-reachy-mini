# Motor-Positionen, Limits und kanonische Posen des Reachy Mini

Status: draft

## Kontext

Wer einen Reachy Mini bewegt, braucht zwei Arten von Antworten, die in der bestehenden Doku verstreut oder gar nicht beantwortet sind: *Welche Stellung darf jeder einzelne Motor erreichen?* und *Welche Kombinationen dieser Stellungen sind tatsächlich valide — physikalisch erreichbar, mechanisch unbedenklich, im Sinne des SDK-IK lösbar?* Die Spec [`reachy-mini/control-surface`](../control-surface/de.md) beantwortet die erste Frage auf hoher Abstraktionsebene (Inventar, Nominal-Operations-Range) und nennt die Stewart-Joint-Limits nur pauschal als „asymmetrisch je Joint". Konkrete Werte pro Motor, kanonische Ruhestellungen mit ihren exakten Joint-Vektoren und die kombinatorischen Konflikt-Konstellationen sind dort bewusst ausgespart. Diese Spec liefert genau diese fehlende Tiefe.

Sie wird vom [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md)-Skill konsumiert, wenn Snippets gegen Limits validiert werden, vom [`dance-choreography`](../../claude/dance-choreography/de.md)-Skill bei der Komposition extremer Posen, vom [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/de.md)-Skill bei der Plausibilität von State-Reads, und von jeder zukünftigen `app-scaffold`-Vorlage als Quelle der Wahrheit für Pose-Defaults. Die Werte sind zum Stand 2026-05-12 verifiziert gegen einen laufenden Wireless-Daemon und den Pollen-SDK-Source auf [`main`](https://github.com/pollen-robotics/reachy_mini).

Wichtiger Befund vorweg: der Daemon meldet `engine: AnalyticalKinematics, collision check: false`. Das SDK validiert eine Pose **nur** gegen das Inverse-Kinematik-Polytop, nicht gegen Selbstkollision, mechanische Anschläge der Antennen oder Kabelbäume. Eine Pose, die das IK akzeptiert, ist *kinematisch erreichbar* — nicht automatisch *mechanisch unbedenklich*. Die Spec hält das durchgehend explizit.

## Ziele

- Pro Motor / Joint die exakten Werte aus dem live URDF dokumentieren — Lower-Limit, Upper-Limit, Velocity, Effort — mit klarer Markierung, dass URDF (mechanische Grenze) gegenüber `kinematics_data.json` (Software-Grenze ±π) Vorrang hat
- Die kanonischen Posen aus dem SDK-Source (`INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS`, `SLEEP_HEAD_POSE`, `SLEEP_ANTENNAS_JOINT_POSITIONS` plus die hartcodierten Joint-Vektoren für jeden) verbatim spiegeln, inklusive der `wake_up`- und `goto_sleep`-Trajektorien
- Die drei Schichten der Validität klar trennen: Joint-Limit (URDF), kinematische Erreichbarkeit (IK-Polytop), mechanische Unbedenklichkeit (empirisch)
- Bekannte Konflikt-Konstellationen aus der Stewart-Plattform-Geometrie nennen, sodass eine Komposition sie früh als „IK-unlösbar" einschätzen kann, ohne das Gerät zu fragen
- Alle Werte mit Datums- und Quellen-Marker versehen, damit eine spätere URDF- oder SDK-Änderung in einem Folge-Audit sofort sichtbar wird

## Nicht-Ziele

- Bewegungs-Komposition, Easing-Profile, Anticipation/Follow-Through — gehört zu [`reachy-mini/control-surface`](../control-surface/de.md) §"Bewegungs-Design"
- Konkrete Choreographien — gehört zu [`reachy-mini/motions/`](../motions/) (Einzeldateien) und zum [`dance-choreography`](../../claude/dance-choreography/de.md)-Skill
- Implementierungsanleitung für ein Skill / einen Agent — diese Spec ist normative Wissensbasis, keine Operations-Vorschrift
- Audio-, Vision- oder LED-Steuerung — andere Subsysteme
- Selbstkollisions-Algorithmus oder URDF-basierter Mesh-Check — explizit nicht im SDK; bis das nachgerüstet ist, bleibt die Spec auf einer „bekannte Risiken"-Aufzählung
- Hardware-Bringup, Kalibrierung, Firmware-Flash — eigene Skills (geplant)
- Werte für die Simulations-Variante, wenn sie von der Hardware abweichen — die Sim teilt die URDF-Limits, aber elektromechanisches Verhalten (Effort, Velocity unter Last) ist sim-irrelevant

## Anforderungen

### Schicht 1 — Joint-Limits pro Motor (aus dem live URDF)

Werte abgerufen am 2026-05-12 von `http://reachy-mini.local:8000/api/kinematics/urdf`, generiert aus [`src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf) (onshape-to-robot-Generierung).

#### Aktive Joints (steuerbar, mechanische Grenzen)

| Joint | Typ | Min (rad) | Max (rad) | Min (deg) | Max (deg) | Velocity (rad/s) | Effort (N·m) | Motor |
|---|---|---|---|---|---|---|---|---|
| `stewart_1` | revolute | −0.8378 | +1.3963 | **−48°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_2` | revolute | −1.3963 | +1.2217 | **−80°** | **+70°** | 8 | 10 | XL330-M288-T |
| `stewart_3` | revolute | −0.8378 | +1.3963 | **−48°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_4` | revolute | −1.3963 | +0.8378 | **−80°** | **+48°** | 8 | 10 | XL330-M288-T |
| `stewart_5` | revolute | −1.2217 | +1.3963 | **−70°** | **+80°** | 8 | 10 | XL330-M288-T |
| `stewart_6` | revolute | −1.3963 | +0.8378 | **−80°** | **+48°** | 8 | 10 | XL330-M288-T |
| `right_antenna` | revolute | −π | +π | −180° | +180° | 8 | 10 | XL330-M077-T |
| `left_antenna` | revolute | −π | +π | −180° | +180° | 8 | 10 | XL330-M077-T |
| `yaw_body` | revolute | −2.7925 | +2.7925 | **−160°** | **+160°** | 8 | 10 | XC330-M288-PG (custom) |

#### Stewart-Asymmetrie-Muster

Die sechs Stewart-Aktuatoren sind paarweise gespiegelt arrangiert. Daraus ergibt sich ein striktes Asymmetrie-Muster, das in der `control-surface`-Spec nur pauschal angedeutet war:

| Paar | Joints | Lower–Upper (deg) | Interpretation |
|---|---|---|---|
| **A** | `stewart_1`, `stewart_3` | **−48° / +80°** | Mehr Hub „nach oben/innen", weniger „nach unten/außen" |
| **B** | `stewart_4`, `stewart_6` | **−80° / +48°** | Spiegelbild von Paar A |
| **C-1** | `stewart_2` | **−80° / +70°** | Praktisch symmetrisch, leicht nach unten verschoben |
| **C-2** | `stewart_5` | **−70° / +80°** | Spiegelbild von `stewart_2` |

Konsequenz: eine Head-Pose, die Stewart-1 nach +80° fährt, beansprucht Stewart-4 in die gleiche Richtung; weil Stewart-4 dort nur +48° kann, ist die symmetrische Maximalauslenkung des Kopfes nach einer Seite **enger** als nach der anderen. Die effektive Pitch/Roll-Range hängt damit auch von der Yaw-Richtung ab.

#### Software-Grenze (`kinematics_data.json`) vs. mechanische Grenze (URDF)

Die [`assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json) listet für jeden Stewart-Motor `limits: [-π, +π]` (also ±180°). Das ist **nicht** die mechanische Grenze, sondern eine Software-Default-Schranke des IK-Solvers. **Autoritativ ist das URDF.** Wer einen Wert gegen die Limits validiert, nimmt die URDF-Tabelle oben — nicht das JSON.

#### Passive Joints (kinematisch nötig, nicht steuerbar)

Das URDF deklariert zusätzlich 21 passive Revolute-Joints (`passive_1_x/y/z` … `passive_7_x/y/z`) mit Limits `±π`, Velocity `1e+08` (effektiv unbegrenzt) und Effort `10` N·m. Das sind die Kugelgelenke der Stewart-Anbindung, die zur Schließung der kinematischen Schleifen nötig sind. Sie tauchen weder in der REST-API noch im Python-SDK als steuerbare Größe auf — sie werden vom IK implizit gesetzt und sind hier nur der Vollständigkeit halber erwähnt.

### Schicht 2 — Inverse Kinematik und Workspace

Engine: `AnalyticalKinematics` (Rust-Kern mit Python-Bindings, Source: [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)). Live-Identifikation: `GET /api/kinematics/info` meldet `{"engine":"AnalyticalKinematics","collision check":false}`.

#### Kinematik-Parameter

Aus [`assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json):

| Parameter | Wert | Bedeutung |
|---|---|---|
| `motor_arm_length` | 40 mm | Hebellänge des Motor-Horns |
| `rod_length` | 85 mm | Länge der starren Stange Motor-Horn → Plattform |
| `head_z_offset` | 177 mm | Vertikaler Offset vom Stewart-Basis-Frame zum Head-Frame; das IK addiert ihn intern auf jede Z-Komponente einer Pose |

#### IK-eingebaute Safety-Schwellen

Aus `analytical_kinematics.py`, im `inverse_kinematics_safe`-Pfad (aktiv wenn `automatic_body_yaw=True`):

| Konstante | Wert | Bedeutung |
|---|---|---|
| `max_relative_yaw` | `np.deg2rad(65)` = **±65°** | Maximaler Yaw-Winkel des Head-Frames *relativ* zum Body-Yaw; übersteigt eine Ziel-Pose das, wird der Body-Yaw mitgedreht |
| `max_body_yaw` | `np.deg2rad(160)` = **±160°** | Maximaler Body-Yaw; identisch zur URDF-Grenze von `yaw_body` |

Im `automatic_body_yaw=False`-Pfad wird der Body-Yaw vom Aufrufer gesetzt und der IK schlägt fehl, wenn die Stewart-Lösung nicht in das URDF-Polytop fällt — kein automatisches Anpassen.

#### Workspace-Polytop

Der erreichbare Head-Pose-Raum ist die Menge aller 4×4-Transforms, deren inverse-kinematische Stewart-Lösung in den URDF-Limits liegt. Eine geschlossene Form gibt es nicht, weil die sechs Stewart-Limits asymmetrisch sind und die Solverfunktion eine analytische Stewart-Inversion ist. Empirisch — gestützt auf die `control-surface`-Doku und die Pollen-Datasheet-Tabelle (`platforms/reachy_mini/hardware`) — gelten als **nominale Operations-Range**:

| Achse | Min | Max | Quelle |
|---|---|---|---|
| Head-Roll (Rx) | −40° | +40° | Pollen `dof_table.png` |
| Head-Pitch (Ry) | −40° | +40° | Pollen `dof_table.png` |
| Head-Yaw (Rz, relativ zum Body) | −60° | +60° | Pollen `dof_table.png` (eng), `max_relative_yaw` IK-intern +65° |
| Body-Yaw (Rz) | −155° | +155° | Pollen `dof_table.png` (eng), URDF +160° |
| Antenne rechts (R) | −180° | +180° | URDF |
| Antenne links (R) | −180° | +180° | URDF |
| Head-Translation x | ≈ −20 … +20 mm | ⚠ TBD: am realen Gerät messen | IK-Polytop |
| Head-Translation y | ≈ −20 … +20 mm | ⚠ TBD: am realen Gerät messen | IK-Polytop |
| Head-Translation z | ≈ −45 … +45 mm relativ zum head_z_offset | ⚠ TBD | IK-Polytop |

Die Translation-Bereiche sind aus den SLEEP-Pose-Werten (siehe Schicht 3) extrapoliert und **nicht** verifiziert; ein Folge-Audit am Gerät kann sie pinnen.

#### IK-Bisection — gemessene Polytop-Grenzen (Phase A)

Per `reachy_mini==1.7.2` lokal ausgeführter Bisection-Sweep gegen `AnalyticalKinematics.ik(...)` (2026-05-12). Bisection mit ε = 1e-4, Stopp-Kriterium: entweder IK lehnt ab **oder** mindestens ein Stewart-Joint überschreitet die URDF-Grenze.

| Achse | Δ_max (IK + URDF-konform) | limit-triggernder Joint | s1 (°) | s2 (°) | s3 (°) | s4 (°) | s5 (°) | s6 (°) | body (°) |
|---|---|---|---|---|---|---|---|---|---|
| `tx_pos` | +50.88 mm | s3, s4 (±80°) | +25.85 | −77.09 | +79.95 | −79.95 | +77.09 | −25.85 | 0 |
| `tx_neg` | −46.78 mm | s1, s6 (±80°) | +79.91 | −32.10 | +50.05 | −50.05 | +32.10 | −79.91 | 0 |
| `ty_pos` | +47.56 mm | s5 (+80°) | +64.25 | −35.50 | +23.69 | −78.49 | +79.99 | −50.45 | 0 |
| `ty_neg` | −47.56 mm | s2 (−80°) | +50.45 | −79.99 | +78.49 | −23.69 | +35.50 | −64.25 | 0 |
| **`tz_pos`** (Kopf voll oben) | **+23.05 mm** | alle 6 gleichzeitig | +79.70 | −79.70 | +79.70 | −79.70 | +79.70 | −79.70 | 0 |
| `tz_neg` (Kopf voll unten) | −50.78 mm | s1, s3 (−48°) + s4, s6 (+48°) | −47.83 | +47.83 | −47.83 | +47.83 | −47.83 | +47.83 | 0 |
| `roll_pos` | +47.71° | s2 (−80°) | +60.44 | −80.00 | +44.34 | −30.67 | +14.01 | −13.90 | 0 |
| `roll_neg` | −47.71° | s5 (+80°) | +13.90 | −14.01 | +30.67 | −44.34 | +80.00 | −60.44 | 0 |
| `pitch_pos` (Kopf nach oben kippen) | **+48.01°** | s3, s4 (±80°) | +21.26 | −26.33 | +80.00 | −80.00 | +26.33 | −21.26 | 0 |
| `pitch_neg` (Kopf nach unten kippen) | **−72.43°** | s1 (+80°), s6 (−80°) | +80.00 | −45.01 | +6.55 | −6.55 | +45.01 | −80.00 | 0 |
| `yaw_pos` | +90° (probe-cap) | keine — IK akzeptiert mehr | +62.57 | −33.89 | +62.57 | −33.89 | +62.57 | −33.89 | +25.00 |
| `yaw_neg` | −90° (probe-cap) | keine | +33.89 | −62.57 | +33.89 | −62.57 | +33.89 | −62.57 | −25.00 |

Beobachtungen:

- **Pitch ist deutlich asymmetrisch**: +48° („Kopf hoch") gegen −72° („Kopf vor"). Das spiegelt die paarweise Spiegelung der Stewart-Aktuatoren.
- **Max-Heave („Kopf voll oben") = +23.05 mm** mit allen 6 Stewart-Joints am eigenen ±80°-Limit, paarweise antisymmetrisch. Heave nach unten ist deutlich tiefer möglich (−50.78 mm), durch die andere Limit-Asymmetrie.
- **Yaw bleibt von URDF unbeschränkt**: die IK akzeptiert Head-Yaw über die Probe-Grenze 90° hinaus; theoretisches Maximum = `max_relative_yaw + max_body_yaw` = 65° + 160° = **225°**.

> **⚠ WARNUNG — Diese Werte sind NICHT mechanisch sicher.** Sie sind die mathematische Polytop-Grenze des analytischen IK *unter Respektierung der URDF-Grenzen*. Eine reale Bewegung an die IK-Grenze kann eine Selbstkollision auslösen — siehe Schicht 4 §"Phase-B-Live-Vorfall 2026-05-12". Verbindlich für Komposition ist die **Pollen-Nominal-Operations-Range** (±40° Pitch/Roll), nicht diese Tabelle. Diese Tabelle dokumentiert *was die IK akzeptieren würde*, nicht *was die Hardware aushält*.

#### Drei Schichten der Validität — Reihenfolge der Prüfung

Eine Behavior-Komposition **MUSS** in dieser Reihenfolge prüfen, von außen nach innen:

1. **IK-Polytop** (Software-Default, mathematisch). Akzeptiert auch Posen außerhalb der URDF-Grenzen, weil der Solver nur die `kinematics_data.json`-Limits (±π) kennt. **Nicht** als Sicherheits-Gate ausreichend.
2. **URDF-mechanische Grenzen** (Schicht 1). Bilden die echte Motor-Bewegungsfreiheit. Eine Pose, die einen Stewart-Joint über diese Grenze treibt, wird vom Motor abgeklippt — oder, wie der `set_mode/enabled`-Pfad zeigt, von der `goto`-API stillschweigend ignoriert.
3. **Pollen-Nominal-Operations-Range** (Pollen-Datasheet, ±40° Pitch/Roll, ±60° Head-Yaw, ±155° Body-Yaw). Die *empfohlene* Range, in der die Hardware verlässlich, ohne Selbstkollision und ohne mechanischen Stress operiert. **Diese ist verbindlich für jede Bewegungs-Komposition.**

Schicht 3 ⊊ Schicht 2 ⊊ Schicht 1. Wer auf Schicht 1 stoppt, hat keine Sicherheits-Aussage. Wer auf Schicht 2 stoppt, hat keine Selbstkollisions-Garantie. **Nur Schicht 3 ist verbindlich.**

#### Yaw-Aufteilung Body / Head

Eine Drehung „Reachy schaut 70° nach links" wird vom `inverse_kinematics_safe` automatisch in `head_yaw=65°` + `body_yaw=5°` zerlegt, weil `max_relative_yaw=65°` überschritten würde. Im `automatic_body_yaw=False`-Modus muss der Aufrufer die Aufteilung selbst vorgeben — eine reine Head-Yaw-Anforderung von 70° schlägt fehl, statt vom Body mitkompensiert zu werden.

### Schicht 3 — Kanonische Posen aus dem SDK-Source

Werte verbatim aus [`src/reachy_mini/reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py) und [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py), Stand `main` 2026-05-12.

#### Init-Pose (`wake_up`-Endzustand, neutrale Referenz)

`INIT_HEAD_POSE`:

```
np.eye(4)          # 4×4-Identitätsmatrix
# Rotation: keine (Roll = Pitch = Yaw = 0)
# Translation (Head-Frame): (0, 0, 0) m
# Effektive Welt-Translation: (0, 0, head_z_offset) = (0, 0, 0.177) m
```

`INIT_ANTENNAS_JOINT_POSITIONS`:

| Antenne | rad | deg |
|---|---|---|
| `right_antenna` | −0.1745 | **−10°** |
| `left_antenna` | +0.1745 | **+10°** |

Konstanten-Begründung im SDK-Source: *„~10° offset to reduce shaking at vertical"* — beide Antennen sind bewusst leicht nach außen geneigt, um vertikalen Mikro-Tremor durch Motor-Backlash zu vermeiden. **Eine echte 0/0-Antennenstellung ist explizit nicht die Default-Ruhestellung.**

Hartkodierte IK-Lösung für `INIT_HEAD_POSE` (im SDK-Source als Fallback verwendet, wenn der Daemon noch keine eigene Pose liefern kann):

```
[body_yaw, stewart_1, stewart_2, stewart_3, stewart_4, stewart_5, stewart_6] =
[6.96e-07, +0.5252, −0.6687, +0.6067, −0.6067, +0.6687, −0.5252]    # rad
[~0°,      +30.09°, −38.31°, +34.76°, −34.76°, +38.31°, −30.09°]    # deg
```

Beobachtung: die Stewart-Vektoren sind **paarweise antisymmetrisch** (s1 vs. s6, s2 vs. s5, s3 vs. s4) — was die mechanische Spiegelung der Aktuatoren bestätigt.

#### Sleep-Pose (`goto_sleep`-Endzustand)

`SLEEP_HEAD_POSE`:

```
[[ 0.911,  0.004,  0.413, -0.021],
 [-0.004,  1.0,   -0.001,  0.001],
 [-0.413, -0.001,  0.911, -0.044],
 [ 0.0,    0.0,    0.0,    1.0  ]]
```

| Komponente | Wert | Bedeutung |
|---|---|---|
| Rotation R (extrahiert) | Pitch ≈ **−24.4°** (arctan(0.413 / 0.911)) | Kopf nach vorne-unten geneigt |
| Translation (Head-Frame) | x = **−21 mm**, y = **+1 mm**, z = **−44 mm** | Kopf zurückversetzt und abgesenkt |
| Effektive Welt-Z | head_z_offset + z = 0.177 − 0.044 = **0.133 m** | Endgültige Sleep-Höhe der Plattform |

`SLEEP_ANTENNAS_JOINT_POSITIONS`:

| Antenne | rad | deg | Distanz zum Limit |
|---|---|---|---|
| `right_antenna` | **−3.05** | **−174.7°** | 5.3° vor dem ±180°-Anschlag |
| `left_antenna` | **+3.05** | **+174.7°** | 5.3° vor dem ±180°-Anschlag |

Die Antennen klappen also fast vollständig nach hinten — sie lehnen praktisch an ihren mechanischen Stops an. Diese 5.3°-Marge ist bewusst: bei einem geringfügigen Encoder-Drift während des Schlafs kommen die Antennen nicht direkt am Stop zu liegen.

Hartkodierte IK-Lösung für `SLEEP_HEAD_POSE`:

```
[body_yaw, stewart_1, stewart_2, stewart_3, stewart_4, stewart_5, stewart_6] =
[0.0, −0.9848, +1.2625, −0.2439, +0.2056, −1.2364, +1.0032]    # rad
[0°,  −56.43°, +72.32°, −13.97°, +11.78°, −70.84°, +57.48°]    # deg
```

Beobachtung: `stewart_2` erreicht **+72.32°** und liegt damit nur 2.32° unter seinem URDF-Maximum von +70° — **das ist eine Diskrepanz**, die Aufmerksamkeit verdient. ⚠ TBD: prüfen, ob die SDK-Konstante oder das URDF-Limit korrigiert werden muss; ggf. ist die Sleep-Pose nahe einer kinematischen Grenze und schlägt auf einigen Geräten mit etwas anderen Kalibrierungen fehl. `stewart_5` zeigt mit −70.84° das gleiche Muster knapp jenseits seines −70°-URDF-Limits — ebenfalls TBD.

#### `wake_up`-Trajektorie

Aus `ReachyMini.wake_up()` (`reachy_mini.py:575`):

1. `goto_target(INIT_HEAD_POSE, antennas=INIT_ANTENNAS_JOINT_POSITIONS, duration=2.0 s)` — von beliebiger Pose zu Neutral
2. `time.sleep(0.1)`
3. Sound `wake_up.wav` (Toudoum-Glocke)
4. `goto_target(pose_roll20, duration=0.2 s)` — wobei `pose_roll20` = `INIT_HEAD_POSE` mit Roll +20° (xyz-Euler) — Kopf lehnt kurz nach links
5. `goto_target(INIT_HEAD_POSE, duration=0.2 s)` — zurück zur Neutral-Pose

Endzustand der `wake_up`-Sequenz ist `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS`.

#### `goto_sleep`-Trajektorie

Aus `ReachyMini.goto_sleep()` (`reachy_mini.py:591`):

1. `get_current_joint_positions()` → Distanz-Check zur hartkodierten `init_positions` (siehe oben); wenn `np.linalg.norm > 0.2 rad`, dann
   `goto_target(INIT_HEAD_POSE, antennas=INIT_ANTENNAS_JOINT_POSITIONS, duration=1.0 s)` + `time.sleep(0.2)`
2. Sound `go_sleep.wav` (Pfiou-Seufzer)
3. `goto_target(SLEEP_HEAD_POSE, antennas=SLEEP_ANTENNAS_JOINT_POSITIONS, duration=2.0 s)`
4. `time.sleep(2)`

Beobachtung: Schritt 1 macht den Schlaf-Pfad **zustandsabhängig**. Eine App kann nicht einfach `goto_sleep` aufrufen und sich auf eine vorhersagbare 2-Sekunden-Bewegung verlassen — bei großer Distanz dauert es 3.2 s plus.

### Schicht 4 — Bekannte Konflikt-Konstellationen

Das SDK macht **keinen** Collision Check (`collision check: false`). Die folgenden Konstellationen sind nach aktuellem Wissensstand riskant; sie werden vom IK akzeptiert oder mit einem schwer interpretierbaren Fehler abgelehnt, sind aber für eine Behavior-Komposition besser von vornherein zu vermeiden.

#### IK-Fehler-Klassen

- **Pose außerhalb des Stewart-Polytops**: extreme Pitch + extreme Roll gleichzeitig — z. B. Roll +35° kombiniert mit Pitch +35°. Einzeln innerhalb der nominalen ±40°, kombiniert oft nicht IK-lösbar. **Konsequenz**: `goto_target` wirft eine kinematics-Exception; im REST-Pfad meldet `POST /api/move/goto` einen 4xx-Status.
- **Translation außerhalb des erreichbaren Volumens**: Head-Frame-Translation z = +50 mm bei gleichzeitiger Pitch-Auslenkung — die effektive Stewart-Beinlänge übersteigt `motor_arm_length + rod_length`. Selbe Fehler-Klasse.
- **Head-Yaw ohne Body-Kompensation jenseits ±65°**: nur im `automatic_body_yaw=False`-Pfad. Im Default-Pfad still vom IK kompensiert.

#### Mechanische Konflikte (vom IK *nicht* gefangen)

- **Antennen-Crossing**: beide Antennen-Joints in Richtungen, die ihre Spitzen sich kreuzen lassen — z. B. `right_antenna = +90°` und `left_antenna = −90°`. Die Antennen können sich physisch berühren oder das Kopfgehäuse streifen. ⚠ TBD: am Gerät validieren, ob es einen sicheren „Crossing-Forbidden-Zone" gibt.
- **Antennen-Anschlag-Schaden**: Antenne längere Zeit an ±180° gegen Stop fahren, mit weiterem Drehmoment beaufschlagt. Das URDF-Effort-Limit (10 N·m) schützt nicht vollständig — der Motor versucht weiter zu drehen. **Empfehlung**: nach Erreichen einer Sleep-nahen Pose den Antennen-Motor in `disabled` oder `gravity_compensation` schalten (Mode-Namen siehe Schicht 5); das passiert in `goto_sleep` derzeit nicht.
- **Sleep-Pose nahe URDF-Limit**: `SLEEP_HEAD_JOINT_POSITIONS` enthält `stewart_2 = +72.32°` und `stewart_5 = −70.84°` — beide jenseits ihrer URDF-Limits (`+70°` bzw. `−70°`). Auf einem nominal kalibrierten Gerät wird das IK das vermutlich klippen oder ablehnen. ⚠ TBD: ob die SDK-Konstante stale ist (URDF wurde nach `SLEEP_HEAD_JOINT_POSITIONS` enger gesetzt) oder ob die Konstante das Soll ist und das URDF zu konservativ.
- **Camera-Kabel im Pitch-Extrem**: ⚠ TBD — die Pollen-Doku nennt keinen Kabelweg explizit, aber Kameramodelle mit USB-Kabel können bei aggressivem Pitch ihren Bewegungsspielraum überschreiten.

#### Zustands-Konflikte

- **Motoren-Mode-Wechsel während laufender App**: ein `POST /api/motors/set_mode/{mode}` von `enabled` auf `disabled` oder `gravity_compensation` während eines `set_target`-Streams kann die laufende Pose nicht-deterministisch unterbrechen. Konvention: Mode-Wechsel sind selten, bewusst gesetzte Operationen — siehe `reachy-mini/daemon-rest-api` §"Motors". Die korrekten Mode-Namen sind in Schicht 5 dokumentiert.

#### Phase-B-Live-Vorfall 2026-05-12 (Selbstkollision dokumentiert)

Während eines Versuchs, die in Schicht 2 §"IK-Bisection" gefundenen Polytop-Grenzen am echten Wireless live zu verifizieren, hat der Reachy bei den IK-validen Pitch-Posen Selbstkollisionen ausgeführt: der Kopf hat „sehr stark gegen seinen Körper geknallt" (Operator-Beobachtung). Sequenz, in der Reihenfolge der Anfahrt:

1. `tz_pos` (Ziel z = +23.05 mm) — Reachy schaffte effektiv nur ≈ +10 mm; head_pose blieb bei pitch +0.005 rad, z ≈ −145 mm (Baseline −155 mm)
2. `tz_neg` (Ziel z = −50.78 mm) — Reachy schaffte effektiv ≈ −53 mm; head_pose pitch +0.093 rad, z = −208 mm
3. `pitch_pos` (Ziel pitch = +48°) — gelesen pitch = +17.8°, **30° Diskrepanz**; mechanische Kollision hier vermutlich initial
4. `pitch_neg` (Ziel pitch = −72°) — gelesen pitch = −15.6°, **56° Diskrepanz**; weitere Kollision

Folgezustand nach Sweep:
- `head_pose` bleibt byte-identisch bei pitch ≈ +0.75 rad, z ≈ −187 mm trotz `POST /api/move/goto INIT` mit `duration=6.0` und `motors/set_mode/enabled`
- `head_joints: null` in `/api/state/full` — Daemon kann Stewart-Positionen nicht mehr auslesen
- `backend_status.ready: false`, `backend_status.last_alive: null` — Motor-Backend (USB-Verbindung zu den Dynamixel-Motoren) offline
- `POST /api/motors/set_mode/gravity_compensation` → **500 Internal Server Error**
- `POST /api/move/goto` → 200 OK mit UUID, aber **keine** Bewegung passiert

Interpretation: die Dynamixel-Motoren haben sich vermutlich durch Overload-Schutz oder Position-Error selbst offline gesetzt. Recovery erfordert vermutlich einen Daemon-Restart (`POST /api/daemon/restart`) oder einen Power-Cycle des Reachy. **Eine reine REST-Recovery aus diesem Zustand war in 2026-05-12 nicht möglich.**

Lehren (verbindlich):

1. **Niemals** IK-Polytop-Grenzwerte aus Schicht 2 §"IK-Bisection" als Live-Targets verwenden. Live-Bewegungen halten sich an die Pollen-Nominal-Operations-Range (±40° Pitch/Roll, ±60° Head-Yaw).
2. Bei jedem Live-Sweep `duration` ≥ 5.0 s wählen, damit der Daemon Zeit hat zu reagieren und ein Abbruch durch den Operator möglich ist.
3. Pre-Flight: vor jeder Bewegungs-Sequenz die App-Lock-Lage und den `backend_status.ready` prüfen — wenn `ready != true`, **nicht** mit `goto` starten.
4. Bei `backend_status.ready: false` und `head_joints: null` ist der Reachy **nicht** softwareseitig recoverbar; physische Inspektion und ggf. Power-Cycle.

### Schicht 5 — Motor-Modi und Backend-Health

Korrektur zur `reachy-mini/control-surface`-Spec, die Modi `stiff` / `compliant` benannte: die tatsächlichen Mode-Namen des Daemons (Stand `reachy_mini==1.7.1`, geprüft live 2026-05-12) sind:

| Mode-Name | API-Verhalten | Wirkung |
|---|---|---|
| `enabled` | Default; `set_target` und `goto` werden gefahren | Motoren halten aktiv die Sollwert-Pose („stiff") |
| `disabled` | Motoren entlastet; `goto` wird angenommen, fährt aber nicht | Roboter ist frei beweglich von Hand; Schwerkraft drückt den Kopf nach unten |
| `gravity_compensation` | Theoretisch: hält aktuelle Pose gegen Schwerkraft ohne aktiven Sollwert | ⚠ TBD: hat in 2026-05-12 mit `500 Internal Server Error` geantwortet, wenn aus `disabled` umgeschaltet; möglicherweise nur aus `enabled` heraus zulässig |

API: `POST /api/motors/set_mode/{mode}` — der Pfad-Parameter ist eine Enum, die nur diese drei Werte akzeptiert; falsche Namen geben **422 Unprocessable Entity**. `GET /api/motors/status` liefert `{"mode": "<aktueller_mode>"}`.

#### Backend-Status — wann der Reachy nicht reagiert

`GET /api/daemon/status` liefert ein `backend_status`-Objekt, das den Motor-Controller beschreibt. Die wichtigsten Felder:

| Feld | Bedeutung | Sicherer Wert |
|---|---|---|
| `backend_status.ready` | `true` wenn der Motor-Controller die Dynamixel-Bus-Verbindung aktiv hält | `true` |
| `backend_status.last_alive` | Letzter Heartbeat-Zeitstempel | nicht-null |
| `backend_status.motor_control_mode` | Spiegelt `GET /api/motors/status`-Mode | `enabled` / `disabled` / `gravity_compensation` |
| `backend_status.control_loop_stats.mean_control_loop_frequency` | Nominal ≈ 50 Hz (gemessen 49.7 Hz) | > 40 Hz |
| `backend_status.control_loop_stats.nb_error` | Zähler der Motor-Controller-Fehler | 0 |
| `backend_status.error` | Letzter Daemon-seitiger Fehler-String | `null` |

Eine Live-`goto`-Bewegung **MUSS** vorab `backend_status.ready == true` prüfen. Wenn `false`, ist `head_joints` in `/api/state/full` typischerweise `null` und die `goto`-API akzeptiert Bewegungen still (200 mit UUID), ohne sie auszuführen. In diesem Zustand stoppt man Live-Tests sofort und triagiert per Daemon-Restart oder Power-Cycle.

## Akzeptanzkriterien

- [ ] Spec existiert unter `spec/reachy-mini/motor-positions/de.md` (kanonisch) und `spec/reachy-mini/motor-positions/en.md` (Übersetzung)
- [ ] Jeder aktive Joint aus dem live URDF ist in Schicht 1 mit exakten Werten (rad + deg) gelistet
- [ ] Stewart-Asymmetrie ist als Paar-Muster (A, B, C-1, C-2) ausgewiesen
- [ ] Diskrepanz URDF-Grenze vs. `kinematics_data.json`-Grenze ist explizit benannt; URDF gilt als autoritativ
- [ ] IK-Parameter (motor_arm_length, rod_length, head_z_offset) und Safety-Schwellen (`max_relative_yaw`, `max_body_yaw`) sind genannt
- [ ] Kanonische Posen `INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS`, `SLEEP_HEAD_POSE`, `SLEEP_ANTENNAS_JOINT_POSITIONS` sind als Matrix bzw. Vektor verbatim aus dem SDK-Source übernommen
- [ ] Hartkodierte Stewart-Joint-Vektoren für init- und sleep-Pose sind als rad und deg gelistet, Antisymmetrie ist benannt
- [ ] `wake_up`- und `goto_sleep`-Trajektorien sind schrittweise mit Dauer und Sound-Asset dokumentiert
- [ ] `collision check: false` ist als zentrale Aussage des SDK an mehreren Stellen markiert
- [ ] Diskrepanz zwischen `SLEEP_HEAD_JOINT_POSITIONS` und URDF-Limits (stewart_2, stewart_5) ist als ⚠ TBD geflaggt, nicht stillschweigend geglättet
- [ ] Bekannte Konflikt-Konstellationen (Antennen-Crossing, Antennen-Stop, IK-Polytop-Verletzung) sind aufgeführt, mit klarer Trennung in „vom IK gefangen" vs. „vom IK ignoriert"
- [ ] IK-Bisection-Tabelle (Phase A) ist mit allen 12 Extrem-Posen + URDF-limit-triggernden Joints aufgeführt, klar als „nicht mechanisch sicher" markiert
- [ ] „Drei Schichten der Validität" (IK-Polytop, URDF-Limits, Pollen-Nominal-Range) sind als verbindliche Prüfreihenfolge dokumentiert
- [ ] Phase-B-Vorfall vom 2026-05-12 ist als Selbstkollisions-Warnung mit Sequenz und Folgezustand (`backend_status.ready: false`, `head_joints: null`) dokumentiert
- [ ] Motor-Mode-Namen sind korrekt benannt: `enabled` / `disabled` / `gravity_compensation`, nicht `stiff` / `compliant`
- [ ] `backend_status.ready`-Check ist als Pre-Flight-Pflicht vor jeder Live-Bewegung benannt
- [ ] DE- und EN-Version sind strukturell synchron
- [ ] Jede konkrete Zahl trägt eine Quelle (URDF, `analytical_kinematics.py`, `reachy_mini.py`, `kinematics_data.json`, Pollen-Datasheet, Phase-A-IK-Sweep)

## Referenzen

- Live-Quelle der Joint-Limits: `http://<daemon-host>:8000/api/kinematics/urdf` (siehe [`spec/reachy-mini/daemon-rest-api/`](../daemon-rest-api/de.md))
- URDF-Quelle: [`src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/descriptions/reachy_mini/urdf/robot.urdf)
- Kinematik-Konstanten und Solver: [`src/reachy_mini/kinematics/analytical_kinematics.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/kinematics/analytical_kinematics.py)
- Kinematik-Geometrie: [`src/reachy_mini/assets/kinematics_data.json`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/assets/kinematics_data.json)
- Kanonische Posen + `wake_up`/`goto_sleep`: [`src/reachy_mini/reachy_mini.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)
- Pollen-Hardware-Datasheet (Nominal Operations Range, DOF-Table): <https://huggingface.co/docs/reachy_mini/platforms/reachy_mini/hardware>
- Verwandte Specs im Repo: [`reachy-mini/control-surface`](../control-surface/de.md) (Inventar + Bewegungs-Design), [`reachy-mini/daemon-rest-api`](../daemon-rest-api/de.md) (REST-Endpoints), [`reachy-mini/motions/`](../motions/) (konkrete Bewegungsabläufe)
- Konsumierende Skills: [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md), [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/de.md), [`dance-choreography`](../../claude/dance-choreography/de.md), [`app-scaffold`](../../claude/app-scaffold/de.md)

## Offene Fragen

- Sind die Stewart-Joint-Werte aus `SLEEP_HEAD_JOINT_POSITIONS` (`stewart_2 = +72.32°`, `stewart_5 = −70.84°`) konsistent mit den engeren URDF-Limits (`+70°` bzw. `−70°`)? Wenn nein, ist die SDK-Konstante stale, die URDF-Grenze zu konservativ oder die Sleep-Pose grundsätzlich am Rand des Polytops gewählt? Klärung am realen Gerät und ggf. mit Pollen abstimmen
- Wie groß ist der reale Workspace für Head-Translationen x, y, z im IK-erreichbaren Polytop? Ein kurzer Sweep-Skript am realen Gerät kann die ⚠ TBD-Werte in Schicht 2 pinnen
- Gibt es eine geometrische „Crossing-Forbidden-Zone" für die Antennen, in der sich linke und rechte Antenne mechanisch berühren? Vermessen
- Sollte `goto_sleep` die Antennen-Motoren am Ende in `compliant` umschalten, um Drift am Endanschlag zu vermeiden? Empfehlung an Pollen, oder Workaround in unserem `reachy-mini-inspect`/`reachy-mini-start`-Pfad? Hier nur dokumentieren, nicht entscheiden
- Sind die `max_relative_yaw=65°` und `max_body_yaw=160°`-Konstanten in zukünftigen SDK-Versionen stabil, oder werden sie konfigurierbar? Bei einem Bump dieser Werte muss die Spec neu verifiziert werden
- Welche Geschwindigkeits- und Beschleunigungs-Profile fährt der `goto_target`-Default? Das URDF nennt nur Velocity = 8 rad/s als Hard-Cap, aber der reale Profil-Generator dürfte glätten. Bei Bedarf in einer Folge-Spec dokumentieren
- Soll diese Spec einen Drift-Check-Skript-Verweis bekommen (z. B. `scripts/check-motor-positions-spec.py`), analog zur `daemon-rest-api`-Spec? Sinnvoll, wenn das URDF in einer SDK-Version geändert wird
