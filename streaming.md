---
title: Streaming
---

# Streaming

Streaming systems move an ongoing sequence of events from producers to
consumers. They decouple services in time, support multiple independent views of
the same events, and make high-throughput asynchronous processing possible.

## Events and logs

An event records something that happened: an order was placed, a file was
uploaded, or a sensor produced a reading. An append-only log stores events in a
stable order. Consumers track their position with an offset and can replay old
events to rebuild state or add a new derived view.

Events should have a stable identifier, timestamp, type, schema version, and
enough context for consumers to interpret them. Prefer facts over commands when
the stream is intended for multiple independent consumers.

## Partitions and ordering

A stream is often divided into partitions for parallelism. Events within one
partition have an order; events in different partitions generally do not. A
partition key places related events together—for example, all changes for one
account—so their local order is preserved.

The number of partitions limits consumer parallelism and affects rebalancing
cost. Skewed keys create hot partitions, so partitioning must reflect the actual
traffic distribution.

## Delivery semantics

**At-most-once** delivery may lose an event but does not redeliver it.
**At-least-once** delivery retries until acknowledged, so consumers may see
duplicates. **Exactly-once** processing requires coordination across reading,
state changes, and output; platforms that offer it do so within specific
boundaries.

At-least-once delivery with idempotent consumers is a robust default. Consumers
can deduplicate by event ID, use conditional updates, or make the intended state
converge regardless of repetitions.

## Event time and windows

Processing time is when a consumer handles an event; event time is when the
event actually occurred. Network and service delays cause events to arrive late
or out of order. Windowed computations therefore need a policy for lateness,
often expressed with a watermark that estimates how far event time has advanced.

Tumbling windows are fixed and non-overlapping. Sliding windows overlap. Session
windows group activity separated by gaps. The business question determines the
window, not the implementation convenience.

## Backpressure and recovery

When producers outpace consumers, lag grows. Durable retention absorbs temporary
bursts, but it does not solve a permanent capacity mismatch. Monitor consumer
lag in both records and time, then scale consumers, reduce per-event work, or
partition further.

Poison events that fail repeatedly should not block a partition forever. After a
bounded retry policy, preserve the event and failure context in a dead-letter
stream for inspection and controlled replay.

## Stream versus queue

A work queue is ideal when one of several workers should perform a task. A
durable stream is ideal when multiple consumer groups need independent access to
an ordered history. Real systems often use both: a stream distributes facts,
while queues dispatch retryable units of work.

## Redis messaging

[Redis](./caching/redis.md) provides two distinct messaging models. Pub/Sub is a
live, at-most-once broadcast: a disconnected subscriber misses messages sent
while it is away. Redis Streams retains an append-only sequence and supports
consumer groups, acknowledgements, pending entries, and replay, making
at-least-once processing possible.

Use Pub/Sub for ephemeral notifications such as presence or cache invalidation
when loss is acceptable or repaired elsewhere. Consider Redis Streams for a
compact operational workflow, and compare it with a dedicated streaming
platform when the design needs long retention, many independent consumer
groups, large-scale replay, or a broader stream-processing ecosystem.

## Design checklist

- What is the event schema, identity, and retention period?
- What ordering is actually required?
- How is the partition key chosen?
- Can consumers process duplicates safely?
- How are late, malformed, and poison events handled?
- How will lag, replay, and schema evolution be operated?
