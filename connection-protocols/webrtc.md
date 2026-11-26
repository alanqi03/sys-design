---
title: WebRTC
description: Real-time peer media and data, with plain-language explanations of IP addresses, NAT, signaling, STUN, TURN, and ICE.
---

# WebRTC

WebRTC lets two browsers or apps send **audio, video, screen shares, or data
directly to each other**, with low delay and built-in encryption.

**Running example:** Alice, on her home Wi-Fi, video-calls Bob, who is on his
phone's mobile network.

Sending the video straight from Alice to Bob, instead of through your server,
cuts delay and saves you paying for the bandwidth. The hard part is getting the
two devices to find and reach each other. To see why, you need two networking
ideas first.

## Background: why "just connect directly" is hard

### Public and private IP addresses

Every device on the internet needs an address to send and receive data. IPv4
only has about 4.3 billion addresses, which is far fewer than the number of
phones, laptops, and TVs in the world. So most networks share one:

- Your internet provider gives your home router **one public IP**, such as
  `203.0.113.7`. Anyone on the internet can send data to it.
- The router gives each device at home a **private IP**, such as
  `192.168.1.23`. It only means something inside your home network. Millions of
  other homes use `192.168.1.23` at the same moment.

Think of an apartment building: the street address is public, but "apartment
4B" only makes sense once you're inside the building.

So if Alice tells Bob "send the video to `192.168.1.23`," it's useless. That
address points to some device on *Bob's* network, or to nothing.

### NAT: the router translates addresses

**NAT (Network Address Translation)** is how the router lets many private
devices share one public IP. When Alice's laptop sends a packet out, the router
swaps the private address for its public one and remembers the swap:

```{mermaid}
sequenceDiagram
  participant L as Alice's laptop (192.168.1.23)
  participant R as Home router (203.0.113.7)
  participant W as Website
  L->>R: from 192.168.1.23:5000
  Note over R: remember 61001 = 192.168.1.23:5000
  R->>W: from 203.0.113.7:61001
  W-->>R: reply to 203.0.113.7:61001
  Note over R: look up 61001
  R-->>L: reply to 192.168.1.23:5000
```

The router's table now has one entry:

```text
203.0.113.7:61001  ↔  192.168.1.23:5000   (expires after a short idle time)
```

That leads to the key rule: **a NAT only lets in traffic that answers something
you sent out.** A packet arriving out of the blue matches no entry, so the router
drops it. It's like a mailroom that only accepts replies to letters you mailed.
Firewalls behave the same way, often more strictly.

That's fine for browsing, since you always start the conversation. But in a
call, *both* sides are behind NATs, and each router drops the other side's
first packet. Neither can go first.

## How WebRTC gets around it

WebRTC uses four pieces. Each solves one part of the problem.

| Piece | Problem it solves | Analogy |
| --- | --- | --- |
| **Signaling** | The peers have no way to talk yet | Texting a friend to plan a phone call |
| **STUN** | "What's my public address?" | Asking a friend what number your caller ID shows |
| **TURN** | A direct path is impossible | Passing notes through a mutual friend |
| **ICE** | Which path should we use? | Trying every route and keeping the best one |

### 1. Signaling: swap notes through your server

Before any media flows, Alice and Bob must agree on two things:

- **What** to send: audio and video formats (codecs), resolution, and
  encryption keys. Alice sends an **offer** and Bob replies with an **answer**.
  Both are text blobs in a format called SDP.
- **Where** to send it: a list of addresses each side might be reachable at,
  called **candidates**.

They exchange these through *your* app server, usually over a
[WebSocket](websockets.md). WebRTC doesn't define the signaling method; you
build it. Your server must also check that Bob is allowed to join Alice's call.

### 2. STUN: "what's my public address?"

Alice's laptop only knows its private IP. It sends a tiny request to a **STUN
server** on the public internet, which replies:

```text
Your request came from 203.0.113.7:61001
```

Now Alice knows the public address and port her router assigned, and she sends
it to Bob through signaling. Bob does the same.

Next comes **hole punching**: both sides send packets to each other's public
address at about the same time. Alice's outgoing packet creates an entry in her
router's NAT table, so when Bob's packet arrives, it looks like a reply and gets
let in. The same happens on Bob's side. This works for most home networks.

STUN is cheap to run: it answers small questions, and no video passes through
it.

### 3. TURN: relay when direct doesn't work

Sometimes hole punching fails:

- **Strict (symmetric) NATs**, common on corporate and some mobile networks,
  give a *different* public port for every destination. The address the STUN
  server saw isn't the one Bob would need to use.
- **Firewalls** may block the UDP traffic WebRTC prefers.

Then both sides connect *out* to a **TURN server**, which relays every packet
between them. Outgoing connections are almost always allowed, and TURN can even
run over TCP on port 443 to look like normal web traffic, so this nearly always
works.

```{mermaid}
flowchart LR
  A[Alice] -->|"video out"| T[TURN server]
  T -->|"video in"| B[Bob]
  B -->|"video out"| T
  T -->|"video in"| A
```

The cost: all the video flows through your server, so you pay for that
bandwidth and add a little delay. Only a minority of calls need TURN, but
without it those calls fail completely, so production apps always provide one.

### 4. ICE: try every path, keep the best

**ICE** is the process that ties it together. Each side gathers every candidate
it can find:

| Candidate type | Where it comes from | Works when |
| --- | --- | --- |
| **Host** | The device's own private IP | Both are on the same network |
| **Server-reflexive** | The public address STUN reported | Hole punching succeeds |
| **Relay** | An address on the TURN server | Always (as a fallback) |

The peers swap candidates through signaling, test pairs in order of preference,
and use the best one that works: direct first, relay last.

### The whole flow

```{mermaid}
sequenceDiagram
  participant A as Alice
  participant S as Signaling server
  participant ST as STUN / TURN
  participant B as Bob
  A->>S: Offer (I want audio and video)
  S->>B: Offer
  B->>S: Answer
  S->>A: Answer
  A->>ST: What's my public address?
  B->>ST: What's my public address?
  A->>S: My candidates
  S->>B: Alice's candidates
  B->>S: My candidates
  S->>A: Bob's candidates
  Note over A,B: ICE tests the pairs
  A<<->>B: Encrypted video, direct if possible
  Note over A,B: Otherwise relayed through TURN
```

## Browser code

```javascript
const connection = new RTCPeerConnection({
  iceServers: [
    { urls: "stun:stun.example.com" },
    { urls: "turn:turn.example.com", username, credential }
  ]
});

// Send my camera and microphone
for (const track of localStream.getTracks()) connection.addTrack(track, localStream);

// Each time ICE finds a candidate, send it to the other side via signaling
connection.onicecandidate = ({ candidate }) => {
  if (candidate) signaling.send({ type: "candidate", candidate });
};

// Show the other person's video when it arrives
connection.ontrack = event => { remoteVideo.srcObject = event.streams[0]; };
```

The offer/answer exchange is left out to keep this short, but a real app needs
it.

## Media and data channels

- **Media tracks** carry microphone, camera, or screen video. WebRTC picks
  codecs, adjusts quality as bandwidth changes, and encrypts everything.
- **Data channels** (`RTCDataChannel`) carry any data, such as chat, game
  moves, or file chunks. Each channel can be reliable and ordered, or faster
  with occasional drops.

**Use WebRTC for** calls, video meetings, screen sharing, and peer-to-peer file
transfer. If you just need server-to-browser messages, a
[WebSocket](websockets.md) or [SSE](server-sent-events.md) is far simpler.

## Group calls: mesh vs. SFU

In a **mesh**, everyone sends a copy of their video to everyone else. With 6
people, each person uploads 5 video streams, which overwhelms a home connection.

Most group call apps use an **SFU (Selective Forwarding Unit)**: each person
uploads **one** stream to a server, and the SFU forwards it to the others.

```{mermaid}
flowchart LR
  a[Alice] <--> sfu[SFU]
  b[Bob] <--> sfu
  c[Carol] <--> sfu
  d[Diego] <--> sfu
```

Uploads stay small, but the SFU now carries a lot of bandwidth. Place SFUs near
users, and plan for what happens when one fails mid-call.

## Checklist

- Always run TURN in production, and budget for its bandwidth.
- Use short-lived TURN credentials, and authorize who can join each call.
- Track call setup time, ICE failures, how often calls use the relay, packet
  loss, and frozen video.
- Handle denied camera permission, device changes, and switching from Wi-Fi to
  cellular mid-call.

## Further reading

- [W3C WebRTC specification](https://www.w3.org/TR/webrtc/)
- [MDN: WebRTC protocols (ICE, STUN, NAT, TURN, SDP)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)
- [RFC 8445: Interactive Connectivity Establishment (ICE)](https://www.rfc-editor.org/rfc/rfc8445.html)
- [RFC 8656: Traversal Using Relays around NAT (TURN)](https://www.rfc-editor.org/rfc/rfc8656.html)
