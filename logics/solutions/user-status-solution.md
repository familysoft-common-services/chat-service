# User status in chat application
- Online/Offline
- Last seen
- Typing 


## User Presence System – Complete Technical Documentation
- Overview

- This document explains the complete architecture and reasoning behind implementing a real-time User Online/Offline Presence System for chat applications (like WhatsApp, Instagram, Messenger, Slack, Telegram).

### The document includes:

- Why presence cannot be decided only by app foreground state

- Why socket connection alone is insufficient

- Why presence must be stored in Redis, not SQL DB

- How presence is used by backend services

- How multi-device scenarios work

- How real-time updates reach other users

1️⃣ What Is “User Presence”?

- User presence means showing real-time status such as:

- Online

- Offline

- Last Seen

- Typing…

- Recently active

- In background

- Presence is a volatile, rapidly changing, and extremely high-frequency signal.

- It must always reflect the server’s authoritative truth.

2️⃣ Why You Cannot Rely Only on App Foreground/Background

- Mobile OS behavior makes foreground/background detection unreliable.
- The app can go background due to:

- User pressing home button

- Notification appearing

- OS killing the app

- Device going to sleep

- Low memory killing background tasks

- Network drops

### Therefore:
❌ Foreground detection ≠ true online status
❌ Background state ≠ offline
❌ Client cannot be trusted as source of truth

- Clients can be hacked or modified → server cannot trust client-only presence updates.

3️⃣ Why You Cannot Use Only WebSocket Connection as Presence Indicator

- A WebSocket connection may stay open even if:
- App is minimized
- Screen is locked
- Network has changed
- OS throttles traffic
- App runs in background service
- Also:
- There may be multiple devices
- Edge networks cause delayed socket disconnect

Thus:

❌ “Socket open = online” is false
❌ “Socket closed = offline” is not guaranteed

- Socket events must be combined with:
- Heartbeat
- App lifecycle events
- Server-side timeout
- TTL-based presence expiry

4️⃣ Why Backend MUST Store Presence (Not Client)
- Mobile clients cannot be trusted because:
- They can lie (rooted devices, hacks)
- They may not send disconnect events
- They may lose connectivity suddenly
- They may freeze in the background
- Backend is source of truth for all users:
- Other users need your presence
- Chat delivery depends on it
- Typing updates rely on backend
- Multi-device sync requires centralization

Thus:

👉 Only server can maintain accurate presence data.
5️⃣ Why You Should NOT Store Presence in SQL Database

### Presence changes extremely frequently:
- App opens
- App moves to background
- Screen locks
- Socket reconnects
- Device switches networks
- Heartbeat updates
- A single user may generate 20–50 presence events per minute.
- With thousands of users → millions of DB writes per hour.

### SQL database cannot handle this:

❌ Too many writes
❌ Disk I/O will spike
❌ Locking issues
❌ Slow reads
❌ Heavy indexing
❌ Expensive storage
❌ Stale online status
- Presence is temporary, not permanent → SQL is wrong place.

6️⃣ Why Redis (In-Memory Store) Is the BEST Choice

- Redis is built for real-time, volatile, high-speed operations.
✔️ Super-Fast (Microseconds)
- Real-time presence requires millisecond response → Redis perfect.
✔️ Auto-Expire (TTL) Support
- Example:
- SET user:123:online true EX 30

- If heartbeats stop → Redis auto-expires → user becomes offline.

- SQL cannot do this.
✔️ No Heavy Storage
- Redis holds only current presence, not history.
✔️ High Scalability

### Redis supports:
- clustering
- replication
- sharding
- Chat apps scale to millions of users using Redis.

✔️ Prevents Load on Main DB
- Main DB stays free for:
- messages
- chats
- posts
- transactions
- user accounts

Presence = fast ephemeral data → store in Redis.

7️⃣ Complete Real-Time Presence Architecture
Mobile App → WebSocket → Backend Gateway → Redis Presence Store → Other Users

### Steps:

- App connects to WebSocket → backend marks user as online
- App sends heartbeat every 20–30 sec → refresh TTL
- If app goes background → client sends “background” event
- Backend updates Redis and notifies contacts
- If user closes app/disconnects → TTL expires → user becomes offline
- Backend broadcast presence to:
- chat partner
- contact lists
- group members

8️⃣ What Gets Stored in Redis?

Example record:

user:123:presence = {
    online: true,
    last_seen: 1732039200,
    device: "android",
    socket_id: "xyz123",
    ttl: 30
}


Values auto-expire after TTL → no cleanup needed.

9️⃣ Why a Central Presence System Is Needed
Business needs:

"Online now" indicator

"Last seen recently"

"Typing..."

“Offline 2 minutes ago”

- Delivery logic (should push notification or show real-time?)
- Multi-device sync
- Real-time chat experience
- Without centralized presence:
- Users see wrong online status
- Delays occur
- Notifications behave incorrectly
- Chat feels unreliable

Presence is critical to messaging UX.

🔟 Example Use Cases Covered by This System
✔️ WhatsApp-like online indicator
✔️ Show "typing..."
✔️ Hide/Show last seen depending on privacy setting
✔️ Detect if user is active on mobile or web
✔️ Multi-device presence
✔️ Server decides whether to send push notifications
✔️ Chat delivery optimizations
✔️ Group activity indicators

All of this requires presence to be stored in Redis, NOT the SQL database.

🏁 Summary — Key Points
❌ NOT Recommended
- Do not store online/offline in SQL DB
- Do not rely only on foreground detection
- Do not rely only on Socket connection
- Do not store presence on device

✔️ Recommended (Industry Standard)
- Feature	Best Practice
- Presence storage	Redis
- TTL	20–60 seconds
- Updates	WebSocket + Heartbeat
- Offline detection	TTL expiration
- Multi-device	One Redis record per device
- Notify others	WebSocket broadcast

### This is exactly how:
- WhatsApp
- Instagram
- Messenger
- Slack
- Telegram
- Discord

implement presence tracking.