---
title: Pub/Sub and RSS Feeds
description: A guide to publish-subscribe messaging, fan-out, delivery behavior, and how RSS feeds fit the subscription model.
---

# Pub/Sub and RSS Feeds

**Publish/subscribe (Pub/Sub)** lets a sender announce that something happened
without knowing who will use the announcement. Publishers send messages to a
named **topic**; subscribers express interest in that topic and receive its
messages.

Think of a university announcement list. The professor posts once to
`course:system-design`; every student subscribed to that list gets a copy. The
professor does not call every student separately, and students can join or leave
without changing the professor's code.

## How it works

```{mermaid}
flowchart LR
  P[News service] -->|ArticlePublished| T[Topic: articles]
  T --> S1[Email subscription]
  T --> S2[Search subscription]
  T --> S3[Analytics subscription]
  S1 --> E[Send newsletter]
  S2 --> I[Update search index]
  S3 --> A[Count publications]
```

1. The news service publishes an `ArticlePublished` message to the `articles`
   topic.
2. The broker copies it to each independent subscription.
3. Email, search, and analytics process the same event at their own speed.
4. With a durable broker, a subscriber commonly acknowledges a message after
   processing it. An unacknowledged message may be retried.

The publisher is now **decoupled** from its subscribers. Adding a translation
service does not require redeploying the news service, and a temporary analytics
failure does not have to block publication.

```json
{
  "eventId": "evt_8472",
  "type": "ArticlePublished",
  "occurredAt": "2026-09-05T17:30:00Z",
  "articleId": "article_123"
}
```

Publish facts that already happened—`ArticlePublished`—rather than vague
instructions such as `DoWork`. Include a unique event ID so a subscriber can
recognize a retry and remain idempotent.

## Pub/Sub versus a queue

- A **queue** normally distributes jobs: several workers share one logical
  subscription, and one worker handles each job.
- **Pub/Sub** normally fans out events: each subscription gets its own copy, so
  several systems can react independently.

The patterns can be combined. One topic may fan an order event out to billing
and fulfillment subscriptions, while ten fulfillment workers compete for work
within the fulfillment subscription.

## When Pub/Sub is useful

- **Order placed:** notify billing, inventory, fulfillment, and analytics.
- **User updated:** invalidate caches and refresh a search index.
- **Database change:** distribute a change-data-capture event to downstream
  systems.
- **Live scores:** publish each score update to many regional delivery servers.
- **Notifications:** let email, push, and SMS services react independently.

Pub/Sub is less attractive when the caller needs an immediate answer. A price
lookup is usually a synchronous request; `OrderPlaced` side effects can happen
asynchronously after the order transaction succeeds.

## Redis Pub/Sub

[Redis Pub/Sub](../caching/redis.md) is the lightweight, live-broadcast version
of this pattern. One client subscribes to a channel, and any client can publish
a message that Redis immediately pushes to every currently connected
subscriber.

```text
# Subscriber
SUBSCRIBE events:orders

# Publisher
PUBLISH events:orders '{"type":"OrderPlaced","orderId":"order_123"}'
```

Channel names are strings and commonly use colon-separated names such as
`events:orders`; channels are separate from normal Redis keys. This is useful
for online-presence changes, best-effort cache invalidation, or broadcasting an
update across a fleet of WebSocket servers.

The key trade-off is **at-most-once delivery**: Redis does not save Pub/Sub
messages or wait for acknowledgments. If a subscriber is disconnected or fails
while handling a message, that message is gone. Use Redis Streams or another
durable broker when consumers must catch up, retry, or replay history.

## Delivery questions

Pub/Sub does not automatically mean that every message arrives once, in order,
and forever. Decide:

- **Durable or ephemeral:** Can an offline subscriber catch up later?
- **Delivery:** Can messages be lost, or can retries create duplicates?
- **Ordering:** Is order global, per topic, or only per partition/key?
- **Failure:** When are messages retried or moved to a dead-letter queue?
- **Backpressure:** What happens when publishers are faster than subscribers?

At-least-once delivery is common: a message may be delivered again when its
acknowledgment is late or lost. Subscribers should therefore be idempotent and
track a stable `eventId` when duplicate effects would be harmful.

## How RSS feeds fit

**RSS (Really Simple Syndication)** is an XML format for publishing a feed of
web content such as articles, podcasts, or release notes. A feed has a stable
URL, and each reader subscribes to the URLs it cares about.

```{mermaid}
flowchart LR
  W[Website] -->|writes new entries| F[RSS feed URL]
  R[RSS reader] -->|polls periodically| F
  F -->|returns feed items| R
  R --> U[Shows unread items]
```

RSS has the *idea* of publishers and subscribers, but ordinary RSS is not a
push broker:

- the website publishes by updating an XML document at a URL;
- the reader periodically fetches that URL;
- the reader uses an item's `guid`, link, or publication date to identify what
  is new; and
- the publisher usually does not know who subscribed.

```xml
<item>
  <title>Designing an event pipeline</title>
  <link>https://example.com/event-pipeline</link>
  <guid>article-123</guid>
  <pubDate>Sat, 05 Sep 2026 17:30:00 GMT</pubDate>
</item>
```

RSS is excellent for open, interoperable content syndication: one feed can work
with many independent readers without accounts or a vendor-specific SDK. The
trade-off is freshness. If a reader polls every 15 minutes, an update can be
roughly 15 minutes late. HTTP cache validators such as `ETag` and
`Last-Modified` help avoid downloading an unchanged feed repeatedly.

For faster web-feed delivery, **WebSub** adds a hub. A subscriber registers a
callback URL, the publisher tells the hub when the feed changes, and the hub
pushes an HTTP notification to subscribers. It is closer to broker-style
Pub/Sub, but it requires a reachable subscriber endpoint.

## Choosing the delivery path

- Use a **message broker** for low-latency events between applications you
  operate, especially when you need durable subscriptions, retries, and access
  control.
- Use **RSS** for public articles, podcasts, changelogs, or other content that
  should work across independent readers and can tolerate polling delay.
- Use **WebSub** when feed subscribers expose callbacks and need faster updates.
- Use a broker behind the service plus **SSE or WebSocket** at the edge when
  browsers need immediate updates; browsers usually should not connect directly
  to an internal broker.

## Further reading

- [Google Cloud: Pub/Sub service overview](https://docs.cloud.google.com/pubsub/docs/pubsub-basics)
- [Redis: Pub/Sub](https://redis.io/docs/latest/develop/pubsub/)
- [Redis: Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [RSS Advisory Board: RSS 2.0 specification](https://www.rssboard.org/rss-specification)
- [W3C: WebSub](https://www.w3.org/TR/websub/)
