---
title: Authentication — Sessions vs. JWTs
description: How server-side sessions and signed JWT bearer tokens work, their trade-offs, and when to use each.
---

# Authentication: Sessions vs. JWTs

After a user logs in, the client needs a credential for later requests. Two
common choices are an **opaque session ID** and a **JSON Web Token (JWT)**.

- With a **session**, the credential points to state stored by the server.
- With a **JWT**, the credential contains signed claims that an API can verify.

Both are bearer credentials: anyone who steals one can generally act as its
owner until it expires or is revoked. Use HTTPS and never log the credential.

## The two flows

### Session-based authentication

```{mermaid}
sequenceDiagram
  participant C as Browser
  participant A as Application server
  participant S as Session store

  C->>A: POST /login + credentials
  A->>A: Verify credentials
  A->>S: Store sessionId → user, expiry, metadata
  S-->>A: Stored
  A-->>C: Set-Cookie: sessionId=opaque-random-value
  Note over C,A: Later request
  C->>A: Cookie: sessionId=opaque-random-value
  A->>S: Look up sessionId
  S-->>A: User and session state
  A-->>C: Protected response
```

The server generates a long, random session ID and stores the associated user
and expiry in Redis, SQL, or another shared session store. The browser keeps
only the opaque ID and automatically returns it in the `Cookie` header.

```http
Set-Cookie: __Host-session=7bY...random...Q2; Path=/; Secure; HttpOnly; SameSite=Lax
```

```http
GET /account HTTP/1.1
Host: api.example.com
Cookie: __Host-session=7bY...random...Q2
```

Deleting the session record can log the user out immediately. Rotate the ID
after login or a privilege change to prevent session-fixation attacks.

### JWT-based authentication

```{mermaid}
sequenceDiagram
  participant C as Client
  participant A as Authentication service
  participant P as Protected API

  C->>A: POST /login + credentials
  A->>A: Create claims and sign with private key
  A-->>C: Short-lived JWT access token
  Note over C,P: Later request
  C->>P: Authorization: Bearer &lt;JWT&gt;
  P->>P: Verify signature with public key<br/>and validate claims
  P-->>C: Protected response
```

The authentication service creates claims such as the subject and expiry, then
signs them with its **private key**. An API can use the corresponding public key
to verify that the token was issued by that service and was not modified. This
example uses asymmetric signing; JWTs can also use a shared-secret MAC, but any
service with that secret can both verify and create tokens.

```json
{
  "iss": "https://auth.example.com",
  "sub": "user_123",
  "aud": "orders-api",
  "iat": 1788566400,
  "exp": 1788567300
}
```

```http
GET /orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

A signed JWT is **not encrypted**: its header and payload are readable. Do not
put passwords, secrets, or unnecessary personal data in its claims.

## Cookie or `Authorization` header?

These are different transport choices:

- An `HttpOnly` cookie is automatically sent by the browser in the `Cookie`
  header. JavaScript cannot read it to copy it into another header.
- A bearer token is explicitly added by the client as
  `Authorization: Bearer <token>`.

A JWT *can* be placed in a cookie, but it then uses cookie transport and inherits
its CSRF considerations. A token accessible to browser JavaScript is more
exposed to token theft during an XSS bug; avoid casually putting long-lived
access tokens in `localStorage`.

For browser applications, a common alternative is a **backend for frontend
(BFF)**: the browser receives a secure session cookie, while the BFF stores or
obtains the access tokens used to call downstream APIs.

## Quick comparison

| Question | Server-side session | JWT access token |
| --- | --- | --- |
| What does the client hold? | Small, opaque random ID | Signed claims plus metadata |
| Work on each request | Look up the session | Verify signature and claims |
| Server-side state | Session record is required | No session lookup is required for the access token |
| Immediate revocation | Delete or disable the session | Usually needs a denylist or user/token-version lookup |
| Permission freshness | Read current state from the session or database | Claims may be stale until the token expires |
| Scaling requirement | Share or replicate the session store | Distribute verification keys and rotate them safely |
| Typical fit | First-party browser application | APIs, mobile clients, and service-to-service calls |

JWT authentication does not make the whole system stateless. Refresh-token
rotation, logout, revocation, and account disablement often require stored
state. Adding a denylist lookup to every request also gives up much of the
no-session-lookup advantage.

## When to choose each

Choose **sessions** when:

- the main client is a first-party web browser;
- immediate logout or account revocation matters; or
- permissions change often and should take effect quickly.

Choose **short-lived JWT access tokens** when:

- many independent APIs need to verify the same identity;
- mobile, command-line, or third-party clients call the API; or
- service-to-service calls should be verified without a central session lookup.

For a conventional browser application, a secure server-side session is often
the simpler default. For a distributed API ecosystem, use short-lived access
tokens and keep any refresh tokens protected, rotated, and revocable.

## Security checklist

For either approach:

- use HTTPS, rate-limit login attempts, and never place credentials in URLs;
- authorize every protected operation rather than treating authentication as
  permission to access everything; and
- avoid logging cookies, access tokens, or refresh tokens.

For sessions:

- generate unpredictable IDs; store only the ID—not user data—in the cookie;
- set `Secure`, `HttpOnly`, and an appropriate `SameSite` policy;
- add CSRF protection where cookie policy alone is insufficient; and
- expire records, rotate the ID after login, and delete it on logout.

For JWTs:

- allow only expected algorithms and verify the signature;
- validate `iss` (issuer), `aud` (audience), `exp` (expiry), and, when used,
  `nbf` (not before); and
- use short expirations, rotate signing keys, and plan for emergency revocation.

## Further reading

- [RFC 7519: JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519.html)
- [RFC 6750: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
- [RFC 8725: JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 6265: HTTP State Management Mechanism](https://www.rfc-editor.org/rfc/rfc6265.html)
