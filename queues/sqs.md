---
title: Amazon SQS
description: A concise system-design guide to Amazon Simple Queue Service.
---

# Amazon SQS

Amazon Simple Queue Service (SQS) is a fully managed queue. You don't run any
servers: producers send messages, workers poll for them, and a worker deletes a
message when it's done.

## Example: a worker

```python
while True:
    resp = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=10,
        WaitTimeSeconds=20,          # long polling: wait up to 20 s for messages
    )
    for msg in resp.get("Messages", []):
        make_thumbnails(msg["Body"])          # must be safe to run twice
        sqs.delete_message(QueueUrl=queue_url,
                           ReceiptHandle=msg["ReceiptHandle"])
```

```{mermaid}
sequenceDiagram
  participant W as Worker
  participant Q as SQS
  W->>Q: ReceiveMessage
  Q-->>W: message + receipt handle
  Note over Q: message hidden (visibility timeout)
  W->>W: make thumbnails
  W->>Q: DeleteMessage
```

If the worker crashes before `DeleteMessage`, the **visibility timeout** runs
out and the message reappears for another worker. For long jobs, the worker can
extend the timeout with `ChangeMessageVisibility`.

## Standard vs. FIFO

| | Standard | FIFO |
| --- | --- | --- |
| Delivery | At least once (duplicates possible) | Duplicate sends removed within a 5-minute window |
| Order | Roughly in order | Strict order within a message group |
| Throughput | Nearly unlimited | Lower, parallel across groups |

Use Standard unless you truly need order. With FIFO, set the **message group
ID** to something like `order_id`, so each order's messages stay in order while
different orders run in parallel. Putting everything in one group means only one
message is processed at a time.

## Delayed messages

SQS can hide a new message for **up to 15 minutes**, e.g. "send this reminder
in 10 minutes":

```python
sqs.send_message(QueueUrl=queue_url, MessageBody=body, DelaySeconds=600)
```

The delay is a minimum, not an exact time. For anything further out or exactly
scheduled, use Amazon EventBridge Scheduler.

The **delay** hides a message right after it's sent. The **visibility timeout**
hides it after a worker receives it.

## Retries and DLQs

SQS counts how many times each message has been received. With a redrive policy
like `maxReceiveCount: 5`, a message that's been received 5 times without being
deleted moves to a dead-letter queue. Your code doesn't need to count retries
itself.

## Numbers to know

- Visibility timeout: default **30 s**, max **12 h**.
- Retention: default **4 days**, max **14 days**.
- Delay: up to **15 minutes**.
- Message size: up to **1 MiB**.
- Long polling: up to **20 s** per request.

## Pros and cons

**Pros**

- Nothing to operate; scales automatically.
- Works directly with Lambda, SNS, EventBridge, and IAM.
- Handles big traffic bursts.

**Cons**

- Standard queues deliver duplicates, so workers must be idempotent.
- FIFO queues have lower throughput.
- AWS only.
- It's a queue, not a replayable log: deleted messages are gone.

## When to choose SQS

Choose it for background jobs on AWS when you want zero operations work. Choose
RabbitMQ if you need flexible routing or must run outside AWS. Choose Kafka if
several consumers need to replay the same history.

## Further reading

- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Amazon SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon SQS delay queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)
