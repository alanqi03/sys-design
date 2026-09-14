---
title: Core Components
description: How API gateways, load balancers, and servers receive, route, protect, and process service traffic.
---

# Core Components

An **API gateway** provides a controlled entry point to an application's APIs.
A **load balancer** distributes connections or requests across healthy targets.
**Servers** run the application code that validates requests, applies business
rules, and talks to stateful dependencies. These roles can overlap in a small
system, but their design responsibilities are different.

```{mermaid}
flowchart LR
  clients[Clients] --> edge[DNS / edge]
  edge --> l4[L4 load balancer]
  l4 --> gateway[API gateway / L7 load balancer]
  gateway --> users[User service]
  gateway --> orders[Order service]
  gateway --> search[Search service]
```

This is one possible layout, not a requirement. A small system may need only one
managed Layer 7 load balancer. A high-volume or multi-protocol system may put a
Layer 4 entry point in front of a fleet of gateways.

## Quick comparison

| Component | Main decision | Can inspect | Typical responsibilities |
| --- | --- | --- | --- |
| Layer 4 load balancer | Which target receives this network flow? | IP addresses, ports, transport protocol | TCP/UDP distribution, connection-level health, TLS passthrough or termination |
| Layer 7 load balancer | Which target receives this application request? | HTTP method, host, path, headers, cookies | Content routing, TLS termination, HTTP health checks, redirects |
| API gateway | Is this API call allowed, and how should it reach the backend? | Application request and API identity | Validation, authentication, rate limits, routing, protocol and response transformation, caching |
| Application server | What business operation should run? | Validated request and application data | Business logic, authorization, database access, external calls, and response generation |

Read [API Gateway](./traffic-management/api-gateway.md) for the public API
request pipeline. Read [Load Balancing](./traffic-management/load-balancing.md)
for the L4-versus-L7 choice, algorithms, health checks, affinity, and failure
handling. Read [Servers](./basics/servers.md) for concurrency, capacity,
horizontal scaling, overload, and graceful failure.

## Where each belongs

- Use a **Layer 4 load balancer** for high-throughput TCP or UDP, non-HTTP
  protocols, static entry addresses, or end-to-end TLS passthrough.
- Use a **Layer 7 load balancer** when HTTP host, path, method, or headers should
  determine the backend.
- Use an **API gateway** when clients need a stable public contract and services
  need shared policy enforcement or protocol adaptation.
- Use **application servers** to own business logic and APIs. Keep them
  stateless when possible so a load balancer can send any request to any healthy
  instance.
- Use service discovery or an internal load balancer for service-to-service
  distribution when public API features are unnecessary.

## Questions to ask

- Is the traffic HTTP, raw TCP, UDP, or a mixture?
- Must the intermediary terminate TLS or preserve end-to-end encryption?
- Is routing based on an address and port, or on HTTP content?
- Which policies must be uniform across APIs, and which belong to the service?
- Are connections short, multiplexed, or long-lived like WebSockets and SSE?
- How will unhealthy targets be removed and in-flight requests drained?
- How many requests, active connections, CPU-heavy tasks, or downstream calls
  can each server handle before it becomes overloaded?
- What happens if the load balancer or gateway itself is overloaded or unavailable?

## Further reading

- [Google Cloud: Layer 4 and Layer 7 load balancing](https://cloud.google.com/load-balancing/docs/load-balancing-overview)
- [AWS: How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [AWS API Gateway: Request and response transformations](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-parameter-mapping.html)
