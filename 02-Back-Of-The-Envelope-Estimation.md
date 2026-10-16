<div align="center">

[← Chapter 1: Scale From Zero To Millions Of Users](01-Scale-From-Zero-To-Millions-Of-Users.md) · [📚 All chapters](../README.md) · [Chapter 3: A Framework For System Design Interviews →](03-A-Framework-For-System-Design-Interviews.md)

# Chapter 2 · Back-of-the-envelope Estimation

*Quick, rounded math that tells you whether a design can survive the load.*

</div>

> **TL;DR:** Interviewers often ask "how much traffic, storage, or how many servers?" You aren't expected to be exact. You're expected to reason with a few memorized numbers (powers of two, latency figures, availability nines), state your assumptions, round boldly, and keep your units straight. The chapter gives you those reference numbers and walks through a Twitter-style estimate.

---

## 🔢 Powers of two

Data sizes grow in steps of 2¹⁰ ≈ 1,000. One ASCII character is 1 byte (8 bits).

| Power | Roughly | Unit |
|:---:|---|---|
| 2¹⁰ | a thousand | 1 **KB** (kilobyte) |
| 2²⁰ | a million | 1 **MB** (megabyte) |
| 2³⁰ | a billion | 1 **GB** (gigabyte) |
| 2⁴⁰ | a trillion | 1 **TB** (terabyte) |
| 2⁵⁰ | a quadrillion | 1 **PB** (petabyte) |

## ⏱️ Latency numbers every programmer should know

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch02-latency-dark.svg">
  <img src="../images/ch02-latency-light.svg" width="100%" alt="Horizontal log-scale bar chart of latency numbers: L1 cache 0.5 ns, branch mispredict 5 ns, L2 cache 7 ns, mutex 100 ns, main memory 100 ns, compress 1 KB 10 µs, send 2 KB over 1 Gbps 20 µs, read 1 MB from memory 250 µs, data-center round trip 500 µs, disk seek 10 ms, read 1 MB from network 10 ms, read 1 MB from disk 30 ms, California–Netherlands round trip 150 ms.">
</picture>

These are 2010 figures. Hardware has gotten faster since, but the **ratios** are what matter:

- **Memory is fast; disk is slow.** Reading 1 MB from disk is about 120× slower than from memory.
- **Avoid disk seeks** when you can.
- **Simple compression is cheap**, so it pays to shrink data before it crosses the network.
- **Crossing regions costs real time:** a round trip between continents is roughly 300× a round trip inside one data center.

<sub>Units: 1 ns = 10⁻⁹ s · 1 µs = 1,000 ns · 1 ms = 1,000 µs = 1,000,000 ns</sub>

## 🟢 Availability in "nines"

**High availability** means a system stays up for a long, continuous time, measured as a percentage of uptime. An **SLA** (service level agreement) is the provider's formal uptime promise to customers; the big cloud providers promise **99.9% or more**. Each extra nine cuts the allowed downtime by 10×:

| Availability | Downtime per day | Downtime per year |
|---|---|---|
| 99% | 14.4 minutes | 3.65 days |
| 99.9% | 1.44 minutes | 8.77 hours |
| 99.99% | 8.64 seconds | 52.6 minutes |
| 99.999% | 864 ms | 5.26 minutes |
| 99.9999% | 86.4 ms | 31.6 seconds |

## 🧮 Worked example: a Twitter-like service

**Assumptions** (made up for practice): 300 million monthly active users · 50% use it daily · 2 posts per user per day · 10% of posts include media (~1 MB each) · keep data for 5 years.

| Quantity | Calculation | Result |
|---|---|---|
| Daily active users | 300 M × 50% | **150 M** |
| Average write QPS | 150 M × 2 ÷ 86,400 s | **≈ 3,500** |
| Peak QPS | 2 × average | **≈ 7,000** |
| New media per day | 150 M × 2 × 10% × 1 MB | **30 TB** |
| Media for 5 years | 30 TB × 365 × 5 | **≈ 55 PB** |

The post ID (64 bytes) and text (140 bytes) are tiny next to the media, which is why only media storage is estimated.

## 💡 How to do it well

- **Round and approximate.** Nobody wants long division on a whiteboard. Turn 86,400 into "about 100,000" and keep moving.
- **Write your assumptions down** so you and the interviewer can refer back to them.
- **Label every unit.** "5" is ambiguous; "5 MB" isn't.
- **Practice the usual suspects:** QPS, peak QPS, storage, cache size, number of servers.

> 🧠 **Shortcuts worth memorizing:** a day has 86,400 s ≈ 10⁵ s, so **1 million requests per day ≈ 12 QPS**, and **1 KB × 1 million = 1 GB**.

## 🔗 Go deeper
- [Google Pro Tip: use back-of-the-envelope calculations to choose the best design](http://highscalability.com/blog/2011/1/26/google-pro-tip-use-back-of-the-envelope-calculations-to-choo.html)
- [Latency numbers every programmer should know (interactive, by year)](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [The System Design Primer](https://github.com/donnemartin/system-design-primer)

---

<div align="center">

[← Chapter 1](01-Scale-From-Zero-To-Millions-Of-Users.md) · [📚 All chapters](../README.md) · [Chapter 3 →](03-A-Framework-For-System-Design-Interviews.md)

</div>
