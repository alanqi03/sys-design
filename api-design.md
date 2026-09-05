---
title: API Design
description: Practical choices for clear, secure, reliable APIs.
---

# API Design

An API is a contract between a client and a service. Good API design makes that
contract predictable when requirements change, clients retry, and parts of the
system fail.

Before choosing URLs or field names, decide:

- **Operations:** What can clients do, and which operations change state?
- **Contracts:** What do requests, successful responses, and errors look like?
- **Compatibility:** How can the API evolve without breaking existing clients?
- **Reliability:** Which writes need idempotency, timeouts, or asynchronous work?
- **Security:** How does the service authenticate a caller and authorize an
  action?

Authentication proves **who the caller is**. Authorization decides **what that
caller may do**. They are related, but a valid login never removes the need for
authorization checks on protected resources.

## In this section

- [Authentication: Sessions vs. JWTs](./api-design/authentication.md) compares
  opaque server-side sessions with signed JWT bearer tokens, including request
  flows, trade-offs, and safe defaults.

```{tip}
Design the public contract around the client's job, not around the current
database tables or internal service boundaries.
```
