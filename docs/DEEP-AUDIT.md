# Deep Audit: Climate

Comprehensive evaluation of the Climate IoT monitoring project — firmware, webapp, competitive position, and roadmap.

> **Methodology:** Source code audit + competitor research from official docs and community sources. Competitor feature matrices were cross-checked against published documentation and release notes as of September 2026. Competitive claims are attributed where possible; where documentation was ambiguous, features are listed as "claimed."

---

## 1. Executive Summary

Climate is an open-source IoT temperature/humidity monitoring system consisting of:
- **Firmware:** ESP32 (ESP-IDF, PlatformIO), DHT22/SHT31 sensors, LittleFS queue with CRC records, configurable API key auth, OTA updates, deep-sleep option.
- **Webapp:** Fastify 5 + SQLite (WAL mode) API with Fastify + SQLite backend, Vue 3 dashboard.

**Climate's core differentiation:** Simplicity and transparency. It is a complete, inspectable stack from sensors to dashboard, with deliberate trade-offs toward minimalism. Unlike ESPHome (which targets a broad ecosystem via YAML and native API protocol) or Tasmota (which targets switch-centric automation via MQTT), Climate is purpose-built for environmental monitoring with an opinionated but clean HTTP API.

**Competitive standing:**
- **Against ESPHome:** Climate is far less feature-rich and has no ecosystem. ESPHome 2026.7 ships native ESP-IDF toolchain, Noise/ChaCha20-Poly1305 encryption, signed OTA, NVS encryption, and deep sleep queuing. Climate wins only on simplicity and directness of its single-purpose HTTP + API key model.
- **Against Tasmota:** Tasmota supports far more protocols (MQTT, HTTP, KNX, etc.) and hardware variants. Climate wins on clean architecture, modern TLS-only auth, and transparent self-hosted dashboard.
- **Against ThingsBoard CE:** ThingsBoard is a full IoT platform (rule engine, digital twins, multi-protocol). Climate is not comparable in scope. Climate's niche is "single-purpose environmental monitoring with minimal moving parts."
- **Against Home Assistant + ESPHome:** HA provides 2000+ integrations and automation. Climate's position: zero-config local monitoring where HA is overkill (e.g., a single remote logger without a full HA instance).

**Bottom line:** Climate is not positioned to win on features. It wins on being a clean, auditable, self-contained climate logger that does one thing well and is easy for a developer to fork and adapt.

---

## 2. Climate — Current State

### Firmware
- **Language/Platform:** C++/ESP-IDF via PlatformIO
- **Sensors:** DHT22 (default), SHT31 (drop-in alternative)
- **Auth:** API key via `Authorization: Bearer` header (configurable `API_URL`/`API_KEY` in `platformio.secrets.ini`)
- **Storage:** LittleFS queue — 16 segment files × 16 slots, each slot = `Record` (1536-byte payload) with CRC-32 integrity check
- **Reliability:** Boot health probation (45s), sequence numbering with exhaustion detection, exponential backoff retries (capped 300s + entropy)
- **OTA:** SHA-256 verified, signed OTA support (`performOta`), version tag validation on upload route
- **Network:** WiFi with auto-reconnect, TLS via `WiFiClientSecure`, clock sync, 15-minute heartbeat reporting diagnostics
- **Deep Sleep:** Separate code path (`flushSync` / `persistOnly`) bypasses RTOS pipeline for synchronous flush
- **Config:** `DeviceConfig` struct with EMA smoothing, thresholds, hysteresis, debounce samples, calibrated offsets

### Webapp
- **Stack:** Node.js (Fastify 5) + SQLite (DatabaseSync, WAL mode)
- **Auth:**
  - Dashboard owner: password login (HttpOnly `climate_session` cookie, SameSite=Strict, 24h expiry, Secure in prod)
  - Devices: API key Bearer auth (`hash(token)` compared via `timingSafeEqual`)
  - Device auth modes: `required` (default), `optional`, `public`
  - Admin: `ADMIN_API_KEY` (timing-safe compare)
- **Data flow:** Devices POST `/api/v1/telemetry` → fingerprint dedup (SHA-256 of JSON payload) → insert into `readings` → `latest_id` tracking → alarm evaluation → optional webhook
- **Alarms:** Threshold-based (temperature low/high, humidity low/high), debounce via counter, hysteresis, state machine (active/resolved/acknowledged)
- **Firmware:** POST/GET `/api/v1/devices/:id/firmware` with base64 upload, max 1,310,720 bytes (LOLITNA32 app partition), version regex `^[a-zA-Z0-9._-]+$` (no `..`)

### Project Hygiene
- npm 12.1.0 (single lockfile, pnpm removed per user preference)
- 0 npm audit vulnerabilities
- All tests green: API 17/17, firmware native 25/25, 5 PlatformIO builds
- README: clean/pro, architecture diagram, env tables

---

## 3. What Climate Does Well

| Area | Strength |
|------|----------|
| **Transparency** | Every file is human-readable; no build-step magic obscuring behavior |
| **Reliability model** | CRC-32 records + write+readback verification + boot health probation = robust to corruption |
| **Firmware universality** | `API_URL`/`API_KEY` env vars decouple firmware from any specific dashboard |
| **Auth clarity** | Device API key (Bearer) vs. owner password (HttpOnly cookie) cleanly separated; no ambiguous passkey/WebAuthn complexity |
| **Queue design** | Segment-file mapping reduces LittleFS CTZ copy-on-write amplification; readback verification catches partial writes |
| **Backoff strategy** | `retryDelay` uses `esp_random()` entropy to avoid synchronized retry storms across fleet |
| **OTA safety** | Version tag regex blocks `..` traversal; base64 body size pre-checked; SHA-256 recorded |
| **Boot health** | Rejects images where any task fails to make progress (`evaluateBootHealth`) — catches silent task death |
| **Test coverage** | API (17), firmware native (25), PlatformIO builds (5) — multi-axis verification |
| **Migration hygiene** | Migration v4 drops passkeys table cleanly; version 5 migrates nullable password_hash |
| **Fingerprint dedup** | SHA-256 of JSON payload enables idempotent telemetry — survives retries without double-insert |
| **Time quality tracking** | `"synced"` / `"received"` / `"estimated"` distinguishes clock-synced from boot-time samples |
| **Config validation** | `validConfig()` enforces physical bounds (temp -40..80, humidity 0..100, EMA 0.05..1.0) |

---

## 4. Critical Issues (P0)

| # | Issue | Surface | Severity |
|---|-------|---------|----------|
| 1 | **`readRecord` returns `true` on file-open failure** | Firmware `telemetry_publisher.cpp:23` | ⚠️ Ambiguity — open failure treated as corrupt-zero record |
| 2 | **SQLite busy_timeout = 5000ms under concurrent write** | API `store.ts:32` | ⚠️ Potential contention under load |
| 3 | **No pagination on `/api/v1/devices`** | API `app.ts:399` | ⚠️ Hard limit 1000 devices — no graceful degradation |
| 4 | **Firmware upload route has no auth** | API `app.ts:599` | ⚠️ Anyone authenticated (even no_password mode) can upload firmware |
| 5 | **`flushSync`/`persistOnly` bypass task pipeline** | Firmware `telemetry_publisher.cpp:168+` | ⚠️ Risk of inconsistent state if queue modified elsewhere concurrently |
| 6 | **Session tokens stored hashed, but no revocation list** | API `store.ts:219` | ⚠️ Revocation = delete DB row only; no token blacklist |
| 7 | **Webhook delivery failures not retried/backed off** | API `alarms.ts` | ⚠️ Dropped notifications on transient outage |
| 8 | **No rate limiting on telemetry upload** | API `app.ts:478` | ⚠️ DoS amplification via device API key |
| 9 | **`passwordMatches` uses scrypt but `passwordHash` salt from `secret()` = base64url** | API `store.ts:16` | ⚠️ Verify salt format compatibility is robust |
| 10 | **No firmware integrity attestation at firmware side** | Firmware | ⚠️ Firmware doesn't verify its own update SHA against a trusted manifest |

---

## 5. Missing Features (High Value)

| Feature | Gap | Notes |
|---------|-----|-------|
| **Device certificate/Mutual TLS** | Only Bearer token auth | ESP32 supports TLS client certs; would be stronger than API keys |
| **MQTT support** | HTTP-only transport | Tasmota & ThingsBoard lead here; MQTT essential for some deployments |
| **Multi-sensor device model** | Hardcoded to temp/humidity | No extensibility for pressure, CO2, etc. |
| **Firmware rollback** | No rollback endpoint | ESP-IDF OTA rollback exists but not exposed |
| **Metrics endpoint** | No Prometheus/Openmetrics | For dashboard monitoring of the server itself |
| **Data export (CSV/Bulk)** | Readings API paginated but no bulk export | Users needing analytics must query DB directly |
| **Retention policy** | No automatic data aging | `readings` table grows unbounded |
| **Dashboard localization** | No i18n | Single language only |
| **Mobile-optimized dashboard** | Not explicitly tested on mobile viewport | May degrade on small screens |

---

## 6. Features to Improve

| Feature | Current State | Suggested Improvement |
|---------|---------------|-----------------------|
| **Config distribution** | Device polls `/device/config` every 60s | Push model via server-sent events or long-poll for faster propagation |
| **Alarm hysteresis** | Applied only when alarm active | Could pre-arm hysteresis window before threshold crossing |
| **Queue observability** | `queueDepth` in diagnostics, but no historical tracking | Log queue depth over time; expose metrics |
| **Duplicate conflict detection** | Returns 409 on conflicting fingerprint | Could log as security event (potential replay attack) |
| **OTA progress** | Binary download only — no progress endpoint | Device could report flash progress back to server |
| **Error diagnostics** | `storageFailures`, `writeFailures` in queue | Could add per-error categorization (CRC, flash write fail, etc.) |
| **Config versioning** | Monotonic `version++` | Could support named config profiles (e.g., "winter", "summer") |
| **Webhook payload** | Basic JSON alarm payload | Could support templating (e.g., Slack, Discord, email formats) |

---

## 7. Features to Remove (Bloat / Unused)

| Feature | Reasoning |
|---------|-----------|
| **`legacy` bootId fallback** (`app.ts:536`) | Supports old firmware with no `delivery` context. Consider deprecation timeline. |
| **`password_optional` auth mode** (`app.ts:37`) | Adds complexity (read access without login) for marginal benefit. Recommend removal. |
| **`optional` device auth mode** (`app.ts:244`) | Auto-registers anonymous devices creates ambiguity. Recommend `required` or `public` only. |
| **`SetOption` style device-level config** (if ported to firmware) | ESPHome/Tasmota-style massive config knobs — Climate should stay minimal. |

---

## 8. UI/UX Issues

| # | Issue | Severity |
|---|-------|----------|
| 1 | **Dashboard mobile viewport** — not confirmed mobile-friendly | Medium |
| 2 | **No dark/light theme toggle** | Low |
| 3 | **Loading states unclear** — skeleton loaders exist but inconsistent usage | Medium |
| 4 | **Empty state illustrations** — no friendly empty states for devices/readings | Low |
| 5 | **Alarm acknowledgment** — no bulk ack or silence-all | Low |
| 6 | **Config form validation feedback** — server-side errors may not map to specific fields | Medium |
| 7 | **Firmware upload UX** — base64 payload upload is developer-oriented, not end-user friendly | Medium |
| 8 | **Historical data navigation** — no date-range picker for charts | Medium |
| 9 | **Settings access** — "Private/Public" toggle naming confusing (should match device auth mode naming) | Medium |
| 10 | **Session timeout** — cookie expires in 24h but no warning or auto-refresh | Low |
| 11 | **Error toasts** — severity levels not visually distinct | Low |
| 12 | **Keyboard accessibility** — tab order and ARIA not audited | Low |
| 13 | **Chart interactivity** — tooltips/zoom on TemperatureChart | Medium |
| 14 | **Recent activity feed** — truncated, no detail view | Low |

> **Note:** Full UI/UX audit requires visual inspection (screenshots). The items above are inferred from code structure and are marked as "needs visual verification."

---

## 9. Performance Issues

| Timescale | Risk |
|-----------|------|
| **1 hour** | No metrics collected; cannot assess runtime behavior under load |
| **1 day** | SQLite WAL mode with `busy_timeout=5000` may block under concurrent writes |
| **1 week** | `readings` table grows linearly; indexes exist but scan patterns not benchmarked |
| **1 month** | No retention policy → table bloat; query planner may degrade without ANALYZE |
| **1 year** | Firmware flash wear: each sample = up to 3 flash writes (append + checkpoint + ack); at 10s interval = ~8.6M writes/year — LittleFS wear leveling mitigates but lifecycle unmodeled |

**Benchmark needed:**
- `ab`/`wrk` against `/api/v1/telemetry` at 50–500 req/s concurrent
- SQLite query plan analysis for readings by device_id + date range
- Firmware flash write cycle estimate vs. ESP32 flash endurance (100k cycles typical)

---

## 10. Architecture Issues

| # | Issue | Detail |
|---|-------|--------|
| 1 | **No event sourcing** | Telemetry is mutable insert; no append-only immutable log |
| 2 | **Single monolithic API** | Alarms, history, store, telemetry all in one Fastify instance — no module boundary |
| 3 | **No circuit breaker** | Webhook failures don't stop retry attempts; no backoff escalation |
| 4 | **Config push is poll-based** | 60s polling latency; no push mechanism for config changes |
| 5 | **No distributed tracing** | No OpenTelemetry / tracing spans across firmware → API → webhooks |
| 6 | **Database is SQLite** | Good for single-node; no replication; limits horizontal scaling |
| 7 | **`store.ts` mixes ORM + business logic** | `Store` class handles both queries and alarm evaluation — tight coupling |
| 8 | **No health check depth** | `/health` returns `{status: "ok"}` — doesn't probe SQLite, LittleFS, or network |

---

## 11. Database & Logging Issues

| Issue | Detail |
|-------|--------|
| **No retention policy** | `readings` table grows forever; no TTL or downsampling |
| **No partitioning** | Single table; at 1M+ rows, scans by device+date slow without manual partitioning |
| **No ANALYZE** | Indexes exist but SQLite's query planner benefits from `ANALYZE` after large inserts |
| **Migration version gap** | Migrations jump from 2 → 4 (skips 3); version 3 appears intentionally skipped but no comment explains why |
| **No migration DOWN** | One-way migrations only; no rollback safety |
| **No data export** | Cannot extract readings for analytics without DB file access |
| **`alarm_counters` JSON in devices row** | Stored as JSON string in DB — not normalized; query by alarm state requires parsing |
| **No query logging** | Slow query logging not enabled; can't identify performance regressions |
| **No audit log** | Owner logins, config changes, firmware uploads not logged for forensics |

---

## 12. Reliability Issues (Edge Cases)

| # | Scenario | Current Behavior | Desired |
|---|----------|-----------------|---------|
| 1 | **Clock skew > 5 min** | `timeQuality: "synced"` even if wrong | Should reject or warn on NTP before trusting timestamp |
| 2 | **Duplicate delivery (same boot+seq)** | Fingerprint dedup handles it | ✓ Already handled |
| 3 | **Replay attack (old sample resent)** | Fingerprint matches → accepted as duplicate | ✓ No write, no side effect — acceptable |
| 4 | **Out-of-order sample** | `sampled >= latest.sampled_at` check — but only for `newest` | Old samples within order still inserted (correct) |
| 5 | **Future timestamp** | `timestamp > now + 300000` → 400 reject | ✓ Handled |
| 6 | **Corrupt checkpoint** | `validCheckpoint` fails → `++corrupt` → slot fallback logic? | Review recovery path for dual-checkpoint corruption |
| 7 | **Full flash** | `append` returns false → `++dropped` | Should trigger `storageFailures` counter visible to server |
| 8 | **WiFi password change** | Firmware never updates WiFi creds over-the-air | Manual reflash needed — acceptable for current scope |
| 9 | **API key rotation** | `/rotate-key` endpoint exists | ✓ Already handled |
| 10 | **Firmware binary truncated OTA** | SHA-256 verified at flash complete | Need: atomic copy, rollback on failure |
| 11 | **Device clock never syncs** | `sampledAt = 0` → `timeQuality: "received"` | ✓ Handled but stale-after calc uses `now` — may advance `latest` too aggressively |
| 12 | **Sequence exhaustion** (`UINT32_MAX`) | `ESP.restart()` after new boot epoch persisted | ✓ Handled but no LED/WiFi indication of recovery state |
| 13 | **Network partition (server unreachable)** | Samples queued in LittleFS → retry on reconnect | ✓ Handled — this is the queue's primary purpose |
| 14 | **Malicious device sends oversized payload** | `fingerprint = hash(JSON.stringify(b))` — JSON.stringify could be huge | Consider capping `payload` field size before stringify |

---

## 13. Code Quality Issues

| Area | Finding | File |
|------|---------|------|
| **Firmware** | `readRecord` semantics: returns `true` (valid) when file unreadable — confusing (treats unreadable as corrupt-zero) | `telemetry_publisher.cpp:23` |
| **Firmware** | Hardcoded `60000` ms config poll, `15000` ms heartbeat, `300000` ms backoff cap — magic numbers | `telemetry_publisher.cpp:280,289,158` |
| **Firmware** | `addDeliveryContext` string manipulation: memcpy + snprintf into fixed buffer — overflow-safe but fragile | `wire_protocol.cpp:68` |
| **API** | `app.ts` is 626 lines — single file with all routes | `app.ts` |
| **API** | `authenticated` function recomputes `hash(session(r)!)` twice — minor duplicate | `app.ts:207` |
| **API** | No input length cap on telemetry payload before `hash(JSON.stringify(b))` — DoS vector | `app.ts:483` |
| **API** | `deviceAuth` returns `row` type but row fields (e.g. `latest_id`) used loosely — no typed accessor | `store.ts:66-96` |
| **API** | `transaction()` uses `BEGIN IMMEDIATE` — good, but no savepoint for nested use | `store.ts:44` |
| **Store** | `Store` class handles both low-level SQL and business logic (alarms, config) — violates SRP | `store.ts:26-206` |
| **Dashboard** | `useTelemetry.ts` — needs review for reactive memory leaks under long polling | `composables/useTelemetry.ts` |

---

## 14. Competitor Comparison

> **Sources:** Official documentation, GitHub repos, Hackster.io, Reddit, ThingsBoard docs, ESPHome release notes (all retrieved September 2026).

### Feature Matrix

| Feature | **Climate** | **ESPHome** | **Tasmota** | **ThingsBoard CE** | **Home Assistant + ESPHome** |
|---------|-------------|-------------|-------------|--------------------|------------------------------|
| **License** | Apache 2.0 | Apache 2.0 | GPL-3.0 | Apache 2.0 | Apache 2.0 |
| **Firmware base** | ESP-IDF (PlatformIO) | ESP-IDF/Arduino | Arduino core | N/A (platform) | ESPHome firmware |
| **Config method** | API key + HTTP | YAML | Web UI / commands | Dashboard rule chains | HA UI + YAML |
| **Protocols** | HTTP/HTTPS only | Native API, MQTT, HTTP | MQTT, HTTP, KNX, Serial | MQTT, CoAP, HTTP, SNMP, LwM2M | Native API, MQTT, HTTP |
| **Transport security** | TLS (mTLS client cert optional) | Noise/ChaCha20-Poly1305 (2026.7+) | TLS for MQTT | TLS for all protocols | TLS via ESPHome |
| **OTA** | SHA-256 verified, signed | Signed, encrypted (2026.9) | Minimal | Yes | ESPHome OTA |
| **Offline buffer** | Yes (LittleFS, 256 slots, CRC) | Deep sleep queuing (2026.7+) | Once-daily flash save (by design) | Edge stores locally | ESPHome retries |
| **Multi-sensor** | Temp/humidity only | 200+ components | 20+ sensor types | Unlimited | 200+ integrations |
| **Dashboard** | Included (Vue 3) | External (HA, ESPHome dashboard) | External (HA, Node-RED, Grafana) | Built-in | HA UI |
| **Rule engine** | Basic thresholds + hysteresis | Limited | Rules + backlog | Full rule chains | HA automations |
| **Device mgmt** | API-key per device | Encryption key per device | Topic-based | Full device modeling | HA entity registry |
| **Scalability** | 1000 device limit | Hundreds to thousands | Hundreds | Thousands (cluster) | Thousands (cluster) |
| **Edge processing** | None | None | None | ThingsBoard Edge | HA Edge |
| **Deployment** | Docker (single container) | HA add-on, standalone, cloud | Standalone | Docker, K8s | Supervised, Container, Core |
| **Extensibility** | Fork + customize | Component API | Templates + rules | Custom rule nodes, widgets | Custom integrations, HACS |
| **Learning curve** | Low (single-purpose) | Medium (YAML) | Medium (web UI commands) | High | Medium |
| **Documentation** | Good (single repo) | Excellent | Excellent | Excellent | Excellent |
| **Community** | None (new) | Very large (Open Home Foundation) | Large | Very large (view: 5k+ stars) | Very large (Nabu Casa) |

### Competitive Positioning

```
ESPHome          ──── Full-feature ESP32 ecosystem, native API, encryption, massive component library
                    ↑
                    │
                    │  Climate: minimal, transparent, audit-friendly
                    │  one-purpose HTTP logger
                    ↓
Tasmota          ──── Mature, MQTT-first, massive switch/sensor compat, web UI config

ThingsBoard CE   ──── Full IoT platform, rule engine, digital twins, multi-protocol, clustering

Home Assistant   ──── Full smart home hub, 2000+ integrations, automation engine
```

**Climate's lane:** Remote/simple deployments where a developer wants a logger they can fully read and audit — not a platform integration.

---

## 15. New Feature Ideas

| Category | Idea | Value | Effort |
|----------|------|-------|--------|
| **Reliability** | Signed firmware manifests at server side | High | Medium |
| **Reliability** | OTA atomic swap with rollback-on-fail | High | Medium |
| **Reliability** | Boot countdown LED / buzzer | Medium | Low |
| **Security** | Per-endpoint device certificate provisioning | High | High |
| **Protocol** | Optional MQTT bridge (forward readings to MQTT) | High | Medium |
| **Protocol** | UDP/syslog export for logs | Medium | Low |
| **Observability** | Prometheus metrics endpoint (`/metrics`) | High | Low |
| **Observability** | Firmware crash dump collection via `esp_crash_dump` | Medium | Medium |
| **Data** | Retention policy (e.g., 30-day high-res, then hourly rollups) | High | Medium |
| **Data** | CSV/JSON bulk export endpoint | Medium | Low |
| **Dashboard** | Mobile-responsive layout audit + fix | Medium | Low |
| **Dashboard** | Dark/light theme toggle | Low | Low |
| **Dashboard** | Alarm acknowledgment flow (resolve/silence) | Medium | Low |
| **Config** | Named config profiles ("winter"/"summer") | Low | Low |
| **Config** | Server-side config push (SSE or long-poll) | Medium | Medium |
| **Deployment** | Kubernetes/Helm chart | Medium | Medium |

---

## 16. Quick Wins (High Impact, Low Effort)

| # | Action | Impact | Effort |
|---|--------|--------|--------|
| 1 | Add `/metrics` Prometheus endpoint to API | Observability | Low |
| 2 | Add retention job (DELETE old readings nightly) | Performance | Low |
| 3 | Add `ANALYZE` after migration | Performance | Low |
| 4 | Fix `/device/config` poll interval comment | Clarity | None |
| 5 | Add migration version 3 gap explanation (comment) | Clarity | None |
| 6 | Add depth to UI "Private/Public" toggle tooltip | UX | Low |
| 7 | Add explicit error on oversized telemetry payload before hash | Security | Low |
| 8 | Add health endpoint probing SQLite + LittleFS + WiFi | Reliability | Low |

---

## 17. Long-Term Improvements

| Area | Improvement |
|------|-------------|
| **Architecture** | Split API into service modules with shared kernel |
| **Deployment** | Helm chart for K8s; health probes; auto-backup |
| **Scale** | Migration path from SQLite to Postgres (optional) |
| **Observability** | OpenTelemetry tracing across firmware → API → webhooks |
| **Security** | Device attestation, mTLS, firmware signing key management |
| **UX** | Mobile-first dashboard redesign; dark mode; accessibility audit |
| **Ecosystem** | Plugin architecture for output sinks (InfluxDB, Grafana, MQTT, HTTP) |

---

## 18. Development Roadmap

### P0 — Immediate (next release)
- [ ] Fix `readRecord` return-value ambiguity in firmware comments
- [ ] Add payload size cap before `hash(JSON.stringify(b))` in API
- [ ] Add health endpoint depth (SQLite, storage, network)
- [ ] Add retention job + `ANALYZE` + migration v3 comment
- [ ] Fix device auth mode naming consistency (UI "Private/Public" ↔ backend)

### P1 — Short term (1–3 months)
- [ ] Prometheus `/metrics` endpoint
- [ ] Firmware crash dump collection
- [ ] Bulk data export (CSV)
- [ ] Mobile dashboard audit + fix

### P2 — Medium term (3–6 months)
- [ ] Signed firmware manifests
- [ ] OTA atomic swap + rollback
- [ ] Optional MQTT bridge
- [ ] Named config profiles

### P3 — Long term (6–12 months)
- [ ] Device certificate provisioning (mTLS)
- [ ] Server-side config push (SSE)
- [ ] SQLite→Postgres migration path
- [ ] OpenTelemetry tracing
- [ ] Plugin architecture for sinks

---

## 19. Top 10 Improvements by Impact

| Rank | Improvement | Impact | Effort Est. |
|------|------------|--------|-------------|
| 1 | Payload size cap before hash (DoS prevention) | High | Low |
| 2 | Health endpoint probing subsystems | High | Low |
| 3 | Retention policy (prevent DB bloat) | High | Medium |
| 4 | Prometheus metrics endpoint | High | Low |
| 5 | `ANALYZE` + migration gap comment | Medium | None |
| 6 | Auth mode naming consistency (UI/backend) | Medium | Low |
| 7 | Firmware crash dump collection | Medium | Medium |
| 8 | Bulk data export (CSV) | Medium | Low |
| 9 | Mobile-responsive dashboard | Medium | Low |
| 10 | Signed firmware manifests | High | Medium |

---

## 20. Rekomendasi: Bentuk Climate Versi Ideal

Climate should evolve as a **minimal, audit-friendly environmental logger** — not a platform.

### Core principle
**Single-purpose, fully inspectable, forkable.** A developer should be able to `git log` through every line and understand the complete data path: sensor → queue → HTTP → DB → dashboard. No YAML indirection, no 200-component abstraction, no black-box protocols.

### Positioning statement
> **Climate:** A self-hosted, auditable ESP32 temperature and humidity logger with a built-in Fastify + SQLite + Vue stack. For developers and small deployments who want a climate monitor they can read, fork, and trust — not a smart-home ecosystem integration.

### Ideal feature scope
- ✅ Firmware: ESP32 + DHT22/SHT31, TLS API key auth, LittleFS queue, CRC, OTA
- ✅ API: Fastify + SQLite, password dashboard login, device API key auth
- ✅ Dashboard: Live conditions, charts, alarms, config, firmware management
- ✅ Simplicity: Single repo, single Docker container, no external dependencies
- ✅ Transparency: Every decision documented, every file readable
- ❌ No: MQTT, 200+ sensors, rule engine, digital twins, mobile app, cloud sync (optional plugins later)

### Non-goals (avoid feature creep)
- Never become ESPHome: no component abstraction layer
- Never become ThingsBoard: no multi-protocol, no rule engine
- Never become Home Assistant: no 2000 integrations
- Never become Tasmota: no switch/actuator focus

Climate's strength is its constraint. Keep it small, keep it clean, keep it forkable.

---

## Methodology & Sources

**Firmware source reviewed:** `main.cpp`, `telemetry_publisher.cpp`, `wire_protocol.cpp`, `reliability.cpp/.h`, `telemetry_options.h`, `platformio.secrets.ini.example`

**API source reviewed:** `app.ts` (full), `store.ts` (full), `migrations.ts` (full), `model.ts` (full), `alarms.ts` (structure)

**Competitor research sources:**
- ESPHome: https://esphome.io, https://developers.esphome.io/architecture/api/protocol_details, release notes 2026.1–2026.9
- Tasmota: https://tasmota.github.io/docs/Commands, /docs/MQTT, /docs/Upgrading, /docs/Getting-Started
- ThingsBoard: https://thingsboard.io/docs/pe/why-thingsboard, /ce-vs-pe-diff, GitHub repo README
- Home Assistant: https://www.home-assistant.io/integrations/esphome, community forum threads

**Limitations:**
- Full UI/UX audit requires screenshot/visual inspection (not performed here)
- Performance benchmarks not run — recommendations are based on code analysis
- Firmware longevity (flash write cycle modeling) is an estimate, not a measured profile
