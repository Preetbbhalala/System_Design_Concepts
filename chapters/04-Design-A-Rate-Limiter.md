<div align="center">

[← Chapter 3: A Framework For System Design Interviews](03-A-Framework-For-System-Design-Interviews.md) · [📚 All chapters](../README.md) · [Chapter 5: Design Consistent Hashing →](05-Design-Consistent-Hashing.md)

# Chapter 4 · Design A Rate Limiter

*Decide, in microseconds and across many servers, whether a request is one too many.*

![Redis](https://img.shields.io/badge/Redis-FF4438?style=flat&logo=redis&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=flat&logo=stripe&logoColor=white)
![Lyft](https://img.shields.io/badge/Lyft-FF00BF?style=flat&logo=lyft&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=flat&logo=cloudflare&logoColor=white)
![Shopify](https://img.shields.io/badge/Shopify-7AB55C?style=flat&logo=shopify&logoColor=white)

</div>

> **TL;DR:** A rate limiter caps how many requests a client (a user, IP or device) may make in a time window, and rejects the rest with **HTTP 429**. The design boils down to: pick an algorithm (token bucket is the popular default), keep the **counters in an in-memory store like Redis**, read the **rules from config**, and solve the two distributed headaches, **race conditions** and **synchronization** across limiter instances.

---

## 🎯 Why rate limit?
- **Stop resource starvation** from DoS attacks, whether intentional or accidental. Almost every big public API has limits.
- **Cut costs:** fewer servers, and fewer calls to *paid* third-party APIs (credit checks, payments…).
- **Protect servers** from bots and misbehaving clients.

## 1️⃣ Scope and requirements
The interview settles on a **server-side API rate limiter** that supports **flexible rules** (by IP, user ID, etc.), handles **large scale**, works in a **distributed** setup, and **tells users** when they're throttled. Whether it's a separate service or part of the application code is your call.

| Requirement | What it means |
|---|---|
| Accurate | Really does block excess requests |
| Low latency | Doesn't slow down normal HTTP responses |
| Small memory footprint | Counters must be cheap |
| Distributed | Shared across many servers and processes |
| Clear errors | Throttled users get an explicit response |
| Fault tolerant | If the limiter (e.g., its cache) breaks, the whole system doesn't |

## 2️⃣ Where does the limiter live?
- **Client side:** unreliable. Requests can be forged and you may not control the client.
- **Server side:** inside the API servers.
- **Middleware / API gateway:** a layer in front of the APIs. Managed **API gateways** already do rate limiting, along with authentication, SSL termination and IP allow-lists.

No single answer is right. Weigh your **tech stack**, how much **control over the algorithm** you need, whether you **already run a gateway**, and how much **engineering time** you have (a commercial gateway is the fast path).

## 🧮 The five algorithms

| Algorithm | How it works | 👍 | 👎 | Known users |
|---|---|---|---|---|
| **Token bucket** | A bucket holds up to *N* tokens and is refilled at a fixed rate. Each request spends one token; no token, no entry. | Simple, memory-light, **allows short bursts** | Bucket size and refill rate are hard to tune | Amazon, Stripe |
| **Leaking bucket** | Requests join a FIFO queue that's drained at a **fixed rate**; a full queue drops new ones. | Memory-light, **steady outflow** | Bursts fill the queue with old requests, so new ones get dropped | Shopify |
| **Fixed window counter** | Count requests per fixed window (e.g., per minute); block once the count hits the limit. | Very simple, cheap, quota resets at round times | **Spikes at window edges** can let up to 2× through | |
| **Sliding window log** | Keep a timestamp per request (e.g., a Redis sorted set), drop the expired ones, and allow if the log is within the limit. | **Exact** in every rolling window | Heavy memory: even rejected requests leave timestamps | |
| **Sliding window counter** | Estimate the rolling count from the current window plus a weighted share of the previous one. | Smooths spikes, memory-light | An approximation (assumes the previous window was even) | Cloudflare |

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch04-token-bucket-dark.svg">
  <img src="../images/ch04-token-bucket-light.svg" width="100%" alt="Token bucket: a refiller adds 2 tokens per second to a bucket that holds at most 4 (extra tokens spill over). Each request takes one token and is forwarded; if the bucket is empty the request is dropped with HTTP 429.">
</picture>

**How many buckets?** It depends on the rules: one per **endpoint per user** (for example, separate buckets for posting, adding friends and liking), one per **IP address**, or a single **global** bucket (e.g., at most 10,000 requests per second for the whole system).

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch04-fixed-window-dark.svg">
  <img src="../images/ch04-fixed-window-light.svg" width="100%" alt="Fixed window problem: with a limit of 5 per minute, 5 requests at the end of the first minute and 5 at the start of the next are all allowed, so the one-minute span from 2:00:30 to 2:01:30 sees 10 requests, twice the limit.">
</picture>

**The sliding-window-counter formula.** Rolling count = this window's count + last window's count × (the share of the rolling window that overlaps last window). With a limit of 7 per minute, 5 requests last minute, 3 so far this minute, and a new request 30% into the minute: 3 + 5 × 0.7 = **6.5**, rounded down to 6, so the request is allowed. The limit will be hit after one more request. Cloudflare measured only **0.003%** of 400 million requests being wrongly allowed or limited with this approximation.

---

## 🏗️ The design

Counters live **in memory**, not in a database, because disk is too slow. **Redis** fits perfectly: `INCR` bumps a counter and `EXPIRE` makes it vanish when the window ends.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch04-architecture-dark.svg">
  <img src="../images/ch04-architecture-light.svg" width="100%" alt="Rate limiter detailed design: the client calls the rate limiter middleware, which loads rules from a cache (filled by workers from config files on disk) and reads and increments counters in Redis. Requests under the limit go to the API servers; throttled ones get HTTP 429 with a Retry-After header and can optionally be queued for later.">
</picture>

1. **Rules** are written as config files on disk. **Workers** pull them regularly into a **rules cache**.
2. Each request hits the **rate limiter middleware** first. It loads the matching rule and reads the counter (and last-request timestamp) from **Redis**.
3. **Under the limit:** the request is forwarded to the API servers and the counter is incremented.
4. **Over the limit:** the client gets **HTTP 429 Too Many Requests**. The request is dropped, or **queued** to be processed later (e.g., orders during an overload).

**Rules** follow the style of Lyft's open-source limiter: a domain, a descriptor and a rate, for example:

```yaml
domain: search_api
descriptors:
  - key: plan
    value: free_tier
    rate_limit:
      unit: second
      requests_per_unit: 10   # free-tier users get at most 10 searches per second
```

**Tell the client what's happening** with response headers:

| Header | Meaning |
|---|---|
| `X-Ratelimit-Limit` | Calls allowed per window |
| `X-Ratelimit-Remaining` | Calls left in the current window |
| `X-Ratelimit-Retry-After` | Seconds to wait before retrying (sent with the 429) |

## 3️⃣ Deep dive: making it work at scale

**Race condition.** "Read counter → check → write counter + 1" isn't atomic. Two concurrent requests can both read 3 and both write 4, when the right value is 5. Locks fix it but are slow. Use a **Lua script** (which runs atomically in Redis) or **Redis sorted sets** instead.

**Synchronization.** With several limiter instances behind a stateless tier, a client's requests land on different instances. **Sticky sessions** would work but don't scale and aren't flexible. Put the counters in a **central store (Redis)** that every instance shares.

**Performance.** Run the limiter in **many data centers / edge locations** close to users (Cloudflare had 194 edge locations in 2020), and sync counters with **eventual consistency** (see [Chapter 6](06-Design-A-Key-Value-Store.md)).

**Monitoring.** Check that both the **algorithm** and the **rules** are working. If valid traffic gets dropped, relax the rules. If bursty events like **flash sales** break things, switch to an algorithm that tolerates bursts, such as the **token bucket**.

## 4️⃣ Wrap-up talking points
- **Hard vs soft limits:** never exceed the threshold, or allow short overshoots.
- **Other layers:** this chapter limits at **layer 7** (HTTP). You can also limit by IP at **layer 3**, e.g., with iptables.
- **Client etiquette:** cache responses, respect the limits, handle errors gracefully, and **back off** before retrying.

## 🔗 Go deeper
- [Stripe: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Cloudflare: How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [Lyft's open-source rate limiter, now maintained under Envoy](https://github.com/envoyproxy/ratelimit)
- [Cloudflare Learning Center: What is rate limiting?](https://www.cloudflare.com/learning/bots/what-is-rate-limiting/)

---

<div align="center">

[← Chapter 3](03-A-Framework-For-System-Design-Interviews.md) · [📚 All chapters](../README.md) · [Chapter 5 →](05-Design-Consistent-Hashing.md)

</div>
