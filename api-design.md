---
title: API Design
description: Practical choices for clear, secure, reliable APIs.
---

# API Design

An API is a contract between a client and a service. A good one stays
predictable when requirements change, clients retry, and things fail.

**Example:** a ride-sharing app's "request a ride" endpoint.

```http
POST /rides
Idempotency-Key: 9f1c2a
Authorization: Bearer eyJhbGciOi...

{ "pickup": "37.77,-122.41", "dropoff": "37.80,-122.27" }
```

```http
201 Created

{ "ride_id": "ride_881", "status": "searching" }
```

Even this small call answers the five questions every API should:

| Question | In the example |
| --- | --- |
| **What can clients do?** | Create a ride (`POST /rides`) |
| **What do requests, responses, and errors look like?** | JSON body in; `201` with `ride_id` out; `400` for a bad location |
| **How does it change without breaking clients?** | Add new optional fields; never rename or remove old ones |
| **What happens on retry?** | The `Idempotency-Key` stops a retried request from booking two rides |
| **Who is calling, and are they allowed?** | The bearer token says who the rider is |

**Authentication** proves who the caller is. **Authorization** decides what they
may do. Being logged in as rider 42 doesn't let you cancel rider 77's ride.

## In this section

- [API Types: REST, gRPC, and GraphQL](./api-design/api-types.md): three API
  styles and when to use each.
- [Authentication: Sessions vs. JWTs](./api-design/authentication.md): two ways
  to remember who's logged in.

```{tip}
Design the API around what the client needs to do, not around your database
tables.
```
