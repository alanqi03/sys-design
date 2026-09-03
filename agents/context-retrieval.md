---
title: Context Retrieval
description: Retrieving relevant, authorized, fresh, and attributable context for an agent.
---

# Context Retrieval

Models have finite context and do not automatically know private or current
facts. Context retrieval selects evidence for the present task from conversation
state, documents, databases, APIs, memories, and prior tool results.

## Sources of context

- **Conversation state:** recent messages, a verified summary, prior approvals,
  and unresolved questions.
- **Unstructured knowledge:** policies, manuals, tickets, wikis, and contracts
  retrieved by keyword or semantic similarity.
- **Structured data:** accounts, orders, inventory, permissions, and metrics
  queried through an API or read-only SQL tool.
- **Agent state:** current plan, completed steps, tool results, and remaining
  budgets.
- **Long-term memory:** deliberately saved user preferences or prior outcomes,
  with consent, provenance, expiry, and deletion behavior.

Do not use model context as the authoritative copy of business state. Fetch a
balance, order status, or permission from its source of truth when correctness
depends on its current value.

## Retrieval pipeline

```{mermaid}
flowchart LR
  query[Task and query] --> rewrite[Rewrite or decompose]
  rewrite --> filters[Identity and metadata filters]
  filters --> retrieve[Keyword + vector + structured retrieval]
  retrieve --> rerank[Rerank and deduplicate]
  rerank --> pack[Pack within context budget]
  pack --> model[Model with evidence and provenance]
```

For documents, split content into chunks that preserve meaningful units such as
a policy section, index those chunks, and retain source ID, version, timestamps,
and access-control metadata. Hybrid retrieval combines exact keyword matching
with semantic search; reranking spends additional compute on a smaller candidate
set.

Apply access filters during retrieval, not only after text returns. Retrieving a
forbidden document and asking the model not to reveal it is not an authorization
boundary.

## Retrieve just in time

Two common designs are:

- **Prefetch:** application code always retrieves known context before the model
  call. This is predictable and works well for a fixed RAG chain.
- **Retrieval tool:** the agent decides when and how to search. This supports
  adaptive queries but needs call limits and protection against repeated searches.

A hybrid often works best: inject small, always-required policy and identity
context, then let the agent request scoped domain data as needed.

For “Can I return order 123?”, retrieve the authenticated user's order through
a structured tool and the current return policy through document search. Do not
place the entire order history and policy library in every prompt.

## Context packing

Context competes with instructions, conversation, tool schemas, and output for
the model's token window. Allocate explicit budgets and prefer:

1. governing policy and task constraints;
2. current authoritative facts needed for the decision;
3. the most relevant evidence with source labels; and
4. a concise conversation or execution summary.

Remove duplicates and low-value boilerplate. Preserve enough surrounding text
to interpret a passage, and never truncate away a condition, exception, unit, or
date that changes its meaning.

## Freshness, provenance, and citations

Every context item should carry a source, version or update time, retrieval time,
and access scope. The response should cite evidence when users need to verify a
claim. If sources conflict, surface the conflict or use an explicit authority
rule rather than letting retrieval rank decide policy.

Cache stable retrieval results when safe, but include tenant, permissions,
query, index version, and other response-changing fields in the key. Short TTLs
do not fix caching data that a user was never authorized to see.

## Multi-agent context

Do not copy the full parent conversation into every specialist. Give each agent:

- a typed subtask and success criteria;
- the minimum relevant evidence and identity scope;
- its own allowed retrieval sources and tools; and
- a schema for returning conclusions with citations and uncertainty.

The coordinator should merge structured results, not concatenate several large
agent transcripts. Shared mutable memory creates races and confusing ownership;
prefer versioned task state and append-only observations.

## Failure modes and evaluation

- **Missed evidence:** the needed item is not in the candidate set.
- **Wrong ranking:** relevant evidence exists but is displaced by plausible noise.
- **Stale context:** an old policy or record drives a current action.
- **Context poisoning:** untrusted content instructs the agent to ignore policy
  or expose secrets.
- **Overstuffing:** excessive context increases latency and distracts the model.
- **Missing provenance:** a correct-looking answer cannot be verified.

Evaluate retrieval separately from generation with recall at `k`, precision at
`k`, ranking quality, freshness, authorization leakage tests, citation coverage,
and downstream task success. A generation score alone cannot reveal whether the
right evidence ever reached the model.

## Further reading

- [Databricks: AI and agent concepts](https://docs.databricks.com/aws/en/agents/concepts/)
