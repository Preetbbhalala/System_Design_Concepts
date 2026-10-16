<div align="center">

[← Chapter 15: Design Google Drive](15-Design-Google-Drive.md) · [📚 All chapters](../README.md)

# Chapter 16 · The Learning Continues

*The book ends with a reading list. This is a curated, link-checked version, grouped by the chapter each item deepens.*

</div>

> **TL;DR:** Good system design comes from years of accumulated knowledge, and the shortcut is to **study real architectures**. Read them for two things: the **principles they share** (partitioning, replication, caching, queues, trade-offs) and the **technologies underneath**, asking what problem each one solves. The book's own list is mostly old `goo.gl` short links; below are direct links to the papers, talks and engineering blogs, plus a simple loop for practicing with them.

---

## 🧭 How to read an architecture write-up

Run every article through the same [4-step framework](03-A-Framework-For-System-Design-Interviews.md) you use in interviews:

| Ask | Why |
|---|---|
| What problem, at what **scale**? | Numbers explain the design ([Chapter 2](02-Back-Of-The-Envelope-Estimation.md)) |
| What was the **bottleneck**? | Most write-ups exist because something broke |
| Which **trade-off** did they pick, and what did they give up? | Consistency vs availability, latency vs cost… |
| Which **building blocks** show up? | Cache, CDN, queue, sharding, consistent hashing, replication |
| What would **you** have done? | Turns reading into practice |

## 📄 Papers and deep dives, by chapter

| Pairs with | Read | What it teaches |
|---|---|---|
| [Ch 1](01-Scale-From-Zero-To-Millions-Of-Users.md) · caching | [Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) | Running a cache tier for billions of requests per second; invalidation and regional consistency |
| [Ch 4](04-Design-A-Rate-Limiter.md) · rate limiting | [Stripe: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters) | The four kinds of limiters Stripe runs in production |
| [Ch 5](05-Design-Consistent-Hashing.md) · hashing | [Maglev: Google's software load balancer](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) | Consistent hashing inside a high-throughput network load balancer |
| [Ch 6](06-Design-A-Key-Value-Store.md) · KV stores | [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) | The blueprint for Chapter 6: rings, quorums, vector clocks, hinted handoff, Merkle trees |
| [Ch 6](06-Design-A-Key-Value-Store.md) · KV stores | [Bigtable](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/) · [Cassandra](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf) | Memtables, SSTables and wide-column data models |
| [Ch 6](06-Design-A-Key-Value-Store.md) · consensus | [Raft: In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) | How replicas agree on one log, the strongly consistent alternative to Dynamo |
| [Ch 7](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md) · IDs | [Announcing Snowflake](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake) · [Sharding & IDs at Instagram](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c) | Two real 64-bit, time-sortable ID schemes |
| [Ch 11](11-Design-A-News-Feed-System.md) · feeds | [Timelines at Scale (Twitter, QCon 2013)](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/) | Fanout on write in production, and the celebrity problem |
| [Ch 11](11-Design-A-News-Feed-System.md) · social graph | [TAO: Facebook's Distributed Data Store for the Social Graph](https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson) | A graph-aware cache in front of MySQL |
| [Ch 12](12-Design-A-Chat-System.md) · chat | [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Cassandra to ScyllaDB, hot partitions, and request coalescing |
| [Ch 12](12-Design-A-Chat-System.md) · coordination | [ZooKeeper: Wait-free Coordination for Internet-scale Systems](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf) | The service-discovery and coordination workhorse |
| [Ch 13](13-Design-A-Search-Autocomplete-System.md) · batch data | [MapReduce: Simplified Data Processing on Large Clusters](https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/) | The offline aggregation model behind weekly trie rebuilds |
| [Ch 14](14-Design-YouTube.md) · video | [SVE: Distributed Video Processing at Facebook Scale](https://www.cs.princeton.edu/~wlloyd/papers/sve-sosp17.pdf) | The DAG-based transcoding engine Chapter 14 borrows |
| [Ch 14](14-Design-YouTube.md) · photos | [Finding a Needle in Haystack: Facebook's Photo Storage](https://www.usenix.org/conference/osdi10/finding-needle-haystack-facebooks-photo-storage) | Why billions of small files need a custom blob store |
| [Ch 15](15-Design-Google-Drive.md) · storage | [The Google File System](https://research.google/pubs/the-google-file-system/) | Chunked, replicated storage on commodity machines |
| [Ch 15](15-Design-Google-Drive.md) · sync | [Differential Synchronization](https://neil.fraser.name/writing/sync/) | Keeping copies of a document in sync while people edit |

## 🎥 Real-world architecture stories

| Company | Story | Lesson |
|---|---|---|
| Dropbox | [How We've Scaled Dropbox (talk)](https://www.youtube.com/watch?v=PE4gwstWhmc) | Block storage, metadata, and long-poll notifications from the people who built them |
| LinkedIn | [A Brief History of Scaling LinkedIn](https://engineering.linkedin.com/architecture/brief-history-scaling-linkedin) | One monolith to hundreds of services, step by step |
| WhatsApp | [The WhatsApp Architecture Facebook Bought for $19 Billion](https://highscalability.com/the-whatsapp-architecture-facebook-bought-for-19-billion/) | Tiny team, Erlang, and millions of connections per server |
| Uber | [How Uber Scales Their Real-Time Market Platform](https://highscalability.com/how-uber-scales-their-real-time-market-platform/) | Matching supply and demand with geospatial sharding |
| Netflix | [Content Popularity for Open Connect](https://netflixtechblog.com/content-popularity-for-open-connect-b86d56f613b) | Using popularity to decide what lives on the CDN |
| Slack | [Flannel: an application-level edge cache](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/) | Caching near users to make clients load fast |

## 📰 Engineering blogs worth following

If you're interviewing somewhere, read that company's blog first: you'll learn what they build with and what problems they care about.

| | | |
|---|---|---|
| [Netflix TechBlog](https://netflixtechblog.com/) | [Meta Engineering](https://engineering.fb.com/) | [Uber Engineering](https://www.uber.com/blog/engineering/) |
| [Airbnb Engineering](https://medium.com/airbnb-engineering) | [Discord Engineering](https://discord.com/category/engineering) | [Slack Engineering](https://slack.engineering/) |
| [Stripe Engineering](https://stripe.dev/blog/topic/engineering) | [Cloudflare Blog](https://blog.cloudflare.com/) | [GitHub Engineering](https://github.blog/engineering/) |
| [Dropbox Tech](https://dropbox.tech/) | [Pinterest Engineering](https://medium.com/pinterest-engineering) | [Spotify Engineering](https://engineering.atspotify.com/) |
| [Lyft Engineering](https://eng.lyft.com/) | [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/) | [High Scalability](https://highscalability.com/) |

## 📚 Books and courses

- [Designing Data-Intensive Applications](https://dataintensive.net/) by Martin Kleppmann: the theory behind almost every chapter here.
- [ByteByteGo](https://blog.bytebytego.com/): Alex Xu's newsletter, continuing where this book stops (there's also a Volume 2 of the book).
- [The System Design Primer](https://github.com/donnemartin/system-design-primer): a free, community-built companion.
- [Google's SRE books](https://sre.google/books/), free online: running systems reliably once they're built.
- [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.824/): lectures and labs (you build Raft yourself).

## 🔁 A practice loop

1. Pick a product you use (a ride-hailing app, a code host, a game lobby).
2. Write the **requirements** and the **estimates** as if you were in the interview.
3. Sketch the **high-level design** and pick **two components** to go deep on.
4. Find the company's own write-up or talk, and **compare**: what did they do differently, and why?
5. Add what you learned to your notes, or to this repo.

---

<div align="center">

[← Chapter 15](15-Design-Google-Drive.md) · [📚 All chapters](../README.md)

</div>
