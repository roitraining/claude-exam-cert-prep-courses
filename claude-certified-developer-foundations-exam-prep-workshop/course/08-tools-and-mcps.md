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
- Keep per-agent tool sets small (4 to 5 role-relevant tools) to maximize selection accuracy and prevent capability bloat.

### 3. Structured Error Handling (`isError`)
- Return `isError: true` with `errorCategory` (`transient`, `validation`, `business_rule`, `permission`) and `isRetryable` booleans so the agent can recover intelligently.

### 4. Approval Patterns for Sensitive Actions
- Require explicit human approval or deterministic `PreToolUse` verification before executing destructive or state-changing tools.

---

<!-- layout: 2-column -->
# Client-Side vs Server-Side Tools

### Client-Side Tools (Harness Dispatched)
- Defined in the `tools` array with a JSON `input_schema`
- Claude returns `stop_reason: "tool_use"` with the tool `name` and `input` arguments
- **Your application harness** executes the function (calling your internal DB or API) and sends the result back in a `tool_result` block on the next turn

### Server-Side / Managed Tools
- Executed directly within the managed Anthropic environment (such as managed web search or sandboxed code execution) without requiring a client round-trip for execution
- Cannot directly reach private internal services inside your corporate VPC unless exposed via a secure tunnel or remote MCP server

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
# MCP Transports (`stdio` vs Sockets / HTTP) and Authoring

### `stdio` Transport (Local Process)
- MCP Host spawns the MCP Server as a local child process and communicates over standard input/output (`stdin`/`stdout`)
- Ideal for local developer tooling in Claude Code or Claude Desktop
- **Important rule**: never write debug logs to `stdout` in a `stdio` MCP server (it corrupts JSON-RPC frames); log to `stderr` instead

### HTTP / SSE / Socket Transport (Remote Service)
- MCP Server runs as a standalone network service shared across multiple applications and teams
- Supports centralized authentication, container scaling, and independent deployment life cycles

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
    """Checks real-time stock quantity for a SKU (format: SKU-NNNNN) in a specific warehouse."""
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
- Use to package reusable domain instructions, checklists, and tool restrictions (`context: fork`, `allowed-tools`) that load on demand inside Claude Code.

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

---

# Domain 8 Quiz: Tools and MCPs

- Open the Exam Prep Program to complete the Domain 8 review questions

<!-- TODO: Add Exam Prep Program URL for Domain 8 Quiz -->

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
