---
title: Polling and Webhooks
description: Polling, long polling, and webhooks for learning when remote state changes.
---

# Polling and Webhooks

All three answer the same question: **how does one system find out that
something changed in another?** The difference is who starts the request.

| Pattern | Who sends the request, and when | When the answer comes | Example |
| --- | --- | --- | --- |
| Polling | The client, every N seconds | Right away, even if nothing changed | A browser asks "is my report ready?" every 5 s |
| Long polling | The client, again right after each answer | Only when there's news, or after a timeout (~30 s) | A browser waits for the next chat message |
| Webhook | The provider, when the event happens | No waiting: the request *is* the news | Stripe calls your server when a payment succeeds |

## Polling

Ask for the current state on a schedule. The server answers immediately, even
if nothing changed.

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /jobs/job_123
  A-->>B: status: processing
  Note over B: wait 5 s
  B->>A: GET /jobs/job_123
  A-->>B: status: complete
```

```javascript
async function waitForJob(jobId) {
  while (true) {
    const job = await (await fetch(`/api/jobs/${jobId}`)).json();
    if (job.status === "complete" || job.status === "failed") return job;
    await new Promise(r => setTimeout(r, 5000));
  }
}
```

**The trade-off is freshness vs. load.** With a 10-second interval, updates show
up about 5 seconds late on average. And the load is constant even when nothing
changes:

```text
100,000 open pages ÷ 10 s = 10,000 requests/second
```

Tips:

- Stop polling when the job is done or the tab is hidden.
- Add random jitter so clients don't all poll at the same moment.
- Use `ETag` / `If-None-Match` so unchanged responses come back as a tiny
  `304 Not Modified`.

**Use it for** report generation, import progress, and dashboards where a few
seconds' delay is fine.

## Long Polling

The client asks, and the server **holds the request open** until there's news
or a timeout (say, 30 seconds). As soon as a response arrives, the client asks
again.

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /events?after=41
  Note over A: waits...
  A-->>B: event 42: OrderShipped
  B->>A: GET /events?after=42
  Note over A: 30 s, nothing new
  A-->>B: 204 No Content
  B->>A: GET /events?after=42
```

The `after=42` cursor means a reconnecting client never misses or repeats an
event.

- The server needs async I/O, so thousands of waiting requests don't each hold
  a thread.
- Time out before your load balancer's idle timeout.
- Only server-to-client: the client sends commands with normal requests.

**Use it** when you need near-real-time updates but can't use SSE or
WebSockets. Otherwise prefer [SSE](server-sent-events.md) for one-way updates.

## Webhooks

A webhook flips the direction: you give the provider a URL, and it calls you
when something happens.

**Example:** you register `https://shop.example.com/webhooks/payments` with your
payment provider. When a payment succeeds, it sends:

```http
POST /webhooks/payments
Webhook-Id: evt_8472
Webhook-Timestamp: 1788658200
Webhook-Signature: v1=5d41...

{"type": "payment.succeeded", "orderId": "order_123"}
```

```{mermaid}
sequenceDiagram
  participant P as Payment provider
  participant C as Your endpoint
  participant Q as Queue
  participant W as Worker
  P->>C: POST payment.succeeded
  C->>C: verify signature, check evt_8472 not seen
  C->>Q: enqueue
  C-->>P: 200 OK (fast)
  Q->>W: mark order paid, send receipt
```

### Receiving webhooks safely

1. **Verify the signature** with the shared secret, so nobody can fake a
   "payment succeeded" call. Reject old timestamps to block replays.
2. **Deduplicate** by `Webhook-Id`. Providers retry, so you'll get duplicates.
3. **Respond quickly** with `2xx` and do the slow work in a background worker.
   Otherwise the provider times out and retries.
4. **Don't assume order.** If in doubt, fetch the latest state from the
   provider's API.

**Use webhooks for** payments, GitHub events, and finished third-party jobs:
server-to-server notifications.

Many integrations use **both**: webhooks for fast updates, plus an occasional
poll to catch any webhook that was missed.

## Further reading

- [RFC 6202: Long polling and streaming](https://www.rfc-editor.org/rfc/rfc6202.html)
- [GitHub: Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
- [GitHub: Best practices for webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks)
