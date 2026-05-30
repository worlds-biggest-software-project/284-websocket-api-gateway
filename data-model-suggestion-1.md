# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: WebSocket API Gateway · Candidate #284

## Summary

A fully normalized relational schema using PostgreSQL as the primary data store. All entities -- connections, channels, messages, presence records, rate-limit configurations, and audit logs -- are modeled as discrete tables with foreign-key relationships and referential integrity. This is the most traditional approach and provides strong consistency guarantees, mature tooling, and well-understood query patterns.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
tenants 1──* api_keys
tenants 1──* channels
tenants 1──* users
users   1──* connections
users   1──* presence_records
channels 1──* channel_subscriptions *──1 connections
channels 1──* messages
connections 1──* messages (as sender)
connections 1──* connection_events
tenants 1──* rate_limit_policies
rate_limit_policies 1──* rate_limit_counters
```

### Schema Snippets

```sql
-- Multi-tenant isolation
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(63) NOT NULL UNIQUE,
    plan            VARCHAR(32) NOT NULL DEFAULT 'free',
    max_connections  INTEGER NOT NULL DEFAULT 1000,
    max_channels     INTEGER NOT NULL DEFAULT 100,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    key_hash    VARCHAR(128) NOT NULL UNIQUE,
    label       VARCHAR(255),
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users authenticated through the gateway
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

-- Active and historical WebSocket connections
CREATE TABLE connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    connection_token VARCHAR(128) NOT NULL UNIQUE,
    server_node     VARCHAR(255) NOT NULL,
    client_ip       INET,
    user_agent      TEXT,
    transport       VARCHAR(16) NOT NULL DEFAULT 'websocket',  -- websocket | sse | mqtt
    state           VARCHAR(16) NOT NULL DEFAULT 'open',       -- open | closing | closed
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    disconnected_at TIMESTAMPTZ,
    disconnect_reason VARCHAR(64)
);

CREATE INDEX idx_connections_tenant_state ON connections(tenant_id, state);
CREATE INDEX idx_connections_user ON connections(user_id) WHERE state = 'open';
CREATE INDEX idx_connections_server ON connections(server_node) WHERE state = 'open';

-- Channels / rooms / topics
CREATE TABLE channels (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    type        VARCHAR(16) NOT NULL DEFAULT 'public', -- public | private | presence | direct
    max_members INTEGER,
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- Many-to-many: connections subscribed to channels
CREATE TABLE channel_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    connection_id   UUID NOT NULL REFERENCES connections(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (channel_id, connection_id)
);

CREATE INDEX idx_subscriptions_channel ON channel_subscriptions(channel_id);
CREATE INDEX idx_subscriptions_connection ON channel_subscriptions(connection_id);

-- Message history and persistence
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID REFERENCES channels(id),
    sender_conn_id  UUID REFERENCES connections(id),
    sender_user_id  UUID REFERENCES users(id),
    message_type    VARCHAR(32) NOT NULL DEFAULT 'text', -- text | binary | system | control
    payload         TEXT,
    payload_size    INTEGER NOT NULL DEFAULT 0,
    idempotency_key VARCHAR(128),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_channel_time ON messages(channel_id, created_at DESC);
CREATE INDEX idx_messages_tenant_time ON messages(tenant_id, created_at DESC);

-- Presence tracking
CREATE TABLE presence_records (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    user_id     UUID NOT NULL REFERENCES users(id),
    channel_id  UUID REFERENCES channels(id),
    status      VARCHAR(16) NOT NULL DEFAULT 'online',  -- online | away | busy | offline
    custom_data JSONB DEFAULT '{}',
    last_seen   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, channel_id)
);

CREATE INDEX idx_presence_channel ON presence_records(channel_id, status);

-- Connection lifecycle events for auditing
CREATE TABLE connection_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    connection_id   UUID NOT NULL REFERENCES connections(id),
    event_type      VARCHAR(32) NOT NULL,  -- connected | authenticated | subscribed | unsubscribed | disconnected | error
    details         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conn_events_connection ON connection_events(connection_id, created_at);

-- Rate limiting configuration
CREATE TABLE rate_limit_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    scope           VARCHAR(32) NOT NULL,    -- tenant | user | connection | channel
    max_requests    INTEGER NOT NULL,
    window_seconds  INTEGER NOT NULL,
    burst_limit     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, scope)
);

-- Offline message queue
CREATE TABLE offline_messages (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    message_id      UUID NOT NULL REFERENCES messages(id),
    delivered       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    delivered_at    TIMESTAMPTZ
);

CREATE INDEX idx_offline_pending ON offline_messages(user_id, delivered) WHERE delivered = false;
```

---

## Pros

- **Strong consistency guarantees.** ACID transactions ensure that connection state, subscriptions, and messages are always in a consistent state. No partial writes or orphaned records.
- **Referential integrity.** Foreign keys enforce valid relationships between tenants, users, connections, channels, and messages. Prevents data corruption at the database level.
- **Mature query capabilities.** Complex queries across entities (e.g., "find all messages sent by user X in channel Y in the last hour") are straightforward with SQL joins.
- **Excellent tooling ecosystem.** PostgreSQL has decades of tooling for backups, replication, monitoring, schema migration (Flyway, Alembic, Prisma Migrate), and ORM support.
- **Multi-tenant isolation.** Row-level security (RLS) policies in PostgreSQL can enforce tenant isolation without application-level checks.
- **Auditing and compliance.** Relational schema makes it simple to query complete audit trails, aggregate billing data, and generate compliance reports.
- **Well-understood by teams.** Most backend engineers are fluent in relational modeling and SQL. Reduced onboarding friction.

## Cons

- **High-frequency writes are costly.** Every WebSocket message, presence heartbeat, and connection event results in a database write. At 10K messages/second, PostgreSQL will become a bottleneck without significant write optimization (partitioning, batching, write-ahead log tuning).
- **Presence tracking latency.** Real-time presence queries (who is online in this channel right now?) incur JOIN overhead compared to in-memory stores like Redis.
- **Connection state is ephemeral.** Storing transient connection data (which will be deleted on disconnect) in a durable relational store is wasteful I/O. The connections table will churn heavily.
- **Horizontal scaling is complex.** PostgreSQL does not natively shard. Scaling beyond a single write node requires Citus, logical partitioning by tenant, or application-level sharding.
- **No built-in pub/sub fan-out.** PostgreSQL LISTEN/NOTIFY is limited (8000 bytes per payload, no persistence, single-node). It cannot serve as the real-time message delivery mechanism.
- **Schema rigidity.** Adding new fields to messages or presence metadata requires migrations. In a rapidly evolving real-time protocol, this can slow iteration.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ |
| Connection pooling | PgBouncer (transaction-mode) |
| Schema migrations | Prisma Migrate, Flyway, or golang-migrate |
| Partitioning | Range-partition `messages` and `connection_events` by `created_at` (monthly) |
| Read replicas | Streaming replication for dashboard/analytics queries |
| Caching layer | Redis as a read-through cache for hot connection and presence data |
| Real-time fan-out | Redis Pub/Sub or NATS for actual message delivery (PostgreSQL stores the durable record) |
| Tenant isolation | PostgreSQL Row-Level Security policies keyed on `tenant_id` |

---

## Migration and Scaling Considerations

### Data Growth Projections

At moderate scale (10K concurrent connections, 1K messages/second):
- **messages** table grows ~86M rows/day. Must be partitioned and pruned aggressively.
- **connection_events** grows proportionally to connection churn. Partition by time.
- **connections** table should archive closed connections to a separate `connections_archive` table or partition.

### Partitioning Strategy

```sql
-- Partition messages by month for efficient pruning
CREATE TABLE messages (
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE messages_2026_05 PARTITION OF messages
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Scaling Path

1. **Vertical first.** PostgreSQL on modern hardware (NVMe, 64GB+ RAM) handles significant write loads. Tune `shared_buffers`, `wal_buffers`, `max_wal_size`, and `synchronous_commit = off` for non-critical writes.
2. **Read replicas.** Offload dashboard, analytics, and history queries to streaming replicas.
3. **Table partitioning.** Partition high-volume tables by time. Drop old partitions to manage storage.
4. **Tenant sharding.** When single-node write capacity is exceeded, shard by tenant using Citus or application-level routing.
5. **Hybrid approach.** Move hot-path data (active connections, presence, rate-limit counters) to Redis while PostgreSQL remains the system of record for durable data.

### Migration Notes

- Start with this normalized schema for the MVP.
- Monitor write throughput on `messages` and `connection_events` tables.
- When write latency exceeds SLA, introduce Redis for ephemeral state and batch-insert messages asynchronously.
- Consider migrating to the hybrid approach (Suggestion 3) as traffic grows.
