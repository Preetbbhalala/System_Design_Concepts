<div align="center">

[← Chapter 10: Design A Notification System](10-Design-A-Notification-System.md) · [📚 All chapters](../README.md) · [Chapter 12: Design A Chat System →](12-Design-A-Chat-System.md)

# Chapter 11 · Design A News Feed System

*Publish a post once, show it to thousands of friends, and load everyone's feed in milliseconds.*

![Facebook](https://img.shields.io/badge/Facebook-0866FF?style=flat&logo=facebook&logoColor=white)
![Instagram](https://img.shields.io/badge/Instagram-FF0069?style=flat&logo=instagram&logoColor=white)
![X](https://img.shields.io/badge/X_timeline-000000?style=flat&logo=x&logoColor=white)
![Neo4j](https://img.shields.io/badge/Neo4j-4581C3?style=flat&logo=neo4j&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?style=flat&logo=redis&logoColor=white)

</div>

> **TL;DR:** A news feed is the ever-updating list of posts from people you follow (think Facebook's News Feed, Instagram, the X timeline). The design splits into two flows: **publishing** (save the post, then **fan it out** to friends' feeds) and **retrieval** (read a list of post IDs, then **hydrate** them with user and post data). The key decision is **fanout on write vs fanout on read**; the answer is a **hybrid**: push for normal users, pull for celebrities. Caches do most of the work.

---

## 1️⃣ Scope

| Question | Answer |
|---|---|
| Platforms | Mobile app and web |
| Core features | Publish a post; see friends' posts on the feed |
| Feed order | Reverse chronological (no ranking, to keep it simple) |
| Friends per user | Up to 5,000 |
| Traffic | 10 million daily active users |
| Content | Text, images and videos |

## 2️⃣ High-level design

**Two APIs**, both authenticated with an `auth_token`:

| API | Purpose |
|---|---|
| `POST /v1/me/feed` with `content` | Publish a post |
| `GET /v1/me/feed` | Retrieve my feed |

- **Publishing:** the post is written to the cache and database, then pushed into friends' feeds. A **notification service** tells friends there's something new.
- **Building the feed:** collect friends' posts in reverse chronological order. The **news feed cache** holds the IDs needed to render it.

## 3️⃣ Deep dive: publishing

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch11-publish-dark.svg">
  <img src="../images/ch11-publish-light.svg" width="100%" alt="Feed publishing in detail: the user's POST goes through a load balancer to web servers that enforce authentication and rate limits. The post service saves the post to the post cache and database. The fanout service 1 gets friend IDs from a graph database, 2 filters them with settings from the user cache (for example muted friends), 3 puts the friend list and post ID on a queue, and 4 fanout workers 5 append post_id, user_id entries to the news feed cache.">
</picture>

**Web servers** do more than route: only users with a valid `auth_token` may post, and **rate limiting** caps how many posts one user can make in a period, to stop spam and abuse.

### Fanout: push or pull?
**Fanout** is delivering a post to all of the author's friends.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch11-fanout-models-dark.svg">
  <img src="../images/ch11-fanout-models-light.svg" width="100%" alt="Two fanout models. Fanout on write (push): a new post goes through the fanout service into each friend's precomputed feed, up to 5,000 cache writes, so reading a feed is a single fast lookup. Fanout on read (pull): when a reader opens the app, the posts of every friend are fetched and merged newest first, so no work is wasted on inactive users but reads are slower. The chapter uses a hybrid: push for most users, pull for celebrities.">
</picture>

| | Fanout on write (push) | Fanout on read (pull) |
|---|---|---|
| When the feed is built | When the post is published | When the reader opens the feed |
| 👍 | Real-time delivery; **reading is fast** because the feed is precomputed | No work wasted on **inactive users**; no hotkey problem |
| 👎 | Users with huge friend lists cause slow, heavy fanouts (the **hotkey problem**); precomputing for users who never log in is waste | **Reading is slow** because nothing is precomputed |

**The hybrid:** fast reads matter most, so **push** for the majority of users, but let followers **pull** posts from **celebrities** and other accounts with huge followings on demand. **Consistent hashing** ([Chapter 5](05-Design-Consistent-Hashing.md)) also helps by spreading hot keys more evenly.

### Inside the fanout service
1. **Get friend IDs** from a **graph database**, which suits friend relationships and recommendations.
2. **Filter friends** using the **user cache**: someone who muted the author, or a post shared only with certain people, means some friends are skipped.
3. **Enqueue** the friend list and the new post ID.
4. **Fanout workers** take jobs from the queue…
5. …and append `<post_id, user_id>` entries to the **news feed cache**.

Only **IDs** go into the feed cache; full user and post objects would use far too much memory. The cache also keeps only the **most recent** entries per user, up to a configurable limit: almost nobody scrolls through thousands of posts, so misses are rare.

## 🔍 Deep dive: retrieval

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch11-retrieve-dark.svg">
  <img src="../images/ch11-retrieve-light.svg" width="100%" alt="Feed retrieval in detail: the user's GET goes through a load balancer and web servers to the news feed service, which 1 looks up the feed's post IDs in the news feed cache, 2 hydrates them with full user objects (names, profile pictures) and post objects (text, image links) from the user and post caches, and 3 returns the hydrated feed as JSON. Images and videos are loaded from a CDN.">
</picture>

The request goes load balancer → web servers → **news feed service**, which looks up the feed's **post IDs** in the news feed cache, then pulls in the full **user and post objects** (usernames, profile pictures, text, images) to build a **fully hydrated** feed, returned as **JSON**. Images and videos come straight from a **CDN**.

## 🗄️ Five layers of cache

| Layer | Stores |
|---|---|
| **News feed** | The IDs that make up each user's feed |
| **Content** | Post data, with popular posts in a **hot cache** |
| **Social graph** | Follower and following relationships |
| **Action** | Whether a user liked, replied to, or otherwise acted on a post |
| **Counters** | Like, reply, follower and following counts |

## 4️⃣ Wrap-up talking points

There's no single right design; every company has its own constraints. With time left, talk about scale:

- **Database:** vertical vs horizontal scaling, SQL vs NoSQL, primary-replica replication, read replicas, consistency models, sharding.
- **Everything else:** keep the web tier **stateless**, **cache** as much as possible, support **multiple data centers**, decouple components with **message queues**, and **monitor** key metrics such as peak QPS and feed refresh latency.

## 🔗 Go deeper
- [Meta Transparency Center: how Facebook Feed works](https://transparency.meta.com/features/explaining-ranking/fb-feed/)
- [Neo4j: what is a graph database?](https://neo4j.com/docs/getting-started/graph-database/)
- [Redis documentation](https://redis.io/docs/latest/)

---

<div align="center">

[← Chapter 10](10-Design-A-Notification-System.md) · [📚 All chapters](../README.md) · [Chapter 12 →](12-Design-A-Chat-System.md)

</div>
