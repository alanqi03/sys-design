---
title: Tool Execution
description: Safely executing agent tools with typed contracts, least privilege, idempotency, timeouts, and observability.
---

# Tool Execution

Tools connect an agent to live information and external effects. A retrieval
tool reads; an action tool can send a message, change a record, spend money, or
run code. The tool boundary must be treated like an API boundary, not a trusted
extension of model output.

## Design narrow tool contracts

A tool should do one well-defined operation with a typed schema:

```json
{
  "name": "create_return",
  "description": "Create a return for an eligible order after approval",
  "input": {
    "orderId": "string",
    "itemIds": ["string"],
    "reasonCode": "enum",
    "idempotencyKey": "string"
  },
  "output": {
    "returnId": "string",
    "status": "created | already_exists",
    "labelUrl": "string"
  }
}
```

Prefer `create_return` over a general `execute_http_request` tool. Narrow tools
are easier to authorize, validate, test, observe, and explain to the model. Keep
descriptions precise enough to distinguish similar tools and return structured,
bounded results rather than pages of raw text.

## Execution pipeline

```{mermaid}
flowchart LR
  proposal[Model proposes call] --> schema[Validate schema]
  schema --> auth[Authorize actor and agent]
  auth --> approval{Approval required?}
  approval -->|Yes| human[Human approval]
  human --> execute[Execute with deadline]
  approval -->|No| execute
  execute --> normalize[Normalize and classify result]
  normalize --> audit[Audit and return observation]
```

For every call:

1. Resolve the tool from a server-controlled registry.
2. Validate types, ranges, resource identifiers, and payload size.
3. Authorize the original actor and agent for this specific operation.
4. Apply approval, rate-limit, budget, and policy checks.
5. Execute with a deadline, cancellation, and isolated credentials.
6. Normalize success and error output into a bounded schema.
7. Record an audit event and return the observation to the decision loop.

Validate again at execution time. A plan created earlier does not reserve
permission or prove that the resource is still in the expected state.

## Reads, writes, and approvals

Classify tools by risk:

| Class | Example | Typical control |
| --- | --- | --- |
| Read-only | Search policy, get order | Scoped identity, row/document authorization, result limits |
| Reversible write | Create draft ticket | Confirmation, audit log, undo path |
| External or costly write | Send email, provision resource | Explicit scope, quota, idempotency key |
| Irreversible or high impact | Transfer funds, delete production data | Strong authorization and human approval |
| Code execution | Analyze file with generated code | Network/filesystem sandbox, resource and time limits |

Approval should display the exact action, target, important parameters, and
consequences. Approval for “manage my account” is too broad to authorize an
unrelated later action.

## Retries and idempotency

A timeout is ambiguous: the remote service may have completed the action even
though the agent did not receive the response. Retry reads when safe. Retry
writes only when the tool supports a stable idempotency key or an equivalent
deduplication rule.

```text
task ID: task_123
tool step: create_return
idempotency key: task_123:create_return:order_456
```

The key must remain the same across network retries and process restarts. Record
the tool call as pending, succeeded, failed, or unknown so recovery can reconcile
an ambiguous outcome instead of blindly repeating it.

Use exponential backoff with jitter and a small retry budget shared across
layers. If the client, agent runtime, tool gateway, and downstream service all
retry independently, one failure can create a retry storm.

## Parallel execution

Independent read tools can run concurrently to reduce latency. Do not parallelize
steps with data dependencies or conflicting writes. Bound concurrency so one
request cannot exhaust connection pools, API quotas, or model context with many
results.

```{mermaid}
flowchart TD
  plan[Validated plan] --> orders[Read order]
  plan --> policy[Read policy]
  orders --> join{Join results}
  policy --> join
  join --> eligible{Eligible?}
  eligible -->|Yes, approved| write[Create return once]
```

## Multi-agent tool ownership

Give each specialist only the tools required by its role. Centralize policy in
a tool gateway or runtime so an agent cannot gain privilege by handing a task to
another agent. Propagate the original actor, tenant, task ID, trace ID, and
approval evidence through every delegation and tool call.

Avoid two agents owning the same side effect. The coordinator should assign one
step owner and use durable task state or a lease if execution can move between
workers.

## Treat results as untrusted input

Tool output can be malformed, oversized, stale, or malicious. A retrieved web
page or ticket can contain instructions aimed at the agent. Parse expected
fields, cap sizes, label provenance, escape content for its destination, and do
not allow tool text to replace system policy or grant access.

Monitor call count, latency, timeout rate, error category, retries, approvals,
denials, output size, cost, and duplicate suppression by tool and version. Use
circuit breakers and fallbacks when a dependency is failing.

## Further reading

- [Databricks: Agent tools and production guidance](https://docs.databricks.com/aws/en/agents/concepts/)
