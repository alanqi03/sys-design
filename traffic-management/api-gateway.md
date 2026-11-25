---
title: API Gateway
description: Request validation, middleware, routing, protocol translation, response transformation, and caching at an API gateway.
---

# API Gateway

An API gateway is the single front door to your APIs. It handles the checks
every service would otherwise repeat (is the request valid, who is calling,
are they over their limit) and then forwards the request to the right service.

## Example: one request through the gateway

A mobile app sends:

```http
POST /v1/orders
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "productId": "p_42", "quantity": 2 }
```

```{mermaid}
flowchart LR
  app[Mobile app] --> v[1. Validate]
  v --> m["2. Auth, rate limit,<br/>request ID"]
  m --> r["3. Route"]
  r --> t["4. JSON → gRPC"]
  t --> svc[Order service]
  svc --> f["5. Format response"]
  f --> app
```

1. **Validate:** the body has a `productId` and a `quantity` between 1 and 100.
   If not, return `400` without bothering the order service.
2. **Middleware:** check the token, check that the user is under 100 requests a
   minute, and attach a request ID for tracing.
3. **Route:** `/v1/orders` goes to the order service.
4. **Translate:** convert the JSON request into the order service's gRPC call.
5. **Transform:** turn the gRPC result into the public JSON format.

Caching is skipped because creating an order isn't a repeatable read.

The order service still checks inventory, makes sure the user may buy the
product, and saves the order. **The gateway guards the door; business rules stay
in the service.**

## What a gateway does

### Request validation

Reject bad requests early:

| Problem | Response |
| --- | --- |
| Missing `productId` or wrong type | `400 Bad Request` |
| Body over the size limit | `413 Payload Too Large` |
| Wrong content type | `415 Unsupported Media Type` |

Check the *shape* of the request here. Rules like "does this product exist?"
belong in the service.

### Middleware

Shared checks that run on every request:

- authentication (see [Authentication](../api-design/authentication.md));
- rate limits and quotas per user or API key;
- request IDs, logging, metrics, and tracing;
- CORS and security headers.

Order matters: authenticate first, so you know whose rate limit to apply.

### Routing

Map public URLs to backend services:

```text
/users/*                 → user-service
/orders/*                → order-service
Header X-API-Version: 2  → orders-v2
5% of traffic            → orders-canary   (testing a new version)
```

The gateway picks the *service*; a load balancer then picks one *server* of that
service.

### Protocol translation

Clients speak one protocol; backends can speak another. The gateway can turn
JSON over HTTPS into a gRPC call, a serverless function call, or a message on a
queue. If the work goes onto a queue, return `202 Accepted` and a URL where the
client can check its status.

### Response transformation

Clean up responses before they leave:

- give every error the same shape, without stack traces;
- remove internal headers and fields;
- keep an old response format working for old clients during a migration.

```json
{ "error": { "code": "ORDER_NOT_FOUND", "message": "Order order_123 does not exist", "requestId": "req_789" } }
```

### Caching

The gateway can answer repeated `GET` requests from a cache. The cache key must
include everything that changes the answer, **including who the user is** for
private data. Otherwise one user can be shown another user's orders. Start by
caching public responses only. See [Caching Fundamentals](../caching/fundamentals.md).

## Keeping it reliable

- Run several gateway instances behind a load balancer, so the gateway isn't a
  single point of failure.
- Set timeouts on calls to services, and only retry requests that are safe to
  repeat.
- Limit request size and the number of in-flight requests per client.
- Keep business logic out of the gateway. A gateway full of business logic
  forces every team to deploy through it.

## Further reading

- [AWS API Gateway: Request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-validation-set-up.html)
- [AWS API Gateway: Caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
