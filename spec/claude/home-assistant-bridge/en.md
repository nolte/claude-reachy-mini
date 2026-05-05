# Home Assistant Bridge Skill

Status: draft

## Context
Reachy Mini is meant to be a co-resident in a smart-home setup: Home Assistant (HA) triggers Reachy motions (e.g. "nod when the door sensor opens"), and Reachy in turn calls HA services (e.g. "dim the lights when I dance"). Both directions go through the official HA APIs — REST and WebSocket — covering auth, state lookups, service calls, event subscriptions, and webhooks. Hand-rolled glue tends to land on `requests` snippets without reconnect, plaintext tokens, and unsafe TLS workarounds. The `home-assistant-bridge` skill ships the knowledge and idiom base for that integration: it activates exactly when code touches HA APIs in a Reachy context, and produces idiomatic, secure Python code verified against a named HA version.

## Goals
- Claude Code reliably detects when a task touches HA and activates this skill
- Generated code uses the official HA APIs (REST and WebSocket) idiomatically and avoids the usual traps (auth phase, reconnect, backpressure)
- Security defaults are strict: tokens never logged, TLS verified, no `verify=False` in generated code
- Both directions — HA → Reachy and Reachy → HA — are covered by clear patterns
- Version drift against the HA API surfaces visibly instead of disappearing

## Non-Goals
- Developing a Home Assistant custom component inside HA itself (separate repo, separate skill bouquet)
- Reachy motion idiomatics (`reachy-mini-sdk`'s job)
- Scaffolding a new behavior (`app-scaffold`)
- Audio and beat tracking (`audio-beat-tracking`, planned)
- Live deployment / on-device testing (agent `reachy-mini-on-device`, planned)
- General smart-home architecture, MQTT brokers, Zigbee stacks — only the HA interface is in scope

## Requirements

### Triggering and activation
- **MUST** ship a `description` that activates Claude Code as soon as code touches HA REST endpoints (`/api/states`, `/api/services`, `/api/events`, webhooks), the HA WebSocket API, a Long-Lived Access Token, or typical HA terms (entity, service, automation, webhook) in a Reachy context
- **MUST** name the key terms in the description: Home Assistant, HA, REST, WebSocket, service call, entity, webhook, Reachy
- **SHOULD** state explicitly when _not_ to activate (e.g. tasks that handle Reachy SDK motions without HA, or pure HA custom-component development)

### Knowledge base — REST API
- **MUST** document the central REST endpoints: read states (`GET /api/states`, `GET /api/states/<entity_id>`), call services (`POST /api/services/<domain>/<service>`), fire events (`POST /api/events/<event_type>`), webhook endpoints (`POST /api/webhook/<id>`)
- **MUST** describe auth via Long-Lived Access Token in the `Authorization: Bearer <token>` header, including how to mint a token in the HA UI
- **MUST** name error handling (4xx, 5xx, timeouts) and prescribe retry patterns with exponential backoff
- **SHOULD** include performance hints on state polling (prefer WebSocket subscribe over polling)

### Knowledge base — WebSocket API
- **MUST** cover the auth phase: open connection, receive `auth_required`, send `auth`, receive `auth_ok`
- **MUST** document core operations: `subscribe_events` (e.g. `state_changed`), `call_service`, `get_states`
- **MUST** describe reconnect and resync patterns (backoff, state resync after reconnect, idempotent subscriptions)
- **SHOULD** address backpressure: subscriptions can deliver high-frequency events; the Reachy context needs sample / filter strategies
- **MAY** include a tiny sequence diagram of the auth phase

### Patterns
- **MUST** show the "HA → Reachy" pattern: an HA webhook or WebSocket event triggers a Reachy behavior; example with auth, event filtering, and a clean behavior call
- **MUST** show the "Reachy → HA" pattern: a Reachy behavior calls an HA service (e.g. `light.turn_on`, `notify.send_message`) via REST or WebSocket; example with token handling and error path
- **SHOULD** give a long-running-connection pattern: the WebSocket stays open while Reachy plays multiple behaviors
- **MAY** include patterns for state consolidation (multiple HA entities trigger one Reachy behavior)

### HTTP and WebSocket client recommendations
- **MUST** recommend `httpx` as the default for REST calls (async-capable, modern API); REST examples use `httpx.AsyncClient`
- **MUST** name a canonical WebSocket client library (e.g. `websockets` or `aiohttp.ClientSession.ws_connect`); the final choice is recorded in Open Questions
- **MUST NOT** show `requests`-based examples without justification — `requests` is established but blocking and a poor fit for the typical async behavior loop
- **SHOULD** show how a single HTTP client is reused across the bridge's lifetime (connection pooling)

### Configuration and secrets
- **MUST** require tokens and hostnames to be loaded from environment variables or a `.env` file, never inlined into code
- **MUST** document that `.env` lives in `.gitignore` and is never committed; `.env.example` is the contract
- **MUST NOT** ship examples in which a token appears in plaintext in code, log, or commit

### Security
- **MUST** enforce TLS validation as the default (`verify=True` or equivalent); no `verify=False` snippet without explicit, documented justification
- **MUST** require that tokens are never logged — examples show masking (`token[:4] + "…"`) where logging is unavoidable
- **SHOULD** include short hints on token lifetime and token rotation
- **MAY** include hints on HA webhook secret verification, when HA offers it

### Version pinning and drift detection
- **MUST** name a minimum supported HA version in the skill body (e.g. `homeassistant>=<TBD>`); without hardware / HA available, this is TBD-marked
- **SHOULD** define a drift check: API incompatibilities of new HA releases are checked at the next touchpoint, modelled on the `reachy-mini-sdk` drift mechanism
- **MUST** mark unverified assumptions with `> ⚠ TBD: validate against current Home Assistant API`

### Boundaries to neighbouring skills
- **SHOULD** point at `reachy-mini-sdk` whenever the task is dominated by motion idioms
- **SHOULD** point at `app-scaffold` whenever a _new_ behavior is to grow out of an HA trigger
- **SHOULD** point at the agent `reachy-mini-on-device` whenever the HA-triggered motion is to be tested live on the device
- **SHOULD** point at `audio-beat-tracking` whenever Reachy is to react to music streamed independently of HA

## Acceptance Criteria
- [ ] The skill exists at `skills/home-assistant-bridge/SKILL.md` with valid frontmatter (`name: home-assistant-bridge`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The `description` activates on a test prompt that mentions HA endpoints, a Long-Lived Access Token, or the WebSocket API in a Reachy context
- [ ] The knowledge base documents: REST (states, services, events, webhooks), WebSocket (auth phase, subscribe_events, call_service), reconnect patterns
- [ ] Both directions (HA → Reachy, Reachy → HA) are covered by runnable mini-examples
- [ ] Examples use `httpx` for REST and a named WebSocket client; no `requests` example without justification
- [ ] Tokens and hosts are read exclusively from environment variables / `.env` in examples
- [ ] No snippet contains `verify=False`, a plaintext token, or an unmasked token in a log line
- [ ] The minimum supported HA version is visibly documented in the skill body (TBD until verified)
- [ ] Statements without verification carry a `⚠ TBD: validate against current Home Assistant API` marker
- [ ] Out-of-scope concerns (HA custom-component dev, Reachy SDK idioms, behavior scaffold, audio) are marked as "use skill / agent X for this"
- [ ] The MkDocs catalog renders the skill without build errors (`task docs --strict` green)

## Open Questions
- Which minimum supported HA version do we pin initially (`homeassistant>=2024.x` or newer)? To be settled once the target HA instance is known.
- Which WebSocket client is canonical — `websockets`, `aiohttp`, or a third? Leaning: `aiohttp`, since it unifies REST and WebSocket; but `websockets` is leaner. Final choice goes into the skill.
- Should the skill point at the official `homeassistant_api` Python package, or stick to native httpx / websocket calls? Depends on the package's maturity / maintenance state.
- How deep do we go into HA authentication flows beyond Long-Lived Access Tokens (OAuth flow for integrations)? Proposal: LLAT first, OAuth later.
- Should the skill optionally include a webhook-server snippet (FastAPI endpoint receiving an HA webhook), or stay strictly client-side?
- How do we react to HA API breaking changes (endpoint removals between releases)? Agree on a drift-audit cadence.
- Which example services make sense in the body? Proposal: `light.turn_on`, `notify.send_message`, `automation.trigger`, `event.fire`.
- Should the skill specifically address TLS verification for local HA instances without an official certificate (typical in home LANs)? Proposal: yes, with a recommendation to "ship a dedicated CA bundle instead of verify=False".
