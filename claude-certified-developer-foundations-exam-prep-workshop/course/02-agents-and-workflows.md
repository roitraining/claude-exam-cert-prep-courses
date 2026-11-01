<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 2: Agents and Workflows

---

# Chapter 2: Objectives

- Choose between deterministic workflows and autonomous agents
- Design supervisor hierarchies and subagent delegation
- Build agents with the Claude Agent SDK and lifecycle hooks
- Compare deployment models and agentic frameworks

---

# Domain 1: Agents and Workflows

| Task # | Task Statement |
| :--- | :--- |
| **1.1** | Agent Architecture: Principles, patterns, and trade-offs of workflow vs agent architecture, manager/supervisor hierarchies, and subagents |
| **1.2** | Agent Construction with Claude: Claude Agent SDK, custom agent loops and harnesses, managed vs self-hosted deployment, and deterministic hooks |
| **1.3** | Agent Patterns and Frameworks: Tool-use loops, subagents, memory, context-window management, and frameworks (`Strands`, `LangGraph`, `PydanticAI`) |

---

<!-- layout: navigation -->
# Chapter 2

- **Agent Architecture**
- Agent Construction with Claude
- Agent Patterns and Frameworks

---

<!-- layout: stacked -->
# Decision Criteria: Workflow vs Agent

- **Choose a Workflow** when the task follows a predictable, repeatable sequence of steps that can be orchestrated in application code (prompt chaining, routing, parallel execution)
- **Choose an Agent** when the problem is open-ended and the number or order of tool calls depends dynamically on what the model discovers at runtime
- **Hybrid reliability**: wrap autonomous agent loops inside deterministic guardrails and prerequisite hooks

![Workflow vs Agent Architecture](images/ch02-workflow-vs-agent-architecture.svg)

---

# Five Workflow Patterns to Recognize

| Pattern | How It Works | Choose It When |
| :--- | :--- | :--- |
| **Prompt chaining** | Output of step 1 feeds step 2, with code “gates” between steps | The task splits cleanly into fixed sequential stages |
| **Routing** | A cheap classifier call sends each input to a specialized prompt or model | Inputs fall into distinct categories (billing vs technical vs refund) |
| **Parallelization** | Sectioning (independent subtasks) or voting (same task run several times) | Subtasks are independent, or you need higher confidence |
| **Orchestrator-workers** | A lead model decides subtasks at runtime and delegates to workers | You cannot predict the subtasks in advance |
| **Evaluator-optimizer** | One call generates, another critiques, then loop | Clear quality criteria exist and iteration helps |

---

<!-- layout: 2-column -->
# Manager/Supervisor Hierarchies and Subagents

### Manager / Supervisor Role
- Receives the high-level goal, decomposes it into scoped subtasks, and delegates each subtask to the appropriate specialist subagent
- Routes all inter-agent communication, aggregates results, and evaluates whether additional iterations are needed
- Holds the `Agent` tool (formerly named `Task`) in its allowed tools so it can spawn subagents

### Role of Specialized Subagents
- **Context isolation**: subagents execute verbose searches or file inspections in separate context windows so raw outputs do not bloat the supervisor’s history
- **Focused tool sets**: each subagent is restricted to 4 to 5 role-specific tools, improving tool-selection accuracy
- **Explicit context passing**: a standard subagent does not inherit the supervisor’s history; pass all required inputs in the task prompt (only “fork” subagents copy the parent context)

---

<!-- layout: navigation -->
# Chapter 2

- Agent Architecture
- **Agent Construction with Claude**
- Agent Patterns and Frameworks

---

# Constructing Custom Agent Loops and Harnesses

- **Handle every `stop_reason` and cap iterations**: unhandled values loop forever

```python
async def run_agent_loop(user_prompt: str, tools: list[dict], max_turns: int = 20):
    messages = [{"role": "user", "content": user_prompt}]
    for _ in range(max_turns):
        resp = await client.messages.create(model="claude-sonnet-5-5",
            max_tokens=16000, tools=tools, messages=messages)
        messages.append({"role": "assistant", "content": resp.content})
        if resp.stop_reason == "end_turn": return resp
        if resp.stop_reason == "tool_use":  # all results in ONE user message
            messages.append({"role": "user", "content": await run_tools(resp.content)})
        elif resp.stop_reason != "pause_turn":  # pause_turn: resend to resume
            raise AgentStopError(resp.stop_reason)  # max_tokens, refusal
    raise AgentStopError("max_turns exceeded")
```
---

# Building with the Claude Agent SDK

- **Claude Code’s harness as a library**: built-in tools, loop, permissions, subagents, hooks

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Glob", "Agent"],  # Agent = spawn subagents
    permission_mode="default",
    agents={"security-reviewer": AgentDefinition(
        description="Reviews diffs for exploitable vulnerabilities",
        prompt="Report only exploitable flaws.", tools=["Read", "Grep"])},
)

async for message in query(prompt="Audit the auth module", options=options):
    print(message)
```

---

# Four Ways to Build a Claude Agent

| Approach | Who Runs the Loop | Who Hosts It | Choose It When |
| :--- | :--- | :--- | :--- |
| **Manual loop** (Messages API) | You write it | You | You need full control of every turn |
| **Tool Runner** (`client.beta.messages.tool_runner`) | The API SDK, over tools you define | You | Custom-tool agent without hand-writing the loop |
| **Claude Agent SDK** (`claude-agent-sdk`) | Claude Code harness with built-in tools | You (your VPC, containers) | Coding or filesystem agents on your own infrastructure |
| **Claude Managed Agents** (beta) | Anthropic | Anthropic: per-session sandbox | Hosted, long-running, versioned agents with minimal infrastructure |

> [!NOTE]
> Self-hosted: direct access to private systems, but you own scaling and sandboxing. Managed: versioned agents, hosted sandboxes, vault credentials; expose private systems via remote MCP.

---

<!-- layout: card-layout -->
# Deterministic Actions with Agent SDK Hooks

### `PreToolUse` Interception
- Fires before a tool call executes and can return `permissionDecision: "deny"`, `"ask"`, or `"allow"`.
- Use to verify prerequisites (such as requiring `get_customer` before `process_refund`), enforce monetary thresholds, or block destructive shell commands (`rm -rf`).

### `PostToolUse` Normalization and Pruning
- Fires immediately after a tool returns its raw output, before the result is appended to Claude’s context window.
- Use to normalize inconsistent date/status formats and strip verbose, irrelevant JSON fields.

### Why Hooks Beat Prompt Rules
- Prompt instructions (“Never issue a refund over $500”) are probabilistic.
- Hooks run in deterministic code on every matching call, so the rule is enforced exactly as written: the hook is only as complete as its logic.

---

<!-- layout: navigation -->
# Chapter 2

- Agent Architecture
- Agent Construction with Claude
- **Agent Patterns and Frameworks**

---

<!-- layout: card-layout -->
# Core Agent Design Patterns

### Tool-Use Loop Pattern
- Iterative cycle where Claude emits one or more `tool_use` blocks, inspects the returned `tool_result` data, and adapts its next step until `stop_reason == "end_turn"`.

### Subagent Delegation Pattern
- Spawning isolated worker agents (often in parallel by emitting multiple `Agent` tool calls, formerly `Task`, in a single turn) to prevent context window bloat on the main coordinator.

### Short-Term vs Long-Term Memory
- **Short-term memory**: the active `messages` array plus a persistent in-session “case facts” block.
- **Long-term memory**: external vector/relational stores, scratchpad files, or `CLAUDE.md` files persisted across sessions.

### Context-Window Compaction
- Pruning verbose tool outputs and summarizing older turns so multi-step agents do not hit context drift or token ceilings.

---

<!-- layout: 3-column -->
# Selecting Agentic Abstraction Frameworks

### PydanticAI
- Built around **type-safe structured outputs**, Pydantic schema validation, and clean dependency injection
- Automatically feeds Pydantic validation errors back to Claude for self-correction retries
- **Best when**: strict data contracts and Python type safety are top priorities

### LangGraph
- Models agent workflows as **stateful cyclic graphs** with explicit nodes, conditional edges, and durable state checkpointing
- Supports time-travel debugging and human-in-the-loop pause/resume interrupts
- **Best when**: orchestrating complex multi-step state machines with explicit branching and persistence

### Strands (Strands Agents)
- Lightweight, **model-driven** agent framework emphasizing simple tool registration, MCP integration, and letting Claude’s native reasoning drive the loop
- Minimal boilerplate compared to heavy graph DSLs
- **Best when**: building clean, model-centric tool-use agents and supervisor workflows rapidly

---

# Domain 1 Quiz: Agents and Workflows

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-10-ccdv-f-realistic-instructor.json)

---

# Lab 1: Building Agents and Workflows

**Time:** 45 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-developer-foundations-exam-prep-workshop/labs/lab-01-building-agents-and-workflows)

---

# What You Learned

- Chose between deterministic workflows and autonomous agents
- Designed supervisor hierarchies and subagent delegation
- Built agents with the Claude Agent SDK and lifecycle hooks
- Compared deployment models and agentic frameworks

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
