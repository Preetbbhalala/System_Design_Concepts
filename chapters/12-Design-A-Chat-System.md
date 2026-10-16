<div align="center">

[← Chapter 11: Design A News Feed System](11-Design-A-News-Feed-System.md) · [📚 All chapters](../README.md) · [Chapter 13: Design A Search Autocomplete System →](13-Design-A-Search-Autocomplete-System.md)

# Chapter 12 · Design A Chat System

*Real-time 1-on-1 and group messaging for 50 million daily users, with "online" dots that don't flicker.*

![Messenger](https://img.shields.io/badge/Messenger-0866FF?style=flat&logo=messenger&logoColor=white)
![WhatsApp](https://img.shields.io/badge/WhatsApp-25D366?style=flat&logo=whatsapp&logoColor=white)
![WeChat](https://img.shields.io/badge/WeChat-07C160?style=flat&logo=wechat&logoColor=white)
![Discord](https://img.shields.io/badge/Discord-5865F2?style=flat&logo=discord&logoColor=white)
![Apache HBase](https://img.shields.io/badge/Apache_HBase-BE160C?style=flat&logo=apachehbase&logoColor=white)
![Apache Cassandra](https://img.shields.io/badge/Apache_Cassandra-1287B1?style=flat&logo=apachecassandra&logoColor=white)
![ZooKeeper](https://img.shields.io/badge/Apache_ZooKeeper-6e7681?style=flat)

</div>

> **TL;DR:** Clients never talk to each other directly; they keep a **WebSocket** open to a **chat server**. Messages get an **ordered ID**, go through a per-recipient **sync queue** (an inbox), are saved in a **key-value store**, and are delivered over the recipient's WebSocket or as a **push notification** if they're offline. **Service discovery** (ZooKeeper) assigns chat servers, devices **sync** by remembering the last message ID they've seen, and **presence** uses **heartbeats** so a flaky connection doesn't make your status blink.

---

## 1️⃣ Scope

| Question | Answer |
|---|---|
| Chat types | **1-on-1** and **group** (up to 100 members) |
| Platforms | Mobile and web; one account can be logged in on **several devices** |
| Scale | **50 million DAU** |
| Features | 1-on-1 chat with low latency, small group chat, **online presence**, **push notifications** |
| Content | Text only, under 100,000 characters |
| End-to-end encryption | Not now (a wrap-up topic) |
| History | Kept **forever** |

Pin this down early: a 1-on-1 messenger (WhatsApp, Messenger, WeChat), an office chat built around groups (Slack), and a gaming chat with huge groups and voice (Discord) are very different designs.

## 2️⃣ How does the server push messages?

The chat service must **receive** messages, **relay** them to the right recipients, and **hold** them for recipients who are offline. Sending is easy: plain **HTTP with keep-alive** works (Facebook's chat started that way). **Receiving** is the hard part, because HTTP is always started by the client.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch12-protocols-dark.svg">
  <img src="../images/ch12-protocols-light.svg" width="100%" alt="Three ways for a server to push chat messages, as sequence diagrams. Polling: the client asks on a timer and usually hears nothing. Long polling: the server holds each request open until a message arrives or a timeout fires, then the client asks again immediately. WebSocket: one HTTP request is upgraded (101 Switching Protocols) into a persistent connection that carries messages in both directions.">
</picture>

| Technique | Verdict |
|---|---|
| **Polling** | Wasteful: most answers are "nothing", and each costs server resources |
| **Long polling** | Better, but the sender's and receiver's chat servers may differ (stateless HTTP + round-robin), the server can't tell when a client has disconnected, and quiet users still reconnect after every timeout |
| **WebSocket** ✅ | Client-initiated, starts as HTTP and is **upgraded**; then **persistent and two-way**. It uses ports 80/443, so it gets through most firewalls |

Use WebSocket in **both directions**, for outgoing as well as incoming messages; it keeps client and server simple. The price: the servers must manage **many long-lived connections** efficiently. Everything else (sign-up, login, profile) stays ordinary **HTTP**.

## 🏗️ High-level design

| Kind | What | Notes |
|---|---|---|
| **Stateless** | API servers for login, sign-up, profiles… behind a load balancer | Can often be bought or reused. **Service discovery** hands each client the host name of a chat server to use. |
| **Stateful** | **Chat servers** | Each client holds a persistent connection and stays on its chat server while it's healthy; service discovery keeps servers from being overloaded |
| **Third-party** | **Push notifications** | Tells users about new messages when the app isn't running ([Chapter 10](10-Design-A-Notification-System.md)) |

**Could one server do it?** In theory, yes: 1 million concurrent connections × about 10 KB each ≈ **10 GB of memory**. But a single server is a **single point of failure**, and proposing it is a red flag. Starting there is fine; just say it's only the starting point.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch12-architecture-dark.svg">
  <img src="../images/ch12-architecture-light.svg" width="100%" alt="One-on-one message flow. 1 User A sends a message over WebSocket to chat server 1. 2 The server gets a message ID from the ID generator. 3 It puts the message on the message sync queue (B's inbox). 4 The message is stored in the key-value store. 5a If User B is online, it goes to chat server 2, which B is connected to; 5b if B is offline, push notification servers alert B. 6 Chat server 2 delivers it to User B over WebSocket. Presence servers, API servers and service discovery (ZooKeeper) complete the system.">
</picture>

**The 1-on-1 flow:** ① A sends to chat server 1 → ② it gets a **message ID** → ③ puts the message on the **sync queue** → ④ the message is stored in the **KV store** → ⑤a if B is online, it goes to chat server 2 where B is connected, or ⑤b if B is offline, the **push servers** notify B → ⑥ chat server 2 delivers it over B's WebSocket.

### Storage
- **Generic data** (profiles, settings, friend lists) → a **relational database** with replication and sharding.
- **Chat history** is a different beast: the volume is enormous (Messenger and WhatsApp together handled **60 billion messages a day** in 2016), mostly **recent** chats are read, yet search, mentions and jump-to-message need **random access**, and 1-on-1 chat reads and writes about **1:1**.

That points to a **key-value store**: it scales horizontally, has very low latency, and copes with the **long tail** of old data, where a relational database's growing indexes make random access expensive. It's also proven: **Facebook Messenger uses HBase** and **Discord uses Cassandra**.

### Data model

```text
message                        (1-on-1)        group_message                  (group)
-----------------------------                  -----------------------------------
message_id    bigint   PK                      channel_id    bigint   PK (partition key)
message_from  bigint                           message_id    bigint   PK
message_to    bigint                           user_id       bigint
content       text                             content       text
created_at    timestamp                        created_at    timestamp
```

**Ordering comes from `message_id`, not `created_at`**: two messages can share a timestamp. In a group, every query targets a single channel, so `channel_id` serves as the **partition key**.

**Generating message IDs** (they must be unique and sortable by time):
- `auto_increment` — not available in most NoSQL stores.
- A **global 64-bit generator** like Snowflake ([Chapter 7](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md)).
- A **local sequence per channel** — enough, since order only matters within one conversation, and easier to build.

## 3️⃣ Deep dive

### Service discovery
**Apache ZooKeeper** registers every chat server and recommends the best one for each client, using criteria such as **location** and **capacity**. On login: the load balancer sends the request to the API servers → after authentication, service discovery picks a chat server (say, server 2) and returns it → the client opens a **WebSocket** to that server.

### Syncing several devices
Each device remembers `cur_max_message_id`, the newest message ID it has. A message is new for that device if it's **addressed to the logged-in user** and its **ID exceeds** `cur_max_message_id`. Each device then pulls whatever it's missing out of the KV store on its own.

### Small group chat

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch12-group-dark.svg">
  <img src="../images/ch12-group-light.svg" width="100%" alt="Small group chat with per-member inboxes (message sync queues). Sending: when User A posts in a group of three, the chat server copies the message into B's inbox and C's inbox. Receiving: B's single inbox collects messages from many senders (A, C, D), so B only ever reads its own inbox.">
</picture>

Each member has an **inbox** (message sync queue), and a group message is **copied into every other member's inbox**. Clients stay simple (check one inbox), and the copies are cheap when groups are small. WeChat works this way with groups capped at 500 members; for very large groups, a copy per member is too expensive.

### Online presence
**Presence servers** track status over WebSocket:
- **Login:** status becomes online, and `last_active_at` is saved in the KV store.
- **Logout:** status becomes offline.
- **Disconnects:** marking users offline on every dropped connection would make the indicator flicker (think of a train going through tunnels). Instead, clients send a **heartbeat** every few seconds, and a user is offline only after **no heartbeat for x seconds**:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch12-heartbeat-dark.svg">
  <img src="../images/ch12-heartbeat-light.svg" width="100%" alt="Heartbeat-based presence with a 30-second threshold. Top: the client sends a heartbeat every 5 seconds, loses its connection from about 12 to 23 seconds, then resumes; it never goes 30 seconds without a heartbeat, so the status stays online. Bottom: heartbeats stop after 10 seconds, and at 40 seconds (30 seconds later) the status changes to offline.">
</picture>

**Telling friends:** presence uses **publish-subscribe** with a channel per friend pair. When A's status changes, the event goes out on one channel per friend (A↔B, A↔C, A↔D), and each friend listens on theirs. That's fine for small groups, but in a 100,000-member group every status change would mean 100,000 events. There, fetch presence only when a user **opens the group** or **refreshes** the friend list.

## 4️⃣ Wrap-up talking points

- **Media messages:** photos and videos are far bigger than text, so discuss compression, cloud storage and thumbnails.
- **End-to-end encryption:** only sender and recipient can read messages (as in WhatsApp).
- **Client-side caching** of messages to cut data transfer.
- **Faster load times:** Slack built a geographically distributed edge cache for users, channels and more.
- **Error handling:** if a chat server dies, service discovery assigns clients a new one; **retries and queues** resend failed messages.

## 🔗 Go deeper
- [How Discord Stores Billions of Messages](https://discord.com/blog/how-discord-stores-billions-of-messages)
- [Flannel: An Application-Level Edge Cache to Make Slack Scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/)
- [MDN: The WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) · [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [Apache ZooKeeper](https://zookeeper.apache.org/)
- [End-to-end encryption, explained](https://en.wikipedia.org/wiki/End-to-end_encryption)

---

<div align="center">

[← Chapter 11](11-Design-A-News-Feed-System.md) · [📚 All chapters](../README.md) · [Chapter 13 →](13-Design-A-Search-Autocomplete-System.md)

</div>
