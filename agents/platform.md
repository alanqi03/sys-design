---
title: Platform and Operations
description: Model APIs, state, observability, evals, deployment, and security for running agents in production.
---

# Platform and Operations

**In plain terms:** The orchestrator and tools are the agent itself. The platform is everything that makes it work for real users: the model it calls, where it saves its progress, how you watch it, how you measure it, how it scales, and how you keep it secure.

## Components

*Top-3 picks are a judgment based on common practice and public write-ups, not survey data.*

### LLM API

The agent's brain. Choosing a model is a trade-off between smart, fast, and cheap. Put the provider behind an interface so you can swap models, route by task, and fail over during outages.

**Top 3 options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Frontier model via API** | Best reasoning and tool use; no infra to run | Per-token cost; rate limits; data leaves your network (check vendor terms) | The main agent loop, where quality matters most (the default) |
| **Smaller/faster API model** | Cheap and low latency | Weaker on hard multi-step tasks | Subagents, summarization, compaction, classification |
| **Cloud-provider hosted (Bedrock, Vertex, etc.)** | Enterprise compliance; uses existing cloud contracts | New features sometimes arrive later | Enterprise customers already on that cloud |

**Biggest cost lever:** prompt caching with a stable prompt prefix, then routing easy work to smaller models.

### State Store

Two kinds of memory: **session state** (the record of one task, needed to pause, resume, recover from crashes, and debug) and **long-term memory** (facts learned across sessions). A common production combo uses all three options below together.

**Top 3 options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Postgres / relational DB** | Durable; queryable; transactional | Slower for very frequent writes | Source of truth for sessions, users, jobs, and status |
| **Redis** | Very fast; TTLs; pub/sub for live events | Weaker durability; memory cost | Hot session state and streaming live events to the UI |
| **Object storage (S3)** | Cheap for large data | Not queryable; higher latency | Full transcripts, logs, diffs, and sandbox snapshots |

### Observability

A flight recorder for the agent: every LLM call, tool call, token count, timing, and error, organized as one trace per session. Without it, debugging a 50-step non-deterministic run is guesswork.

**Top 3 options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **OpenTelemetry → existing APM (Datadog, Honeycomb, Grafana)** | Standard; fits existing infra and alerts | No LLM-specific views like transcripts or token costs | Teams with an existing observability stack |
| **LLM-specific tracing tools (Langfuse, LangSmith, Arize Phoenix)** | Transcript views; token and cost dashboards; eval integration | Another vendor or service to run | Agent-heavy teams |
| **Logs + transcripts in S3** | Cheap and complete | Hard to query | Baseline: always keep raw transcripts |

### Evals

Unit tests for agent behavior. Run a fixed set of real tasks, score them automatically, and gate changes to prompts, models, or tools on the results. Agents are non-deterministic, so run several trials per task.

**Top 3 options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Custom task suite from your own repos** | Reflects your real work; the most valuable eval | Effort to build and maintain | Always, eventually; gate releases on it |
| **Public benchmarks (SWE-bench, Terminal-Bench, etc.)** | Comparable across the industry | May not match your use case; contamination risk | Model selection and sanity checks |
| **LLM-as-judge** | Scales to subjective criteria | Judge bias; needs calibration against humans | Code style, explanation and PR description quality |

### Deployment and Scaling

Running thousands of agent sessions at once: a task queue, stateless workers that checkpoint every step, a warm pool of sandboxes, and per-tenant rate limits. The backbone choice is how tasks get queued and run reliably.

**Top 3 task execution backbones:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Managed queue + stateless workers (e.g. SQS)** | Simple; managed; scales horizontally | You build checkpointing, retries, and approval waits yourself | Most async agent services |
| **Redis queue / streams + workers** | Simple and fast; doubles as live event stream | Weaker durability guarantees | Lower-scale services or teams already running Redis |
| **Durable workflow engine (Temporal)** | Automatic retries, timers, and resume after crashes; long waits for human approval | Learning curve; another system to run | Long-running, mission-critical agents with human-in-the-loop steps |

### Security

The number one agent-specific risk is **prompt injection**: hidden instructions in web pages, issues, or tool output that the model might follow. Watch for the "lethal trifecta": private data + untrusted content + a way to send data out. Remove any one leg and exfiltration risk drops sharply.

**Top 3 mitigations:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Default-deny network in the sandbox** | Stolen data has nowhere to go | Must allowlist package registries and git hosts | Always; blocks the "send data out" leg |
| **No secrets in the sandbox; short-lived scoped tokens held by the harness** | Nothing valuable to steal; limits blast radius | More plumbing for auth | Always |
| **Human approval for external side effects** | A person sees risky actions (push, post, send) before they happen | Slows autonomy; approval fatigue | Anything that leaves the sandbox: pushes, comments, messages, deploys |

Wrapping tool output in "untrusted content" tags helps but isn't enough on its own. The real controls are the sandbox, network policy, and approvals.
