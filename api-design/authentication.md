---
title: Authentication — Sessions vs. JWTs
description: How server-side sessions and signed JWT bearer tokens work, their trade-offs, and when to use each.
---

# Authentication: Sessions vs. JWTs

After Ana logs in, every later request must prove "this is Ana" without sending
her password again. There are two common ways:

- **Session:** the server stores "session `7bY…` = Ana" and gives the browser
  only that random ID.
- **JWT:** the server gives the client a signed token that itself says "this is
  Ana, valid for 15 minutes."

Either way, anyone who steals the credential can act as Ana until it expires.
Always use HTTPS and never log credentials.

## Sessions

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as App server
  participant S as Session store (Redis)

  B->>A: POST /login (email + password)
  A->>S: save 7bY… → user 42, expires in 30 min
  A-->>B: Set-Cookie: session=7bY…
  Note over B,A: Later
  B->>A: GET /account (Cookie: session=7bY…)
  A->>S: who is 7bY…?
  S-->>A: user 42
  A-->>B: Ana's account page
```

The cookie holds only a random ID. To log Ana out everywhere, delete her session
from the store, and it stops working on the very next request.

```http
Set-Cookie: session=7bY...Q2; Secure; HttpOnly; SameSite=Lax
```

`HttpOnly` stops JavaScript from reading the cookie, `Secure` sends it only over
HTTPS, and `SameSite` limits cross-site requests.

## JWTs

```{mermaid}
sequenceDiagram
  participant C as Client
  participant Auth as Auth service
  participant API as Orders API

  C->>Auth: POST /login
  Auth-->>C: JWT signed with Auth's private key
  Note over C,API: Later
  C->>API: GET /orders (Authorization: Bearer JWT)
  API->>API: check signature with Auth's public key
  API-->>C: Ana's orders
```

The token's payload is readable JSON:

```json
{ "sub": "user_42", "aud": "orders-api", "exp": 1788567300 }
```

The Orders API checks the signature and expiry itself, with no database lookup.
That's the appeal: any service can verify the token on its own.

The catch: a JWT stays valid until it expires. If Ana's account is banned, her
token still works until `exp`, unless every API also checks a denylist. Keep
access tokens short-lived (5–15 minutes) and use a refresh token to get new
ones.

A JWT is signed, **not encrypted**. Never put secrets or personal data in it.

## Comparison

| | Session | JWT |
| --- | --- | --- |
| Client holds | A random ID | Signed claims |
| Each request | Look up the ID in the store | Verify the signature |
| Instant logout or ban | Yes, delete the session | No, valid until it expires |
| Permission changes | Take effect immediately | Take effect at the next token |
| Typical fit | Your own website | Mobile apps, third-party clients, service-to-service |

## Which to choose

- **Your own web app:** sessions. They're simpler, and logout is instant.
- **Many services or non-browser clients:** short-lived JWTs plus refresh
  tokens.

Don't store long-lived tokens in browser `localStorage`, where any injected
script can read them. A common browser setup keeps a session cookie in the
browser while a backend server holds the tokens it uses to call other APIs.

## Security checklist

- Use HTTPS, rate-limit login, and never put credentials in URLs or logs.
- Check authorization on every request, not just whether the user is logged in.
- **Sessions:** use random IDs, set `Secure; HttpOnly; SameSite`, issue a new
  ID after login, and delete it on logout.
- **JWTs:** verify the signature, accept only expected algorithms, check `iss`,
  `aud`, and `exp`, keep tokens short-lived, and rotate signing keys.

## Further reading

- [RFC 7519: JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519.html)
- [RFC 8725: JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 6265: HTTP State Management Mechanism](https://www.rfc-editor.org/rfc/rfc6265.html)
