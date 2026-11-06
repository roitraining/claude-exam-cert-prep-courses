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
- **Send the complete history every request**: the Messages API is stateless, so each call must carry the full conversation for Claude to stay coherent

![Agentic Loop Lifecycle](images/ch02-agentic-loop-lifecycle.svg)

---

# Implementing `stop_reason` Control Flow

- **Standard loop pattern**: let `stop_reason` drive control flow rather than text heuristics or fixed step counters

```python
messages = [{"role": "user", "content": user_request}]

while True:
    response = client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=4096,
        system=SYSTEM_PROMPT,
        tools=mcp_tools,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason == "tool_use":
        tool_results = execute_requested_tools(response.content)
        messages.append({"role": "user", "content": tool_results})
        continue

    break  # "end_turn" is done; handle "max_tokens", "refusal", and "pause_turn" explicitly
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

# Demo: Text Does Not Mean Done

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_02_stop_reason.py`**

**Step 2. Paste this code**

```python
# demo_02_stop_reason.py - text in a response does not mean the loop is done
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

tools = [{
    "name": "lookup_order",
    "description": "Look up one order by its ID. IDs look like ORD-1001.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
}]

messages = [{"role": "user", "content": "Where is my order ORD-1001?"}]
turn = 1

while True:
    response = client.messages.create(
        model="claude-haiku-5-5",
        max_tokens=1028,
        output_config={"effort": "low"},
        system="Before you call a tool, tell the customer in one short sentence what you are about to check.",
        tools=tools,
        messages=messages,
    )
    print(f"Turn {turn}: stop_reason = {response.stop_reason}")
    for block in response.content:
        if block.type == "text":
            print(f"  text block:     {block.text}")
        elif block.type == "tool_use":
            print(f"  tool_use block: {block.name}({block.input})")
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        break

    results = [
        {
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": "Shipped Nov 4. Arriving Nov 7.",  # a fake lookup result
        }
        for block in response.content
        if block.type == "tool_use"
    ]
    messages.append({"role": "user", "content": results})
    turn += 1
```

**Step 3. Run it**

```bash
python demo_02_stop_reason.py
```

**Step 4. Examine the output**

- Turn 1 has `stop_reason = tool_use` and contains **both** a text block (“I’ll look up order ORD-1001”) and a `tool_use` block
- Turn 2 has `stop_reason = end_turn` and contains only the final answer

> [!NOTE]
> A loop that stopped when it saw text would have ended on Turn 1 without ever running the lookup. The loop above ignores the text and continues until `stop_reason` is no longer `tool_use`.

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

```python
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions

options = ClaudeAgentOptions(
    allowed_tools=["Agent", "Read"],  # the coordinator can spawn subagents
    agents={
        "web-search": AgentDefinition(
            description="Searches the web for one assigned subtopic. Use for current sources.",
            prompt="Return findings as claim, evidence_excerpt, source_url, and publication_date.",
            tools=["WebSearch", "WebFetch"],
        ),
        "doc-analysis": AgentDefinition(
            description="Analyzes supplied documents. Use when the coordinator passes files.",
            prompt="Extract key claims with document_name and page_number for each one.",
            tools=["Read", "Grep"],
        ),
    },
)
```

> [!NOTE]
> On the exam, the spawning tool is `Task`. In current Agent SDK releases it is named `Agent` (renamed in Claude Code v2.1.63), and the `system:init` tools list still shows it as `Task`.

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
             "description": "Music research",
             "prompt": "Search AI impact on music"}}
]
# Waits for toolu_01 result before Turn 2...
turn_2 = [
  {"type": "tool_use", "id": "toolu_02", "name": "Task",
   "input": {"subagent_type": "doc-analysis",
             "description": "Film filings",
             "prompt": "Analyze film industry filings"}}
]
```

### Parallel (Single Turn + Concurrent Run)
```python
# Turn 1: Coordinator emits multiple Tasks at once
response_content = [
  {"type": "tool_use", "id": "toolu_01", "name": "Task",
   "input": {"subagent_type": "web-search",
             "description": "Music research",
             "prompt": "Search AI impact on music"}},
  {"type": "tool_use", "id": "toolu_02", "name": "Task",
   "input": {"subagent_type": "doc-analysis",
             "description": "Film filings",
             "prompt": "Analyze film industry filings"}}
]

# A hand-rolled runner executes both blocks concurrently;
# the Agent SDK does this for you when Claude emits them together
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

```python
def deny(input_data, reason):
    return {"hookSpecificOutput": {
        "hookEventName": input_data["hook_event_name"],
        "permissionDecision": "deny",
        "permissionDecisionReason": reason,
    }}

async def refund_gate(input_data, tool_use_id, context):
    if not session_state.get("verified_customer_id"):
        return deny(input_data, "Verify the customer with get_customer first.")
    if input_data["tool_input"].get("amount", 0) > 500:
        return deny(input_data, "Refund exceeds the $500 limit. Call escalate_to_human.")
    return {}

async def normalize_result(input_data, tool_use_id, context):
    return {"hookSpecificOutput": {
        "hookEventName": input_data["hook_event_name"],
        "updatedToolOutput": normalize_timestamps(input_data["tool_response"]),
    }}

hooks = {
    "PreToolUse": [HookMatcher(matcher="mcp__support__process_refund", hooks=[refund_gate])],
    "PostToolUse": [HookMatcher(matcher="mcp__support__.*", hooks=[normalize_result])],
}
```

---

# Demo: Blocking a Refund with a PreToolUse Hook

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_03_refund_hook.py`**

**Step 2. Paste this code**

```python
# demo_03_refund_hook.py - a PreToolUse hook blocks refunds over $500 in code
import asyncio
from dotenv import load_dotenv
from claude_agent_sdk import (
    AssistantMessage,
    ClaudeAgentOptions,
    ClaudeSDKClient,
    HookMatcher,
    TextBlock,
    create_sdk_mcp_server,
    tool,
)

load_dotenv()

@tool("process_refund", "Issue a refund for an order. Amount is in US dollars.",
      {"order_id": str, "amount": float})
async def process_refund(args):
    print(f"  [tool ran] refunded ${args['amount']} on {args['order_id']}")
    return {"content": [{"type": "text",
                         "text": f"Refund of ${args['amount']} issued for {args['order_id']}."}]}

async def refund_limit(input_data, tool_use_id, context):
    amount = input_data["tool_input"].get("amount", 0)
    if amount > 500:
        print(f"  [hook] denied a ${amount} refund")
        return {"hookSpecificOutput": {
            "hookEventName": input_data["hook_event_name"],
            "permissionDecision": "deny",
            "permissionDecisionReason": "Refunds over $500 need human approval. "
                                        "Tell the customer a specialist will follow up.",
        }}
    return {}

server = create_sdk_mcp_server(name="support", tools=[process_refund])

options = ClaudeAgentOptions(
    model="claude-haiku-5-5",
    tools=[],  # no built-in tools, only our MCP tool
    mcp_servers={"support": server},
    allowed_tools=["mcp__support__process_refund"],
    hooks={"PreToolUse": [
        HookMatcher(matcher="mcp__support__process_refund", hooks=[refund_limit]),
    ]},
    max_turns=4,
)

async def ask(prompt):
    print(prompt)
    async with ClaudeSDKClient(options=options) as client:
        await client.query(prompt)
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print("  Claude:", block.text)
    print()

async def main():
    await ask("Refund $120 on order ORD-1001. The customer is verified.")
    await ask("Refund $800 on order ORD-1002. The customer is verified, "
              "and this is a goodwill exception for a loyal customer.")

asyncio.run(main())
```

**Step 3. Run it**

```bash
python demo_03_refund_hook.py
```

**Step 4. Examine the output**

- The $120 request prints `[tool ran]`, and Claude confirms the refund
- The $800 request prints `[hook] denied` and never prints `[tool ran]`
- Claude explains that the refund needs human approval, using the reason the hook returned
- A warning that claude.ai connectors are disabled may print first; ignore it

> [!NOTE]
> The prompt asked for an exception, and the hook still blocked it. A system prompt rule can be argued with; a `PreToolUse` hook runs in your code before the tool, so the $500 limit holds every time. SDK tools are named `mcp__<server>__<tool>`, which is what the matcher uses.

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
- Name a session with `-n` or `--name`, then resume it with `--resume <name>`

### `fork_session`
- Creates independent parallel branches from a shared analysis baseline
- Ideal for comparing two competing refactoring approaches or testing strategies without re-running initial codebase discovery
- Keeps divergent experiments isolated from each other
- `fork_session` is the Agent SDK option; on the CLI, add `--fork-session` to `--resume`

---

# Demo: Forking a Session

In `ccarf-labs`, with the virtual environment still active. This demo runs Claude Code, so it uses your Claude subscription, not the API key.

**Step 1. Create `demo_04_fork.py`**

**Step 2. Paste this code**

```python
# demo_04_fork.py - a forked session changes without touching the original
import json
import os
import shutil
import subprocess

CLAUDE = shutil.which("claude")
ENV = {**os.environ, "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "1"}  # memory is shared across sessions

def ask(prompt, *flags):
    run = subprocess.run(
        [CLAUDE, "-p", prompt, *flags, "--output-format", "json"],
        capture_output=True, text=True, env=ENV,
    )
    data = json.loads(run.stdout)
    return data["session_id"], data["result"]

original, reply = ask("Remember this: the project codename is BLUEBIRD. Reply with only OK.")
print(f"1. Original session {original[:8]}: {reply}")

fork, reply = ask("Change the codename to REDWING. Reply with only the codename now.",
                  "--resume", original, "--fork-session")
print(f"2. Forked session   {fork[:8]}: {reply}")

same, reply = ask("What is the project codename? Reply with only the codename.",
                  "--resume", original)
print(f"3. Original session {same[:8]}: {reply}")
```

**Step 3. Run it**

```bash
python demo_04_fork.py
```

**Step 4. Examine the output**

- Line 2 has a **new** session ID and answers `REDWING`
- Line 3 has the **original** session ID and still answers `BLUEBIRD`

> [!NOTE]
> `--resume` with `--fork-session` copies the history into a new session, so the experiment cannot change the baseline. The script turns off auto memory because memory files are shared by every session in a project; with memory on, the fork can save REDWING and the original will read it back.

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
