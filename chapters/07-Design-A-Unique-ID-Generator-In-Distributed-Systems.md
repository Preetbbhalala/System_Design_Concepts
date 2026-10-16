<div align="center">

[← Chapter 6: Design A Key-value Store](06-Design-A-Key-Value-Store.md) · [📚 All chapters](../README.md) · [Chapter 8: Design A URL Shortener →](08-Design-A-URL-Shortener.md)

# Chapter 7 · Design A Unique ID Generator In Distributed Systems

*Hand out 10,000+ unique, time-ordered 64-bit numbers every second, with no central bottleneck.*

![X (Twitter) Snowflake](https://img.shields.io/badge/X_%28Twitter%29_Snowflake-000000?style=flat&logo=x&logoColor=white)
![Flickr](https://img.shields.io/badge/Flickr-0063DC?style=flat&logo=flickr&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Discord](https://img.shields.io/badge/Discord-5865F2?style=flat&logo=discord&logoColor=white)

</div>

> **TL;DR:** A database's `auto_increment` breaks down once data lives on many servers. The chapter weighs four options (multi-master `auto_increment`, UUIDs, a ticket server, and Twitter's Snowflake) and picks **Snowflake**: split a 64-bit number into **timestamp + datacenter ID + machine ID + sequence**, so every machine can mint IDs on its own, with no coordination, and the IDs still **sort by time**.

---

## 1️⃣ Requirements

| Requirement | Detail |
|---|---|
| Unique | No two records ever get the same ID |
| Numeric | Numbers only |
| 64-bit | Must fit in a 64-bit integer |
| Ordered by time | Later IDs are bigger, though not necessarily by exactly 1 (an evening ID beats a morning ID) |
| Fast | More than **10,000 IDs per second** |

**Why not plain `auto_increment`?** One database server can't keep up at this scale, and keeping counters unique across many databases without slowing everything down is hard.

## 2️⃣ Four candidate designs

| Approach | How it works | 👍 | 👎 |
|---|---|---|---|
| **Multi-master replication** | Each database uses `auto_increment`, but steps by **k** (the number of DB servers). With k = 2, server A hands out 1, 3, 5… and server B 2, 4, 6… | Scales with the number of DB servers | Hard across **data centers**; IDs don't follow time across servers; adding or removing a server breaks the pattern |
| **UUID** | Every web server generates 128-bit UUIDs locally, e.g. `09c93e62-50b4-468d-bf8a-c07e1040bfb2` | Simple, **no coordination**, scales with web servers | **128 bits**, not 64; not time-ordered; can contain letters |
| **Ticket server** | One central database with `auto_increment` hands out IDs to everyone (Flickr's approach) | Numeric, easy, fine for small to medium scale | **Single point of failure**; adding more ticket servers brings sync problems |
| **Twitter Snowflake** ✅ | Split the 64 bits into sections that each machine fills in on its own | Meets every requirement | Depends on reasonably synced clocks |

> UUID collisions are astronomically unlikely: per Wikipedia, you'd have to create about a billion UUIDs per second for roughly a century before the chance of a single duplicate reached 50%. Size and ordering, not collisions, rule them out here.

## 🧊 The Snowflake layout

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch07-snowflake-dark.svg">
  <img src="../images/ch07-snowflake-light.svg" width="100%" alt="Snowflake 64-bit ID layout: 1 sign bit (always 0), 41 bits of timestamp, 5 bits of datacenter ID, 5 bits of machine ID and 12 bits of sequence number. The example decodes to 503,308,025,343 ms after Twitter's epoch (2026-10-16 09:30:00 UTC), datacenter 3, machine 7, sequence 5, which together read as the ID 2111026863928668165.">
</picture>

| Section | Bits | What it holds | Capacity |
|---|:---:|---|---|
| **S**ign | 1 | Always `0`, reserved (it also keeps the ID positive in a signed 64-bit integer) | n/a |
| **Timestamp** | 41 | Milliseconds since a **custom epoch** (Twitter's default: `1288834974657` = Nov 04 2010, 01:42:54 UTC) | 2⁴¹ ms ≈ **69.7 years** |
| **Datacenter ID** | 5 | Which data center made the ID | 2⁵ = **32** data centers |
| **Machine ID** | 5 | Which machine in that data center | **32** machines each |
| **Sequence** | 12 | Counter that resets to 0 every millisecond | 2¹² = **4,096 IDs per ms** per machine |

## 3️⃣ Deep dive

**Fixed vs moving parts.** The datacenter and machine IDs are set **when the generator starts** and normally never change. Changing them by accident can produce duplicate IDs, so treat any change carefully. The timestamp and sequence are filled in **at runtime**.

**Timestamp.** It occupies the **most significant bits**, which is exactly why the IDs sort by time. 41 bits of milliseconds last about 69.7 years: with Twitter's 2010 epoch that runs out in **July 2080**. Picking an epoch close to your launch date pushes the overflow further out; after that you need a new epoch or a migration.

**Sequence.** It's `0` unless the same machine creates several IDs in one millisecond. 4,096 per ms means **4.1 million IDs per second per machine**, and up to **~4.2 billion per second** across all 32 × 32 machines. The requirement was 10,000.

<details>
<summary><b>💻 A Snowflake generator in ~30 lines of Python</b></summary>

```python
import threading
import time

EPOCH_MS = 1288834974657          # Twitter's custom epoch: Nov 04 2010, 01:42:54 UTC


class Snowflake:
    def __init__(self, datacenter_id: int, machine_id: int):
        assert 0 <= datacenter_id < 32 and 0 <= machine_id < 32   # 5 bits each
        self.datacenter_id, self.machine_id = datacenter_id, machine_id
        self.last_ms, self.sequence = -1, 0
        self.lock = threading.Lock()

    def next_id(self) -> int:
        with self.lock:
            now = time.time_ns() // 1_000_000
            if now < self.last_ms:
                raise RuntimeError("clock moved backwards; refusing to risk duplicates")
            if now == self.last_ms:
                self.sequence = (self.sequence + 1) & 0xFFF     # 12 bits: 0..4095
                if self.sequence == 0:                        # 4,096 IDs used in this ms
                    while now <= self.last_ms:                # wait for the next ms
                        now = time.time_ns() // 1_000_000
            else:
                self.sequence = 0
            self.last_ms = now
            return ((now - EPOCH_MS) << 22) | (self.datacenter_id << 17) \
                | (self.machine_id << 12) | self.sequence


def decode(id_: int) -> dict:
    return {
        "time_ms": (id_ >> 22) + EPOCH_MS,
        "datacenter_id": (id_ >> 17) & 0b11111,
        "machine_id": (id_ >> 12) & 0b11111,
        "sequence": id_ & 0xFFF,
    }


gen = Snowflake(datacenter_id=3, machine_id=7)
print(decode(gen.next_id()))
```

The shifts mirror the layout: the timestamp moves left past the 22 bits below it (5 + 5 + 12), the datacenter ID past 17, the machine ID past 12.

</details>

## 4️⃣ Wrap-up talking points

- **Clock synchronization.** The design assumes every ID server has the same clock, which isn't guaranteed across machines (or even cores). **NTP** is the usual fix; also decide what to do if a clock jumps backwards (the code above refuses).
- **Tune the section lengths.** A low-traffic, long-lived system can trade sequence bits for timestamp bits:

  | Timestamp bits | Lasts about |
  |:---:|---|
  | 40 | 34.8 years |
  | **41** | **69.7 years** |
  | 42 | 139 years |
  | 43 | 279 years |

- **High availability.** Every write in the company may need an ID, so the generator is mission-critical and must not go down.

> 🌍 **Seen in the wild:** Discord, Instagram and many others use Snowflake-style IDs. Instagram's variant swaps the machine bits for a logical shard ID and generates IDs inside PostgreSQL.

## 🔗 Go deeper
- [Announcing Snowflake (Twitter Engineering, 2010)](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake)
- [Ticket Servers: Distributed Unique Primary Keys on the Cheap (Flickr)](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
- [Sharding & IDs at Instagram](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- [Discord API reference: Snowflakes](https://docs.discord.com/developers/reference#snowflakes)
- [Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier) · [Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol)

---

<div align="center">

[← Chapter 6](06-Design-A-Key-Value-Store.md) · [📚 All chapters](../README.md) · [Chapter 8 →](08-Design-A-URL-Shortener.md)

</div>
