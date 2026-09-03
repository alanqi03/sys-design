---
title: Evaluation, Feedback, and Response
description: Evaluating agent outcomes and trajectories, monitoring production quality, learning from feedback, and constructing grounded responses.
---

# Evaluation, Feedback, and Response

The final stage decides whether the task is complete, constructs a response, and
turns traces and feedback into evidence for improving the system. Evaluation is
not only a pre-release test: agent behavior, models, data, tools, and users all
change over time.

## Evaluate the whole system

```{mermaid}
flowchart LR
  dataset[Representative tasks] --> agent[Agent version]
  agent --> traces[Outputs and traces]
  traces --> rules[Deterministic checks]
  traces --> judges[Model-based scorers]
  traces --> humans[Domain experts]
  rules --> report[Metrics and failures]
  judges --> report
  humans --> report
  report --> improve[Prompt, tools, retrieval,<br/>policy, or workflow changes]
  improve --> dataset
```

Evaluate at several levels:

- **Task outcome:** Was the requested job completed correctly?
- **Response quality:** Is the answer relevant, clear, grounded, safe, and
  appropriately uncertain?
- **Trajectory:** Did the agent choose valid tools, avoid unnecessary loops, and
  follow approvals and policy?
- **Retrieval:** Did it find authorized, current, relevant evidence?
- **Tool effects:** Were arguments correct, side effects idempotent, and partial
  failures handled?
- **Operations:** What were latency, availability, token use, tool calls, and cost?

A polished final answer can hide a dangerous trajectory, such as calling a write
tool twice. Outcome and process both matter.

## Build an evaluation dataset

Start with real, de-identified tasks and expert-written expectations. Include:

- common happy paths and important business workflows;
- ambiguous requests that should trigger clarification;
- permission failures and actions requiring approval;
- stale, missing, conflicting, and adversarial context;
- tool timeouts, malformed results, and partial outages;
- long conversations and repeated requests; and
- cases that previously failed in production.

Record expected outcomes and invariants rather than requiring one exact sentence.
For example: “No return is created, the expired policy is not cited, and the
answer explains escalation” is more useful than a single reference response.

## Choose the right scorer

- Use **deterministic code** for schema validity, required citations, exact
  calculations, tool-call count, authorization, and side-effect invariants.
- Use **model-based judges** for criteria such as relevance or clarity when they
  are calibrated against human labels.
- Use **domain experts** for nuanced correctness, policy, safety, and high-impact
  edge cases.
- Use **user feedback** as a signal, not unquestioned ground truth; satisfaction
  can differ from factual or policy correctness.

Track results by task category and risk. An average score can hide a severe
regression in refunds, permissions, or one customer language. Compare candidate
and baseline versions on the same dataset before release.

## Multi-agent evaluation

Evaluate each specialist contract, the coordinator's routing and merge behavior,
and the end-to-end outcome. Useful checks include routing accuracy, handoff
count, delegation depth, conflicting results, context leakage, per-agent tool
authorization, and whether one agent's failure is handled without corrupting the
whole task.

Preserve a task graph with parent and child trace IDs. Without it, an incorrect
final answer is difficult to attribute to retrieval, a specialist, the
supervisor, or aggregation.

## Construct the response

Before returning output:

1. verify that success criteria or a declared fallback condition is met;
2. reconcile tool results with the latest authoritative state;
3. separate verified facts, inferences, and unresolved uncertainty;
4. cite sources or identify completed actions and their IDs;
5. remove secrets, internal instructions, unsafe content, and raw exceptions;
6. format for the caller's requested channel and level of detail; and
7. persist the outcome and any continuation state.

```json
{
  "status": "completed",
  "message": "Return ret_789 was created for order_123.",
  "artifacts": [{"type": "shipping_label", "url": "https://example/label"}],
  "evidence": [{"source": "return-policy", "version": "2026-08-15"}],
  "requestId": "req_456"
}
```

Stream progress only when it improves the experience, and distinguish tentative
progress from committed actions. Never tell the user an action succeeded before
the authoritative tool confirms it.

## Feedback and improvement loop

```{mermaid}
flowchart LR
  production[Sampled production traces] --> feedback[User and expert feedback]
  feedback --> triage[Classify root cause]
  triage --> regression[Add regression case]
  regression --> change[Versioned change]
  change --> offline[Offline evaluation]
  offline --> canary[Canary release]
  canary --> production
```

Classify failures before changing prompts: input/routing, retrieval, reasoning,
tool contract, tool reliability, policy, response, or evaluation error. Fix the
owning component and add the case to the regression set. Version prompts, model,
tools, retrieval index, policies, and evaluators so comparisons are reproducible.

Do not automatically turn arbitrary user feedback into long-term memory or
training data. Apply consent, privacy, retention, quality review, and poisoning
controls.

## Production monitoring

Monitor both conventional service health and agent quality:

- request and task success, error rate, latency, saturation, and availability;
- tokens, model calls, tool calls, cost, retries, and loop-limit exits;
- retrieval quality, citation coverage, tool correctness, and policy violations;
- approval and escalation rates, user corrections, and abandonment; and
- score distributions and regressions by task, tenant, language, model, prompt,
  and tool version.

Sample traces for automated scoring and expert review, with higher coverage for
risky actions and newly released behavior. Alerts should lead to a safe action:
rollback a version, disable a tool, reduce autonomy, route to a deterministic
fallback, or require human review.

## Further reading

- [Databricks: Evaluate and monitor agents](https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/)
- [Databricks: Agent system production guidance](https://docs.databricks.com/aws/en/agents/agent-system-design-patterns)
