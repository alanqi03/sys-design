---
title: Load Balancing
description: Layer 4 versus Layer 7 load balancing, routing algorithms, health checks, affinity, and resilient operation.
---

# Load Balancing

A load balancer gives clients a stable endpoint and distributes traffic across
healthy targets. It improves capacity and availability only when the targets
are independently useful and the load balancer itself is redundant.

## Layer 4 versus Layer 7

“Layer 4” and “Layer 7” refer to the transport and application layers of the OSI
model. The practical difference is how much of the traffic the load balancer
understands.

| | Layer 4 (transport) | Layer 7 (application) |
| --- | --- | --- |
| Understands | Source/destination address, port, TCP/UDP and connection state | HTTP method, host, path, headers, cookies and status |
| Balancing unit | Connection or transport flow | HTTP request or stream, depending on protocol and implementation |
| Routing examples | `TCP :5432` to database proxies; `UDP :53` to DNS servers | `/images` to media; `/orders` to order service |
| Protocols | TCP, UDP, TLS passthrough and other transport traffic | Usually HTTP, HTTPS, HTTP/2, gRPC and WebSocket handshakes |
| Advantages | Protocol agnostic, less application parsing, can preserve end-to-end TLS | Rich content routing, HTTP-aware health checks, redirects and header policy |
| Costs | Cannot route by URL or HTTP identity | More processing, configuration and application-level failure modes |

```{mermaid}
flowchart TB
  subgraph l4[Layer 4]
    c1[Client connection] --> nlb[L4 load balancer]
    nlb -->|TCP/UDP flow| t1[Target]
  end
  subgraph l7[Layer 7]
    c2[HTTP client] --> alb[L7 load balancer]
    alb -->|Host: api.example.com| api[API targets]
    alb -->|Path: /static| static[Static targets]
  end
```

### Choose Layer 4 when

- the traffic is non-HTTP TCP or UDP;
- very high connection throughput or low proxy overhead is important;
- the backend must terminate TLS for end-to-end encryption; or
- routing by address and port is sufficient.

### Choose Layer 7 when

- hosts, paths, methods, headers, or cookies determine the destination;
- the edge should terminate TLS, redirect HTTP, or apply HTTP policy;
- application-aware health checks are required; or
- several HTTP services share one public endpoint.

Some architectures use both: a Layer 4 entry point distributes connections to
a fleet of Layer 7 proxies. That adds another network hop, so use it only when
the L4 features or separation of responsibilities are valuable.

## How a target is selected

- **Round robin:** rotate evenly through healthy targets; works when requests
  and instances have similar cost and capacity.
- **Weighted round robin:** send more traffic to larger targets or use small
  weights for a canary deployment.
- **Least connections:** prefer the target with fewer open connections; useful
  when connection durations differ, but one connection may carry many HTTP/2
  requests.
- **Least outstanding requests:** prefer the target with less active work; more
  closely models request load when the proxy can observe it.
- **Hash or consistent hash:** use a client, header, or application key to choose
  a target; useful for affinity and distributed caches, but a hot key can create
  an uneven target.
- **Flow hash:** an L4 balancer commonly hashes connection properties such as
  source/destination addresses, ports, and protocol so packets in one flow reach
  the same target.

No algorithm knows the true cost of future work. Combine a reasonable policy
with target-side concurrency limits and load shedding.

## Health checks and draining

An **active health check** probes a target periodically. A **passive check**
learns from real connection or request failures. Layer 4 checks can confirm that
a port accepts a connection; Layer 7 checks can request an endpoint and verify
an HTTP result.

Use a lightweight readiness endpoint that reflects whether the instance can
serve new traffic. Do not include every optional dependency: a shared analytics
outage should not remove all otherwise useful instances. Require several failed
checks before removal and several successful checks before restoration to avoid
flapping.

During deployment or scale-in:

1. mark the target unavailable for new work;
2. allow connection and request draining for a bounded period;
3. finish, hand off, or terminate remaining work safely; and
4. stop the target.

WebSockets, SSE, uploads, and long-running requests need longer or
protocol-aware draining. A load balancer normally cannot move an established
connection to another target.

## TLS and client identity

With **TLS termination**, the load balancer decrypts traffic, can inspect HTTP,
and may open a new TLS connection to the backend. With **TLS passthrough**, the
backend owns the certificate and encryption remains end to end, but the
balancer cannot inspect HTTP content.

When a proxy terminates a client connection, the backend's network peer is the
proxy. Preserve the original client address using a trusted mechanism such as a
provider feature, PROXY protocol, or sanitized forwarding header. Never trust a
client-supplied `X-Forwarded-For` unless the trusted proxy removes or replaces
untrusted values.

## Session affinity

Sticky sessions route a client to the same target using a cookie, client hash,
or connection. They can help with a stateful legacy application, local caches,
or long-lived connections, but they create uneven load and make target failure
more disruptive.

Prefer stateless service instances with session state in a signed token or
shared store. If affinity is required, bound its lifetime and ensure any target
can recover the session after failure.

## Global and regional balancing

```{mermaid}
flowchart TD
  users[Users] --> global[DNS / anycast / global traffic manager]
  global -->|Nearest healthy region| west[Regional load balancer West]
  global -->|Nearest healthy region| east[Regional load balancer East]
  west --> w1[Zone A targets]
  west --> w2[Zone B targets]
  east --> e1[Zone A targets]
  east --> e2[Zone B targets]
```

Global routing considers region health, latency, geography, capacity, and data
residency. Regional balancing spreads traffic across zones and instances. A
service is not multi-region merely because traffic can reach another region;
its data, dependencies, consistency model, and failover procedure must work
there too.

## Common failure modes

- **Retry amplification:** the load balancer, gateway, client, and service each
  retry, multiplying one failure into many requests.
- **Connection imbalance:** a few targets keep many long-lived connections even
  after new targets are added.
- **Slow targets:** a target remains technically healthy but builds a queue and
  raises tail latency.
- **Bad health checks:** all targets are removed because one optional shared
  dependency fails.
- **Uneven zones:** traffic is balanced evenly across zones with different
  healthy capacity.
- **Hidden overload:** the load balancer accepts more work than backends can
  complete, so queues and timeouts grow.

Monitor healthy target count, new and active connections, requests and bytes,
per-target distribution, rejected work, retries, backend latency, and response
codes. Capacity-test the balancer and its configuration rather than assuming it
is unlimited.

## Further reading

- [Google Cloud: Layer 4 and Layer 7 load balancing](https://cloud.google.com/load-balancing/docs/load-balancing-overview)
- [AWS: Network Load Balancer overview](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)
- [AWS: Elastic Load Balancing routing algorithms](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
