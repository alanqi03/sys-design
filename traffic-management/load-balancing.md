---
title: Load Balancing
description: Layer 4 versus Layer 7 load balancing, routing algorithms, health checks, affinity, and resilient operation.
---

# Load Balancing

A load balancer gives clients one address and spreads their traffic across many
servers. If a server dies, the load balancer stops sending it traffic.

**Running example:** a web app with three servers behind
`api.example.com`. Each server handles about 500 requests per second, so
together they handle about 1,500, and losing one doesn't take the site down.

## Layer 4 vs. Layer 7

The difference is how much of the traffic the load balancer reads.

| | Layer 4 | Layer 7 |
| --- | --- | --- |
| Reads | IP address and port only | The HTTP request: URL, headers, cookies |
| Can route | "Port 5432 → database proxies" | "`/images/*` → media servers, `/api/*` → API servers" |
| Handles | Any TCP or UDP traffic | HTTP, HTTPS, gRPC, WebSockets |
| Pros | Fast; works for any protocol; can leave TLS encryption untouched | Smart routing; HTTP-aware health checks |
| Example | AWS Network Load Balancer | AWS Application Load Balancer, NGINX |

```{mermaid}
flowchart TB
  subgraph L4["Layer 4: sees only address and port"]
    c1[Client] --> nlb[L4 LB]
    nlb --> t1[Server]
  end
  subgraph L7["Layer 7: reads the HTTP request"]
    c2[Client] --> alb[L7 LB]
    alb -->|"/api/*"| api[API servers]
    alb -->|"/images/*"| img[Media servers]
  end
```

**Use Layer 7** for most web apps, where you want to route by URL. **Use Layer
4** for non-HTTP traffic (databases, game servers, DNS), very high connection
volume, or when servers must handle TLS themselves.

## Choosing a server

| Algorithm | How it picks | Good when |
| --- | --- | --- |
| Round robin | 1, 2, 3, 1, 2, 3... | Servers and requests are similar |
| Weighted | A bigger server gets a larger share | Servers differ in size, or for canaries (5% to the new version) |
| Least connections | The server with the fewest open connections | Requests vary in length |
| Hash | `hash(user_id)` always picks the same server | You want a user to stick to one server, e.g. for its cache |

No algorithm can predict how expensive a request will be, so servers should
still cap their own concurrency.

## Health checks

The load balancer calls something like `GET /health` on each server every few
seconds. After 3 failures in a row, it stops sending that server traffic. After
a few successes, it adds the server back.

Keep `/health` simple: "can this server handle requests?" Don't make it fail
because an optional dependency is down. If every server's health check depends
on the analytics service, an analytics outage removes *all* servers.

**Draining:** when deploying or scaling down, stop sending a server *new*
requests, let current ones finish (say, up to 30 seconds), then shut it down.
Long-lived connections like WebSockets need longer.

## TLS and client IPs

- **TLS termination:** the load balancer decrypts HTTPS, so it can read and route
  by URL. This is the usual setup.
- **TLS passthrough:** the load balancer forwards encrypted traffic untouched.
  The server decrypts it. Only possible with Layer 4.

Because the load balancer sits in the middle, servers see *its* IP address, not
the user's. The load balancer passes the real IP in a header like
`X-Forwarded-For`. Only trust that header when it comes from your own load
balancer, since clients can fake it.

## Sticky sessions

Sticky sessions send the same user to the same server every time, usually via a
cookie. They help apps that keep session data in server memory, but load gets
uneven, and if that server dies the user's session is lost.

Better: keep servers **stateless**. Store sessions in Redis or a signed token so
any server can handle any request.

## Global load balancing

```{mermaid}
flowchart TD
  users[Users] --> dns[Global DNS / anycast]
  dns -->|"Users in Europe"| eu[EU load balancer]
  dns -->|"Users in the US"| us[US load balancer]
  eu --> eu1[EU servers]
  us --> us1[US servers]
```

Global routing sends users to the nearest healthy region; each region's load
balancer then spreads traffic across its servers. Failing over to another region
only works if your data and dependencies also exist there.

## What goes wrong

- **Retry storms:** the client, load balancer, and service each retry, turning
  one failure into many requests.
- **Uneven long connections:** new servers get no traffic because existing
  WebSocket connections stay on old servers.
- **Slow but "healthy" servers:** a server passes health checks but responds
  slowly, dragging down response times.
- **Bad health checks:** one shared dependency fails and every server is
  removed.

## Further reading

- [Google Cloud: Layer 4 and Layer 7 load balancing](https://cloud.google.com/load-balancing/docs/load-balancing-overview)
- [AWS: Elastic Load Balancing routing algorithms](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
