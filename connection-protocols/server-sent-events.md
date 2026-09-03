---
title: Server-Sent Events (SSE)
description: One-way HTTP event streams for browser updates, including EventSource, reconnection, and scaling.
---

# Server-Sent Events (SSE)

Server-Sent Events keep an HTTP response open so the server can send a sequence
of UTF-8 text events to a browser. The browser's `EventSource` API handles the
stream, reconnects after interruptions, and can send the last received event ID
when resuming.

## How it works

```{mermaid}
sequenceDiagram
  participant B as Browser EventSource
  participant A as API
  B->>A: GET /events
  A-->>B: Content-Type: text/event-stream
  A-->>B: id: 41<br/>event: progress<br/>data: {"percent": 25}
  A-->>B: id: 42<br/>event: progress<br/>data: {"percent": 60}
  Note over B,A: Connection drops
  B->>A: GET /events<br/>Last-Event-ID: 42
  A-->>B: Resume after event 42
```

An event stream is separated by blank lines:

```text
id: 42
event: order-status
data: {"orderId":"order_123","status":"shipped"}

retry: 3000

```

The browser client is small:

```javascript
const source = new EventSource("/api/events");

source.addEventListener("order-status", event => {
  const update = JSON.parse(event.data);
  renderOrder(update);
});

source.onerror = () => showConnectionState("reconnecting");
```

Use ordinary `POST`, `PATCH`, or `DELETE` requests for commands from the client.
SSE is one-way; together, SSE and regular HTTP often make a simpler application
model than a full-duplex socket.

## Good use cases

- live dashboards, scores, prices, and operational metrics;
- notifications and order-status updates;
- job progress and log tailing; and
- streaming generated text or incremental search results.

SSE is less suitable for binary payloads, very high message rates, peer-to-peer
media, or applications where client and server send frequent interdependent
messages.

## Reconnection and delivery

Set `id` on events that may need replay. After reconnecting, the browser can
send `Last-Event-ID`; the server resumes after that position when history is
available. The `retry` field suggests a reconnection delay. The client must
still tolerate duplicates because the connection can fail at an ambiguous
moment.

Send occasional comment lines such as `: heartbeat` to keep an otherwise idle
stream visible to intermediaries. Heartbeats should be infrequent enough not to
waste capacity.

## Operating at scale

- Use asynchronous servers and track open connections, bytes, and disconnects.
- Disable proxy buffering and set idle timeouts above the heartbeat interval.
- Fan events from a broker or pub/sub layer to stateless SSE gateways.
- Apply per-client buffering limits; disconnect a slow consumer rather than
  letting its memory grow without bound.
- Authenticate when opening the stream and revalidate long-lived sessions as
  required by the security model.

## Further reading

- [HTML Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
