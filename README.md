# WebSocket API Gateway

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native real-time messaging gateway that delivers managed-grade presence, routing, and observability without the per-message price tag of incumbent SaaS.

WebSocket API Gateway is real-time messaging infrastructure for developers building chat, live dashboards, multiplayer experiences, and collaborative SaaS. It provides connection management, pub/sub routing, presence tracking, and rate limiting with first-class AI capabilities for routing, scaling, and anomaly detection — closing the gap between expensive managed services and self-hosted libraries that leave operations to you.

---

## Why WebSocket API Gateway?

- **Managed services are expensive at scale.** Ably and PubNub enterprise deals run $1K–$10K+/month, and AWS API Gateway WebSocket bills per connection-minute plus messages with a hard 2-hour idle timeout that breaks long-lived sessions.
- **Open-source alternatives shift the burden to you.** Socket.IO and Soketi are free but require operating, scaling, and monitoring your own fleet — exactly what teams adopt managed gateways to avoid.
- **Edge-native options come with lock-in.** Cloudflare Durable Objects is powerful but ties applications to one cloud provider's runtime model.
- **Presence is the most-requested capability after messaging**, yet implementations vary widely in correctness, especially across reconnects, network partitions, and multi-region deployments.
- **Routing today is static.** Channels and topics are fixed at publish time, leaving no room to route by message content, user attributes, or live signals without bolting on a separate service.

---

## Key Features

### Connection & Messaging Core

- WebSocket connection management with lifecycle hooks
- Message broadcasting to rooms and channels
- User-to-user direct messaging
- Group channel management
- Connection rate limiting and throttling

### Presence & State

- User presence tracking and status
- Offline message queue
- Message history and persistence
- Connection state recovery across reconnects

### Operations & Observability

- Message logging and auditing
- Basic dashboard for monitoring
- Monitoring and alerting
- Cost analytics per user and per channel
- CLI tools for management

### Security & Integration

- User authentication integration
- Integration with authentication providers
- Automatic failover and redundancy
- Global edge network deployment

---

## AI-Native Advantage

The gateway routes messages by semantic content classification rather than only by static topic channels, and predicts traffic spikes from upstream signals (calendar events, deployment windows, marketing campaigns) to scale connection infrastructure before load arrives. Real-time anomaly detection flags flooding, oscillating reconnects, and unexpected message-size spikes. Operators can configure routing rules, rate limits, and presence groups in natural language, and reconstruct a user's connection lifecycle and message exchange to debug support issues without trawling raw logs.

---

## Tech Stack & Deployment

The project targets RFC 6455 (WebSocket Protocol) as its foundation, with adjacent support for Server-Sent Events for read-heavy unidirectional feeds, MQTT for IoT and low-bandwidth telemetry, AMQP where durable broker semantics are required, and gRPC streaming for internal service-to-service use. Deployment is intended to support self-hosted, cloud, and global edge models, including serverless integration paths similar to AWS API Gateway WebSocket.

---

## Market Context

The real-time communication and messaging API market is estimated at USD 7.4 billion in 2025, growing at roughly 20% CAGR, with WebSocket-based infrastructure a primary driver. Incumbent pricing ranges from Pusher's $49/month Startup plan up to $1K–$10K+/month enterprise deals with Ably or PubNub. Primary buyers are full-stack developers building chat and live dashboards, platform engineers standardising real-time infrastructure, gaming backend engineers needing low-latency messaging, and SaaS product teams adding presence indicators and live cursors.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
