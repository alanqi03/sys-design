---
title: Multi-step Processes
description: A plain-language guide to reliable multi-step workflows, sagas, Temporal, and AWS Step Functions.
---

# Multi-step Processes

A **multi-step process** is one user goal that takes several steps across
services.

**Running example:** checkout.

```text
Create order → Reserve inventory → Charge card → Book shipment → Done
```

The happy path is easy. The hard part is **what happens when something fails
halfway**.

## Why a simple chain of calls breaks

If one API call runs each step in turn:

- The card is charged, then the server crashes before saving that fact. Did the
  customer pay?
- Shipping is down after the card was already charged.
- A timeout leaves you unsure whether the charge happened, and a retry charges
  twice.
- A step waits hours for a human approval while servers restart.

The fix: **save progress after each step**, and make every step safe to retry.

## Pick the simplest option that works

| Situation | Start with |
| --- | --- |
| Everything is in one database | One transaction |
| A few background steps | A queue plus a `status` column |
| Many steps, retries, timers, or human approvals | A workflow engine (Temporal, Step Functions) |
| Earlier steps may need undoing | A saga, usually run by a workflow engine |

## Sagas: undoing completed steps

A **saga** pairs each step with a **compensating action** that undoes it. If a
later step fails for good, run the compensations backward.

| Step | Compensation |
| --- | --- |
| Create pending order | Cancel order |
| Reserve inventory | Release inventory |
| Charge card | Refund |
| Book shipment | Cancel shipment |

```{mermaid}
flowchart LR
  a[Create order] --> b[Reserve inventory]
  b --> c[Charge card]
  c --> d[Book shipment]
  d -->|success| done[Confirm order]
  d -->|"fails for good"| r[Refund]
  r --> rel[Release inventory]
  rel --> cancel[Cancel order]
```

Compensation isn't a rollback. A refund is a new transaction, and a sent email
can't be unsent. It returns things to an acceptable state, not the exact past.
Compensations can fail too, so they need retries as well.

**Who runs the steps?**

- **Orchestration:** one coordinator tells each service what to do next. Easy
  to follow and change. Usually the better choice.
- **Choreography:** each service reacts to the previous service's event, with no
  coordinator. Fewer central pieces, but the overall flow is hard to see.

To reliably save data *and* publish the event that triggers the next step, use
a [transactional outbox](./database/change-data-capture-outbox.md).

## Durable execution

A **workflow engine** writes down each completed step. If a worker crashes,
another one reads that history and continues where it left off, instead of
starting over.

```{mermaid}
flowchart LR
  app[App] --> engine[("Workflow engine<br/>saves every step")]
  engine --> workers[Workers]
  workers --> services[Payment, inventory, ...]
  workers -->|result| engine
```

A worker can still finish a step and crash before reporting it, so the step runs
again. Steps still need idempotency keys; for example, pass the order ID to the
payment provider so a retry doesn't charge twice.

## Temporal

You write the workflow as ordinary code:

```python
T = timedelta(seconds=30)   # max time per attempt

@workflow.defn
class Checkout:
    @workflow.run
    async def run(self, order_id: str):
        await workflow.execute_activity(reserve_inventory, order_id, start_to_close_timeout=T)
        try:
            await workflow.execute_activity(charge_card, order_id, start_to_close_timeout=T)
        except ActivityError:
            await workflow.execute_activity(release_inventory, order_id, start_to_close_timeout=T)
            raise
        await workflow.execute_activity(book_shipment, order_id, start_to_close_timeout=T)
```

- A **Workflow** is the sequence of steps and decisions. It must be
  deterministic, because Temporal rebuilds it by replaying its history.
- An **Activity** does real work, such as calling the payment API. Activities are
  retried automatically and should be idempotent.

**Good for:** complex logic in code; workflows that run for minutes to months.
**Cost:** another system to run (or pay for), and care when changing code while
old workflows are still running.

## AWS Step Functions

You define a **state machine** in JSON or a visual editor: steps, choices,
waits, and parallel branches. AWS runs it and tracks each execution.

```json
"ChargeCard": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:...:charge-card",
  "Retry": [{ "ErrorEquals": ["States.Timeout"], "MaxAttempts": 3 }],
  "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory" }],
  "Next": "BookShipment"
}
```

`Retry` handles temporary failures; `Catch` routes permanent ones to a
compensation step.

**Good for:** systems already on AWS that tie together Lambda, ECS, and other
AWS services, especially when a visual diagram helps. **Cost:** AWS-only, and
complex logic gets awkward in JSON.

## Temporal vs. Step Functions

| | Temporal | Step Functions |
| --- | --- | --- |
| Written as | Code (Go, Java, Python, TypeScript...) | JSON or a visual editor |
| Runs on | Your workers | AWS (Lambda, ECS, service calls) |
| Operated by | You, or Temporal Cloud | AWS |
| Best for | Complex, long-running app logic | Wiring AWS services together |

A saga is the **pattern**; Temporal and Step Functions are **tools** that run it
reliably.

## Checklist

- What's the workflow ID and current status?
- What's the timeout, retry policy, and idempotency key for each step?
- Which errors are temporary, which are permanent, and which need a human?
- What's the compensation for each step?
- What does the user see while it's pending, done, or failed?
- How do operators find and fix a stuck workflow?

## Further reading

- [Saga pattern](https://microservices.io/patterns/data/saga.html)
- [Temporal documentation](https://docs.temporal.io/)
- [AWS Step Functions state machines](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-statemachines.html)
- [AWS Step Functions error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
