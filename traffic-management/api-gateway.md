---
title: API Gateway
description: Request validation, middleware, routing, protocol translation, response transformation, and caching at an API gateway.
---

# API Gateway

An API gateway is the front door for an API. It centralizes concerns that apply
to many services and presents clients with a stable contract even when the
backend is split into several services or uses different protocols.

```{mermaid}
flowchart LR
  client[Client] --> validation[1. Request validation]
  validation --> middleware[2. Middleware]
  middleware --> cache{6. Cache hit?}
  cache -->|Yes| transform[5. Response transformation]
  cache -->|No| routing[3. Routing]
  routing --> protocol[4. Protocol translation]
  protocol --> service[Backend service]
  service --> transform
  transform --> client
```

The exact order is configurable. Authentication and rate limiting normally run
before a cache lookup so cached private data cannot bypass authorization or
usage limits.

## 1. Request Validation

Reject malformed requests before they consume backend capacity. Validation can
check:

- required headers, path parameters, query parameters, and body fields;
- JSON or another payload schema, types, ranges, and allowed values;
- content type, HTTP method, body size, and header size; and
- a supported API version.

```yaml
POST /orders
Content-Type: application/json

body:
  productId: required string
  quantity: required integer, minimum 1, maximum 100
```

A missing `productId` can receive `400 Bad Request`, an unsupported media type
`415`, and an oversized body `413` without calling the order service. Keep
domain rules—such as whether the product exists or the user may buy it—in the
owning service. Gateway validation protects the interface; it should not become
a second copy of business logic.

## 2. Middleware

Middleware applies cross-cutting behavior consistently around a request:

- authenticate credentials and authorize an API scope;
- enforce quotas, per-client rate limits, and concurrency limits;
- attach a request ID and propagate trace context;
- record access logs, metrics, audit events, and billing usage;
- apply CORS, security headers, and IP or network policies; and
- enforce deadlines before work reaches the backend.

Order matters. Authentication must run before a per-user rate limit, and request
IDs should be created before logging. Fail closed for required security policy,
but decide explicitly whether failure of optional telemetry should block user
traffic.

Do not place every shared library or business workflow in the gateway. A large,
stateful gateway becomes a deployment bottleneck and couples unrelated teams.

## 3. Routing

Routing maps the external API to a backend destination. A Layer 7 gateway can
route by host, path, method, header, API version, or a deliberate traffic split:

```text
api.example.com/users/*        -> user-service
api.example.com/orders/*       -> order-service
api.example.com/search/*       -> search-service
Header: X-API-Version: 2       -> orders-v2
5% of eligible traffic         -> orders-canary
```

Use explicit priorities and a safe default route. Weighted routing enables
canaries and gradual migrations, but compare error and latency metrics and make
rollback automatic or fast. Do not trust an unrestricted client header to opt
into a privileged internal backend.

Routing and load balancing are separate steps: a route chooses a service or
target group, then a load-balancing algorithm chooses one healthy instance
within it.

## 4. Translate Protocols

The gateway can expose one client-friendly protocol and speak another to the
backend:

```{mermaid}
flowchart LR
  mobile[Mobile app<br/>JSON over HTTPS] --> gateway[API gateway]
  gateway -->|gRPC| catalog[Catalog service]
  gateway -->|Invoke| function[Serverless function]
  gateway -->|Publish task| queue[[Durable queue]]
```

Examples include REST/JSON to gRPC, an HTTP request to a serverless invocation,
or an accepted command to a queue. Translation must define:

- field and type mappings, including missing or unknown fields;
- deadlines, cancellation, streaming, and backpressure behavior;
- HTTP status, gRPC status, and application error mappings; and
- authentication identity and trace-context propagation.

Protocol translation cannot hide every semantic difference. Turning a
long-running operation into a queued task changes the API to asynchronous
completion and should normally return `202 Accepted` plus a status resource.

## 5. Response Transformation

A gateway may reshape a backend response before returning it:

- rename or remove fields and wrap an older payload in a new envelope;
- convert backend errors into a consistent public error format;
- add pagination links, request IDs, deprecation notices, or security headers;
- translate gRPC status into an HTTP status; and
- remove internal headers and implementation details.

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order order_123 does not exist",
    "requestId": "req_789"
  }
}
```

Transformation is useful as a compatibility layer during migrations. Keep it
small, deterministic, observable, and version controlled; complex joins and
business decisions belong in a backend or purpose-built aggregation service.
Do not leak internal exception text or stack traces.

## 6. Caching

Gateway caching can answer repeated safe reads without reaching the backend.
Define the cache key from every request property that can change the response:

```text
method + host + path + normalized query + tenant/user scope + representation
```

A cache key that omits authorization scope can expose one user's data to
another. Cache public or shared `GET` and `HEAD` responses first; cache
personalized responses only with deliberate isolation. Respect freshness
requirements, choose a TTL, define invalidation, and decide whether negative or
error responses may be cached.

Protect a cold or expired key from a cache stampede with request coalescing,
jittered TTLs, or stale-while-revalidate behavior. Measure hit ratio, latency,
evictions, response age, and backend load. See
[Caching Fundamentals](../caching/fundamentals.md).

## End-to-end example

For `POST /v1/orders`, the gateway might:

1. verify the JSON shape and reject a missing `productId`;
2. authenticate the token, enforce the user's quota, and attach a trace ID;
3. route `/v1/orders` to the order-service target group;
4. translate the JSON request to the service's gRPC method;
5. map the gRPC result to the public JSON envelope; and
6. skip caching because creating an order is not a safe read.

The order service still authorizes access to the domain object, checks
inventory, enforces idempotency, and commits the transaction. The gateway is
not the only security or correctness boundary.

## Reliability and operations

- Run gateway instances across failure domains and load balance them; the
  gateway must not become a new single point of failure.
- Set request, connection, and backend timeouts. Retry only safe or explicitly
  idempotent operations with a small retry budget.
- Bound body size, buffering, in-flight requests, and per-client concurrency.
- Drain connections during deployment and support long-lived SSE and WebSocket
  behavior deliberately.
- Redact secrets from logs and metrics while retaining request IDs, route,
  status, latency, cache outcome, and backend attempt details.
- Test configuration changes like code, use canaries, and retain a fast rollback.

## Further reading

- [AWS API Gateway: Request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-validation-set-up.html)
- [AWS API Gateway: Request and response transformation](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-parameter-mapping.html)
- [AWS API Gateway: Caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
