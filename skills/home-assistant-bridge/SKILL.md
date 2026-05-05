---
name: home-assistant-bridge
description: >-
  Knowledge and idiom base for bidirectional integration between Reachy Mini
  and Home Assistant. Activate on any task touching HA REST endpoints
  (`/api/states`, `/api/services`, `/api/events`, webhooks), the HA WebSocket
  API, a Long-Lived Access Token, HA service calls like `light.turn_on` or
  `notify.send_message`, or webhook reception in a Reachy context. Trigger
  phrasings include "trigger Reachy from a Home Assistant automation", "call
  an HA service from a behavior", "subscribe to HA state_changed and react",
  "secure the HA token". Do not activate on Home Assistant custom-component
  development, on Reachy SDK motion tasks without an HA touchpoint, or on
  pure audio-processing tasks.
tags: [reachy-mini, home-assistant, integration, security]
---

# Home Assistant Bridge

Pinned HA minimum version: **`homeassistant>=<TBD>`** — set this on the first integration with the target HA instance and bump it through a deliberate spec revision afterward.

Security defaults are non-negotiable: **TLS verification on, tokens never inlined, tokens never logged unmasked, `.env` in `.gitignore`**.

> ⚠ TBD: validate against current Home Assistant API — every concrete endpoint shape, payload, and behavior below is faithful to the official docs at the time of writing, but must be confirmed against the target HA version before relying on it.

## When this skill activates

- HTTP calls against `/api/states`, `/api/services`, `/api/events`, or `/api/webhook/<id>`
- WebSocket connections to `/api/websocket` for `subscribe_events`, `call_service`, `get_states`
- Long-Lived Access Token usage in an `Authorization: Bearer …` header
- Reachy behavior code that reacts to HA events or invokes HA services

## When NOT to activate

- developing a Home Assistant custom component (lives in HA's own integrations tree)
- Reachy SDK motion work without HA → use `reachy-mini-sdk`
- audio analysis / beat tracking → `audio-beat-tracking` (planned)
- live on-device test → agent `reachy-mini-on-device` (planned)

## Source of truth

In order, on conflict:

1. The official HA developer docs:
   - REST: <https://developers.home-assistant.io/docs/api/rest/>
   - WebSocket: <https://developers.home-assistant.io/docs/api/websocket/>
2. The running HA instance you're integrating against (responses are the ground truth)
3. This skill (curated summary; loses to the docs and the live instance)

Before generating endpoint-shaped code, **read the relevant section of the developer docs** and confirm the exact payload — HA evolves between minor releases.

## REST API

| Operation | Method + Path | Notes |
|---|---|---|
| Read all states | `GET /api/states` | Returns every entity's state |
| Read one state | `GET /api/states/<entity_id>` | 404 when entity missing |
| Call a service | `POST /api/services/<domain>/<service>` | JSON body is the service data |
| Fire an event | `POST /api/events/<event_type>` | JSON body becomes event data |
| Webhook | `POST /api/webhook/<id>` | Defined by HA `webhook` trigger; body schema is yours |

Auth header on every call:

```
Authorization: Bearer <LONG_LIVED_ACCESS_TOKEN>
Content-Type: application/json
```

Treat 4xx / 5xx as failures and back off exponentially on retry. Do **not** poll states — use the WebSocket `subscribe_events` instead.

## WebSocket API

Connect to `wss://<host>/api/websocket` (or `ws://` only on a trusted local network). The handshake is:

1. Server sends `{"type": "auth_required", "ha_version": "..."}`
2. Client sends `{"type": "auth", "access_token": "<token>"}`
3. Server replies `{"type": "auth_ok", ...}` (or `auth_invalid` — close & escalate)

After auth, every command needs an incrementing `id` and gets a matching `result`:

```json
{"id": 1, "type": "subscribe_events", "event_type": "state_changed"}
{"id": 2, "type": "call_service", "domain": "light", "service": "turn_on", "service_data": {"entity_id": "light.kitchen"}}
```

Reconnect with backoff on close. After reconnect, **resync state** (`get_states`) and re-issue every subscription — IDs reset, subscriptions don't carry over.

## Patterns

### HA → Reachy

```python
# > ⚠ TBD: validate end-to-end against the target HA instance and SDK
import os
import aiohttp  # canonical WS client for this skill (see Open Questions in the spec)

HA_URL = os.environ["HA_URL"]                    # e.g. https://ha.local:8123
HA_TOKEN = os.environ["HA_TOKEN"]                # Long-Lived Access Token

async def listen_and_react(reachy):
    async with aiohttp.ClientSession() as session:
        async with session.ws_connect(f"{HA_URL.replace('http', 'ws')}/api/websocket") as ws:
            await ws.receive_json()                                       # auth_required
            await ws.send_json({"type": "auth", "access_token": HA_TOKEN})
            assert (await ws.receive_json())["type"] == "auth_ok"
            await ws.send_json({"id": 1, "type": "subscribe_events", "event_type": "state_changed"})
            async for msg in ws:
                event = msg.json().get("event", {}).get("data", {})
                if event.get("entity_id") == "binary_sensor.front_door" and event["new_state"]["state"] == "on":
                    await reachy.head.goto(pan=0, tilt=-0.2, roll=0, duration=0.4)  # see reachy-mini-sdk
```

### Reachy → HA

```python
# > ⚠ TBD: validate against the target HA instance
import os
import httpx

HA_URL = os.environ["HA_URL"]
HA_TOKEN = os.environ["HA_TOKEN"]
HEADERS = {"Authorization": f"Bearer {HA_TOKEN}", "Content-Type": "application/json"}

async def dim_lights_for_dance():
    async with httpx.AsyncClient(base_url=HA_URL, headers=HEADERS, timeout=5.0) as client:
        r = await client.post("/api/services/light/turn_on",
                              json={"entity_id": "light.living_room", "brightness_pct": 40})
        r.raise_for_status()
```

Reuse one `httpx.AsyncClient` and one WebSocket connection per bridge lifetime — connection pooling matters at behavior-tick frequencies.

## Configuration & secrets

- Read `HA_URL`, `HA_TOKEN`, and any per-entity IDs from env vars or `.env`. Never inline.
- `.env` is in `.gitignore`; the contract for required env vars lives in `.env.example`.
- Mint Long-Lived Access Tokens in the HA UI under your user profile → "Long-Lived Access Tokens".

## Security hard rules

- **MUST** keep TLS verification on. No `verify=False` on `httpx.AsyncClient`. For local CAs, ship a CA bundle and pass it via `verify="/path/to/ca.pem"` instead.
- **MUST NOT** log a token. If a log line is unavoidable, mask: `f"{token[:4]}…{token[-2:]}"`.
- **MUST NOT** show `requests`-based examples — `requests` is blocking and a poor fit for the behavior loop.
- **MUST** delegate motion idioms to `reachy-mini-sdk` rather than re-explaining them here.

## Drift check

- Re-validate every section against the HA docs on each major HA release, or whenever an example fails against the live instance — whichever comes first.
- Bump the pinned minimum HA version explicitly in the header above; never let it silently fall behind.
- On conflict between this skill and the official HA docs, the docs win.

## Boundaries to neighbouring skills

- Reachy motion idioms → `reachy-mini-sdk`
- Scaffolding a brand-new behavior triggered from HA → `app-scaffold`
- Audio / beat / tempo for dance behaviors → `audio-beat-tracking` (planned)
- Live on-device test of an HA-triggered motion → agent `reachy-mini-on-device` (planned)
