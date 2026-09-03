---
title: WebRTC
description: Real-time peer media and data using signaling, ICE, STUN, TURN, and scalable call topologies.
---

# WebRTC

WebRTC carries real-time audio, video, screen sharing, and arbitrary data
between peers. It provides browser APIs plus encrypted, congestion-aware media
and data transports. It is a stack rather than a single connection protocol.

## Establishing a connection

Peers cannot usually connect by knowing only each other's private IP addresses.
They first use an application-provided **signaling channel**—often HTTPS or
WebSocket—to exchange session descriptions and network candidates. WebRTC does
not prescribe the signaling transport or message format.

```{mermaid}
sequenceDiagram
  participant A as Alice
  participant S as Signaling service
  participant B as Bob
  participant I as STUN/TURN
  A->>S: Offer
  S->>B: Offer
  B->>S: Answer
  S->>A: Answer
  A->>I: Gather ICE candidates
  B->>I: Gather ICE candidates
  A-->>B: Exchange candidates through signaling
  A<<->>B: Encrypted media/data, direct when possible
  Note over A,B: TURN relays traffic if a direct path fails
```

- **ICE** tests candidate network paths and selects a working pair.
- **STUN** helps a peer discover the public-facing address assigned by NAT.
- **TURN** relays traffic when firewalls or NAT prevent a direct connection.

A stripped-down browser setup looks like this:

```javascript
const connection = new RTCPeerConnection({
  iceServers: [{ urls: "turn:turn.example.com", username, credential }]
});

for (const track of localStream.getTracks()) {
  connection.addTrack(track, localStream);
}

connection.onicecandidate = ({ candidate }) => {
  if (candidate) signaling.send({ type: "candidate", candidate });
};

connection.ontrack = event => {
  remoteVideo.srcObject = event.streams[0];
};
```

The omitted offer/answer handling is essential in a real application. Signaling
messages also need authentication and authorization so a user cannot join an
unrelated call.

## Media and data channels

Media tracks carry microphone, camera, or screen content. WebRTC negotiates
codecs, responds to changing bandwidth, and encrypts traffic. A
`RTCDataChannel` carries arbitrary peer data and can be configured for reliable
ordered delivery or lower-latency partial reliability.

Good use cases include one-to-one calls, video conferences, screen sharing,
remote assistance, peer file transfer, and low-latency game or device data.
Ordinary HTTP or WebSocket is simpler when peer media and NAT traversal are not
needed.

## From one-to-one to group calls

Sending one copy of every media stream to every participant—a full mesh—causes
upload bandwidth and connection count to grow quickly. Most multi-party systems
use a **Selective Forwarding Unit (SFU)**:

```{mermaid}
flowchart LR
  a[Alice] <--> sfu[SFU]
  b[Bob] <--> sfu
  c[Carol] <--> sfu
  d[Diego] <--> sfu
```

Each participant uploads to the SFU, which forwards selected streams to others.
This improves client scalability but makes the SFU's bandwidth, geographic
placement, congestion behavior, and failure recovery core design concerns.

## Operating concerns

- Provide TURN in production and capacity-plan for the relay bandwidth; direct
  peer connectivity is never guaranteed.
- Collect call setup time, ICE failure rate, selected candidate type, packet
  loss, jitter, round-trip time, bitrate, and freeze duration.
- Handle device permission denial, camera changes, network handoffs, and
  renegotiation.
- Separate signaling availability from media-plane availability and decide how
  each recovers after failure.
- Use short-lived TURN credentials and authorize rooms, participants, and
  signaling actions.

## Further reading

- [W3C WebRTC specification](https://www.w3.org/TR/webrtc/)
- [RFC 8445: Interactive Connectivity Establishment (ICE)](https://www.rfc-editor.org/rfc/rfc8445.html)
- [RFC 8656: Traversal Using Relays around NAT (TURN)](https://www.rfc-editor.org/rfc/rfc8656.html)
