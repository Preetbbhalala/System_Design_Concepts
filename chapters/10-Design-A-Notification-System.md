<div align="center">

[← Chapter 9: Design A Web Crawler](09-Design-A-Web-Crawler.md) · [📚 All chapters](../README.md) · [Chapter 11: Design A News Feed System →](11-Design-A-News-Feed-System.md)

# Chapter 10 · Design A Notification System

*Send 16 million push notifications, texts and emails a day, and never lose one.*

![Apple Push Notification service](https://img.shields.io/badge/Apple_Push_%28APNs%29-000000?style=flat&logo=apple&logoColor=white)
![Firebase Cloud Messaging](https://img.shields.io/badge/Firebase_Cloud_Messaging-DD2C00?style=flat&logo=firebase&logoColor=white)
![Twilio](https://img.shields.io/badge/Twilio-6e7681?style=flat)
![SendGrid](https://img.shields.io/badge/SendGrid-6e7681?style=flat)
![Mailchimp](https://img.shields.io/badge/Mailchimp-FFE01B?style=flat&logo=mailchimp&logoColor=black)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=flat&logo=rabbitmq&logoColor=white)

</div>

> **TL;DR:** Your services don't talk to Apple, Google, Twilio or SendGrid directly. They call **notification servers**, which check auth, rate limits and user settings, then drop each message onto a **queue per channel**. **Workers** drain the queues and call the **third-party providers**, which deliver to devices. A **notification log** plus **retries** guarantee nothing is lost, an **event-ID dedupe** keeps duplicates rare, and **templates, settings, monitoring and analytics** round it out.

---

## 1️⃣ Scope

| Question | Answer |
|---|---|
| Channels | **Push** (iOS and Android), **SMS**, **email** |
| Real-time? | **Soft** real-time: as fast as possible, small delays are OK under heavy load |
| Devices | iOS, Android, laptop/desktop |
| Triggers | Client apps, or jobs **scheduled** on the server side |
| Opt-out | Supported: opted-out users get nothing |
| Volume per day | 10 M push + 1 M SMS + 5 M email = **16 M** (≈ 185 per second on average) |

## 2️⃣ How each channel works

Every channel follows the same shape: **your provider code → a third-party service → the user's device.**

| Channel | Third-party service | You send it |
|---|---|---|
| iOS push | **APNs** (Apple Push Notification service) | A **device token** (unique ID for the device) + a JSON **payload** |
| Android push | **FCM** (Firebase Cloud Messaging) | Same idea: token + payload |
| SMS | **Twilio**, **Nexmo** (now Vonage)… | Phone number + text |
| Email | **SendGrid**, **Mailchimp**, or your own mail servers | Address + content; commercial services give better delivery rates and analytics |

An iOS payload looks like this:

```json
{
  "aps": {
    "alert": { "title": "Your order has shipped", "body": "Package #1042 arrives tomorrow." },
    "badge": 1,
    "sound": "default"
  }
}
```

> 🌏 Providers aren't available everywhere: FCM doesn't work in China, where services like JPush or Pushy fill the gap. Design so a provider can be **plugged in or out** easily.

**Collecting contact info.** When a user installs the app or signs up, the API servers save their contact details: **email and phone** on the `user` row, **device tokens** in a `device` table. One user can have **many devices**, and a push goes to all of them.

## 3️⃣ From one server to a real design

**First draft:** services → one notification server → providers. It has three problems: a **single point of failure**, it's **hard to scale** (database, cache and processing can't grow independently), and it's a **performance bottleneck** (rendering HTML and waiting on providers is slow, especially at peak).

**Improved design:** give the **database and cache** their own servers, run **many notification servers** with auto-scaling, and put **message queues** between the components.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch10-architecture-dark.svg">
  <img src="../images/ch10-architecture-light.svg" width="100%" alt="Notification system design. 1 Services 1 to N call the notification servers, which handle authentication, rate limiting and validation; 2 the servers read users, devices, settings and templates from a cache and database; 3 they put each notification on a queue for its channel (iOS push, Android push, SMS, email); 4 workers pull from the queues; 5 workers call the providers APNs, FCM, Twilio and SendGrid; 6 the providers deliver to the user's devices. Workers persist every notification to a notification log so failures can be retried, and delivery events flow to an analytics service.">
</picture>

| Component | Role |
|---|---|
| **Services 1…N** | Anything that triggers a notification: a microservice, a cron job, a billing reminder, a "your package arrives tomorrow" SMS |
| **Notification servers** | Internal-only (or verified-client-only) API to **prevent spam**; validate emails and phone numbers; fetch the data needed to render; enqueue |
| **Cache + DB** | Users, devices, settings and templates (cached), and notification records |
| **One queue per channel** | Buffers traffic spikes, and **isolates failures**: if one provider is down, the other channels keep flowing |
| **Workers** | Pull events from the queues and call the matching provider |

A send request might look like this:

```http
POST /v1/notifications
Content-Type: application/json

{
  "user_id": 81623,
  "channel": "email",
  "template_id": "order_shipped",
  "params": { "order_id": "1042", "eta": "tomorrow" }
}
```

## 🔍 Deep dive

### Reliability
- **Never lose a notification.** Delays and reordering are acceptable; loss is not. Workers persist each notification in a **notification log** database and **retry** failures.
- **Exactly once? No.** Delivery is exactly-once most of the time, but a distributed system can send duplicates. Reduce them with a **dedupe check**: when an event arrives, look up its **event ID**; if it's been seen, drop it.

### Everything else a real system needs

| Feature | What it does |
|---|---|
| **Templates** | Reusable, pre-formatted messages with parameters, e.g. `Hi {first_name}, order {order_id} ships today. Track it: {link}`. Consistent format, fewer mistakes, less time. |
| **User settings** | A `notification_setting` table (`user_id`, `channel`, `opt_in`). **Check it before every send.** |
| **Rate limiting** | Cap how many notifications one user gets; too many and they'll switch them all off ([Chapter 4](04-Design-A-Rate-Limiter.md)) |
| **Retry** | If a provider fails, put the notification back on the queue; if it keeps failing, **alert the developers** |
| **Security** | Push APIs are protected with an **appKey / appSecret** pair; only authenticated clients may send |
| **Queue monitoring** | Watch the **number of queued notifications**; if it keeps growing, workers can't keep up, so add more |
| **Event tracking** | Feed open rate, click rate and engagement into an **analytics** service |

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../images/ch10-events-dark.svg">
  <img src="../images/ch10-events-light.svg" width="100%" alt="Notification events tracked for analytics: start, pending, sent, delivered and clicked. If the provider fails, the notification goes to error and is retried from pending, with an alert to developers after repeated failures. A delivered notification can also lead the user to unsubscribe.">
</picture>

**The updated design** therefore adds **authentication and rate limiting** to the notification servers, a **retry loop** (failed events go back on the queue for a set number of attempts), **templates**, and **monitoring and tracking**.

## 4️⃣ Wrap-up

- **Reliability:** persistent log + retries keep the failure rate low.
- **Security:** appKey/appSecret so only verified clients can send.
- **Tracking and monitoring** at every stage of the flow.
- **Respect user settings:** check opt-in before sending.
- **Rate limiting:** users appreciate a frequency cap.
- **Message queues** decouple everything, which is the single biggest idea in the chapter.

## 🔗 Go deeper
- [Apple: Sending notification requests to APNs](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns)
- [Firebase Cloud Messaging documentation](https://firebase.google.com/docs/cloud-messaging)
- [You Cannot Have Exactly-Once Delivery (Brave New Geek)](https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/)
- [Twilio Messaging docs](https://www.twilio.com/docs/messaging) · [SendGrid docs](https://www.twilio.com/docs/sendgrid)

---

<div align="center">

[← Chapter 9](09-Design-A-Web-Crawler.md) · [📚 All chapters](../README.md) · [Chapter 11 →](11-Design-A-News-Feed-System.md)

</div>
