---
title: API Types — REST, gRPC, and GraphQL
description: An ELI18 comparison of REST, gRPC, and GraphQL, including their strengths, trade-offs, and best use cases.
---

# API Types: REST, gRPC, and GraphQL

REST, gRPC, and GraphQL can all connect a client to a service. They mainly
differ in **who defines the request shape**, **how the contract is described**,
and **which tools understand the protocol**.

## Summary

| Type | Quick mental model | Top advantages | Use it when |
| --- | --- | --- | --- |
| **REST** | The server exposes resources at URLs; clients use standard HTTP operations. | Familiar, easy to inspect, broad client and infrastructure support, HTTP caching | Building public APIs, browser/mobile backends, or straightforward CRUD services |
| **gRPC** | The server exposes typed functions; generated clients call them with compact binary messages. | Fast serialization, strict contracts, code generation, first-class streaming | Connecting internal services you control, especially across several programming languages or at high throughput |
| **GraphQL** | The server exposes a typed graph; each client asks for exactly the fields it needs. | Flexible response shapes, fewer aggregation round trips, strong schema tooling | Several UI clients need different views of related data or those views change frequently |

**Good default:** start with REST for a conventional public or product API.
Choose gRPC when internal-service efficiency and generated contracts matter.
Choose GraphQL when client-driven data composition solves a real UI problem.

## REST, ELI18

REST treats data and concepts as **resources**. Each resource has an address,
and standard HTTP methods describe the action:

```http
GET /users/42
GET /users/42/orders?limit=10
POST /orders
PATCH /orders/abc123
DELETE /orders/abc123
```

Most APIs called “REST APIs” send JSON over HTTP. Strictly speaking, REST is an
architectural style rather than a wire protocol: it emphasizes resources, a
uniform interface, stateless requests, cacheable responses, and layers such as
proxies and gateways.

### How a request works

```{mermaid}
flowchart LR
  client[Client] -->|GET /orders/abc123| api[REST API]
  api --> database[(Database)]
  database --> api
  api -->|JSON + HTTP status| client
```

The URL identifies the resource. The method communicates the intent, headers
carry metadata, the body carries data, and the HTTP status communicates the
outcome. For example, a successful lookup may return `200 OK`, while creating a
resource may return `201 Created`.

### Pros

- Works naturally with browsers, command-line tools, gateways, proxies, CDNs,
  and almost every programming language.
- Human-readable HTTP and JSON make requests easy to inspect and debug.
- Standard HTTP semantics support caching, conditional requests,
  authentication, and observability.
- Resources and operations can evolve without generating a client library for
  every change.

### Trade-offs

- A screen may need several requests, such as fetching a user and then their
  orders, or one custom endpoint that combines the data.
- A general response may return fields a client does not need or omit related
  fields it does need.
- JSON is larger and slower to encode than compact binary formats for some
  high-throughput internal workloads.
- An OpenAPI document can provide a strong contract, but teams must keep it in
  sync with the implementation.

### When to use REST

Use REST for a public API, a straightforward web or mobile backend, CRUD-style
resources, or any case where universal HTTP compatibility matters more than
maximum wire efficiency. It is usually the simplest choice for clients outside
your organization.

## gRPC, ELI18

gRPC makes a remote service look like a set of typed functions. First define
the service and messages in a `.proto` file:

```protobuf
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc WatchOrder(GetOrderRequest) returns (stream OrderUpdate);
}

message GetOrderRequest {
  string order_id = 1;
}
```

Tools generate client and server code for languages such as Go, Java, Python,
and C#. Calling `GetOrder` still crosses a network and can time out or fail; it
only feels like a local function in the source code.

gRPC commonly uses Protocol Buffers for compact binary messages and HTTP/2 for
transport. It supports a normal request/response call, client streaming, server
streaming, and bidirectional streaming.

### Pros

- The schema is a precise, language-neutral contract, and generated clients
  catch many mistakes before runtime.
- Binary messages are compact and efficient to serialize.
- HTTP/2 supports many concurrent calls over a connection and first-class
  streaming in either direction.
- Deadlines, cancellation, status codes, and metadata behave consistently
  across supported languages.

### Trade-offs

- Binary payloads are less convenient to read with ordinary browser and HTTP
  tools.
- Browsers usually need gRPC-Web or a translating proxy rather than calling a
  standard gRPC service directly.
- Schema evolution requires discipline: field numbers cannot be casually
  reused, and generated code becomes part of the build process.
- A function-looking call is still distributed; callers need deadlines,
  bounded retries, and idempotency for retried writes.

### When to use gRPC

Use gRPC between internal services when both ends are controlled by your teams,
when several languages need generated clients, or when high-throughput typed
messages and streaming matter. It is less natural as a general-purpose public
browser API.

## GraphQL, ELI18

GraphQL exposes a typed graph of available data. Instead of choosing among many
response-shaped endpoints, the client sends a query describing the exact
fields it wants:

```graphql
query OrderPage($orderId: ID!) {
  order(id: $orderId) {
    id
    status
    customer {
      name
    }
    items {
      productName
      quantity
    }
  }
}
```

The server validates the query against its schema and runs **resolvers** that
fetch each field from databases or downstream services. GraphQL is an API query
language and runtime; it is not a database and does not make slow data sources
fast by itself.

### Pros

- Each client requests only the fields it needs, which is useful when web,
  mobile, and partner clients need different views.
- One query can combine related data that would otherwise require several API
  calls.
- A typed schema, validation, introspection, and generated tooling make the API
  discoverable.
- Clients can often add an existing field to a screen without waiting for a new
  endpoint or response shape.

### Trade-offs

- A short-looking query can trigger expensive nested work or an N+1 query
  problem unless resolvers batch and cache carefully.
- Authorization must still be enforced for objects, fields, and actions; schema
  visibility is not permission.
- Arbitrary query shapes make cost limits, rate limits, caching, and performance
  monitoring more complicated than for fixed endpoints.
- It adds a schema and resolver layer even when an API only needs simple CRUD.

### When to use GraphQL

Use GraphQL for UI-focused APIs when multiple clients need different,
frequently changing combinations of related data. Avoid choosing it only to
reduce endpoint count; for a small stable API, REST is often easier to operate.

## Side-by-side example

Suppose a screen needs an order and the customer's name:

- **REST:** call `GET /orders/abc123`; either make another customer request or
  design the order response to include customer information.
- **gRPC:** call a predefined method such as `GetOrderPage`, returning the
  predefined response message.
- **GraphQL:** query `order`, then select `customer { name }` as part of the
  requested response shape.

None is automatically best. Choose based on who owns the clients, how often
their data needs change, the required performance, and the operational tooling
your team can support.

## Further reading

- [Roy Fielding's REST architectural style](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [GraphQL documentation](https://graphql.org/learn/)
