<div align="center">

[← Chapter 4: Design A Rate Limiter](04-Design-A-Rate-Limiter.md) · [📚 All chapters](../README.md) · [Chapter 6: Design A Key-value Store →](06-Design-A-Key-Value-Store.md)

# Chapter 5 · Design Consistent Hashing

*Spread keys across servers so that adding or removing a server moves as few keys as possible.*

![Cassandra](https://img.shields.io/badge/Cassandra-1287B1?style=flat&logo=apachecassandra&logoColor=white)
![Discord](https://img.shields.io/badge/Discord-5865F2?style=flat&logo=discord&logoColor=white)
![Akamai](https://img.shields.io/badge/Akamai-0096D6?style=flat&logo=akamai&logoColor=white)
![Google Maglev](https://img.shields.io/badge/Google_Maglev-4285F4?style=flat&logo=google&logoColor=white)
![Amazon Dynamo](https://img.shields.io/badge/Amazon_Dynamo-6e7681?style=flat)

</div>

> **TL;DR:** The obvious way to pick a server, `hash(key) % N`, falls apart when `N` changes: almost every key moves, and a cache gets a storm of misses. **Consistent hashing** places both servers and keys on a **ring** and sends each key **clockwise to the next server**, so a change only affects the keys next to the server that joined or left. **Virtual nodes** fix the uneven load of the basic version. This one idea powers Dynamo, Cassandra, Discord, Akamai and Google's Maglev.

---

## 💥 The rehashing problem

With 4 servers, `server = hash(key) % 4`. Now one server fails and the pool shrinks to 3, so every key is re-mapped with `% 3`:

| Key | hash(key) *(example)* | % 4 | % 3 | |
|---|---:|:---:|:---:|---|
| key0 | 69,708,792 | 0 | 0 | stays |
| key1 | 24,506,317 | 1 | 1 | stays |
| key2 | 10,463,706 | 2 | 0 | **moves** |
| key3 | 78,389,136 | 0 | 0 | stays |
| key4 | 18,308,801 | 1 | 2 | **moves** |
| key5 | 34,321,215 | 3 | 0 | **moves** |
| key6 | 56,757,958 | 2 | 1 | **moves** |
| key7 | 50,004,462 | 2 | 0 | **moves** |

Most keys move, not just the ones on the failed server. On average **3 out of 4 keys** move when going from 4 servers to 3. For a cache, that means most clients suddenly ask the wrong server.

## 💍 How consistent hashing works

1. **Hash space → ring.** A hash function like SHA-1 outputs numbers from 0 to 2¹⁶⁰ − 1. Join the two ends and you have a ring.
2. **Place the servers** on the ring by hashing their IP or name.
3. **Place the keys** on the ring with the same hash function (no `% N` anywhere).
4. **Lookup:** from a key's position, walk **clockwise** until you hit a server. That server owns the key.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch05-ring-dark.svg">
  <img src="../images/ch05-ring-light.svg" width="100%" alt="Consistent hashing ring. Before: servers s0 to s3 and keys k0 to k3 sit on a ring, and each key goes clockwise to the first server. After adding s4 between s3 and s0, only the keys in the range from s3 to s4 (here just k0) move to s4; every other key stays put.">
</picture>

- **Adding a server** only takes over the keys between it and the server **before it** (going counter-clockwise).
- **Removing a server** hands its keys to the **next server clockwise**; nothing else changes.
- **Finding the affected keys:** start at the server that joined or left and walk **counter-clockwise** to the previous server. The keys in that range are the ones to move.

Formally: when the table resizes, only about **k / n** keys move on average (k keys, n slots), instead of nearly all of them.

## 🧩 Virtual nodes

The basic ring has two problems:
1. **Uneven partitions:** servers sit at random spots, so their slices of the ring differ in size, and when one leaves, its neighbor's slice can double.
2. **Uneven keys:** even with equal slices, keys can bunch up on one server while others sit empty.

The fix is to give each real server **many positions** on the ring, called **virtual nodes**.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch05-vnodes-dark.svg">
  <img src="../images/ch05-vnodes-light.svg" width="100%" alt="Virtual nodes: server 0 and server 1 each appear three times on the ring (s0_0, s0_1, s0_2 and s1_0, s1_1, s1_2), so each owns several smaller arcs. Key k0 walks clockwise to s1_1 and is stored on server 1. With 100 virtual nodes the standard deviation of load is about 10% of the mean; with 200 it is about 5%.">
</picture>

Real systems use far more than three virtual nodes per server. With 100–200 of them, the spread of load (standard deviation) drops to roughly **5–10% of the average**. The price is **more metadata**, so tune the count to your needs.

## ✅ Why it matters

| Benefit | Because |
|---|---|
| **Minimal data movement** | Only the neighbors of a changed server are affected |
| **Easy horizontal scaling** | Data spreads evenly, especially with virtual nodes |
| **Fewer hot spots** | Popular keys (the "celebrity problem" from [Chapter 1](01-Scale-From-Zero-To-Millions-Of-Users.md)) are less likely to pile onto one shard |

**Used in the real world by:** the partitioning layer of **Amazon Dynamo**, **Apache Cassandra**, the **Discord** chat app, the **Akamai** CDN, and Google's **Maglev** load balancer. You'll see it again in [Chapter 6](06-Design-A-Key-Value-Store.md) (key-value store).

## 🔗 Go deeper
- [Consistent Hashing, by Tom White](https://tom-e-white.com/2007/11/consistent-hashing.html): a short, classic explanation with code
- [Dynamo: Amazon's Highly Available Key-value Store (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Stanford CS168, Lecture 1: Consistent Hashing](http://theory.stanford.edu/~tim/s16/l/l1.pdf)
- [Maglev: A Fast and Reliable Software Network Load Balancer](https://research.google/pubs/pub44824/)

---

<div align="center">

[← Chapter 4](04-Design-A-Rate-Limiter.md) · [📚 All chapters](../README.md) · [Chapter 6 →](06-Design-A-Key-Value-Store.md)

</div>
