# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: WebSocket API Gateway · Candidate #284

## Summary

A hybrid schema that uses PostgreSQL as the primary store but strategically uses JSONB columns for flexible, semi-structured data -- message payloads, presence metadata, routing rules, channel configuration, and connection attributes. Core relational structure (foreign keys, indexes, constraints) provides integrity for the entity graph, while JSONB columns absorb the variability and extensibility requirements of a real-time messaging protocol without requiring schema migrations for every new field.

This approach balances the consistency and query power of a relational database with the schema flexibility of a document store, avoiding the need for a separate MongoDB or similar system.

---

## Key Entities and Relationships

### Design Principles

1. **Relational columns for identity, relationships, and query predicates** -- anything you filter, join, or aggregate on gets a proper column with an index.
2. **JSONB columns for extensible payload and metadata** -- anything that varies by tenant, message type, or protocol version lives in JSONB.
3. **GIN indexes on JSONB** for queries that need to reach into document fields (e.g., filtering messages by a field in the payload).

### Schema Snippets

```sql
-- Tenants with flexible configuration
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(63) NOT NULL UNIQUE,
    plan            VARCHAR(32) NOT NULL DEFAULT 'free',
    -- Relational columns for common limits
    max_connections  INTEGER NOT NULL DEFAULT 1000,
    max_channels     INTEGER NOT NULL DEFAULT 100,
    -- JSONB for tenant-specific configuration
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  config example:
        {
          "features": ["presence", "history", "analytics"],
          "auth": { "provider": "auth0", "domain": "tenant.auth0.com" },
          "rate_limits": {
            "connections_per_minute": 100,
            "messages_per_second": 50
          },
          "webhooks": {
            "on_connect": "https://api.example.com/ws-hook",
            "on_disconnect": "https://api.example.com/ws-hook"
          },
          "ai_routing": { "enabled": true, "model": "classifier-v2" }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users with extensible profile metadata
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    -- JSONB for user-specific data that varies by tenant
    profile         JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  profile example:
        {
          "avatar_url": "https://...",
          "role": "admin",
          "preferences": { "notifications": true, "theme": "dark" },
          "tags": ["vip", "beta-tester"]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_users_profile ON users USING GIN (profile jsonb_path_ops);

-- Connections with flexible attributes
CREATE TABLE connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    server_node     VARCHAR(255) NOT NULL,
    state           VARCHAR(16) NOT NULL DEFAULT 'open',
    transport       VARCHAR(16) NOT NULL DEFAULT 'websocket',
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    disconnected_at TIMESTAMPTZ,
    -- JSONB for variable connection attributes
    attributes      JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  attributes example:
        {
          "client_ip": "203.0.113.42",
          "user_agent": "Mozilla/5.0...",
          "geo": { "country": "US", "region": "us-east-1" },
          "auth": { "method": "jwt", "claims": { "sub": "user-456" } },
          "protocol_version": "1.2",
          "capabilities": ["binary", "compression"]
        }
    */
    -- JSONB for disconnect details (varies by reason)
    disconnect_info JSONB DEFAULT NULL
);

CREATE INDEX idx_connections_tenant_state ON connections(tenant_id, state);
CREATE INDEX idx_connections_user_open ON connections(user_id) WHERE state = 'open';
CREATE INDEX idx_connections_server ON connections(server_node) WHERE state = 'open';
CREATE INDEX idx_connections_attrs ON connections USING GIN (attributes jsonb_path_ops);

-- Channels with flexible configuration and metadata
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(16) NOT NULL DEFAULT 'public',
    -- JSONB for channel-specific settings
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  config example:
        {
          "max_members": 1000,
          "history_ttl_days": 30,
          "permissions": { "publish": ["admin", "member"], "subscribe": ["*"] },
          "ai_routing": {
            "enabled": true,
            "rules": [
              { "match": { "intent": "support" }, "forward_to": "support-channel" },
              { "match": { "lang": "es" }, "forward_to": "spanish-channel" }
            ]
          },
          "webhooks": { "on_message": "https://api.example.com/hook" },
          "rate_limit": { "messages_per_second": 10 }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_channels_config ON channels USING GIN (config jsonb_path_ops);

-- Channel subscriptions (pure relational -- this is a critical join table)
CREATE TABLE channel_subscriptions (
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    connection_id   UUID NOT NULL REFERENCES connections(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (channel_id, connection_id)
);

CREATE INDEX idx_subs_connection ON channel_subscriptions(connection_id);

-- Messages with JSONB payload (the core hybrid advantage)
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID REFERENCES channels(id),
    sender_user_id  UUID REFERENCES users(id),
    -- Relational columns for common query predicates
    message_type    VARCHAR(32) NOT NULL DEFAULT 'text',
    payload_size    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for the actual message content and metadata
    payload         JSONB NOT NULL,
    /*  payload examples:
        -- Text message:
        { "text": "Hello!", "format": "plain" }

        -- Rich message:
        {
          "text": "Check this out",
          "attachments": [
            { "type": "image", "url": "https://...", "width": 800 }
          ],
          "mentions": ["user-789"]
        }

        -- System message:
        { "event": "user_joined", "user_id": "user-456" }

        -- AI-classified message:
        {
          "text": "My order hasn't arrived",
          "ai_classification": {
            "intent": "support",
            "sentiment": "negative",
            "confidence": 0.92,
            "suggested_route": "support-escalation"
          }
        }
    */
    headers         JSONB DEFAULT '{}'::jsonb,  -- custom headers, correlation IDs
    idempotency_key VARCHAR(128)
);

CREATE INDEX idx_messages_channel_time ON messages(channel_id, created_at DESC);
CREATE INDEX idx_messages_tenant_time ON messages(tenant_id, created_at DESC);
CREATE INDEX idx_messages_payload ON messages USING GIN (payload jsonb_path_ops);
-- Partial index for AI-classified messages
CREATE INDEX idx_messages_ai_intent ON messages((payload->>'ai_classification'))
    WHERE payload ? 'ai_classification';

-- Presence with flexible custom data
CREATE TABLE presence (
    user_id         UUID NOT NULL REFERENCES users(id),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID REFERENCES channels(id),
    status          VARCHAR(16) NOT NULL DEFAULT 'online',
    -- JSONB for presence extensions
    custom_data     JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  custom_data example:
        {
          "cursor_position": { "x": 150, "y": 300 },
          "typing": true,
          "viewing_page": "/dashboard",
          "device": { "type": "desktop", "os": "macOS" }
        }
    */
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, COALESCE(channel_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_presence_channel ON presence(channel_id, status);

-- Routing rules stored as JSONB documents
CREATE TABLE routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    -- The rule definition is inherently document-shaped
    rule            JSONB NOT NULL,
    /*  rule example:
        {
          "match": {
            "channel_pattern": "support-*",
            "message_fields": { "priority": "high" },
            "user_tags": ["vip"]
          },
          "actions": [
            { "type": "forward", "target": "escalation-channel" },
            { "type": "notify_webhook", "url": "https://..." },
            { "type": "enrich", "ai_model": "sentiment-v2" }
          ]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_routing_rules_tenant ON routing_rules(tenant_id, priority, enabled);
CREATE INDEX idx_routing_rules_rule ON routing_rules USING GIN (rule jsonb_path_ops);

-- Connection events / audit log with flexible details
CREATE TABLE connection_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    connection_id   UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(32) NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conn_events_conn ON connection_events(connection_id, created_at);
CREATE INDEX idx_conn_events_tenant ON connection_events(tenant_id, created_at DESC);

-- Rate limit configuration (document-shaped policies)
CREATE TABLE rate_limit_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    scope           VARCHAR(32) NOT NULL,
    -- JSONB for flexible rate limit definitions
    policy          JSONB NOT NULL,
    /*  policy example:
        {
          "window_seconds": 60,
          "max_requests": 100,
          "burst": 20,
          "penalty_action": "throttle",
          "penalty_duration_seconds": 300,
          "exemptions": { "user_tags": ["admin", "bot"] }
        }
    */
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

---

## Querying JSONB Data

The hybrid model shines when queries combine relational predicates with document field access:

```sql
-- Find VIP users currently online in a channel
SELECT u.display_name, p.status, p.custom_data
FROM presence p
JOIN users u ON u.id = p.user_id
WHERE p.channel_id = 'channel-abc'
  AND p.status = 'online'
  AND u.profile @> '{"tags": ["vip"]}';

-- Find messages with high-priority AI classification in the last hour
SELECT m.id, m.payload->>'text' AS text,
       m.payload->'ai_classification'->>'intent' AS intent,
       m.created_at
FROM messages m
WHERE m.channel_id = 'channel-support'
  AND m.created_at > now() - INTERVAL '1 hour'
  AND m.payload @> '{"ai_classification": {"intent": "support"}}'
ORDER BY m.created_at DESC;

-- Find channels with AI routing enabled
SELECT name, config->'ai_routing'->'rules' AS rules
FROM channels
WHERE tenant_id = 'tenant-123'
  AND config @> '{"ai_routing": {"enabled": true}}';

-- Aggregate connection attributes by country
SELECT attributes->'geo'->>'country' AS country,
       COUNT(*) AS active_connections
FROM connections
WHERE tenant_id = 'tenant-123'
  AND state = 'open'
GROUP BY attributes->'geo'->>'country'
ORDER BY active_connections DESC;
```

---

## Pros

- **Schema flexibility without sacrificing structure.** Core entities (tenants, users, connections, channels) have relational integrity, while extensible fields (metadata, payloads, configuration) use JSONB. No need for migrations when adding a new message field or presence attribute.
- **Single database to operate.** Unlike a polyglot persistence approach (PostgreSQL + MongoDB + Redis), everything lives in one PostgreSQL instance. Simpler ops, backups, and transactions.
- **GIN-indexed JSONB queries are fast.** PostgreSQL's JSONB indexing supports containment queries (`@>`), existence checks (`?`), and path queries with good performance. Suitable for filtering messages by payload fields or users by profile attributes.
- **Natural fit for AI-native features.** AI classification results, semantic routing metadata, and enrichment data can be added to message payloads as JSONB fields without schema changes. The routing_rules table stores complex rule definitions as documents.
- **Progressive migration path.** Start with more JSONB columns for rapid iteration during MVP, then extract heavily-queried JSONB fields into proper columns as patterns stabilize. This is a one-way migration (JSONB to column) that is safe and non-disruptive.
- **Relational power where it matters.** JOINs for channel membership, foreign keys for tenant isolation, and aggregate queries for billing all work naturally.
- **Multi-protocol support.** Different transports (WebSocket, SSE, MQTT) can store protocol-specific connection attributes in the JSONB `attributes` column without separate tables.

## Cons

- **JSONB is not free.** JSONB storage is larger than equivalent normalized columns (key names are stored per-row). GIN indexes on large JSONB columns consume significant disk and memory.
- **No schema enforcement on JSONB.** Application code must validate JSONB structure. PostgreSQL CHECK constraints with `jsonb_typeof` and JSON Schema validation (via extensions like `pg_jsonschema`) can help but are not as robust as column-level constraints.
- **Query complexity.** JSONB path queries (`->>`, `->`, `@>`, `?`) are less readable than simple column references. Developers must understand PostgreSQL's JSONB operators.
- **Partial index limitations.** While you can create partial indexes on JSONB expressions, the query planner may not always use them effectively for complex nested paths.
- **Same write throughput constraints as normalized.** The high-volume message and event tables face the same PostgreSQL write limits as Suggestion 1. JSONB does not help with write throughput.
- **TOAST overhead.** Large JSONB values (> 2KB) are stored in PostgreSQL's TOAST table, adding indirection and potential read latency for large message payloads.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ (with `pg_jsonschema` extension for validation) |
| JSONB validation | Application-level with Zod/Joi/JSON Schema; optionally `pg_jsonschema` CHECK constraints |
| Connection pooling | PgBouncer (transaction-mode) |
| Partitioning | Range-partition `messages` and `connection_events` by `created_at` |
| JSONB indexing | GIN indexes with `jsonb_path_ops` for containment queries on payload, config, attributes |
| Caching | Redis for hot presence data and active connection lookups |
| Full-text search | PostgreSQL `tsvector` on message text fields, or Elasticsearch for cross-tenant search |
| Schema migrations | Standard migration tool (Prisma, Flyway) for structural columns; JSONB fields evolve without migrations |

---

## Migration and Scaling Considerations

### JSONB Column Evolution Strategy

As the product matures and query patterns stabilize, promote frequently-queried JSONB fields to proper columns:

```sql
-- Example: promoting message intent to a column
ALTER TABLE messages ADD COLUMN ai_intent VARCHAR(64);

-- Backfill from JSONB
UPDATE messages
SET ai_intent = payload->'ai_classification'->>'intent'
WHERE payload ? 'ai_classification';

-- Create index on the new column
CREATE INDEX idx_messages_intent ON messages(ai_intent) WHERE ai_intent IS NOT NULL;

-- Application writes both the column and JSONB during transition
-- Eventually, queries shift from JSONB path to column
```

### Partitioning

Same strategy as Suggestion 1: range-partition `messages` and `connection_events` by time. JSONB columns partition along with their parent rows.

### Scaling Path

1. **Single PostgreSQL node** for MVP. JSONB flexibility accelerates development.
2. **Read replicas** for dashboard, analytics, and history queries.
3. **Redis cache** for hot-path presence and active connection queries (cache invalidation on state change events).
4. **Promote JSONB fields** to columns as query patterns mature.
5. **Partition and prune** high-volume tables monthly.
6. **Citus sharding** by tenant_id if single-node write capacity is exhausted.

### Why This May Be the Best Starting Point

This approach combines the reliability of Suggestion 1 with the flexibility to evolve quickly during the early product phase. The JSONB columns absorb uncertainty about message formats, AI enrichment schemas, routing rule shapes, and tenant-specific configuration -- all of which will change rapidly during the first year of development. Unlike a pure document database, you retain relational integrity for the entity graph.
