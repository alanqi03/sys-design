---
title: WebSockets
description: Full-duplex persistent connections for interactive messaging, with lifecycle and scaling patterns.
---

# WebSockets

WebSocket provides a persistent, full-duplex channel: client and server can send
messages independently at any time. A handshake establishes the channel, then
text, binary, and control frames flow without a complete HTTP request around
every application message.

## How it works

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant G as WebSocket gateway
  B->>G: HTTP opening handshake
  G-->>B: 101 Switching Protocols
  B->>G: {type: "chat.send", text: "Hello"}
  G-->>B: {type: "chat.accepted", id: "msg_42"}
  G-->>B: {type: "presence.changed", online: 18}
  Note over B,G: Either side may send until close
```

```javascript
const socket = new WebSocket("wss://api.example.com/realtime");

socket.addEventListener("open", () => {
  socket.send(JSON.stringify({
    type: "room.join",
    requestId: crypto.randomUUID(),
    roomId: "system-design"
  }));
});

socket.addEventListener("message", event => {
  routeMessage(JSON.parse(event.data));
});
```

Use `wss://` in production so the channel is protected by TLS. Define an
application envelope with a message type, schema version, request or event ID,
and payload. WebSocket supplies transport, not the business message format.

## Good use cases

- chat, presence, typing indicators, and multiplayer games;
- collaborative editors and shared cursors;
- interactive control or telemetry where both sides send frequently; and
- subscriptions in which clients dynamically join and leave many topics.

For one-way browser notifications, [SSE](server-sent-events.md) is often simpler.
For voice or video, use [WebRTC](webrtc.md); sending live media through a browser
WebSocket misses WebRTC's congestion control and media pipeline.

## Connection lifecycle

A socket can silently die when a device sleeps, changes networks, or passes
through an idle proxy. Design for:

- authentication during or immediately after connection establishment;
- heartbeats and deadlines that detect half-open connections;
- exponential backoff with jitter when reconnecting;
- resubscription and state resynchronization after reconnect; and
- stable message IDs so retried commands and replayed events can be deduplicated.

WebSocket preserves frame order within one connection. Reconnection, multiple
servers, and backend event sources require application-level ordering and
recovery rules.

## Scaling gateways

Long-lived connections make gateways stateful even when business services are
stateless. A common layout separates socket management from domain processing:

```{mermaid}
flowchart LR
  clients[Connected clients] --> lb[Load balancer]
  lb --> g1[Gateway 1]
  lb --> g2[Gateway 2]
  g1 <--> bus[(Pub/sub or event bus)]
  g2 <--> bus
  bus <--> services[Application services]
```

The gateway maps users and subscriptions to connections; the shared bus routes
events to whichever gateway currently owns each connection. Measure concurrent
connections, reconnect rate, messages and bytes per second, send-buffer size,
and event-loop delay. Enforce a bounded outbound buffer so one slow client
cannot consume unbounded memory.

## Further reading

- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
