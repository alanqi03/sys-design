---
title: Reasoning and Decision Making
description: Building a bounded agent decision loop with deterministic controls, approvals, and multi-agent coordination.
---

# Reasoning and Decision Making

Reasoning and decision making turns the current task, evidence, and tool results
into the next permitted action. The model proposes; the runtime validates,
executes, records, and decides whether the loop may continue.

## A bounded decision loop

```{mermaid}
stateDiagram-v2
  [*] --> Observe
  Observe --> Decide
  Decide --> CallTool: more evidence or action needed
  CallTool --> Observe: result or error
  Decide --> RequestApproval: risky action
  RequestApproval --> CallTool: approved
  RequestApproval --> Respond: denied
  Decide --> Respond: task complete or cannot proceed
  Respond --> [*]
```

Each iteration should produce a structured decision such as:

```json
{
  "decision": "call_tool",
  "tool": "lookup_order",
  "arguments": {"orderId": "order_123"},
  "reasonCode": "NEED_CURRENT_ORDER_STATUS",
  "expectedEvidence": "status and deliveredAt"
}
```

The reason code is an auditable summary, not hidden chain-of-thought. Validate
the decision against a finite schema and tool registry. If parsing fails, retry
once with correction or follow a deterministic fallback rather than executing
ambiguous output.

## Put deterministic controls around model choices

Application code should own hard constraints:

- authorization, data access, spend limits, and approval requirements;
- maximum iterations, model calls, tool calls, tokens, cost, and wall time;
- state-machine transitions and forbidden action sequences;
- argument validation and idempotency policy; and
- terminal states such as completed, declined, timed out, escalated, or failed.

Use the model for decisions where language and incomplete information matter.
Use code for stable rules that must hold every time. For example, a model can
identify that a refund is appropriate; code should enforce that refunds over
`$100` require human approval.

## Deciding when to stop

A useful loop ends when one of these conditions is true:

- the success criteria are met with sufficient evidence;
- required information is missing and the user must clarify;
- policy requires human approval or escalation;
- a dependency failed and no safe fallback remains; or
- an iteration, time, token, or cost budget is exhausted.

Repeatedly calling the same tool with equivalent arguments is a loop signal.
Track normalized calls and force the agent to change strategy, ask for help, or
stop after a small threshold.

## Handling uncertainty and conflict

Model confidence is not a security or correctness guarantee. Base high-impact
actions on verified data, constraints, and approval. When evidence conflicts,
apply a declared authority order, retrieve a fresher source, or escalate.

Distinguish:

- **known:** supported by current authoritative evidence;
- **inferred:** a conclusion from evidence, with assumptions stated;
- **unknown:** missing information that must be retrieved or requested; and
- **prohibited:** an action the runtime will not permit.

This distinction helps the final response avoid presenting a guess as a database
fact.

## Multi-agent decisions

A multi-agent coordinator decides which specialist owns each subtask, which can
run in parallel, how results are merged, and whether another delegation is
allowed.

```{mermaid}
flowchart TD
  supervisor[Supervisor] -->|Typed task| a[Agent A]
  supervisor -->|Typed task| b[Agent B]
  a -->|Result + evidence| join{Join policy}
  b -->|Result + evidence| join
  join -->|Agreement| supervisor
  join -->|Conflict| resolver[Rule, verifier,<br/>or human]
  resolver --> supervisor
```

Parallel agents help only when their work is independent. If Agent B needs Agent
A's output, represent that dependency rather than running both and hoping the
aggregator can repair the result.

Assign one owner to each side effect. Two agents should not independently refund
the same charge. Use a task ID and idempotency key across delegation, and prevent
cycles with maximum depth and hop counts.

## Failure handling

- Retry transient model or tool failures with a small, shared budget and jitter.
- Do not retry invalid or unauthorized actions without changing the input.
- Preserve completed step results so a restart does not repeat expensive or
  irreversible work.
- Re-plan after a material state change, but retain the original goal and scope.
- Fall back to a deterministic workflow or human operator when autonomy cannot
  complete safely.

Trace decisions, inputs, retrieved evidence IDs, tool calls, state transitions,
model and prompt versions, latency, tokens, and cost. Redact secrets and minimize
stored personal data.

## Further reading

- [Databricks: Single-agent and multi-agent design patterns](https://docs.databricks.com/aws/en/agents/agent-system-design-patterns)
