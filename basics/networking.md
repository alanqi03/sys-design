---
title: Networking
---

# Networking

For a distributed system, the network is both the connection between components
and a major source of delay and failure. A designer does not need to implement a
network stack, but must understand what happens to a request in transit.

## From a name to a response

When a client requests `https://example.com/items/42`, it usually performs these
steps:

1. **DNS resolution** maps the hostname to an IP address.
2. **Connection setup** establishes a transport connection, commonly TCP or
   QUIC.
3. **TLS negotiation** authenticates the server and creates an encrypted channel.
4. **HTTP exchange** sends a request and receives a response.
5. Intermediaries such as a CDN, reverse proxy, or load balancer may route or
   answer the request before it reaches the application.

Connection reuse, DNS caching, and colocating services reduce the number and
cost of these round trips.

## Latency and throughput

**Latency** is the time required to complete one operation. **Throughput** is the
number of operations completed per unit time. They are related but not
interchangeable: batching may improve throughput while increasing the time an
individual item waits.

For user-facing systems, averages hide poor experiences. Percentiles are more
useful: a p99 latency of 800 ms means 1% of requests take at least 800 ms.
Tail latency compounds when one page depends on many downstream calls.

## Transport choices

TCP provides an ordered, reliable byte stream. Lost packets are retransmitted,
which is convenient for most application protocols but can add delay. UDP sends
independent datagrams without delivery or ordering guarantees. QUIC builds a
secure, multiplexed transport on UDP and is used by HTTP/3.

Choose at the application level unless the problem specifically requires a
custom protocol:

| Need | Common choice |
| --- | --- |
| Public request/response API | HTTPS with REST or GraphQL |
| Typed service-to-service RPC | gRPC over HTTP/2 |
| Browser updates | WebSocket or server-sent events |
| Asynchronous delivery | A message broker or event stream |

## Routing and load balancing

A load balancer distributes traffic across healthy server instances. Common
strategies include round robin, least connections, weighted routing, and
consistent hashing. Layer 4 load balancers route using transport information;
Layer 7 load balancers understand application protocols such as HTTP and can
route by host, path, or header.

Health checks should reflect whether an instance can serve traffic, but they
must be cheap and stable. A dependency outage should not necessarily make every
application instance fail its health check and trigger a restart storm.

## Timeouts, retries, and idempotency

Every remote call needs a finite timeout. A retry can mask a transient failure,
but it also adds load precisely when a dependency may be struggling. Robust
clients generally use:

- a bounded number of attempts;
- exponential backoff;
- random jitter so clients do not retry in lockstep; and
- an overall deadline shared across nested calls.

A retry is safest when an operation is **idempotent**: performing it more than
once has the same intended effect as performing it once. APIs can make a
non-idempotent action, such as creating a payment, safely retryable by accepting
an idempotency key and remembering the result.

```{warning}
A client timeout does not prove that the server did no work. The response may
have been lost after the operation committed.
```

## Design checklist

- Where are network boundaries in the request path?
- Which calls can run in parallel?
- What are the connection, request, and overall timeouts?
- Which operations may be retried safely?
- How does the system behave during packet loss, high latency, or a partial
  regional outage?

