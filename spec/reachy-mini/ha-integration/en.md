# Home Assistant Integration: Architecture

Status: draft

## Context
Reachy Mini should be a native smart-home resident in Home Assistant (HA) — bidirectional, without bridge code in third-party systems, with all HA features that matter for smart-home use: voice assist pipeline, camera stream, media player, behavior triggers via service calls, telemetry as sensors, behaviors as reactions to automations. This specification defines how the integration is distributed, discovered, and how it communicates; which layers it carries; and how HA and Reachy meet on the entity surface. It is the architectural source against which the `home-assistant-bridge` skill, the `app-scaffold` skill, and the `reachy-mini-on-device` agent calibrate their suggestions.

## Goals
- Reachy Mini appears in HA as a native custom-component entry, installable via HACS
- Maximum HA feature coverage: entities, services, events, voice assist pipeline, camera stream, media player
- Voice assist as the primary use case: Reachy is a Wyoming voice satellite with local wake-word detection, embedded in the HA voice assist pipeline
- Discovery and setup without YAML — config flow plus mDNS / Zeroconf
- Secure by default: TLS, token auth, spam protection on service calls
- Clearly separated: voice layer (Wyoming) and robotics layer (REST/WebSocket via the Reachy daemon) are orthogonal layers, joined in the HA glue layer

## Non-Goals
- Own wake-word engine, own STT, own TTS — the HA voice assist pipeline is the source of truth
- Cloud dependency as a requirement: everything local must run without internet access
- HA add-on packaging — the integration is a custom component, not an add-on
- Other smart-home hubs (Hubitat, openHAB, SmartThings) — `home-assistant-bridge` only covers HA
- A recording-stream editor or behavior-authoring UI in HA — that belongs in a separate app, not in the integration

## Requirements

### Distribution form
- **MUST** ship as an HA custom integration in its own repository (`nolte/reachy-mini-hass` proposed), layout `custom_components/reachy_mini/` with `manifest.json` and `hacs.json`
- **MUST** be installable via HACS
- **MUST NOT** live in the same repository as the `claude-reachy-mini` plugin — the plugin skills are the toolbox; the integration is the app built with that toolbox
- **SHOULD** use semantic versioning, pinned to the supported `reachy_mini` SDK version

### Architectural layers
Three clearly separated layers:

1. **Voice layer (Wyoming)** — Reachy Mini as a Wyoming voice satellite. Audio in (4× PDM MEMS mic array, 16 kHz) and audio out (5 W @ 4 Ω speaker) talk to HA via the Wyoming protocol. The HA voice assist pipeline (wake word → STT → intent → TTS) handles all speech processing.
2. **Robotics layer (REST/WebSocket)** — pose, joint control, behavior playback, telemetry, and camera run via the official Reachy daemon API. Conventions for auth, reconnect, backpressure follow the `home-assistant-bridge` skill.
3. **HA glue layer (custom integration)** — joins both layers, exposes HA entities/services/events, and coordinates voice-triggered behaviors with robot motion.

This separation is mandatory: voice-pipeline changes (e.g. a new wake-word engine in HA) must not affect robotics code, and robotics updates (e.g. new behaviors) must not break the voice stack.

### Discovery and setup
- **MUST** announce a Zeroconf/mDNS service `_reachy_mini._tcp.local.` so HA discovers the robot automatically
- **MUST** ship an HA config flow — no YAML setup
- **MUST** capture in the config flow: hostname / IP, port, long-lived access token (Reachy daemon), detected platform (Wireless / Lite / Simulation), preferred voice pipeline (HA pipeline selector), default wake word
- **SHOULD** trigger a test motion (`wake_up`) on first setup so the user verifies the hardware connection immediately
- **MUST** support multiple parallel config entries for multiple Reachy Mini devices (each as its own device)

### Connection to the Reachy daemon
- **MUST** use REST for synchronous actions (pose command, behavior trigger, status queries)
- **MUST** use WebSocket for telemetry streams (`JointPositionsMsg`, `HeadPoseMsg`, `ImuDataMsg` at 50 Hz)
- **MUST** implement reconnect with exponential backoff — a lost connection must not block HA startup; after reconnect: state resync via `get_states`
- **MUST** attach the long-lived access token in the `Authorization` header on every request
- **MUST NOT** print the token in HA logs — masking is mandatory (`token[:4] + "…"`)

### HA entity surface

| Entity | Type | Description |
|---|---|---|
| `camera.reachy_mini_head` | Camera | head camera stream (Sony IMX708, 12 MP, autofocus); WebRTC backend |
| `media_player.reachy_mini_speaker` | Media Player | 5 W @ 4 Ω speaker, volume 0–100, accepts TTS |
| `light.reachy_mini_led_ring` | Light | LED ring on the mic module (`LED_EFFECT`, `LED_BRIGHTNESS`, `LED_GAMMIFY`, `LED_SPEED`) |
| `sensor.reachy_mini_imu_*` | Sensor | accelerometer, gyroscope, quaternion, temperature — **Wireless only** |
| `sensor.reachy_mini_battery` | Sensor | battery percentage — **Wireless only** |
| `sensor.reachy_mini_head_pose` | Sensor | current head pose (4×4 matrix as attribute, Euler angles as state) |
| `select.reachy_mini_idle_mode` | Select | options: `waiting-idle` / `alert-listening` / `thinking` / `off` |
| `select.reachy_mini_behavior` | Select | choose one of the 29 motion slugs as a one-shot trigger |
| `switch.reachy_mini_motors_enabled` | Switch | motors on/off (`enable_motors` / `disable_motors`) |
| `switch.reachy_mini_gravity_compensation` | Switch | gravity compensation on the head motors on/off |
| `switch.reachy_mini_automatic_body_yaw` | Switch | `set_automatic_body_yaw` on/off |
| `binary_sensor.reachy_mini_person_detected` | Binary Sensor | person detected in the camera frame |
| `binary_sensor.reachy_mini_sound_detected` | Binary Sensor | sound detected in the mic array (DOA-capable) |
| `binary_sensor.reachy_mini_motion_active` | Binary Sensor | a behavior is currently running |
| `update.reachy_mini_firmware` | Update | current daemon version vs. latest available |
| `assist_satellite.reachy_mini` | Assist Satellite | Wyoming voice-satellite integration (HA `assist_satellite` domain, stable since `2024.10`) |

- **MUST** create every entity with the correct `device_class` and `state_class` so long-term statistics work in HA
- **SHOULD** honour platform profiles: IMU and battery sensors do not appear on Lite or Simulation

### Number-entity setpoint semantics

HA sliders (NumberEntity) follow a **setpoint model**, not a hardware-position model: the slider value reflects the **user intent**, not the live servo reading. Ignoring this builds a visible off-by-one echo bug into the slider, because HA polls the entity state for validation right after every push and writes the returned value back into the slider.

- **MUST** the setter of a number entity (antennas, head-x/y/z, head-roll/pitch/yaw, body-yaw and any other pose axis) write the app-side state setpoint **synchronously, in the same tick** the setter runs in. If the app's architecture delegates the actual hardware command asynchronously through a command queue to a control loop, the state **MUST** still be written synchronously before the setter returns — not only on the next queue drain
- **MUST** the getter of the same number entity return the **app-side state setpoint**, **not** the live hardware joint position read back from the daemon; the hardware lags the setpoint by hundreds of milliseconds because of servo kinematics, while the state read is always faster than the physical motion. The slider shows user intent, not servo lag
- **MUST** for the same axis, the setter's write target and the getter's read source reference the same state field — asymmetry between the two (setter writes app state, getter reads hardware) produces the off-by-one echo symptom
- **MUST NOT** the NumberEntity implementation, in the same handle block of a `NumberCommandRequest`, yield the state echo response before the state has been updated; the order must be write-first, yield-second
- **SHOULD** the pattern be encapsulated in a helper / mixin in the app code base so every pose axis (antennas, head axes, body yaw) follows the same synchronous setter/getter scheme. Otherwise the pattern is latently broken again for every new pose axis

### Custom services
- **MUST** expose the following services:
  - `reachy_mini.play_behavior(behavior: str, speed: float = 1.0)` — start one of the 29 motion specs
  - `reachy_mini.cancel_behavior()` — abort the currently running behavior
  - `reachy_mini.goto_pose(x_mm, y_mm, z_mm, roll_deg, pitch_deg, yaw_deg, duration: float = 1.0, method: str = "min_jerk")` — direct pose command with an interpolation mode from the `InterpolationTechnique` enum
  - `reachy_mini.set_antennas(left_deg: float, right_deg: float, duration: float = 0.5)`
  - `reachy_mini.set_body_yaw(value_deg: float, duration: float = 0.5)`
  - `reachy_mini.look_at_world(x: float, y: float, z: float, duration: float = 0.5)` — world-frame tracking
  - `reachy_mini.look_at_image(u: int, v: int, duration: float = 0.5)` — pixel tracking on the camera
  - `reachy_mini.speak(message: str, voice: str = None, behavior_during: str = None)` — TTS via the HA pipeline on the speaker, optionally a concurrent behavior
  - `reachy_mini.set_idle_mode(mode: str)` — switch the idle state
  - `reachy_mini.wake_up()` / `reachy_mini.goto_sleep()` — high-level SDK methods
  - `reachy_mini.start_recording()` / `reachy_mini.stop_recording()` — trajectory capture, returned as an HA event
- **MUST** validate every service against an HA service schema (`vol.Schema`) — no unchecked input
- **SHOULD** carry rate limits against service spam: at most 10 behaviors / minute, at most one `goto_pose` / 0.5 s

### Custom events
- **MUST** fire the following HA events:
  - `reachy_mini_behavior_started(behavior: str, source: str)` — `source` ∈ {`service_call`, `voice`, `automation`}
  - `reachy_mini_behavior_finished(behavior: str, status: str)` — `status` ∈ {`PASS`, `FAIL`, `ABORTED`} matching the `reachy-mini-on-device` agent
  - `reachy_mini_wake_word_detected(wake_word: str)`
  - `reachy_mini_voice_command_received(intent: str, slots: dict)` — derived from the HA intent pipeline
  - `reachy_mini_person_detected(confidence: float, bounding_box: dict)`
  - `reachy_mini_sound_direction_detected(angle_deg: float, confidence: float)` — DOA from the mic array
  - `reachy_mini_low_battery(percentage: float)` — Wireless only
  - `reachy_mini_safety_threshold_exceeded(metric: str, value: float)` — e.g. temperature, current, joint limit
- **MUST NOT** fire any event type at more than 5 Hz — the HA state bus would otherwise overflow

### Voice assist (Wyoming)
- **MUST** register Reachy as an HA Wyoming satellite, with audio input (mic array) and audio output (speaker)
- **MUST** run local wake-word detection — engine `microWakeWord` (locally on the RPi 4 CM4) or `openWakeWord` (on the HA host); configurable wake word, default proposal `> ⚠ TBD: pick portfolio default`
- **MUST** run voice-activity detection (VAD) on the audio stream so recording stops on silence
- **MUST** stream audio to HA in 16 kHz, mono, 16-bit PCM (Wyoming standard)
- **MUST** receive TTS audio from HA and play it on the Reachy speaker
- **SHOULD** trigger appropriate behaviors during voice interaction: `alert-listening` while wake word + listening, `thinking` while STT/intent, `recognition` or `agreeing-nod` after a successful intent, `confused` after a failed intent
- **MUST NOT** push audio to the cloud when the HA pipeline is configured locally
- Wyoming protocol docs: <https://github.com/rhasspy/wyoming>

### Audio routing
- **MUST** use the same audio path for Wyoming TTS and `reachy_mini.speak` service calls — no separate path
- **SHOULD** optionally damp input audio during active motion (servo whirring): echo cancellation or mic mute during large pose changes — `> ⚠ TBD: empirically test`
- **SHOULD** drive volume contextually (e.g. quieter at night via an HA sensor)

### Camera stream
- **MUST** expose the camera stream as a WebRTC stream via the Reachy daemon WebRTC API (`media_server.py`, `webrtc_client_gstreamer.py` from the SDK)
- **SHOULD** additionally expose an RTSP endpoint so HA-side object-detection integrations (Frigate, DeepStack, Doods) can consume the stream
- **MAY** offer a snapshot service (`reachy_mini.take_snapshot`) as a convenience

### Behavior mapping to the motion catalogue
- **MUST** accept all 29 motion slugs from `spec/reachy-mini/motions/` as valid values for the `select.reachy_mini_behavior` entity and the `reachy_mini.play_behavior` service
- **MUST** apply the `speed` parameter (0.5–2.0) proportionally to all phase durations — `speed=2.0` halves all durations, `speed=0.5` doubles them
- **MUST NOT** play two behaviors concurrently — a new `play_behavior` call aborts the running behavior via `cancel_move`
- **SHOULD** for loopable behaviors (`thinking`, `alert-listening`, `waiting-idle`, BPM dance blocks) accept a loop count from the HA caller automatically, or run until the next service call

### Reachy → HA patterns
- **MUST** translate telemetry events (person detected, sound detected, low battery) into HA events so automations can react
- **MUST** route Reachy-triggered HA service calls through the HA long-lived access token auth — see `home-assistant-bridge`
- **SHOULD** match telemetry frequency to HA state-update practice — IMU publishes at 50 Hz from the daemon, but the HA sensor should update at most at 1–5 Hz, otherwise HA databases get overloaded (recorder protection)

### Security
- **MUST** enforce TLS verification on the connection between HA and the Reachy daemon (`verify=True`); for local setups with a self-signed cert use a dedicated CA bundle, not `verify=False`
- **MUST** use long-lived access tokens both ways (HA → Reachy and Reachy → HA)
- **MUST** keep tokens in HA secrets storage and not display them as plaintext in the config flow
- **MUST** filter all service calls by HA user permissions — not every HA user may, e.g., call `reachy_mini.disable_motors`
- **SHOULD** implement rate limits against service spam (max. 10 behaviors / minute, max. 1 `goto_pose` / 0.5 s)
- **MUST NOT** push audio streams unencrypted across public networks — Reachy and HA must be on the same LAN or via VPN

### Platform profiles
- **MUST** distinguish the three platforms:
  - **Wireless** — full feature set, IMU available, battery sensor available
  - **Lite** — same actuator set as Wireless, but no IMU telemetry and no battery sensor; corresponding sensor entities are not created
  - **Simulation** — every actuator command accepted, no sensor data (apart from pose read), no audio
- **SHOULD** detect the platform automatically on first setup (`DaemonStatus.no_media` as an indicator for Sim, `DaemonStatus.camera_specs_name` for hardware variant)

### Versioning and compatibility
- **MUST** declare the supported HA version range in `manifest.json` — proposal `>=2024.10` (`assist_satellite` domain stable since `2024.10`); `> ⚠ TBD: validate against current HA voice assist features`
- **MUST** pin the compatible `reachy_mini` SDK version — on an SDK major upgrade the integration is re-validated
- **SHOULD** provide migration paths for config-flow changes via HA `async_migrate_entry`
- **SHOULD** enable Renovate config in the custom-component repo so SDK pin and HA minimum version are tracked automatically

## Acceptance Criteria
- [ ] HA finds Reachy via mDNS automatically and starts the config flow
- [ ] After setup, Reachy appears with all relevant entities in the HA dashboard
- [ ] The Wyoming voice satellite is registered and replies to a wake word with `alert-listening`
- [ ] STT/intent/TTS pipeline runs end-to-end without the cloud
- [ ] All 29 motion slugs are callable as service parameters
- [ ] The camera stream is visible in the HA dashboard (WebRTC)
- [ ] The LED ring reacts to `light.reachy_mini_led_ring` commands
- [ ] IMU sensors publish on Wireless at ~1 Hz HA update rate
- [ ] Reachy → HA service calls work (e.g. `light.turn_on`)
- [ ] `reachy_mini_low_battery` event fires below 20 % battery
- [ ] TLS verification is enabled by default and only relaxed with an explicit CA bundle
- [ ] Tokens are not printed in logs (masking on every code path)
- [ ] Platform profiles correctly hide incompatible entities
- [ ] HACS install works without manual intervention
- [ ] Multiple parallel Reachy Mini devices configurable as separate devices
- [ ] Migration from a previous config-flow version runs without user intervention

## References
- Upstream SDK repo (source of truth for daemon API, IO protocol, media stack): <https://github.com/pollen-robotics/reachy_mini>
- Daemon (REST API, app lock, lifecycle — what the HA integration talks to as the Reachy endpoint): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- IO protocol (`JointPositionsMsg`, `HeadPoseMsg`, `ImuDataMsg`, LED / mic commands like `SetMicrophoneVolumeCmd`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- Media stack (camera, WebRTC via GStreamer, audio DOA, speaker — basis for the `camera.*` and `media_player.*` entities): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- REST API docs (the HA integration primarily talks to these): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx> with OpenAPI schema at <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/openapi.json>
- SDK integration docs (examples for external consumers — analogous to the HA bridge): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/integration.md>
- Wyoming protocol (voice layer, external standard): <https://github.com/rhasspy/wyoming>

## Open Questions
- Which minimum HA version do we set exactly? Proposal `>=2024.10` for the `assist_satellite` domain.
- Which default wake word? "Hey Reachy"? A local model must be trained — `microWakeWord` is trainable, ~30 min effort.
- How is audio latency between Wyoming TTS and behavior trigger synchronised? Honour the `control-surface` pattern (~50 ms audio buffer) — possibly a shared phase-lock logic.
- Should the integration optionally ship cloud AI bindings (OpenAI, Anthropic), or strictly local? Leaning: strictly local in the integration; HA Cloud / Nabu Casa can extend it on the pipeline side.
- Which WebRTC resolution as default? 720p with config to 480p / 1080p.
- Should the HACS repo slug be `reachy-mini-hass` or `home-assistant-reachy-mini`? HACS convention prefers the `<integration-name>-hass` pattern.
- Is there a plugin service API for dance apps (`reachy_mini.start_dance(bpm: float, duration: float)`), or must app code orchestrate the BPM dance blocks directly?
- Which LED-effect vocabulary is exposed in `light.reachy_mini_led_ring` (`solid`, `breathing`, `chasing`, `pulse`)? Depends on ReSpeaker firmware — `> ⚠ TBD`.
- How does the integration behave on a daemon reboot mid-behavior? Proposal: persist current behavior state, drive to the neutral pose after reconnect.
- Should the custom component ship a diagnostics API (`config/diagnostics`) for HA bug reports? HA convention since `2022.x` — should be included.
- How does the integration integrate with the HA energy dashboard? Proposal: a `sensor.reachy_mini_power_consumption` (estimated, or from the daemon if available).
- Should the Wyoming satellite run standalone on Reachy or as a bridge inside the HA custom integration? Leaning: standalone Wyoming server on Reachy (RPi 4 CM4 is strong enough), HA addresses it as a client.
