WhatsApp-inspired Reference Architecture (for chat-service)

This document sketches a practical, production-ready reference architecture inspired by WhatsApp's design principles — suitable for a mid-to-large scale chat service.

Goals
- Low latency delivery
- High concurrency (millions of online sockets)
- Durability for message history
- Efficient media storage and delivery
- Fault isolation and easy horizontal scaling

High-level components

1) Client layer
- Web (WebSocket / Socket.IO), iOS and Android (persistent socket), and a thin web client.
- Use short-lived push notifications (APNs / FCM) when clients are offline.

2) Edge & API tier
- Ingress gateways / load balancers: NGINX / Envoy to handle TLS termination and sticky/proxy logic.
- API Gateway for REST endpoints (auth, profile, message search) — rate limiting, auth and request validation.

3) Real-time connection and messaging tier
- Stateless frontends accept socket connections and route events to back-end processing.
- Worker processes (actor model) per shard/partition: Erlang/Elixir or Go with Goroutines / Java with Vert.x.
- Presence service in Redis for fast presence checks and routing table.

4) Message routing, delivery & persistence
- Partition messages by conversation id or user id across shards (consistent hashing) so a message is handled by a small subset of nodes.
- Use Kafka (or Redis Streams) for durable, ordered message streams per partition; consumers deliver messages to the correct connected sessions and persist to DB.
- Use a fast write-optimized store (Cassandra / ScyllaDB / Dynamo) for message history; Postgres for metadata, indexing and complex queries.

5) Media storage
- Store files (images, videos, attachments) in object storage (S3-compatible) and serve via CDN.
- Keep only metadata and references in DB; support streaming transfer for large files and background transcoding.

6) Background & supporting systems
- Worker queues for notifications, push, analytics, heavy processing.
- Search indexing (Elasticsearch) for message search and discovery.
- Audit/logging, monitoring (Prometheus + Grafana), tracing (Jaeger/OpenTelemetry), and crash reporting.

7) Security
- TLS everywhere; rotate certs centrally. Optional end-to-end encryption using Signal-like protocols.
- Rate limiting, per-user quotas, abuse detection and automated throttling.

Example tech stack (practical)
- Real-time servers: Elixir (Phoenix Channels) or Go + Gorilla WebSocket / Cloud-native scale-out.
- Presence/cache: Redis Cluster
- Messaging stream: Kafka (or Redis Streams for simpler setups)
- Message store: Cassandra / ScyllaDB (for append-heavy history) + Postgres for metadata
- Object storage: AWS S3, MinIO for on-prem; CDN: CloudFront / Fastly
- Push notifications: FCM + APNs
- Load balancer / proxy: Envoy / NGINX
- Observability: Prometheus, Grafana, Jaeger, Sentry

Operational guidance — scaling & resilience
- Shard by conversation or user id for predictable load distribution.
- Keep frontends stateless; route sockets using membership + consistent hash ring.
- Use replication + anti-entropy for message store; prefer tunable-consistency stores for high throughput.
- Rate-limit writes and use back-pressure (queues) to prevent overload.

Start small, scale as needed
- Local dev: Postgres + MinIO + Redis + simple Kafka (or Redis Streams). WebSockets using socket.io or Phoenix Channels.
- Production: migrate streams to Kafka, message store to Cassandra/Scylla, global deployments with regional read replicas.

---

## WhatsApp's actual tech stack (known/reported)

Based on public information, WhatsApp uses or has used the following technologies:

### Core messaging & concurrency
- **Erlang/OTP (BEAM VM)**: WhatsApp was famously built on Erlang for its lightweight process model, fault tolerance, and massive concurrency. This allowed WhatsApp to handle millions of concurrent connections per server with minimal resource usage.
- **Custom protocol**: Initially based on XMPP, WhatsApp evolved to use a highly optimized custom binary protocol to minimize bandwidth and latency.
- **FreeBSD**: WhatsApp historically ran on FreeBSD servers for stability and performance tuning.

### Encryption & security
- **Signal Protocol**: End-to-end encryption for all messages, calls, and media using the Signal Protocol (formerly Axolotl/TextSecure).
- **Noise Protocol Framework**: Used for establishing secure connections and key exchanges.

### Data storage
- **Mnesia**: Erlang's built-in distributed database was used initially for metadata and routing tables.
- **MySQL/MariaDB**: Used for persistent storage and user metadata (post-Facebook acquisition, likely integrated with Facebook's infrastructure).
- **Cassandra**: For distributed, high-write message history storage at scale (part of Meta infrastructure).
- **RocksDB**: Fast key-value storage for local caching and state management.

### Media & content delivery
- **Custom object storage**: Media files stored in distributed object storage systems.
- **CDN**: Facebook/Meta CDN infrastructure for global media delivery.

### Infrastructure & operations
- **Meta's infrastructure**: After acquisition, WhatsApp integrated with Facebook/Meta's global infrastructure including:
  - **TAO (The Associations and Objects)**: Facebook's distributed data store
  - **Scribe**: Log aggregation system
  - **ODS (Operational Data Store)**: Time-series metrics
  - **Thrift**: RPC framework for service communication
- **Custom load balancing**: Optimized routing and connection management
- **Push notifications**: APNs (Apple), FCM (Google), and custom solutions

### Programming languages
- **Erlang**: Core messaging server and concurrency
- **C/C++**: Performance-critical components and client libraries
- **Objective-C/Swift**: iOS client
- **Java/Kotlin**: Android client
- **JavaScript/React Native**: Some UI components

### Monitoring & operations
- **Custom monitoring**: Built-in Erlang monitoring tools
- **Meta's observability stack**: Scuba (analytics), ODS (metrics), and custom alerting systems
- **Automated failover**: Built-in Erlang supervision trees and custom orchestration

### Key architectural decisions
- **Minimal server-side storage**: Messages deleted from servers after delivery to reduce storage costs and privacy exposure
- **Per-server capacity**: At peak, WhatsApp achieved ~3 million concurrent connections per server using Erlang
- **Small team philosophy**: WhatsApp famously operated with a very small engineering team (~50 engineers) serving hundreds of millions of users before acquisition
- **No ads, no tracking**: Product philosophy of privacy-first design

### Evolution post-Meta acquisition (2014+)
- Integrated with Meta's infrastructure for scaling, reliability, and compliance
- Maintained end-to-end encryption commitment
- Expanded to support WhatsApp Business, Payments, and other features
- Migrated some components to Meta's shared services while keeping core messaging on Erlang

---

## Architecture Diagram

A PlantUML diagram has been created at `docs/architecture-diagram.puml` showing the complete architecture with:
- Client layer (Web, iOS, Android)
- Edge tier (Load Balancer, API Gateway)
- Real-time messaging tier (WebSocket servers, Router, Presence)
- Message processing (Kafka, Delivery, Persistence)
- Storage layer (Cassandra, PostgreSQL, Redis, S3/CDN)
- Supporting systems (Workers, Search, Monitoring, Tracing, Push)

### How to view the diagram

**Option 1: VS Code (easiest)**
1. Install the "PlantUML" extension by jebbs
2. Open `docs/architecture-diagram.puml`
3. Press `Alt+D` (Windows/Linux) or `Option+D` (Mac) to preview

**Option 2: Online renderer**
- Visit https://www.plantuml.com/plantuml/uml/
- Copy/paste the contents of `architecture-diagram.puml`
- Click "Submit" to generate PNG/SVG

**Option 3: Command line**
```bash
# Install PlantUML (requires Java)
# Ubuntu/Debian
sudo apt-get install plantuml

# macOS
brew install plantuml

# Generate PNG
plantuml docs/architecture-diagram.puml

# Generate SVG
plantuml -tsvg docs/architecture-diagram.puml
```

The diagram shows data flows, component interactions, and key architectural patterns like sharding, caching, and async processing.
