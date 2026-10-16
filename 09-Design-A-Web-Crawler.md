<div align="center">

[← Chapter 8: Design A URL Shortener](08-Design-A-URL-Shortener.md) · [📚 All chapters](../README.md) · [Chapter 10: Design A Notification System →](10-Design-A-Notification-System.md)

# Chapter 9 · Design A Web Crawler

*Download a billion pages a month without getting lost, getting trapped, or hammering anyone's server.*

![Googlebot](https://img.shields.io/badge/Googlebot-4285F4?style=flat&logo=google&logoColor=white)
![Internet Archive](https://img.shields.io/badge/Internet_Archive-666666?style=flat&logo=internetarchive&logoColor=white)
![Scrapy](https://img.shields.io/badge/Scrapy-60A839?style=flat&logo=scrapy&logoColor=white)
![Puppeteer](https://img.shields.io/badge/Puppeteer-40B5A4?style=flat&logo=puppeteer&logoColor=white)

</div>

> **TL;DR:** The core loop is tiny: download pages, extract their links, queue the new ones, repeat. The design work is everything around it: a **URL frontier** that enforces **politeness** (one host per queue, with pauses) and **priority**, **"seen?" checks** for duplicate content and URLs, a cached **DNS resolver**, respect for **robots.txt**, and defenses against **spider traps** and junk. Target: 1 billion HTML pages a month (~400 pages/s), kept for 5 years (~30 PB).

---

## 🕷️ What crawlers are used for

| Use | Example |
|---|---|
| **Search engine indexing** (the most common) | Googlebot builds the index behind Google Search |
| **Web archiving** | National libraries preserving websites, e.g., the US Library of Congress and the EU web archive |
| **Web mining** | Finance firms collecting annual reports and shareholder-meeting documents |
| **Web monitoring** | Spotting copyright and trademark infringement (e.g., Digimarc) |

The basic algorithm is three steps on repeat: **download** the pages for a set of URLs → **extract** their links → **add** the new links to the to-do list. Easy for a school project; very hard at web scale.

## 1️⃣ Scope and estimates

Agreed scope: crawl for **search engine indexing**, **HTML only**, **1 billion pages per month**, include **new and edited pages**, **store pages for 5 years**, and **ignore duplicate content**.

A good crawler is also:

| Quality | Why it matters |
|---|---|
| **Scalable** | Billions of pages: parallelize everything |
| **Robust** | Bad HTML, dead servers, crashes and malicious links are normal on the web |
| **Polite** | Never flood one website with requests |
| **Extensible** | New content types (say, images) shouldn't need a redesign |

| Estimate | Math | Result |
|---|---|---|
| Average QPS | 1 B ÷ 30 days ÷ 86,400 s | **≈ 400 pages/s** |
| Peak QPS | 2 × average | **≈ 800 pages/s** |
| Storage per month | 1 B pages × 500 KB | **500 TB** |
| Storage for 5 years | 500 TB × 12 × 5 | **30 PB** |

## 2️⃣ High-level design

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch09-architecture-dark.svg">
  <img src="../images/ch09-architecture-light.svg" width="100%" alt="Web crawler architecture as a loop with 11 numbered steps: 1 seed URLs go into the URL Frontier; 2 the HTML Downloader takes URLs from it; 3 it resolves hosts with the DNS resolver; 4 the content parser validates pages; 5 and 6 Content Seen? drops pages whose content is already in content storage; 7 the URL extractor pulls out links; 8 the URL filter removes unwanted ones; 9 and 10 URL Seen? checks them against URL storage; 11 only new URLs go back into the frontier.">
</picture>

| Step | Component | Job |
|:---:|---|---|
| 1 | **Seed URLs** | Starting points. Pick them to reach as much of the web as possible, e.g., by country or by topic (shopping, sports, health…). There's no perfect answer; think out loud. |
| 2 | **URL Frontier** | The queue of URLs still **to be downloaded** (think FIFO for now) |
| 3 | **HTML Downloader** + **DNS resolver** | Turns a host name into an IP address, then fetches the page |
| 4 | **Content parser** | Parses and validates pages; malformed ones waste storage. A separate component so it doesn't slow the downloaders. |
| 5–6 | **Content Seen?** + **content storage** | About 29% of pages are duplicates. Compare **hashes** of pages (not characters) and drop content you already have. Storage is mostly disk, with popular content in memory. |
| 7 | **URL extractor** | Pulls links out of HTML and turns relative paths into absolute URLs |
| 8 | **URL filter** | Drops unwanted file types, error links and blocklisted sites |
| 9–10 | **URL Seen?** + **URL storage** | Skips URLs already visited or already queued (a **bloom filter** or hash table), which prevents repeat work and infinite loops |
| 11 | back to the frontier | Only brand-new URLs are queued |

## 3️⃣ Deep dive

### BFS, not DFS
The web is a directed graph: pages are nodes, links are edges. **DFS** can go impossibly deep, so crawlers use **BFS** with a FIFO queue. Plain BFS has two flaws:
1. **It's impolite.** Most links on a page point to the same host, so a parallel crawler ends up hammering that one site.
2. **It ignores priority.** A random forum post and a major home page get equal treatment.

### The URL frontier
The frontier fixes both flaws with two layers of queues:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch09-frontier-dark.svg">
  <img src="../images/ch09-frontier-light.svg" width="100%" alt="URL frontier with two modules. Front queues handle priority: a prioritizer scores incoming URLs and puts them in queues f1 (high) to fn (low), and a selector picks from them at random with a bias toward high priority. Back queues handle politeness: a router uses a host-to-queue mapping table so each queue b1 to bn holds one host's URLs, and each worker thread drains exactly one queue, one page at a time with a delay.">
</picture>

- **Front queues → priority.** A **prioritizer** scores each URL by usefulness (PageRank, site traffic, how often it changes…) and drops it into one of the queues f1…fn. A **selector** picks queues at random, **biased toward high priority**.
- **Back queues → politeness.** A **router** uses a **host → queue mapping table** so every back queue holds URLs from **one host only**. Each **worker thread** owns one queue and downloads **one page at a time**, optionally **pausing** between them.
- **Freshness.** Pages change, so recrawl them, guided by each page's **update history**, and give **important pages** an earlier and more frequent turn. Recrawling everything is too expensive.
- **Storage.** A real frontier can hold **hundreds of millions** of URLs. Memory alone is neither durable nor big enough; disk alone is too slow. Keep most URLs **on disk** with **in-memory buffers** for enqueue/dequeue, flushed to disk periodically.

### HTML downloader
**robots.txt** (the Robots Exclusion Protocol) tells crawlers which paths they may fetch. Check it **before** crawling a site, follow it, and **cache** it per host, refreshing periodically. An example:

```text
User-agent: *
Disallow: /checkout/
Disallow: /internal-search

User-agent: ExampleBot
Disallow: /
```

**Performance tricks:**

| Technique | Why |
|---|---|
| **Distributed crawl** | Partition the URL space across many servers, each running many threads |
| **DNS cache** | DNS lookups take 10–200 ms and many DNS APIs are synchronous, so one slow lookup blocks other threads. Keep a host → IP cache refreshed by cron jobs. |
| **Locality** | Put crawl servers (and their caches, queues and storage) near the sites they crawl |
| **Short timeouts** | Give up on slow or silent servers and move on to other pages |

### Robustness and extensibility
- **Consistent hashing** spreads load across downloaders and lets you add or remove them easily ([Chapter 5](05-Design-Consistent-Hashing.md)).
- **Save crawl state and data** so a failed crawl can resume where it stopped.
- **Handle exceptions** without crashing, and **validate data** to prevent errors.
- **Pluggable modules:** new content types are new modules, e.g., a **PNG downloader** or a **web monitor** that looks for copyright and trademark violations.

### Problematic content

| Problem | Defense |
|---|---|
| **Redundant content** (nearly 30% of pages) | Hashes or checksums to spot duplicates |
| **Spider traps**, e.g., endlessly nested paths like `/foo/bar/foo/bar/…` | Cap the URL length; sites with an absurd number of pages stand out, so verify them manually and exclude them or add custom filters |
| **Data noise:** ads, code snippets, spam URLs | Exclude it when possible; it adds no value |

## 4️⃣ Wrap-up talking points

- **Server-side (dynamic) rendering:** many links are created by JavaScript, so render pages before parsing them, or you'll miss those links.
- **Filter out spam and low-quality pages** with an anti-spam component; storage and crawl capacity are finite.
- **Database replication and sharding** for the data layer.
- **Horizontal scaling** with stateless download servers; big crawls use hundreds or thousands of them.
- **Availability, consistency, reliability** ([Chapter 1](01-Scale-From-Zero-To-Millions-Of-Users.md)) and **analytics** to tune the crawler.

## 🔗 Go deeper
- [Web Crawling, by Christopher Olston and Marc Najork (survey)](http://infolab.stanford.edu/~olston/publications/crawling_survey.pdf)
- [RFC 9309: Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309)
- [IRLbot: Scaling to 6 Billion Pages and Beyond](https://irl.cs.tamu.edu/people/hsin-tsang/papers/www2008.pdf)
- [PageRank, the link-based ranking behind URL prioritization](https://en.wikipedia.org/wiki/PageRank)

---

<div align="center">

[← Chapter 8](08-Design-A-URL-Shortener.md) · [📚 All chapters](../README.md) · [Chapter 10 →](10-Design-A-Notification-System.md)

</div>
