# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: WebSocket API Gateway · Candidate #284

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the gateway -- connection opened, message sent, user subscribed to a channel, presence changed -- is recorded as an immutable event in an append-only event store. Read-optimized projections (materialized views) are derived from these events to serve queries. This approach treats the event log as the single source of truth and rebuilds current state by replaying events.

This architecture is a natural fit for a WebSocket gateway because the gateway's domain is inherently event-driven: connections open and close, messages flow, users join and leave channels. Rather than fighting this reality by mapping it into mutable rows, event sourcing embraces it.

---

## Key Entities and Relationships

### Event Store Structure

```
event_store (append-only, immutable)
  └── ConnectionOpened
  └── ConnectionAuthenticated
  └── ConnectionClosed
  └── ChannelCreated
  └── ChannelDeleted
  └── UserSubscribed
  └── UserUnsubscribed
  └── MessagePublished
  └── MessageDelivered
  └── PresenceChanged
  └── RateLimitConfigured
  └── RateLimitExceeded

Read Models (projections, rebuilt from events)
  ├── active_connections_view
  ├── channel_members_view
  ├── presence_view
  ├── message_history_view
  ├── tenant_usage_view
  └── rate_limit_state_view
```

### Schema Snippets

#### Event Store (Write Side)

```sql
-- The single append-only event store
CREATE TABLE events (
    event_id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_type  VARCHAR(32) NOT NULL,   -- connection | channel | user | tenant
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    event_version   INTEGER NOT NULL,
    tenant_id       UUID NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, actor
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Optimistic concurrency: ensure no two events for the same aggregate
-- have the same version
CREATE UNIQUE INDEX idx_events_aggregate_version
    ON events(aggregate_type, aggregate_id, event_version);

-- Sequential read for projections
CREATE INDEX idx_events_sequence ON events(event_id);

-- Per-tenant event stream
CREATE INDEX idx_events_tenant ON events(tenant_id, created_at);

-- Partition by time for manageability
-- (events are immutable, so old partitions can be archived)
```

#### Example Event Payloads

```json
// ConnectionOpened
{
  "event_type": "ConnectionOpened",
  "aggregate_type": "connection",
  "aggregate_id": "conn-550e8400-...",
  "tenant_id": "tenant-123",
  "payload": {
    "user_id": "user-456",
    "server_node": "gateway-us-east-1a",
    "client_ip": "203.0.113.42",
    "user_agent": "Mozilla/5.0...",
    "transport": "websocket",
    "auth_method": "jwt"
  },
  "metadata": {
    "correlation_id": "req-789",
    "source": "gateway-us-east-1a"
  }
}

// MessagePublished
{
  "event_type": "MessagePublished",
  "aggregate_type": "channel",
  "aggregate_id": "channel-abc",
  "tenant_id": "tenant-123",
  "payload": {
    "message_id": "msg-def",
    "sender_user_id": "user-456",
    "sender_connection_id": "conn-550e8400-...",
    "content_type": "text/plain",
    "payload_size": 256,
    "payload_hash": "sha256:...",
    "content": "Hello, world!"
  },
  "metadata": {
    "correlation_id": "req-790",
    "idempotency_key": "client-key-001"
  }
}

// PresenceChanged
{
  "event_type": "PresenceChanged",
  "aggregate_type": "user",
  "aggregate_id": "user-456",
  "tenant_id": "tenant-123",
  "payload": {
    "channel_id": "channel-abc",
    "previous_status": "online",
    "new_status": "away",
    "custom_data": { "last_typing_at": "2026-05-25T10:30:00Z" }
  }
}
```

#### Command Handlers (Write Side)

```typescript
// Pseudocode: command handler for publishing a message
async function handlePublishMessage(command: PublishMessageCommand): Promise<void> {
  // Load aggregate (channel) by replaying its events
  const channel = await eventStore.loadAggregate('channel', command.channelId);

  // Business rules (rate limiting, authorization, validation)
  channel.assertUserIsSubscribed(command.userId);
  channel.assertNotRateLimited(command.userId);
  channel.assertPayloadWithinLimits(command.payload);

  // Emit event (appended to event store)
  await eventStore.append({
    aggregateType: 'channel',
    aggregateId: command.channelId,
    eventType: 'MessagePublished',
    eventVersion: channel.version + 1,
    tenantId: command.tenantId,
    payload: {
      messageId: generateId(),
      senderUserId: command.userId,
      content: command.payload,
      payloadSize: command.payload.length,
    },
  });
}
```

#### Read Model Projections (Query Side)

```sql
-- Projection: active connections (rebuilt from ConnectionOpened/ConnectionClosed events)
CREATE TABLE active_connections_view (
    connection_id   UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    server_node     VARCHAR(255) NOT NULL,
    client_ip       INET,
    transport       VARCHAR(16),
    connected_at    TIMESTAMPTZ NOT NULL,
    last_event_id   BIGINT NOT NULL  -- watermark for rebuilding
);

CREATE INDEX idx_active_conn_tenant ON active_connections_view(tenant_id);
CREATE INDEX idx_active_conn_user ON active_connections_view(user_id);

-- Projection: channel membership
CREATE TABLE channel_members_view (
    channel_id      UUID NOT NULL,
    user_id         UUID NOT NULL,
    connection_id   UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    subscribed_at   TIMESTAMPTZ NOT NULL,
    last_event_id   BIGINT NOT NULL,
    PRIMARY KEY (channel_id, connection_id)
);

-- Projection: current presence
CREATE TABLE presence_view (
    user_id         UUID NOT NULL,
    channel_id      UUID,
    tenant_id       UUID NOT NULL,
    status          VARCHAR(16) NOT NULL,
    custom_data     JSONB DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   BIGINT NOT NULL,
    PRIMARY KEY (user_id, channel_id)
);

-- Projection: message history (denormalized for fast reads)
CREATE TABLE message_history_view (
    message_id      UUID PRIMARY KEY,
    channel_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    sender_user_id  UUID,
    sender_name     VARCHAR(255),
    content_type    VARCHAR(32),
    content         TEXT,
    payload_size    INTEGER,
    published_at    TIMESTAMPTZ NOT NULL,
    last_event_id   BIGINT NOT NULL
);

CREATE INDEX idx_msg_history_channel ON message_history_view(channel_id, published_at DESC);

-- Projection: tenant usage for billing and analytics
CREATE TABLE tenant_usage_view (
    tenant_id           UUID NOT NULL,
    period_start        DATE NOT NULL,
    total_connections    BIGINT DEFAULT 0,
    total_messages       BIGINT DEFAULT 0,
    total_bytes          BIGINT DEFAULT 0,
    peak_concurrent      INTEGER DEFAULT 0,
    unique_users         INTEGER DEFAULT 0,
    last_event_id        BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, period_start)
);
```

#### Projection Processor

```typescript
// Pseudocode: projection that maintains the active_connections_view
class ActiveConnectionsProjection implements EventHandler {
  async handle(event: Event): Promise<void> {
    switch (event.eventType) {
      case 'ConnectionOpened':
        await db.query(`
          INSERT INTO active_connections_view
          (connection_id, tenant_id, user_id, server_node, client_ip, transport, connected_at, last_event_id)
          VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        `, [event.aggregateId, event.tenantId, event.payload.user_id,
            event.payload.server_node, event.payload.client_ip,
            event.payload.transport, event.createdAt, event.eventId]);
        break;

      case 'ConnectionClosed':
        await db.query(`
          DELETE FROM active_connections_view WHERE connection_id = $1
        `, [event.aggregateId]);
        break;
    }
  }
}
```

---

## Pros

- **Complete audit trail by design.** Every state change is permanently recorded. Connection lifecycle, message delivery, presence transitions, and rate-limit violations are all in the event log. Debugging a user's session is a matter of querying events by aggregate ID.
- **Temporal queries.** "What was the state of channel X at 3:00 PM?" is answerable by replaying events up to that timestamp. This is extremely valuable for debugging real-time systems.
- **Natural fit for the domain.** WebSocket gateways are inherently event-driven. Connections open, messages flow, users join/leave. Event sourcing models the domain directly rather than forcing it into mutable rows.
- **Independent read/write scaling.** The event store (write side) and projections (read side) can be scaled independently. High-throughput event appends can use a fast sequential-write store while projections can be replicated across read replicas.
- **Multiple read models from one event stream.** The same events can power a real-time presence view, a message history view, a billing/usage view, and an analytics pipeline -- each optimized for its query patterns.
- **Supports replay and recovery.** If a read model becomes corrupted or a new projection is needed, it can be rebuilt from scratch by replaying the event log.
- **AI-native debugging.** The AI-powered session reconstruction feature described in the README is trivially implemented: the LLM summarizes the event stream for a given connection or user.

## Cons

- **Complexity.** Event sourcing + CQRS introduces significant architectural complexity: event store, command handlers, event handlers, projection processors, snapshotting, and idempotency management. This is a steep learning curve for teams unfamiliar with the pattern.
- **Eventual consistency on reads.** Projections may lag behind the event store. A user who just sent a message may not see it in the history view for a few milliseconds. This is usually acceptable for real-time messaging but requires careful UX handling.
- **Event schema evolution.** Changing the shape of events (adding/removing fields) requires versioning and upcasting strategies. This is manageable but adds overhead compared to a simple ALTER TABLE.
- **Storage growth.** The event store grows indefinitely. At 1K events/second, the store accumulates ~86M events/day. Snapshotting reduces replay cost but does not reduce storage.
- **Projection rebuild time.** Rebuilding a projection from scratch (e.g., after adding a new read model) can take hours on a large event store. Snapshotting mitigates this but adds complexity.
- **Overkill for simple queries.** "List all channels for tenant X" requires maintaining a separate projection rather than a simple SELECT query. Every new query pattern may require a new projection.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL (append-only table, partitioned by time) or EventStoreDB for a purpose-built solution |
| Event bus | Apache Kafka or NATS JetStream for durable event streaming between services |
| Projection database | PostgreSQL for relational projections; Redis for hot presence/connection views |
| Command handling | Application-level (Node.js/Go/Rust) with optimistic concurrency via event versioning |
| Snapshotting | Periodic aggregate snapshots stored in a separate table or object store (S3) |
| Event schema registry | AsyncAPI or Avro schema registry for event payload versioning |
| Replay tooling | Custom replay CLI or Kafka consumer group reset for projection rebuilds |
| Monitoring | Track projection lag (difference between latest event ID and projection watermark) |

---

## Migration and Scaling Considerations

### Event Store Partitioning

```sql
-- Partition event store by month
CREATE TABLE events (
    ...
) PARTITION BY RANGE (created_at);

-- Archive old partitions to cold storage (S3 + Parquet)
-- after projection snapshots are taken
```

### Snapshotting Strategy

For aggregates with long event histories (e.g., a channel with millions of messages), loading the aggregate by replaying all events is too slow. Snapshots capture the aggregate state at a point in time:

```sql
CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(32) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

Rebuild the aggregate by loading the latest snapshot and replaying only events after the snapshot version.

### Scaling Path

1. **Single-node PostgreSQL.** The event store is append-only, which is the best-case workload for PostgreSQL WAL. A single node handles high write throughput.
2. **Kafka as the event bus.** Introduce Kafka between the event store and projections for decoupled, scalable event distribution. Projections become Kafka consumers.
3. **Separate projection databases.** Move hot projections (active connections, presence) to Redis. Keep analytical projections (usage, billing) in PostgreSQL read replicas.
4. **Event store sharding.** Partition by tenant ID for multi-tenant isolation and horizontal write scaling. Each tenant's event stream is independent.
5. **Cold storage archival.** After snapshotting, archive old event partitions to object storage (S3 + Parquet) for cost-efficient long-term retention. Events remain queryable via tools like DuckDB or Athena.

### Migration from Other Approaches

- If starting with the normalized relational model (Suggestion 1), introduce event sourcing incrementally: add an events table alongside the existing tables, dual-write during transition, then migrate read queries to projections.
- The event store can coexist with the hybrid model (Suggestion 3) by using the event log as the write path and the hybrid tables as materialized projections.
