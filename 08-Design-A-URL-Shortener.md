<div align="center">

[← Chapter 7: Design A Unique ID Generator](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md) · [📚 All chapters](../README.md) · [Chapter 9: Design A Web Crawler →](09-Design-A-Web-Crawler.md)

# Chapter 8 · Design A URL Shortener

*Turn a long link into seven characters, and send 11,600 people a second back to the right page.*

![TinyURL-style](https://img.shields.io/badge/TinyURL--style-6e7681?style=flat)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?style=flat&logo=redis&logoColor=white)
![HTTP 301 / 302](https://img.shields.io/badge/HTTP-301_%2F_302-6e7681?style=flat)

</div>

> **TL;DR:** Two endpoints: one creates a short URL, the other redirects. The heart of the chapter is **how to make the 7-character code**: hash the URL and resolve collisions, or take a **unique ID** ([Chapter 7](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md)) and write it in **base 62**. The design picks base 62, stores `<shortURL, longURL>` in a relational database, and puts a **cache** in front because reads outnumber writes 10 to 1.

---

## 1️⃣ Scope and estimates

**Example:** a long link such as `https://www.example.com/blog/2026/10/designing-a-url-shortener?utm_source=newsletter` gets a short alias like `<short-domain>/TjbIC7r`, and clicking the alias redirects to the original.

- **Use cases:** shorten (long → short), redirect (short → long), plus high availability, scalability and fault tolerance.
- **Traffic:** 100 million new URLs per day.
- **Code:** as short as possible, using only `0-9`, `a-z` and `A-Z`.
- **Simplification:** short URLs are never deleted or updated.

| Estimate | Math | Result |
|---|---|---|
| Write QPS | 100 M ÷ 86,400 s | **≈ 1,160** |
| Read QPS | 10 reads per write | **≈ 11,600** |
| Records over 10 years | 100 M × 365 × 10 | **365 billion** |
| Storage (100-byte average URL) | 365 B × 100 bytes | **≈ 36.5 TB** |

> 📝 Some printings show 365 TB for the storage line, which multiplies by 10 years twice. 365 billion records × 100 bytes is about 36.5 TB.

## 2️⃣ High-level design

**API (REST):**

| Endpoint | Request | Response |
|---|---|---|
| `POST api/v1/data/shorten` | `{longUrl: "…"}` | the short URL |
| `GET api/v1/{shortUrl}` | | a redirect to the long URL |

**301 or 302?** The server answers a short URL with a redirect, and the status code matters:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch08-redirects-dark.svg">
  <img src="../images/ch08-redirects-light.svg" width="100%" alt="301 versus 302 redirects, drawn as two sequence diagrams. On the first click both work the same way: the browser asks the shortener for /TjbIC7r, gets a 301 or 302 with a Location header, and then loads the long URL from the website. On the second click, a 301 is served from the browser's cache and the shortener is not contacted, while a 302 sends the browser through the shortener again, so every click can be counted.">
</picture>

| | 301 Moved Permanently | 302 Found (temporary) |
|---|---|---|
| Browser caches it? | Yes | No |
| Later clicks hit our servers? | No, they go straight to the target | Yes, every time |
| Pick it when you care about | **Less server load** | **Analytics** (click counts, sources) |

**The simplest model** is a hash table of `<shortURL, longURL>`: redirecting is `longURL = table.get(shortURL)`. Shortening needs a function that turns a long URL into a short `hashValue` such that each long URL gets one value and each value maps back to its URL.

## 3️⃣ Deep dive

### Data model
Memory is limited and expensive, so the mapping moves to a **relational database** table with three columns: `id`, `shortURL`, `longURL`.

### How long must the code be?
62 characters are allowed (10 + 26 + 26). Find the smallest *n* with 62ⁿ ≥ 365 billion:

| Length *n* | 62ⁿ (codes available) | Enough? |
|:---:|---:|:---:|
| 5 | 916 million | ❌ |
| 6 | 56.8 billion | ❌ |
| **7** | **3.5 trillion** | ✅ |
| 8 | 218 trillion | ✅ |

So the code is **7 characters**.

### Two ways to make the code

**A. Hash + collision resolution.** Standard hashes are too long. For this repo's URL, `https://github.com/Preetbbhalala/System_Design_Concepts`:

| Hash | Output | Length |
|---|---|---:|
| CRC32 | `59adfe3e` | 8 |
| MD5 | `6e1e9ca43e2119a8e4a4f14475647f4a` | 32 |
| SHA-1 | `41b81e857d782651e320fa9bdb4eb9dac9e058ad` | 40 |

Keep the **first 7 characters**. If that code is already taken, append a fixed string to the URL and hash again until it's free. Checking the database on every attempt is slow, so a **bloom filter** (a compact, probabilistic "have I seen this?" check) can answer most lookups first.

**B. Base 62 conversion.** Take a unique numeric ID and write it in base 62, with digits `0–9` = 0–9, `a–z` = 10–35 and `A–Z` = 36–61. For example, `3141592653589` splits into the base-62 digits 55·19·11·44·38·7·27, which spell **`TjbIC7r`**.

```python
ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"   # 10 + 26 + 26 = 62


def encode(n: int) -> str:
    if n == 0:
        return ALPHABET[0]
    out = []
    while n:
        n, r = divmod(n, 62)
        out.append(ALPHABET[r])
    return "".join(reversed(out))


def decode(s: str) -> int:
    n = 0
    for ch in s:
        n = n * 62 + ALPHABET.index(ch)
    return n


print(encode(3141592653589))   # TjbIC7r
```

| | A. Hash + collision resolution | B. Base 62 of a unique ID ✅ |
|---|---|---|
| Code length | Always 7 | Grows with the ID (7 characters cover 3.5 trillion IDs) |
| Needs an ID generator? | No | Yes ([Chapter 7](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md)) |
| Collisions? | Possible, must be detected and retried | Impossible: unique ID in, unique code out |
| Can someone guess the next code? | No | Yes, if IDs go up by 1, which is a **security concern** |

### Shortening flow (base 62)

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch08-shorten-dark.svg">
  <img src="../images/ch08-shorten-light.svg" width="100%" alt="URL shortening flow with base 62: take the long URL and check whether it is already in the database. If yes, return the stored short URL. If no, get a new unique ID from the ID generator (for example 3141592653589), base-62 encode it into 7 characters (TjbIC7r), save the row (id, shortURL, longURL) and return the short URL.">
</picture>

If the long URL was shortened before, return the existing code. Otherwise get a fresh ID from the distributed ID generator, encode it, store the new row, and return the short URL.

### Redirect flow

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch08-redirect-arch-dark.svg">
  <img src="../images/ch08-redirect-arch-light.svg" width="100%" alt="URL redirect architecture: the user's GET /TjbIC7r goes through a load balancer to stateless web servers. They look up the short URL in a cache first and fall back to the database on a miss, then answer with a 301 or 302 redirect carrying the long URL. A code found in neither place is invalid (404).">
</picture>

Reads dominate, so the `<shortURL, longURL>` pairs are **cached**. On a cache miss, read the database; if the code isn't there either, the user typed an invalid link.

## 4️⃣ Wrap-up talking points

- **Rate limiting** against users who flood the shorten endpoint, by IP or other rules ([Chapter 4](04-Design-A-Rate-Limiter.md)).
- **Web tier scaling** is easy because the web servers are stateless.
- **Database scaling** with replication and sharding.
- **Analytics:** how many people click a link, and when (another reason to prefer 302).
- **Availability, consistency and reliability**, as covered in [Chapter 1](01-Scale-From-Zero-To-Millions-Of-Users.md).

## 🔗 Go deeper
- [MDN: 301 Moved Permanently](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301) · [MDN: 302 Found](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/302)
- [REST API Tutorial](https://www.restapitutorial.com/)
- [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)

---

<div align="center">

[← Chapter 7](07-Design-A-Unique-ID-Generator-In-Distributed-Systems.md) · [📚 All chapters](../README.md) · [Chapter 9 →](09-Design-A-Web-Crawler.md)

</div>
