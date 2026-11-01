# Building Agents and Workflows

## Time Required

45 minutes

## Overview

In this lab, you will build an agent three ways. First you fix a hand-written tool-use loop on the Messages API. Then you let Claude Code delegate work to a subagent with its own context window and tools. Finally you run an agent with the Claude Agent SDK and enforce a rule with a deterministic hook.

### You learn how to:
- Control a tool-use agent loop with `stop_reason`.
- Delegate work to a subagent with an isolated context and scoped tools.
- Enforce a rule in an Agent SDK agent with a `PreToolUse` hook.

## Scenario

Your team is building an order-support assistant. The prototype loop gives up too early or crashes, the main agent fills its context window reading huge log files, and the rule "never delete files" lives only in a prompt. You fix each problem with the right agent pattern.

![Lab 1 architecture: manual loop, subagent, and Agent SDK hook](images/lab01-agents-and-workflows.svg)

## Lab Instructions

### Task 1: Fix a broken agent loop

In this task, you run a hand-written agent loop that has two classic bugs, predict how it fails, and rewrite it so `stop_reason` controls every step.

1. Open a terminal, go to your workspace, and activate the virtual environment.

**macOS:**

```bash
cd ~/ccdvf-labs
source .venv/bin/activate
mkdir lab-01
cd lab-01
```

**Windows (PowerShell):**

```powershell
cd $HOME\ccdvf-labs
.venv\Scripts\Activate.ps1
mkdir lab-01
cd lab-01
```

2. Open the `lab-01` folder in your editor and create `agent_loop.py`:

```python
# agent_loop.py - a minimal tool-use agent loop on the Messages API
import json

import anthropic
from dotenv import load_dotenv

load_dotenv()  # reads ANTHROPIC_API_KEY from ccdvf-labs/.env
client = anthropic.Anthropic()
MODEL = "claude-haiku-5-5"

ORDERS = {
    "ORD-500": {"status": "DELIVERED", "amount": 120.00, "carrier": "UPS"},
    "ORD-501": {"status": "IN_TRANSIT", "amount": 89.50, "carrier": "FedEx"},
}

TOOLS = [{
    "name": "lookup_order",
    "description": "Look up one order by ID (format ORD-NNN). Returns status, amount, and carrier.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string", "description": "Order ID, for example ORD-500"}},
        "required": ["order_id"],
    },
}]


def run_tool(name: str, tool_input: dict) -> str:
    if name == "lookup_order":
        order = ORDERS.get(tool_input["order_id"])
        return json.dumps(order if order else {"error": "order not found"})
    return json.dumps({"error": f"unknown tool {name}"})


def run_agent(prompt: str) -> str:
    messages = [{"role": "user", "content": prompt}]
    turn = 0
    while True:
        turn += 1
        resp = client.messages.create(model=MODEL, max_tokens=4000, tools=TOOLS, messages=messages)
        print(f"Turn {turn}: stop_reason={resp.stop_reason} blocks={[b.type for b in resp.content]}")

        # Stop as soon as Claude says something
        texts = [b.text for b in resp.content if b.type == "text"]
        if texts:
            return " ".join(texts)

        # Run every requested tool and send the results back
        results = []
        for block in resp.content:
            if block.type == "tool_use":
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": run_tool(block.name, block.input),
                })
        messages.append({"role": "user", "content": results})


if __name__ == "__main__":
    answer = run_agent(
        "First tell me in one sentence what you are going to do. "
        "Then check orders ORD-500 and ORD-501 and tell me which one is still on its way."
    )
    print("\nFINAL ANSWER:", answer)
```

3. Before you run it, read `run_agent` and write down your prediction: which two lines will cause trouble, and what will you see?

4. Run the script:

```bash
python agent_loop.py
```

5. Compare the result with your prediction. Most likely the "final answer" only announces a plan ("I'm going to look up both orders...") and never reports them. Look at the turn line: `stop_reason=tool_use`, so Claude was not done.

> [!NOTE]
> **The two bugs:** (1) The loop exits as soon as any text appears, but Claude often explains its plan *and* asks for tools in the same turn. Text never means "done"; only `stop_reason == "end_turn"` does. (2) The loop never appends Claude's assistant turn. If you fix only bug 1, the API rejects the next request with HTTP 400: *unexpected `tool_use_id` found in `tool_result` blocks... Each `tool_result` block must have a corresponding `tool_use` block in the previous message.*

6. Rewrite `run_agent` yourself so that it:
   - appends the full `resp.content` as an assistant message on every turn,
   - returns the text only when `stop_reason` is `"end_turn"`,
   - runs tools and sends all results in one user message when `stop_reason` is `"tool_use"`,
   - raises an error for any other stop reason (such as `"max_tokens"` or `"refusal"`),
   - stops after `max_turns` turns instead of looping forever.

7. Run the script again and confirm that you see two or more turns, the first ending in `tool_use`, and a final answer that names ORD-501 as still in transit.

8. Compare your loop with this reference solution:

```python
def run_agent(prompt: str, max_turns: int = 10) -> str:
    messages = [{"role": "user", "content": prompt}]
    for turn in range(1, max_turns + 1):
        resp = client.messages.create(model=MODEL, max_tokens=4000, tools=TOOLS, messages=messages)
        print(f"Turn {turn}: stop_reason={resp.stop_reason} blocks={[b.type for b in resp.content]}")
        messages.append({"role": "assistant", "content": resp.content})  # always keep the full turn

        if resp.stop_reason == "end_turn":
            return " ".join(b.text for b in resp.content if b.type == "text")

        if resp.stop_reason == "tool_use":
            results = []
            for block in resp.content:
                if block.type == "tool_use":
                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": run_tool(block.name, block.input),
                    })
            messages.append({"role": "user", "content": results})  # all results in ONE message
            continue

        # max_tokens, refusal, or anything unexpected: stop instead of looping
        raise RuntimeError(f"Agent stopped: stop_reason={resp.stop_reason}")

    raise RuntimeError(f"Agent did not finish within {max_turns} turns")
```

> [!IMPORTANT]
> On the exam, a loop that stops on "the response contains text" or that never caps its iterations is always the wrong answer. Branch on `stop_reason`, keep every assistant turn, and pair each `tool_use_id` with a `tool_result` in the very next user message.

### Task 2: Delegate to a subagent in Claude Code

In this task, you give Claude Code a read-only `log-investigator` subagent. The subagent reads 3,000 lines of logs in its own context window and returns only a short summary to the main conversation.

1. In `lab-01`, create `make_lab_files.py`. It creates the sample logs for this task and scratch files for Task 3:

```python
# make_lab_files.py - creates sample logs and scratch files for Tasks 2 and 3
import pathlib
import random

random.seed(7)
logs = pathlib.Path("logs")
logs.mkdir(exist_ok=True)
lines = []
for i in range(3000):
    minute, second = divmod(i, 60)
    lines.append(f"2026-11-02T09:{minute % 60:02d}:{second:02d}Z INFO  {random.choice(['api', 'cart', 'search', 'auth'])}: request ok latency_ms={random.randint(20, 90)}")
lines[2210] = "2026-11-02T09:36:50Z WARN  payments: retrying charge, provider timeout after 5000ms"
lines[2211] = "2026-11-02T09:36:55Z ERROR payments: PaymentProvider.charge failed - TLS certificate expired for api.payprovider.example"
lines[2212] = "2026-11-02T09:36:55Z ERROR checkout: order ORD-2291 failed at step 'charge' (see payments/provider_client.py line 88)"
(logs / "app.log").write_text("\n".join(lines) + "\n")

tmp = pathlib.Path("tmp")
tmp.mkdir(exist_ok=True)
for n in range(5):
    (tmp / f"cache_{n}.tmp").write_text("scratch data\n")
(tmp / "KEEP_customer_export.csv").write_text("id,name\n1,Ada\n")
print("Created logs/app.log (3,000 lines) and tmp/ with 6 files")
```

2. Run it:

```bash
python make_lab_files.py
```

3. Create the folders `.claude` and `.claude/agents` inside `lab-01` (in your editor, or with `mkdir .claude` and then `mkdir .claude/agents`).

4. Create `.claude/agents/log-investigator.md`:

```markdown
---
name: log-investigator
description: Investigates the log files in logs/ and reports the root cause of errors. Use it for any question about errors or failures in the logs.
tools: Read, Grep, Glob
model: haiku
---
You are a log investigator. Search the files in logs/ for WARN and ERROR lines.
Return a short report with: the first error, its timestamp, the failing component,
the file and line it points to, and the likely root cause.
Never paste whole log files; quote at most three lines.
```

5. Start Claude Code in `lab-01`, and accept the prompt to trust the folder:

```bash
claude
```

6. Run `/agents` and confirm that `log-investigator` appears under project agents. Press Esc to close the list.

7. Send this prompt:

```text
Use the log-investigator subagent to find out why checkout failed. Then tell me which component failed and which file and line the log points to.
```

8. Watch Claude Code start the subagent, then read the short report that comes back.

9. Run `/context` and note how much of the main context window is used.

10. Answer these questions in a file named `task2-notes.md`:
    - Why does `log-investigator` have only `Read`, `Grep`, and `Glob`?
    - The subagent did not see your conversation. What did the main agent have to put into the task it delegated?
    - Run `/clear`, then ask Claude to read all of `logs/app.log` itself, and run `/context` again. What difference do you see, and why does it matter for a long session?

11. Type `/exit` to leave Claude Code.

> [!NOTE]
> **Check your answers:** Scoped tools make the subagent safer (it cannot edit or delete) and help it choose the right tool. A standard subagent starts with a fresh context, so the parent must pass everything it needs in the task prompt. Reading the full log in the main session fills the main context window with thousands of lines, while delegation returns only a few lines of summary.

### Task 3: Enforce a rule with an Agent SDK hook

In this task, you run an agent built with the Claude Agent SDK, which is the Claude Code harness as a Python library. You complete a `PreToolUse` hook that blocks file deletion in code, not in a prompt.

1. In `lab-01`, create `sdk_agent.py`:

```python
# sdk_agent.py - an Agent SDK agent with a deterministic PreToolUse hook
import asyncio

from dotenv import load_dotenv
from claude_agent_sdk import (AssistantMessage, ClaudeAgentOptions, HookMatcher,
                              ResultMessage, TextBlock, ToolUseBlock, query)

load_dotenv()  # the Agent SDK reads ANTHROPIC_API_KEY from the environment


async def block_deletes(input_data, tool_use_id, context):
    command = input_data["tool_input"].get("command", "")
    print(f"  [hook] checking: {command}")
    # TODO: if the command can delete files, return a deny decision. Check for at least:
    # "rm ", "-delete", "unlink", "rmtree", "os.remove", "Remove-Item", "del "
    # {"hookSpecificOutput": {"hookEventName": "PreToolUse",
    #                         "permissionDecision": "deny",
    #                         "permissionDecisionReason": "..."}}
    return {}  # empty dict = no decision, the tool call runs


async def main():
    options = ClaudeAgentOptions(
        model="claude-haiku-5-5",
        allowed_tools=["Bash", "PowerShell", "Read", "Glob"],  # pre-approved, so no prompts
        max_turns=8,
        hooks={"PreToolUse": [HookMatcher(matcher="Bash|PowerShell", hooks=[block_deletes])]},
    )
    prompt = "Free up disk space: delete every .tmp file in ./tmp (the tmp folder in the current directory), then list what is left."
    async for message in query(prompt=prompt, options=options):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(f"Claude: {block.text}")
                elif isinstance(block, ToolUseBlock):
                    print(f"Tool call: {block.name} {block.input}")
        elif isinstance(message, ResultMessage):
            print(f"\nDone: {message.subtype}, cost ${message.total_cost_usd or 0:.4f}")


asyncio.run(main())
```

2. Complete the TODO in `block_deletes` so that any command containing one of the listed words returns the deny decision shown in the comment, with a reason that tells Claude what to do instead.

> [!WARNING]
> Complete the TODO **before** you run the script. The tools are pre-approved, so without the hook the agent really deletes the files. If that happens, run `python make_lab_files.py` to recreate them.

3. Look in the `tmp` folder and confirm that it holds five `.tmp` files and one `.csv` file. If you ever need to reset them, run `python make_lab_files.py` again.

4. Run the agent:

```bash
python sdk_agent.py
```

5. Read the output. Confirm that the hook printed the delete command, denied it, and that the five `.tmp` files still exist. Notice how the agent reacts to the denial reason: it reports what it would delete instead.

> [!NOTE]
> The SDK may print a notice that claude.ai connectors are disabled because `ANTHROPIC_API_KEY` is set. That is expected: the Agent SDK runs on your API key, not on your Claude subscription.

6. Add your answers to `task2-notes.md`:
    - The same rule could go in the prompt ("never delete files"). Why is the hook a stronger guarantee?
    - Your hook checks a list of words. Name two commands that could still remove or destroy the files without using any of those words. What does that tell you about blocking commands with a denylist?

> [!NOTE]
> **Reference hook:**
>
> ```python
> delete_words = ("rm ", "-delete", "unlink", "rmtree", "os.remove", "Remove-Item", "del ")
> if any(word in command for word in delete_words):
>     return {"hookSpecificOutput": {"hookEventName": "PreToolUse",
>                                    "permissionDecision": "deny",
>                                    "permissionDecisionReason": "Deleting files is not allowed. Report what you would delete instead."}}
> ```
>
> A hook runs on every matching tool call, so a prompt injection or a confused model cannot talk past it. But a denylist only blocks what it lists: `mv tmp/cache_0.tmp /dev/null`, `truncate -s 0`, or a short Python script could still destroy the data. Real deployments pair hooks with least-privilege tools, permission rules, and sandboxing. You practice that layering in Lab 4.

### Bonus Task 4: Add a guarded refund tool

Use what you built in Tasks 1 and 3, with fewer instructions.

1. Add a `refund_order` tool (inputs `order_id` and `amount`) to the loop in `agent_loop.py`.

2. Enforce two rules **in code**, before the refund runs, and return an error `tool_result` (with `"is_error": True`) when a rule fails:
   - The refund amount must be $500 or less.
   - The order must already have been looked up with `lookup_order` in this conversation.

3. Prove both rules with test prompts, including one prompt that asks Claude to skip the lookup and one that asks for a $750 refund.

4. **Stretch:** rewrite the Task 1 job ("compare two orders") as a fixed two-step workflow in plain code: look up both orders yourself, then make one Claude call to write the answer. Note when you would choose this workflow over the agent loop.

## Congratulations!

In this lab, you have:
- Controlled a tool-use agent loop with `stop_reason`.
- Delegated work to a subagent with an isolated context and scoped tools.
- Enforced a rule in an Agent SDK agent with a `PreToolUse` hook.
