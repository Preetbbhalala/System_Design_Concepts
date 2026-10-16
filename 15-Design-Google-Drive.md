<div align="center">

[← Chapter 14: Design YouTube](14-Design-YouTube.md) · [📚 All chapters](../README.md) · [Chapter 16: The Learning Continues →](16-The-Learning-Continues.md)

# Chapter 15 · Design Google Drive

*Store everyone's files safely, keep every device in sync, and send as few bytes as possible.*

![Google Drive](https://img.shields.io/badge/Google_Drive-4285F4?style=flat&logo=googledrive&logoColor=white)
![Dropbox](https://img.shields.io/badge/Dropbox-0061FF?style=flat&logo=dropbox&logoColor=white)
![iCloud](https://img.shields.io/badge/iCloud-3693F3?style=flat&logo=icloud&logoColor=white)
![OneDrive](https://img.shields.io/badge/OneDrive-6e7681?style=flat)
![Amazon S3](https://img.shields.io/badge/Amazon_S3-6e7681?style=flat)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)

</div>

> **TL;DR:** Grow from one server to a real design. Files are split into **blocks** that **block servers** compress, encrypt and store in **cloud storage** (S3, replicated across regions); edits upload only the **changed blocks** (delta sync). A **strongly consistent** relational **metadata DB** (with a cache) tracks users, devices, files, versions and blocks. A **notification service** using **long polling** tells other devices to pull changes, and an **offline backup queue** catches up devices that were away. Save space with **block dedupe**, **version limits** and **cold storage**.

---

## 1️⃣ Scope and estimates

**In scope:** add files (drag and drop), download files, **sync across devices**, view **revisions**, **share** files, and **notify** users whenever someone edits, deletes or shares a file with them. **Out of scope:** real-time collaborative editing like Google Docs.

| Question | Answer |
|---|---|
| Clients | Mobile and web |
| File types | Any |
| Encryption | Files must be encrypted in storage |
| Max file size | 10 GB |
| Users | 50 M signed up, **10 M DAU** |

**Non-functional:** **reliability** (losing data is unacceptable), **fast sync**, **low bandwidth** (think of mobile data plans), **scalability**, **high availability**.

| Estimate | Math | Result |
|---|---|---|
| Space to allocate | 50 M users × 10 GB free | **500 PB** |
| Upload QPS | 10 M × 2 uploads ÷ 86,400 s (average file 500 KB) | **≈ 230** (the book rounds to ~240) |
| Peak upload QPS | 2 × average | **≈ 480** |

Reads and writes are about 1:1.

## 2️⃣ Start with one server, then grow

**Version 0:** one machine running **Apache** (the web server) and **MySQL** (for metadata), plus 1 TB of disk. Files live under a `drive/` directory with one **namespace** (folder) per user, keeping their original names, so `namespace + relative path` identifies every file:

```text
drive/
├── alice/
│   └── photos/2026/lake.jpg
└── bob/
    └── taxes/2025/return.pdf
```

**APIs** (all authenticated, all over HTTPS):

| API | Purpose | Example parameters |
|---|---|---|
| `POST /files/upload?uploadType=resumable` | Upload; **simple** for small files, **resumable** for large files on flaky networks | `data`: the local file |
| `GET /files/download` | Download a file | `{"path": "/photos/2026/lake.jpg"}` |
| `GET /files/list_revisions` | A file's history | `{"path": "/taxes/2025/return.pdf", "limit": 20}` |

A **resumable upload** has three steps: request a resumable upload URL → upload while tracking progress → if interrupted, **resume** from where it stopped.

**Growing pains.** The disk fills up, so you **shard by `user_id`** across storage servers. Then you worry about losing data when a server dies, and move files to **Amazon S3**, which can **replicate within a region or across regions** (a *bucket* is like a top-level folder). Next: add a **load balancer** and more **web servers**, move the **metadata DB** off the box with replication and sharding, and keep files **replicated in two regions**. Web servers, metadata and file storage are now decoupled.

**Sync conflicts.** When two people change one file simultaneously, the version **processed first wins** and the other gets a **conflict**. That user is shown **both copies** (their local one and the latest from the server) and can **merge** them or **keep one**.

## 🏗️ High-level design

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch15-architecture-dark.svg">
  <img src="../images/ch15-architecture-light.svg" width="100%" alt="Google Drive high-level design. The user's app sends file blocks through block servers, which chunk, compress and encrypt them into cloud storage such as S3, with inactive files moved to cold storage. Metadata requests go through a load balancer to API servers, backed by a metadata cache and a metadata database. When a file changes, the notification service tells clients over long polling to pull the change, and an offline backup queue holds changes for clients that are offline.">
</picture>

| Component | Role |
|---|---|
| **Block servers** | Split files into blocks (max **4 MB**, following Dropbox), each with a **hash** recorded in the metadata DB, and upload them to cloud storage. A file is rebuilt by joining its blocks in order. |
| **Cloud storage** | Holds the blocks |
| **Cold storage** | Cheaper storage for files nobody has touched in a long time |
| **API servers** (behind a load balancer) | Everything except the upload itself: authentication, profiles, file metadata… |
| **Metadata DB + cache** | Users, files, blocks, versions; the hot parts are cached. Files themselves are never in the DB. |
| **Notification service** | Publish/subscribe: whenever another device adds, edits or removes a file, it tells the affected clients to pull the change |
| **Offline backup queue** | Holds changes for offline clients until they reconnect |

## 3️⃣ Deep dive

### Block servers: send less data

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch15-blocks-dark.svg">
  <img src="../images/ch15-blocks-light.svg" width="100%" alt="Block server processing. A new file is split into blocks of at most 4 MB, each block is compressed (for example with gzip or bzip2) and encrypted, then uploaded to cloud storage. When a file is edited, delta sync compares block hashes: here only blocks 3 and 6 have new hashes, so only those 2 of the 8 blocks are uploaded.">
</picture>

- **Delta sync:** when a file changes, only the **modified blocks** are uploaded (using an rsync-style algorithm), not the whole file.
- **Compression** per block, picked by file type: gzip or bzip2 for text, other codecs for images and video.
- **Encryption** of every block before it leaves the block server.

### Strong consistency
Different clients must never see different versions of a file at the same time, so the **metadata cache and DB must be strongly consistent**. Caches are eventually consistent by default, so keep **cache replicas consistent with the primary** and **invalidate the cache on every DB write**. A **relational database** gives you **ACID** out of the box (NoSQL would need it in application logic), so that's the choice.

### Metadata schema (simplified)

| Table | Holds |
|---|---|
| `user` | Name, email, profile photo… |
| `device` | A user's devices, with a `push_id` for mobile push notifications |
| `namespace` | Each user's root directory |
| `file` | Everything about the **latest** version of a file |
| `file_version` | Version history; rows are **read-only** to protect it |
| `block` | Every block; joining a version's blocks in order rebuilds that version |

### Upload flow

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch15-upload-seq-dark.svg">
  <img src="../images/ch15-upload-seq-light.svg" width="100%" alt="Upload sequence with two parallel flows from client 1. Metadata flow: 1 client 1 sends the new file's metadata to the API servers, 2 which store it with status pending, 3 tell the notification service, and 4 client 2 learns a file is being uploaded. File content flow: 2.1 client 1 sends the content to block servers, 2.2 which upload blocks to cloud storage, 2.3 cloud storage calls back the API servers, 2.4 the status becomes uploaded, 2.5 the notification service is told, and 2.6 client 2 learns the file is ready.">
</picture>

Two requests leave client 1 **in parallel**: the **metadata** (saved with status `pending`, and other clients are told an upload has started) and the **file content** (through block servers to cloud storage, whose completion callback flips the status to `uploaded` and triggers a "ready" notification). Edits follow the same path.

### Download flow
A client learns about a change in one of two ways: if it's **online**, the notification service tells it; if it was **offline**, the changes waited for it and it pulls them when it reconnects. Then it **asks the API servers for the new metadata** (read from the metadata DB), and **downloads the new blocks** through the block servers (which fetch them from cloud storage), and rebuilds the file.

### Notification service: long polling
**WebSocket** would work, but **long polling** (which Dropbox uses) fits better: notifications only go **server → client**, and they're **infrequent**, with no bursts. Each client keeps a long-poll request open; when a change arrives the connection closes, the client fetches the change from the metadata servers, then immediately opens a new long poll (it does the same after a timeout).

### Saving storage space
Keeping many versions in several data centers fills disks fast:
- **Dedupe blocks** within an account: two blocks with the **same hash** are identical, so store one.
- **Smarter backups:** **cap the number of versions** (the oldest is dropped when the cap is hit), and **keep only valuable versions**: a heavily edited file could otherwise produce 1,000+ versions in a short time, so favor recent ones and tune the count by experiment.
- **Cold storage** for data untouched for months or years, e.g. **Amazon S3 Glacier**, which costs far less than standard S3.

### Failure handling

| Failure | What happens |
|---|---|
| Load balancer | A standby takes over; load balancers watch each other's **heartbeats** |
| Block server | Other block servers pick up its unfinished or pending jobs |
| Cloud storage | S3 is replicated across regions; read from another region |
| API server | Stateless; the load balancer sends traffic elsewhere |
| Metadata cache | Replicated; read another node and replace the dead one |
| Metadata DB | Primary down: promote a replica (and add a new one). Replica down: read from another, replace it |
| Notification server | Each holds a huge number of long-poll connections (Dropbox reported over 1 million per machine in 2012). Clients must reconnect elsewhere, and reconnecting that many is slow |
| Offline backup queue | Queues are replicated; consumers switch over by subscribing to the replica |

## 4️⃣ Wrap-up talking points

The design balances **strong consistency**, **low bandwidth** and **fast sync** across two flows (file metadata and file sync), with long-polling notifications in between. Alternatives worth discussing:

- **Upload straight from the client to cloud storage?** It's faster (one transfer instead of two), but the chunking, compression and **encryption** logic would have to be rebuilt for iOS, Android and web (error-prone), and a client can be hacked, so client-side encryption is risky. Centralizing it in block servers avoids both.
- **Split presence into its own service**, so other services can reuse online/offline status.

## 🔗 Go deeper
- [Google Drive API: upload file data (simple and resumable uploads)](https://developers.google.com/drive/api/guides/manage-uploads)
- [Amazon S3: replicating objects across buckets and regions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [The rsync algorithm (Tridgell and Mackerras)](https://rsync.samba.org/tech_report/)
- [Differential Synchronization, by Neil Fraser](https://neil.fraser.name/writing/sync/)
- [Amazon S3 Glacier storage classes](https://aws.amazon.com/s3/storage-classes/glacier/)

---

<div align="center">

[← Chapter 14](14-Design-YouTube.md) · [📚 All chapters](../README.md) · [Chapter 16 →](16-The-Learning-Continues.md)

</div>
