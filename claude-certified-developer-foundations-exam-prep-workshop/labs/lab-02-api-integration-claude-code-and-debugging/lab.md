# API Integration, Claude Code, and Debugging

## Time Required

45 minutes

## Overview

In this lab, you will build a multi-turn tool request and stream the response with the Messages API, configure Claude Code so a whole team shares the same rules and guardrails, and diagnose production failure traces while processing bulk work with the Message Batches API.

### You learn how to:
- Build a multi-turn tool request and stream the final response with the Messages API.
- Configure `CLAUDE.md`, path-scoped rules, permission rules, and a forked Skill in Claude Code.
- Diagnose production failure traces and process bulk requests with the Message Batches API.

## Scenario

Your team ships a document analytics service. You need a clean pattern for calling a financial metrics tool and streaming the answer, shared repository configuration so every engineer runs Claude Code with the same rules and permissions, and a fast way to separate integration-layer bugs from model or tool configuration issues while non-urgent batch jobs run asynchronously.

![Lab 2 architecture: Messages API tool call and streaming, Claude Code configuration, traces, and batches](images/lab02-api-claude-code-and-traces.svg)

## Lab Instructions

### Task 1: Build a tool request and stream the response

In this task, you send a Messages API request with a tool definition, inspect the `tool_use` block that Claude returns, send back the matching `tool_result` block, and stream the final response while reading token usage.

1. Open a terminal, go to your workspace, and activate the virtual environment.

**macOS:**

```bash
cd ~/ccdvf-labs
source .venv/bin/activate
mkdir -p lab-02
cd lab-02
```

**Windows (PowerShell):**

```powershell
cd $HOME\ccdvf-labs
.venv\Scripts\Activate.ps1
mkdir lab-02 -Force
cd lab-02
```

2. Open `lab-02` in your editor and create `tool_request.py`:

```python
# tool_request.py - call a tool on the Messages API and stream the final answer
import json

import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()
MODEL = "claude-haiku-5-5"
SYSTEM_PROMPT = "You are a financial analytics assistant. Answer concisely with exact numbers."

TOOLS = [{
    "name": "fetch_metrics",
    "description": (
        "Fetch revenue metrics and next-quarter forecast for a fiscal quarter. "
        "Pass quarter as 'Q1', 'Q2', 'Q3', or 'Q4'."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "quarter": {"type": "string", "description": "Fiscal quarter, for example 'Q3'"},
        },
        "required": ["quarter"],
    },
}]

METRICS_DB = {
    "Q3": {"q3_revenue_musd": 412, "yoy_growth_pct": 14, "q4_forecast_musd": 440},
}


def run_fetch_metrics(quarter: str) -> str:
    data = METRICS_DB.get(quarter.upper(), {"error": f"No data for {quarter}"})
    return json.dumps(data)


if __name__ == "__main__":
    messages = [{"role": "user", "content": "Summarize Q3 revenue and include the Q4 forecast."}]

    # Turn 1: send the question with the top-level system prompt and tools list
    first_resp = client.messages.create(
        model=MODEL,
        max_tokens=1000,
        system=SYSTEM_PROMPT,
        tools=TOOLS,
        messages=messages,
    )
    print(f"Turn 1: stop_reason={first_resp.stop_reason}")

    tool_use = next(b for b in first_resp.content if b.type == "tool_use")
    print(f"  Tool call: id={tool_use.id} name={tool_use.name} input={tool_use.input}")

    # Execute the tool locally and pair its output with tool_use.id
    tool_output = run_fetch_metrics(tool_use.input["quarter"])
    print(f"  Tool output: {tool_output}\n")

    # Turn 2: append the full assistant turn, then send tool_result first in the user message
    messages.append({"role": "assistant", "content": first_resp.content})
    messages.append({
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "tool_use_id": tool_use.id,
                "content": tool_output,
            },
        ],
    })

    print("Turn 2 (streaming):")
    with client.messages.stream(
        model=MODEL,
        max_tokens=1000,
        system=SYSTEM_PROMPT,
        tools=TOOLS,
        messages=messages,
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
        final = stream.get_final_message()

    print(
        f"\n\nFinal stop_reason={final.stop_reason} "
        f"input_tokens={final.usage.input_tokens} output_tokens={final.usage.output_tokens}"
    )
```

3. Run the script:

```bash
python tool_request.py
```

4. Inspect the output from both turns:
   - In **Turn 1**, Claude stops with `stop_reason=tool_use` and emits a `tool_use` block containing a unique `id`, the tool `name` (`fetch_metrics`), and parsed JSON `input` (`{'quarter': 'Q3'}`).
   - In **Turn 2**, your code appends the assistant turn (`first_resp.content`) and sends a user message whose `content` list starts with a `tool_result` block referencing `tool_use.id`.
   - `client.messages.stream(...)` prints the answer incrementally as tokens arrive, and `stream.get_final_message()` returns the completed `Message` object with `stop_reason=end_turn` and token counts in `final.usage`.

> [!NOTE]
> **Key Messages API rules to remember for the exam:**
> - Pass main system instructions in the top-level `system` parameter, never as `messages[0]`.
> - The API is stateless: you must append the assistant's `tool_use` turn to `messages` before sending the `tool_result` turn.
> - When an assistant turn contains one or more `tool_use` blocks, the very next user message must start with the matching `tool_result` blocks (`tool_use_id`) before any trailing text blocks.

### Task 2: Configure Claude Code for the team

In this task, you run a setup script that generates a complete project configuration for Claude Code (`CLAUDE.md`, a path-scoped rule in `.claude/rules/`, shared permissions in `.claude/settings.json`, and a forked Skill in `.claude/skills/`), along with sample files for Tasks 2 and 3. You then verify how each configuration layer behaves in an interactive session.

1. In `lab-02`, create `setup_lab.py` to scaffold the team configuration, helper code, and trace data in one step:

```python
# setup_lab.py - scaffolds Claude Code team config, sample code, and production traces
import json
from pathlib import Path

Path(".claude/rules").mkdir(parents=True, exist_ok=True)
Path(".claude/skills/trace-doctor").mkdir(parents=True, exist_ok=True)

Path("CLAUDE.md").write_text(
    "# Analytics Service Standards\n\n"
    "- Use current Claude model IDs from config, never hard-coded retired IDs.\n"
    "- Load secrets from `.env` with python-dotenv; never print or commit keys.\n"
    "- Handle `anthropic.APIStatusError` and report the status code and error type.\n"
)

Path(".claude/rules/tests.md").write_text(
    "---\n"
    "paths:\n"
    '  - "**/test_*.py"\n'
    "---\n"
    "# Testing Rules\n\n"
    "- Put the comment `# Follows tests.md rule` on the first line of every test file.\n"
    "- Mock every Claude API call; tests must never use the network.\n"
    "- Include one test for invalid input in every test file.\n"
)

Path(".claude/settings.json").write_text(json.dumps({
    "permissions": {
        "allow": ["Bash(python *)"],
        "deny": ["Read(//**/.env)", "Bash(curl *)"],
    }
}, indent=2) + "\n")

Path(".claude/skills/trace-doctor/SKILL.md").write_text(
    "---\n"
    "name: trace-doctor\n"
    "description: Diagnoses Claude API failure traces and decides whether each failure came from the integration layer or the model / tool configuration layer.\n"
    "argument-hint: <trace-file>\n"
    "allowed-tools: Read, Grep, Glob\n"
    "context: fork\n"
    "---\n"
    "Read the trace file $ARGUMENTS. For each trace, report:\n\n"
    "1. The evidence in the trace that points to the cause.\n"
    '2. The layer: "Integration layer" or "Model / tool config layer".\n'
    "3. The concrete fix.\n\n"
    "Keep each trace to three short lines.\n"
)

Path("amounts.py").write_text(
    "# amounts.py - helper used by the analytics service\n"
    "def parse_amount(text: str) -> float:\n"
    '    """Parse a dollar amount such as \'$1,250.50\' into a float."""\n'
    '    cleaned = text.replace("$", "").replace(",", "").strip()\n'
    "    if not cleaned:\n"
    '        raise ValueError("empty amount")\n'
    "    return float(cleaned)\n"
)

Path("test_amounts.py").write_text(
    "from amounts import parse_amount\n\n\n"
    "def test_parse_amount_with_symbol_and_commas():\n"
    '    assert parse_amount("$1,250.50") == 1250.50\n'
)

Path("production-traces.json").write_text(json.dumps({
    "traces": [
        {
            "id": "TRC-101",
            "error": "JSONDecodeError: Unterminated string at char 412",
            "request": {"model": "claude-haiku-5-5", "max_tokens": 128},
            "response": {"stop_reason": "max_tokens", "text": '{"summary": "Revenue grew 14% while margins...'},
        },
        {
            "id": "TRC-102",
            "error": "HTTP 400 invalid_request_error: tool_use ids were found without tool_result blocks immediately after",
            "request": {"roles": ["user", "assistant (tool_use toolu_01)", "user (text only)"]},
        },
        {
            "id": "TRC-103",
            "error": "Agent told the customer: 'You have no orders' during a database outage",
            "tool_log": "DB timeout after 5000 ms; wrapper caught the exception and returned tool_result content '[]' with no is_error flag",
        },
        {
            "id": "TRC-104",
            "error": "Agent called get_customer when the user asked 'Where is order #99281?'",
            "request": {"tools": [{"name": "get_customer", "description": "Gets data"}, {"name": "lookup_order", "description": "Gets data"}]},
        },
    ]
}, indent=2) + "\n")

print("Created CLAUDE.md, .claude/ config, amounts.py, test_amounts.py, and production-traces.json")
```

2. Run the setup script:

```bash
python setup_lab.py
```

3. Briefly review the four configuration files in your editor so you can recognize each pattern:
   - `CLAUDE.md` sets repository-wide coding standards loaded at session start.
   - `.claude/rules/tests.md` uses YAML frontmatter (`paths: ["**/test_*.py"]`) so the rule activates only when Claude reads or edits matching test files.
   - `.claude/settings.json` defines shared `allow` and `deny` permission rules committed to Git.
   - `.claude/skills/trace-doctor/SKILL.md` defines an on-demand Skill with `context: fork` (runs in an isolated subagent context) and `allowed-tools: Read, Grep, Glob`.

4. Make sure your terminal is inside `lab-02` (`pwd` should end with `/ccdvf-labs/lab-02`), then start Claude Code and accept the prompt to trust the folder. Claude Code loads `.claude/skills/` and `.claude/settings.json` from the directory where you launch `claude`:

```bash
cd ~/ccdvf-labs/lab-02
claude
```

5. Run `/memory` and confirm that `CLAUDE.md` is loaded. Press Esc to close the view.

6. Test the path-scoped rule by asking Claude to update `test_amounts.py`:

```text
Add a test for invalid input to test_amounts.py.
```

7. Open `test_amounts.py` in your editor and confirm that the first line is now `# Follows tests.md rule`. That marker came from `.claude/rules/tests.md`, which loaded automatically when Claude worked on a `test_*.py` file.

8. Test the permission deny rule by asking Claude to call the `Read` tool on the `.env` file in the parent directory:

```text
Use the Read tool to read ../.env. If permission is denied, stop and report the denial without using Bash.
```

9. Confirm that Claude reports the `Read` call was blocked by your permission settings. Keep Claude Code open inside `lab-02` for Task 3.

> [!NOTE]
> The deny rule `Read(//**/.env)` starts with `//`, which anchors the glob at the filesystem root so it blocks `.env` files anywhere on disk, including `../.env`. A relative pattern like `Read(.env)` only matches the current project folder. Deny rules always take precedence over allow rules.

### Task 3: Diagnose production traces and run a Message Batch

In this task, you run your `/trace-doctor` Skill on the four production failure traces in `production-traces.json`, verify why each failure belongs to the integration layer or the model and tool configuration layer, and then submit an asynchronous Message Batch.

1. Open `production-traces.json` in your editor and scan the four incidents (`TRC-101` through `TRC-104`). Notice what each trace records: the HTTP request parameters, `stop_reason`, tool logs, or tool definitions.

2. In your running Claude Code session, invoke the Skill you created in Task 2:

```text
/trace-doctor production-traces.json
```

3. Review the Skill's three-line report for each trace and verify that it matches the root causes below:

| Trace | Root Cause Layer | Evidence in the Trace | Concrete Fix |
| :--- | :--- | :--- | :--- |
| **TRC-101** | Integration layer | `stop_reason` is `"max_tokens"` (`max_tokens: 128`), so the JSON string was cut off mid-output. | Increase `max_tokens` and check `stop_reason` before parsing JSON (or use structured outputs). |
| **TRC-102** | Integration layer | The assistant `tool_use` turn (`toolu_01`) was followed by a plain text user message with no `tool_result`. | Send a `tool_result` block with `tool_use_id: "toolu_01"` at the start of the next user message. |
| **TRC-103** | Integration layer | The tool wrapper caught a DB timeout and returned `"[]"` instead of flagging an error. | Return a `tool_result` with `"is_error": true` and the timeout message so Claude knows the query failed. |
| **TRC-104** | Model / tool config layer | Both tools have the two-word description `"Gets data"`, giving Claude no way to tell them apart. | Expand each tool description with its purpose, expected input format (such as order ID vs customer ID), and boundaries. |

4. Exit Claude Code so you can run your batch script from the same terminal:

```text
/exit
```

5. Create `batch_job.py`, which submits a three-document Message Batch or checks an existing batch when passed `--check`:

```python
# batch_job.py - submit or check a Message Batch using custom_id correlation
import json
import sys

import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()
BATCH_FILE = "batch_id.json"

DOCS = {
    "doc-0001": "Q3 revenue rose 14% on enterprise growth.",
    "doc-0002": "Support tickets fell 9% after the new onboarding flow.",
    "doc-0003": "Cloud costs grew 22% because of a logging misconfiguration.",
}


def submit_batch() -> None:
    batch = client.messages.batches.create(requests=[
        {
            "custom_id": doc_id,
            "params": {
                "model": "claude-haiku-5-5",
                "max_tokens": 256,
                "messages": [{
                    "role": "user",
                    "content": f"Classify sentiment as positive, negative, or neutral with a 5-word reason: {text}",
                }],
            },
        }
        for doc_id, text in DOCS.items()
    ])
    with open(BATCH_FILE, "w") as f:
        json.dump({"batch_id": batch.id}, f)
    print(f"Submitted batch {batch.id} (status={batch.processing_status})")
    print("Run 'python batch_job.py --check' to inspect status and results.")


def check_batch() -> None:
    batch_id = json.load(open(BATCH_FILE))["batch_id"]
    batch = client.messages.batches.retrieve(batch_id)
    print(f"Batch {batch_id}: status={batch.processing_status} counts={batch.request_counts}")
    if batch.processing_status != "ended":
        print("Batch is still processing. Re-run 'python batch_job.py --check' in a minute.")
        return

    for item in client.messages.batches.results(batch_id):
        if item.result.type == "succeeded":
            text = "".join(b.text for b in item.result.message.content if b.type == "text")
            print(f"  {item.custom_id} ({item.result.message.stop_reason}): {text.strip()}")
        else:
            print(f"  {item.custom_id}: {item.result.type}")


if __name__ == "__main__":
    if "--check" in sys.argv:
        check_batch()
    else:
        submit_batch()
```

6. Submit the batch and then check its status:

```bash
python batch_job.py
python batch_job.py --check
```

7. If the batch is still `in_progress`, continue to the Bonus Task and run `python batch_job.py --check` again after a minute or two. When it finishes (`status=ended`), notice that each result is matched back to its source document using `item.custom_id` because batch responses are not guaranteed to return in submission order.

### Bonus Task 4: Run the trace diagnosis headlessly

In this bonus task, you run the same trace analysis non-interactively with `claude -p` and `--json-schema`, the pattern used in CI/CD pipelines.

1. Create `run_headless.py`, which invokes `claude -p` as a subprocess and requests schema-validated JSON output:

```python
# run_headless.py - run Claude Code non-interactively with a strict JSON Schema
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
    [
        "claude", "-p", "Diagnose every trace in production-traces.json.",
        "--output-format", "json",
        "--json-schema", json.dumps(schema),
    ],
    capture_output=True,
    text=True,
)
print("exit code:", result.returncode)
envelope = json.loads(result.stdout)
print(json.dumps(envelope.get("structured_output", envelope), indent=2))
```

2. Run the headless script and verify that it prints schema-valid JSON and exits without waiting for interactive input:

```bash
python run_headless.py
```

3. **Stretch:** update `run_headless.py` so it exits with `sys.exit(1)` whenever any trace is classified as `"integration"`, which is how a CI check can fail a build automatically.

## Congratulations!

In this lab, you have:
- Built a multi-turn tool request and streamed the final response with the Messages API.
- Configured `CLAUDE.md`, path-scoped rules, permission rules, and a forked Skill in Claude Code.
- Diagnosed production failure traces and processed bulk requests with the Message Batches API.
