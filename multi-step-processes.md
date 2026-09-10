---
title: Multi-step Processes
description: An ELI18 guide to reliable multi-step workflows, sagas, Temporal, and AWS Step Functions.
---

# Multi-step Processes

A **multi-step process** is one user goal that requires several pieces of work.
Checking out might create an order, reserve inventory, charge a card, book a
shipment, and send a confirmation. Customer onboarding, insurance claims,
video processing, and data pipelines have the same shape.

The happy path is easy to draw:

```text
Do A -> Do B -> Do C -> Done
```

The real design problem is remembering progress and deciding what to do when a
process, service, or machine fails halfway through.

## Why a normal request chain is fragile

Imagine an API calling each service in order:

```{mermaid}
flowchart LR
  client[Client] --> orders[Create order]
  orders --> inventory[Reserve inventory]
  inventory --> payment[Charge payment]
  payment --> shipping[Book shipment]
```

Several awkward cases appear:

- Payment succeeds, but the process crashes before recording the result.
- Shipping is unavailable after inventory and payment have succeeded.
- A timeout leaves the caller unsure whether an operation happened.
- A retry performs the same charge twice.
- A process waits hours for a human approval while servers restart or deploy.

Do not hold one database transaction or HTTP request open across a long-running
distributed process. Persist the workflow's state between steps and make every
external action safe to retry.

## Start with the smallest reliable option

| Situation | Good starting point |
| --- | --- |
| Every change is in one database | One short ACID transaction |
| A few asynchronous steps with simple branching | A queue plus a workflow-status table |
| Many steps, retries, timers, branches, or human approvals | A durable workflow engine |
| Several services commit independently and earlier work may need undoing | A saga, often run by a workflow engine |

A workflow engine is not automatically necessary. It earns its place when the
recovery logic would otherwise become a home-grown collection of status rows,
queues, timers, retry loops, and repair scripts.

## Sagas: undo the business effect

In software, a **saga** is a sequence of local steps. Each step that may need to
be undone has a matching **compensating action**. Run the steps in order; if a
later step fails permanently, walk backward and compensate for the work that
already succeeded.

| Forward step | Compensating action |
| --- | --- |
| Create pending order | Mark order canceled |
| Reserve inventory | Release inventory |
| Charge payment | Issue refund |
| Book shipment | Cancel shipment if it has not left |

```{mermaid}
flowchart LR
  a[Create pending order] --> b[Reserve inventory]
  b --> c[Charge payment]
  c --> d[Book shipment]
  d -->|Success| done[Confirm order]
  d -->|Permanent failure| u3[Refund payment]
  u3 --> u2[Release inventory]
  u2 --> u1[Cancel order]
```

### Compensation is not time travel

A database rollback can erase uncommitted changes. A saga cannot erase facts
that other systems have already observed. A refund is a new transaction, an
email cannot be unsent, and a parcel that has shipped may require a return.
Compensation restores an acceptable business state rather than recreating the
exact past.

Compensations can also fail, so they need retries, idempotency, monitoring, and
sometimes a manual-repair path. Record which forward and compensating steps
succeeded.

### Who decides the next step?

- **Orchestration:** one coordinator tells each participant what to do next.
  The full path is easier to see and change, but the coordinator becomes an
  important dependency. Temporal and AWS Step Functions commonly fill this role.
- **Choreography:** services react to one another's events without a central
  coordinator. This can reduce central coupling, but a long process becomes
  harder to understand, debug, and change.

For a workflow with several branches or compensations, orchestration is usually
the clearer starting point. Use a [transactional outbox](./database/change-data-capture-outbox.md)
when a service must reliably commit local data and publish the event that moves
the process forward.

## Durable execution, ELI18

A **durable execution engine** is like a coordinator that writes down every
completed step before moving on. If the coordinator or worker crashes, another
one reads the durable history and continues from the recorded point instead of
starting the entire process from memory.

```{mermaid}
flowchart LR
  app[Application] --> engine[(Workflow engine<br/>state and history)]
  engine --> tasks[[Task queue]]
  tasks --> workers[Workers]
  workers --> services[Databases and services]
  workers -->|Result| engine
```

The engine normally owns workflow state, timers, retries, and branching.
Workers perform the real side effects. Since a worker can finish its action and
crash before reporting success, those actions still need idempotency keys or
deduplication; durable execution does not make arbitrary side effects
exactly-once.

## Temporal, ELI18

[Temporal](https://docs.temporal.io/) lets developers express a workflow in
application code while the Temporal service durably stores its event history.
If a worker disappears, Temporal can reconstruct the workflow's state from that
history and schedule the unfinished work again.

The two main concepts are:

- A **Workflow** contains the sequence, branches, timers, and decisions. Its
  code must be deterministic so replaying the same history reaches the same
  decision.
- An **Activity** performs an external action such as calling a payment API or
  writing to a database. Activities can be retried and should be idempotent.

Temporal is a good fit when workflows live naturally in application code, may
run for minutes to months, or need rich control flow and durable timers. The
trade-off is operating or buying another platform, learning its programming
model, and safely versioning workflow code that may have old executions still
running.

## AWS Step Functions, ELI18

[AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
is a managed AWS workflow service. You define a **state machine**: a diagram-like
set of task, choice, wait, parallel, and map states. AWS stores each execution's
state and moves it to the next state.

A task can invoke Lambda, run an ECS or Batch job, call AWS service APIs, make
an HTTPS request, or wait for an external callback. `Retry` handles temporary
failures; `Catch` routes a permanent failure to a fallback or compensation
path.

Step Functions is a good fit when the system already runs on AWS, visual state
machines help operators understand the flow, or the workflow coordinates many
AWS services. The trade-offs are AWS-specific definitions and permissions,
service quotas and pricing, and state-machine data transformations that can
become cumbersome for code-heavy business logic.

## Temporal versus Step Functions

| Question | Temporal | AWS Step Functions |
| --- | --- | --- |
| How is the flow written? | Workflow code using a supported SDK | Amazon States Language or Workflow Studio |
| Where does work run? | Your workers execute Activities | AWS integrations, Lambda, containers, HTTP APIs, or external workers |
| Who operates the engine? | Self-host Temporal or use Temporal Cloud | AWS operates the service |
| Natural fit | Code-heavy, long-running application workflows | AWS service orchestration and visibly modeled state machines |
| Main learning cost | Deterministic workflow code, replay, workers, versioning | State definitions, data mapping, IAM, quotas, and workflow types |

Both can orchestrate a saga. A saga is the **business recovery pattern**;
Temporal and Step Functions are **tools that can remember and execute the
pattern reliably**.

## Design checklist

For every multi-step process, write down:

- The durable workflow ID and current business status.
- The timeout, retry policy, and idempotency key for each step.
- Which errors are temporary, which are permanent, and which need a person.
- The compensation for every completed step that may need to be reversed.
- What the client sees while work is pending, succeeds, fails, or is canceled.
- How operators inspect history, retry safely, and repair a stuck execution.
- How changes remain compatible with workflows that started on older code.

## Further reading

- [Saga pattern](https://microservices.io/patterns/data/saga.html)
- [Temporal documentation](https://docs.temporal.io/)
- [AWS Step Functions state machines](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-statemachines.html)
- [AWS Step Functions error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
- [AWS Step Functions service integrations](https://docs.aws.amazon.com/step-functions/latest/dg/integrate-services.html)
