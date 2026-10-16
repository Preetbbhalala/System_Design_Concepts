<div align="center">

[← Chapter 12: Design A Chat System](12-Design-A-Chat-System.md) · [📚 All chapters](../README.md) · [Chapter 14: Design YouTube →](14-Design-YouTube.md)

# Chapter 13 · Design A Search Autocomplete System

*Suggest the top 5 completions for every keystroke, in under 100 ms.*

![Google](https://img.shields.io/badge/Google_Search-4285F4?style=flat&logo=google&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-231F20?style=flat&logo=apachekafka&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache_Spark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Apache Hadoop](https://img.shields.io/badge/Apache_Hadoop-66CCFF?style=flat&logo=apachehadoop&logoColor=black)

</div>

> **TL;DR:** Also called typeahead or "design top k". The core is a **trie** (prefix tree) where **every node caches its top k queries**, so a lookup is just "walk to the prefix node, read the list". Two services surround it: an offline **data gathering** pipeline (logs → aggregate → rebuild the trie weekly) and a fast **query service** (load balancer → API servers → **filter layer** → in-memory **trie cache**). Browser caching, AJAX and smart sharding keep it fast at scale.

---

## 1️⃣ Scope and estimates

| Question | Answer |
|---|---|
| Where does matching happen? | Only at the **start** of the query (prefix match) |
| How many suggestions? | **5**, ranked by **historical query frequency** |
| Spell check? | No |
| Language / characters | English, lowercase letters only (multi-language is a wrap-up topic) |
| Users | **10 million DAU** |

**Requirements:** **fast** (Facebook found results must come back within about **100 ms** or typing feels stuttery), **relevant**, **sorted** by popularity, **scalable**, and **highly available**.

| Estimate | Math | Result |
|---|---|---|
| Requests | Every keystroke is a request: typing `cars` sends `?q=c`, `?q=ca`, `?q=car`, `?q=cars` | ~20 requests per search |
| QPS | 10 M users × 10 searches × 20 requests ÷ 86,400 s | **≈ 23,000** (the book rounds to ~24,000) |
| Peak QPS | 2 × average | **≈ 46,000** |
| New data per day | 10 M × 10 × 20 bytes × 20% new | **0.4 GB** |

## 2️⃣ High-level design

- **Data gathering service:** collects the queries users search for and keeps a **frequency table** (`query`, `frequency`). Updating it on every search is the simple starting point.
- **Query service:** given a prefix, return the 5 most frequent matches. With a database that's a `LIKE 'prefix%'` filter ordered by frequency with a limit of 5. Fine for small data; a **bottleneck** at scale.

## 3️⃣ Deep dive

### The trie, upgraded
A **trie** (say "try", from re*trie*val) is a tree where the root is the empty string, each node is a **prefix**, and each child adds one character. Store each query's **frequency** on its node.

The basic lookup is: (1) **find the prefix node**, O(p) for a prefix of length p; (2) **collect every query** below it, O(c) for c children; (3) **sort** and take the top k, O(c log c). In the worst case that's the whole trie, which is too slow. Two optimizations fix it:

1. **Cap the prefix length.** People rarely type long queries, so limit p to, say, 50 characters. Finding the prefix becomes **O(1)**.
2. **Cache the top k queries at every node.** Now there's nothing to collect or sort: just read the list, also **O(1)**. It costs a lot of memory, but fast responses are worth it.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch13-trie-dark.svg">
  <img src="../images/ch13-trie-light.svg" width="100%" alt="A trie with cached top results. Each node is a prefix and stores the 2 most popular queries below it, for example the root and the prefixes c and ca store cat 40 and car 35, and car stores car 35 and card 22. Typing ca means walking two steps from the root and reading the cached answer, cat and car, without searching the subtree.">
</picture>

<details>
<summary><b>💻 A trie with cached top-k in ~30 lines of Python</b></summary>

```python
K = 5             # suggestions to return
MAX_PREFIX = 50   # users rarely type more, so lookups are O(1)


class Node:
    def __init__(self):
        self.children = {}
        self.top = []                      # cached (frequency, query) pairs, best first


def build(frequencies: dict) -> Node:
    root = Node()
    for query, freq in frequencies.items():
        node, path = root, [root]
        for ch in query:
            node = node.children.setdefault(ch, Node())
            path.append(node)
        for n in path:                     # the query is a candidate for every prefix on its path
            n.top = sorted(n.top + [(freq, query)], reverse=True)[:K]
    return root


def suggest(root: Node, prefix: str) -> list:
    node = root
    for ch in prefix[:MAX_PREFIX]:
        node = node.children.get(ch)
        if node is None:
            return []
    return [query for _, query in node.top]


trie = build({"cat": 40, "car": 35, "card": 22, "care": 18, "cart": 12, "dog": 30, "do": 25, "dot": 9})
print(suggest(trie, "ca"))    # ['cat', 'car', 'card', 'care', 'cart']
```

</details>

### Data gathering: rebuild, don't update live
Updating the trie on every search is impractical: there can be **billions of queries a day**, and the top suggestions barely change from one query to the next. A real-time product (like X) needs fresher data than, say, most Google searches, but either way the trie is built from **analytics logs**.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch13-pipeline-dark.svg">
  <img src="../images/ch13-pipeline-light.svg" width="100%" alt="Autocomplete architecture in two lanes. Data gathering, offline and rebuilt weekly: analytics logs are aggregated into weekly query counts, workers build the trie from them and store it in the trie database. Query service, on every keystroke: the user's request goes through a load balancer to API servers, then a filter layer that removes unsafe suggestions, to an in-memory trie cache that takes a weekly snapshot of the trie database and is refilled from it on a miss.">
</picture>

| Component | Role |
|---|---|
| **Analytics logs** | Raw search events (query + time); append-only, not indexed |
| **Aggregators** | Roll the logs up into counts. Real-time products aggregate over short windows; weekly is enough for many. **The design assumes a weekly rebuild.** |
| **Aggregated data** | One row per query per week: `query`, `week_start`, `frequency` |
| **Workers** | Scheduled jobs that build the trie and save it to the **Trie DB** |
| **Trie cache** | A distributed in-memory copy for fast reads; loads a weekly snapshot |
| **Trie DB** | Persistent storage, as either a **document store** (serialize the whole trie, e.g. MongoDB) or a **key-value store** where each **prefix is a key** and the node's data (its top-k list) is the value |

The key-value layout for the trie above looks like:

| Key (prefix) | Value (cached top queries) |
|---|---|
| `c` | `[cat: 40, car: 35]` |
| `ca` | `[cat: 40, car: 35]` |
| `car` | `[car: 35, card: 22]` |
| `d` | `[dog: 30, do: 25]` |

### Query service
The request goes through the load balancer to the **API servers**, which read from the **trie cache**; on a miss (a cache node out of memory or down), the data is reloaded from the Trie DB so later requests hit the cache. Speed-ups:
- **AJAX** requests, so the page never reloads.
- **Browser caching:** suggestions change slowly, so let the browser keep them. When the book was written, Google's autocomplete responses carried `Cache-Control: private, max-age=3600`: cache for an hour, in this user's browser only (no shared caches).
- **Data sampling:** logging every query is costly, so log only 1 in every N.

### Trie operations
- **Create:** workers build it from aggregated data.
- **Update:** normally, **replace the whole trie weekly**. Updating single nodes is slow, and every ancestor's cached top-k would have to change too; OK only for small tries.
- **Delete:** hateful, violent, sexually explicit or dangerous suggestions must go. A **filter layer** in front of the trie cache removes them immediately (with flexible rules), and they're **deleted from the database asynchronously** so the next rebuild is clean.

### Scaling the storage
Shard by the **first letter**: two servers could take `a–m` and `n–z`; with up to 26 servers, one letter each; beyond that, shard on the second or third letter (e.g. `aa–ag`, `ah–an`, `ao–au`, `av–az`). But far more words start with `c` than with `x`, so the shards end up uneven. The fix is a **shard map manager** that groups prefixes by **historical traffic**: if `s` alone gets about as many queries as `u` through `z` together, give each group its own shard.

## 4️⃣ Wrap-up talking points

- **Other languages:** store **Unicode** characters in the trie nodes.
- **Different countries, different top queries:** build a trie per country and serve it from **CDNs**.
- **Trending (real-time) queries:** a weekly rebuild can't catch breaking news. Ideas: shard to shrink the working set, **weight recent searches more** in the ranking, and process the query stream with **stream-processing** tools like Kafka, Spark Streaming, Storm or Hadoop MapReduce.

## 🔗 Go deeper
- [Trie (prefix tree)](https://en.wikipedia.org/wiki/Trie)
- [Prefix Hash Tree: an indexing data structure over distributed hash tables](https://people.eecs.berkeley.edu/~sylvia/papers/pht.pdf)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [Unicode FAQ: the basics](https://www.unicode.org/faq/basic_q.html)

---

<div align="center">

[← Chapter 12](12-Design-A-Chat-System.md) · [📚 All chapters](../README.md) · [Chapter 14 →](14-Design-YouTube.md)

</div>
