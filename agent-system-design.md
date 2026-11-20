---
title: Agent System Design
description: Designing AI agent systems around an orchestrator, a tool layer, and a production platform, using a coding agent as the running example.
---

# Agent System Design

An agent system pairs an LLM with application code that builds its context,
executes its tool calls, enforces limits, and saves its progress. These notes
use a coding agent as the running example. Everything generalizes to other
agents (research, support, data analysis): swap the tools and the success
criteria.

## The big picture

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    U["User"] --> I["Interface<br/>CLI · IDE · web · Slack · webhook"]
    I <--> O

    subgraph O["Orchestrator (agent runtime)"]
        direction LR
        L["Agent loop"]
        C["Context manager"]
        G["Guardrails"]
    end

    O <--> LLM["LLM API"]
    O <--> T["Tool layer"]
    O <--> S[("State store")]

    T --> SB["Sandbox"]
    T --> KB[("Knowledge base")]
    T --> EX["External APIs (MCP)"]
```

The LLM never executes anything. It only *proposes* actions. The orchestrator
is the only component that talks to everything, and it is where safety, limits,
and correctness are enforced. Observability, evals, security, and cost controls
cut across every layer.

## Layers

| Layer | What it does | Components |
| --- | --- | --- |
| [Orchestrator](./agents/orchestrator-layer.md) | Turns model proposals into controlled actions | Agent loop, context manager, guardrails |
| [Tool layer](./agents/tool-layer.md) | Gives the agent hands: validates and runs tool calls | Sandbox, knowledge base, external APIs |
| [Platform](./agents/platform.md) | Makes the agent work for real users at scale | LLM API, state store, observability, evals, deployment, security |

## In this section

1. [Orchestrator Layer](./agents/orchestrator-layer.md)
2. [Tool Layer](./agents/tool-layer.md)
3. [Platform and Operations](./agents/platform.md)
4. [Agent Diagrams](./agents/diagrams.md): loops, workflows, multi-agent
   patterns, sync and async execution, caching, sandboxing, security, and evals
