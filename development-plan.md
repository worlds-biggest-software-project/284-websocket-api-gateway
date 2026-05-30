# WebSocket API Gateway — Phased Development Plan

> Project: 284-websocket-api-gateway · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, a project structure, and an additive sequence of implementation phases.

---

## Product Summary

WebSocket API Gateway is open, AI-native real-time messaging infrastructure: connection management, pub/sub routing, presence tracking, rate limiting, and observability — without per-message SaaS pricing or the operational burden of raw libraries. It targets RFC 6455 as the foundation, exposes an AsyncAPI-described surface, and authenticates at the HTTP upgrade stage with JWT/OIDC per OWASP guidance.

**Primary personas:** full-stack developers (chat, live dashboards), platform engineers (standardising real-time infra), gaming backend engineers (low-latency), SaaS teams (presence, live cursors).

**Differentiators (AI-native):** semantic content routing, predictive autoscaling, real-time anomaly detection (flooding, oscillating reconnects, size spikes), natural-language configuration of routing/rate-limit/presence rules, and LLM-driven session reconstruction for debugging.

**Deployment model:** self-hosted-first (Docker / Docker Compose / Helm), horizontally scalable, with a clear path to cloud and global-edge deployment. API/CLI/SDK plus a thin monitoring dashboard.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Go 1.23+** | The hot path is tens of thousands of concurrent long-lived connections — Go's goroutine-per-connection model, low GC pressure, and mature net stack are the right fit. Outperforms Node/Python for connection density and is easier to operate than Rust for a team. |
| WebSocket library | **`nhooyr.io/websocket` (coder/websocket)** | RFC 6455-compliant, context-aware, supports `wss`, ping/pong control frames, and per-message compression (permessage-deflate). Cleaner API than gorilla/websocket and actively maintained. |
| HTTP framework | **`chi` router** + stdlib `net/http` | Lightweight, idiomatic, middleware-friendly. The upgrade handshake and management REST API share one router. |
| Real-time state store | **Redis 7+ (Cluster-capable)** | Per Data Model Suggestion 4: authoritative store for ephemeral hot-path state (connection registry, presence sorted sets, channel membership sets, sliding-window rate-limit counters) plus cross-node pub/sub fan-out. Sub-millisecond hot-path latency; the production-proven pattern (Discord, Slack). |
| Durable store | **PostgreSQL 16+** | Cold path: tenants, users, api_keys, channels, durable message history, connection-event audit log, routing rules, usage/billing aggregates. Partitioned `messages` and `connection_events` by time. |
| Cross-node fan-out | **Redis Pub/Sub (MVP) → Redis Streams (v1.1)** | Pub/Sub gives basic fan-out with zero extra infra for the MVP; Streams adds guaranteed delivery + consumer groups for the write-behind pipeline and offline queue when needed. |
| Write-behind pipeline | **In-process batched workers** | Buffer messages/events in memory, batch-insert to Postgres on interval or batch-size threshold. Keeps the hot path off disk. |
| DB access | **`pgx` (jackc/pgx v5)** + `sqlc` | `pgx` is the fastest Postgres driver for Go; `sqlc` generates type-safe Go from SQL, avoiding ORM overhead on the hot path. |
| Migrations | **`golang-migrate`** | Plain SQL up/down migrations, CI-friendly, supports Postgres partitioning DDL. |
| AI / LLM | **Provider-agnostic interface over the OpenAI-compatible API** (default `gpt-4o-mini`-class for classification, larger model for NL-config/summarisation) | Semantic routing needs cheap fast classification; NL config and session summaries need a stronger model. An interface allows self-hosted models (Ollama, vLLM) for the privacy-conscious self-host audience. |
| Config | **`koanf`** (YAML file + env override) | Twelve-factor config; YAML for self-host, env vars for containers. |
| Auth | **JWT (RFC 7519) validated at upgrade**, OIDC discovery (Okta/Auth0/Keycloak), API keys for server-side publish | Per standards.md and the OWASP WebSocket Cheat Sheet: reject unauthenticated connections at the HTTP upgrade with a real status code; never cookie-only auth. |
| API description | **AsyncAPI 3.x** (channels/messages) + **OpenAPI 3.1** (management REST) | standards.md identifies the AsyncAPI documentation gap as a differentiator; auto-publish a spec. |
| Frontend (dashboard) | **React + Vite + TypeScript**, served as static assets | Thin monitoring dashboard (connections, channels, presence, rate-limit events, anomalies). Not the core product — kept minimal. |
| CLI | **`cobra`** | Standard Go CLI framework for the `wsgw` management/admin tool. |
| Observability | **Prometheus** metrics + **OpenTelemetry** traces + structured logs (`slog`) | Connection counts, message rates, fan-out latency, rate-limit rejections, projection/pipeline lag. |
| Testing | **stdlib `testing`** + `testify` + `dockertest`/Testcontainers-go | Unit tests pure-Go; integration tests spin up real Redis + Postgres in containers. WebSocket E2E uses the same client library. |
| Quality | **`golangci-lint`**, `gofmt`/`goimports`, `go vet` | Standard Go quality gate. |
| Packaging | **Docker** multi-stage build + **Docker Compose** (gateway + Redis + Postgres) + **Helm chart** (v1.1) | Self-host-first deployment. |
| Load testing | **`k6`** with the `xk6-websockets` extension | Validate connection density, message throughput, and rate-limit behaviour. |

### Why the Redis-centric data model (Suggestion 4)

A WebSocket gateway's primary workload is ephemeral, high-frequency, and latency-sensitive — exactly Redis's strength — while durable records (history, audit, billing) belong in Postgres. Suggestion 1 (pure relational) churns the `connections` table and can't serve fan-out. Suggestion 2 (event-sourcing/CQRS) is elegant for the audit/session-reconstruction story but adds projection lag and operational complexity disproportionate to an MVP. Suggestion 4 adopts the JSONB conventions of Suggestion 3 for the Postgres cold-path columns (`config`, `profile`, `payload`, `rule`), so the relational schema stays flexible. The event-sourcing strengths of Suggestion 2 are recovered cheaply by writing an append-only `connection_events` log — enough to power AI session reconstruction without full CQRS.

### Project Structure

```
websocket-api-gateway/
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
├── docker-compose.yml
├── .golangci.yml
├── config.example.yaml
├── api/
│   ├── asyncapi.yaml              # generated/maintained AsyncAPI 3.x spec
│   └── openapi.yaml               # management REST OpenAPI 3.1 spec
├── cmd/
│   ├── gateway/main.go            # the gateway server
│   └── wsgw/main.go               # cobra CLI (admin/management)
├── internal/
│   ├── config/                    # koanf loading, validation
│   ├── transport/
│   │   ├── transport.go           # Transport interface (ws, sse later)
│   │   ├── websocket.go           # RFC 6455 handshake + frame handling
│   │   └── sse.go                 # (v1.1) Server-Sent Events
│   ├── auth/
│   │   ├── jwt.go                 # RFC 7519 validation
│   │   ├── oidc.go                # OIDC discovery + JWKS
│   │   └── apikey.go              # server-side publish keys
│   ├── connection/
│   │   ├── manager.go             # connection lifecycle, hooks
│   │   ├── registry.go            # Redis connection registry + indexes
│   │   └── state.go               # connection state machine
│   ├── channel/
│   │   ├── channel.go             # subscribe/unsubscribe, membership
│   │   └── broadcast.go           # fan-out via Redis pub/sub
│   ├── presence/
│   │   └── presence.go            # sorted-set presence, heartbeats, TTL
│   ├── ratelimit/
│   │   └── slidingwindow.go       # two-dimensional rate limiting (Lua)
│   ├── routing/
│   │   ├── rules.go               # static rule matching engine
│   │   └── semantic.go            # (Phase 8) AI semantic classifier
│   ├── persistence/
│   │   ├── store.go               # pgx pool, sqlc queries
│   │   ├── queries/               # *.sql for sqlc
│   │   └── writebehind.go         # batched async writer
│   ├── offline/
│   │   └── queue.go               # (v1.1) per-user offline message lists
│   ├── ai/
│   │   ├── client.go              # provider-agnostic LLM interface
│   │   ├── classify.go            # (Phase 8) message classification
│   │   ├── anomaly.go             # (Phase 9) anomaly detection
│   │   ├── nlconfig.go            # (Phase 10) NL → config translation
│   │   └── session.go             # (Phase 10) session reconstruction
│   ├── observability/
│   │   ├── metrics.go             # Prometheus
│   │   └── tracing.go             # OpenTelemetry
│   └── management/
│       └── rest.go                # management REST API (chi)
├── migrations/
│   └── *.up.sql / *.down.sql
├── scripts/
│   └── redis/*.lua                # atomic cleanup, rate-limit scripts
├── web/dashboard/                 # React + Vite monitoring UI (Phase 7)
├── deploy/
│   └── helm/                      # (v1.1) Helm chart
└── test/
    ├── integration/               # Testcontainers-backed
    ├── e2e/                       # full client→gateway→delivery
    └── load/                      # k6 scripts
```

The structure is grouped by concern, not by phase; later phases add files (e.g. `ai/anomaly.go`, `transport/sse.go`) without restructuring earlier ones.

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish a buildable, testable, containerised Go project with config loading, structured logging, a health endpoint, and the Redis + Postgres connections wired up. After this phase, `docker compose up` brings up the gateway, Redis, and Postgres, and `/healthz` reports dependency readiness. Everything else builds on this.

### Tasks

#### 1.1 — Repository, build tooling, and CI gate

**What**: Initialise the Go module, Makefile, linter config, and a Dockerfile + Compose stack.

**Design**:
- `go.mod` module path `github.com/worlds-biggest-software-project/websocket-api-gateway`, Go 1.23.
- `Makefile` targets: `build`, `test`, `test-integration`, `lint`, `migrate-up`, `migrate-down`, `docker`, `run`.
- `.golangci.yml` enabling `errcheck, govet, staticcheck, revive, ineffassign, gofmt, goimports`.
- Multi-stage `Dockerfile`: build stage (`golang:1.23`) → distroless/`scratch` runtime, non-root user, exposes `8080` (HTTP/WS) and `9090` (metrics).
- `docker-compose.yml`: services `gateway`, `redis:7-alpine`, `postgres:16-alpine` with healthchecks and a shared network.

**Testing**:
- `Unit: make build → binary produced, exit 0`.
- `Unit: go vet ./... → no findings`.
- `CI: golangci-lint run → passes`.
- `Integration: docker compose up → all three services reach healthy state`.

#### 1.2 — Configuration loader

**What**: Load layered config (defaults → YAML file → env overrides) into a validated struct.

**Design**:
```go
type Config struct {
    Server   ServerConfig   `koanf:"server"`
    Redis    RedisConfig    `koanf:"redis"`
    Postgres PostgresConfig `koanf:"postgres"`
    Auth     AuthConfig     `koanf:"auth"`
    Limits   LimitsConfig   `koanf:"limits"`
    AI       AIConfig       `koanf:"ai"`
    Log      LogConfig      `koanf:"log"`
}

type ServerConfig struct {
    HTTPAddr    string        `koanf:"http_addr"`    // default ":8080"
    MetricsAddr string        `koanf:"metrics_addr"` // default ":9090"
    NodeID      string        `koanf:"node_id"`      // default hostname
    ReadTimeout time.Duration `koanf:"read_timeout"` // default 60s
}

type LimitsConfig struct {
    MaxMessageBytes      int           `koanf:"max_message_bytes"`       // default 65536
    MaxConnectionsPerIP  int           `koanf:"max_connections_per_ip"`  // default 100
    HeartbeatInterval    time.Duration `koanf:"heartbeat_interval"`      // default 30s
    PresenceTTL          time.Duration `koanf:"presence_ttl"`            // default 120s
}
```
- Env override convention: `WSGW_SERVER_HTTP_ADDR`, `WSGW_REDIS_URL`, etc.
- Validation on load: required fields (`redis.url`, `postgres.dsn`), positive durations, `max_message_bytes` within `[1KB, 16MB]`. Return an aggregated error listing every invalid field.

**Testing**:
- `Unit: valid YAML → Config with documented defaults applied`.
- `Unit: env var WSGW_SERVER_HTTP_ADDR=:9999 → overrides YAML value`.
- `Unit: missing redis.url → validation error naming "redis.url"`.
- `Unit: max_message_bytes=0 → validation error`.

#### 1.3 — Dependency clients, logging, and health endpoint

**What**: Construct the pgx pool and Redis client, structured logging, and a readiness endpoint.

**Design**:
- `slog` JSON handler; log level from config; every log line carries `node_id`.
- `persistence.NewPool(ctx, cfg) (*pgxpool.Pool, error)` with pool sizing from config.
- `redisclient.New(cfg) (*redis.Client, error)` (go-redis v9), Cluster-aware via URL scheme.
- `GET /healthz` → `200` with `{"status":"ok","redis":"up","postgres":"up","node_id":"..."}`; `503` if either dependency ping fails. `GET /readyz` distinct from liveness.

**Testing**:
- `Integration (real): healthz with Redis+PG up → 200, both "up"`.
- `Integration (real): healthz with Redis down → 503, redis:"down"`.
- `Unit: log output is valid JSON containing node_id`.

#### 1.4 — Database schema & migrations (cold path)

**What**: Author the Postgres schema from Suggestion 4 as `golang-migrate` migrations and generate `sqlc` accessors.

**Design**: Tables `tenants`, `users`, `api_keys`, `channels`, `routing_rules`, `rate_limit_policies`, `messages` (RANGE-partitioned by `created_at`), `connection_events` (RANGE-partitioned by `created_at`), `tenant_usage`. JSONB columns `config`/`profile`/`payload`/`rule` per Suggestion 3 conventions. Indexes exactly as specified in Suggestion 4 (e.g. `idx_messages_channel_time`, `idx_conn_events_conn`). Include a `create_monthly_partition(table, month)` SQL helper function. `api_keys.key_hash` stores an Argon2id/SHA-256 hash, never the raw key.

**Testing**:
- `Integration (real): migrate-up then migrate-down → schema clean on both ends`.
- `Integration (real): insert tenant + user + channel → FK constraints hold; duplicate (tenant_id, name) channel → unique violation`.
- `Integration (real): insert message into current month → lands in the correct partition`.
- `Unit (sqlc): generated queries compile and round-trip a tenant row`.

---

## Phase 2: Authentication at the Upgrade Stage

### Purpose
Authentication must happen at the HTTP upgrade per OWASP guidance — it is the only point where an unauthenticated connection can be rejected with a real HTTP status code. This phase delivers JWT validation, OIDC discovery, Origin validation (CSWSH prevention), and API-key auth for server-side publishing, all before any WebSocket is accepted.

### Tasks

#### 2.1 — JWT validation (RFC 7519)

**What**: Validate a bearer JWT presented in the upgrade request.

**Design**:
```go
type Claims struct {
    Subject  string         `json:"sub"`
    TenantID string         `json:"tenant_id"`
    Scopes   []string       `json:"scopes"`
    Expiry   int64          `json:"exp"`
    Custom   map[string]any `json:"-"`
}

type Authenticator interface {
    Authenticate(r *http.Request) (*Principal, error)
}

type Principal struct {
    TenantID string
    UserID   string   // external_id mapped to users.id
    Scopes   []string
    Method   string   // "jwt" | "oidc" | "apikey"
}
```
- Token source order (configurable): `Authorization: Bearer`, then `?token=` query param (browsers can't set headers on `WebSocket`), then nowhere → reject.
- Validate `exp`, `nbf`, `iss`, `aud`, signature (HS256 with shared secret or RS256 with configured public key).
- On success, upsert `users(tenant_id, external_id=sub)` and resolve `users.id`.

**Testing**:
- `Unit: valid RS256 token → Principal with tenant_id/sub`.
- `Unit: expired token → ErrTokenExpired`.
- `Unit: bad signature → ErrInvalidSignature`.
- `Unit: token in query param when header absent → accepted`.
- `Unit: no token → ErrUnauthenticated`.

#### 2.2 — OIDC discovery & JWKS

**What**: Validate RS256 tokens from an external OIDC provider via JWKS.

**Design**:
- On startup, fetch `{issuer}/.well-known/openid-configuration`, then the `jwks_uri`. Cache JWKS keyed by `kid`, refresh on cache miss or every `jwks_ttl` (default 1h).
- Select key by token header `kid`; reject unknown `kid` after one forced refresh.

**Testing**:
- `Integration (mocked OIDC server): valid token signed by published JWKS key → authenticated`.
- `Unit: unknown kid → triggers exactly one JWKS refresh, then rejects`.
- `Unit: JWKS endpoint unreachable at startup → server logs warning, continues (lazy fetch)`.

#### 2.3 — Origin validation & API keys

**What**: Enforce Origin allowlist (CSWSH) and validate API keys for the server publish API.

**Design**:
- `auth.allowed_origins` config (list, supports `*` only when explicitly set). Reject mismatched `Origin` on upgrade with `403`.
- API key: `Authorization: Bearer wsk_<random>`; look up `api_keys.key_hash` (constant-time compare on hash), check `expires_at` and `scopes`. Used only on the management/publish REST API, never to open client connections.

**Testing**:
- `Unit: Origin not in allowlist → 403, no upgrade`.
- `Unit: wildcard origin configured → any Origin accepted`.
- `Integration (real): valid api key with publish scope → 200; expired key → 401; wrong scope → 403`.

---

## Phase 3: Connection Lifecycle & Registry

### Purpose
This is the heart of the gateway: accept WebSocket upgrades, manage the connection lifecycle and state machine, register connections (and their indexes) in Redis, run heartbeats, and clean up atomically on disconnect — including after a node crash. After this phase a client can connect, stay alive via ping/pong, and be observed in Redis.

### Tasks

#### 3.1 — Transport abstraction & WebSocket upgrade

**What**: Define a `Transport` interface and implement RFC 6455 WebSocket on top of it.

**Design**:
```go
type Transport interface {
    Send(ctx context.Context, msg []byte) error
    Receive(ctx context.Context) ([]byte, error)
    Close(code int, reason string) error
    RemoteAddr() net.Addr
}
```
- WebSocket upgrade flow: run `Authenticator` → on failure reject pre-upgrade with `401/403`; on success accept with `websocket.Accept`, enforce `MaxMessageBytes`, enable permessage-deflate if the client negotiates it.
- The interface lets Phase (v1.1) add SSE without touching `connection.Manager`.

**Testing**:
- `Integration (real): authenticated upgrade → 101 Switching Protocols`.
- `Integration (real): unauthenticated upgrade → 401, connection not accepted`.
- `Integration (real): message exceeding MaxMessageBytes → connection closed with 1009 (message too big)`.

#### 3.2 — Connection state machine & manager

**What**: Model the connection lifecycle and own per-connection goroutines.

**Design**:
- States: `connecting → authenticated → open → closing → closed`. Illegal transitions are programming errors (panic in tests, logged + force-close in prod).
- `Manager.Handle(conn)` spawns a read loop and a write loop joined by a buffered outbound channel (`chan []byte`, capacity from config; full channel → slow-consumer disconnect with close code `1011`).
- Lifecycle hooks: `OnConnect`, `OnAuthenticated`, `OnMessage`, `OnDisconnect` — implemented as an ordered slice of handlers so presence/registry/audit each subscribe.

**Testing**:
- `Unit: open → closing → closed valid; open → connecting invalid → error`.
- `Unit (mocked transport): slow consumer (outbound buffer full) → disconnect with 1011`.
- `Unit: OnDisconnect fires exactly once even on concurrent close + error`.

#### 3.3 — Redis connection registry & indexes

**What**: Register connections and maintain the four Redis indexes from Suggestion 4.

**Design**: On open, set `conn:{tenant}:{conn_id}` Hash (TTL = max lifetime, default 24h) and `SADD` to `user_conns:{tenant}:{user}`, `node_conns:{node}`, `tenant_conns:{tenant}`. Use hash tag `{tenant_id}` in keys so all of a tenant's keys co-locate on one Cluster shard, enabling the cleanup Lua script. Persist a `ConnectionOpened` `connection_event` via the write-behind pipeline (Phase 6 wires the pipeline; here just enqueue).

**Testing**:
- `Integration (real Redis): open connection → hash + 3 index memberships present`.
- `Integration (real Redis): connection hash carries TTL > 0`.

#### 3.4 — Heartbeat & atomic disconnect cleanup

**What**: Ping/pong keepalive and the atomic multi-key cleanup Lua script.

**Design**:
- Server sends WebSocket ping every `heartbeat_interval`; missing pong within `2×interval` → close `1001`.
- `scripts/redis/disconnect_cleanup.lua` (from Suggestion 4): removes the connection from every channel member set, deletes the subscription set, removes from user/node/tenant indexes, deletes the connection hash, and returns the user's remaining connection count (0 → caller marks presence offline).
- Node-crash recovery: a background reaper scans `node_conns:{node}` for nodes failing health checks and runs the cleanup script for each orphaned connection; Redis TTLs are the backstop.

**Testing**:
- `Integration (real Redis): clean disconnect → all keys/indexes removed, script returns remaining count`.
- `Integration (real Redis): user with 2 connections, close 1 → script returns 1; close 2nd → returns 0`.
- `Integration (real): no pong within timeout → server closes with 1001`.

---

## Phase 4: Channels, Subscriptions & Broadcast

### Purpose
Deliver the core messaging capability: clients subscribe to channels, publish messages, and receive fan-out across all gateway nodes via Redis Pub/Sub. This makes the gateway useful — chat, live dashboards, and multiplayer all reduce to channel pub/sub.

### Tasks

#### 4.1 — Client protocol (JSON envelope)

**What**: Define the wire protocol for client↔gateway messages.

**Design**:
```jsonc
// client → gateway
{ "op": "subscribe",   "channel": "chat-general", "id": "req-1" }
{ "op": "unsubscribe", "channel": "chat-general", "id": "req-2" }
{ "op": "publish",     "channel": "chat-general", "data": { "text": "hi" }, "id": "req-3", "idempotency_key": "k1" }
{ "op": "presence",    "channel": "chat-general", "status": "online", "custom": { "typing": true } }
{ "op": "ping" }

// gateway → client
{ "op": "ack",     "id": "req-1" }
{ "op": "error",   "id": "req-3", "code": "rate_limited", "message": "..." }
{ "op": "message", "channel": "chat-general", "data": {...}, "from": "user-456", "msg_id": "msg-..", "ts": "..." }
{ "op": "presence_event", "channel": "chat-general", "user": "user-789", "status": "online" }
```
- `op` values are validated; unknown op → `error` with code `unknown_op`. This envelope is the source for the published AsyncAPI document (Phase 7).

**Testing**:
- `Unit: parse each valid op → correct typed struct`.
- `Unit: malformed JSON → error envelope, connection stays open`.
- `Unit: unknown op → error code "unknown_op"`.

#### 4.2 — Channel subscription management

**What**: Subscribe/unsubscribe a connection to a channel with authorization.

**Design**: On `subscribe`, check channel `config.permissions.subscribe` against the principal's scopes/tags; `SADD channel_members:{tenant}:{name}` and `SADD conn_channels:{tenant}:{conn}`; auto-create transient channels if tenant config allows, else require a pre-defined `channels` row. Each gateway node maintains an in-process map `channel → set<localConn>` for local fan-out and `SUBSCRIBE`s to the Redis channel only when it gains its first local member (and `UNSUBSCRIBE`s when it loses the last) to bound Redis subscriptions.

**Testing**:
- `Integration (real Redis): subscribe → membership set + reverse index updated; node subscribes to Redis channel`.
- `Integration (real Redis): unsubscribe last local member → node unsubscribes from Redis channel`.
- `Unit: subscribe to private channel without permission → error code "forbidden"`.

#### 4.3 — Publish & cross-node broadcast

**What**: Publish a message and fan it out to all subscribers across all nodes.

**Design**: Publish flow: validate size & rate limit (Phase 5 enforces; here call the no-op limiter) → assign `msg_id` (UUIDv7 for time-ordering) → dedupe on `idempotency_key` (Redis `SET NX` with TTL) → `PUBLISH tenant:{name}` the JSON message → enqueue the durable `messages` row to the write-behind pipeline. Each node's Pub/Sub subscriber receives the message and writes it to every local subscriber's outbound channel. The originating connection does **not** echo unless it subscribed (standard pub/sub semantics).

**Testing**:
- `Integration (real Redis, 2 gateway instances): client on node A publishes → subscriber on node B receives within 50ms`.
- `Integration (real Redis): duplicate idempotency_key within TTL → published once`.
- `E2E: two clients subscribe, one publishes → other receives exactly one message with correct from/msg_id/ts`.

---

## Phase 5: Two-Dimensional Rate Limiting

### Purpose
WebSocket connections are long-lived, so rate limiting must cover both connection-level limits (max concurrent connections per user/IP/tenant) and message-level limits (messages/sec per connection). standards.md flags this as underserved and a significant implementation challenge — this phase addresses it with atomic Redis sliding windows.

### Tasks

#### 5.1 — Sliding-window message rate limiter

**What**: Per-connection / per-user message throttling using a Redis sorted set.

**Design**: Lua script `scripts/redis/sliding_window.lua` performing, atomically: `ZREMRANGEBYSCORE` (drop entries older than the window) → `ZADD` (record this request) → `ZCARD` (count) → `EXPIRE`. Returns `(allowed bool, remaining int, retry_after_ms int)`. Scopes resolved from `rate_limit_policies` (Phase 1 schema), cached in-process with a short TTL. Default policy: 50 msgs/sec/connection, burst 20.

**Testing**:
- `Integration (real Redis): 51 messages in 1s with limit 50 → 51st rejected, retry_after_ms > 0`.
- `Integration (real Redis): window slides → after window passes, requests allowed again`.
- `Unit: policy lookup falls back to tenant default when no connection-scope policy`.

#### 5.2 — Connection-level limits & enforcement

**What**: Enforce max concurrent connections per IP/user/tenant at the upgrade, and wire message limits into the publish path.

**Design**: At upgrade, after auth, `SCARD user_conns:{tenant}:{user}` and a per-IP sorted set count; if over `max_connections_per_ip` or the tenant's `max_connections`, reject with `429` (and `Retry-After`). On `publish`, call the message limiter; on rejection emit an `error` envelope with code `rate_limited` and increment a Prometheus counter. Repeated violations beyond a threshold trigger a temporary connection-level throttle (penalty action from policy JSONB).

**Testing**:
- `Integration (real): N+1th connection for a user at limit N → 429`.
- `Integration (real): publish over message limit → error envelope, connection stays open`.
- `Unit: penalty action "throttle" applied after K violations in window`.

---

## Phase 6: Presence, Persistence & Write-Behind

### Purpose
Presence is the most-requested capability after messaging. This phase adds correct presence (including across reconnects and missed heartbeats via TTL), durable message history, the audit event log, and the async write-behind pipeline that moves hot-path data to Postgres without blocking delivery.

### Tasks

#### 6.1 — Presence tracking

**What**: Track per-user and per-channel presence with heartbeat-driven TTL.

**Design**: Per Suggestion 4: `presence:{tenant}:{user}` Hash (status, last_seen, custom_data) with refreshed TTL; `channel_presence:{tenant}:{name}` Sorted Set scored by last-heartbeat Unix ts. "Who's online" = `ZRANGEBYSCORE (now-staleWindow) +inf`. On each heartbeat, `ZADD` updates the score. A background sweeper periodically `ZREMRANGEBYSCORE -inf (now-staleWindow)` and emits `presence_event` (offline) for removed users. On disconnect, if the cleanup script returns 0 remaining connections, mark the user offline and broadcast.

**Testing**:
- `Integration (real Redis): heartbeat updates score → user appears in online range`.
- `Integration (real Redis): no heartbeat past stale window → sweeper removes user, emits offline presence_event`.
- `Integration (real): user with 2 connections closes 1 → still online; closes both → offline event`.

#### 6.2 — Write-behind pipeline

**What**: Batch-persist messages and connection events to Postgres asynchronously.

**Design**: `WriteBehindPipeline` with an in-memory buffer, `flushInterval` (default 1s) and `maxBatchSize` (default 500). `Enqueue(rec)` is non-blocking; a worker flushes on timer or when the buffer fills, doing batched `COPY`/multi-row `INSERT` via pgx. On flush failure: retry with backoff, then route to a Redis Stream dead-letter (`dlq:writebehind`) so nothing is silently lost. Expose `writebehind_buffer_depth` and `writebehind_flush_errors` metrics.

**Testing**:
- `Integration (real PG): enqueue 600 messages → two batches flushed, all rows present`.
- `Integration (real PG): timer flush of a partial batch after flushInterval`.
- `Integration (real): PG insert fails → records land in DLQ stream, error metric increments`.

#### 6.3 — Message history API

**What**: REST endpoint to fetch recent channel history from Postgres.

**Design**: `GET /v1/channels/{name}/messages?limit=50&before=<ts>` (api-key authed) → reads `message_history` from the partitioned `messages` table ordered by `created_at DESC`, cursor-paginated on `(created_at, id)`. Respects channel `config.history_ttl_days`.

**Testing**:
- `Integration (real PG): publish 3 messages → history returns them newest-first`.
- `Integration (real PG): before-cursor pagination returns the prior page without gaps/dupes`.
- `Unit: limit > 100 → clamped to 100`.

---

## Phase 7: Management API, Dashboard, CLI & AsyncAPI

### Purpose
Operators need to manage tenants, channels, keys, and rate-limit policies, observe live activity, and document the API. This phase delivers the management REST API (OpenAPI 3.1), a thin React monitoring dashboard, the `wsgw` CLI, and an auto-published AsyncAPI 3.x document — closing the documentation gap flagged in standards.md.

### Tasks

#### 7.1 — Management REST API

**What**: CRUD for tenants, api_keys, channels, routing_rules, rate_limit_policies, plus live stats.

**Design**: chi router under `/v1`, api-key authed with admin scope. Endpoints: `POST/GET/DELETE /v1/tenants`, `/v1/api-keys`, `/v1/channels`, `/v1/routing-rules`, `/v1/rate-limit-policies`; `GET /v1/stats` (live counts from Redis: connections, channels, online users, by tenant). All request/response bodies validated against the OpenAPI 3.1 schema in `api/openapi.yaml`.

**Testing**:
- `Integration (real): create channel → 201; GET lists it; DELETE → 204`.
- `Integration (real): non-admin api key → 403 on all management routes`.
- `Integration (real): /v1/stats reflects an open connection`.

#### 7.2 — Monitoring dashboard

**What**: React + Vite SPA showing live connections, channels, presence, rate-limit rejections, and (later) anomalies.

**Design**: Served as static assets by the gateway under `/dashboard`. Polls `/v1/stats` and streams a read-only SSE feed of recent `connection_events`. Minimal — tables and sparklines, no write actions beyond what the REST API exposes.

**Testing**:
- `Unit (vitest): stats table renders counts from a mocked /v1/stats`.
- `E2E (Playwright): open a WS connection → dashboard connection count increments`.

#### 7.3 — `wsgw` CLI

**What**: Cobra CLI for admin operations against the management API.

**Design**: Commands: `wsgw tenant create|list`, `wsgw key create --scope publish`, `wsgw channel create|list`, `wsgw stats`, `wsgw tail --tenant X` (live event tail via SSE), `wsgw session <conn_id>` (Phase 10 reconstruction). Config via `~/.wsgw.yaml` or `--api-url`/`--api-key` flags.

**Testing**:
- `Integration (mocked API): wsgw channel create → POST issued, success printed, exit 0`.
- `Unit: missing api key → clear error, exit 1`.

#### 7.4 — AsyncAPI document generation

**What**: Publish an AsyncAPI 3.x document describing channels and message envelopes.

**Design**: Maintain `api/asyncapi.yaml` declaring the WebSocket server binding, the client/gateway message envelopes from Phase 4.1 as `messages` with JSON Schema `payload`s, and channel address patterns. Serve at `GET /asyncapi.yaml` and render via a bundled AsyncAPI HTML viewer at `/docs`. A `make asyncapi-validate` target runs the AsyncAPI CLI validator in CI.

**Testing**:
- `CI: asyncapi validate api/asyncapi.yaml → valid`.
- `Integration: GET /asyncapi.yaml → 200, content-type yaml`.

---

## Phase 8: AI-Native Semantic Routing

### Purpose
The first differentiating AI feature: route messages by semantic content classification rather than only static channels. A support message in a general channel can be auto-forwarded to a support channel; non-English messages to a localized channel. This is the headline AI-native capability from the README.

### Tasks

#### 8.1 — LLM client abstraction

**What**: Provider-agnostic interface for classification and generation.

**Design**:
```go
type LLM interface {
    Classify(ctx context.Context, text string, labels []string) (Classification, error)
    Complete(ctx context.Context, system, user string) (string, error)
}
type Classification struct {
    Label      string
    Confidence float64
    Extra      map[string]any // e.g. sentiment, lang
}
```
- Default impl targets the OpenAI-compatible `/chat/completions` with JSON-mode/structured output; base URL + model + key from config so Ollama/vLLM work for self-hosters. Built-in timeout, retry, and a circuit breaker — AI must never block the hot path.

**Testing**:
- `Integration (mocked HTTP): Classify returns parsed label + confidence`.
- `Unit: provider timeout → returns error, circuit breaker opens after N failures`.

#### 8.2 — Semantic routing engine

**What**: Apply tenant routing rules, including AI-classified matches, to published messages.

**Design**: `routing.Engine.Apply(msg)` evaluates `routing_rules` (from Suggestion 3/4 JSONB shape) in `priority` order. Rule `match` may reference static fields (`channel_pattern`, `message_fields`, `user_tags`) and/or an `ai` clause (`{ "intent": "support", "min_confidence": 0.8 }`). For AI clauses, classify the message text asynchronously; classification results are written into the message `payload.ai_classification` JSONB. Actions: `forward` (re-publish to target channel), `enrich` (attach classification), `notify_webhook`. AI routing runs off the hot path — original delivery is never delayed; forwarded delivery is best-effort.

**Testing**:
- `Unit: static rule channel_pattern "support-*" matches "support-billing"`.
- `Integration (mocked LLM): message classified intent=support with conf 0.9, rule min_confidence 0.8 → forwarded to target channel`.
- `Integration (mocked LLM): confidence 0.6 below threshold → not forwarded`.
- `Unit: LLM unavailable → original delivery unaffected, no forward, warning logged`.

---

## Phase 9: Anomaly Detection & Predictive Autoscaling Signals

### Purpose
Real-time anomaly detection (flooding, oscillating reconnects, message-size spikes) protects the gateway and surfaces incidents; predictive scaling signals let operators pre-warm capacity ahead of known load. These are the operational AI-native advantages.

### Tasks

#### 9.1 — Anomaly detection

**What**: Detect abnormal connection and message patterns in real time and raise alerts.

**Design**: A streaming detector consuming the connection-event/message metric stream maintains per-(tenant,user,connection) rolling stats. Rule-based first line: reconnect rate over threshold within window (oscillating reconnect), message rate spike vs. EWMA baseline (flooding), payload-size z-score over threshold (size spike). Each detection emits an `Anomaly{type, subject, severity, evidence, ts}` to an `anomalies` Redis Stream and a Prometheus alert. An optional LLM pass (`Complete`) turns the evidence into a human-readable explanation for the dashboard.

**Testing**:
- `Unit: 20 reconnects in 10s for one user → oscillating_reconnect anomaly`.
- `Unit: message rate 10× EWMA baseline → flooding anomaly`.
- `Unit: payload size z-score > 4 → size_spike anomaly`.
- `Integration (mocked LLM): anomaly → human-readable explanation attached`.

#### 9.2 — Predictive autoscaling signals

**What**: Ingest upstream signals and emit scale-ahead recommendations.

**Design**: `POST /v1/signals` accepts events (`{ "type": "deployment|campaign|calendar", "expected_factor": 3.0, "at": "<ts>", "ttl_seconds": 3600 }`). A forecaster combines recent connection-growth trend with active signals to emit a `ScaleRecommendation{target_nodes, reason, window}` on a metrics gauge and the dashboard. The gateway does not orchestrate infra itself — it publishes the recommendation (and a Prometheus gauge an HPA/KEDA can consume).

**Testing**:
- `Unit: campaign signal expected_factor 3 + current 100 conns → recommendation ~3× capacity`.
- `Unit: expired signal (past ttl) → ignored`.
- `Integration: recommendation exposed as Prometheus gauge`.

---

## Phase 10: Natural-Language Config & Session Reconstruction

### Purpose
The remaining AI-native promises: configure routing rules, rate limits, and presence groups in natural language, and reconstruct a user's connection lifecycle and message exchange for debugging without trawling raw logs. These lower the operational barrier and make the audit log actionable.

### Tasks

#### 10.1 — Natural-language configuration

**What**: Translate a natural-language instruction into a validated config object (routing rule or rate-limit policy).

**Design**: `POST /v1/config/nl` `{ "instruction": "limit free-tier users to 10 messages a second and forward angry support messages to the escalation channel" }`. The LLM (`Complete` with a strict system prompt + JSON schema of the target config) returns one or more proposed `routing_rules`/`rate_limit_policies`. The proposal is validated against the same schemas the REST API uses and returned for confirmation (not auto-applied); `?apply=true` with admin scope persists it. Every NL translation is logged with the instruction and resulting diff.

**Testing**:
- `Integration (mocked LLM): instruction → valid rate_limit_policy JSON passing schema validation`.
- `Unit: LLM returns invalid JSON/schema → 422 with validation detail, nothing persisted`.
- `Integration: apply=true with admin scope → policy persisted; without admin → 403`.

#### 10.2 — AI session reconstruction

**What**: Summarise a connection's or user's lifecycle and message exchange for debugging.

**Design**: `GET /v1/sessions/{connection_id}` (and `?user_id=`) reads the ordered `connection_events` and related `messages` for the subject, then calls `Complete` with a system prompt instructing a chronological narrative: connect → auth → subscriptions → notable messages → rate-limit/anomaly events → disconnect with reason. Returns both the raw timeline and the LLM narrative. Surfaced via `wsgw session <conn_id>`. The append-only `connection_events` log (Phase 3/6) is what makes this cheap — recovering the event-sourcing benefit without full CQRS.

**Testing**:
- `Integration (real PG, mocked LLM): seed a connection's events → narrative references connect, subscribe, disconnect in order`.
- `Unit: unknown connection_id → 404`.
- `Integration: raw timeline returned alongside narrative even if LLM fails (degraded mode)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton            ─── required by everything
    │
Phase 2: Auth at Upgrade                  ─── requires Phase 1
    │
Phase 3: Connection Lifecycle & Registry  ─── requires Phase 2
    │
Phase 4: Channels & Broadcast             ─── requires Phase 3   ◄── core value ships here
    │
    ├── Phase 5: Rate Limiting             ─── requires Phase 4
    └── Phase 6: Presence & Persistence    ─── requires Phase 4 (can parallel Phase 5)
            │
Phase 7: Management API / Dashboard / CLI / AsyncAPI ─── requires Phase 6 (REST/stats need persistence)
    │
    ├── Phase 8: AI Semantic Routing       ─── requires Phase 4 + Phase 7 (rules CRUD)
    ├── Phase 9: Anomaly + Scaling Signals ─── requires Phase 6 (event stream) — can parallel Phase 8
    └── Phase 10: NL Config + Session Recon─── requires Phase 7 (config CRUD) + Phase 8 (LLM client) — can parallel Phase 9
```

**Parallelism opportunities:**
- Phases 5 and 6 can be developed concurrently once Phase 4 lands.
- Phases 8, 9, and 10 are largely independent after their prerequisites; the LLM client (8.1) should be built first as 9 and 10 reuse it.
- The dashboard (7.2) and CLI (7.3) can be built in parallel with each other once 7.1 exists.

**MVP boundary:** Phases 1–7 deliver the full feature-survey MVP (connection management, broadcast, presence, rate limiting, logging/audit, auth, dashboard, CLI). Phases 8–10 deliver the AI-native differentiators. v1.1 items (SSE transport, offline queue, DM channels, Helm/edge deployment, cost analytics) slot in as additive modules already accounted for in the project structure.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit tests and integration tests (Testcontainers-backed Redis + Postgres) pass.
3. `golangci-lint run`, `gofmt`/`goimports`, and `go vet ./...` pass with no findings.
4. `make build` and the multi-stage `docker build` succeed; `docker compose up` brings the stack to healthy.
5. The phase's capability works end-to-end against the running stack (demonstrated by at least one E2E test).
6. New config options are added to `config.example.yaml` and documented.
7. New management endpoints appear in `api/openapi.yaml`; new client message ops appear in `api/asyncapi.yaml`, and both validate in CI.
8. New database tables/columns have forward and reverse `golang-migrate` migrations, and `sqlc generate` is up to date.
9. New Redis Lua scripts are committed under `scripts/redis/` with a corresponding integration test.
10. New Prometheus metrics are registered and surfaced on `:9090/metrics`.
11. Security checks for the phase pass: auth enforced at upgrade, Origin validated, message-size limits enforced, no secrets logged (OWASP WebSocket Cheat Sheet items addressed).
