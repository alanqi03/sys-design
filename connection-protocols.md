---
title: Real Time Updates
description: A practical guide to Pub/Sub, RSS feeds, polling, webhooks, Server-Sent Events, WebSockets, and WebRTC.
---

# Real Time Updates

Applications need different ways to learn about new data. A job-status page can
check occasionally, a dashboard needs server-driven updates, chat needs both
sides to speak at any time, and a video call must carry latency-sensitive media.

This section covers both **delivery patterns** and **connection choices**.
Pub/Sub describes who receives an update, while polling, webhooks, SSE,
WebSocket, and WebRTC describe how applications move updates across a network.
RSS applies a subscription idea to public web content, usually through periodic
polling.

## Quick comparison

| Choice | Communication | Best fit | Main trade-off |
| --- | --- | --- | --- |
| [Pub/Sub and RSS](connection-protocols/pub-sub.md) | Publishers emit updates without calling each subscriber directly | Fan-out events and syndicated web content | Delivery, replay, and freshness depend on the broker or feed reader |
| [Polling](connection-protocols/polling.md#polling) | Client requests, server responds | Rare or delay-tolerant updates | Empty requests waste work; interval adds latency |
| [Long polling](connection-protocols/polling.md#long-polling) | Server delays each HTTP response | Near-real-time updates with broad HTTP compatibility | Reconnect and HTTP overhead after every response |
| [Webhooks](connection-protocols/polling.md#webhooks) | Provider calls a consumer's HTTPS endpoint | Server-to-server event notifications | Receiver must be reachable, secure, and idempotent |
| [SSE](connection-protocols/server-sent-events.md) | Server streams text events to client | Notifications, dashboards, progress, AI text output | Server-to-client only; text format |
| [WebSocket](connection-protocols/websockets.md) | Full-duplex messages | Chat, collaboration, multiplayer state | Stateful connections and recovery are more complex |
| [WebRTC](connection-protocols/webrtc.md) | Peer media and/or data | Voice, video, screen sharing, peer data | Signaling, NAT traversal, relays, and group topology |

## Choosing a client connection

After deciding which subscribers should receive an update, choose how the last
hop reaches the client:

```{mermaid}
flowchart TD
  start{What must move in real time?}
  start -->|Peer audio, video, or screen| rtc[WebRTC]
  start -->|Application data| both{Must both sides send often?}
  both -->|Yes| ws[WebSocket]
  both -->|No, mostly server to client| push{Are updates frequent or latency-sensitive?}
  push -->|Yes| sse[SSE]
  push -->|Sometimes, and HTTP compatibility matters| lp[Long polling]
  push -->|No| poll[Polling]
```

This is a starting point, not a rule. A WebRTC application commonly uses
WebSocket for signaling, and an SSE application uses ordinary HTTP requests for
client commands. For server-to-server notifications, use a webhook when the
provider can reach the consumer; otherwise, the consumer can poll. Prefer the
simplest option that meets the latency, direction, browser support,
infrastructure, and scale requirements.

## Managing Connections

With SSE or WebSockets, a client keeps a connection open to **one** gateway.
That gateway knows which users or chat rooms its local connections belong to;
another gateway does not automatically know about those sockets. For example,
Alice and Bob may be in the same chat room but connected to different gateways.

```{mermaid}
flowchart LR
  Alice[Alice] -->|connect| LB[Ingress]
  Bob[Bob] -->|connect| LB
  LB -->|Alice's user ID| GA[Gateway A]
  LB -->|Bob's user ID| GB[Gateway B]
  GA -->|send message| Chat[Chat service]
  Chat --> DB[(Messages database)]
  Chat -->|publish room:42 update| Redis[Redis Pub/Sub]
  Redis --> GA
  Redis --> GB
  GA -->|push update| Alice
  GB -->|push update| Bob
```

### Consistent hashing, ELI18

Think of **consistent hashing** as a seating chart for new connections: hash a
stable value such as `userId`, then send that user's connection to its assigned
gateway. If a gateway is added or removed, only some users get a new assignment
instead of reshuffling everyone. The ingress must actually route using that
stable key; an ordinary round-robin load balancer does not do this by itself.

This is **optional**. A WebSocket stays on the gateway that accepted it even
without consistent hashing. Hashing can make placement predictable, but it does
not copy a socket to another gateway or deliver a message to everyone in a
room. Existing connections stay put until they disconnect; after a gateway
failure, clients reconnect and resubscribe. Watch for uneven load from very
active users.

### Redis Pub/Sub, ELI18

Redis Pub/Sub is the **intercom between gateways**. Each gateway with local
listeners for `room:42` subscribes to that channel. After the chat service
saves a message, it publishes an update (often a message ID) to `room:42`.
Redis sends the update to subscribed gateways, and each gateway pushes it to
its own connected clients. This works whether connections were placed by
consistent hashing or by a regular load balancer.

Pub/Sub is for **live notification, not message storage**: a disconnected
gateway misses updates, and Redis does not replay them. Keep chat messages in
the database and fetch missed messages on reconnect. If downstream processing
must never miss an event, use an outbox plus a durable stream or queue instead.

## Questions to ask

- Does data flow client-to-server, server-to-client, or both?
- How stale may an update be: minutes, seconds, or milliseconds?
- Is the payload occasional JSON, a continuous event feed, or live media?
- How many simultaneous connections must gateways and servers hold?
- What happens on reconnect: resume, replay, deduplicate, or accept data loss?
- Can proxies, load balancers, and firewalls carry long-lived connections?

## Further reading

- [RFC 6202: HTTP long polling and streaming](https://www.rfc-editor.org/rfc/rfc6202.html)
- [HTML Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [W3C WebRTC specification](https://www.w3.org/TR/webrtc/)
- [NGINX: consistent-hash upstream routing](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash)
- [Redis: Pub/Sub delivery semantics](https://redis.io/docs/latest/develop/pubsub/)
