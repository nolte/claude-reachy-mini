# Reachy Mini daemon REST API

Status: draft

## Context
The Pollen daemon (`reachy_mini` package, a FastAPI app) is the central server that controls every Reachy Mini device (Wireless, Lite) locally. It exposes an HTTP/REST surface at `http://<host>:8000/` whose only authoritative description is the live-served `openapi.json`. Existing specs in this repo document only the endpoint subsets that are immediately relevant to their topic — `app-architecture` covers `/api/apps/*` and parts of `/api/daemon/*`, `mcp-server` lists individual read endpoints as MCP tool mappings, `host-provisioning` references the lock status. A central inventory of all endpoints against which drift can be measured has been missing. This spec consolidates the full endpoint inventory (state: live-verified against `http://reachy-mini.local:8000/openapi.json`, OpenAPI 3.1.0, FastAPI info title `FastAPI`, version `0.1.0`), grouped by namespace and with a brief usage note per group.

## Goals
- Every HTTP endpoint the daemon exposes is listed here with method, path, and summary
- Each namespace has a usage note that clarifies *what* the endpoint family is for and which existing skills/agents consume it
- Other specs may point at this list instead of maintaining their own endpoint tables
- Drift against the live `openapi.json` is detectable via an automated check (existence-level, not schema-diff)
- The tables are intended as an existence inventory — request/response schemas remain the domain of the live OpenAPI

## Non-Goals
- Full request/response schemas per endpoint — the `openapi.json` stays the sole authoritative source for those
- Daemon implementation details (router wiring, dependency injection)
- WebSocket surfaces of the daemon — OpenAPI 3.x does not describe WebSockets; if any exist, they belong in a separate spec
- Endpoints from third-party systems (Home Assistant Core, Hugging Face Hub) — even when syntactically `/api/...`, they are not in scope here
- REST-API migration or versioning policy

## Requirements

### Authoritative source and maintenance

- The endpoints listed in this spec **MUST** match the `openapi.json` of a running daemon reachable at `http://<daemon-host>:8000/openapi.json`
- Any change to this spec **MUST** be motivated by a live OpenAPI fetch and the pull-request body **MUST** name the daemon host used plus the date
- A reference to an endpoint from another spec, skill, or agent **MAY** point at this spec and **MUST NOT** maintain its own existence table on top
- This spec **MUST NOT** duplicate request or response schemas; it lists only (method, path, summary)

### Endpoint inventory

The tables below mirror every HTTP endpoint served by the daemon. Method and path are copy-pasteable as string literals; path parameters (`{name}`) follow FastAPI conventions.

#### Apps — app lifecycle (`/api/apps/*`)

Lifecycle surface for daemon-managed apps: install, list, start, stop, update, remove. Consumed by `app-architecture`, `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`, `app-log-triage`.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/apps/check-updates` | Check App Updates |
| `GET` | `/api/apps/current-app-status` | Current App Status |
| `POST` | `/api/apps/install` | Install App |
| `POST` | `/api/apps/install-private-space` | Install Private Space |
| `GET` | `/api/apps/job-status/{job_id}` | Job Status |
| `GET` | `/api/apps/list-available` | List All Available Apps |
| `GET` | `/api/apps/list-available/{source_kind}` | List Available Apps |
| `POST` | `/api/apps/remove/{app_name}` | Remove App |
| `POST` | `/api/apps/restart-current-app` | Restart App |
| `POST` | `/api/apps/start-app/{app_name}` | Start App |
| `POST` | `/api/apps/stop-current-app` | Stop App |
| `POST` | `/api/apps/update/{app_name}` | Update App |

#### Daemon — daemon lifecycle and app lock (`/api/daemon/*`)

Controls the daemon itself (start/stop/restart) and reads the app lock that today allows only one app at a time. Consumed by the robot-busy check in `reachy-mini-deploy`, `reachy-mini-on-device`, `host-provisioning`, `mcp-server-bootstrap`.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/api/daemon/restart` | Restart Daemon |
| `GET` | `/api/daemon/robot-app-lock-status` | Get Robot App Lock Status |
| `POST` | `/api/daemon/start` | Start Daemon |
| `GET` | `/api/daemon/status` | Get Daemon Status |
| `POST` | `/api/daemon/stop` | Stop Daemon |

#### Move — motion and move playback (`/api/move/*`)

Primary motion surface: direct goto, continuous set-target streaming, stop, wake-up / goto-sleep, and playback of recorded move datasets. Consumed by `reachy-mini-sdk`, `dance-choreography`, `home-assistant-bridge` (via the MCP layer), and any behavior that does not directly use the `ReachyMini` Python object.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/api/move/goto` | Goto |
| `POST` | `/api/move/play/goto_sleep` | Play Goto Sleep |
| `POST` | `/api/move/play/recorded-move-dataset/{dataset_name}/{move_name}` | Play Recorded Move Dataset |
| `POST` | `/api/move/play/wake_up` | Play Wake Up |
| `GET` | `/api/move/recorded-move-datasets/list/{dataset_name}` | List Recorded Move Dataset |
| `GET` | `/api/move/running` | Get Running Moves |
| `POST` | `/api/move/set_target` | Set Target |
| `POST` | `/api/move/stop` | Stop Move |

#### State — sensor and pose reads (`/api/state/*`)

Pure read surface for current pose, body yaw, antenna joint positions, direction-of-arrival (DoA), and the aggregated full state. Consumed by `reachy-mini-sdk` (`mini.imu`-style access for REST clients), `mcp-server-bootstrap`, `home-assistant-bridge` (for sensor entities).

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/state/doa` | Get Doa |
| `GET` | `/api/state/full` | Get Full State |
| `GET` | `/api/state/present_antenna_joint_positions` | Get Antenna Joint Positions |
| `GET` | `/api/state/present_body_yaw` | Get Body Yaw |
| `GET` | `/api/state/present_head_pose` | Get Head Pose |

#### Motors — motor mode and diagnostics (`/api/motors/*`)

Switch between motor modes (e.g. compliance / stiffness) and read motor status for diagnostics. Touches hardware safety directly — consumers **SHOULD** treat mode switching as a rare, deliberate operation, not a per-loop step.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/api/motors/set_mode/{mode}` | Set Motor Mode |
| `GET` | `/api/motors/status` | Get Motor Status |

#### Media — audio output, sounds, and mic lock (`/api/media/*`)

Acquire/release on the media subsystem (exclusive lock for the audio path), sound management (upload, list, delete), playback (start, stop). Consumed by `audio-beat-tracking`, `dance-choreography`, `app-scaffold` (sound-asset conventions).

| Method | Path | Summary |
|---|---|---|
| `POST` | `/api/media/acquire` | Acquire Media |
| `POST` | `/api/media/play_sound` | Play Sound |
| `POST` | `/api/media/release` | Release Media |
| `GET` | `/api/media/sounds` | List Sounds |
| `POST` | `/api/media/sounds/upload` | Upload Sound |
| `DELETE` | `/api/media/sounds/{filename}` | Delete Sound |
| `GET` | `/api/media/status` | Media Status |
| `POST` | `/api/media/stop_sound` | Stop Sound |

#### Volume — speaker and microphone volume (`/api/volume/*`)

Read/write speaker and microphone volume, plus a test-sound trigger. Consumed by `home-assistant-bridge` (volume entities) and bring-up routines.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/volume/current` | Get Volume |
| `GET` | `/api/volume/microphone/current` | Get Microphone Volume |
| `POST` | `/api/volume/microphone/set` | Set Microphone Volume |
| `POST` | `/api/volume/set` | Set Volume |
| `POST` | `/api/volume/test-sound` | Play Test Sound |

#### Camera — camera introspection (`/api/camera/*`)

As of v0.1.0 only a specs query; actual frame streams are not REST content. Consumers that need images speak to the corresponding streaming channel (out of scope for this spec).

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/camera/specs` | Get Camera Specs |

#### Kinematics — URDF and STL (`/api/kinematics/*`)

Kinematics metadata, URDF file, and STL meshes. Consumed by sim/visualization and by `reachy-mini-sdk` for collision/limit validation.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/kinematics/info` | Get Kinematics Info |
| `GET` | `/api/kinematics/stl/{filename}` | Get Stl File |
| `GET` | `/api/kinematics/urdf` | Get Urdf |

#### HF-Auth — Hugging Face OAuth and tokens (`/api/hf-auth/*`)

OAuth flow against Hugging Face, token persistence, status queries for the "central robot" and the relay. Consumed by Spaces / private-app installs (`/api/apps/install-private-space`) and any consumer that triggers HF-authenticated actions.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/hf-auth/central-robot-status` | Get Central Robot Status |
| `GET` | `/api/hf-auth/oauth/callback` | Oauth Callback |
| `GET` | `/api/hf-auth/oauth/configured` | Is Oauth Configured |
| `DELETE` | `/api/hf-auth/oauth/session/{session_id}` | Cancel Oauth Session |
| `GET` | `/api/hf-auth/oauth/start` | Start Oauth |
| `GET` | `/api/hf-auth/oauth/status/{session_id}` | Get Oauth Status |
| `GET` | `/api/hf-auth/relay-status` | Get Relay Status |
| `POST` | `/api/hf-auth/save-token` | Save Token |
| `GET` | `/api/hf-auth/status` | Get Auth Status |
| `DELETE` | `/api/hf-auth/token` | Delete Token |

#### Cache — system caches (`/cache/*`)

Reset Hugging Face cache and app cache. Operations are destructive (they delete local files) — consumers **SHOULD** invoke them only on explicit user request, never as an auto-recovery path.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/cache/clear-hf` | Clear Huggingface Cache |
| `POST` | `/cache/reset-apps` | Reset Apps |

#### Wifi — Wireless network configuration (`/wifi/*`)

Scan, connect, forget, setup-hotspot, status, error handling. Practically only relevant on the Wireless variant; on Lite the WLAN belongs to the host PC.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/wifi/connect` | Connect To Wifi Network |
| `GET` | `/wifi/error` | Get Last Wifi Error |
| `POST` | `/wifi/forget` | Forget Wifi Network |
| `POST` | `/wifi/forget_all` | Forget All Wifi Networks |
| `POST` | `/wifi/reset_error` | Reset Last Wifi Error |
| `POST` | `/wifi/scan_and_list` | Scan Wifi |
| `POST` | `/wifi/setup_hotspot` | Setup Hotspot |
| `GET` | `/wifi/status` | Get Wifi Status |

#### Update — system updates (`/update/*`)

Read endpoints (`available`, `info`, `install-source`, `validate-ref`) plus two mutation endpoints (`start`, `start-from-ref`). Consumed by the `host-provisioning` pull-service spec that coordinates the update path against the app lock.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/update/available` | Available |
| `GET` | `/update/info` | Get Update Info |
| `GET` | `/update/install-source` | Install Source |
| `POST` | `/update/start` | Start Update |
| `POST` | `/update/start-from-ref` | Start Update From Ref |
| `GET` | `/update/validate-ref` | Validate Ref |

#### Health-Check — liveness probe (`/health-check`)

Single endpoint; invoked via `POST`. Suitable for wait loops in `mcp-server-bootstrap` and for external monitoring probes.

| Method | Path | Summary |
|---|---|---|
| `POST` | `/health-check` | Health Check |

#### Dashboard / HTML pages (`/`, `/logs`, `/settings`)

HTML pages served by the daemon UI; no JSON, **SHOULD NOT** be consumed as programmatic endpoints. Listed here for existence only, so documentation drift does not produce false "endpoint missing" warnings.

| Method | Path | Summary |
|---|---|---|
| `GET` | `/` | Dashboard |
| `GET` | `/logs` | Logs Page |
| `GET` | `/settings` | Settings |

### Drift detection

- A programmatic check **MUST** be possible that reconciles every (method, path) from the live `openapi.json` against the tables in this spec
- The check **SHOULD** be wired up as a small script under `scripts/` (e.g. `scripts/check-daemon-rest-api-spec.py`) once the spec leaves `draft` status
- On divergence the check **SHOULD** exit non-zero with a list of affected endpoints; it **MAY** be wired into `task lint`

## Acceptance Criteria

- [ ] Spec exists at `spec/reachy-mini/daemon-rest-api/de.md` (canonical) and `spec/reachy-mini/daemon-rest-api/en.md` (translation)
- [ ] Every endpoint from `http://reachy-mini.local:8000/openapi.json` (state 2026-05-12, OpenAPI 3.1.0, version `0.1.0`, 79 operations) is listed at least once in the DE file
- [ ] Every endpoint table lists method, path, and summary in exactly that order
- [ ] Every namespace section ships a usage note that is readable without schema knowledge
- [ ] The spec explicitly cites the live `openapi.json` as authoritative and forbids schema duplication
- [ ] Paths use FastAPI parameter notation `{name}` rather than `<name>` or free-form text
- [ ] DE and EN list the same endpoint set; diffing the `|` table rows yields only language differences in note/summary, never missing or extra paths
- [ ] Existing specs that duplicate single endpoint families (`app-architecture` for `/api/apps/*`) keep their content; new drift is fixed here instead

## References

- Authoritative live source: `http://<daemon-host>:8000/openapi.json` — typical hosts: `http://reachy-mini.local:8000` (Wireless), `http://127.0.0.1:8000` (Lite or sim)
- Daemon source (router wiring, implementation truth): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollen's hosted REST API docs: <https://huggingface.co/docs/reachy_mini/API/rest-api> — note that this doc can lag the live OpenAPI
- Cross-references within this repo: `spec/reachy-mini/app-architecture/` (app lifecycle), `spec/reachy-mini/mcp-server/` (MCP wrapper), `spec/reachy-mini/ha-integration/` (HA bridge), `spec/reachy-mini/host-provisioning/` (update and pull-service path)

## Open Questions

- Which endpoints are platform-specific (Wireless only: `/wifi/*`; Lite only: ?) and should be marked in a dedicated column? Verify on first Lite hardware contact
- Does the daemon expose a WebSocket surface (e.g. for streaming pose targets, an event bus)? If yes, this belongs in a separate spec — the present one stays REST-only
- Should the drift check live as a pre-commit hook or as a CI step? Pre-commit is fast but fails without network access to the device — CI with a mocked `openapi.json` is likely more robust
- Is `version: 0.1.0` in the FastAPI info block the actual daemon release version, or does it stay constant independent of releases? Link consuming specs at the first observed bump
