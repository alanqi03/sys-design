---
title: Queues
---

# Queues

A queue holds tasks until workers are ready to perform them. It moves slow,
failure-prone, or bursty work out of the request path and lets a worker pool
process that work at a controlled rate.

```{mermaid}
flowchart LR
  producer[Producer] --> queue[[Queue]]
  queue --> worker1[Worker 1]
  queue --> worker2[Worker 2]
  queue --> worker3[Worker 3]
  worker1 --> result[(Result)]
  worker2 --> result
  worker3 --> result
```

One task is normally handled by one worker. Delivery may happen more than once,
so the queue supplies acknowledgement and retry mechanics while the worker must
make repeated processing safe.

## In this section

- [Queue Fundamentals](./queues/queue-fundamentals.md) covers worker lifecycle,
  acknowledgements, visibility timeouts, retries, dead-letter queues,
  idempotency, ordering, and scaling.
- [Amazon SQS](./queues/sqs.md) explains a managed pull-based queue with
  Standard and FIFO modes.
- [RabbitMQ](./queues/rabbitmq.md) explains exchanges, bindings, queues,
  acknowledgements, routing, and quorum queues.

## Queue versus stream

- A **queue message** is usually an instruction with one owner:
  `GenerateInvoicePDF`.
- A **stream event** is usually a reusable fact for independent consumers:
  `OrderPlaced`.

See [Streaming](./streaming.md) when ordered history, replay, multiple consumer
groups, or stateful processing is the main requirement.
