---
title: Servers
---

# Servers

A server takes a request, runs your code, and returns a result. In production
that's usually a fleet of identical servers behind a
[load balancer](../traffic-management/load-balancing.md).

**Running example:** an order service handling `GET /orders/123`.

## What happens in a request

| Step | Example | Time |
| --- | --- | --- |
| 1. Parse the request | Read the path and headers | ~0.1 ms |
| 2. Check who's calling | Verify the token; may this user see order 123? | ~1 ms |
| 3. Validate input | Is `123` a valid order ID? | ~0.1 ms |
| 4. Load data | Query the database | ~10 ms |
| 5. Build the response | Turn the row into JSON | ~0.5 ms |
| 6. Record it | Log, metrics, trace | ~0.1 ms |

Most of the time is spent **waiting** on the database, not computing. That's why
one server can work on many requests at once: while one waits, another runs.

## Handling many requests at once

- **Processes:** fully isolated, so one crash doesn't affect others, but heavy.
- **Threads:** lighter, and share memory, which needs care to avoid races.
- **Async I/O (event loop):** a few threads juggle thousands of waiting requests.
  Node.js and Python `asyncio` work this way.

Waiting-heavy work (database calls, HTTP calls) suits async or many threads.
CPU-heavy work, like resizing images, needs a limit of about one task per core,
often in a separate worker pool.

Always cap concurrency. A server can run out of memory or database connections
long before CPU hits 100%.

## Scaling

- **Vertical:** a bigger machine. Simple, but there's a ceiling, and it's still
  one machine that can fail.
- **Horizontal:** more machines. More capacity and redundancy, but traffic must
  be spread across them.

Horizontal scaling is easy when servers are **stateless**: keep data in the
database and sessions in Redis or a signed token, so any server can handle any
request.

## How many requests are in flight?

**Little's Law:**

$$
L = \lambda W
$$

In-flight requests ($L$) = arrival rate ($\lambda$) × time per request ($W$).

**Example:** 1,000 requests/s × 0.1 s each = **100 requests in flight** at any
moment. If the database slows down and each request takes 1 s, that becomes
1,000 in flight, possibly more than the server's connection pool can hold.

Response times jump sharply as a server nears full capacity, so plan headroom
for bursts, deployments, and lost servers. A common target is 50–70% average
utilization.

## Surviving overload

When more requests arrive than the server can handle:

| Technique | What it does | Example |
| --- | --- | --- |
| **Bounded queue** | Caps how many requests wait | At most 200 waiting; reject the rest |
| **Rate limit** | Caps each client | 100 requests/min per API key |
| **Load shedding** | Rejects work early with `503` | Better than a 30-second timeout |
| **Circuit breaker** | Stops calling a failing dependency for a while | Payments down → fail fast for 30 s |
| **Bulkhead** | Separate limits per workload | Reports can't use up the checkout threads |

Returning a quick `503` beats accepting work that will time out anyway. You can
also turn off optional features (recommendations) to keep core ones (checkout)
working.

## Deploys and monitoring

- **Deploys:** rolling (replace a few servers at a time), blue-green (switch all
  traffic to a new fleet), or canary (send 5% of traffic first). On shutdown,
  stop taking new requests and finish current ones.
- **Monitor:** request rate, error rate, latency (p50 and p99), and saturation
  (CPU, memory, connection pools). Logs explain single requests, metrics show
  trends, and traces show where time goes across services.
