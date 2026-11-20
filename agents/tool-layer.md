---
title: Tool Layer
description: "How agents act: tool design, sandboxes, knowledge retrieval, and external API integrations."
---

# Tool Layer

**In plain terms:** Tools are the agent's hands. Each tool has a name, a description (so the model knows when to use it), and an input schema (so the model knows what arguments to pass). The tool layer receives the model's tool call, validates it, runs it, and returns the result.

## Typical coding-agent tool set

| Tool | Purpose |
|---|---|
| `read_file(path, start_line?, end_line?)` | Read code |
| `list_dir(path)` / `glob(pattern)` | Explore structure |
| `grep(pattern, path?)` | Find symbols, usages |
| `edit_file(path, old_str, new_str)` | Precise edits (string replace) |
| `write_file(path, content)` | Create new files |
| `run_command(cmd, timeout?)` | Tests, builds, linters |
| `git_*` / `create_pr` | Version control, delivery |
| `web_search` / `fetch_url` | Look up docs |
| `todo_write` | Planning and progress tracking |

## Tool design principles

1. **Descriptions are prompts.** Write them like docs for a new engineer: when to use, when not to, examples.
2. **Fewer, well-designed tools beat many overlapping ones.** Overlap confuses the model.
3. **Return high-signal output.** Trim noise; include what the model needs for the next step.
4. **Actionable errors.** "old_str not found in file; it appears 0 times. Re-read the file first." beats "ValueError."
5. **Idempotency where possible.** Retries shouldn't double-apply side effects.
6. **Edit by string replacement, not full rewrites.** Cheaper, fewer accidental deletions, and easier to diff.

## Components

*Top-3 picks are a judgment based on common practice and public write-ups, not survey data.*

### Sandbox

A locked room where agent-written code runs, so mistakes or prompt-injected commands can't damage the host, read secrets, or reach the internet. A **sandbox** is the goal; containers, microVMs, and VMs are ways to build it. The key difference is whether the code shares the host's kernel or gets its own.

**Top 3 options (by prevalence):**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Docker container** | Starts in ~1s; dense packing; familiar tooling; easy resource caps | Shared kernel, so one kernel exploit can escape to the host and other tenants | Internal or single-tenant tools, trusted users, MVPs |
| **Firecracker microVM (self-hosted or via providers like E2B)** | Own kernel per sandbox with hardware-enforced isolation; ~125ms boot; thousands per host; snapshots | More infra work (needs KVM hosts); nested Docker is awkward | Multi-tenant SaaS running untrusted or prompt-injected code at scale |
| **gVisor (e.g. Modal)** | Much stronger than plain Docker; works with existing container images; no KVM needed | No hardware boundary; slower for syscall-heavy work like big builds | Multi-tenant with moderate risk, when you want container ergonomics |

**Full VMs** are the specialist choice for agents that need a full desktop or nested Docker.

### Knowledge Base

Where the agent looks up information that isn't in the code or its training: docs, runbooks, past incidents, team conventions. Expose retrieval as a tool so the agent decides when it needs it.

**Top 3 retrieval options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Agentic search (grep, glob, read)** | Always fresh; exact matches on identifiers; no index to maintain | More steps; weak on fuzzy, meaning-based queries | Code (the default) |
| **Vector search (embeddings)** | Finds related text by meaning ("auth" ↔ "login") | Goes stale as content changes; chunking errors; misses exact identifiers | Docs, tickets, wikis, Q&A over prose |
| **Project memory file** | Zero infra; highest signal per token | Manual upkeep; must stay short | Build commands, conventions, gotchas loaded every session |

### External APIs

The outside services the agent works with: GitHub, Jira, CI, Slack, databases. The agent only asks for a tool call; the tool layer makes the real request, holds the credentials, and routes risky side effects to a human.

**Top 3 integration options:**

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **Native function calling** | Fastest and simplest; full control | Every integration is custom code | Core tools like file edits and shell commands |
| **MCP servers** | Standard protocol; reuse existing servers (GitHub, Jira, databases); plug in without code changes | Extra hop; quality varies by server; more attack surface | Pluggable third-party integrations and enterprise extensibility |
| **Code execution as the tool** | Chains many operations in one step; fewer round trips; less context | Needs a strong sandbox; harder to audit each action | Data-heavy tasks with many chained calls |

---

## Code examples

*Illustrative and simplified; library APIs change, so check current docs.*

### Tool registry with schema validation

```python
import json, subprocess
from pathlib import Path
from jsonschema import validate, ValidationError

WORKSPACE = Path("/workspace").resolve()

class ToolRegistry:
    def __init__(self):
        self.tools = {}

    def register(self, name: str, description: str, schema: dict):
        def deco(fn):
            self.tools[name] = {"fn": fn, "description": description, "schema": schema}
            return fn
        return deco

    def definitions(self) -> list[dict]:
        return [{"name": n, "description": t["description"], "input_schema": t["schema"]}
                for n, t in self.tools.items()]

    def execute(self, name: str, args: dict) -> str:
        if name not in self.tools:
            return f"Error: unknown tool '{name}'. Available: {list(self.tools)}"
        try:
            validate(args, self.tools[name]["schema"])
        except ValidationError as e:
            return f"Error: invalid arguments: {e.message}"
        return self.tools[name]["fn"](**args)

registry = ToolRegistry()

def safe_path(p: str) -> Path:
    full = (WORKSPACE / p).resolve()
    if not full.is_relative_to(WORKSPACE):
        raise PermissionError("Path escapes the workspace")
    return full

@registry.register(
    "edit_file",
    "Replace an exact, unique string in a file. Read the file first. "
    "old_str must match exactly once, including whitespace.",
    {"type": "object",
     "properties": {"path": {"type": "string"},
                    "old_str": {"type": "string"},
                    "new_str": {"type": "string"}},
     "required": ["path", "old_str", "new_str"]})
def edit_file(path: str, old_str: str, new_str: str) -> str:
    f = safe_path(path)
    text = f.read_text()
    count = text.count(old_str)
    if count != 1:
        return (f"Error: old_str appears {count} times in {path}. "
                "It must appear exactly once. Include more surrounding lines.")
    f.write_text(text.replace(old_str, new_str))
    return f"Edited {path}."

@registry.register(
    "run_command",
    "Run a shell command in the repo (tests, builds, linters). Times out after 120s.",
    {"type": "object",
     "properties": {"cmd": {"type": "string"}},
     "required": ["cmd"]})
def run_command(cmd: str) -> str:
    # in production this executes inside the sandbox, not on the host
    r = subprocess.run(cmd, shell=True, cwd=WORKSPACE, capture_output=True,
                       text=True, timeout=120)
    return f"exit={r.returncode}\nstdout:\n{r.stdout}\nstderr:\n{r.stderr}"
```
