<div align="center">

[← Chapter 13: Design A Search Autocomplete System](13-Design-A-Search-Autocomplete-System.md) · [📚 All chapters](../README.md) · [Chapter 15: Design Google Drive →](15-Design-Google-Drive.md)

# Chapter 14 · Design YouTube

*Upload, transcode and stream video to millions of people, without the CDN bill eating the company.*

![YouTube](https://img.shields.io/badge/YouTube-FF0000?style=flat&logo=youtube&logoColor=white)
![Netflix](https://img.shields.io/badge/Netflix-E50914?style=flat&logo=netflix&logoColor=white)
![FFmpeg](https://img.shields.io/badge/FFmpeg-007808?style=flat&logo=ffmpeg&logoColor=white)
![Akamai](https://img.shields.io/badge/Akamai-0096D6?style=flat&logo=akamai&logoColor=white)
![Amazon S3](https://img.shields.io/badge/Amazon_S3-6e7681?style=flat)
![Amazon CloudFront](https://img.shields.io/badge/Amazon_CloudFront-6e7681?style=flat)

</div>

> **TL;DR:** Don't build what you can rent: use **blob storage** for files and a **CDN** for streaming. The interesting parts are the **upload pipeline** (original storage → **transcoding** into many formats and resolutions → transcoded storage → CDN, with a **completion queue** updating metadata) and the **transcoding engine**, modeled as a **DAG** of tasks run in parallel by a resource manager and workers. Then optimize: **chunked, parallel uploads**, **upload centers** near users, **pre-signed URLs**, **DRM/encryption/watermarks**, and the big one, **cost**: only popular videos go on the CDN.

---

## 1️⃣ Scope and estimates

YouTube is huge (in 2020: 2 billion monthly users and 5 billion videos watched a day), so narrow it down:

| Question | Answer |
|---|---|
| Features | **Uploading** and **watching** videos |
| Clients | Mobile apps, web browsers, smart TVs |
| Users | **5 million DAU**, 30 minutes a day on average, many international |
| Formats | Most resolutions and formats accepted |
| Encryption | Required |
| File size | Small and medium videos, **max 1 GB** |
| Build or rent? | Use existing **cloud services** (Amazon, Google, Microsoft) where possible |

Design goals: **fast uploads**, **smooth streaming**, **switchable quality**, a **small infrastructure bill**, and high availability, scalability and reliability.

| Estimate | Math | Result |
|---|---|---|
| New storage per day | 5 M × 10% upload × 300 MB | **150 TB/day** |
| Video streamed per day | 5 M × 5 videos × 0.3 GB | **7.5 PB/day** |
| CDN cost per day | 7.5 PB × $0.02 per GB (the book's assumed Amazon CloudFront rate, all traffic in the US) | **≈ $150,000/day** (about $55 M a year) |

That CDN bill is why cost optimization gets its own section.

## 2️⃣ High-level design

**Why rent instead of build?** An interview is about choosing the right tool, not re-deriving blob storage. And building a CDN or blob store is enormously hard: even Netflix runs on Amazon's cloud, and Facebook uses Akamai's CDN.

Three parts: the **client** (computer, phone, smart TV), the **CDN** (stores and streams the videos), and **API servers** (everything else: feed recommendations, upload URLs, metadata, sign-up…).

### Upload flow

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch14-upload-dark.svg">
  <img src="../images/ch14-upload-light.svg" width="100%" alt="Video upload flow. Flow a: 1 the client uploads the video to original blob storage; 2 transcoding servers fetch and transcode it; 3a the results go to transcoded storage and 3a.1 on to the CDN; in parallel 3b a completion event is queued, and 3b.1 a completion handler 3b.1a updates the metadata cache and 3b.1b the metadata database; 4 the API servers tell the client the video is ready. Flow b, at the same time: the client sends the video's metadata to the API servers, which save it.">
</picture>

| Component | Role |
|---|---|
| **Original storage** | Blob storage for uploaded files (a BLOB is binary data stored as one object) |
| **Transcoding servers** | Convert video into the formats and bitrates different devices and connections need |
| **Transcoded storage** | Blob storage for the results, which are pushed to the **CDN** |
| **Completion queue + handler** | Transcoding-done events; workers then update the **metadata cache and DB** (which is sharded and replicated) |

Two flows run **in parallel**: **(a)** the file itself goes through steps 1–4 above, while **(b)** the client sends the **metadata** (file name, size, format…) to the API servers.

### Streaming flow
**Streaming** means the device keeps receiving small pieces and plays them right away, instead of downloading the whole file first. Videos stream **straight from the nearest CDN edge**, so latency is low. A **streaming protocol** decides how the data flows: MPEG-DASH, Apple HLS, Microsoft Smooth Streaming, Adobe HDS. You don't need their details, just that they differ in supported encodings and players, so pick one that fits.

## 3️⃣ Deep dive

### Why transcode?
- **Raw video is huge:** an hour of HD at 60 fps can be **hundreds of GB**.
- **Compatibility:** devices and browsers support different formats.
- **Quality vs bandwidth:** send high resolution to fast connections and low resolution to slow ones…
- …and **switch automatically** as a phone's network changes.

An encoded file has two parts: the **container** (the "basket" holding video, audio and metadata, recognizable by extension: `.mp4`, `.mov`, `.avi`) and the **codec** (the compression algorithm: **H.264**, **VP9**, **HEVC**). The **bitrate** (bits per second) drives quality and the bandwidth needed.

### Transcoding as a DAG
Creators need different processing (watermarks, custom thumbnails, HD or not), and transcoding is slow, so it should be **configurable and parallel**. Following Facebook's streaming video engine, the work is a **directed acyclic graph (DAG)** of tasks that run in sequence or in parallel:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch14-dag-dark.svg">
  <img src="../images/ch14-dag-light.svg" width="100%" alt="A transcoding DAG. Stage 1 splits the original video into video, audio and metadata. Stage 2 runs tasks in parallel: the video is inspected, then encoded into several resolutions, given a thumbnail and watermarked, while the audio is encoded. Stage 3 assembles everything into output files such as clip_360p.mp4, clip_720p.mp4 and clip_1080p.mp4.">
</picture>

Typical tasks: **inspection** (is the file valid and of good quality?), **video encodings** (resolutions, codecs, bitrates), **thumbnails** (uploaded or generated), **watermarks** (an identifying image overlay).

### The transcoding engine

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch14-transcoder-dark.svg">
  <img src="../images/ch14-transcoder-light.svg" width="100%" alt="Video transcoding architecture. The preprocessor splits the video into GOP chunks and builds the DAG; the DAG scheduler turns it into stages of tasks in the resource manager's task queue. Inside the resource manager, the scheduler 1 takes the highest-priority task, 2 picks the best worker from the worker queue, 3 tells that task worker (video encoder, thumbnail or watermark) to run it, and 4 tracks it in the running queue until it finishes. Workers write the output files. Temporary storage keeps metadata in memory and video and audio in blob storage until the job is done.">
</picture>

| Component | Job |
|---|---|
| **Preprocessor** | Splits the video into **GOPs** (Groups of Pictures: independently playable chunks of a few seconds), also for old clients that can't split; builds the **DAG** from config files written by client programmers; **caches** GOPs and metadata in temporary storage so failed encodes can retry |
| **DAG scheduler** | Breaks the DAG into **stages** of tasks (e.g. stage 1: video, audio, metadata; stage 2: video encoding, thumbnail, audio encoding) and queues them |
| **Resource manager** | A **task queue** (by priority), a **worker queue** (by worker load), a **running queue**, and a **task scheduler** that matches the top task with the best worker, runs it, tracks it, and clears it when done |
| **Task workers** | Run the DAG's tasks; different workers do different jobs |
| **Temporary storage** | Small, hot metadata in memory; video and audio in blob storage; freed when processing ends |
| **Encoded video** | The final output, e.g. `clip_720p.mp4` |

### Optimizations

**Speed**
- **Chunked, parallel uploads:** the **client** splits the file by GOP, so chunks upload in parallel and a failed upload **resumes** instead of restarting.
- **Upload centers near users:** use **CDN** locations as upload points (North America, Asia…).
- **Parallelism everywhere:** put **message queues** between pipeline stages (download → encode → upload to CDN) so each stage works on whatever is ready instead of waiting for the previous one.

**Safety**
- **Pre-signed upload URLs:** the client asks the API servers for a URL that grants upload permission to one specific object, then uploads straight to storage with it. (Amazon S3 calls them pre-signed URLs; Azure calls them **Shared Access Signatures**.)
- **Protect creators' videos:** **DRM** (Apple FairPlay, Google Widevine, Microsoft PlayReady), **AES encryption** with an authorization policy (decrypted only at playback for authorized users), or a **visual watermark** (logo or name overlay).

**Cost**: video popularity follows a **long-tail** distribution:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch14-long-tail-dark.svg">
  <img src="../images/ch14-long-tail-light.svg" width="100%" alt="Schematic long-tail curve of video popularity (illustrative shape, not real data): a small head of very popular videos gets most views and is served from the CDN; the long tail of many rarely watched videos is served from the company's own video servers, encoded on demand, and kept in the regions where it is popular.">
</picture>

1. Serve **only popular videos from the CDN**; the rest from your own high-capacity **video servers**.
2. **Fewer encoded versions** for unpopular videos; encode short ones **on demand**.
3. Don't push **regionally popular** videos to other regions.
4. At very large scale, **build your own CDN** and partner with **ISPs** (as Netflix does with Open Connect): closer to viewers and cheaper bandwidth.

Base these choices on **historical viewing patterns**.

### Error handling
- **Recoverable** errors (a segment fails to transcode): **retry** a few times; if it keeps failing, return an error code.
- **Non-recoverable** errors (a malformed video): **stop** the video's tasks and return an error code.

| Failure | Playbook |
|---|---|
| Upload error | Try again, a limited number of times |
| Old client can't split by GOP | Send the whole file; split on the server |
| Transcoding error | Retry |
| Preprocessor error | Regenerate the DAG |
| DAG scheduler error | Reschedule the task |
| Resource manager queue down | Fail over to its replica |
| Task worker down | Run the task again on a different worker |
| API server down | Stateless, so another server takes the request |
| Metadata cache node down | Data is replicated; read from another node and replace the dead one |
| Metadata DB primary down | Promote a replica to primary |
| Metadata DB replica down | Read from another replica and replace the dead one |

## 4️⃣ Wrap-up talking points

- **Scale the API tier** horizontally (stateless) and the **database** with replication and sharding.
- **Live streaming** shares upload, encoding and streaming with this design, but needs **lower latency** (maybe another protocol), **less parallelism** (small chunks are processed in real time), and **faster error handling**.
- **Takedowns:** remove videos that violate copyright or the law, found at upload time or through user flags.

## 🔗 Go deeper
- [SVE: Distributed Video Processing at Facebook Scale (SOSP 2017)](https://www.cs.princeton.edu/~wlloyd/papers/sve-sosp17.pdf)
- [Netflix: Content Popularity for Open Connect](https://netflixtechblog.com/content-popularity-for-open-connect-b86d56f613b)
- [Amazon S3: uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [Azure: grant limited access with shared access signatures](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Amazon CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/)

---

<div align="center">

[← Chapter 13](13-Design-A-Search-Autocomplete-System.md) · [📚 All chapters](../README.md) · [Chapter 15 →](15-Design-Google-Drive.md)

</div>
