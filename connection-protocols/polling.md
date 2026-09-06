---
title: Polling and Webhooks
description: Polling, long polling, and webhooks for learning when remote state changes.
---

# Polling and Webhooks

Polling and webhooks solve the same basic problem: **How does one system learn
that something changed in another system?** The main difference is who starts
the request.

| Pattern | Who starts the update request? | Best fit |
| --- | --- | --- |
| Polling | Receiver asks repeatedly | Simple or delay-tolerant updates |
| Long polling | Receiver asks; provider waits to answer | Near-real-time client updates over ordinary HTTP |
| Webhook | Provider calls the receiver | Server-to-server event notifications |

## Polling

Polling asks an API for current state on a schedule. Every request receives a
prompt response, even when nothing changed.

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /jobs/job_123
  A-->>B: 200 status is processing
  Note over B: Wait 5 seconds
  B->>A: GET /jobs/job_123
  A-->>B: 200 status is complete
  Note over B: Stop polling
```

A browser can poll while a job remains unfinished:

```javascript
async function waitForJob(jobId) {
  while (true) {
    const response = await fetch(`/api/jobs/${jobId}`);
    const job = await response.json();

    if (job.status === "complete" || job.status === "failed") return job;
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
}
```

Polling fits report generation, payment settlement, import progress, and
dashboards that tolerate slightly stale data.

### Choosing the interval

The interval trades freshness for server load. With a 10-second interval, an
update is noticed after about 5 seconds on average and almost 10 seconds in the
worst case:

```text
100,000 active clients / 10 seconds = 10,000 requests per second
```

That load exists even when nothing changes. Stop after a terminal state or when
the page becomes hidden. Add jitter so clients do not all poll on the same
clock boundary, and use exponential backoff after failures.

### Avoid downloading unchanged data

The server can return an `ETag`. The next request sends `If-None-Match`; if the
resource is unchanged, the server returns `304 Not Modified` without a response
body. This saves bandwidth, although the request still reaches the server.

```http
GET /api/jobs/job_123 HTTP/1.1
If-None-Match: "status-v7"
```

Use polling when simplicity matters more than immediate delivery. Use long
polling or [SSE](server-sent-events.md) when short intervals create too much
empty traffic.

## Long Polling

Long polling starts like polling, but the server does not immediately return an
empty answer. It holds the request until data is available or a timeout is
reached. After every response, the client opens the next request immediately.

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /events after cursor 41
  Note over A: Hold request until an event exists
  A-->>B: 200 event 42 OrderShipped
  B->>A: GET /events after cursor 42
  Note over A: No event before timeout
  A-->>B: 204 No Content
  B->>A: Reconnect after cursor 42
```

The client keeps one request outstanding:

```javascript
let cursor = 0;

async function receiveEvents() {
  while (true) {
    try {
      const response = await fetch(`/api/events?after=${cursor}`);
      if (response.status === 204) continue;

      const event = await response.json();
      render(event);
      cursor = event.id;
    } catch {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

### Server and delivery design

- Use asynchronous I/O instead of blocking one operating-system thread per
  waiting client.
- Return before the shortest proxy or load-balancer idle timeout, then let the
  client reconnect.
- Include a cursor or event ID so reconnecting clients can request missed
  events and deduplicate repeats.
- Retain enough event history to fill a temporary connection gap.
- Add jittered backoff so an outage does not create a reconnect storm.

Long polling is server-to-client only; client commands still use separate HTTP
requests. Choose it when an environment cannot reliably use streaming responses
or WebSockets. Prefer SSE for a modern one-way browser feed and WebSocket when
both sides exchange frequent messages.

## Webhooks

A webhook is a **server-to-server HTTP callback**. Instead of your service
asking a provider “Did anything change?”, you give the provider an HTTPS URL
and choose events. When one happens, the provider sends an HTTP request to that
URL.

For example, a payment provider can call
`POST https://shop.example.com/webhooks/payments` after a payment succeeds.

```{mermaid}
sequenceDiagram
  participant C as Consumer endpoint
  participant P as Payment provider
  participant Q as Durable queue
  participant W as Worker
  C->>P: Register HTTPS callback and event types
  Note over P: Payment succeeds
  P->>C: POST event payload and signature
  C->>C: Verify signature and event ID
  C->>Q: Store event for processing
  C-->>P: 202 Accepted
  Q-->>W: Deliver event
  W->>W: Update order and send receipt
```

A delivery might look like:

```http
POST /webhooks/payments HTTP/1.1
Content-Type: application/json
Webhook-Id: evt_8472
Webhook-Timestamp: 1788658200
Webhook-Signature: v1=...

{"type":"payment.succeeded","orderId":"order_123"}
```

### Receiving webhooks safely

1. Read the original request body and verify its signature with the shared
   webhook secret. Also validate the timestamp to limit replay attacks.
2. Check the event type, schema, and tenant or account before trusting the
   payload.
3. Atomically record the stable event or delivery ID and enqueue the work.
   If the ID already exists, treat the delivery as a harmless duplicate.
4. Return a `2xx` response quickly, then perform slow work in a background
   worker.

Providers often retry after a timeout or non-`2xx` response, but retry schedules
and retention vary. Design for duplicates and out-of-order events even if the
provider normally delivers in order. Fetch the current resource from the
provider when an event may be stale or incomplete.

### When to use webhooks

Use webhooks for payment status, source-control events, third-party account
changes, or completed asynchronous jobs. They avoid empty polling traffic and
can deliver quickly, but the receiver must expose a reliable HTTPS endpoint and
secure it against forged or replayed requests.

Polling remains useful when the receiver cannot accept inbound traffic, the
provider has no webhook support, or periodic reconciliation must recover from a
missed delivery. Many robust integrations use both: webhooks for fast updates
and slower polling to repair gaps.

### Further reading

- [RFC 6202: Long polling and streaming](https://www.rfc-editor.org/rfc/rfc6202.html)
- [RFC 9110: Conditional requests](https://www.rfc-editor.org/rfc/rfc9110.html#name-conditional-requests)
- [GitHub: Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
- [GitHub: Best practices for webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks)
