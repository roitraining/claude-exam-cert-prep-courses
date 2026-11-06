<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 3: Tool Design and MCP Integration

---

# Chapter 3: Objectives

- Design clear tool descriptions and selection boundaries
- Implement structured MCP error responses
- Scope agent tool sets and configure `tool_choice`
- Integrate MCP servers and built-in tools effectively

---

# Domain 2: Tool Design and MCP Integration

| Task # | Task Statement |
| :--- | :--- |
| **2.1** | Design effective tool interfaces with clear descriptions and boundaries |
| **2.2** | Implement structured error responses for MCP tools |
| **2.3** | Distribute tools appropriately across agents and configure tool choice |
| **2.4** | Integrate MCP servers into Claude Code and agent workflows |
| **2.5** | Select and apply built-in tools (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`) effectively |

---

<!-- layout: navigation -->
# Chapter 3

- **Tool Interface Design**
- Structured Error Responses
- Tool Distribution and Built-In Tools
- MCP Server Integration

---

# Tool Descriptions Drive Tool Selection

- **Primary selection mechanism**: LLMs rely on tool names and descriptions to choose which tool to invoke
- **Root cause of misrouting**: when two tools have minimal or overlapping descriptions (such as `"Retrieves customer info"` vs `"Retrieves order details"`), Claude frequently picks the wrong tool on ambiguous user queries
- **First-step architectural fix**: expand tool descriptions with explicit input formats, example queries, edge cases, and boundary explanations before building external classifiers or consolidating backends
- **System prompt keyword traps**: audit system prompts for keyword-sensitive instructions that unintentionally bias Claude toward a specific tool regardless of its description

> [!IMPORTANT]
> Exam Rule: When an agent misroutes calls between two similar tools with minimal descriptions, the most effective **first step** is expanding the tool descriptions - not adding 8 few-shot examples or building a pre-routing classifier.

---

<!-- layout: 2-column -->
# Weak vs High-Reliability Tool Descriptions

### Weak Description (Causes Misrouting)
```json
{
  "name": "lookup_order",
  "description": "Retrieves order details",
  "input_schema": {
    "type": "object",
    "properties": {
      "id": { "type": "string" }
    }
  }
}
```
- No format guidance on `id`
- No boundary distinguishing order lookups from `get_customer`

### Strong Description (Clear Boundaries)
```json
{
  "name": "lookup_order",
  "description": "Fetches order status, line items, and shipment tracking using an order ID (format: ORD-XXXXX or #12345). Use when the user asks about a purchase or delivery. Do NOT use for account profile lookups; call get_customer first if only an email or name is provided.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string" }
    }
  }
}
```

---

<!-- layout: card-layout -->
# Eliminating Functional Overlap Across Tools

### Rename Ambiguous Tools
- Near-identical names like `analyze_content` and `analyze_document` cause frequent selection collisions.
- Rename `analyze_content` to `extract_web_results` with a web-specific description so the domain boundary is unmistakable.

### Split Overloaded Generic Tools
- A catch-all `analyze_document` tool with dozens of mode flags forces complex parameter guessing.
- Split it into purpose-specific tools with strict input/output contracts: `extract_data_points`, `summarize_content`, and `verify_claim_against_source`.

### Constrain Generic Network Tools
- Replace unconstrained tools like `fetch_url` with role-validated alternatives such as `load_document` that verify allowed document storage URLs.

---

<!-- layout: navigation -->
# Chapter 3

- Tool Interface Design
- **Structured Error Responses**
- Tool Distribution and Built-In Tools
- MCP Server Integration

---

<!-- layout: card-layout -->
# Four Error Categories in MCP Tools

### 1. Transient Errors
- Network timeouts, rate limits, or temporary `503 Service Unavailable` responses.
- Mark `isRetryable: true` so the subagent can retry locally with backoff before escalating.

### 2. Validation Errors
- Malformed identifier format or missing required query fields.
- Mark `isRetryable: true` with specific parameter feedback so the model reformats its call.

### 3. Business / Policy Errors
- Requests that violate policy, such as a return outside the 30-day window or a refund over limit.
- Mark `isRetryable: false` and include a customer-friendly explanation so the agent stops retrying and explains or escalates.

### 4. Permission Errors
- Unauthorized access to restricted records or missing credentials.
- Mark `isRetryable: false` and propagate immediately to the coordinator or human escalation flow.

---

# Returning Structured Error Metadata

- **Why generic errors fail**: returning a uniform `"Operation failed"` string prevents the agent from knowing whether to retry, reformulate inputs, or explain a policy limit
- **MCP `isError` pattern**: return `isError: true` alongside structured JSON metadata

```json
{
  "isError": true,
  "content": [
    {
      "type": "text",
      "text": "{\"errorCategory\": \"business_rule\", \"isRetryable\": false, \"code\": \"RETURN_WINDOW_EXPIRED\", \"userMessage\": \"Orders delivered more than 30 days ago require supervisor approval for return.\", \"attemptedOrderId\": \"ORD-99281\"}"
    }
  ]
}
```

---

# Demo: A Policy Error Versus a Transient Error

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_05_error_category.py`**

**Step 2. Paste this code**

```python
# demo_05_error_category.py - the same tool failure, two different error categories
import json
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

tools = [{
    "name": "process_refund",
    "description": "Issue a refund for a delivered order.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}, "amount": {"type": "number"}},
        "required": ["order_id", "amount"],
    },
}]

FAILURES = {
    "transient": {"errorCategory": "transient", "isRetryable": True,
                  "message": "Payment service timed out."},
    "business_rule": {"errorCategory": "business_rule", "isRetryable": False,
                      "code": "RETURN_WINDOW_EXPIRED",
                      "userMessage": "Orders delivered more than 30 days ago cannot be refunded online."},
}

def run(failure):
    messages = [{"role": "user", "content": "Please refund $45 on order ORD-2001."}]
    for _ in range(2):
        response = client.messages.create(
            model="claude-haiku-5-5",
            max_tokens=1028,
            output_config={"effort": "low"},
            system="You are a support agent. Read tool error metadata before you decide what to do next.",
            tools=tools,
            messages=messages,
        )
        calls = [b for b in response.content if b.type == "tool_use"]
        if not calls:
            break
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": [{
            "type": "tool_result", "tool_use_id": calls[0].id,
            "is_error": True, "content": json.dumps(failure),
        }]})
    if calls:
        print("  Claude called process_refund again (a retry).")
    else:
        text = "".join(b.text for b in response.content if b.type == "text")
        print("  Claude:", text)

for name, failure in FAILURES.items():
    print(f"Error category: {name}")
    run(failure)
    print()
```

**Step 3. Run it**

```bash
python demo_05_error_category.py
```

**Step 4. Examine the output**

- For `transient`, Claude should call `process_refund` again
- For `business_rule`, Claude should stop, explain the 30-day window to the customer, and offer an escalation instead of retrying

> [!NOTE]
> The tool failed both times; only the metadata changed. `errorCategory` and `isRetryable` are what let the agent choose between a retry and an explanation. A bare `"Operation failed"` string gives it nothing to choose with.

---

<!-- layout: 2-column -->
# Access Failures vs Valid Empty Results

### Access Failure (Error State)
- Database timeout, unreachable API, or expired auth token
- Must return `isError: true` with `errorCategory: "transient"` or `"permission"`
- Subagent attempts local retry for transient failures; if unresolved, propagates structured error context and partial results to the coordinator

### Valid Empty Result (Success State)
- Query executed cleanly against the backend, and zero records matched `"email: unknown@example.com"`
- Must return `isError: false` with an empty list `[]`
- Tells the agent definitively that no matching record exists so it can ask the user for another identifier

<!-- below-columns -->

> [!WARNING]
> Never catch a timeout inside a tool or subagent and return an empty array `[]` marked as successful. Suppressing access errors tricks the coordinator into reporting that no data exists.

---

<!-- layout: navigation -->
# Chapter 3

- Tool Interface Design
- Structured Error Responses
- **Tool Distribution and Built-In Tools**
- MCP Server Integration

---

<!-- layout: stacked -->
# Scoping Tool Sets Across Agents

- **Avoid tool bloat**: giving a single agent 18 tools degrades selection accuracy compared to 4 to 5 role-specific tools
- **Prevent cross-specialization misuse**: a synthesis agent given general web search tools will bypass coordination and run uncoordinated searches
- **Scoped cross-role exception**: if 85% of synthesis verifications are simple fact-checks (dates, names, stats), grant a constrained `verify_fact` tool while routing the 15% complex investigations back through the coordinator

![Scoped Tool Distribution](images/ch03-scoped-tool-distribution.svg)

---

<!-- layout: 3-column -->
# Configuring `tool_choice` Options

### `"auto"` (Default)
- Model decides whether to return conversational text or call one or more tools
- Use for open-ended customer support and interactive coding sessions

### `"any"`
- Forces the model to call **at least one** tool from the provided list rather than returning plain text
- Ideal when multiple extraction schemas exist and the model must pick the matching schema tool

### `{"type": "tool", "name": "..."}`
- Forces the model to call **one specific named tool** on the current turn
- Use to guarantee `extract_metadata` runs on turn 1 before enrichment tools run on follow-up turns

<!-- below-columns -->

> [!NOTE]
> On the exam, `"any"` and forced selection are correct answers. In current releases, they work on Claude Haiku 5.5 and Opus 5 but return a 400 error on Opus 5.5, Sonnet 5.5, Fable 5.1, and Mythos 5.1, and a forced call skips thinking. There is also a fourth option, `"none"`, which prevents tool calls.

---

<!-- layout: 2-column -->
# Selecting Built-In Search Tools: `Grep` vs `Glob`

### `Grep` (Content Search)
- Searches **inside file contents** using regular expressions
- Use to find all callers of a function, locate specific error message strings, or trace `import` statements across a codebase

### `Glob` (File Path Matching)
- Matches **file names and directory paths** by wildcard pattern
- Use to locate files by extension or naming convention, such as `**/*.test.tsx` or `src/**/schema.json`

<!-- below-columns -->

> [!TIP]
> Incremental exploration rule: start with `Grep` or `Glob` to locate entry points, then use `Read` to follow imports and trace flows - never read every file in a repository upfront.

---

# File Operations and Wrapper Tracing (`Read`, `Write`, `Edit`)

- **Targeted edits with `Edit`**: replaces a unique text anchor inside a file without rewriting the entire file
- **Fallback when `Edit` fails**: if `Edit` fails because the anchor text appears multiple times (non-unique match), fall back to **`Read`** (load full file contents) followed by **`Write`** (save updated file)
- **Tracing usage across wrapper modules**:
  1. Read the wrapper module first to identify all exported aliases and re-exported function names
  2. Run `Grep` across the codebase for each exported name to find every call site accurately

---

<!-- layout: navigation -->
# Chapter 3

- Tool Interface Design
- Structured Error Responses
- Tool Distribution and Built-In Tools
- **MCP Server Integration**

---

<!-- layout: 2-column -->
# Project-Scoped vs User-Scoped MCP Configuration

### Project-Scoped (`.mcp.json`)
- Checked into version control at the repository root so the entire team shares the same MCP servers
- Uses environment variable expansion (such as `"${GITHUB_TOKEN}"`) to inject secrets at runtime without committing credentials
- Best for standard team integrations (Jira, GitHub, internal APIs)

### User-Scoped (`~/.claude.json`)
- Stored in the individual developer’s home directory outside the repository
- Best for personal productivity tools or experimental MCP servers being tested by one engineer
- All configured project and user MCP servers connect simultaneously at session start

---

# `.mcp.json` with Environment Variable Expansion

- **Never hard-code tokens in `.mcp.json`**: use `${VAR_NAME}` expansion so teams share configuration safely via Git

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    }
  }
}
```

---

<!-- layout: card-layout -->
# MCP Resources and Adoption Best Practices

### Expose Catalogs via MCP Resources
- Use **MCP resources** to expose read-only content catalogs such as issue summaries, documentation trees, and database schemas.
- Gives agents immediate visibility into available data without burning turns on exploratory tool calls.

### Prevent Built-In Tool Fallback
- If an MCP tool has a vague description, Claude Code often prefers familiar built-in tools like `Grep` or `Bash`.
- Write detailed MCP tool descriptions explaining exact capabilities and rich outputs so the agent chooses the MCP tool over `Grep`.

### Community vs Custom MCP Servers
- Choose vetted community or official MCP servers for standard platforms like Jira, GitHub, or Postgres.
- Reserve custom MCP server development for proprietary internal workflows and domain-specific business logic.

---

# Demo: An MCP Server with a Tool and a Resource

In `ccarf-labs`, with the virtual environment still active. This demo runs Claude Code, so it uses your Claude subscription.

**Step 1. Create `runbook_server.py` and paste this code**

```python
# runbook_server.py - one resource (the catalog) and one tool (fetch a runbook)
from mcp.server.mcpserver import MCPServer

mcp = MCPServer("runbooks")

RUNBOOKS = {
    "db-failover": "1. Page the on-call DBA. 2. Promote the read replica. 3. Update the app connection string.",
    "cache-flush": "1. Announce in #ops. 2. Run the flush job. 3. Watch the hit rate for 10 minutes.",
    "cert-renewal": "1. Request the new certificate. 2. Upload it to the load balancer. 3. Verify the expiry date.",
}

@mcp.resource("runbooks://catalog")
def catalog() -> str:
    """The names of all available runbooks, one per line."""
    return "\n".join(RUNBOOKS)

@mcp.tool()
def get_runbook(name: str) -> str:
    """Return the steps for one runbook. Use a name from the runbooks://catalog resource."""
    return RUNBOOKS.get(name, f"No runbook named {name}.")

if __name__ == "__main__":
    mcp.run()
```

**Step 2. Create `.mcp.json` in the same folder**

```json
{
  "mcpServers": {
    "runbooks": {
      "command": "python",
      "args": ["runbook_server.py"]
    }
  }
}
```

**Step 3. Start Claude Code from the same terminal**

```bash
claude
```

When Claude Code asks whether to use the MCP server in `.mcp.json`, approve `runbooks`.

**Step 4. Run `/mcp` and confirm that `runbooks` is connected, then press Esc**

**Step 5. Ask Claude**

```text
Using the runbooks MCP server, which runbooks exist, and what are the steps for db-failover? Say which MCP tools or resources you used.
```

**Step 6. Examine the output**

- Claude reads the `runbooks://catalog` resource to list the three runbook names
- Claude calls the `get_runbook` tool (shown as `mcp__runbooks__get_runbook`) for the db-failover steps
- Type `/exit` when you are done

> [!NOTE]
> The catalog is a **resource**: read-only content the agent can see without guessing. Fetching one runbook is a **tool**. Because `.mcp.json` lives in the project, everyone who clones the repository gets the same server.

---

# Domain 2 Quiz: Tool Design and MCP Integration

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-14-46-ccar-f-realistic-instructor.json)

---

# What You Learned

- Designed clear tool descriptions and selection boundaries
- Implemented structured MCP error responses
- Scoped agent tool sets and configured `tool_choice`
- Integrated MCP servers and built-in tools effectively

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
