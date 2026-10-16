<div align="center">

[← Chapter 5: Design Consistent Hashing](05-Design-Consistent-Hashing.md) · [📚 All chapters](../README.md) · [Chapter 7: Design A Unique ID Generator →](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md)

# Chapter 6 · Design A Key-value Store

*Build a Dynamo-style database: two operations, many machines, and every failure case you can think of.*

![Apache Cassandra](https://img.shields.io/badge/Apache_Cassandra-1287B1?style=flat&logo=apachecassandra&logoColor=white)
![Google Bigtable](https://img.shields.io/badge/Google_Bigtable-669DF6?style=flat&logo=googlebigtable&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?style=flat&logo=redis&logoColor=white)
![Amazon Dynamo](https://img.shields.io/badge/Amazon_Dynamo-6e7681?style=flat)
![Memcached](https://img.shields.io/badge/Memcached-6e7681?style=flat)

</div>

> **TL;DR:** The API is just `put(key, value)` and `get(key)`. One server can't hold the data or survive failures, so the store is spread over a ring of nodes. The chapter assembles it from proven building blocks: **consistent hashing** to partition, **replication** for availability, **quorums** to tune consistency, **vector clocks** to detect conflicting writes, **gossip** to detect failures, **hinted handoff** and **Merkle trees** to repair them, and an **LSM-style storage engine** (commit log → memtable → SSTables) on each node. It's essentially a tour of Amazon's Dynamo and Apache Cassandra.

---

## 🎯 What we're building

A key is a unique identifier, either plain text (`last_logged_in_at`) or a hash (`253DDEC4`); short keys are faster. A value is an opaque blob: a string, a list, an object. The store doesn't care what's inside.

| Requirement | Target |
|---|---|
| Item size | Small, under 10 KB per key-value pair |
| Data volume | Big, more than one machine can hold |
| Availability | Responds quickly, **even during failures** |
| Scalability | Grows to very large data sets; servers are added and removed **automatically** with traffic |
| Consistency | **Tunable** per use case |
| Latency | Low |

## 🖥️ Start simple: one server

An **in-memory hash table** is the easiest version and it's very fast, but memory runs out quickly. You can stretch it by **compressing** values and keeping only **hot data in memory** with the rest on disk. Still, a single server hits its limit and is a single point of failure, so the real design is **distributed**.

## ⚖️ The CAP theorem: pick two of three

A distributed system can't give you all three of these at the same time:

| Letter | Property | Meaning |
|:---:|---|---|
| **C** | Consistency | Every client sees the same data at the same moment, whichever node it asks |
| **A** | Availability | Every request gets a response, even when some nodes are down |
| **P** | Partition tolerance | The system keeps working when the network between nodes breaks |

Networks *do* fail, so **P is not optional** in practice and a "CA" system can't really exist. The real choice is what to give up **during a partition**:

- **CP (consistency over availability):** refuse writes, or return errors, until the nodes can agree again. Banks choose this, since a wrong balance is worse than a short outage.
- **AP (availability over consistency):** keep serving reads (possibly stale) and accepting writes, then **sync after the network heals**.

Ask the interviewer which one the product needs. That single answer drives a lot of the design below.

## 🧰 The toolbox at a glance

| Part of the design | Technique | One-line idea |
|---|---|---|
| Spreading data | **Consistent hashing** | Keys and servers on a ring; adding a node moves few keys ([Chapter 5](05-Design-Consistent-Hashing.md)) |
| Mixed hardware | **Virtual nodes** | A bigger server gets more positions on the ring |
| Surviving node loss | **Replication** | Each key lives on N nodes, ideally in different data centers |
| Tuning consistency | **Quorum (N, W, R)** | Choose how many replicas must answer a write or a read |
| Concurrent writes | **Vector clocks** | Version every value; detect when two versions conflict |
| Noticing dead nodes | **Gossip protocol** | Nodes trade heartbeat lists with random peers |
| Short outages | **Sloppy quorum + hinted handoff** | A healthy node stands in, then hands the data back |
| Lost replicas | **Anti-entropy with Merkle trees** | Compare hash trees to sync only what differs |
| Data center loss | **Cross-data-center replication** | Keep copies in other regions |
| Storage on each node | **Commit log, memtable, SSTables, bloom filters** | Fast sequential writes, quick lookups |

---

## 🧱 Partitioning and replication

**Partitioning** uses consistent hashing ([Chapter 5](05-Design-Consistent-Hashing.md)), which gives two things for free: **automatic scaling** (nodes join and leave with little data movement) and **heterogeneity** (give a powerful server more virtual nodes).

**Replication:** after hashing a key onto the ring, walk clockwise and store it on the **first N servers** you meet. With virtual nodes, count only **distinct physical servers**, otherwise two replicas could end up on one machine. For extra safety, put replicas in **different data centers** connected by fast links, since a whole data center can lose power or network.

## 🗳️ Quorum consensus: tunable consistency

One node, the **coordinator**, receives each request and talks to the replicas on the client's behalf.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch06-quorum-dark.svg">
  <img src="../images/ch06-quorum-light.svg" width="100%" alt="Quorum consensus with N = 3 replicas: a write succeeds after W = 2 acknowledgements (s0 and s1) and a read waits for R = 2 replies (s1 and s2). Because W + R > N, the two sets always share a replica (here s1) that has the latest write.">
</picture>

- **N** = number of replicas, **W** = acknowledgements a write must collect, **R** = responses a read must collect.
- The coordinator still sends the write to all N replicas; W only says **how many it waits for** before reporting success.
- W and R trade **latency** against **consistency**:

| Setting | Effect |
|---|---|
| **W + R > N** | Strong consistency: some replica is in both the write set and the read set (typical: N = 3, W = R = 2) |
| R = 1, W = N | Optimized for **fast reads** |
| W = 1, R = N | Optimized for **fast writes** |
| W + R ≤ N | Strong consistency is **not** guaranteed |

## 🔄 Consistency models

- **Strong:** a read always returns the latest write. Usually done by blocking reads or writes until every replica agrees, which hurts availability.
- **Weak:** later reads may not see the latest write.
- **Eventual:** a form of weak consistency where, given enough time, every replica converges.

Dynamo and Cassandra choose **eventual consistency**. They accept concurrent writes and leave it to the **client to reconcile** conflicting versions. That needs versioning.

## 🕰️ Versioning with vector clocks

Every write creates a new, immutable version. A **vector clock** tags each version with a list of `[server: counter]` pairs, and a server increments its own counter when it handles a write.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch06-vector-clock-dark.svg">
  <img src="../images/ch06-vector-clock-light.svg" width="100%" alt="Vector clocks: D1 [Sx:1] is updated through Sx to D2 [Sx:2]. Two clients then update D2 at the same time, through Sy (D3 [Sx:2, Sy:1]) and through Sz (D4 [Sx:2, Sz:1]). Neither clock dominates the other, so they conflict; a client reconciles them and writes D5 [Sx:3, Sy:1, Sz:1] through Sx.">
</picture>

**Reading the clocks:**
- **Ancestor:** if every counter in A is ≤ the matching counter in B, then A is older than B and can be thrown away. Example: `[Sx:1]` → `[Sx:2]`.
- **Conflict (siblings):** if A has a bigger counter somewhere and B has a bigger counter somewhere else, neither came first. Example: `[Sx:2, Sy:1]` vs `[Sx:2, Sz:1]`.

**Costs:** clients now carry conflict-resolution logic, and clocks can grow long. The fix for growth is to **cap the number of pairs** and drop the oldest, which may make reconciliation less precise. Amazon reports this hasn't been a problem in production.

## 🩺 Handling failures

**1. Detecting failures.** One node saying "s2 is down" isn't enough; you want at least two independent sources. Having every node ping every other node (all-to-all) doesn't scale. The **gossip protocol** does:
- Each node keeps a **membership list** of node IDs with **heartbeat counters**.
- Each node periodically **bumps its own counter** and sends its list to a few **random** nodes, which pass it on.
- If a node's counter **hasn't increased for a while**, everyone gradually agrees it's offline.

**2. Temporary failures: sloppy quorum + hinted handoff.** Instead of insisting on the exact N replicas, the system uses the **first W healthy servers** for writes and the **first R healthy servers** for reads on the ring, skipping the dead one. A stand-in keeps the data with a **hint** about where it belongs, and **hands it back** when the original node returns. Availability stays high.

**3. Permanent failures: anti-entropy with Merkle trees.** To resync a replica that lost data, compare the replicas efficiently with a **Merkle (hash) tree**: split the key space into **buckets**, hash every key, hash each bucket, then keep hashing pairs up to a single **root**.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch06-merkle-dark.svg">
  <img src="../images/ch06-merkle-light.svg" width="100%" alt="Merkle trees on two replicas. Keys are grouped into 4 buckets, each bucket is hashed, and pairs of hashes are hashed again up to a root. The roots differ, so the replicas compare children: the left halves match and the right halves differ, which leads straight to bucket 3, the only one that has to be synced.">
</picture>

Equal roots mean the replicas are identical. Otherwise, descend only into the subtrees whose hashes differ. The data transferred is proportional to the **differences**, not the total size. At real scale, a billion keys might be split into a million buckets of about a thousand keys each.

**4. Data center outage.** Power cuts, network failures and natural disasters happen. **Replicate across data centers** so users can still read and write when one is gone.

---

## 🏗️ System architecture

- Clients talk to the store through two APIs: `get(key)` and `put(key, value)`.
- A **coordinator** node sits in front of the replicas and forwards each client request to them.
- Nodes sit on a **consistent hashing ring**.
- The system is **fully decentralized**: adding or moving nodes is automatic.
- Data is **replicated** on multiple nodes.
- There is **no single point of failure**, because every node has the same jobs: serving the client API, detecting failures, resolving conflicts, repairing failures, replicating data, and running the storage engine.

## ✍️ Write path and 📖 read path

The per-node storage engine follows **Cassandra**'s design, a log-structured merge (LSM) approach.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch06-paths-dark.svg">
  <img src="../images/ch06-paths-light.svg" width="100%" alt="Write and read paths. Write: the request is appended to a commit log on disk, then applied to an in-memory memtable, which is flushed to immutable SSTables on disk when full. Read: check the memtable first; on a miss, bloom filters say which SSTables might hold the key, and only those are read.">
</picture>

**Write:** append to the **commit log** (so a crash can't lose it) → update the **memtable** in memory → when it fills up, **flush** it to disk as an **SSTable**, a sorted, immutable file of key-value pairs.

**Read:** look in the **memtable** first; if the key isn't there, ask the **bloom filters** which SSTables *might* contain it (a bloom filter can say "maybe" but never wrongly says "no"), read only those SSTables, and return the result.

## 🗣️ Interview talking points

- Start by asking about the **CAP** choice; it decides whether you lean toward quorums with W + R > N or toward availability with sloppy quorums.
- Say **why** each technique is there; the toolbox table above is your checklist.
- Mention the trade-offs: vector clocks push complexity to the client, and eventual consistency means a read can briefly return stale data.

## 🔗 Go deeper
- [Dynamo: Amazon's Highly Available Key-value Store (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf): the paper this chapter follows most closely
- [Apache Cassandra: architecture overview](https://cassandra.apache.org/doc/latest/cassandra/architecture/overview.html)
- [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/)
- [SSTable and log-structured storage: LevelDB (Ilya Grigorik)](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)
- [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) · [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree)

---

<div align="center">

[← Chapter 5](05-Design-Consistent-Hashing.md) · [📚 All chapters](../README.md) · [Chapter 7 →](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md)

</div>
