---
title: Orchestrator Layer
description: "The agent runtime: the agent loop, context manager, and guardrails that turn model proposals into controlled actions."
---

# Orchestrator Layer

**In plain terms:** The orchestrator is the agent's "manager." The LLM is the brain that suggests what to do, but it can't actually do anything itself. The orchestrator is your code that takes the LLM's suggestion, checks it's allowed, carries it out, and reports back what happened.

The orchestrator sits at the center of the
[big-picture architecture](../agent-system-design.md#the-big-picture): it is the
only component that talks to the interface, the LLM API, the tool layer, and the
state store.

**Key insight to say in interviews:** The LLM never executes anything. It only *proposes* actions. The orchestrator is where you enforce safety, limits, and correctness. This separation is what makes agents controllable.

## Components

*Top-3 picks are a judgment based on common practice and public write-ups, not survey data.*

### Agent Loop

The heartbeat of the agent: ask the LLM what to do, check it's allowed, do it, report back, repeat until done or a limit is hit. The big design choice is **who decides the order of steps**: you, in code, or the LLM at runtime.

**Top 3 loop architectures (by prevalence in production):**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Agentic workflow (state machine / graph)** | Predictable; code-enforced gates (tests, approvals); small focused agents; easy to debug and evaluate per step | Less flexible; tasks that don't fit the flowchart get stuck; more upfront design | Production pipelines: support, document processing, async coding agents (issue → plan → code → test → PR) |
| **Plain workflow (chain)** | Simplest and cheapest; fully deterministic; easy to test | Can't adapt to surprises; no real autonomy | Steps are known in advance: RAG Q&A, summarization, extraction, classification |
| **Single ReAct loop** | Very flexible; handles novel tasks; ~50 lines to build | Less predictable; cost and length vary; can drift on long tasks | Interactive assistants where a human is watching, e.g. coding assistants, chat with tools |

### Context Manager

The LLM re-reads everything on every turn, and the context window has a limit. The context manager decides what goes into each prompt, keeping it small, relevant, and ordered stable-first so prompt caching cuts cost and latency.

**Top 3 techniques to save context:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Truncate tool output** | Simple; huge savings; stops one result from flooding the context | May cut a useful middle section | Always |
| **Just-in-time retrieval** | Always fresh; only loads what's needed | More steps and tool calls | Codebases and anything that changes often (the default) |
| **Compaction (summarization)** | Enables very long tasks | Lossy; extra LLM call; breaks the prompt cache once | When a session nears the context limit |

**Also key:** prompt caching. Order the prompt system → tools → project notes → history, and keep timestamps and other changing values out of the prefix.

### Guardrails

Rules the agent can't break, enforced by code rather than by asking the model nicely. Before any tool runs, guardrails decide: allow, deny, or ask a human. Think defense in depth: prompt instructions are the weakest layer, code rules are stronger, and the sandbox is the backstop.

**Top 3 permission models:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Risk tiers** | Good balance of safety and autonomy; reads and sandboxed writes run freely, risky actions get reviewed | You have to classify every tool | Most production agents (the default) |
| **Rule-based allow/deny lists** | Predictable and auditable | Misses novel dangerous commands | The foundation under any other model |
| **Fully autonomous in sandbox, human reviews the PR** | High throughput; review happens at a natural checkpoint | Mistakes only caught at review time | Async coding agents, where the PR is the approval gate |
