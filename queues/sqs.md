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
- Queue-level delivery delay: up to **15 minutes**.
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

- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Creating and configuring a Standard queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/creating-sqs-standard-queues.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
