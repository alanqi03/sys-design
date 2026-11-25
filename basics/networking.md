---
title: Networking
---

# Networking

In a distributed system, every call between components crosses the network,
and the network is slow and unreliable compared with a function call. You don't
need to build a network stack, but you do need to know what a request goes
through.

## From a URL to a response

When a browser opens `https://example.com/items/42`:

| Step | What happens | Typical cost |
| --- | --- | --- |
| 1. DNS | Look up the IP for `example.com` | 0 ms if cached, ~20–100 ms if not |
| 2. TCP | Open a connection | 1 round trip |
| 3. TLS | Verify the server's certificate and set up encryption | 1 more round trip |
| 4. HTTP | Send `GET /items/42` and get the response | 1 round trip + server time |

A CDN, proxy, or load balancer may answer or forward the request along the way.

With a 50 ms round trip, a brand-new connection costs about 150 ms before the
server does any work. Reusing connections, caching DNS, and putting servers
near users cut this down.

## Latency vs. throughput

- **Latency:** how long one request takes (e.g. 80 ms).
- **Throughput:** how many requests finish per second (e.g. 5,000/s).

Batching usually raises throughput but makes each item wait longer.

Averages hide slow requests, so use **percentiles**. A p99 of 800 ms means 1 in
100 requests takes at least 800 ms. If a page makes 10 backend calls, the chance
that at least one hits its p99 is about 10%.

## Transport choices

- **TCP:** reliable and ordered; lost packets are resent. Used by most
  protocols.
- **UDP:** no guarantees, but low overhead. Used for games, video calls, DNS.
- **QUIC:** reliable and encrypted, built on UDP. Used by HTTP/3.

In practice, pick at the application level:

| Need | Common choice |
| --- | --- |
| Public request/response API | HTTPS with REST or GraphQL |
| Service-to-service calls | gRPC |
| Live updates to a browser | WebSockets or server-sent events |
| Background work | A queue or event stream |

## Load balancing

A load balancer spreads requests across healthy servers. Layer 4 balancers route
by IP and port; Layer 7 balancers read HTTP and can route by URL or header. See
[Load Balancing](../traffic-management/load-balancing.md).

## Timeouts, retries, and idempotency

Every network call needs a **timeout**. Without one, a hung dependency ties up
your server forever.

Retries hide brief failures but add load right when a service is struggling.
Retry carefully:

```text
attempt 1 → fails
wait ~100 ms (+ random jitter) → attempt 2 → fails
wait ~200 ms (+ random jitter) → attempt 3 → give up
```

- Limit the number of attempts.
- Wait longer each time (exponential backoff).
- Add randomness (jitter) so thousands of clients don't retry in sync.
- Share one overall deadline across nested calls.

Only retry **idempotent** operations, meaning doing them twice has the same
effect as once. `GET /items/42` is safe to retry. `POST /payments` is not,
unless the client sends an **idempotency key** that the server uses to ignore
duplicates.

```{warning}
A timeout doesn't mean the server did nothing. The payment may have gone
through and only the response was lost. That's why idempotency keys matter.
```

## Checklist

- Where does the request cross the network?
- Which calls can run in parallel?
- What are the timeouts?
- Which operations are safe to retry?
- What happens under packet loss, high latency, or a regional outage?
