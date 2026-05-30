# Data Model Suggestion 4: Redis-Centric with PostgreSQL Persistence

> Project: WebSocket API Gateway · Candidate #284

## Summary

A Redis-primary architecture where all hot-path, real-time state -- active connections, presence, channel subscriptions, rate-limit counters, and pub/sub message routing -- lives in Redis data structures. PostgreSQL serves as the cold-path persistence layer for message history, audit logs, billing data, and tenant configuration. This architecture recognizes that a WebSocket API gateway's primary workload is ephemeral, high-frequency, and latency-sensitive -- exactly what Redis is built for -- while durable business data belongs in a relational store.

This is the architecture used in production by systems like Discord (Redis + Cassandra), Slack (Redis + MySQL), and most WebSocket infrastructure at scale. Redis is not just a cache in this model; it is the authoritative source of truth for real-time state.

---

## Key Entities and Data Structures

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Redis Cluster                             │
│                                                               │
│  Hot Path (authoritative for real-time state):                │
│  ├── Connection Registry     (Hash per connection)            │
│  ├── User Connection Index   (Set per user)                   │
│  ├── Server Node Index       (Set per node)                   │
│  ├── Channel Membership      (Set per channel)                │
│  ├── Channel Metadata        (Hash per channel)               │
│  ├── Presence State          (Hash per user-channel)          │
│  ├── Presence Channel Index  (Sorted Set per channel)         │
│  ├── Rate Limit Counters     (Sorted Set per scope)           │
│  ├── Offline Message Queue   (List per user)                  │
│  └── Pub/Sub Channels        (native Redis pub/sub)           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
          │ async write-behind │
          ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│                    PostgreSQL                                 │
│                                                               │
│  Cold Path (durable persistence):                             │
│  ├── tenants                 (configuration, billing)         │
│  ├── users                   (identity, profiles)             │
│  ├── api_keys                (authentication)                 │
│  ├── channels                (durable channel definitions)    │
│  ├── messages                (message history archive)        │
│  ├── connection_events       (audit log)                      │
│  ├── routing_rules           (AI routing configuration)       │
│  ├── rate_limit_policies     (rate limit definitions)         │
│  └── tenant_usage            (billing aggregates)             │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Redis Data Structures

#### Connection Registry

```
# Each connection is a Redis Hash
# Key: conn:{tenant_id}:{connection_id}
# TTL: auto-expire after max connection lifetime (e.g., 24h)

HSET conn:tenant-123:conn-abc \
  user_id       "user-456" \
  server_node   "gw-us-east-1a" \
  client_ip     "203.0.113.42" \
  transport     "websocket" \
  connected_at  "2026-05-25T10:00:00Z" \
  user_agent    "Mozilla/5.0..." \
  auth_method   "jwt" \
  protocol_ver  "1.2"

EXPIRE conn:tenant-123:conn-abc 86400

# Index: connections per user (for disconnect-all, presence)
# Key: user_conns:{tenant_id}:{user_id}
SADD user_conns:tenant-123:user-456 "conn-abc"

# Index: connections per server node (for node failover)
# Key: node_conns:{server_node}
SADD node_conns:gw-us-east-1a "tenant-123:conn-abc"

# Index: all connections per tenant (for tenant-wide operations)
# Key: tenant_conns:{tenant_id}
SADD tenant_conns:tenant-123 "conn-abc"
```

#### Channel Membership

```
# Channel metadata as a Hash
# Key: channel:{tenant_id}:{channel_name}
HSET channel:tenant-123:chat-general \
  id            "channel-xyz" \
  type          "public" \
  created_at    "2026-05-25T09:00:00Z" \
  max_members   1000 \
  message_count 0

# Channel members as a Set of connection IDs
# Key: channel_members:{tenant_id}:{channel_name}
SADD channel_members:tenant-123:chat-general "conn-abc" "conn-def" "conn-ghi"

# Reverse index: channels per connection (for cleanup on disconnect)
# Key: conn_channels:{tenant_id}:{connection_id}
SADD conn_channels:tenant-123:conn-abc "chat-general" "chat-support"
```

#### Presence Tracking

```
# Per-user presence state as a Hash
# Key: presence:{tenant_id}:{user_id}
HSET presence:tenant-123:user-456 \
  status        "online" \
  last_seen     "2026-05-25T10:05:00Z" \
  custom_data   '{"typing": false, "viewing": "/dashboard"}'

EXPIRE presence:tenant-123:user-456 120  # 2-minute TTL, refreshed by heartbeats

# Per-channel presence as a Sorted Set (score = last_seen timestamp)
# Key: channel_presence:{tenant_id}:{channel_name}
# Score: Unix timestamp of last heartbeat
ZADD channel_presence:tenant-123:chat-general \
  1748167500 "user-456" \
  1748167490 "user-789"

# Query: who is online in this channel?
# (users with heartbeat in the last 60 seconds)
ZRANGEBYSCORE channel_presence:tenant-123:chat-general \
  (1748167440 +inf

# Query: how many users online in this channel?
ZCOUNT channel_presence:tenant-123:chat-general \
  (1748167440 +inf

# Detect stale presence: remove users who missed heartbeats
ZRANGEBYSCORE channel_presence:tenant-123:chat-general \
  -inf 1748167440
# -> returns user IDs to mark as offline
```

#### Rate Limiting

```
# Sliding window rate limiter using Sorted Sets
# Key: ratelimit:{scope}:{tenant_id}:{identifier}
# Score: request timestamp (microseconds)
# Member: unique request ID (to avoid collisions)

# Record a message send event
ZADD ratelimit:msg:tenant-123:user-456 1748167500123 "req-001"

# Remove entries outside the window (60-second window)
ZREMRANGEBYSCORE ratelimit:msg:tenant-123:user-456 -inf (1748167440123

# Count requests in the current window
ZCARD ratelimit:msg:tenant-123:user-456
# -> if count > max_requests, reject

# Connection-level rate limiting (connections per minute per IP)
ZADD ratelimit:conn:tenant-123:203.0.113.42 1748167500123 "conn-abc"

# All three operations in a single MULTI/EXEC for atomicity
MULTI
  ZREMRANGEBYSCORE ratelimit:msg:tenant-123:user-456 -inf (1748167440123
  ZADD ratelimit:msg:tenant-123:user-456 1748167500123 "req-001"
  ZCARD ratelimit:msg:tenant-123:user-456
EXEC

# Auto-expire the key after the window passes (cleanup)
EXPIRE ratelimit:msg:tenant-123:user-456 120
```

#### Pub/Sub Message Routing

```
# Redis Pub/Sub for cross-node message fan-out
# Gateway nodes subscribe to channels their connections care about

# Node gw-us-east-1a subscribes to channels it has members in:
SUBSCRIBE tenant-123:chat-general
SUBSCRIBE tenant-123:chat-support
SUBSCRIBE tenant-123:dm:user-456:user-789

# Publishing a message (from the node that received it):
PUBLISH tenant-123:chat-general '{"msg_id":"msg-001","user_id":"user-456","text":"Hello!"}'

# Pattern subscriptions for wildcard routing:
PSUBSCRIBE tenant-123:support-*
# Matches: tenant-123:support-billing, tenant-123:support-technical, etc.
```

#### Offline Message Queue

```
# Pending messages for offline users as a List
# Key: offline:{tenant_id}:{user_id}
RPUSH offline:tenant-123:user-456 \
  '{"msg_id":"msg-002","channel":"chat-general","text":"Missed message","ts":"2026-05-25T10:10:00Z"}'

# When user reconnects, drain the queue:
LRANGE offline:tenant-123:user-456 0 -1
DEL offline:tenant-123:user-456

# Cap the queue to prevent unbounded growth:
LTRIM offline:tenant-123:user-456 0 999  # Keep last 1000 messages
```

### PostgreSQL Schema (Cold Path)

```sql
-- Tenant and user tables are the same as Suggestion 1/3
-- These are the source of truth for durable identity

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(63) NOT NULL UNIQUE,
    plan            VARCHAR(32) NOT NULL DEFAULT 'free',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

-- Message archive (async write-behind from Redis)
CREATE TABLE messages (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_name    VARCHAR(255) NOT NULL,
    sender_user_id  UUID REFERENCES users(id),
    message_type    VARCHAR(32) NOT NULL DEFAULT 'text',
    payload         JSONB NOT NULL,
    payload_size    INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_messages_channel_time ON messages(channel_name, created_at DESC);

-- Connection event audit log (async write-behind)
CREATE TABLE connection_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    connection_id   UUID NOT NULL,
    user_id         UUID,
    event_type      VARCHAR(32) NOT NULL,
    server_node     VARCHAR(255),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_conn_events_conn ON connection_events(connection_id, created_at);

-- Tenant usage aggregates (computed from events, used for billing)
CREATE TABLE tenant_usage (
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    period_start        DATE NOT NULL,
    total_connections    BIGINT DEFAULT 0,
    total_messages       BIGINT DEFAULT 0,
    total_bytes          BIGINT DEFAULT 0,
    peak_concurrent      INTEGER DEFAULT 0,
    unique_users         INTEGER DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, period_start)
);

-- Routing rules (loaded into Redis on startup / change)
CREATE TABLE routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    rule            JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- Durable channel definitions (synced to Redis)
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(16) NOT NULL DEFAULT 'public',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

### Write-Behind Pipeline

```typescript
// Pseudocode: async pipeline from Redis to PostgreSQL

class WriteBehindPipeline {
  private buffer: Event[] = [];
  private flushInterval = 1000; // 1 second
  private maxBatchSize = 500;

  async onMessagePublished(msg: Message): Promise<void> {
    // 1. Publish to Redis Pub/Sub immediately (real-time delivery)
    await redis.publish(`${msg.tenantId}:${msg.channelName}`, JSON.stringify(msg));

    // 2. Buffer for async PostgreSQL write
    this.buffer.push({
      type: 'message',
      data: msg,
      timestamp: Date.now(),
    });

    // 3. Flush when buffer is full
    if (this.buffer.length >= this.maxBatchSize) {
      await this.flush();
    }
  }

  async flush(): Promise<void> {
    if (this.buffer.length === 0) return;

    const batch = this.buffer.splice(0, this.maxBatchSize);
    const messages = batch.filter(e => e.type === 'message');
    const events = batch.filter(e => e.type === 'connection_event');

    // Batch insert into PostgreSQL
    await Promise.all([
      this.batchInsertMessages(messages),
      this.batchInsertEvents(events),
    ]);
  }
}
```

---

## Redis Cluster Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    Redis Cluster (6 nodes)                   │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Primary 1    │  │ Primary 2    │  │ Primary 3    │      │
│  │ Slots 0-5460 │  │ Slots 5461-  │  │ Slots 10923- │      │
│  │              │  │ 10922        │  │ 16383        │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                 │                 │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐      │
│  │ Replica 1    │  │ Replica 2    │  │ Replica 3    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  Key distribution:                                           │
│  - {tenant-123}:conn:abc  → hash({tenant-123}) → slot N     │
│  - {tenant-123}:channel:X → hash({tenant-123}) → slot N     │
│  - Keys with same {hash_tag} co-locate on same shard         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Use Redis hash tags `{tenant_id}` to ensure all keys for a single tenant land on the same shard, enabling multi-key operations (MULTI/EXEC, Lua scripts) within a tenant.

---

## Connection Disconnect Cleanup (Lua Script)

```lua
-- Atomic cleanup when a connection disconnects
-- Runs as a Redis Lua script for atomicity
-- KEYS[1] = conn:{tenant}:{conn_id}
-- KEYS[2] = user_conns:{tenant}:{user_id}
-- KEYS[3] = node_conns:{server_node}
-- KEYS[4] = tenant_conns:{tenant}
-- ARGV[1] = conn_id
-- ARGV[2] = tenant:conn_id (for node index)

local conn_key = KEYS[1]
local user_conns_key = KEYS[2]
local node_conns_key = KEYS[3]
local tenant_conns_key = KEYS[4]
local conn_id = ARGV[1]
local tenant_conn_id = ARGV[2]

-- Get channel subscriptions before cleanup
local channels_key = string.gsub(conn_key, "^conn:", "conn_channels:")
local channels = redis.call('SMEMBERS', channels_key)

-- Remove from each channel's member set
for _, channel_name in ipairs(channels) do
  local members_key = string.gsub(conn_key, "conn:[^:]+:" .. conn_id, "channel_members:" .. channel_name)
  redis.call('SREM', members_key, conn_id)
end

-- Remove channel subscriptions set
redis.call('DEL', channels_key)

-- Remove from user connections index
redis.call('SREM', user_conns_key, conn_id)

-- Remove from node connections index
redis.call('SREM', node_conns_key, tenant_conn_id)

-- Remove from tenant connections index
redis.call('SREM', tenant_conns_key, conn_id)

-- Delete the connection hash
redis.call('DEL', conn_key)

-- Check if user has any remaining connections
local remaining = redis.call('SCARD', user_conns_key)
return remaining  -- if 0, caller should update presence to offline
```

---

## Pros

- **Sub-millisecond latency for hot-path operations.** Connection lookups, presence queries, rate-limit checks, and channel membership are all in-memory Redis operations. No disk I/O on the critical path.
- **Purpose-built data structures.** Redis Sorted Sets for presence (ordered by last-seen), Sets for channel membership, Hashes for connection metadata, and Lists for offline queues -- each data structure matches its use case perfectly.
- **Native pub/sub.** Redis Pub/Sub provides the cross-node message fan-out mechanism built into the data layer. No additional message broker needed for basic routing.
- **Atomic multi-key operations.** Lua scripts and MULTI/EXEC transactions ensure connection cleanup, subscription changes, and rate-limit checks are atomic -- critical for correctness in a concurrent gateway.
- **Natural TTL-based cleanup.** Redis key expiration handles stale connections (node crash without clean disconnect) and presence timeout (missed heartbeats) automatically. No cron jobs or garbage collection processes.
- **Horizontal scaling.** Redis Cluster distributes data across shards. Hash tags ensure per-tenant locality for multi-key operations.
- **Production-proven pattern.** Discord, Slack, Twitch, and most large-scale real-time systems use Redis as their real-time state store. The pattern is well-documented and battle-tested.

## Cons

- **Two systems to operate.** Both Redis Cluster and PostgreSQL must be deployed, monitored, backed up, and scaled. Operational complexity is higher than a single-database approach.
- **Data consistency between Redis and PostgreSQL.** The write-behind pipeline introduces eventual consistency between real-time state and durable records. If the pipeline fails, messages may be delivered but not persisted. Requires careful error handling and dead-letter queues.
- **Redis memory cost.** All hot-path data must fit in RAM. At 100K concurrent connections with metadata, channel memberships, and presence state, Redis memory usage can reach 10-50 GB depending on payload sizes.
- **Redis Pub/Sub is fire-and-forget.** Messages published when no subscriber is listening are lost. This is fine for real-time delivery but means the pub/sub layer cannot guarantee delivery. Offline queues must be implemented separately.
- **Lua script complexity.** Atomic operations (connection cleanup, rate limiting) require Lua scripts that are harder to develop, test, and debug than SQL transactions.
- **No ad-hoc queries on Redis.** You cannot run analytical queries (e.g., "average message size by channel over the last week") against Redis. All analytics must go through PostgreSQL after the write-behind pipeline.
- **Redis Cluster limitations.** Multi-key operations only work within the same hash slot. Cross-tenant operations (e.g., global admin dashboard) require scatter-gather across shards.
- **Persistence risk.** Redis RDB snapshots and AOF logging provide durability, but a crash between snapshots can lose recent state. Acceptable for ephemeral connection data but requires the write-behind pipeline to PostgreSQL for anything that must survive.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Real-time state store | Redis 7+ Cluster (3 primaries + 3 replicas minimum) |
| Durable persistence | PostgreSQL 16+ (partitioned tables for messages and events) |
| Write-behind pipeline | Application-level async batching, or Redis Streams as an intermediate buffer |
| Cross-node pub/sub | Redis Pub/Sub for basic fan-out; upgrade to Redis Streams or NATS JetStream if guaranteed delivery is needed |
| Rate limiting | Redis Sorted Sets with Lua scripts (sliding window algorithm) |
| Presence | Redis Sorted Sets (score = heartbeat timestamp) with TTL-based expiry |
| Connection cleanup | Redis Lua scripts for atomic multi-key cleanup on disconnect |
| Monitoring | Redis `INFO`, `MEMORY USAGE`, `SLOWLOG`; Prometheus with redis_exporter |
| Failover | Redis Sentinel or Cluster auto-failover; application-level reconnection |

---

## Migration and Scaling Considerations

### Memory Estimation

| Data Type | Per-Unit Size (approx.) | At 100K Connections |
|-----------|------------------------|---------------------|
| Connection Hash | ~500 bytes | 50 MB |
| User Connection Set | ~100 bytes | 10 MB |
| Channel Member Set | ~50 bytes/member | 50 MB (avg 10 channels/user) |
| Presence Hash | ~200 bytes | 20 MB |
| Rate Limit Sorted Set | ~100 bytes/window | 10 MB |
| Pub/Sub overhead | ~negligible | ~negligible |
| **Total** | | **~140 MB** |

At 1M concurrent connections, expect ~1.4 GB of Redis memory for real-time state. Well within a single Redis node's capacity (typical deployments use 25-50 GB nodes).

### Scaling Path

1. **Single Redis instance + PostgreSQL.** Sufficient for MVP and early production (up to ~50K concurrent connections).
2. **Redis Sentinel.** Add replicas and automatic failover for high availability.
3. **Redis Cluster.** Shard by tenant hash tag when a single node's memory or throughput is exceeded.
4. **Redis Streams.** Replace basic Pub/Sub with Redis Streams for guaranteed delivery and consumer groups (e.g., for the write-behind pipeline).
5. **Multi-region.** Deploy Redis Cluster per region with cross-region sync via Redis Enterprise Active-Active (CRDT-based) or application-level event replication through Kafka.

### Node Failure Recovery

When a gateway node crashes without clean disconnection:

1. Redis key TTLs automatically expire stale connection data (within the configured TTL window).
2. Other gateway nodes detect the failure via health checks and proactively clean up connections for the failed node using the `node_conns:{server_node}` index.
3. The write-behind pipeline records `ConnectionClosed` events with `reason: "node_failure"` in PostgreSQL.
4. Clients reconnect to healthy nodes and re-establish their subscriptions.

### Migration from Other Approaches

- From Suggestion 1 (pure PostgreSQL): Introduce Redis as a read-through/write-through cache for connections and presence. Gradually shift real-time operations to Redis while keeping PostgreSQL as the write-behind target.
- From Suggestion 3 (hybrid): Same migration path -- move hot-path JSONB reads/writes to Redis, keep PostgreSQL for durable data.
- The PostgreSQL schema in this suggestion is intentionally compatible with Suggestions 1 and 3, making migration straightforward.
