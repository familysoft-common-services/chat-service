# Chat
- One to one chat
## DB operations
- Store chat in database for chat history
- Get all chat history
- delete chat 
- update chat
- creat chat
- Once user connect to socket user should able to fetch its chat history via get all chat list


## Chat feature
- Text chat
- sending file or attachment
- sending images
- sending video
- sending zip

## Chat should be scalable and asynchronously scalable
- can follow queue macanisum
- can use redis for requiremnt or lastest chat
- sort chat based on date (lastest chat would be first)
- Pagination chat 


## Most popular product of chat

### Top 10 — Most popular chat apps (global, MAU, 2024–2025 estimates)

1. WhatsApp — 3+ billion MAUs (global; widely reported by WhatsApp / Meta in 2025) — https://blog.whatsapp.com/
2. Facebook Messenger — ~1.2–1.4 billion MAUs (large global reach via Meta / Messenger product pages & reports) — https://www.messenger.com/
3. WeChat — ~1.2–1.3 billion MAUs (China-first super-app with messaging + payments / services) — https://www.wechat.com/
4. Instagram (Direct Messages) — Instagram overall ~1.5–1.9 billion accounts (DMs are heavily used inside Instagram) — https://www.instagram.com/
5. Telegram — ~1.0+ billion MAUs (fast-growing, large groups/channels; founder / press announcements 2024–2025) — https://telegram.org/
6. QQ — ~500–600 million users (Tencent legacy IM; still strong in mainland China) — https://www.tencent.com/
7. Snapchat — ~300+ million MAUs (ephemeral-first messaging and media) — https://www.snapchat.com/
8. Discord — 200+ million MAUs (communities, voice/video + text) — https://discord.com/
9. Viber — hundreds of millions of users (strong in Eastern Europe, parts of Asia) — https://www.viber.com/
10. LINE — ~150–200 million MAUs (popular in Japan, Taiwan, Thailand; sticker economy + services) — https://linecorp.com/

Notes: numbers above are best-effort estimates (MAU = monthly active users where available) based on official company posts, press coverage and public reports from 2024–2025; exact counts can vary depending on the source and reporting period.

### WhatsApp-inspired architecture (short)

WhatsApp is designed for low-latency, high-concurrency messaging at global scale. Key ideas you can apply to a chat-service:

- Lightweight, persistent connections (WebSocket / custom TCP) with efficient framing to keep per-connection memory small.
- Actor-like concurrency (Erlang/BEAM or evented servers) to isolate failures and support millions of simultaneous sessions.
- Partitioning/sharding by user or conversation id so routing and storage targets a small subset of servers.
- Keep servers as stateless as practical: use Redis / in-memory caches for presence and routing, durable stores for history and metadata.
- Offload media to object storage (S3-compatible) + CDN; keep only metadata in DB.
- Use asynchronous queues/pub-sub for delivery, retries, background processing and analytics (Kafka / RabbitMQ / Redis Streams).
- End-to-end encryption for message privacy (Signal protocol) if required by product constraints.
- Monitor, automate, and scale horizontally: health checks, retries, autoscaling groups, and global regions with replication and failover.

This short summary can be expanded into a reference architecture and concrete tech stack if you'd like more detail in a separate document.


