---
title: Pub/Sub and RSS Feeds
description: A guide to publish-subscribe messaging, fan-out, delivery behavior, and how RSS feeds fit the subscription model.
---

# Pub/Sub and RSS Feeds

**Publish/subscribe (Pub/Sub)** lets a sender announce something without knowing
who's listening. Publishers send messages to a **topic**; every subscriber to
that topic gets a copy.

## Example: a news site

```{mermaid}
flowchart LR
  P[News service] -->|ArticlePublished| T[Topic: articles]
  T --> S1[Email service]
  T --> S2[Search service]
  T --> S3[Analytics service]
```

The news service publishes once:

```json
{ "eventId": "evt_8472", "type": "ArticlePublished", "articleId": "article_123" }
```

- Email sends a newsletter, search indexes the article, and analytics counts it,
  each at its own pace.
- Adding a translation service means adding one more subscriber. The news
  service doesn't change.
- If analytics is down, publishing still works.

Name events as facts (`ArticlePublished`), and include an `eventId` so
subscribers can ignore duplicates.

## When to use it

- `OrderPlaced` → billing, inventory, email, analytics
- `UserUpdated` → clear caches, update search
- Live scores → many WebSocket servers

Not for requests that need an immediate answer, like "what's the price?". Use a
normal API call for those.

## Redis Pub/Sub

[Redis](../caching/redis.md) has a lightweight version:

```text
SUBSCRIBE events:orders                                     # subscriber
PUBLISH events:orders '{"type":"OrderPlaced","id":"o_123"}'  # publisher
```

It's **fire-and-forget**: Redis doesn't store messages. A subscriber that's
disconnected misses them for good. That's fine for live updates between
WebSocket servers or presence changes. If subscribers must catch up after
downtime, use Redis Streams, Kafka, or a managed service like Google Pub/Sub or
SNS + SQS.

## Questions to ask

- Can an offline subscriber catch up later, or are missed messages lost?
- Can messages arrive twice? (Usually yes, so subscribers must be idempotent.)
- Does order matter: globally, per key, or not at all?
- What happens to messages that keep failing?

## How RSS fits

**RSS** is a subscription idea for web content: a site publishes an XML feed at
a URL, and readers check it periodically.

```xml
<item>
  <title>Designing an event pipeline</title>
  <link>https://example.com/event-pipeline</link>
  <guid>article-123</guid>
  <pubDate>Sat, 05 Sep 2026 17:30:00 GMT</pubDate>
</item>
```

```{mermaid}
flowchart LR
  W[Website] -->|updates| F[Feed URL]
  R[RSS reader] -->|"polls every 15 min"| F
```

It looks like Pub/Sub, but it's really **polling**: the publisher just updates a
file, readers fetch it on a timer and use each item's `guid` to spot new ones,
and the publisher doesn't know who's subscribed. Updates can be up to one poll
interval late.

**WebSub** adds push to feeds: a hub notifies subscribers' callback URLs when
the feed changes.

## Which to use

- **Between your own services:** a message broker (Kafka, SNS/SQS, Google
  Pub/Sub).
- **Public content** (blogs, podcasts, changelogs): RSS.
- **To browsers:** a broker behind your servers, then SSE or WebSockets to the
  browser. Browsers shouldn't connect to your broker directly.

## Further reading

- [Google Cloud: Pub/Sub service overview](https://docs.cloud.google.com/pubsub/docs/pubsub-basics)
- [Redis: Pub/Sub](https://redis.io/docs/latest/develop/pubsub/)
- [RSS Advisory Board: RSS 2.0 specification](https://www.rssboard.org/rss-specification)
- [W3C: WebSub](https://www.w3.org/TR/websub/)
