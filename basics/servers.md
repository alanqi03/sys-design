---
title: Servers
---

# Servers

A server accepts work, applies business logic, and returns or emits a result.
At small scale this may be one process on one machine. At larger scale it is a
fleet of instances behind a [load balancer](../traffic-management/load-balancing.md).

## The request lifecycle

A typical service performs the following work:

1. accept and parse a request;
2. authenticate the caller and authorize the action;
3. validate input;
4. read or update state through dependencies;
5. construct a response; and
6. record metrics, logs, and traces.

CPU time is only part of the cost. Many services spend most of a request waiting
for databases, caches, or other services. Concurrency lets a server use that
waiting time to process other requests.

## Processes, threads, and asynchronous I/O

Processes isolate memory and failures but are comparatively expensive. Threads
share memory and are cheaper, but require careful synchronization. Event-driven
servers use asynchronous I/O to manage many connections with a small number of
threads. The right model depends on the language runtime and whether the work is
CPU-bound or I/O-bound.

CPU-heavy work needs bounded parallelism and often a separate worker pool.
Unbounded concurrency can exhaust memory, file descriptors, connection pools, or
downstream capacity long before CPU reaches 100%.

## Scaling

**Vertical scaling** gives one machine more CPU, memory, or storage. It is simple
but has hardware limits and can leave a large failure domain. **Horizontal
scaling** adds instances. It provides greater capacity and redundancy, but
requires traffic distribution and coordination.

Stateless request handlers are easiest to scale horizontally. Durable state
should normally live in a database or object store, while short-lived session
state can live in a shared cache or be encoded in a signed token. If requests
must return to a particular server, deployments and failures become harder.

## Capacity and queueing

Little's Law provides a useful approximation for a stable system:

$$
L = \lambda W
$$

where $L$ is the average number of requests in the system, $\lambda$ is the
arrival rate, and $W$ is the average time each request spends in the system. At
1,000 requests per second and 100 ms average latency, roughly 100 requests are in
flight on average.

As utilization approaches a resource's limit, queues grow and latency rises
sharply. Capacity plans therefore need headroom for bursts, failures, and
deployments—not merely the average load.

## Overload and resilience

A resilient server protects itself and its dependencies with:

- **bounded queues** to cap memory use;
- **rate limits** to control unfair or excessive traffic;
- **load shedding** to reject work it cannot finish in time;
- **circuit breakers** to stop repeatedly calling a failing dependency; and
- **bulkheads** to prevent one workload from consuming every resource.

Graceful degradation can preserve core operations by disabling optional
features. Returning a prompt `503 Service Unavailable` is often better than
accepting work that will time out after consuming resources.

## Deployment and observability

Rolling, blue-green, and canary deployments trade speed against risk and
resource cost. Instances should stop accepting new traffic, finish bounded
in-flight work, and then exit during a graceful shutdown.

At minimum, observe request rate, error rate, and latency, plus saturation of
critical resources. Structured logs explain individual events; metrics reveal
trends; distributed traces show how time moves through a request path.
