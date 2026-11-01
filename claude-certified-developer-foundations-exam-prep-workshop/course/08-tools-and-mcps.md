<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 8: Tools and MCPs

---

# Chapter 8: Objectives

- Implement custom tools with clear schemas and error handling
- Build MCP servers exposing Tools, Resources, and Prompts
- Select between Built-in Tools, Custom Tools, Skills, and MCPs

---

# Domain 8: Tools and MCPs

| Task # | Task Statement |
| :--- | :--- |
| **8.1** | Tool Implementation: Tool use and function calling, external system configuration, tool description writing, error handling, client-side vs server-side tools, and approval patterns |
| **8.2** | MCP Server Development: Server authoring, deployment, application integration, MCP resources, tools, and prompts, and communication patterns (`stdio`, sockets) |
| **8.3** | Agentic Customization: Trade-offs among built-in Tools, custom Tools, Skills, and MCPs for selecting the appropriate approach |

---

<!-- layout: navigation -->
# Chapter 8

- **Tool Implementation**
- MCP Server Development
- Agentic Customization

---

<!-- layout: card-layout -->
# Tool Implementation Best Practices

### 1. Clear Descriptions and Boundaries
- Write descriptions that specify exact purpose, expected parameter formats (such as `ORD-XXXXX`), example queries, edge cases, and when **not** to use the tool relative to similar tools.

### 2. Focused Tool Sets
- Keep per-agent tool sets small and role-relevant to maximize selection accuracy; for very large catalogs, mark tools `defer_loading: true` and add the tool search tool so Claude loads only what it needs.

### 3. Structured Error Handling
- Return failures as results, not silence: `is_error: true` on a Messages API `tool_result` (MCP uses `isError: true`), with a message that says what went wrong and whether a retry can help (a common convention: `transient`, `validation`, `permission` categories).

### 4. Approval Patterns for Sensitive Actions
- Require explicit human approval or deterministic `PreToolUse` verification before executing destructive or state-changing tools.

---

<!-- layout: 3-column -->
# Client-Side vs Server-Side Tools

### Client Tools (You Define)
- Declared in `tools` with `name`, `description`, and `input_schema` (add `strict: true` for guaranteed schema-valid input)
- Claude returns `stop_reason: "tool_use"`; **your harness** runs the function and returns a `tool_result`
- The only option that reaches private systems directly

### Anthropic-Defined Client Tools
- Schema supplied by Anthropic, executed by you: `bash`, `text_editor`, `memory`, computer use
- Declare by `type` (for example, `{"type": "bash_20250124", "name": "bash"}`) with no `input_schema`
- You still own sandboxing and approval

### Server Tools (Anthropic Executes)
- `web_search`, `web_fetch`, `code_execution`, tool search: results arrive in the same response, no round trip
- Long server loops can stop with `stop_reason: "pause_turn"`: resend to resume
- Cannot reach your private VPC services

---

<!-- layout: navigation -->
# Chapter 8

- Tool Implementation
- **MCP Server Development**
- Agentic Customization

---

<!-- layout: stacked -->
# Model Context Protocol (MCP) Architecture and Primitives

- **Standardized integration protocol**: an MCP Host (Claude Code, Claude Desktop, or your custom agent app) uses MCP Clients to maintain 1:1 stateful connections with MCP Servers
- **Three core server primitives**:
  - **Tools (Model-controlled)**: executable functions (`check_stock`, `create_ticket`) that Claude chooses to invoke
  - **Resources (App-controlled)**: read-only URI-addressed data catalogs (`db://inventory/schema`, documentation trees) that reduce exploratory tool calls
  - **Prompts (User-controlled)**: parameterized workflow templates exposed to users as slash commands

![MCP Architecture and Primitives](images/ch08-mcp-architecture-primitives.svg)

---

<!-- layout: 2-column -->
# MCP Transports (`stdio` vs Streamable HTTP) and Integration

### `stdio` Transport (Local Process)
- Host spawns the server as a child process; JSON-RPC over `stdin` / `stdout`
- Ideal for local tooling in Claude Code or Claude Desktop
- **Rule**: never log to `stdout` (it corrupts JSON-RPC frames); use `stderr`
- Claude Code scopes: **local**, **project** (`.mcp.json`, shared), **user**

### Streamable HTTP Transport (Remote Service)
- Current remote transport (HTTP+SSE is deprecated): one service shared by many apps, with OAuth and independent deployment
- Connect via `claude mcp add --transport http`, claude.ai connectors, or the Messages API **MCP connector** (`mcp_servers` + `mcp_toolset`)

---

# Authoring an MCP Server with Tools and Resources

- **Exposing a reusable internal inventory service** via the Python MCP SDK (`FastMCP`):

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("inventory-service")

@mcp.resource("inventory://catalog/schema")
def get_inventory_schema() -> str:
    """Returns read-only warehouse table schema and valid warehouse codes."""
    return "warehouses: [US-EAST, US-WEST, EU-CENTRAL] | SKU format: SKU-NNNNN"

@mcp.tool()
def check_sku_stock(sku: str, warehouse: str) -> dict:
    """Real-time stock for a SKU (format: SKU-NNNNN) in one warehouse."""
    return query_internal_inventory_api(sku=sku, warehouse=warehouse)
```

---

<!-- layout: navigation -->
# Chapter 8

- Tool Implementation
- MCP Server Development
- **Agentic Customization**

---

<!-- layout: card-layout -->
# Choosing the Right Agentic Customization Approach

### Built-In Tools (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`)
- Use for local file inspection, targeted code edits, pattern searching, and local shell execution inside Claude Code or the Agent SDK.
- Built-in tools **cannot** automatically reach arbitrary internal REST APIs without custom wiring.

### Custom In-Process Tools
- Use when a single application needs a lightweight function wrapper around an internal service and no other apps need to share it.

### Claude Skills (`SKILL.md`)
- Use to package reusable know-how (instructions, checklists, scripts) that loads on demand: only the name and description sit in context until the Skill is needed.
- Portable across Claude Code, the Agent SDK, claude.ai, and the API (with code execution). Skills teach *how*; they do not connect to live systems.

### MCP Servers
- Use when an external system integration (such as an internal inventory REST API or Jira/Postgres) must be **reusable across several Claude applications** and **maintained independently** of any single app.

---

# Agentic Customization Decision Matrix

| Requirement Scenario | Best Customization Choice | Why Other Options Fall Short |
| :--- | :--- | :--- |
| **Share an internal inventory REST API across 4 Claude apps** | **MCP Server** exposing inventory tools | Hard-coding into prompts is unmaintainable; pasting data wastes context; built-in tools lack internal API logic |
| **Search a local repo for all callers of `calculateTax()`** | **Built-in `Grep` tool** | `Glob` matches filenames only; building a custom MCP server for local file search is unnecessary overhead |
| **Run a verbose codebase security checklist on demand** | **Claude Skill (`.claude/skills/`)** with `context: fork` | Putting verbose task checklists in root `CLAUDE.md` bloats every unrelated session |
| **Single microservice needs one helper to format currency** | **Custom in-process function tool** | Full standalone MCP server adds deployment overhead when only one app uses the helper |
| **Teach every team the company’s report format and review steps** | **Claude Skill** (shared, or distributed in a plugin) | An MCP server adds a live connection that this static know-how does not need |

---

# Domain 8 Quiz: Tools and MCPs

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-17-ccdv-f-realistic-instructor.json)

---

# Lab 4: Developing Secure Tools and MCP Servers

**Time:** 60 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-developer-foundations-exam-prep-workshop/labs/lab-04-developing-secure-tools-and-mcp-servers)

---

<!-- layout: 2-column -->
# What You Learned

### Tool Implementation and MCP
- Implemented custom tools with clear schemas and error handling
- Built MCP servers exposing Tools, Resources, and Prompts

### Agentic Customization Selection
- Selected between Built-in Tools, Custom Tools, Skills, and MCPs
- Matched reusable multi-app integrations to MCP servers

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
