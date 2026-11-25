---
title: Core Components
description: How API gateways, load balancers, and servers receive, route, protect, and process service traffic.
---

# Core Components

Three components sit between a user and your business logic:

- An **API gateway** is the front door. It checks requests and decides which
  service handles them.
- A **load balancer** spreads traffic across many copies of a service.
- **Servers** run your code.

**Example:** a shopping app calls `GET /orders/123`.

```{mermaid}
flowchart LR
  app[Mobile app] --> lb[Load balancer]
  lb --> gw[API gateway]
  gw -->|"/users/*"| users[User service]
  gw -->|"/orders/*"| lb2[Load balancer]
  lb2 --> o1[Order server 1]
  lb2 --> o2[Order server 2]
  lb2 --> o3[Order server 3]
```

1. The load balancer sends the request to a healthy gateway instance.
2. The gateway checks the user's token and rate limit, sees `/orders/*`, and
   forwards it to the order service.
3. Another load balancer picks one of three order servers.
4. That server loads order 123 from the database and returns it.

A small system might use one managed load balancer for all of this. Larger
systems separate the pieces.

## Quick comparison

| Component | Decides | Looks at | Example job |
| --- | --- | --- | --- |
| Layer 4 load balancer | Which server gets this connection? | IP address and port | Spread TCP connections across database proxies |
| Layer 7 load balancer | Which server gets this HTTP request? | URL, headers, cookies | Send `/images/*` to the media servers |
| API gateway | Is this call allowed, and where does it go? | The full request and who sent it | Reject an expired token; rate-limit a user |
| Application server | What should happen? | Validated request plus your data | Place the order, charge the card |

## In this section

- [API Gateway](./traffic-management/api-gateway.md): validation,
  authentication, rate limiting, routing, and caching at the front door.
- [Load Balancing](./traffic-management/load-balancing.md): Layer 4 vs.
  Layer 7, choosing a server, health checks, and sticky sessions.
- [Servers](./basics/servers.md): concurrency, scaling, capacity, and
  overload.

## Questions to ask

- Is the traffic HTTP, or raw TCP/UDP?
- Should routing depend on the URL, or just the address and port?
- Which checks (auth, rate limits) should every API share?
- Are connections short, or long-lived like WebSockets?
- How are unhealthy servers removed, and in-flight requests finished?
- What happens if the gateway or load balancer itself fails?

## Further reading

- [Google Cloud: Layer 4 and Layer 7 load balancing](https://cloud.google.com/load-balancing/docs/load-balancing-overview)
- [AWS: How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
