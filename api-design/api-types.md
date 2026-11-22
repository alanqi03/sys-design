---
title: API Types — REST, gRPC, and GraphQL
description: A plain-language comparison of REST, gRPC, and GraphQL, including their strengths, trade-offs, and best use cases.
---

# API Types: REST, gRPC, and GraphQL

All three connect a client to a service. They differ in **who decides the shape
of the response** and **how the contract is described**.

**Running example:** an order page that shows an order, its items, and the
customer's name.

| Type | Mental model | Best for |
| --- | --- | --- |
| **REST** | Resources at URLs, used with HTTP methods | Public APIs, web and mobile backends, CRUD |
| **gRPC** | Typed functions called with compact binary messages | Fast calls between your own internal services |
| **GraphQL** | One endpoint; the client asks for exactly the fields it needs | Several UIs that need different slices of related data |

**Default:** REST. Pick gRPC for internal service-to-service speed. Pick GraphQL
when varied UI data needs are a real problem.

## REST

Each thing is a **resource** with a URL, and the HTTP method says what to do:

```http
GET    /orders/abc123          → read the order
GET    /customers/42           → read the customer
POST   /orders                 → create an order
PATCH  /orders/abc123          → update it
DELETE /orders/abc123          → delete it
```

The order page needs two calls (the order, then the customer) unless you build
a custom endpoint that returns both.

**Pros**

- Works with everything: browsers, `curl`, proxies, CDNs, every language.
- Easy to read and debug, since it's plain JSON over HTTP.
- `GET` responses can be cached by HTTP caches and CDNs.

**Cons**

- A screen may need several calls, or get back fields it doesn't use.
- JSON is bigger and slower to parse than binary formats.

**Use it for** public APIs, web and mobile backends, and straightforward CRUD.

## gRPC

You define functions and messages in a `.proto` file:

```protobuf
service OrderService {
  rpc GetOrderPage(GetOrderRequest) returns (OrderPage);
  rpc WatchOrder(GetOrderRequest) returns (stream OrderUpdate);  // live updates
}

message GetOrderRequest { string order_id = 1; }
```

### What "generated code" means

You run the `protoc` compiler on that file, once per language you need. It
writes the **plumbing**, never the **business logic**:

| Generated for you | You still write |
| --- | --- |
| Classes for each message (`GetOrderRequest`, `OrderPage`) | What `GetOrderPage` actually does: query the database, apply rules |
| Code to turn messages into bytes and back | Your database and other service calls |
| A **client stub**: a `GetOrderPage()` method that sends the request over the network | Code that calls the stub and handles errors and timeouts |
| A **server base class** with an empty `GetOrderPage()` to fill in | Starting the server and wiring it up |

For example, the Orders team writes its server in Go and fills in the generated
method:

```go
// OrderServiceServer was generated from the .proto file.
func (s *server) GetOrderPage(ctx context.Context, req *pb.GetOrderRequest) (*pb.OrderPage, error) {
    order, err := s.db.LoadOrder(ctx, req.OrderId)   // your business logic
    if err != nil {
        return nil, err
    }
    return &pb.OrderPage{Status: order.Status}, nil
}
```

Meanwhile the checkout team, written in Python, generates a client from the same
`.proto` file and calls it like a local function:

```python
page = order_client.GetOrderPage(GetOrderRequest(order_id="abc123"), timeout=2)
```

Neither team writes HTTP, JSON, or serialization code, and both work from the
same contract. A wrong field name or type fails at compile time in typed
languages like Go and Java, and immediately at the call site in Python, instead
of becoming a confusing error on the server. It's still a network call, though,
so it can time out.

**Pros**

- A strict, typed contract; many mistakes are caught at compile time.
- Compact binary messages over HTTP/2, so it's fast.
- Built-in streaming in both directions.

**Cons**

- Binary payloads are hard to inspect by hand.
- Browsers can't call it directly; they need gRPC-Web or a proxy.
- Never reuse a field number when changing the schema.

**Use it for** internal services you control, especially in several languages
or at high request rates.

## GraphQL

The server publishes a schema of the available data. The client sends one query
listing exactly what it wants:

```graphql
query {
  order(id: "abc123") {
    status
    customer { name }
    items { productName quantity }
  }
}
```

The order page gets everything in one request, with no unused fields. A mobile
app could ask for just `status`.

**Pros**

- Each client fetches exactly the fields it needs.
- One request replaces several REST calls.
- The typed schema makes the API easy to explore.

**Cons**

- An innocent-looking query can trigger many database calls (the N+1 problem)
  unless resolvers batch their lookups.
- Harder to cache, rate-limit, and monitor than fixed endpoints.
- Overkill for a small, stable API.

**Use it for** UI-heavy products where web, mobile, and partner clients need
different, frequently changing views of the same data.

## Further reading

- [Roy Fielding's REST architectural style](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [GraphQL documentation](https://graphql.org/learn/)
