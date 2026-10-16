<div align="center">

[📚 All chapters](../README.md) · [Chapter 2: Back-of-the-envelope Estimation →](02-Back-Of-The-Envelope-Estimation.md)

# Chapter 1 · Scale From Zero To Millions Of Users

*Start with one server, then fix one bottleneck at a time until the system serves millions.*

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat&logo=postgresql&logoColor=white)
![Cassandra](https://img.shields.io/badge/Cassandra-1287B1?style=flat&logo=apachecassandra&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?style=flat&logo=redis&logoColor=white)
![Memcached](https://img.shields.io/badge/Memcached-6e7681?style=flat)
![Akamai](https://img.shields.io/badge/Akamai-0096D6?style=flat&logo=akamai&logoColor=white)

</div>

> **TL;DR:** Nobody designs for millions of users on day one. You begin with everything on a single machine and keep adding one building block at a time: a separate database, load balancers, replicas, caches, a CDN, a stateless web tier, more data centers, queues and finally sharding. Each one removes a specific bottleneck or single point of failure. This chapter is the vocabulary for the rest of the book.

---

## 🧭 The scaling journey

| Step | What you add | The problem it solves |
|:---:|---|---|
| 1 | **Single server**: web app, database and cache on one box | The simplest possible start. DNS turns the domain into an IP, and the client talks HTTP to that one server. |
| 2 | **Separate database server** | The web tier and the data tier can now grow independently. |
| 3 | **Load balancer + more web servers** | No single web server can take the site down. Clients only see the balancer's public IP, and the servers talk over private IPs. |
| 4 | **Database replication** (one master, many replicas) | Writes go to the master and reads spread across replicas. That gives more throughput, and the data survives a lost machine. |
| 5 | **Cache tier** | Hot data is served from memory instead of hitting the database every time. |
| 6 | **CDN** | Images, CSS and JS are served from a server near the user instead of from your web tier. |
| 7 | **Stateless web tier** | Session data moves to a shared store, so *any* server can handle *any* request, which makes autoscaling easy. |
| 8 | **Multiple data centers** | geoDNS sends users to the nearest data center, and all traffic can shift to a healthy one during an outage. |
| 9 | **Message queue** | Slow work (like photo processing) happens asynchronously, and producers and consumers scale separately. |
| 10 | **Logging, metrics, automation** | At this size you need to *see* problems and ship safely (CI/CD). |
| 11 | **Sharding** | One database can't hold or serve everything, so the data is split across many database servers. |

## 🏗️ Where you end up

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch01-architecture-dark.svg">
  <img src="../images/ch01-architecture-light.svg" width="100%" alt="Final architecture from chapter 1: users get static files from a CDN and an IP from geoDNS, then call a load balancer in front of stateless web servers. Session state lives in a NoSQL store, a cache sits in front of sharded master/replica databases, a message queue feeds workers, and logging and automation cover everything. The data center is replicated across regions.">
</picture>

A request first resolves the domain through geoDNS, which picks the nearest data center, and loads static files from the CDN. Dynamic calls hit the **load balancer**, which spreads them over identical, **stateless** web servers. Those servers read hot data through the **cache**, send writes to the **master** database and reads to **replicas**, push slow jobs onto a **queue**, and keep session data in a shared **NoSQL** store.

---

## 🧩 Key concepts

### SQL or NoSQL?
Relational databases (MySQL, PostgreSQL, Oracle) store rows in tables and support joins. They've been reliable for decades and are the right default. NoSQL stores come in four families: **key-value, graph, column and document** (e.g., Cassandra, HBase, DynamoDB, CouchDB, Neo4j). Most of them don't support joins. Reach for NoSQL when you need **very low latency**, your data is **unstructured** or has no relations, you only **serialize/deserialize** blobs (JSON, XML, YAML), or you must store a **huge** amount of data.

### Vertical vs horizontal scaling
| | Vertical (scale **up**) | Horizontal (scale **out**) |
|---|---|---|
| How | A bigger machine (more CPU, RAM) | More machines |
| Good | Simple; great while traffic is low | No hard ceiling; built-in redundancy |
| Bad | Hard hardware limit, no failover, expensive at the top end | More moving parts (balancing, consistency) |

### Load balancer + replication = redundancy
- If a web server dies, the load balancer sends its traffic to the others, and you add a replacement to the pool.
- If a **replica** dies, reads go to the other replicas (or briefly to the master).
- If the **master** dies, a replica is **promoted**. In real systems that replica may be missing recent writes, so you may need recovery scripts. Multi-master and circular replication exist but are more complex.

### Cache: five things to decide
1. **When to cache:** data that's read often and changed rarely. Cache memory is volatile, so never treat the cache as the source of truth.
2. **Expiration:** too short and you keep reloading from the database; too long and users see stale data.
3. **Consistency:** the database and the cache aren't updated in one transaction, so they can drift, especially across regions.
4. **Failures:** one cache server is a single point of failure. Use several (across data centers) and over-provision memory.
5. **Eviction:** when the cache is full, something must go. **LRU** is the most common policy; LFU and FIFO also exist.

The pattern used here is **read-through**: check the cache, and on a miss read the database and store the result in the cache.

### CDN
The first request for a file is a *miss*: the CDN fetches it from the **origin** (a web server or object storage like S3) and keeps it until its **TTL** expires. Later requests from nearby users are served straight from the edge. Watch out for:
- **cost** (you pay for transfer, so don't push rarely used assets)
- sensible **expiry times**
- a **fallback** to the origin if the CDN is down
- **invalidation**, either through the vendor's API or by **versioning URLs** such as `image.png?v=2`

### Stateful vs stateless
A *stateful* server remembers a client between requests, so that client must keep hitting the same server (**sticky sessions**). That makes adding, removing or losing servers painful. A *stateless* tier keeps state in a shared store, so requests can go anywhere and the tier can **autoscale**.

### Multiple data centers
The three hard parts: **traffic redirection** (geoDNS), **data synchronization** (replicate data across regions, usually asynchronously, so a failed-over user still finds their data) and **testing and deployment** at every location (automate it).

### Message queue
**Producers** publish messages and **consumers** process them whenever they're ready. Either side can be down without blocking the other. When the queue grows, add workers; when it's empty, remove them.

### Monitoring
Track **host metrics** (CPU, memory, disk I/O), **aggregated metrics** (e.g., the whole database or cache tier) and **business metrics** (revenue, retention, DAU). Centralize logs. Automate build, test and deploy.

### Sharding
Every shard has the **same schema** but holds a **different slice** of the data, chosen by a **shard key**. For example, `user_id % 4` picks one of four databases. Pick a key that spreads data **evenly**. The pain points:
- **Resharding** when a shard fills up or the data is uneven. *Consistent hashing* ([Chapter 5](05-Design-Consistent-Hashing.md)) makes this much cheaper.
- **Hot keys (the "celebrity problem"):** one very popular user can overload one shard. That user may need a dedicated shard.
- **Joins across shards** are hard, so you often **denormalize**.

> 💡 **Scale-up can go surprisingly far:** large cloud database machines offer up to 24 TB of RAM, and in 2013 Stack Overflow served over 10 million monthly unique visitors from a single master database. Scaling up is still a dead end eventually, and riskier (a single point of failure) and pricier at the top.

---

## ✅ Checklist: scaling to millions

- [ ] Keep the web tier **stateless**
- [ ] Build **redundancy into every tier**
- [ ] **Cache** as much as you sensibly can
- [ ] Serve static assets from a **CDN**
- [ ] Support **multiple data centers**
- [ ] **Shard** the data tier
- [ ] Split tiers into **independent services**
- [ ] **Monitor** everything and **automate** deployment

## 🔗 Go deeper
- [Scaling Memcache at Facebook (NSDI '13)](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala): caching at enormous scale and keeping it consistent
- [Active-Active for Multi-Regional Resiliency (Netflix)](https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b): running several data centers at once
- [What it takes to run Stack Overflow](https://nickcraver.com/blog/2013/11/22/what-it-takes-to-run-stack-overflow): how far a few big servers can go

---

<div align="center">

[📚 All chapters](../README.md) · [Chapter 2: Back-of-the-envelope Estimation →](02-Back-Of-The-Envelope-Estimation.md)

</div>
