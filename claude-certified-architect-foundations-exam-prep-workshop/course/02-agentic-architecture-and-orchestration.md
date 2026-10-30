<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 2: Agentic Architecture and Orchestration

---

# Chapter 2: Objectives

- Implement agentic loops using `stop_reason` control flow
- Orchestrate coordinator and subagent workflows
- Enforce compliance rules with Agent SDK hooks
- Manage session state, resumption, and forking

---

# Domain 1: Agentic Architecture and Orchestration

| Task # | Task Statement |
| :--- | :--- |
| **1.1** | Design and implement agentic loops for autonomous task execution |
| **1.2** | Orchestrate multi-agent systems with coordinator-subagent patterns |
| **1.3** | Configure subagent invocation, context passing, and spawning |
| **1.4** | Implement multi-step workflows with enforcement and handoff patterns |
| **1.5** | Apply Agent SDK hooks for tool call interception and data normalization |
| **1.6** | Design task decomposition strategies for complex workflows |
| **1.7** | Manage session state, resumption, and forking |

---

<!-- layout: navigation -->
# Chapter 2

- **Agentic Loops**
- Multi-Agent Orchestration
- Workflows and SDK Hooks
- Session State and Forking

---

<!-- layout: stacked -->
# The Agentic Loop Lifecycle

- **Inspect `stop_reason` on every response**: continue the loop when `stop_reason == "tool_use"` and terminate when `stop_reason == "end_turn"`
- **Append tool results to conversation history**: return each `tool_result` block in a `"user"` message so Claude can reason about the new information on the next turn
- **Preserve assistant tool-call blocks**: include the assistant’s full response content in history before appending the corresponding `tool_result`

![Agentic Loop Lifecycle](images/ch02-agentic-loop-lifecycle.svg)

---

# Implementing `stop_reason` Control Flow

- **Standard loop pattern**: let `stop_reason` drive control flow rather than text heuristics or fixed step counters

```python
messages = [{"role": "user", "content": user_request}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        system=SYSTEM_PROMPT,
        tools=mcp_tools,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason == "end_turn":
        break

    if response.stop_reason == "tool_use":
        tool_results = execute_requested_tools(response.content)
        messages.append({"role": "user", "content": tool_results})
```

---

<!-- layout: 2-column -->
# Model-Driven Loops vs Pre-Configured Trees

### Model-Driven Decision-Making
- Claude reasons dynamically about which tool to call next based on live tool outputs and conversation context
- Adapts naturally when a customer raises multiple issues or a lookup returns unexpected results
- Best when investigation paths depend on what each step discovers

### Pre-Configured Decision Trees
- Hard-codes a fixed sequence of tool calls before runtime regardless of intermediate findings
- Brittle when requests are ambiguous or combine multiple concerns
- Use programmatic gates only for mandatory prerequisites (such as identity verification before refunds), not to replace reasoning

<!-- below-columns -->

> [!IMPORTANT]
> Combine both where needed: allow model-driven reasoning within the loop, while enforcing non-negotiable prerequisites with programmatic gates.

---

<!-- layout: card-layout -->
# Three Agentic Loop Anti-Patterns on the Exam

### 1. Parsing Natural Language to Stop
- Never check the assistant’s text for phrases like “I have completed the task” or “FINAL_ANSWER” to exit the loop.
- Natural language phrasing varies and causes premature exits or infinite loops.

### 2. Checking for Assistant Text Presence
- Claude frequently emits explanatory text alongside a `tool_use` block in the same response (for example, “Let me look up your order”).
- Terminating whenever text is present aborts the loop before the requested tool is ever executed.

### 3. Arbitrary Iteration Caps as Primary Stop
- While a high safety ceiling prevents runaway billing on bugs, a low iteration cap (such as 3 turns) must never be the primary stopping mechanism.
- Always terminate normally on `stop_reason == "end_turn"`.

---

<!-- layout: navigation -->
# Chapter 2

- Agentic Loops
- **Multi-Agent Orchestration**
- Workflows and SDK Hooks
- Session State and Forking

---

<!-- layout: title-image -->
# Hub-and-Spoke Coordinator Architecture

![Hub-and-Spoke Multi-Agent Orchestration](images/ch02-hub-and-spoke-agents.svg)

---

# Coordinator Responsibilities and Context Isolation

- **Single control hub**: all inter-subagent communication, error handling, and information routing flow through the coordinator agent
- **Strict context isolation**: subagents operate in isolated context windows and **do not** inherit the coordinator’s conversation history or share memory automatically
- **Dynamic routing over rigid pipelines**: the coordinator analyzes query complexity and invokes only the required subagents rather than forcing every request through the full pipeline
- **Iterative synthesis refinement**: the coordinator evaluates synthesis output for coverage gaps, re-delegates targeted queries to search or analysis subagents, and re-invokes synthesis until complete

> [!WARNING]
> If a downstream synthesis agent is missing information that an upstream search agent found, check whether the coordinator explicitly passed those findings in the synthesis subagent’s prompt.

---

# Spawning Subagents with the `Task` Tool

- **Enabling subagent spawning**: the coordinator’s `allowedTools` configuration **must** include `"Task"` to spawn subagents
- **Configuring `AgentDefinition`**: each subagent type defines a clear description, system prompt, and scoped `allowedTools` list
- **Passing structured context**: pass complete prior findings directly in the subagent prompt, separating content from metadata (`source_url`, `document_name`, `page_number`) to preserve attribution

```typescript
const coordinator = new AgentDefinition({
  name: "research-coordinator",
  description: "Decomposes research queries, spawns subagents, and verifies synthesis coverage.",
  allowedTools: ["Task"],
  systemPrompt: "Specify research goals and quality criteria for each subagent..."
});
```

---

<!-- layout: 2-column -->
# Sequential vs Parallel Subagent Execution

### Sequential Execution (Slow)
- Coordinator emits one `Task` tool call, waits for the result, then emits the next `Task` call on a subsequent turn
- Multiplies end-to-end latency across independent research subtopics
- Appropriate only when Task B strictly depends on the output of Task A

### Parallel Execution (Fast)
- Coordinator emits **multiple `Task` tool calls in a single response turn**
- Web search and document analysis subagents run concurrently across partitioned subtopics
- Dramatically reduces total wall-clock latency for broad investigations

<!-- below-columns -->

> [!TIP]
> Design coordinator prompts to specify **research goals and quality criteria** rather than step-by-step procedural instructions so each subagent can adapt within its domain.

---

<!-- layout: 2-column -->
# Code Comparison: Sequential vs Parallel `Task` Calls

### Sequential (Multiple Turns)
```python
# Turn 1: Coordinator emits only one Task
turn_1 = [
  {"type": "tool_use", "id": "toolu_01", "name": "Task",
   "input": {"subagent_type": "web-search",
             "prompt": "Search AI impact on music"}}
]
# Waits for toolu_01 result before Turn 2...
turn_2 = [
  {"type": "tool_use", "id": "toolu_02", "name": "Task",
   "input": {"subagent_type": "doc-analysis",
             "prompt": "Analyze film industry filings"}}
]
```

### Parallel (Single Turn + Concurrent Run)
```python
# Turn 1: Coordinator emits multiple Tasks at once
response_content = [
  {"type": "tool_use", "id": "toolu_01", "name": "Task",
   "input": {"subagent_type": "web-search",
             "prompt": "Search AI impact on music"}},
  {"type": "tool_use", "id": "toolu_02", "name": "Task",
   "input": {"subagent_type": "doc-analysis",
             "prompt": "Analyze film industry filings"}}
]

# Execute all Task blocks concurrently
results = await asyncio.gather(
  *[run_subagent(block) for block in response_content]
)
```

---

# Avoiding Task Decomposition Pitfalls

- **Overly narrow decomposition**: if a user asks for the “impact of AI on creative industries” and the coordinator only spawns tasks for digital art, graphic design, and photography, the final report will completely miss music, writing, and film
- **Diagnostic rule**: when subagent logs show every subagent succeeded on its assigned task but the final report omits entire topic areas, the root cause is the **coordinator’s task decomposition**
- **Scope partitioning**: assign distinct subtopics or source types to each subagent to maximize breadth and minimize duplicate searches

---

<!-- layout: navigation -->
# Chapter 2

- Agentic Loops
- Multi-Agent Orchestration
- **Workflows and SDK Hooks**
- Session State and Forking

---

<!-- layout: 2-column -->
# Programmatic Enforcement vs Prompt Guidance

### Prompt-Based Guidance (Probabilistic)
- System prompt rules and few-shot examples guide reasoning but always carry a **non-zero failure rate** (for example, skipping `get_customer` in 12% of cases)
- Appropriate for style, tone, and flexible investigation order
- Insufficient when skipping a step causes misidentified accounts or unauthorized financial refunds

### Programmatic Enforcement (Deterministic)
- Prerequisite gates and SDK hooks inspect state in code before allowing a tool call to execute
- Blocks `lookup_order` and `process_refund` until `get_customer` has returned a verified customer ID
- Guarantees 100% compliance for security, identity, and financial boundaries

<!-- below-columns -->

> [!IMPORTANT]
> Exam Rule: Whenever a question involves financial consequences, identity verification, or strict policy limits, choose **programmatic enforcement** (hooks or prerequisite gates) over prompt instructions or few-shot examples.

---

<!-- layout: card-layout -->
# Applying Agent SDK Hooks

### Outgoing Tool Interception Hooks
- Intercept tool calls before execution to enforce hard compliance rules, such as blocking `process_refund` when `amount > 500`.
- Return a structured block message and redirect the agent to `escalate_to_human`.

### PostToolUse Normalization Hooks
- Intercept raw MCP tool outputs before the model processes them.
- Normalize heterogeneous data formats (Unix epoch timestamps, ISO 8601 dates, numeric status codes) into one canonical schema.

### Multi-Concern Decomposition
- When a customer message contains multiple distinct issues, decompose into separate items and investigate them in parallel using shared verified context.
- Synthesize all findings into one unified resolution response.

### Structured Escalation Handoffs
- Human agents often lack access to the live LLM conversation transcript.
- Compile a structured handoff payload containing `customer_id`, `root_cause`, `refund_amount`, and `recommended_action`.

---

# SDK Hook Implementation Pattern

- **Enforcing a $500 refund ceiling and normalizing timestamps** in code rather than relying on prompt compliance:

```typescript
const hooks = {
  PreToolUse: async (toolCall, sessionState) => {
    if (toolCall.name === "process_refund" && !sessionState.verifiedCustomerId) {
      return { block: true, reason: "Must verify customer via get_customer first." };
    }
    if (toolCall.name === "process_refund" && toolCall.input.amount > 500) {
      return { block: true, reason: "Refund exceeds $500 limit. Call escalate_to_human." };
    }
  },
  PostToolUse: async (toolCall, rawResult) => {
    return normalizeTimestampsAndStatusCodes(rawResult);
  }
};
```

---

<!-- layout: 2-column -->
# Task Decomposition Strategies

### Fixed Sequential Pipelines (Prompt Chaining)
- Breaks predictable multi-aspect workflows into a fixed sequence of passes
- Example: analyze each modified file individually in local passes, then run a dedicated cross-file integration pass
- Prevents attention dilution on large pull requests (such as 14+ files)

### Dynamic Adaptive Decomposition
- Generates new subtasks dynamically based on what is discovered at each step
- Example: adding comprehensive tests to an unfamiliar legacy codebase
- First maps structure and identifies high-impact modules, then adapts the prioritized plan as hidden dependencies surface

---

<!-- layout: navigation -->
# Chapter 2

- Agentic Loops
- Multi-Agent Orchestration
- Workflows and SDK Hooks
- **Session State and Forking**

---

<!-- layout: 2-column -->
# Session Resumption vs Forking

### `--resume <session-name>`
- Continues a specific named prior conversation from where it left off
- Best when continuing a single linear investigation across work sessions and prior context remains mostly valid
- If files changed externally, explicitly inform the resumed session which files were modified for targeted re-analysis

### `fork_session`
- Creates independent parallel branches from a shared analysis baseline
- Ideal for comparing two competing refactoring approaches or testing strategies without re-running initial codebase discovery
- Keeps divergent experiments isolated from each other

---

<!-- layout: card-layout -->
# Choosing How to Handle Prior Context

### Resume and Notify
- Use `--resume <name>` when 90% of prior exploration is still accurate and only 1 or 2 files were edited.
- Tell the agent exactly which files changed so it re-analyzes only those files instead of re-exploring the entire repo.

### Fork from Baseline
- Use `fork_session` right after completing a shared architectural mapping phase.
- Explore Approach A in one branch and Approach B in a second branch cleanly.

### Start Fresh with Injected Summary
- When major refactoring has invalidated most prior tool outputs in a long session, resuming with stale tool results causes confusion.
- Start a new session and inject a concise, structured summary of verified findings.

---

# Domain 1 Quiz: Agentic Architecture and Orchestration

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-14-18-ccar-f-realistic-instructor.json)

---

# Lab 1: Building a Multi-Agent System with the Claude Agent SDK

**Time:** 45 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-architect-foundations-exam-prep-workshop/labs/lab-01-building-a-multi-agent-system-with-the-claude-agent-sdk)

---

# What You Learned

- Implemented agentic loops using `stop_reason` control flow
- Orchestrated coordinator and subagent workflows
- Enforced compliance rules with Agent SDK hooks
- Managed session state, resumption, and forking

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
