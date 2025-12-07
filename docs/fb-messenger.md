# Facebook Messenger Architecture

Facebook Messenger is one of the world's largest messaging platforms with over 1.2 billion monthly active users. This document explores Messenger's architecture, technology stack, and design principles.

## Overview

Messenger evolved from Facebook Chat (2008) into a standalone app (2011) and has become a comprehensive communication platform supporting:
- Text messaging
- Voice and video calls
- Group chats and video rooms
- Stories and media sharing
- Payments and commerce
- Chatbots and platform integrations
- End-to-end encryption (Secret Conversations)

## Core Architecture Principles

### 1. Built on Facebook/Meta Infrastructure
Unlike WhatsApp's Erlang-based architecture, Messenger is deeply integrated with Facebook's massive infrastructure, leveraging shared services and data stores.

### 2. Multi-Protocol Support
- **MQTT** (Message Queuing Telemetry Transport): Primary protocol for mobile messaging - lightweight, battery-efficient pub/sub protocol
- **HTTP/HTTPS**: Web client and API endpoints
- **WebSocket**: Real-time web client connections
- **Thrift**: Internal RPC between services

### 3. Microservices Architecture
Messenger uses a service-oriented architecture with specialized services for different functions, all communicating via internal Facebook infrastructure.

## Technology Stack

### Client Layer
- **iOS**: Objective-C/Swift with custom networking layer
- **Android**: Java/Kotlin with custom MQTT client
- **Web**: React.js (built by Facebook) with WebSocket connections
- **Desktop**: Electron-based apps (Windows/Mac)

### Messaging Protocol & Transport
- **MQTT over TLS**: Mobile clients use Facebook's custom MQTT implementation called "Mosquitto-based broker"
- **MQTT benefits**:
  - Lightweight binary protocol (reduced bandwidth)
  - Built-in quality of service (QoS) levels
  - Persistent sessions and message queuing
  - Battery-efficient with long-lived connections
  - Works well on unstable networks
- **Zero Protocol**: Facebook's internal RPC framework for service-to-service communication

### Backend Services & Infrastructure

#### Core Messaging Services
- **Message Router**: Routes messages to correct recipient queues
- **Presence Service**: Tracks online/offline status and "active now" indicators
- **Delivery Service**: Ensures message delivery with retry logic
- **Typing Indicator Service**: Real-time "user is typing..." notifications
- **Read Receipt Service**: "Seen" and delivery confirmations

#### Data Storage Layer
- **TAO (The Associations and Objects)**: Facebook's distributed graph database
  - Stores social graph data, relationships, and associations
  - Caches relationships and connections
  - Handles billions of objects and trillions of associations
- **MySQL/MariaDB**: Sharded databases for message persistence
  - Horizontally sharded by user_id
  - Replicated for high availability
- **RocksDB**: Embedded key-value store for local caching and state
- **Memcached/Redis**: Distributed caching layer
- **Haystack**: Facebook's photo storage system for media files
- **f4**: Warm blob storage for older media (cost-optimized)

#### Message Queue & Async Processing
- **Scribe**: Facebook's distributed log aggregation system (similar to Kafka)
- **Iris**: Messenger's message queueing system built on top of MQTT
  - Handles message ordering and delivery guarantees
  - Provides persistence and replay capabilities
- **Streaming infrastructure**: Processes billions of messages daily for analytics

#### Infrastructure Services
- **TAO**: Social graph queries and caching
- **Unicorn**: Social graph serving tier
- **Dragon**: Distributed tracing system
- **ODS (Operational Data Store)**: Time-series metrics and monitoring
- **Scuba**: Real-time data analytics
- **Thrift**: Cross-service RPC framework

### Media & Content Delivery
- **Haystack**: Custom photo storage infrastructure
  - Optimized for serving billions of small images
  - Minimizes disk seeks and metadata overhead
- **CDN**: Distributed globally via Facebook's edge network
- **Video infrastructure**: Custom video encoding, storage, and streaming
- **Voice messages**: Compressed audio stored in blob storage

### Real-Time Features
- **Live video streaming**: Custom infrastructure for Messenger Rooms and video calls
- **WebRTC**: Peer-to-peer video/audio for calls
- **TURN servers**: Relay servers for NAT traversal
- **Media servers**: Process and mix multi-party video calls

### Security & Privacy
- **End-to-End Encryption (Secret Conversations)**:
  - Signal Protocol implementation
  - Optional mode (not default like WhatsApp)
  - Device-specific encryption keys
- **TLS everywhere**: All transport encrypted
- **Content moderation**: ML-based systems for abuse detection
- **Rate limiting**: Per-user and per-endpoint limits

### Platform & Extensions
- **Messenger Platform**: APIs for chatbots and integrations
- **Send/Receive API**: Webhooks for bots
- **Messenger Code**: QR-like codes for easy friend adds
- **Payments API**: Peer-to-peer payments (in select countries)

## Architecture Patterns

### Sharding Strategy
- **User-based sharding**: Data partitioned by user_id for predictable distribution
- **Consistent hashing**: Routes messages to correct shards
- **Cross-shard queries**: Handled by aggregator services when needed

### Caching Strategy
- **Multi-tier caching**:
  - L1: Client-side caching (SQLite on mobile)
  - L2: Server-side Memcached/Redis
  - L3: TAO distributed cache
- **Cache invalidation**: Event-driven updates via pub/sub
- **Write-through cache**: Updates propagated to cache on writes

### Message Delivery Flow

```
Mobile Client
    ↓ (MQTT over TLS)
MQTT Broker / Load Balancer
    ↓
Message Router Service
    ↓
Iris (Message Queue)
    ↓
┌─────────────────────┬──────────────────────┐
│                     │                      │
Delivery Service   Persistence Layer    Analytics Pipeline
    ↓                  ↓                     ↓
Recipient's        MySQL (sharded)        Scribe/Scuba
MQTT Queue         TAO (graph data)       (metrics/logs)
    ↓
Mobile Client
(Push notification if offline)
```

### Scalability Patterns
- **Horizontal scaling**: Add more servers per service tier
- **Stateless services**: Most services are stateless for easy scaling
- **Async processing**: Heavy operations offloaded to background workers
- **Regional deployments**: Data centers globally with cross-region replication
- **Traffic shaping**: Request routing based on load and health

### Reliability & Resilience
- **Multi-region replication**: Data replicated across data centers
- **Automated failover**: Services fail over to healthy instances
- **Circuit breakers**: Prevent cascading failures
- **Bulkheads**: Isolate critical vs non-critical paths
- **Graceful degradation**: Non-essential features disabled under load

## Key Architectural Decisions

### Why MQTT for Mobile?
- **Battery efficiency**: Long-lived connections with heartbeats
- **Bandwidth savings**: Binary protocol, minimal overhead
- **Built-in QoS**: Delivery guarantees at protocol level
- **Offline support**: Message queuing when client disconnected
- **Network resilience**: Handles poor connectivity gracefully

### Why Not End-to-End Encryption by Default?
- **Platform features**: Bots, search, and cloud sync require server access to messages
- **User experience**: Seamless device switching and backup/restore
- **Compliance**: Easier content moderation and legal compliance
- **Optional E2E**: Available via "Secret Conversations" for privacy-sensitive users

### Integration with Facebook Ecosystem
- **Unified identity**: Facebook account-based authentication
- **Social graph**: Leverage existing friend connections
- **Shared infrastructure**: Cost efficiency and operational simplicity
- **Cross-platform features**: Stories, payments, marketplace integrated

## Performance Characteristics

### Message Delivery Latency
- **P50**: < 100ms for online users
- **P99**: < 500ms for online users
- **Offline queuing**: Messages stored up to 30 days

### Scale Metrics
- **1.2B+ monthly active users** (2024)
- **Billions of messages per day**
- **100M+ voice/video calls daily**
- **Global presence**: 150+ countries

### Throughput
- **Message ingestion**: Millions of messages per second
- **Concurrent connections**: Hundreds of millions
- **Data replication**: Sub-second cross-region sync for critical data

## Evolution & Future Direction

### Recent Additions
- **Messenger Rooms** (2020): Group video calls up to 50 people
- **Instagram integration** (2020): Cross-app messaging
- **E2E encryption expansion** (planned): Default E2E for all chats
- **Payments expansion**: P2P payments in more countries

### Technology Shifts
- **React Native**: Increasing use for cross-platform mobile development
- **GraphQL**: API evolution for more efficient data fetching
- **AI/ML integration**: Smart replies, translation, content understanding
- **Unified messaging infrastructure**: Convergence with Instagram and WhatsApp backends

## Lessons for Building Chat Services

### From Messenger's Architecture

1. **MQTT is excellent for mobile messaging**
   - Lower battery consumption than WebSockets
   - Better handling of network interruptions
   - Native support for offline queuing

2. **Leverage existing infrastructure**
   - Don't rebuild everything from scratch
   - Use proven distributed systems (Kafka, Cassandra, Redis)
   - Integrate with existing auth and user management

3. **Separate hot and cold data**
   - Recent messages in fast storage (Redis, memory)
   - Older messages in cheaper storage (S3, cold blob storage)
   - Use tiered storage strategies

4. **Make E2E encryption optional (if needed)**
   - Allows richer platform features (search, bots, sync)
   - Provide E2E as opt-in for privacy-conscious users
   - Balance security with functionality

5. **Invest in observability early**
   - Distributed tracing (know where latency comes from)
   - Metrics and alerting (catch issues before users do)
   - Log aggregation (debug production issues quickly)

6. **Design for multi-device support**
   - Sync message state across devices
   - Handle device pairing and security
   - Optimize for different network conditions per device

7. **Build for resilience**
   - Multi-region deployment from day one
   - Graceful degradation under load
   - Automated recovery and failover

## Comparing Messenger vs WhatsApp

| Aspect | Messenger | WhatsApp |
|--------|-----------|----------|
| **Core Language** | C++, Hack, Java | Erlang |
| **Protocol** | MQTT | Custom binary |
| **Encryption** | Optional E2E | Default E2E |
| **Architecture** | Microservices on FB infra | Monolithic Erlang |
| **Identity** | Facebook account | Phone number |
| **Platform** | Rich bot ecosystem | Limited business APIs |
| **Storage** | TAO + MySQL | Mnesia + Cassandra |
| **Philosophy** | Feature-rich, integrated | Minimal, privacy-first |

Both are optimized for different use cases and user expectations.

## References & Further Reading

- **Facebook Engineering Blog**: https://engineering.fb.com/
- **MQTT Protocol**: http://mqtt.org/
- **TAO: Facebook's Distributed Data Store**: https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson
- **Building Mobile-First Infrastructure for Messenger**: https://engineering.fb.com/2014/10/09/production-engineering/building-mobile-first-infrastructure-for-messenger/
- **Iris: MQTT-based Message Queue**: Research papers and tech talks
- **Messenger Platform Documentation**: https://developers.facebook.com/docs/messenger-platform

---

**Want more?** I can create a PlantUML diagram showing Messenger's architecture or dive deeper into specific components like MQTT implementation, TAO, or the E2E encryption system.