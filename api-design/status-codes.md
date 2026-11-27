---
title: HTTP Status Codes
description: The HTTP status codes worth knowing, what each means, and which ones are safe to retry.
---

# HTTP Status Codes

Every HTTP response starts with a three-digit **status code** that tells the
client what happened, before it even reads the body.

**Running example:** the ride-sharing API from [API Design](../api-design.md).

```http
POST /rides            →   201 Created
GET  /rides/ride_999   →   404 Not Found
```

## The first digit is what matters most

| Range | Meaning | Think of it as |
| --- | --- | --- |
| **2xx** | Success | "Done." |
| **3xx** | Redirect | "Look over there instead." |
| **4xx** | Client error | "**You** made a mistake. Fix the request." |
| **5xx** | Server error | "**We** made a mistake. Try again later." |

(1xx codes exist too, but you'll rarely deal with them directly.)

The most useful rule: **4xx means the caller must change something, and 5xx
means the server has a problem.** Sending the exact same request again won't
fix a 4xx, but might work for a 5xx.

## 2xx: Success

| Code | Name | When to use it | Ride example |
| --- | --- | --- | --- |
| **200** | OK | The request worked, and here's the result | `GET /rides/ride_881` returns the ride |
| **201** | Created | A new thing was created | `POST /rides` booked a ride. Return the new `ride_id` |
| **202** | Accepted | Got it, but the work happens later | `POST /reports` starts building a trip report. Check back for status |
| **204** | No Content | It worked, and there's nothing to send back | `DELETE /rides/ride_881` cancelled the ride |

## 3xx: Redirects

| Code | Name | When to use it |
| --- | --- | --- |
| **301** | Moved Permanently | The URL changed for good. `/v1/trips` now lives at `/v1/rides` |
| **302** | Found (temporary redirect) | Go elsewhere for now, such as a login page or a presigned S3 download URL |
| **304** | Not Modified | "You already have the latest version." The client reuses its cached copy, so no body is sent |

`304` pairs with the `ETag` header: the client sends `If-None-Match: "v7"`, and
if the data is still version 7, the server replies `304` with an empty body.
See [Polling](../connection-protocols/polling.md#polling).

## 4xx: The client's mistake

These are the ones you'll use and see the most.

| Code | Name | Means | Ride example |
| --- | --- | --- | --- |
| **400** | Bad Request | The request is broken or invalid | The pickup location is missing |
| **401** | Unauthorized | "Who are you?" No login, or the token is invalid or expired | No `Authorization` header |
| **403** | Forbidden | "I know who you are, but you can't do this" | Rider 42 tries to cancel rider 77's ride |
| **404** | Not Found | That thing doesn't exist | `GET /rides/ride_999` |
| **409** | Conflict | The request clashes with the current state | Cancelling a ride that's already finished |
| **422** | Unprocessable Content | Well-formed, but fails a business rule | The dropoff is 2,000 miles away |
| **429** | Too Many Requests | Slow down, you hit the rate limit | An app calls `/rides` 500 times a minute |

### Two pairs that are easy to mix up

**401 vs. 403.** The name "Unauthorized" is misleading.

- **401** = not logged in. *Fix:* log in or refresh the token.
- **403** = logged in, but not allowed. *Fix:* nothing. Logging in again won't
  help.

**400 vs. 422.** Many APIs only use 400, and that's fine. If you use both:

- **400** = the request itself is broken (bad JSON, missing field, wrong type).
- **422** = the request is fine, but the values break a rule.

Some APIs return **404 instead of 403** so they don't reveal that something
exists. For example, rider 42 asking for rider 77's ride gets "not found."

### 429 and Retry-After

A rate-limited response should say when to come back:

```http
429 Too Many Requests
Retry-After: 30
```

The client waits 30 seconds before trying again.

## 5xx: The server's mistake

| Code | Name | Means | Example |
| --- | --- | --- | --- |
| **500** | Internal Server Error | Something broke in our code | An unhandled exception |
| **502** | Bad Gateway | The load balancer or gateway got a bad answer from the server behind it | The ride service crashed mid-response |
| **503** | Service Unavailable | Temporarily can't handle requests | Overloaded, or down for maintenance |
| **504** | Gateway Timeout | The gateway gave up waiting for the server behind it | The ride service took 60 s to respond |

`502`, `503`, and `504` usually come from the
[load balancer or API gateway](../api-gateway-load-balancing.md) in front of
your service, not from your code. `502` means the server answered badly, and
`504` means it didn't answer in time.

## Which ones should a client retry?

| Response | Retry? |
| --- | --- |
| 2xx | No, it worked |
| 400, 401, 403, 404, 409, 422 | **No.** The same request will fail again. Fix it first (for 401, refresh the token) |
| 429 | **Yes**, after the `Retry-After` time |
| 502, 503, 504 | **Yes**, with exponential backoff (1 s, 2 s, 4 s...) and jitter |
| 500 | Maybe once or twice. It might be a bug that fails every time |
| Timeout, no response | Yes, but only if the request is safe to repeat |

**Be careful retrying `POST`.** If `POST /rides` timed out, the ride may
already have been booked. Send an `Idempotency-Key`, so a retry returns the
first result instead of booking a second ride.

## Good error responses

Use the right status code **and** a body that explains the problem in a
consistent shape:

```http
422 Unprocessable Content

{
  "error": {
    "code": "DROPOFF_TOO_FAR",
    "message": "Dropoff must be within 100 miles of pickup.",
    "field": "dropoff"
  }
}
```

- The **status code** tells programs what kind of failure it is (retry or not?).
- The **error code** (`DROPOFF_TOO_FAR`) lets the app show the right message.
- The **message** helps the developer debug.

**Don't return `200 OK` with an error inside the body.** Retry logic,
monitoring, and caches all look at the status code, and they'll think it
worked.

## Other API styles

- **gRPC** uses its own codes, such as `NOT_FOUND`, `PERMISSION_DENIED`, and
  `UNAVAILABLE`, which roughly match 404, 403, and 503.
- **GraphQL** usually returns `200` and lists problems in an `errors` field, so
  clients must check the body. See [API Types](./api-types.md).

## Cheat sheet

If you only remember a dozen:

```text
200 OK            201 Created         204 No Content
301 Moved         304 Not Modified
400 Bad Request   401 Not logged in   403 Not allowed
404 Not Found     409 Conflict        429 Too Many Requests
500 Server bug    502 Bad Gateway     503 Unavailable    504 Timeout
```

## Further reading

- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [RFC 9110: HTTP Semantics, status codes](https://www.rfc-editor.org/rfc/rfc9110.html#name-status-codes)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
