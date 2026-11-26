---
title: Vector Clocks and Version Vectors
description: Detecting whether one update came after another or whether two updates conflict, with a worked shopping cart example.
---

# Vector Clocks and Version Vectors

When different replicas or devices can change the same data at the same time,
you need to tell two situations apart:

- **B came after A:** B already includes A's change, so keep B and discard A.
- **A and B are concurrent:** neither knew about the other, so they
  **conflict**. Keep both and merge them.

Vector clocks and version vectors answer that question with a few counters.

**Running example:** a shared shopping cart that a phone and a laptop can both
edit, even while offline.

## Why timestamps aren't enough

The simple approach is **last write wins**: tag each update with the time, and
keep the newest.

1. The phone (offline) adds **bread** at 10:00:05.
2. The laptop adds **eggs** at 10:00:07.
3. They sync. Eggs is newer, so the cart keeps the laptop's version, and
   **bread silently disappears**.

There are two problems:

- Neither write knew about the other, so neither is really "later." It's a
  conflict, and both should survive.
- Clocks on different machines drift apart by milliseconds or more, so "newer"
  may not even be true.

## How a version vector works

Keep **one counter per writer**. Each version of the cart carries the counters.
The rules are:

1. When a writer changes the data, it **increments its own counter**.
2. When two versions are merged, take the **largest value of each counter**,
   then the merging writer increments its own.

To compare version X with version Y:

- If every counter in X is **≤** the matching counter in Y, then Y came after
  X. Keep Y.
- If X is bigger on some counters and Y is bigger on others, they're
  **concurrent**. That's a conflict.

## Worked example

`P` is the phone's counter, `L` is the laptop's.

| Step | What happens | Cart | Version |
| --- | --- | --- | --- |
| 1 | Phone adds milk | milk | `{P:1, L:0}` |
| 2 | Laptop syncs, then adds eggs | milk, eggs | `{P:1, L:1}` |
| 3 | Phone, offline, adds bread to its own copy | milk, bread | `{P:2, L:0}` |
| 4 | Phone reconnects, and the two versions are compared | | |

**Step 2 vs. step 1:** `{P:1, L:0}` ≤ `{P:1, L:1}` on every counter, so step 2
is simply newer. No conflict.

**Step 3 vs. step 2:** `{P:2, L:0}` vs. `{P:1, L:1}`. The phone is ahead on
`P` (2 > 1), but the laptop is ahead on `L` (1 > 0). Neither saw the other's
change, so they're **concurrent**.

```{mermaid}
flowchart LR
  v1["milk<br/>P:1 L:0"] --> v2["milk, eggs<br/>P:1 L:1"]
  v1 --> v3["milk, bread<br/>P:2 L:0"]
  v2 --> m["milk, eggs, bread<br/>P:3 L:1"]
  v3 --> m
```

The system keeps both versions (often called **siblings**) and merges them. For
a cart, the merge is the union: milk, eggs, bread. The merged version takes the
largest of each counter (`{P:2, L:1}`), and the phone, which did the merge,
increments its own: `{P:3, L:1}`. That version is newer than both parents.

## Resolving conflicts

Detecting a conflict is the easy part. Then you have to decide what to do:

- **Merge automatically** using rules that fit the data. A cart takes the union,
  and a counter adds both increments. Amazon's Dynamo merged carts this way. The
  downside is that a deleted item can come back.
- **Ask the user**, like a "which version do you want to keep?" prompt.
- **Use CRDTs** (Conflict-free Replicated Data Types), which are data
  structures designed so any two versions can always merge automatically.
  Collaborative editors use them.

## Vector clocks vs. version vectors

The mechanics are the same: one counter per participant, compared counter by
counter. They differ in what they track.

| | Vector clock | Version vector |
| --- | --- | --- |
| Tracks | Every event and message in a system | Versions of one piece of data |
| Attached to | Events, such as each message sent | Each stored value, such as the cart |
| Counter per | Process | Replica or writer that updated this value |
| Answers | "Did event A cause event B, or were they concurrent?" | "Is this copy newer, older, or in conflict with that copy?" |
| Used in | Debugging and tracing distributed programs, research | Databases and sync, such as Dynamo, Riak, and file sync |

Databases use **version vectors**, even though the Dynamo paper called them
"vector clocks," which is why the two names are often mixed up.

A simpler relative is the **Lamport clock**, a single counter. It gives every
event an order, but it can't tell "came after" from "concurrent."

## Practical notes

- **Size grows with the number of writers.** Use one counter per *server*, not
  per user or device, and trim old entries.
- **Many databases skip them.** Cassandra uses last-write-wins timestamps.
  DynamoDB uses last write wins plus conditional writes ("update only if
  version = 7"). Riak is the best-known database that exposes version vectors
  and siblings.
- **Single-leader databases rarely need them,** because the leader already puts
  writes in one order. They matter for **leaderless**, **multi-leader**, and
  **offline-first** systems.

## Further reading

- [Amazon Dynamo paper (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), section 4.4
- [Riak: Causal context and siblings](https://docs.riak.com/riak/kv/latest/learn/concepts/causal-context/index.html)
- *Designing Data-Intensive Applications*, chapter 5, "Detecting Concurrent Writes"
