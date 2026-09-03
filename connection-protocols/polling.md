---
title: Polling
description: Periodic HTTP checks, latency and load trade-offs, conditional requests, and safe retry behavior.
---

# Polling

Polling asks the server for current state on a schedule. Every request receives
a prompt response, even when nothing changed. It is often the best first choice
when updates are infrequent or a few seconds of delay is acceptable.

## How it works

```{mermaid}
sequenceDiagram
  participant B as Browser
  participant A as API
  B->>A: GET /jobs/job_123
  A-->>B: 200 {status: "processing"}
  Note over B: Wait 5 seconds
  B->>A: GET /jobs/job_123
  A-->>B: 200 {status: "complete"}
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

Polling is a natural fit for report generation, payment settlement, import
progress, or a dashboard that only needs minute-level freshness.

## Choosing the interval

The interval sets a direct latency-versus-load trade-off. With a 10-second
interval, an update is noticed after about 5 seconds on average and nearly 10
seconds in the worst case. Meanwhile:

```text
100,000 active clients / 10 seconds = 10,000 requests per second
```

That load exists even when nothing changes. Stop polling when the page becomes
hidden or the operation reaches a terminal state. Add random jitter so all
clients do not poll on the same clock boundary, and back off after errors.

## Avoid downloading unchanged data

The server can return an `ETag` with a representation. The next request sends
`If-None-Match`; if the resource is unchanged, the server returns `304 Not
Modified` without the body. This saves bandwidth, although the request still
reaches the HTTP stack.

```http
GET /api/jobs/job_123 HTTP/1.1
If-None-Match: "status-v7"
```

## Reliability and scaling

- Treat each request independently so any API instance can answer it.
- Use timeouts and exponential backoff for failures; do not retry in a tight loop.
- Rate-limit abusive clients and cap how long a completed resource remains pollable.
- Cache shared, read-heavy state when a slightly stale answer is acceptable.
- Include a version or timestamp so the client can reject an older response that
  arrives after a newer one.

## When to move on

Use polling when implementation simplicity matters more than immediate updates.
Move to [long polling](long-polling.md) or [SSE](server-sent-events.md) when short
intervals create excessive empty traffic. Use [WebSocket](websockets.md) when
both sides exchange frequent low-latency messages.

## Further reading

- [RFC 6202, Section 2.1: short and long polling](https://www.rfc-editor.org/rfc/rfc6202.html#section-2.1)
- [RFC 9110: conditional requests](https://www.rfc-editor.org/rfc/rfc9110.html#name-conditional-requests)
