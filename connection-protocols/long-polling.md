---
title: Long Polling
description: Held HTTP requests for near-real-time server updates, with reconnect, timeout, and scaling considerations.
---

# Long Polling

Long polling starts like polling, but the server does not immediately return an
empty response. It holds the request until data is available or a timeout is
reached. After any response, the client opens the next request immediately.

## How it works

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /events?after=41
  Note over A: Hold request
  A-->>B: 200 {id: 42, type: "OrderShipped"}
  B->>A: GET /events?after=42
  Note over A: No event; timeout
  A-->>B: 204 No Content
  B->>A: GET /events?after=42
```

The client keeps one request outstanding:

```javascript
let cursor = 0;

async function receiveEvents() {
  while (true) {
    try {
      const response = await fetch(`/api/events?after=${cursor}`);
      if (response.status === 204) continue;

      const event = await response.json();
      render(event);
      cursor = event.id;
    } catch {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

Useful examples include a legacy browser notification channel, low-volume chat,
or waiting for a remotely processed job where ordinary polling is too slow.

## Server design

Do not dedicate a blocked operating-system thread to every waiting client. Use
asynchronous I/O, register each waiter against the relevant topic or user, and
complete it when an event arrives. Bound the number of waiters per user and the
memory retained for each one.

Return before the shortest proxy or load-balancer idle timeout, then let the
client reconnect. Suppress caching for responses and add jittered backoff after
errors so an outage does not cause a synchronized reconnect storm.

## Delivery and ordering

A connection can fail after the server sends an event but before the client
records it. Include a monotonically increasing cursor or stable event ID; on
reconnect, request events after the last processed position. Consumers should
deduplicate repeated IDs, and the server needs enough retention to fill a
temporary gap.

Long polling is not automatically bidirectional. Client-to-server commands use
separate ordinary HTTP requests, while the outstanding long poll carries
server-to-client updates.

## When to use it

Choose long polling when near-real-time delivery is needed but an environment
cannot reliably use streaming responses or WebSockets. Prefer [SSE](server-sent-events.md)
for a modern browser feed: it avoids a new HTTP response for every event and
provides a native reconnection model. Prefer [WebSocket](websockets.md) for
frequent traffic in both directions.

## Further reading

- [RFC 6202: Known Issues and Best Practices for Long Polling](https://www.rfc-editor.org/rfc/rfc6202.html)
