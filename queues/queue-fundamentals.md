---
title: Queue Fundamentals
description: Worker queues, acknowledgements, retries, dead-letter queues, idempotency, ordering, and scaling.
---

# Queue Fundamentals

**Running example:** a photo app. When a user uploads a photo, the API saves it
and puts a job on a queue. Workers pick up jobs and create thumbnails, so the
upload request returns right away.

```json
{
  "job_id": "job_987",
  "type": "GenerateThumbnails",
  "image_uri": "s3://uploads/photo-123.jpg",
  "sizes": [64, 256, 1024]
}
```

## The job lifecycle

A worker taking a job doesn't delete it. The job stays **in flight** until the
worker confirms success.

```{mermaid}
stateDiagram-v2
  [*] --> Ready: API enqueues
  Ready --> InFlight: worker takes it
  InFlight --> Done: worker ACKs
  InFlight --> Ready: worker fails or crashes
  InFlight --> DLQ: too many attempts
  Done --> [*]
  DLQ --> [*]: inspect and fix
```

1. A worker takes the job. The queue hides it from other workers.
2. **Success:** the worker sends an **ACK** and the job is removed.
3. **Failure or crash:** the job becomes ready again, and another worker can
   retry it.
4. **Too many failures:** the job moves to a **dead-letter queue (DLQ)** for a
   human to look at.

Queues detect a crashed worker in one of two ways:

- **Visibility timeout** (SQS): the job is hidden for, say, 5 minutes. If the
  worker hasn't deleted it by then, it reappears.
- **Connection-based ACK** (RabbitMQ): if the worker's connection drops before
  it ACKs, the job goes back on the queue.

Set the timeout longer than a normal job takes. Too short, and two workers
process the same photo at once. Too long, and a crashed job sits stuck.

## When to use a queue

- Image and video processing
- Sending emails, SMS, and push notifications
- Delivering webhooks with retries
- Generating reports and PDFs
- Smoothing bursts before a slow third-party API

A queue absorbs **bursts**. If jobs arrive faster than workers can *ever* finish
them, the queue just grows forever. Add workers instead.

## Retries and backoff

Retry temporary failures (timeouts, rate limits). Send permanent failures (a
corrupt image file) straight to the DLQ, because retrying won't help.

Wait longer between each attempt, plus a little randomness (jitter) so failed
jobs don't all retry at the same instant:

```text
attempt 1: immediately
attempt 2: ~5 s later
attempt 3: ~10 s later
attempt 4: ~20 s later → then DLQ
```

## Dead-letter queues

A DLQ catches jobs that keep failing, so one bad job doesn't retry forever.
Alert when the DLQ grows, find and fix the cause, then replay the jobs. Replaying
without a fix just fails again.

## Idempotent workers

Queues deliver jobs **at least once**. A worker can finish the thumbnails, crash
before sending the ACK, and the job runs again. So processing a job twice must
give the same result as processing it once.

Easy ways to get there:

- **Write to a fixed location:** save to `thumbs/photo-123-256.jpg`, not a random
  file name. A second run just overwrites the same file.
- **Set, don't increment:** `status = 'shipped'` is safe to repeat;
  `count = count + 1` is not.
- **Record finished jobs** in the same transaction as the work:

  ```sql
  BEGIN;
  INSERT INTO processed_jobs (job_id) VALUES ('job_987')
    ON CONFLICT DO NOTHING;   -- 0 rows inserted = already done, so skip
  -- ...do the work only if the insert succeeded...
  COMMIT;
  ```

- **Pass an idempotency key** to external APIs: payment providers won't charge
  twice for the same key.

## Ordering and priority

- **Ordering:** strict order across the whole queue means only one worker can
  run at a time. Usually you only need order per user or per order, which still
  lets different users' jobs run in parallel.
- **Priority:** low-priority jobs can starve. Use a separate queue with its own
  workers for urgent jobs.
- **Delays:** "run this in 10 minutes" is fine. For multi-day processes with many
  steps, use a workflow engine instead.

## Scaling workers

Scale on **queue depth and the age of the oldest job**, not on CPU. 10,000
waiting jobs is nothing if each takes 10 ms, and a crisis if each takes 10
minutes.

Also watch retry and DLQ rates, and whether the downstream service is
overloaded. When shutting a worker down, stop taking new jobs and finish (or
release) the ones it holds.

## Queue vs. stream

| | Queue | Stream |
| --- | --- | --- |
| A message is | A task to do | A fact that happened |
| Read by | One worker | Every interested consumer group |
| After processing | Deleted | Kept until retention expires |
| Example | `GenerateThumbnails` | `OrderPlaced` |

See [Streaming Fundamentals](../streaming/streaming-fundamentals.md).

## Checklist

- Is it safe if a job runs twice?
- What's the timeout, retry schedule, and maximum number of attempts?
- Which failures are permanent and should skip retries?
- Do you need ordering, and only per user or across everything?
- What drives autoscaling, and who watches the DLQ?

## Further reading

- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/4.1/confirms)
