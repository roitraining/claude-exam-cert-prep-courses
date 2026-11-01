# API Integration, Claude Code, and Debugging

## Time Required

45 minutes

## Overview

In this lab, you will debug a broken Messages API integration by reading real API errors, configure Claude Code so a whole team gets the same rules and guardrails, and diagnose production failures from traces while a Message Batch runs in the background.

### You learn how to:
- Identify and fix Messages API errors from real status codes and error types.
- Configure `CLAUDE.md`, path-scoped rules, permission rules, and a Skill in Claude Code.
- Process requests with the Message Batches API and isolate integration bugs from model failures.

## Scenario

Your team ships a document analytics service. A request builder that "worked last quarter" now fails, developers each configure Claude Code differently, and several production incidents were blamed on the model when the integration code was at fault.

![Lab 2 architecture: API debugging, Claude Code configuration, batches, and traces](images/lab02-api-claude-code-and-traces.svg)

## Lab Instructions

### Task 1: Debug a broken integration against the real API

In this task, you run a request builder that contains three integration bugs. You fix them one at a time by reading the real errors the API returns, then stream the fixed request.

1. Open a terminal, go to your workspace, and activate the virtual environment.

**macOS:**

```bash
cd ~/ccdvf-labs
source .venv/bin/activate
mkdir lab-02
cd lab-02
```

**Windows (PowerShell):**

```powershell
cd $HOME\ccdvf-labs
.venv\Scripts\Activate.ps1
mkdir lab-02
cd lab-02
```

2. Open `lab-02` in your editor and create `broken_request.py`:

```python
# broken_request.py - one Messages API request with three integration bugs
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

TOOLS = [{
    "name": "fetch_metrics",
    "description": "Fetch revenue metrics for one quarter, for example Q3.",
    "input_schema": {"type": "object", "properties": {"quarter": {"type": "string"}}, "required": ["quarter"]},
}]


def build_request(question: str) -> dict:
    return {
        "model": "claude-sonnet-4-20250514",
        "max_tokens": 1000,
        "tools": TOOLS,
        "messages": [
            {"role": "system", "content": "You are a financial analytics assistant."},
            {"role": "user", "content": question},
            {"role": "assistant", "content": [
                {"type": "tool_use", "id": "toolu_99", "name": "fetch_metrics", "input": {"quarter": "Q3"}},
            ]},
            {"role": "user", "content": "Also include the Q4 forecast."},
        ],
    }


if __name__ == "__main__":
    try:
        resp = client.messages.create(**build_request("Summarize Q3 revenue."))
        print(f"SUCCESS (stop_reason={resp.stop_reason}):", " ".join(b.text for b in resp.content if b.type == "text"))
        print("usage:", resp.usage.input_tokens, "in /", resp.usage.output_tokens, "out")
    except anthropic.APIStatusError as err:
        print(f"HTTP {err.status_code} {type(err).__name__}")
        print(err.message)
```

3. Create `task1-errors.md` with this table. You fill it in as you go:

```markdown
| Run | HTTP status | Error type | What was wrong | Fix |
| :-- | :-- | :-- | :-- | :-- |
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
```

4. Run the script:

```bash
python broken_request.py
```

5. Read the status code and message, record them in your table, fix only that problem in `build_request`, and run the script again. Repeat until it prints `SUCCESS`.

> [!TIP]
> Use the error message, not guesswork. The API validates the request structure before it looks up the model, so you meet the errors in this order: a 400 about the `system` role, a 400 about `tool_use` without `tool_result`, then a 404 for the model. For the `tool_result` fix, make the last user message a list that starts with a `tool_result` block for `toolu_99`, using a made-up result such as `'{"q3_revenue_musd": 412, "growth_pct": 14, "q4_forecast_musd": 440}'`, followed by the Q4 question as a text block. If the result lacks the Q4 number, Claude asks for another tool call instead of answering (`stop_reason=tool_use`).

6. Notice the `DeprecationWarning` that the SDK prints about the model on every run. The SDK warned you about the retired model before the API ever rejected it, but warnings are easy to miss in production logs.

7. Create `stream_request.py` to stream the fixed request and read the usage counters:

```python
# stream_request.py - stream the fixed request and read usage from the final message
from broken_request import build_request, client

with client.messages.stream(**build_request("Summarize Q3 revenue.")) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()

print(f"\n\nstop_reason={final.stop_reason} input={final.usage.input_tokens} output={final.usage.output_tokens}")
```

8. Run it and watch the answer appear as it is generated. Confirm that the last line shows `stop_reason=end_turn`:

```bash
python stream_request.py
```

> [!NOTE]
> **Check your fixes:** run 1 returns HTTP 400 `invalid_request_error` (use the top-level `"system"` parameter), run 2 returns HTTP 400 `invalid_request_error` (the user message after `tool_use` must start with `{"type": "tool_result", "tool_use_id": "toolu_99", "content": "..."}`), and run 3 returns HTTP 404 `not_found_error` (use a current model ID such as `claude-haiku-5-5`). All three are integration-layer bugs: no prompt change could fix them, and retrying without changing the request would fail the same way every time.

### Task 2: Configure Claude Code for the team

In this task, you add project files that give every developer the same instructions, rules, guardrails, and a reusable Skill. Then you prove that each one works.

1. In `lab-02`, create the folders `.claude`, `.claude/rules`, `.claude/skills`, and `.claude/skills/trace-doctor` (in your editor, or with one `mkdir` command per folder).

2. Create `CLAUDE.md` in `lab-02`:

```markdown
# Analytics Service Standards

- Use current Claude model IDs from config, never hard-coded retired IDs.
- Load secrets from `.env` with python-dotenv; never print or commit keys.
- Handle `anthropic.APIStatusError` and report the status code and error type.
```

3. Create `.claude/rules/tests.md`. It loads only when Claude works on files whose names match `test_*.py`:

```markdown
---
paths:
  - "**/test_*.py"
---
# Testing Rules

- Put the comment `# Follows tests.md rule` on the first line of every test file.
- Mock every Claude API call; tests must never use the network.
- Include one test for invalid input in every test file.
```

4. Create `.claude/settings.json`, the shared project permissions:

```json
{
  "permissions": {
    "allow": ["Bash(python *)"],
    "deny": ["Read(//**/.env)", "Bash(curl *)"]
  }
}
```

5. Create `.claude/skills/trace-doctor/SKILL.md`:

```markdown
---
name: trace-doctor
description: Diagnoses Claude API failure traces and decides whether each failure came from the integration layer or the model / tool configuration layer.
argument-hint: <trace-file>
allowed-tools: Read, Grep, Glob
context: fork
---
Read the trace file $ARGUMENTS. For each trace, report:

1. The evidence in the trace that points to the cause.
2. The layer: "Integration layer" or "Model / tool config layer".
3. The concrete fix.

Keep each trace to three short lines.
```

6. Create two small code files that Claude will test. First `amounts.py`:

```python
# amounts.py - helper used by the analytics service
def parse_amount(text: str) -> float:
    """Parse a dollar amount such as '$1,250.50' into a float."""
    cleaned = text.replace("$", "").replace(",", "").strip()
    if not cleaned:
        raise ValueError("empty amount")
    return float(cleaned)
```

7. Then `test_amounts.py`:

```python
from amounts import parse_amount


def test_parse_amount_with_symbol_and_commas():
    assert parse_amount("$1,250.50") == 1250.50
```

8. Start Claude Code in `lab-02` and accept the prompt to trust the folder. Trusting it is what lets the project's allow rules apply:

```bash
claude
```

9. Run `/memory` and confirm that the project `CLAUDE.md` is loaded.

10. Prove that the path-scoped rule loads only for test files. Send:

```text
Add a test for invalid input to test_amounts.py.
```

11. Open `test_amounts.py` and confirm that the first line is now `# Follows tests.md rule`. That line came from the rule, which loaded only because Claude worked on a `test_*.py` file.

12. Prove that the deny rule works. Send:

```text
Show me the contents of the .env file in the parent folder.
```

13. Confirm that Claude reports that the read was denied by your settings.

> [!NOTE]
> The deny rule uses `//**/.env`, which is anchored at the filesystem root, so it blocks a `.env` file anywhere. A plain `Read(.env)` would block only `.env` files in this folder and its subfolders, not the one in the parent `ccdvf-labs` folder. Deny rules win over allow rules, and project settings are shared with the team through Git.

### Task 3: Submit a batch and diagnose traces

In this task, you submit a Message Batch first, diagnose four production traces while it processes, and then collect the batch results.

1. Open a **second** terminal, go to `ccdvf-labs/lab-02`, and activate the virtual environment (same commands as Task 1, step 1, without the `mkdir`). Keep Claude Code running in the first terminal.

2. In `lab-02`, create `submit_batch.py`:

```python
# submit_batch.py - submit three small requests as one Message Batch
import json

import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

docs = {
    "doc-0001": "Q3 revenue rose 14% on enterprise growth.",
    "doc-0002": "Support tickets fell 9% after the new onboarding flow.",
    "doc-0003": "Cloud costs grew 22% because of a logging misconfiguration.",
}
batch = client.messages.batches.create(requests=[
    {
        "custom_id": doc_id,
        "params": {
            "model": "claude-haiku-5-5",
            "max_tokens": 1000,
            "output_config": {"effort": "low"},
            "messages": [{"role": "user", "content": f"Give this a sentiment of positive, negative, or neutral, with a 5-word reason: {text}"}],
        },
    }
    for doc_id, text in docs.items()
])
print(f"Submitted batch {batch.id}, status={batch.processing_status}")
with open("batch_id.json", "w") as f:
    json.dump({"batch_id": batch.id}, f)
```

3. In the second terminal, submit the batch:

```bash
python submit_batch.py
```

4. Create `production-traces.json`:

```json
{
  "traces": [
    {
      "id": "TRC-101",
      "error": "JSONDecodeError: Unterminated string at char 412",
      "request": {"model": "claude-haiku-5-5", "max_tokens": 128},
      "response": {"stop_reason": "max_tokens", "text": "{\"summary\": \"Revenue grew 14% while margins..."}
    },
    {
      "id": "TRC-102",
      "error": "HTTP 400 invalid_request_error: tool_use ids were found without tool_result blocks immediately after",
      "request": {"roles": ["user", "assistant (tool_use toolu_01)", "user (text only)"]}
    },
    {
      "id": "TRC-103",
      "error": "Agent told the customer: 'You have no orders' during a database outage",
      "tool_log": "DB timeout after 5000 ms; wrapper caught the exception and returned tool_result content '[]' with no is_error flag"
    },
    {
      "id": "TRC-104",
      "error": "Agent called get_customer when the user asked 'Where is order #99281?'",
      "request": {"tools": [{"name": "get_customer", "description": "Gets data"}, {"name": "lookup_order", "description": "Gets data"}]}
    }
  ]
}
```

5. **Before using Claude**, create `task3-diagnosis.md` and fill in your own diagnosis for each trace: the evidence, the layer (integration or model / tool config), and the fix.

6. In Claude Code (first terminal), run the Skill:

```text
/trace-doctor production-traces.json
```

7. Compare the Skill's answers with yours. Where you disagree, decide who is right and why.

8. In the second terminal, create `get_batch.py`:

```python
# get_batch.py - check the batch and print results keyed by custom_id
import json

import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()
batch_id = json.load(open("batch_id.json"))["batch_id"]

batch = client.messages.batches.retrieve(batch_id)
print(f"Batch {batch_id}: {batch.processing_status} {batch.request_counts}")
if batch.processing_status == "ended":
    for result in client.messages.batches.results(batch_id):
        if result.result.type == "succeeded":
            text = "".join(b.text for b in result.result.message.content if b.type == "text")
            print(f"{result.custom_id} ({result.result.message.stop_reason}): {text.strip()}")
        else:
            print(f"{result.custom_id}: {result.result.type}")
else:
    print("Not finished yet. Run this script again in a minute.")
```

9. Run it. A small batch usually ends within a few minutes. If it has not ended yet, continue with the bonus and try again later:

```bash
python get_batch.py
```

10. Look at the order of the results. They rarely come back in the order you submitted them.

> [!NOTE]
> **Check your diagnosis:** TRC-101 is an integration bug: the output hit `max_tokens` mid-JSON, so raise `max_tokens` and use structured outputs. TRC-102 is an integration bug: the harness must send a `tool_result` for `toolu_01` in the very next user message. TRC-103 is an integration bug: the wrapper swallowed the timeout and returned an empty result instead of a `tool_result` with `"is_error": true`. TRC-104 is a model / tool configuration problem: two-word tool descriptions give Claude nothing to choose between, so describe each tool's purpose, input format, and when not to use it. Batch results can arrive in any order, which is why each request has a `custom_id`. Also note that `max_tokens` includes Claude's thinking: with a very small limit, a request can end with `stop_reason=max_tokens` and an empty answer.

### Bonus Task 4: Run the diagnosis headlessly

Diagnose the same traces in a CI-style run, with fewer instructions. Headless mode (`claude -p`) runs one prompt and exits, and `--json-schema` makes the result machine-readable.

1. Create `run_headless.py`, which calls `claude -p` as a subprocess. Passing arguments as a list avoids shell quoting problems on both macOS and Windows:

```python
# run_headless.py - run Claude Code non-interactively, the way a CI job would
import json
import subprocess

schema = {
    "type": "object",
    "properties": {
        "traces": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "id": {"type": "string"},
                    "layer": {"type": "string", "enum": ["integration", "model_or_tool_config"]},
                    "fix": {"type": "string"},
                },
                "required": ["id", "layer", "fix"],
            },
        }
    },
    "required": ["traces"],
}

result = subprocess.run(
    ["claude", "-p", "Diagnose every trace in production-traces.json.",
     "--output-format", "json", "--json-schema", json.dumps(schema)],
    capture_output=True, text=True,
)
print("exit code:", result.returncode)
envelope = json.loads(result.stdout)
print(json.dumps(envelope.get("structured_output", envelope), indent=2))
```

2. Run it and confirm that you get schema-valid JSON and the process exits without waiting for input. Check every classification against your `task3-diagnosis.md`: a confident, schema-valid answer can still be wrong.

3. Add a fifth trace of your own design to `production-traces.json`, one that could fool a reviewer into blaming the model. Run the headless diagnosis again and see whether it classifies your trace correctly.

4. **Stretch:** change the script so it exits with a non-zero code when any trace is classified as an integration bug, which is how you would fail a CI build.

## Congratulations!

In this lab, you have:
- Identified and fixed Messages API errors from real status codes and error types.
- Configured `CLAUDE.md`, path-scoped rules, permission rules, and a Skill in Claude Code.
- Processed requests with the Message Batches API and isolated integration bugs from model failures.
