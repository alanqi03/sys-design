---
title: Queue Fundamentals
description: Worker queues, acknowledgements, retries, dead-letter queues, idempotency, ordering, and scaling.
---

# Queue Fundamentals

## The basic model

A producer submits a task, the queue stores it, and one worker claims it. The
task is not necessarily deleted when delivered: reliable queues keep it in an
**in-flight** state until the worker acknowledges success.

```json
{
  "job_id": "job_987",
  "type": "GenerateThumbnails",
  "image_uri": "s3://uploads/photo-123.jpg",
  "sizes": [64, 256, 1024],
  "attempt": 1
}
```

```{mermaid}
stateDiagram-v2
  [*] --> Ready: producer enqueues
  Ready --> InFlight: worker receives
  InFlight --> Complete: ACK
  InFlight --> Ready: NACK or lease expires
  InFlight --> DLQ: maximum attempts reached
  Complete --> [*]
  DLQ --> [*]: inspect or redrive
```

The lifecycle is usually:

1. The worker receives the task; the queue marks it in flight or invisible.
2. Processing succeeds; the worker sends an **ACK**, and the queue removes or
   completes the task.
3. Processing fails; the worker sends a **NACK**, or its lease expires after a
   crash. The queue makes the task ready again and records another attempt.
4. The maximum attempt count is reached; the queue moves the task to a
   **dead-letter queue (DLQ)**.

Workers usually use queue-protocol acknowledgements, not HTTP `2XX` and `5XX`.
HTTP status codes apply only when a service delivers messages to an HTTP
endpoint.

## Good queue use cases

- **Media processing:** resize images, transcode video, or scan uploads.
- **Notifications:** send email, SMS, or push notifications outside the request
  path.
- **Webhook delivery:** retry temporary failures with backoff.
- **Document generation:** create invoices, reports, and exports.
- **Scheduled work:** delay a task until a future time.
- **Rate-controlled integration:** protect a slow third-party API from bursts.

Do not put work on a queue merely to hide an overloaded dependency. If work
arrives faster than it can permanently be processed, queue depth will continue
growing.

## ACK, NACK, and visibility

Different brokers use different terms, but two common models are:

- **Visibility lease:** receiving a message hides it for a fixed time. Success
  explicitly deletes it; otherwise it becomes visible again. Amazon SQS uses
  this model.
- **Acknowledgement:** the broker pushes or delivers a message and waits for an
  ACK. A NACK, rejected delivery, closed connection, or timeout can requeue it.
  RabbitMQ uses this model.

Set the lease or acknowledgement timeout above normal processing time, and let
long-running workers extend it with heartbeats. A timeout that is too short
causes concurrent duplicate work; one that is too long delays recovery after a
crash.

## Retries and backoff

Retry transient failures such as timeouts or rate limits. Permanent failures,
such as an invalid file format, should go directly to the DLQ.

A simple exponential schedule is:

```text
delay = min(base_delay × 2^(attempt - 1), maximum_delay) + jitter

attempt 1: immediate
attempt 2: about 5 seconds later
attempt 3: about 10 seconds later
attempt 4: about 20 seconds later
```

Jitter prevents thousands of failed tasks from retrying simultaneously. Define
the limit as **maximum attempts**, including the first delivery; “three retries”
can ambiguously mean four executions.

## Dead-letter queues

A DLQ isolates tasks that cannot complete. It prevents one poison message from
circulating forever, but it is not a trash can.

Store or retain:

- the original message and stable `job_id`;
- attempt count and timestamps;
- the last error category and diagnostic context; and
- the source queue and application version.

Alert on DLQ growth. Operators need a process to inspect, repair, replay, or
discard messages safely. Replaying without fixing the cause only creates another
failure loop.

## Idempotent workers

At-least-once queues can redeliver a task. A worker may finish its side effect,
crash before ACK, and then receive the same task again. An **idempotent worker**
makes repeated processing produce the same correct result as processing once.

For a payment job, pass its stable `job_id` to the payment provider as an
idempotency key. For an internal database operation, record completion under a
unique constraint:

```sql
BEGIN;

INSERT INTO processed_jobs (job_id)
VALUES ('charge_order_123')
ON CONFLICT DO NOTHING;

-- Perform the protected state change only when the insert succeeded.
COMMIT;
```

The completion marker and business change should share a transaction when
possible. A separate “check then act” can race with another worker.

Examples of naturally or deliberately idempotent effects:

- set `order_123.status = 'shipped'` rather than blindly incrementing a state;
- write an invoice to `invoices/order_123.pdf` rather than a random filename;
- upsert a search document using `product_456` as its ID; and
- send `welcome_email:user_42` only if its unique delivery record is absent.

## Ordering, priority, and delayed work

Global ordering limits parallelism because only one worker can safely advance
the entire queue. Prefer ordering within a smaller group such as `account_id` or
`order_id`. Even then, retries can block later work for that group.

Priority queues can starve low-priority jobs. Use a small number of priority
classes, separate queues with reserved worker capacity, or aging that raises a
job's priority over time.

Delayed delivery is useful for scheduled work and retry backoff. It should not
replace a workflow engine when a process lasts days, includes many dependent
steps, or requires human approval.

## Scaling workers

```{mermaid}
flowchart LR
  api[Upload API] --> original[(Object storage)]
  api --> jobs[[Image jobs queue]]
  jobs --> w1[Worker 1]
  jobs --> w2[Worker 2]
  jobs --> w3[Worker 3]
  w1 --> output[(Thumbnails)]
  w2 --> output
  w3 --> output
```

Scale workers from queue age and depth, not CPU alone. A backlog of 100,000 jobs
means something different when jobs take 10 milliseconds versus 10 minutes.

Watch:

- age of the oldest ready message;
- ready and in-flight message counts;
- processing latency and throughput;
- retry, timeout, and DLQ rates; and
- downstream saturation.

Use prefetch or concurrency limits so one worker does not claim more work than
it can finish. Graceful shutdown should stop new claims, finish or release
in-flight work, and then close the queue connection.

## Queue versus stream

| Question | Queue | Stream |
| --- | --- | --- |
| Message meaning | Task to perform | Fact that happened |
| Normal recipient | One worker | One member of every consumer group |
| After success | Removed or completed | Retained until the retention policy expires |
| Progress | Ready/in-flight/complete | Offset per consumer group |
| Example | `GenerateInvoicePDF` | `OrderPlaced` |

See [Streaming Fundamentals](../streaming/streaming-fundamentals.md) for ordered
logs, partitions, offsets, replay, and event-time processing.

## Design checklist

- What makes a job unique, and how will duplicate processing be made safe?
- Is the failure transient, permanent, or unknown?
- What are the visibility timeout, backoff schedule, and maximum attempts?
- What ordering or priority is truly required?
- How will queue depth and oldest-message age drive autoscaling?
- Who monitors, repairs, and replays the DLQ?

## Sources and further reading

- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/4.1/confirms)
- [RabbitMQ consumers and prefetch](https://www.rabbitmq.com/docs/4.1/consumers)
