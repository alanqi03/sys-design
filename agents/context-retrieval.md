---
title: Context Retrieval
description: Choosing the right source for documents, business data, relationships, conversations, and agent state.
---

# Context Retrieval

A model does not automatically know your private or current data. Context
retrieval finds the small amount of information needed for the current task and
gives it to the model with its source.

## The quick rule

- Fetch **current facts** from the system that owns them.
- Use a **search or vector index** to find relevant text.
- Use **SQL or NoSQL** to load structured records.
- Use a **graph database** when multi-hop relationships are the question.
- Use a **cache** only to reuse data that has another source of truth.

A vector database is usually an index, not the authoritative copy of a policy,
order, or account balance.

## Which source should you use?

| Context | Good starting point | Example |
| --- | --- | --- |
| Policies, manuals, and tickets | Documents in a CMS or object store, indexed by keyword and vectors | Find the policy section for damaged items |
| Current business records | Owning API backed by SQL | Read an order's latest status and delivery date |
| Records fetched by one known key at high scale | NoSQL key-value or document database | Load a conversation by `conversation_id` |
| Connected, multi-hop relationships | Graph database, or SQL for fixed and modest joins | Find services affected by a dependency three hops away |
| Recent conversation | Model context plus durable SQL/NoSQL storage | Keep recent turns and a short summary of older turns |
| Agent task state | SQL for transactional workflows; NoSQL for simple keyed state | Store completed steps, approvals, and budget |
| Live external information | Authorized API or retrieval tool | Check current shipment status |

### SQL, NoSQL, vector, or graph?

- **SQL:** use it when transactions, constraints, joins, or flexible filters
  matter. Orders, payments, inventory, and permissions commonly fit here. Give
  the agent a narrow API such as `get_order(user_id, order_id)`, not unrestricted
  production SQL.
- **NoSQL:** use it for predictable key lookups and easy horizontal scaling.
  Sessions and conversation state might use a key such as
  `conversation:tenant_7:conv_456`. NoSQL does not replace a search index.
- **Vector and keyword search:** use vectors when meaning matters even if words
  differ. “How do I send broken headphones back?” can match “Returns for damaged
  electronics.” Use keywords for exact values such as an SKU or error code;
  hybrid search combines both and is a good starting point for documents.
- **Graph database:** use it when the path itself matters, such as nested access
  groups, fraud connections, or service dependencies. For “load an order and
  its customer,” normal SQL joins are usually simpler.

## Example: customer return agent

For “Can I return my last order?”, the agent needs three kinds of context:

```{mermaid}
flowchart LR
  request[Return request] --> agent[Agent]
  agent --> conversation[(SQL or NoSQL<br/>conversation state)]
  agent --> orderAPI[Order API]
  orderAPI --> orders[(SQL orders DB)]
  agent --> search[Hybrid search]
  search --> index[(Vector + keyword index)]
  index -. source link .-> docs[(Policy documents)]
  conversation --> pack[Small context packet]
  orderAPI --> pack
  search --> pack
  pack --> agent
```

- Conversation state identifies which order “last order” refers to.
- The order API verifies ownership and returns the current status and date.
- Hybrid search finds the relevant return-policy passage.

The agent does not need the entire conversation, order database, or policy
library.

## How retrieval should work

For documents, use a short pipeline:

```text
query → permission and metadata filters → keyword + vector search
      → rerank and deduplicate → top passages with sources
```

Split documents into meaningful chunks such as sections, not arbitrary isolated
sentences. Each result should carry its document ID, section, version, update
time, access scope, and text. This lets the response cite evidence and detect an
old policy.

**Prefetch** information every request needs, such as the authenticated user's
account tier. Give the agent a **retrieval tool** for request-dependent searches,
such as selecting the right device manual. Many systems use both.

## Keep context small and trustworthy

Pack context in this order:

1. governing policy and task constraints;
2. fresh facts needed for the decision;
3. the best supporting passages with source labels; and
4. a short conversation or task summary.

Then apply these rules:

- Enforce tenant and user permissions before returning results.
- Treat retrieved text as untrusted data, not as new system instructions.
- Re-read balances, permissions, inventory, and status before a consequential
  action.
- Do not save a model's guess as user memory; saved preferences need consent,
  provenance, expiry, correction, and deletion.
- In a multi-agent system, give each specialist only its subtask's context. A
  billing agent needs the invoice, not the full support transcript.

## Common failures

| Failure | Example | Fix |
| --- | --- | --- |
| Missed evidence | Search misses the damaged-item exception | Test recall, improve chunks, and use hybrid search |
| Stale fact | Cached order still says `processing` | Read consequential state from its owner |
| Unauthorized result | Another tenant's document appears | Filter by access before retrieval |
| Too much context | Twenty pages hide one useful paragraph | Rerank, deduplicate, and cap results |
| Missing source | A rule cannot be verified | Return source, section, version, and timestamp |

Evaluate retrieval separately from the final response. Check whether the needed
item appears in the top results, whether returned items are relevant and fresh,
whether citations are present, whether access controls leak data, and whether
the full task succeeds.

## Further reading

- [Databricks: AI and agent concepts](https://docs.databricks.com/aws/en/agents/concepts/)
- [Databricks: AI Search retrieval quality guide](https://docs.databricks.com/aws/en/ai-search/retrieval-quality)
- [Databricks: Connect agents to structured data](https://docs.databricks.com/aws/en/agents/custom-agents/structured-retrieval-tools)
