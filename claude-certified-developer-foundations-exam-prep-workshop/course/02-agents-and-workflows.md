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

<!-- layout: 2-column -->
# Manager/Supervisor Hierarchies and Subagents

### Manager / Supervisor Role
- Receives the high-level goal, decomposes it into scoped subtasks, and delegates each subtask to the appropriate specialist subagent
- Routes all inter-agent communication, aggregates results, and evaluates whether additional iterations are needed
- Holds the `"Task"` tool in its `allowedTools` configuration

### Role of Specialized Subagents
- **Context isolation**: subagents execute verbose searches or file inspections in separate context windows so raw outputs do not bloat the supervisor’s history
- **Focused tool sets**: each subagent is restricted to 4 to 5 role-specific tools, improving tool-selection accuracy
- **Explicit context passing**: subagents never inherit supervisor history automatically; pass all required inputs in the task prompt

---

<!-- layout: navigation -->
# Chapter 2

- Agent Architecture
- **Agent Construction with Claude**
- Agent Patterns and Frameworks

---

# Constructing Custom Agent Loops and Harnesses

- **Core harness loop**: send `messages` and `tools` -> check `response.stop_reason` -> if `"tool_use"`, execute requested tools and append `tool_result` blocks -> stop when `response.stop_reason == "end_turn"`

```python
async def run_agent_loop(user_prompt: str, tools: list[dict]):
    messages = [{"role": "user", "content": user_prompt}]
    while True:
        resp = await client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            tools=tools,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": resp.content})
        if resp.stop_reason == "end_turn":
            return resp
        if resp.stop_reason == "tool_use":
            results = await dispatch_tool_calls(resp.content)
            messages.append({"role": "user", "content": results})
```

---

<!-- layout: 2-column -->
# Managed vs Self-Hosted Agent Deployment Models

### Self-Hosted Agent Harness
- Your application runs the agent loop, manages state persistence, and executes tool calls inside your own VPC or cloud infrastructure
- **Strengths**: direct low-latency access to private internal databases/APIs, full control over custom Python/TypeScript hooks, and custom persistence stores
- **Trade-off**: your team owns container scaling, sandbox isolation, and session storage

### Anthropic-Hosted / Managed Execution
- The platform manages loop orchestration, container sandboxing, and built-in server-side tool execution
- **Strengths**: rapid deployment, built-in execution sandboxes, and reduced infrastructure maintenance
- **Trade-off**: external private systems must be exposed securely (for example, via authenticated remote MCP servers or client-side tool callbacks)

---

<!-- layout: card-layout -->
# Deterministic Actions with Agent SDK Hooks

### `PreToolUse` Interception
- Fires deterministically before a tool call executes.
- Use to verify prerequisites (such as requiring `get_customer` before `process_refund`), enforce monetary thresholds, or block destructive shell commands (`rm -rf`).

### `PostToolUse` Normalization and Pruning
- Fires immediately after a tool returns its raw output, before the result is appended to Claude’s context window.
- Use to normalize inconsistent date/status formats and strip verbose, irrelevant JSON fields.

### Why Hooks Beat Prompt Rules
- Prompt instructions (“Never issue a refund over $500”) are probabilistic.
- SDK hooks execute in deterministic code and guarantee 100% policy enforcement.

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
- Spawning isolated worker agents (often in parallel by emitting multiple `Task` calls in a single turn) to prevent context window bloat on the main coordinator.

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

- Open the Exam Prep Program to complete the Domain 1 review questions

<!-- TODO: Add Exam Prep Program URL for Domain 1 Quiz -->

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
