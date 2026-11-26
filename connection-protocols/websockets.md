---
title: WebSockets
description: Full-duplex persistent connections for interactive messaging, with lifecycle and scaling patterns.
---

# WebSockets

A WebSocket is a long-lived, two-way connection. After one HTTP handshake,
client and server can each send messages at any time, without a new HTTP
request for every message.

**Running example:** a group chat app.

## How it works

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant S as Chat server
  B->>S: HTTP handshake (Upgrade: websocket)
  S-->>B: 101 Switching Protocols
  B->>S: sendMessage "Hello"
  S-->>B: commandResult ok
  S-->>B: newMessage from Bob
  S-->>B: typingUpdate: Carol is typing
  Note over B,S: either side sends until close
```

```javascript
const socket = new WebSocket("wss://chat.example.com/ws");   // wss = encrypted

socket.onopen = () =>
  socket.send(JSON.stringify({ type: "joinChat", requestId: "req_1", payload: { chatId: "chat_abc" } }));

socket.onmessage = event => handle(JSON.parse(event.data));
```

## API design

A WebSocket is just a pipe; you define the messages that go through it. For a
chat app on a whiteboard:

| Client → server (commands) | Server → client (results and events) |
| --- | --- |
| `createChat` | `commandResult`: reply to a command |
| `sendMessage` | `newMessage`: someone posted |
| `markMessagesRead` | `messageRead` |
| `startTyping` / `stopTyping` | `typingUpdate` |
| | `error` |

Events like `newMessage` can arrive at any time, because they're caused by
*other* users.

### Sending a message

The client sends a command with a `requestId`:

```json
{
  "type": "sendMessage",
  "requestId": "req_456",
  "payload": { "chatId": "chat_abc", "clientMessageId": "cm_789", "text": "Still meeting at 6?" }
}
```

The server saves it and replies with the same `requestId`, so the client knows
which command finished:

```json
{ "type": "commandResult", "requestId": "req_456", "payload": { "ok": true, "messageId": "msg_101" } }
```

Then it pushes an event to everyone in the chat:

```json
{
  "type": "newMessage",
  "eventId": "event_555",
  "payload": { "chatId": "chat_abc", "messageId": "msg_101", "senderId": "user_42", "text": "Still meeting at 6?" }
}
```

```{mermaid}
sequenceDiagram
  participant A as Alice
  participant S as Server
  participant B as Bob
  A->>S: sendMessage (req_456)
  S-->>A: commandResult (req_456)
  S-->>A: newMessage
  S-->>B: newMessage
```

- **`clientMessageId`** is an idempotency key. If Alice's connection drops and
  she resends, the server returns the existing message instead of posting it
  twice.
- **`eventId`** lets clients skip events they've already seen.
- **Errors:** there are no HTTP status codes here, so define your own:

  ```json
  { "type": "commandResult", "requestId": "req_456",
    "payload": { "ok": false, "error": { "code": "CHAT_NOT_FOUND" } } }
  ```

## When to use WebSockets

- Chat, presence, and typing indicators
- Multiplayer games
- Collaborative editing and shared cursors

For one-way updates (notifications, dashboards), [SSE](server-sent-events.md)
is simpler. For voice and video, use [WebRTC](webrtc.md).

## Handling disconnects

Connections die silently when a phone sleeps or switches from Wi-Fi to cellular.
Plan for it:

- **Heartbeats:** ping every ~30 seconds to detect dead connections.
- **Reconnect with backoff and jitter:** 1 s, 2 s, 4 s... so a server restart
  doesn't cause every client to reconnect at the same instant.
- **Catch up on reconnect:** rejoin chats and fetch messages since the last one
  received.
- **Authenticate** when the connection opens.

## Scaling

Each connection stays on one server, so the servers become stateful. Alice and
Bob may be in the same chat but on different servers:

```{mermaid}
flowchart LR
  clients[Clients] --> lb[Load balancer]
  lb --> s1[Server 1: Alice]
  lb --> s2[Server 2: Bob]
  s1 <--> bus[(Redis Pub/Sub)]
  s2 <--> bus
```

When Alice sends a message, server 1 publishes it to the chat's channel, and
server 2 receives it and pushes it to Bob. See
[Managing Connections](../connection-protocols.md#managing-connections) for
the full pattern.

Watch open connections, reconnect rate, and messages per second. Cap each
client's outgoing buffer so one slow phone can't use up a server's memory.

## Further reading

- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
