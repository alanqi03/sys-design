---
title: ZooKeeper
description: A plain-language guide to ZooKeeper, what problems it solves, how to use it, and when to pick it or etcd.
---

# ZooKeeper

**ZooKeeper** is a small shared notice board that all your servers trust. They
use it to answer questions like:

- **Who's the leader?** ("Server A runs the nightly job.")
- **Who's alive?** ("Payments servers A and C are up. B just died.")
- **Is this feature flag on?** ("The new checkout is turned on.")

It runs on 3 or 5 machines and uses [consensus](consensus.md), so every server
sees the same answer, and it keeps working if one machine dies.

**Running example:** a payments service with 3 servers (A, B, C). Exactly one of
them must run the nightly settlement job, never zero and never two.

## Why not just use a database row?

You could write `leader = A` in a database table. The hard part is what happens
when **A crashes**: nothing removes the row, so everyone keeps waiting for a
dead leader.

ZooKeeper solves this by noticing when a server disappears and cleaning up
after it automatically.

## The four ideas you need

**1. Small files in folders.** Data is stored as tiny entries (called
**znodes**) in a folder tree, like a file system:

```text
/services/payments/A   → "10.0.0.5:8080"
/services/payments/C   → "10.0.0.7:8080"
/config/new_checkout   → "on"
```

**2. Entries that disappear when you do.** A server can create an
**ephemeral** entry. While the server is connected, it sends a small "I'm still
here" heartbeat. If the server crashes and the heartbeats stop, ZooKeeper
deletes its entry after a few seconds.

**3. Numbered entries.** ZooKeeper can add an increasing number to a name
(`job_0001`, `job_0002`, ...), like taking a ticket at a deli counter. The
lowest number goes first.

**4. Notifications.** A server can **watch** an entry or folder and get told
when it changes, instead of checking over and over.

Everything else is built by combining these four.

## How to use it

### Leader election: who runs the job?

Each server takes a numbered ticket that disappears if the server dies. The
**lowest ticket is the leader.**

```{mermaid}
flowchart LR
  A["A: ticket 1<br/>leader"] --> B["B: ticket 2<br/>waiting"]
  B --> C["C: ticket 3<br/>waiting"]
```

If A crashes, ticket 1 disappears, B now holds the lowest ticket, and B takes
over. No human needs to step in.

```python
from kazoo.client import KazooClient   # Python ZooKeeper client

zk = KazooClient(hosts="zk1:2181,zk2:2181,zk3:2181")
zk.start()

# Waits until this server is the leader, then runs the job
zk.Election("/election/settlement", "server-A").run(run_settlement_job)
```

### Distributed locks: only one server at a time

A normal lock in your code, like Python's `threading.Lock`, only stops two
threads in the **same program**. It can't stop a different machine. A
**distributed lock** lives in ZooKeeper, where every server can see it, so it
works across all of them.

**Example: don't charge an order twice.**

1. Server A picks up "charge order 123" from a queue and starts charging.
2. A is slow, so the queue assumes it failed and hands the same job to server B.
3. Without a lock, A and B both charge the card, and the customer pays twice.

With a lock on `/locks/order-123`, whichever server gets it first does the
charge. The other waits, then sees the order is already paid and skips it.

```python
with zk.Lock("/locks/order-123", "server-A"):   # waits here if B holds it
    if not already_charged("order-123"):
        charge_order("order-123")
```

It works like leader election: the lowest ticket holds the lock. If A crashes
while holding it, A's ticket disappears and B can continue. With a plain
`locked = true` row in a database, the lock would stay stuck forever.

Other common uses:

- Only one server runs a database migration when many start at once.
- Only one worker processes a given uploaded file.
- Only one instance rebuilds a shared cache at a time.

[Redis locks](../caching/redis.md#distributed-locks) do the same job. They're
simpler and faster, while ZooKeeper's are stricter about correctness.

### Service discovery: who's alive?

Each payments server registers itself with an entry that disappears if it dies:

```python
zk.create("/services/payments/A", b"10.0.0.5:8080", ephemeral=True, makepath=True)
```

Other services list `/services/payments` to find live servers, and watch the
folder to hear about changes. When B crashes, its entry vanishes and everyone
stops sending it traffic.

### Feature flags

Put a flag in `/config/new_checkout`. Every service reads it and watches it.
Flip it to `off`, and all services hear about it within moments, with no
redeploy. The same works for any setting every server must share, like the
current database address.

## When to use it

**Good fit:** small, critical facts that many servers must agree on:

- Picking one leader or one worker for a job
- Locks that must be released even if the holder crashes
- Tracking which servers are alive
- Feature flags shared across services

**Not a good fit:**

- **App data** like users, orders, or messages. ZooKeeper is built for a few
  megabytes, not gigabytes, and handles only thousands of writes per second.
  Use a real database.
- **Simple apps with one server.** There's nothing to coordinate.
- **Kubernetes apps.** Kubernetes already has etcd and a built-in leader
  election API, so use that instead of adding ZooKeeper.

One gotcha: if the leader freezes (for example, a long pause in a Java program),
ZooKeeper may decide it's dead and pick a new leader while the old one is still
running. For work where two leaders would cause damage, have the database
reject writes from the old leader. See
[distributed locks](../caching/redis.md#distributed-locks) for the same issue
with Redis.

## ZooKeeper vs. etcd

**etcd** does the same job and is the more common choice for new systems.

| | ZooKeeper | etcd |
| --- | --- | --- |
| Best known from | Hadoop, HBase, older Kafka | Kubernetes |
| Data | Folders of entries | Keys and values |
| "Disappear when I die" | Ephemeral entries | Keys with a time-limited lease |
| Language world | Java | Go, cloud native |

Kafka used ZooKeeper for years, then replaced it with built-in Raft (KRaft) to
have one less system to run. Kafka 4.0 removed ZooKeeper entirely.

**In an interview,** you rarely need the details. "Use ZooKeeper or etcd for
leader election and service discovery" is usually enough.

## Further reading

- [ZooKeeper overview](https://zookeeper.apache.org/doc/current/zookeeperOver.html)
- [ZooKeeper recipes](https://zookeeper.apache.org/doc/current/recipes.html)
- [Kazoo (Python client) docs](https://kazoo.readthedocs.io/)
