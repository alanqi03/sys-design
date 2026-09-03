---
title: Connection Protocols
description: A practical guide to polling, long polling, Server-Sent Events, WebSockets, and WebRTC.
---

# Connection Protocols

Applications need different ways to learn about new data. A job-status page can
check occasionally, a dashboard needs server-driven updates, chat needs both
sides to speak at any time, and a video call must carry latency-sensitive media.

“Connection protocols” is used broadly here. Polling and long polling are HTTP
request patterns, SSE is an HTTP event-stream format and browser API, WebSocket
is a separate framed protocol established through a handshake, and WebRTC is a
collection of APIs and protocols for peer media and data.

## Quick comparison

| Choice | Communication | Best fit | Main trade-off |
| --- | --- | --- | --- |
| [Polling](connection-protocols/polling.md) | Client requests, server responds | Rare or delay-tolerant updates | Empty requests waste work; interval adds latency |
| [Long polling](connection-protocols/long-polling.md) | Server delays each HTTP response | Near-real-time updates with broad HTTP compatibility | Reconnect and HTTP overhead after every response |
| [SSE](connection-protocols/server-sent-events.md) | Server streams text events to client | Notifications, dashboards, progress, AI text output | Server-to-client only; text format |
| [WebSocket](connection-protocols/websockets.md) | Full-duplex messages | Chat, collaboration, multiplayer state | Stateful connections and recovery are more complex |
| [WebRTC](connection-protocols/webrtc.md) | Peer media and/or data | Voice, video, screen sharing, peer data | Signaling, NAT traversal, relays, and group topology |

## A starting decision

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
client commands. Prefer the simplest option that meets the latency, direction,
browser support, infrastructure, and scale requirements.

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
