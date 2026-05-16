# Home-Assistant-Bridge-Skill

Status: draft

## Kontext
Reachy Mini soll im Smart-Home-Kontext ein Co-Bewohner sein: Home Assistant (HA) triggert Reachy-Bewegungen (z. B. „nicke, wenn der Türsensor öffnet"), und Reachy ruft seinerseits HA-Services auf (z. B. „dimme das Licht, wenn ich tanze"). Beide Richtungen laufen über die offiziellen HA-APIs — REST und WebSocket — mit denen Auth, State-Lookups, Service-Calls, Event-Subscriptions und Webhooks adressiert werden. Wer das ad hoc baut, landet schnell bei `requests`-Snippets ohne Reconnect, Tokens im Klartext und unsicheren TLS-Workarounds. Der Skill `home-assistant-bridge` liefert die Wissens- und Idiom-Basis dafür: er aktiviert genau dann, wenn Code HA-APIs im Reachy-Kontext berührt, und produziert idiomatischen, sicheren Python-Code, der gegen eine namentlich genannte HA-Version verifiziert ist.

## Ziele
- Claude Code erkennt zuverlässig, wann eine Aufgabe HA berührt, und aktiviert dann diesen Skill
- Erzeugter Code nutzt offizielle HA-APIs idiomatisch (REST und WebSocket) und verfehlt nicht typische Stolpersteine (Auth-Phase, Reconnect, Backpressure)
- Sicherheits-Defaults sind streng: Tokens werden nie geloggt, TLS wird verifiziert, kein `verify=False` im generierten Code
- Beide Richtungen — HA → Reachy und Reachy → HA — sind durch klare Patterns abgedeckt
- Versions-Drift gegenüber der HA-API ist sichtbar, statt zu verschwinden

## Nicht-Ziele
- Entwicklung einer Home-Assistant-Custom-Component innerhalb von HA selbst (eigenes Repo, eigenes Skill-Bouquet)
- Reachy-Bewegungs-Idiomatik (Aufgabe von `reachy-mini-sdk`)
- Scaffolding eines neuen Behaviors (`app-scaffold`)
- Audio- und Beat-Tracking (`audio-beat-tracking`, geplant)
- Live-Deployment / On-Device-Test (Agent `reachy-mini-on-device`, geplant)
- Allgemeine Smart-Home-Architektur, MQTT-Brokers, Zigbee-Stacks — nur die HA-Schnittstelle ist im Scope

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine `description` liefern, die Claude Code aktiviert, sobald Code HA-API-Endpoints (`/api/states`, `/api/services`, `/api/events`, Webhooks), die HA-WebSocket-API, einen Long-Lived Access Token oder typische HA-Begriffe (entity, service, automation, webhook) im Reachy-Kontext berührt
- **MUSS [MUST]** Schlüsselbegriffe in der Description nennen: Home Assistant, HA, REST, WebSocket, service call, entity, webhook, Reachy
- **SOLLTE [SHOULD]** explizit benennen, wann _nicht_ zu aktivieren ist (z. B. Aufgaben, die Reachy-SDK-Bewegungen ohne HA-Bezug behandeln, oder reine HA-Custom-Component-Entwicklung)

### Wissensbasis-Inhalt — REST-API
- **MUSS [MUST]** die zentralen REST-Endpoints dokumentieren: States lesen (`GET /api/states`, `GET /api/states/<entity_id>`), Services aufrufen (`POST /api/services/<domain>/<service>`), Events feuern (`POST /api/events/<event_type>`), Webhook-Endpoints (`POST /api/webhook/<id>`)
- **MUSS [MUST]** Auth über Long-Lived Access Token im `Authorization: Bearer <token>`-Header beschreiben, inklusive Token-Generierung in der HA-UI
- **MUSS [MUST]** Fehler-Behandlung benennen (4xx, 5xx, Timeouts) und Retry-Patterns mit Exponential Backoff vorgeben
- **SOLLTE [SHOULD]** Performance-Hinweise zu State-Polling (lieber WebSocket-Subscribe statt Polling) geben

### Wissensbasis-Inhalt — WebSocket-API
- **MUSS [MUST]** die Auth-Phase abdecken: Verbindung öffnen, `auth_required`-Receive, `auth`-Send, `auth_ok`-Receive
- **MUSS [MUST]** die Kern-Operationen dokumentieren: `subscribe_events` (z. B. `state_changed`), `call_service`, `get_states`
- **MUSS [MUST]** Reconnect- und Resync-Patterns beschreiben (Backoff, State-Resync nach Reconnect, idempotente Subscriptions)
- **SOLLTE [SHOULD]** Backpressure adressieren: Subscriptions können hochfrequente Events liefern; der Reachy-Kontext braucht Sample/Filter-Strategien
- **KANN [MAY]** ein Mini-Sequenz-Diagramm der Auth-Phase enthalten

### Patterns
- **MUSS [MUST]** das Pattern „HA → Reachy" zeigen: HA-Webhook oder WebSocket-Event triggert ein Reachy-Behavior; Beispiel mit Auth, Event-Filterung und sauberem Behavior-Aufruf
- **MUSS [MUST]** das Pattern „Reachy → HA" zeigen: Reachy-Behavior ruft einen HA-Service (z. B. `light.turn_on`, `notify.send_message`) über REST oder WebSocket; Beispiel mit Token-Handling und Fehler-Pfad
- **SOLLTE [SHOULD]** ein Pattern für Long-Running-Connections geben: WebSocket bleibt offen, während Reachy mehrere Behaviors abspielt
- **KANN [MAY]** Patterns für State-Konsolidierung enthalten (mehrere HA-Entities lösen ein Reachy-Behavior aus)
- **MUSS [MUST]** das Anti-Pattern „Off-by-one-Echo bei NumberEntity-Slidern" benennen und das Fix-Schema dokumentieren: NumberEntity-Setter, der den Pose-Wert nur asynchron in eine Command-Queue schreibt, plus NumberEntity-Getter, der die Hardware-Joint-Position liest, ergibt einen sichtbaren Slider-Versatz (Slider springt nach jedem Push um einen Schritt zurück, während die Antenne / das Gelenk korrekt folgt). Fix: Setter schreibt den App-State synchron im selben Tick, Getter liest denselben App-State (nicht die Hardware-Position). Querverweis auf [`reachy-mini/ha-integration`](../../reachy-mini/ha-integration/de.md) § Number-Entity-Setpoint-Semantik für die normative Quelle

### HTTP- und WebSocket-Client-Empfehlungen
- **MUSS [MUST]** `httpx` als Default für REST-Calls empfehlen (async-fähig, moderne API); Beispiele für REST nutzen `httpx.AsyncClient`
- **MUSS [MUST]** für WebSocket eine kanonische Client-Bibliothek nennen (z. B. `websockets` oder `aiohttp.ClientSession.ws_connect`); endgültige Wahl in „Offene Fragen"
- **DARF NICHT [MUST NOT]** ohne Begründung `requests`-Beispiele zeigen — `requests` ist zwar etabliert, aber blockierend und passt schlecht zur typischen async-orientierten Behavior-Loop
- **SOLLTE [SHOULD]** zeigen, wie ein einziger HTTP-Client über die Lebenszeit der Bridge wiederverwendet wird (Connection-Pooling)

### Konfiguration und Geheimnisse
- **MUSS [MUST]** verlangen, dass Tokens und Hostnamen aus Umgebungsvariablen oder einem `.env`-File gelesen werden, nicht im Code stehen
- **MUSS [MUST]** dokumentieren, dass `.env` in `.gitignore` steht und niemals committet wird; `.env.example` ist der Vertrag
- **DARF NICHT [MUST NOT]** Beispiele zeigen, in denen ein Token im Klartext im Code, Log oder Commit erscheint

### Sicherheit
- **MUSS [MUST]** TLS-Validierung als Default erzwingen (`verify=True` bzw. äquivalent); kein `verify=False`-Snippet ohne explizite, dokumentierte Begründung
- **MUSS [MUST]** verlangen, dass Tokens nie geloggt werden — Beispiele zeigen Maskierung (`token[:4] + "…"`) wo Logging erforderlich ist
- **SOLLTE [SHOULD]** kurze Hinweise zu Token-Lebensdauer und Token-Rotation enthalten
- **KANN [MAY]** Hinweise zu HA-Webhook-Secret-Verifikation aufnehmen, falls HA das anbietet

### Versions-Pinning und Drift-Erkennung
- **MUSS [MUST]** im Skill-Body eine minimal unterstützte HA-Version benennen (z. B. `homeassistant>=<TBD>`); ohne Hardware/HA-Stand vorerst TBD-markiert
- **SOLLTE [SHOULD]** einen Drift-Check vorsehen: API-Inkompatibilitäten neuer HA-Releases werden beim nächsten Touchpoint geprüft; Anlehnung an den `reachy-mini-sdk`-Drift-Mechanismus
- **MUSS [MUST]** unverifizierte Annahmen mit `> ⚠ TBD: validate against current Home Assistant API` markieren

### Schnittstellen zu benachbarten Skills
- **SOLLTE [SHOULD]** auf `reachy-mini-sdk` verweisen, sobald die Aufgabe Bewegungs-Idiomatik dominiert
- **SOLLTE [SHOULD]** auf `app-scaffold` verweisen, sobald ein _neues_ Behavior aus einem HA-Trigger entstehen soll
- **SOLLTE [SHOULD]** auf den Agent `reachy-mini-on-device` verweisen, sobald die HA-getriggerte Bewegung live auf dem Gerät getestet werden soll
- **SOLLTE [SHOULD]** auf `audio-beat-tracking` verweisen, wenn Reachy auf Musik reagieren soll, die unabhängig von HA gestreamt wird

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/home-assistant-bridge/SKILL.md` mit gültiger Frontmatter (`name: home-assistant-bridge`, `description`, optional Tags) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description` aktiviert in einem Test-Prompt, der HA-Endpoints, Long-Lived Access Token oder die WebSocket-API im Reachy-Kontext erwähnt
- [ ] Die Wissensbasis dokumentiert: REST (States, Services, Events, Webhooks), WebSocket (Auth-Phase, subscribe_events, call_service), Reconnect-Patterns
- [ ] Beide Richtungen (HA → Reachy, Reachy → HA) sind mit lauffähigen Mini-Beispielen abgedeckt
- [ ] Beispiele nutzen `httpx` für REST und einen explizit benannten WebSocket-Client; kein `requests`-Beispiel ohne Begründung
- [ ] Tokens und Hosts werden im Beispiel ausschließlich aus Umgebungsvariablen / `.env` gelesen
- [ ] Kein Snippet enthält `verify=False`, einen Klartext-Token oder ein ungeschütztes Token im Log
- [ ] Die minimal unterstützte HA-Version ist im Skill-Body sichtbar dokumentiert (TBD bis verifiziert)
- [ ] Aussagen ohne Verifikation tragen einen `⚠ TBD: validate against current Home Assistant API`-Hinweis
- [ ] Out-of-Scope-Themen (HA-Custom-Component-Entwicklung, Reachy-SDK-Idiomatik, App-Scaffold, Audio) sind als „dafür gibt es Skill / Agent X" markiert
- [ ] Der MkDocs-Katalog rendert den Skill ohne Build-Fehler (`task docs --strict` grün)

## Offene Fragen
- Welche minimal unterstützte HA-Version pinnen wir initial (`homeassistant>=2024.x` oder neuer)? Zu klären, sobald die Ziel-HA-Instanz bekannt ist.
- Welcher WebSocket-Client ist kanonisch — `websockets`, `aiohttp` oder ein dritter? Tendenz: `aiohttp`, weil es REST und WebSocket unter einer Lib vereint, aber `websockets` ist schmaler. Endgültig im Skill setzen.
- Soll der Skill auf das offizielle `homeassistant_api`-Python-Paket verweisen, oder bei nativen httpx/websocket-Calls bleiben? Hängt vom Reife-/Wartungsstand des Pakets ab.
- Wie tief gehen wir auf HA-Authentication-Flows jenseits Long-Lived Access Tokens ein (OAuth-Flow für integrationen)? Vorschlag: zunächst nur LLAT, OAuth später.
- Soll der Skill optional einen Webhook-Server-Empfehlungs-Snippet liefern (FastAPI-Endpoint, der HA-Webhook empfängt) oder strikt Client-Patterns abdecken?
- Wie reagieren wir auf HA-API-Breaking-Changes (Removal von Endpoints zwischen Releases)? Drift-Audit-Frequenz vereinbaren.
- Welche Beispiel-Services sind im Body sinnvoll? Vorschlag: `light.turn_on`, `notify.send_message`, `automation.trigger`, `event.fire`.
- Soll der Skill TLS-Verifikation für lokale HA-Instanzen ohne offizielles Zertifikat (typisch im Heim-LAN) gesondert adressieren? Vorschlag: ja, mit Empfehlung „dediziertes CA-Bundle einschließen statt verify=False".
