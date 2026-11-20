---
title: Agent Diagrams
description: Visual reference for agent architectures, execution models, context, tools, sandboxing, retrieval, security, and evals.
---

# Agent Diagrams

A visual companion to the layer overviews, using a coding agent as the running
example.

**How to read these:** rectangles are components, arrows are calls or data flow,
rounded boxes are AI agents, and diamonds are decisions.

---

## 1. The big picture

All the components and how they connect. The LLM only *proposes* actions; the orchestrator decides whether and how to carry them out.

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

**Notice:** three tiers. The orchestrator in the middle is the only thing that talks to everything; the LLM never touches tools directly.

---

## 2. Single agent

### 2a. The agent loop

One agent, one loop, deciding every next step itself (the ReAct pattern).

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TD
    Start(["Task arrives"]) --> Build["Context manager builds prompt"]
    Build --> Call["Call LLM"]
    Call --> Q{"Tool calls<br/>in response?"}
    Q -- "No" --> Done(["Return final answer"])
    Q -- "Yes" --> GR{"Guardrails:<br/>allow / deny / ask?"}
    GR -- "Deny" --> Err["Return error as tool result"]
    GR -- "Ask" --> Pause(["Checkpoint and wait<br/>for human approval"])
    GR -- "Allow" --> Exec["Execute tool in sandbox"]
    Exec --> Res["Append (truncated) result"]
    Err --> Res
    Res --> Lim{"Step / time / cost<br/>limit hit?"}
    Lim -- "Yes" --> Stop(["Stop: limit reached"])
    Lim -- "No" --> Build
```

**Notice:** errors go *back to the model* as results (so it can self-correct), and there are two exits besides "done": a human pause and a hard limit.

### 2b. One request, step by step

The same loop as a timeline for a task like "fix the failing auth test."

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant L as LLM
    participant S as Sandbox

    U->>O: "Fix the failing auth test"
    O->>L: system + tools + task
    L-->>O: tool_use: run_tests
    O->>S: pytest
    S-->>O: 1 failed: test_login
    O->>L: history + test output
    L-->>O: tool_use: read_file auth.py
    O->>S: read auth.py
    S-->>O: file contents
    O->>L: history + file
    L-->>O: tool_use: edit_file
    O->>S: apply edit
    O->>L: history + "Edited"
    L-->>O: tool_use: run_tests
    O->>S: pytest
    S-->>O: all passed
    O->>L: history + test output
    L-->>O: "Fixed: token expiry check" (no tool calls)
    O-->>U: summary + diff
```

**Notice:** every LLM call re-sends the growing history. That's why context management and prompt caching matter.

---

## 3. Sync tasks (streaming)

The user watches in real time. Good for short, interactive tasks.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    C["Client<br/>browser / CLI / IDE"] -- "POST /tasks" --> API["API server<br/>runs agent loop in-process"]
    API -- "SSE event stream" --> C
    C -- "POST /approve" --> API
    API <--> LLM["LLM API"]
    API <--> SB["Sandbox"]
    API --> DB[("Postgres<br/>checkpoints")]
```

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
sequenceDiagram
    participant C as Client
    participant A as API server
    participant L as LLM
    participant S as Sandbox

    C->>A: POST /tasks
    A-->>C: SSE: started
    loop each step
        A->>L: prompt
        L-->>A: tool call
        A-->>C: SSE: tool_start (run_tests)
        A->>S: run
        S-->>A: output
        A-->>C: SSE: tool_end
    end
    A-->>C: SSE: done + summary
```

**Notice:** simple, but the connection must stay open, and a server restart can cut off a running task. Checkpointing helps it resume.

---

## 4. Async tasks (queue + workers)

The user submits and walks away; they get notified (often with a PR) when it's done. Good for long tasks.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    C["Client<br/>gets job ID back immediately"] -- "1. POST /tasks" --> API["API server"]
    API -- "2. save session<br/>(status: queued)" --> DB[("Postgres<br/>sessions")]
    API -- "3. enqueue session_id<br/>(only after the save succeeds)" --> Q[["Queue<br/>SQS / Kafka / Redis"]]
    API -. "4. return job ID" .-> C
    Q -- "5. deliver" --> WP

    subgraph WP["Worker pool (stateless, identical, autoscaled)"]
        direction LR
        W1["Worker 1"]
        W2["Worker 2"]
        W3["Worker N"]
    end

    WP -- "6. load session,<br/>checkpoint every step" --> DB
    WP <--> LLM["LLM API"]
    WP <--> SB["Sandbox fleet<br/>one per session"]
    WP --> S3[("S3<br/>transcripts")]
    WP -- "events" --> PS["Redis pub/sub<br/>streams SSE updates to client"]
```

**Notice:** the session is saved to the database *before* it's enqueued, so a worker never receives an ID for a session that doesn't exist yet. Workers are **stateless and identical**. The session lives in the database, so any worker can pick up any session. Workers scale on queue depth; sandboxes scale on active sessions.

### Lifecycle of one async task

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
sequenceDiagram
    participant C as Client
    participant A as API
    participant Q as Queue
    participant W as Worker
    participant D as Postgres

    C->>A: POST /tasks
    A->>D: create session (queued)
    A->>Q: enqueue session_id
    A-->>C: 202 Accepted + job ID
    Q->>W: deliver session_id
    W->>D: load session
    loop agent loop
        W->>W: LLM call + tool call
        W->>D: checkpoint step
    end
    W->>D: status = done
    W->>Q: ack message
    W-->>C: notify (email / PR / webhook)
```

**Notice:** the worker acknowledges the message only at the end. If it crashes mid-task, the queue redelivers to another worker, which resumes from the last checkpoint.

### Inside one worker

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    subgraph Worker["One worker process (async event loop)"]
        direction LR
        A["Session A<br/>waiting on LLM"]
        B["Session B<br/>running tool"]
        Cc["Session C<br/>waiting on sandbox"]
        D["... up to ~20–50"]
    end
    Worker <--> LLM["LLM API (remote)"]
    Worker <--> SB["Sandboxes (separate machines)"]
```

**Notice:** agent work is mostly waiting, so one process juggles many sessions instead of one thread per session.

---

## 5. Agentic workflow

The most common production pattern: **you** fix the stages in code, and a focused agent works inside each stage.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    Issue["New issue"] --> Tri("Triage agent<br/>read-only")
    Tri --> A1{"Actionable?"}
    A1 -- "No" --> Close["Close with reason"]
    A1 -- "Yes" --> Plan("Plan agent<br/>read + search")
    Plan --> H{"Human<br/>approves plan?"}
    H -- "No" --> Stop["Stop"]
    H -- "Yes" --> Impl("Implement agent<br/>edit + run")
    Impl --> Test{"Test gate<br/>(plain code)"}
    Test -- "Fail, < 3 tries" --> Impl
    Test -- "Fail, 3 tries" --> Esc["Escalate to human"]
    Test -- "Pass" --> Rev("Review agent<br/>read-only")
    Rev --> PR["Open PR"]
```

**Notice:** the gates (diamonds) are code or humans, never an LLM's opinion. Each rounded box is its own small ReAct loop (diagram 2a) with only the tools it needs.

### Loops inside loops

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    subgraph Outer["Outer layer: workflow in code (fixed)"]
        direction LR
        N1["Step 1"] --> N2["Step 2"] --> N3["Step 3"]
    end
    subgraph Inner["Inner layer: inside each step"]
        direction LR
        T1["Think"] --> T2["Act"] --> T3["Observe"] --> T1
    end
    N2 -. "runs" .-> Inner
```

---

## 6. Plan-then-execute

The **LLM** writes the step list first (often approved by a human), then works through it.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    Task["Task"] --> P("Planner<br/>writes step list")
    P --> H{"Human<br/>approves?"}
    H -- "Edit" --> P
    H -- "Yes" --> E("Executor<br/>works step by step")
    E --> C{"Plan still<br/>valid?"}
    C -- "Yes, next step" --> E
    C -- "No" --> P
    C -- "All done" --> Done["Result"]
```

**Notice:** the difference from an agentic workflow is *who* wrote the steps. Here the model did, at runtime, so it can re-plan when reality differs.

---

## 7. Multi-agent patterns

### 5a. Orchestrator + subagents

A lead agent hands self-contained jobs to helpers. Each helper works in its **own fresh context** and returns only a short report.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    U["Task"] --> Lead("Lead agent")
    Lead -- "Find all callers of X" --> S1("Subagent 1<br/>read-only")
    Lead -- "Summarize auth module" --> S2("Subagent 2<br/>read-only")
    Lead -- "Check docs for API limits" --> S3("Subagent 3<br/>web search")
    S1 -- "short report" --> Lead
    S2 -- "short report" --> Lead
    S3 -- "short report" --> Lead
    Lead --> R["Final result"]
```

**Notice:** the noisy work (dozens of file reads) stays in the subagents' contexts. The lead's context only gets the summaries. Subagents can run in parallel.

### 5b. Parallel attempts (best-of-N)

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    T["Task"] --> A1("Attempt 1") & A2("Attempt 2") & A3("Attempt 3")
    A1 --> V{"Verify<br/>tests / judge"}
    A2 --> V
    A3 --> V
    V --> Best["Pick best result"]
```

**Notice:** N times the cost, so use it only when the task is valuable *and* you can verify results automatically.

### 5c. Pipeline of specialists and generator + critic

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    subgraph Pipeline["Pipeline of specialists"]
        direction LR
        P1("Planner") --> P2("Coder") --> P3("Reviewer")
    end
    subgraph Critic["Generator + critic"]
        direction LR
        G("Generator") -- "draft" --> K("Critic")
        K -- "feedback" --> G
        K -- "approved" --> Out["Output"]
    end
```

**Notice:** handoffs lose context, and a generator-critic loop needs a cap on rounds or it may never converge.

---

## 8. Human-in-the-loop approval

How a session pauses for approval without tying up a worker.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
stateDiagram-v2
    [*] --> Queued
    Queued --> Running: worker picks up
    Running --> AwaitingApproval: risky action proposed
    AwaitingApproval --> Queued: human approves (re-enqueue)
    AwaitingApproval --> Cancelled: human rejects
    Running --> Done: task complete
    Running --> Failed: error or limit hit
    Running --> Queued: worker crashed (redelivery)
    Done --> [*]
    Failed --> [*]
    Cancelled --> [*]
```

**Notice:** while awaiting approval, *no worker is busy*. The session just sits in the database until a human acts, then any worker resumes it.

---

## 9. Context and prompt caching

What goes into each LLM call, ordered stable to volatile so the cached prefix is as long as possible.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    subgraph Cached["Cached prefix: ~10% price, faster"]
        direction TB
        P1["1. System prompt — never changes"]
        P2["2. Tool definitions — never changes"]
        P3["3. Project notes — rarely changes"]
        P4["4. Conversation so far — only grows"]
        P1 --> P2 --> P3 --> P4
    end
    subgraph Fresh["Read fresh: full price"]
        P5["5. Latest tool result — new this turn"]
    end
    P4 --> P5
```

**Notice:** a change anywhere invalidates everything *after* it. A timestamp at the top means nothing ever gets cached.

### When context fills up

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    T["Tool returns output"] --> Tr["Truncate<br/>keep head + tail"]
    Tr --> Add["Append to history"]
    Add --> F{"Context<br/>> 80% full?"}
    F -- "No" --> Next["Next LLM call"]
    F -- "Yes" --> Cl["Clear stale tool results"]
    Cl --> F2{"Still > 80%?"}
    F2 -- "No" --> Next
    F2 -- "Yes" --> Comp["Compact: LLM summarizes history"]
    Comp --> Next
```

---

## 10. Tool layer and MCP

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    O["Orchestrator"] -- "tool call (JSON)" --> R["Tool registry<br/>validate schema"]
    R --> N["Native tools<br/>read · edit · run · grep"]
    R --> M["MCP client"]
    N --> SB["Sandbox"]
    M -- "stdio / HTTP" --> M1["GitHub MCP server"]
    M --> M2["Jira MCP server"]
    M --> M3["Database MCP server"]
    M1 --> GH["GitHub API"]
    M2 --> JI["Jira API"]
    M3 --> DBx[("Database")]
```

**Notice:** core tools are native (fast, fully controlled). Third-party integrations plug in through MCP without changing agent code.

---

## 11. Sandbox isolation

The key question: does the code share the host's kernel, or get its own?

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    subgraph Container["Container (Docker)"]
        direction TB
        CA["App A"] & CB["App B"] --> HK["Host kernel (SHARED)"]
        HK --> HW1["Hardware"]
    end
    subgraph GV["gVisor"]
        direction TB
        GA["App A"] --> GK["Fake kernel in user space<br/>(filters syscalls)"]
        GK --> HK2["Host kernel"]
        HK2 --> HW2["Hardware"]
    end
    subgraph VM["MicroVM / VM"]
        direction TB
        VA["App A"] --> KA["Kernel A"]
        VB["App B"] --> KB2["Kernel B"]
        KA & KB2 --> HV["Hypervisor<br/>(CPU-enforced)"]
        HV --> HW3["Hardware"]
    end
```

**Notice:** to escape a container, attack the shared kernel. To escape a microVM, you'd have to break the hypervisor, a much smaller and harder target.

### Layered in production

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    subgraph MV["Firecracker microVM (strong walls)"]
        subgraph CT["Docker container (easy packaging)"]
            Code["Agent-written code"]
        end
    end
    MV -. "default-deny network" .-x Internet["Internet"]
    MV -- "allowlisted" --> Reg["pypi / npm / git host"]
```

---

## 12. Knowledge retrieval

Retrieval is exposed as a **tool**, so the agent decides when it needs knowledge.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    Agent("Agent") --> Q{"What kind<br/>of question?"}
    Q -- "Code" --> AS["Agentic search<br/>grep · glob · read"]
    Q -- "Docs / runbooks" --> HY["search_docs tool"]
    HY --> BM["Keyword (BM25)"]
    HY --> VE["Vector search"]
    BM & VE --> RR["Merge + rerank"]
    RR --> Agent
    AS --> Agent
    PM["Project memory file"] -. "loaded every session" .-> Agent
```

---

## 13. Security: the lethal trifecta

An agent is high-risk for data theft when it has **all three**. Remove any one leg and the risk drops sharply.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart TB
    PD["Private data<br/>code, secrets"] --- R(("Data<br/>exfiltration<br/>risk"))
    UC["Untrusted content<br/>web, issues, tool output"] --- R
    EC["External channel<br/>network, comments, PRs"] --- R

    M1["No secrets in sandbox"] -. "cuts" .-> PD
    M2["Treat tool output as data"] -. "weakens" .-> UC
    M3["Default-deny network +<br/>approval for side effects"] -. "cuts" .-> EC
```

### Defense in depth

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    A["Proposed action"] --> L1["Prompt instructions<br/>(weakest)"]
    L1 --> L2["Code guardrails<br/>allow / deny / ask"]
    L2 --> L3["Sandbox + network policy<br/>(backstop)"]
    L3 --> E["Executed"]
```

---

## 14. Eval harness

How you know a change (prompt, model, tool) made the agent better.

```{mermaid}
%%{init: {"markdownAutoWrap": false, "flowchart": {"wrappingWidth": 600, "subGraphTitleMargin": {"top": 4, "bottom": 28}}}}%%
flowchart LR
    TS[("Task suite<br/>real tasks + hidden tests")] --> Run["Run each task<br/>N trials, fresh sandbox"]
    Run --> Agent("Agent")
    Agent --> Score{"Hidden tests<br/>pass?"}
    Score --> Rate["Pass rate + variance"]
    Rate --> Gate{"Better than<br/>current version?"}
    Gate -- "Yes" --> Ship["Ship change"]
    Gate -- "No" --> Read["Read failed transcripts"]
    Read --> Fix["Fix tools / prompts"]
    Fix --> Run
```

**Notice:** most improvements come from reading failed transcripts and fixing tool descriptions and error messages, not from prompt cleverness.
