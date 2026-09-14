---
title: Patterns
description: Reusable system-design patterns for large blobs, multi-step workflows, and real-time updates.
---

# Patterns

A component is a building block; a **pattern** explains how several building
blocks cooperate to solve a recurring problem. Start with the requirement and
choose a pattern only when its added moving parts earn their place.

| Pattern | Use it when | Central idea |
| --- | --- | --- |
| [S3: Handling Large Blobs](./s3-large-blobs.md) | Clients upload or download images, video, documents, backups, or other large files | Let the service authorize access, then transfer bytes directly between the client and object storage |
| [Multi-step Processes](./multi-step-processes.md) | One business operation spans several steps, services, retries, timers, or compensating actions | Persist progress so the workflow survives crashes and can safely retry or unwind work |
| [Real Time Updates](./connection-protocols.md) | Users or services need fresh events without repeatedly fetching the full state | Separate event distribution from the client connection, then choose polling, webhooks, SSE, WebSockets, or WebRTC |

## How to choose

- If the payload is a large file, keep its bytes out of the application server
  and database request path.
- If work must continue after a request ends or survive partial failure, model
  it as a durable multi-step process.
- If a consumer must learn about changing data quickly, define who receives the
  update and how the final network hop delivers it.

These patterns can appear together. A video upload may use a presigned S3 URL,
an S3 event may start a durable processing workflow, and the finished workflow
may push a real-time update to the client.
