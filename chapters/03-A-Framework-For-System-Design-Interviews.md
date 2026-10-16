<div align="center">

[← Chapter 2: Back-of-the-envelope Estimation](02-Back-Of-The-Envelope-Estimation.md) · [📚 All chapters](../README.md) · [Chapter 4: Design A Rate Limiter →](04-Design-A-Rate-Limiter.md)

# Chapter 3 · A Framework For System Design Interviews

*A repeatable 4-step process for an open-ended, 45-minute design conversation.*

</div>

> **TL;DR:** A system design interview isn't a trivia quiz with one right answer. It's a simulation of two engineers working through a vague problem together. The interviewer is judging *how* you get to a design: clarifying requirements, collaborating, weighing trade-offs, and handling feedback. The chapter gives a 4-step framework and a time budget that every later chapter (4–15) follows.

---

## 👀 What the interviewer is really looking for

| ✅ Signals they want | 🚩 Red flags |
|---|---|
| Asking **good questions** to remove ambiguity | Jumping to a solution before understanding the problem |
| **Collaborating** like a teammate | **Over-engineering**: chasing design purity while ignoring trade-offs and cost |
| Staying calm and productive **under pressure** | Narrow-mindedness and stubbornness |
| **Defending** choices and **responding to feedback** | Going silent while thinking |

## 🪜 The 4-step process

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch03-framework-dark.svg">
  <img src="../images/ch03-framework-light.svg" width="100%" alt="The 4-step framework for a 45-minute interview: 1 understand the problem (3–10 min), 2 propose a high-level design (10–15 min), 3 design deep dive (10–25 min), 4 wrap up (3–5 min).">
</picture>

### Step 1: Understand the problem and set the scope *(3–10 min)*
Don't be the student who blurts out the first answer. Answering fast without clear requirements is a **red flag**. Slow down and ask:

- What **features** are we building, exactly?
- How many **users**, and how fast is the company expecting to grow (3 months, 6 months, a year)?
- What's the existing **tech stack**, and which services can we reuse?

If the interviewer tells you to make your own assumptions, **write them down**; you'll need them later.

<details>
<summary><b>Example: scoping a news feed</b></summary>

| You ask | Interviewer says |
|---|---|
| Mobile, web, or both? | Both |
| Most important features? | Publishing a post and seeing friends' posts |
| Feed order? | Reverse chronological, to keep it simple |
| Friends per user? | Up to 5,000 |
| Traffic? | 10 million daily active users |
| Media in posts? | Yes, images and videos |

</details>

### Step 2: Propose a high-level design and get buy-in *(10–15 min)*
- Draft a **blueprint** and treat the interviewer as a teammate: share it and ask for feedback.
- Draw the **key boxes**: clients, APIs, web servers, caches, a CDN, databases, message queues…
- Run **back-of-the-envelope numbers** ([Chapter 2](02-Back-Of-The-Envelope-Estimation.md)) to check that the blueprint fits the scale. Think out loud.
- Walk through a few **concrete use cases**; they often expose edge cases.
- API endpoints and database schema? It depends: too low-level for "design Google search", fair game for "design a poker game backend". Ask.

### Step 3: Design deep dive *(10–25 min)*
By now you agree on the goals and scope, you have a blueprint, and you've had feedback. With the interviewer, pick the **most important components** and go deep. What matters differs by problem: the **hash function** for a URL shortener, **latency and online/offline presence** for a chat system. Some interviewers want bottlenecks and resource estimates (common for senior roles). Manage your time: deep detail on something like a ranking algorithm burns minutes without showing design skill.

### Step 4: Wrap up *(3–5 min)*
- Name the **bottlenecks** and how you'd improve them. Never claim your design is perfect.
- Give a short **recap**, especially if you discussed several options.
- Discuss **error cases**: server failures, network loss.
- Cover **operations**: metrics, logs, how you'd roll the system out.
- Explain what changes for the **next scale curve**, e.g., from 1 million to 10 million users.
- Mention the refinements you'd make with more time.

---

## ✔️ Dos and ❌ don'ts

| Do | Don't |
|---|---|
| Ask for clarification; don't assume your assumptions are right | Show up unprepared for the classic questions |
| Understand the requirements; a startup and a giant need different designs | Jump into a solution before clarifying requirements |
| Think out loud and keep the interviewer in the loop | Dive into one component's details before the big picture |
| Offer more than one approach when you can | Hesitate to ask for a hint when you're stuck |
| Agree on the blueprint, then design the **most critical parts first** | Think in silence |
| Bounce ideas off the interviewer | Assume you're done once you've drawn the design; you're done when they say so |
| Never give up | Wait until the end to ask for feedback; ask early and often |

> 🧭 **How to use the rest of the book:** chapters 4–15 are worked examples of exactly these four steps. Read each one by asking "what did Step 1 decide, what's the Step 2 blueprint, and which parts got the Step 3 deep dive?"

---

<div align="center">

[← Chapter 2](02-Back-Of-The-Envelope-Estimation.md) · [📚 All chapters](../README.md) · [Chapter 4 →](04-Design-A-Rate-Limiter.md)

</div>
