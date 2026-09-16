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

## API Design

WebSocket gives the application a two-way pipe, but it does not define the
messages inside that pipe. For a system-design whiteboard, list the commands the
client may send and the messages the server may send back.

### Chat API whiteboard

**Commands sent from client to server:**

- `createChat`
- `sendMessage`
- `updateChat`
- `markMessagesRead`
- `startTyping`
- `stopTyping`

**Commands received from server:**

- `newMessage`
- `chatUpdate`
- `messageRead`
- `typingUpdate`
- `commandResult`
- `error`

More precisely, the client sends **commands** asking for a change, while the
server usually sends **results** and **events** describing what happened. An
event such as `newMessage` may arrive without the current client requesting it
because another participant sent the message.

```{mermaid}
sequenceDiagram
  participant A as Client A
  participant S as Chat server
  participant B as Client B

  A->>S: sendMessage command with requestId
  S-->>A: commandResult with same requestId
  S-->>A: newMessage event
  S-->>B: newMessage event
```

### Use a consistent envelope

Every message should identify its type and carry an ID. A `requestId` connects
a command to its result; an `eventId` lets clients deduplicate pushed events.

```json
{
  "type": "sendMessage",
  "version": 1,
  "requestId": "req_123",
  "payload": {}
}
```

### `createChat` request and result

```json
{
  "type": "createChat",
  "version": 1,
  "requestId": "req_123",
  "payload": {
    "participants": ["user_42", "user_99"],
    "name": "System Design Study Group"
  }
}
```

```json
{
  "type": "commandResult",
  "version": 1,
  "requestId": "req_123",
  "payload": {
    "ok": true,
    "chatId": "chat_abc"
  }
}
```

The matching `requestId` tells the client which pending command completed.
Other chat participants may separately receive a `chatUpdate` event.

### `sendMessage` request, result, and event

```json
{
  "type": "sendMessage",
  "version": 1,
  "requestId": "req_456",
  "payload": {
    "chatId": "chat_abc",
    "clientMessageId": "client_msg_789",
    "text": "Are we still meeting at 6?"
  }
}
```

```json
{
  "type": "commandResult",
  "version": 1,
  "requestId": "req_456",
  "payload": {
    "ok": true,
    "messageId": "msg_101",
    "createdAt": "2026-09-10T19:30:00Z"
  }
}
```

After saving the message, the server pushes the resulting event to every
connected participant:

```json
{
  "type": "newMessage",
  "version": 1,
  "eventId": "event_555",
  "payload": {
    "chatId": "chat_abc",
    "messageId": "msg_101",
    "senderId": "user_42",
    "text": "Are we still meeting at 6?",
    "createdAt": "2026-09-10T19:30:00Z"
  }
}
```

`clientMessageId` is an idempotency key generated before sending. If the client
reconnects and retries because it missed the result, the server can return the
original saved message instead of creating a duplicate.

### Error result

A WebSocket message does not receive an HTTP status code such as `400` or
`500`, so define application error codes explicitly:

```json
{
  "type": "commandResult",
  "version": 1,
  "requestId": "req_456",
  "payload": {
    "ok": false,
    "error": {
      "code": "CHAT_NOT_FOUND",
      "message": "The chat does not exist or is not accessible."
    }
  }
}
```

Keep the whiteboard contract small: define commands and events, their payloads,
how results correlate to commands, idempotency, authorization, ordering, and
what the client does after reconnecting.

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

See [Managing Connections](../connection-protocols.md#managing-connections) for
an example of optional consistent-hash routing and Redis Pub/Sub across a
gateway fleet.

## Further reading

- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
