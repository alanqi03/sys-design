---
title: Input Processing and Planning
description: Turning user input into an authorized, bounded agent task and an executable plan.
---

# Input Processing and Planning

The first stage converts an untrusted request into a well-defined task. It
establishes identity, scope, constraints, and success criteria before a model or
tool acts.

## Input processing

```{mermaid}
flowchart LR
  raw[Text, files, audio,<br/>conversation] --> normalize[Parse and normalize]
  normalize --> identity[Authenticate and<br/>load authorization]
  identity --> safety[Policy and risk checks]
  safety --> task[Structured task envelope]
  task --> planner[Planner or workflow]
```

A useful task envelope separates data from instructions:

```json
{
  "taskId": "task_123",
  "actor": {"userId": "user_42", "tenantId": "tenant_7"},
  "goal": "Return my most recent eligible order",
  "constraints": {"maxToolCalls": 8, "deadlineMs": 15000},
  "approvalPolicy": {"refundAboveCents": 10000},
  "attachments": [],
  "conversationSummary": "Customer confirmed the delivery address"
}
```

At the boundary:

- parse formats and reject unsupported or oversized content;
- authenticate the caller and preserve tenant and user identity;
- classify the request's domain, risk, and required capabilities;
- resolve ambiguous references or ask a focused clarification;
- separate system policy, user instructions, and untrusted document content; and
- assign a deadline, token budget, tool-call limit, and cancellation signal.

Conversation history is input, not automatically trusted truth. Summaries can
omit or distort facts, and retrieved files can contain prompt injection. Policy
and authorization must live outside text that the user or a document can edit.

## Planning choices

Use the least dynamic planner that fits the work:

- **No explicit plan:** answer or classify in one model call.
- **Deterministic workflow:** application code fixes the steps and uses models
  only inside bounded stages.
- **Model-generated plan:** the model selects and orders steps from an allowed
  vocabulary.
- **Adaptive plan:** the agent updates its next steps as tool results arrive.
- **Hierarchical plan:** a supervisor creates subtasks for specialist agents.

For a return request, a concise executable plan could be:

```json
{
  "steps": [
    {"id": "s1", "action": "lookup_recent_orders"},
    {"id": "s2", "action": "retrieve_return_policy"},
    {"id": "s3", "action": "check_eligibility", "dependsOn": ["s1", "s2"]},
    {"id": "s4", "action": "create_return", "dependsOn": ["s3"], "approval": "conditional"}
  ],
  "doneWhen": "A return is created or an ineligibility reason is explained"
}
```

The plan records intended actions, not private chain-of-thought. Store concise
decisions, tool arguments, evidence, and outcomes that operators can audit.

## Validate the plan

Treat a generated plan as proposed data:

- every action must map to an allowed tool, workflow step, or agent;
- dependencies must form an acyclic graph with a bounded size;
- write actions must satisfy authorization and approval policy;
- the total estimated latency and cost must fit the request budget; and
- the plan must define completion, fallback, and escalation conditions.

Do not let the planner invent tool names or delegate to an arbitrary agent. The
runtime owns the registry and validates every transition again at execution
time because permissions or state may have changed after planning.

## Planning in multi-agent systems

A router should return a typed destination and confidence-independent fallback,
not free-form prose. A supervisor should delegate the minimum context needed and
give every subtask its own ID, deadline, output schema, and allowed capabilities.

```{mermaid}
flowchart TD
  task[Parent task] --> supervisor[Supervisor plan]
  supervisor --> a[Policy subtask]
  supervisor --> b[Order subtask]
  a --> join{Validate and join}
  b --> join
  join -->|Complete| next[Next action]
  join -->|Missing or conflicting| fallback[Clarify, retry, or escalate]
```

Set a maximum delegation depth and hop count. Without them, agents can bounce a
task between specialists or repeatedly re-plan without making progress.

## Failure modes

- **Premature action:** the agent writes before resolving the target or intent.
- **Over-planning:** a simple answer becomes many model calls and tools.
- **Plan drift:** later actions no longer serve the original goal.
- **Injection:** user or retrieved content attempts to replace higher-priority
  policy or exfiltrate data.
- **Unbounded decomposition:** the supervisor creates too many subtasks.
- **Stale authorization:** a plan is approved once and executed after access
  changes.

Measure routing accuracy, clarification rate, plan validity, average planned and
executed steps, abandonment, latency, and cost. Re-evaluate plans after tool
results without silently widening the user's requested scope.

## Further reading

- [Databricks: Agent system design patterns](https://docs.databricks.com/aws/en/agents/agent-system-design-patterns)
