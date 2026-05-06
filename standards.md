# Standards & API Reference

> Project: WebSocket API Gateway · Generated: 2026-05-03

## Industry Standards & Specifications

### IETF / RFC Standards

**RFC 6455 — The WebSocket Protocol**
- URL: https://www.rfc-editor.org/rfc/rfc6455
- The foundational IETF standard (published December 2011) defining the WebSocket handshake, framing, connection lifecycle, and security model. All WebSocket implementations — client libraries, server frameworks, and managed gateways — must conform to RFC 6455. Key areas: the HTTP upgrade handshake, binary and text frame types, control frames (ping/pong/close), and TLS (WSS) operation.

**RFC 8441 — Bootstrapping WebSockets with HTTP/2**
- URL: https://httpwg.org/specs/rfc8441.html
- Extends the HTTP CONNECT method to allow WebSocket streams to be multiplexed over a single HTTP/2 connection. Enables one TCP connection to carry both traditional HTTP/2 traffic and WebSocket tunnels, improving connection efficiency. Directly relevant to gateway designs that proxy WebSocket connections through HTTP/2 upstreams.

**RFC 9220 — Bootstrapping WebSockets with HTTP/3**
- URL: https://datatracker.ietf.org/doc/html/rfc9220
- Published June 2022. Adapts the RFC 8441 mechanism for HTTP/3 (QUIC transport), enabling WebSocket streams over QUIC with multiplexing, 0-RTT connection establishment, and elimination of TCP head-of-line blocking. Browser adoption is nascent as of 2026; relevant for forward-looking gateway designs targeting QUIC infrastructure.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Defines JWT as a compact, URL-safe representation of claims transferred between parties as a signed or encrypted JSON object. The dominant mechanism for authenticating WebSocket connections: clients include a JWT in the initial HTTP upgrade request (query string, Authorization header, or first message), and the gateway validates the token before accepting the connection.

**RFC 9700 — Best Current Practice for OAuth 2.0 Security**
- URL: https://datatracker.ietf.org/doc/rfc9700/
- Published January 2025. Updates and consolidates OAuth 2.0 security advice from RFCs 6749, 6750, and 6819 based on practical deployment experience. Directly applicable to gateway designs that issue or validate OAuth tokens for WebSocket client authentication, particularly for browser-based applications where implicit and hybrid flows have known risks.

**RFC 6902 — JSON Patch**
- URL: https://datatracker.ietf.org/doc/html/rfc6902
- Defines a JSON document format for expressing a sequence of operations to apply to a target JSON document. Widely used in real-time collaboration use cases (shared state synchronisation) to transmit minimal diffs over WebSocket rather than full document snapshots.

### W3C & WHATWG Standards

**WHATWG HTML Living Standard — WebSocket API (EventSource & WebSocket interfaces)**
- URL: https://websockets.spec.whatwg.org/
- The living browser-side specification defining the `WebSocket` JavaScript interface, connection lifecycle events (`onopen`, `onmessage`, `onerror`, `onclose`), and binary data handling (`Blob`, `ArrayBuffer`). Any client-side WebSocket SDK or gateway that targets browsers must conform to this interface. Last updated March 2026.

**WHATWG HTML Living Standard — Server-Sent Events (EventSource)**
- URL: https://html.spec.whatwg.org/dev/server-sent-events.html
- Specifies the `EventSource` interface for unidirectional server-to-client streaming over HTTP using `text/event-stream`. SSE is increasingly used alongside WebSocket gateways to serve read-heavy broadcast feeds (dashboards, notifications) with simpler infrastructure. Gateways should consider offering SSE as a complementary transport.

**W3C WebTransport Working Draft**
- URL: https://www.w3.org/TR/webtransport/
- A W3C Working Draft (last updated March 2026) defining a browser API for low-latency bidirectional transport over HTTP/3 (QUIC), providing unidirectional streams, bidirectional streams, and unreliable datagrams. Positioned as the next-generation successor to WebSockets for latency-sensitive use cases (gaming, media streaming, financial ticks). Gateway architects should monitor this spec for future transport negotiation requirements.

### Messaging Protocol Standards

**MQTT Version 5.0 — OASIS Standard**
- URL: https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- The OASIS standard lightweight publish/subscribe messaging protocol designed for constrained IoT devices and low-bandwidth networks. MQTT v5 adds reason codes, shared subscriptions, message expiry, and topic aliases. Overlaps with WebSocket gateways in IoT telemetry and device-to-cloud scenarios; many WebSocket gateways expose an MQTT-over-WebSocket binding to serve IoT clients alongside web clients.

**STOMP 1.2 — Simple Text Oriented Messaging Protocol**
- URL: https://stomp.github.io/stomp-specification-1.2.html
- A simple, interoperable text-based messaging protocol that runs over any reliable two-way stream (including WebSockets). Defines CONNECT, SEND, SUBSCRIBE, and RECEIPT commands for pub/sub and point-to-point messaging. Widely supported by message brokers (RabbitMQ, ActiveMQ) and used in Spring Boot WebSocket applications; a WebSocket gateway may choose to expose a STOMP endpoint for broker-backed routing.

**AsyncAPI Specification 3.x**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
- An open specification (inspired by OpenAPI) for describing event-driven and asynchronous APIs, with native bindings for WebSocket, Kafka, MQTT, AMQP, and other protocols. The standard for documenting WebSocket gateway APIs: defines channels, messages, schemas, bindings, and server declarations. Gateways should publish AsyncAPI documents to enable SDK generation and developer portal integration.

### Security Standards & Frameworks

**OWASP WebSocket Security Cheat Sheet**
- URL: https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html
- OWASP's authoritative security guidance for WebSocket implementations. Covers Cross-Site WebSocket Hijacking (CSWSH) prevention via Origin header validation, token-based authentication (avoiding cookie-only auth), TLS enforcement (WSS), rate limiting, message size limits, and logging requirements. A WebSocket gateway must address all items in this cheat sheet.

**OWASP Secure API Gateway Blueprint**
- URL: https://owasp.org/www-project-secure-api-gateway-blueprint/
- OWASP project defining security requirements and controls for API gateways, including authentication (JWT, mTLS, API keys), authorisation (RBAC, ABAC, scope-based), threat protection (DDoS, injection), and audit logging. Provides a structured framework for implementing security controls in a WebSocket gateway.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- The identity layer built on OAuth 2.0 that introduces the ID token (a JWT containing authenticated user claims). WebSocket gateways integrate with OIDC providers (Okta, Auth0, Keycloak) to authenticate connections at the HTTP upgrade stage, extracting user identity from the OIDC token for presence attribution and authorisation decisions.

---

## Similar Products — Developer Documentation & APIs

### Ably

- **Description:** Enterprise-grade managed real-time messaging platform with pub/sub channels, presence, connection state recovery, and a global edge network. The most feature-complete managed WebSocket service.
- **API Documentation:** https://ably.com/docs/api
- **REST API Reference:** https://ably.com/docs/api/rest-api
- **Realtime SDK Reference:** https://ably.com/docs/api/realtime-sdk
- **SDKs/Libraries:** JavaScript/TypeScript (`ably` on npm, https://github.com/ably/ably-js), Python, Go, Java, Swift, Kotlin, Ruby, PHP, .NET, Flutter
- **Developer Guide:** https://ably.com/docs
- **Standards:** REST/JSON, SSE (fallback), custom WebSocket framing over RFC 6455, AsyncAPI-compatible channels
- **Authentication:** API Key (HMAC-signed token requests), JWT tokens, Ably Token Auth

### Pusher Channels

- **Description:** Hosted WebSocket pub/sub service offering channels, presence, and client events. The most widely adopted entry-level managed WebSocket service with a large open-source ecosystem.
- **API Documentation:** https://pusher.com/docs/channels/
- **Server API Overview:** https://pusher.com/docs/channels/server_api/overview/
- **HTTP API Reference:** https://pusher.com/docs/channels/library_auth_reference/rest-api/
- **SDKs/Libraries:** JavaScript (`pusher-js`), Python, Ruby, PHP, Go, Java, .NET, iOS, Android; community libraries for 20+ languages (https://pusher.com/docs/channels/channels_libraries/libraries/)
- **Developer Guide:** https://pusher.com/docs/channels/getting_started/
- **Standards:** REST/JSON for server-side publishing, RFC 6455 WebSocket for clients, custom channel auth protocol
- **Authentication:** App Key + Secret (HMAC-SHA256 channel auth), user authentication via server-generated auth tokens

### AWS API Gateway (WebSocket APIs)

- **Description:** AWS-managed WebSocket API service with route-based message dispatch to Lambda functions or HTTP integrations. Serverless, consumption-based, and deeply integrated with the AWS ecosystem.
- **API Documentation:** https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html
- **API Reference (V2):** https://docs.aws.amazon.com/apigatewayv2/latest/api-reference/api-reference.html
- **SDKs/Libraries:** AWS SDK for JavaScript (`@aws-sdk/client-apigatewayv2`), Python (boto3), Java, Go, .NET, Ruby
- **Developer Guide:** https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-overview.html
- **Standards:** JSON-based route selection expressions, REST management API, CloudFormation / Terraform resource model
- **Authentication:** IAM authorisation, Lambda authorisers (custom JWT validation), Cognito User Pools

### Cloudflare Durable Objects

- **Description:** Strongly-consistent stateful edge primitives that serve as a coordination layer for WebSocket connections at Cloudflare's global edge. Each Durable Object is a single-threaded actor with persistent storage; the WebSocket Hibernation API allows connections to persist without billing during idle periods.
- **API Documentation:** https://developers.cloudflare.com/durable-objects/
- **WebSocket Best Practices:** https://developers.cloudflare.com/durable-objects/best-practices/websockets/
- **WebSocket Hibernation Example:** https://developers.cloudflare.com/durable-objects/examples/websocket-hibernation-server/
- **SDKs/Libraries:** Cloudflare Workers SDK (TypeScript/JavaScript only); `wrangler` CLI for deployment
- **Developer Guide:** https://developers.cloudflare.com/workers/
- **Standards:** Web Standard WebSocket API (browser-compatible), Workers Runtime API
- **Authentication:** Cloudflare Access (OIDC/JWT), custom token validation in Worker code

### Liveblocks

- **Description:** Real-time collaboration infrastructure with WebSocket-backed rooms, presence, shared storage (CRDT), comment threads, and notifications. Purpose-built for multiplayer UX in SaaS applications.
- **API Documentation:** https://liveblocks.io/docs/api-reference/
- **Client SDK Reference:** https://liveblocks.io/docs/api-reference/liveblocks-client
- **REST API Reference:** https://liveblocks.io/docs/api-reference/rest-api-endpoints
- **Node SDK Reference:** https://liveblocks.io/docs/api-reference/liveblocks-node
- **SDKs/Libraries:** `@liveblocks/client` (core), `@liveblocks/react` (React hooks), `@liveblocks/node` (server), `@liveblocks/zustand`, `@liveblocks/redux`
- **Developer Guide:** https://liveblocks.io/docs/get-started/
- **Standards:** JSON/REST management API, WebSocket transport (RFC 6455), CRDT-backed storage (Yjs-compatible)
- **Authentication:** Liveblocks secret key + server-generated access tokens (JWT-based)

### Socket.IO

- **Description:** Open-source event-driven WebSocket framework with automatic HTTP fallback (long-polling), room-based broadcasting, binary support, and horizontal scaling via an adapter layer (Redis, Postgres). The most widely deployed self-hosted WebSocket library.
- **API Documentation:** https://socket.io/docs/v4/
- **Server API Reference:** https://socket.io/docs/v4/server-api/
- **Client API Reference:** https://socket.io/docs/v4/client-api/
- **SDKs/Libraries:** JavaScript (`socket.io` server, `socket.io-client`), Python (`python-socketio`), Java, Swift, Dart/Flutter, .NET; GitHub: https://github.com/socketio/socket.io-client
- **Developer Guide:** https://socket.io/docs/v4/tutorial/
- **Standards:** Custom Socket.IO protocol over RFC 6455 WebSocket or HTTP long-polling; Engine.IO transport layer; can be documented with AsyncAPI
- **Authentication:** Custom middleware-based (JWT validation, session cookies); no built-in auth — integrates with Passport.js, express-session

### PubNub

- **Description:** Global real-time data streaming network with pub/sub, presence, message history, and Functions (serverless edge logic). Over 15 years of operation with points of presence worldwide.
- **API Documentation:** https://www.pubnub.com/docs
- **SDK Reference (JavaScript):** https://www.pubnub.com/docs/sdks/javascript
- **Publish/Subscribe API:** https://www.pubnub.com/docs/sdks/javascript/api-reference/publish-and-subscribe
- **SDKs/Libraries:** JavaScript, Python, Java, Swift, Kotlin, C#, Ruby, Go, PHP, Unreal, Unity, and 50+ SDKs
- **Developer Guide:** https://www.pubnub.com/docs/sdks
- **Standards:** REST/JSON management and history API, WebSocket-backed real-time transport; custom PubNub protocol
- **Authentication:** Publish/Subscribe keys (API key pairs), PubNub Access Manager (PAM) token-based authorisation (JWT-like)

### Gravitee API Management (Event-Native Gateway)

- **Description:** Open-source API management platform with native support for both synchronous (REST, GraphQL) and asynchronous (WebSocket, Kafka, MQTT, SSE) APIs. The most standards-compliant open-source gateway for heterogeneous real-time protocols.
- **API Documentation:** https://documentation.gravitee.io/
- **Event-Native APIM Overview:** https://docs.gravitee.io/apim/3.x/event_native_apim_introduction.html
- **SDKs/Libraries:** Java-based gateway (self-hosted); Terraform provider; Management API (REST); supports client libraries in any language via standard WebSocket
- **Developer Guide:** https://documentation.gravitee.io/platform-overview/
- **Standards:** OpenAPI 3.x, AsyncAPI 3.x, WebSocket (RFC 6455), Kafka, MQTT, SSE, Webhook; GraphQL subscriptions
- **Authentication:** OAuth 2.0, OpenID Connect, JWT, API Key, mTLS; integrates with Keycloak, Auth0, Okta

---

## Notes

**AsyncAPI adoption gap:** Most managed WebSocket services (Ably, Pusher, PubNub) have proprietary channel and message models that are not formally described in AsyncAPI documents. There is an opportunity for an AI-native gateway to auto-generate and publish AsyncAPI specifications from observed traffic patterns, lowering the documentation burden for API producers.

**WebTransport as a future transport:** The W3C WebTransport Working Draft (targeting Recommendation) introduces QUIC-based bidirectional streams and unreliable datagrams in the browser. A well-architected WebSocket gateway should plan a transport abstraction layer that can add WebTransport support without breaking existing WebSocket clients, as browser adoption accelerates.

**Authentication timing:** The WebSocket upgrade handshake is the only opportunity to reject unauthenticated connections with an HTTP-level status code. Post-upgrade authentication (first-message token) is possible but leaves the connection open until the token is validated. Gateway implementations should enforce token validation at the HTTP upgrade stage where possible, per OWASP guidance.

**Rate limiting complexity:** Unlike HTTP APIs where each request is independent, WebSocket connections are long-lived. Rate limiting must account for both connection-level limits (max concurrent connections per user/tenant) and message-level limits (messages per second per connection). This two-dimensional rate limiting is underserved by standard API gateway tooling and represents a significant implementation challenge.
