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

| Choice | How it communicates | Direction | Typical delay | Best fit | Main trade-off |
| --- | --- | --- | --- | --- | --- |
| [Polling](connection-protocols/polling.md#polling) | Client sends a new request on a timer (say, every 5 s); server answers right away, even if nothing changed | Client asks, server answers | Up to one interval | Job status, delay-tolerant dashboards | Most requests come back empty; constant load even when idle |
| [Long polling](connection-protocols/polling.md#long-polling) | Client sends a request; server holds it open until there's news or a timeout, then the client immediately asks again | Server → client (client sends commands as normal requests) | Near real time | Live updates where SSE or WebSockets aren't available | A new request after every response; many open requests waiting on the server |
| [SSE](connection-protocols/server-sent-events.md) | Client opens one HTTP connection; server keeps it open and streams text events down it. The browser reconnects on its own | Server → client only | Real time | Notifications, live scores, progress, AI text streaming | One way only; text only; one open connection per client |
| [WebSocket](connection-protocols/websockets.md) | One HTTP handshake upgrades to a persistent connection; either side sends messages at any time | Both ways | Real time | Chat, collaborative editing, multiplayer games | Stateful servers; you design the message format, heartbeats, and reconnects |
| [WebRTC](connection-protocols/webrtc.md) | Peers swap connection details through a signaling server, then send audio, video, or data directly to each other, usually over UDP | Peer ↔ peer | Lowest (built for live media) | Voice, video, screen sharing | Signaling, NAT traversal, TURN relays; hard to scale to large groups |
| [Webhooks](connection-protocols/polling.md#webhooks) | When an event happens, the provider sends an HTTP `POST` to a URL you registered | Server → server | Seconds | Payments, GitHub events, finished third-party jobs | Your endpoint must be public, verify signatures, and handle retries and duplicates |
| [Pub/Sub](connection-protocols/pub-sub.md) | A publisher sends a message to a topic; the broker delivers a copy to every subscriber | One service → many services | Milliseconds to seconds | Fanning out events between backend services | Another system to run; duplicates and missed messages depend on the broker |
| [RSS](connection-protocols/pub-sub.md#how-rss-fits) | The site updates a feed file; readers fetch it on a timer (really polling) | Reader asks, site answers | Up to one poll interval (often minutes) | Blogs, podcasts, changelogs | Slow, and the publisher doesn't know who's subscribed |

Polling, long polling, SSE, WebSocket, and WebRTC carry updates **to a client**
(a browser or app). Webhooks and Pub/Sub usually move updates **between
servers**. A common setup is Pub/Sub between your services, then SSE or
WebSockets for the last hop to the browser.

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

With SSE or WebSockets, a client keeps a connection open to **one** server.
That server knows which users or chat rooms its local connections belong to;
another server does not automatically know about those sockets. For example,
Alice and Bob may be in the same chat room but connected to different servers.

```{mermaid}
flowchart LR
  Alice[Alice] -->|connect| LB[Ingress]
  Bob[Bob] -->|connect| LB
  LB -->|Alice's user ID| SA[Server A]
  LB -->|Bob's user ID| SB[Server B]
  SA -->|send message| Chat[Chat service]
  Chat --> DB[(Messages database)]
  Chat -->|publish room:42 update| Redis[Redis Pub/Sub]
  Redis --> SA
  Redis --> SB
  SA -->|push update| Alice
  SB -->|push update| Bob
```

### Consistent hashing

Think of **consistent hashing** as a seating chart for new connections: hash a
stable value such as `userId`, then send that user's connection to its assigned
server. If a server is added or removed, only some users get a new assignment
instead of reshuffling everyone. The ingress must actually route using that
stable key; an ordinary round-robin load balancer does not do this by itself.

This is **optional**. A WebSocket stays on the server that accepted it even
without consistent hashing. Hashing can make placement predictable, but it does
not copy a socket to another server or deliver a message to everyone in a
room. Existing connections stay put until they disconnect; after a server
failure, clients reconnect and resubscribe. Watch for uneven load from very
active users.

### Redis Pub/Sub

Redis Pub/Sub is the **intercom between servers**. Each server with local
listeners for `room:42` subscribes to that channel. After the chat service
saves a message, it publishes an update (often a message ID) to `room:42`.
Redis sends the update to subscribed servers, and each server pushes it to
its own connected clients. This works whether connections were placed by
consistent hashing or by a regular load balancer.

Pub/Sub is for **live notification, not message storage**: a disconnected
server misses updates, and Redis does not replay them. Keep chat messages in
the database and fetch missed messages on reconnect. If downstream processing
must never miss an event, use an outbox plus a durable stream or queue instead.

## Questions to ask

- Does data flow client-to-server, server-to-client, or both?
- How stale may an update be: minutes, seconds, or milliseconds?
- Is the payload occasional JSON, a continuous event feed, or live media?
- How many simultaneous connections must servers hold?
- What happens on reconnect: resume, replay, deduplicate, or accept data loss?
- Can proxies, load balancers, and firewalls carry long-lived connections?

## Further reading

- [RFC 6202: HTTP long polling and streaming](https://www.rfc-editor.org/rfc/rfc6202.html)
- [HTML Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [W3C WebRTC specification](https://www.w3.org/TR/webrtc/)
- [NGINX: consistent-hash upstream routing](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash)
- [Redis: Pub/Sub delivery semantics](https://redis.io/docs/latest/develop/pubsub/)
