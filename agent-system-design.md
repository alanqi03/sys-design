---
title: Agent System Design
description: Designing reliable single-agent and multi-agent systems from input and planning through retrieval, reasoning, tools, evaluation, and response.
---

# Agent System Design

An agent system combines a model with application code, context, tools, state,
policies, and an execution loop. The model can decide what information it needs,
which permitted action to take, and when it has enough evidence to respond.

The model is only one component. Reliability comes from the surrounding system:
typed interfaces, authorization, durable state, bounded loops, idempotent tools,
traces, evaluations, and explicit approval boundaries.

## The agent lifecycle

```{mermaid}
flowchart LR
  input[1. Input processing<br/>and planning] --> context[2. Context<br/>retrieval]
  context --> reason[3. Reasoning and<br/>decision making]
  reason --> tools[4. Tool<br/>execution]
  tools --> reason
  reason --> evaluate[5. Evaluation, feedback,<br/>and response]
  evaluate --> output[User or calling system]
```

This diagram is a loop, not always a pipeline. Tool results can trigger more
retrieval or a revised plan. The runtime must enforce stop conditions, budgets,
and permissions even when the model asks to continue.

## Start with the simplest sufficient design

| Design | Control flow | Use when | Main trade-off |
| --- | --- | --- | --- |
| Model and prompt | One model call | Generic generation or classification | Little access to live data or actions |
| Deterministic chain | Code chooses fixed steps | Workflow and tool sequence are known | Predictable but inflexible |
| Single-agent system | One agent chooses its next step | Varied tasks within one cohesive domain | Flexible, but loops and tool choices require guardrails |
| Multi-agent system | A coordinator or handoff connects specialists | Domains, contexts, policies, or tool sets are genuinely distinct | Modular but slower and harder to trace, evaluate, and operate |

Agentic behavior is a continuum. A production system can keep most steps
deterministic and let a model decide only which read-only retrieval tool to use.
More autonomy should be added only when evaluation shows that it improves the
task enough to justify extra latency, cost, and unpredictability.

## Single-agent system

A single-agent system has one agent loop and usually one main conversation
context. The agent receives a goal, selects from its tools, observes results,
and either continues or returns a response.

```{mermaid}
flowchart TD
  user[User request] --> agent[Single agent]
  agent --> kb[(Knowledge retrieval)]
  agent --> orders[Order API]
  agent --> returns[Return API]
  kb --> agent
  orders --> agent
  returns --> agent
  agent --> response[One cohesive response]
```

For “Return my last order,” the agent might look up the customer's orders,
retrieve the return policy, check eligibility, request approval for an
exception, create the return, and produce a shipping-label link.

### When to choose it

- Requests vary, but stay within one product or domain.
- One prompt and context can describe the job without becoming unwieldy.
- The tool set is small enough for the model to select reliably.
- One team owns the agent's behavior and operational boundary.

### Benefits and limits

A single agent is easier to trace, evaluate, secure, and optimize than a network
of agents. It can still call many tools and execute multi-step tasks. As the
prompt, context, and tool list grow, tool selection becomes harder and unrelated
instructions can conflict. Split the system only after this becomes a measured
problem.

## Multi-agent system

A multi-agent system has two or more agents with separate instructions,
contexts, tools, or ownership. They exchange structured messages or tasks. A
coordinator may be deterministic code, a model-based supervisor, or a hybrid.

```{mermaid}
flowchart TD
  request[Customer request] --> supervisor[Supervisor]
  supervisor --> shopping[Shopping agent<br/>catalog and reviews]
  supervisor --> support[Support agent<br/>orders and returns]
  supervisor --> billing[Billing agent<br/>payments and refunds]
  shopping --> supervisor
  support --> supervisor
  billing --> supervisor
  supervisor --> final[Final response]
```

### Common coordination patterns

- **Router:** classify once and send the task to one specialist.
- **Supervisor:** a coordinator delegates subtasks, observes results, and decides
  what happens next.
- **Handoff:** one agent transfers ownership and relevant context to another,
  which continues the interaction.
- **Parallel experts:** independent agents analyze different aspects; an
  aggregator compares or merges their outputs.
- **Generator and critic:** one agent drafts while another checks defined
  criteria. The critic needs evidence and a stop rule, not permission to debate
  forever.

### When to choose it

- Domains require materially different instructions, context, permissions, or
  tool sets.
- Separate teams need independently deployable specialists.
- Tasks decompose into parallel work with a clear merge rule.
- A narrow verifier can catch important, measurable failures in another agent.

### Added distributed-systems problems

Agent boundaries become service boundaries. Define task ownership, message
schemas, deadlines, cancellation, retries, deduplication, partial failure, and
trace propagation. Prevent delegation loops with a hop limit and record a task
graph so operators can see who did what.

Do not call a tool wrapper an agent merely to make the diagram modular. A tool
performs a bounded operation; an agent makes open-ended decisions over multiple
steps. Unnecessary agents add model calls and failure modes without adding a
useful decision boundary.

## Single versus multi-agent example

Suppose an employee asks, “Can I travel to Tokyo next month, book an approved
hotel, and explain the tax treatment?”

- A **single travel agent** can handle this if one team owns all policies and the
  tool set remains understandable.
- A **multi-agent design** may be justified if travel, booking, and tax have
  separate data access, approval policy, tool credentials, and owning teams. A
  supervisor produces one response from their structured results.

The multi-agent version is not automatically more capable. It succeeds only if
the decomposition is stable, specialists receive sufficient context, and the
supervisor can resolve conflicting or incomplete results.

## Cross-cutting production requirements

- **State:** persist task status, concise plans, tool calls, approvals, results,
  and the final outcome; do not rely on one model context as the system of record.
- **Security:** authenticate the caller, authorize each tool action, filter
  retrieved data by identity, sandbox code, and require approval for risky writes.
- **Reliability:** apply timeouts, bounded retries, circuit breakers, iteration
  limits, token/cost budgets, and idempotency keys.
- **Observability:** trace model, retrieval, tool, and agent-to-agent spans with
  versions, latency, cost, and error outcomes while protecting sensitive data.
- **Evaluation:** test final outcomes and intermediate trajectories before
  release, then sample production traces for regressions and novel failures.
- **Versioning:** pin or record model, prompt, tool schema, retrieval index, and
  policy versions so a behavior change can be reproduced and rolled back.

## In this section

1. [Input Processing and Planning](./agents/input-processing-planning.md)
2. [Context Retrieval](./agents/context-retrieval.md)
3. [Reasoning and Decision Making](./agents/reasoning-decision-making.md)
4. [Tool Execution](./agents/tool-execution.md)
5. [Evaluation, Feedback, and Response](./agents/evaluation-feedback-response.md)

## Further reading

- [Databricks: Agent system design patterns](https://docs.databricks.com/aws/en/agents/agent-system-design-patterns)
- [Databricks: AI and agent concepts](https://docs.databricks.com/aws/en/agents/concepts/)
