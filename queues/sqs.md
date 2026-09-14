---
title: Amazon SQS
description: A concise system-design guide to Amazon Simple Queue Service.
---

# Amazon SQS

Amazon Simple Queue Service (SQS) is a managed, pull-based message queue. AWS
runs the brokers; applications send messages, workers poll for them, and a
successful worker deletes its message using a receipt handle.

## Standard versus FIFO

| | Standard queue | FIFO queue |
| --- | --- | --- |
| Delivery | At least once | Deduplicated sends within FIFO's deduplication rules |
| Ordering | Best effort | Ordered within each message group |
| Parallelism | Very high | Parallel across message groups |
| Name | Any valid queue name | Must end in `.fifo` |

Choose Standard unless business correctness requires ordered processing within
a group. For a FIFO order queue, use `order_id` or `customer_id` as the message
group ID. One stuck message blocks later messages in the same group, so a single
global group sacrifices most parallelism.

## Delayed delivery

SQS can hide a newly sent message for **0 seconds to 15 minutes** before making
it available to consumers. This is useful for short delays such as “retry in 30
seconds,” “send this reminder in 10 minutes,” or “do not process this job until
its dependency has had time to settle.”

```{mermaid}
sequenceDiagram
  participant P as Producer
  participant Q as SQS
  participant W as Worker
  P->>Q: SendMessage with 5-minute delay
  Note over Q: Message is stored but hidden
  Note over Q: Five minutes pass
  Q-->>W: Message is now eligible for ReceiveMessage
  W->>W: Process job
```

There are two ways to request the initial delay:

- A **delay queue** sets `DelaySeconds` on the queue, so every newly sent message
  uses that delay. Standard and FIFO queues support this, up to 15 minutes.
- A **message timer** sets `DelaySeconds` on an individual `SendMessage` call and
  overrides the queue's default for that message. It is supported by Standard
  queues, but not by FIFO queues.

The delay is relative to when SQS receives the message; it is not an exact job
execution time. Once the delay expires, the message becomes *eligible* to be
received. Actual processing may start later depending on polling, worker
capacity, backlog, and failures. For an exact future schedule or anything more
than 15 minutes away, use a scheduler such as Amazon EventBridge Scheduler to
send the message at the desired time.

Do not confuse a delivery delay with the **visibility timeout**. The delivery
delay hides a message immediately after it is sent. The visibility timeout
hides it after a worker receives it, giving that worker time to finish before
SQS can redeliver it.

## Worker lifecycle

```{mermaid}
sequenceDiagram
  participant W as Worker
  participant Q as SQS
  W->>Q: ReceiveMessage
  Q-->>W: message + receipt handle
  Note over Q: Message is temporarily invisible
  W->>W: Process idempotently
  W->>Q: DeleteMessage(receipt handle)
  Note over Q: Message is complete
```

If the worker crashes before `DeleteMessage`, the visibility timeout expires
and SQS can deliver the message again. For long work, the worker calls
`ChangeMessageVisibility` as a heartbeat.

```python
messages = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20,
)

for message in messages.get("Messages", []):
    process_idempotently(message["Body"])
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=message["ReceiptHandle"],
    )
```

## Retries and DLQs

SQS does not require a worker to mutate `retryCount` in the payload. It tracks
how often a message has been received. A redrive policy moves a message to a DLQ
when its receive count exceeds `maxReceiveCount`.

Use separate queues or change message visibility to create retry delays. Keep
the DLQ's retention period long enough for operators to notice, diagnose, and
redrive failures.

## Numbers to know

- Default visibility timeout: **30 seconds**; configurable up to **12 hours**
  from receipt.
- Message retention: **1 minute to 14 days**; default **4 days**.
- Queue-level delivery delay: up to **15 minutes** for Standard and FIFO queues.
- Per-message timer: up to **15 minutes** for Standard queues; unsupported for
  FIFO queues.
- Message body: up to **1,024 KiB**.
- Long polling: up to **20 seconds** per receive request.

Treat these as service settings, not application defaults. A five-minute video
job needs a different visibility policy from a 50-millisecond email job.

## Pros

- No brokers, replication, patching, or capacity servers to operate.
- Integrates with IAM, Lambda, SNS, EventBridge, CloudWatch, and other AWS
  services.
- Standard queues absorb large bursts and scale worker fleets independently.
- FIFO message groups provide scoped ordering without serializing every task.

## Cons

- Pulling adds polling behavior and can add latency compared with a pushed
  delivery.
- Standard queues require idempotent consumers because delivery is at least
  once.
- FIFO ordering and deduplication add constraints and can reduce parallelism.
- AWS-specific APIs and IAM policies increase platform coupling.
- A queue is not a replayable event log or a long-running workflow engine.

## When to choose SQS

Choose SQS for AWS-hosted workloads that need a durable worker queue with little
operational overhead. Use Standard for ordinary asynchronous jobs and FIFO when
ordering within a well-distributed message group is a real invariant.

Consider RabbitMQ when exchange-based routing, push consumers, AMQP semantics,
or broker portability matter. Consider a stream when independent consumers need
retained history and replay.

## Sources and further reading

- [Amazon SQS delay queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)
- [Amazon SQS message timers](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-message-timers.html)
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Creating and configuring a Standard queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/creating-sqs-standard-queues.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
