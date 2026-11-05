<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 4: Claude Code, Testing, and Debugging

---

# Chapter 4: Objectives

- Author `CLAUDE.md` files, path-scoped `.claude/rules/`, `SKILL.md` workflows, and custom subagents
- Enforce tool permissions and settings precedence in `.claude/settings.json`
- Operate interactive, Plan, headless (`claude -p`), streaming, and `auto` modes
- Handle typed SDK exceptions, rate limits, and `stop_reason` edge cases
- Isolate integration-layer bugs from model-output failures using execution traces and evals

---

# Domain 3: Claude Code and Domain 4: Eval, Testing, and Debugging

| Task # | Task Statement |
| :--- | :--- |
| **3.1** | Claude Code Operation: Core components (Rules, Skills, Commands, Agents, Agent Memory), session management, headless/streaming/auto modes, `CLAUDE.md` hierarchy, and `settings.json` |
| **4.1** | Debugging and Error Handling: Error type identification, recovery strategy selection, trace analysis, and problem origin isolation between the integration layer and model output |

---

<!-- layout: navigation -->
# Chapter 4

- **Core Components and Configuration**
- Session Management and Execution Modes
- Error Handling and Recovery
- Trace Analysis and Problem Isolation

---

# Five Core Components of Claude Code

| Component | Where It Lives | What It Does |
| :--- | :--- | :--- |
| **Rules** | `CLAUDE.md` files and `.claude/rules/*.md` | Persistent instructions; rule files can use `paths:` globs to load only for matching files |
| **Skills** | `.claude/skills/<name>/SKILL.md` | On-demand workflows: only the name and description load until the Skill is invoked (`context: fork`, `allowed-tools`, `argument-hint`) |
| **Commands** | Built-in (`/init`, `/compact`, `/clear`) and custom | Custom slash commands are now Skills you invoke by name; legacy `.claude/commands/*.md` files still work |
| **Agents** | `.claude/agents/<name>.md` | Subagents with their own prompt, tools, model, and context window, spawned through the `Agent` tool (formerly `Task`) |
| **Agent Memory** | Auto memory and subagent `memory:` | Notes Claude writes and reloads across sessions (see next slide) |

---

<!-- layout: 2-column -->
# The `CLAUDE.md` Hierarchy and Agent Memory

### `CLAUDE.md` Hierarchy (You Write It)
- **Managed policy** (organization-wide, set by IT) -> **user** (`~/.claude/CLAUDE.md`) -> **project** (`./CLAUDE.md` or `.claude/CLAUDE.md`, committed) -> **subdirectory** files loaded when Claude works there
- Use `@path/to/file` imports to pull in shared docs without copying them
- Start a repository with `/init`; inspect what loaded with `/memory`
- Team rules belong in the project file, not in `~/.claude/CLAUDE.md`

### Agent Memory (Claude Writes It)
- **Auto memory**: Claude saves learnings to a per-project `MEMORY.md` index plus topic files and reloads them next session
- **Subagent memory**: `memory: user | project | local` in an agent’s frontmatter gives it a persistent directory (for example, `.claude/agent-memory/<agent-name>/`)
- Review memory like code: stale or wrong memories steer future sessions

---

<!-- layout: 2-column -->
# Authoring `CLAUDE.md` and Path-Scoped `.claude/rules/`

### Project Root `CLAUDE.md`
- Keep root `CLAUDE.md` concise: build/test commands, repo layout, and universal conventions
- Pull in shared reference docs with `@docs/architecture.md` instead of pasting thousands of lines

```markdown
# Project Conventions
- Run tests: `pytest -q tests/`
- Typecheck: `mypy src/`
- API handlers must return typed Pydantic models.
- Detailed architecture: @docs/architecture.md
```

### Path-Scoped `.claude/rules/terraform.md`
- When a convention applies to a file type across many directories (`**/*.tf` or `**/*.test.tsx`), use `.claude/rules/` with YAML `paths:` globs instead of scattering `CLAUDE.md` files into every folder

```yaml
# .claude/rules/terraform.md (YAML frontmatter + Markdown body)
paths:
  - "infra/**/*.tf"
  - "modules/**/*.tf"
# Body: Every resource must include environment and cost_center tags.
```

---

<!-- layout: 2-column -->
# Authoring Project Skills (`SKILL.md`) and Subagents (`.claude/agents/`)

### `.claude/skills/codebase-explainer/SKILL.md`
- Use `context: fork` so verbose multi-file exploration runs in an isolated subagent context and returns only a summary
- Use `allowed-tools` to restrict the Skill to read-only operations

```yaml
# .claude/skills/codebase-explainer/SKILL.md frontmatter
name: codebase-explainer
description: Explains module architecture and call graphs on demand.
context: fork
allowed-tools: Read, Grep, Glob
argument-hint: "[module-path]"
# Prompt body: Trace all entrypoints in $ARGUMENTS and return a 1-page summary.
```

### `.claude/agents/security-auditor.md`
- Custom subagents declare their own `model`, scoped `tools`, and optional persistent `memory` scope

```yaml
# .claude/agents/security-auditor.md frontmatter
name: security-auditor
description: Audits diffs for SQL injection, auth bypass, and secret leaks.
model: claude-sonnet-5-5
tools: Read, Grep, Glob
memory: project
# Prompt body: Review changed files for exploitable vulnerabilities.
```

---

# Configuring `settings.json` and Permission Precedence

- **Settings scope precedence (highest first)**: Enterprise managed policy -> CLI arguments -> local (`.claude/settings.local.json`, gitignored) -> project (`.claude/settings.json`, committed) -> user (`~/.claude/settings.json`)
- **Permission evaluation order**: `deny` beats `ask` beats `allow` at **every** scope; a user-level `allow` can **never** override a project-level or managed `deny`

```json
{
  "permissions": {
    "allow": ["Bash(npm run test *)", "Bash(git status)", "Read(src/**)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(./.env*)", "Read(./secrets/**)", "Bash(curl *)", "Bash(rm -rf *)"]
  }
}
```

---

<!-- layout: navigation -->
# Chapter 4

- Core Components and Configuration
- **Session Management and Execution Modes**
- Error Handling and Recovery
- Trace Analysis and Problem Isolation

---

<!-- layout: 3-column -->
# Claude Code Execution and Permission Modes

### Interactive and Plan Mode
- Default conversational terminal or IDE loop
- **Plan Mode** (`--permission-mode plan` or Shift+Tab) reads and researches without editing source files: use it before complex multi-file refactors

### Headless and Streaming
- `claude -p "..."` (`--print`) runs one prompt non-interactively and exits: the CI/CD form
- `--output-format json` (add `--json-schema` for validated structured output) or `stream-json` for real-time events to a harness

### Permission Modes
- `default`, `acceptEdits`, `plan`, `dontAsk`
- **`auto`**: a background classifier reviews each action instead of prompting you; `deny` and `ask` rules still apply
- **`bypassPermissions`**: skips all prompts; use **only** inside isolated containers or VMs with no host credentials

---

# Running Headless `claude -p` in CI/CD with `--json-schema`

- **Automating PR review or release notes in GitHub Actions**: pass `-p` so Claude Code never blocks waiting for interactive stdin, and enforce a JSON Schema on the output

```bash
git diff origin/main...HEAD | claude -p \
  "Review this diff for breaking API changes and SQL injection." \
  --allowedTools "Read,Grep,Glob" \
  --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "blocking_issues": {"type": "array", "items": {"type": "string"}},
      "ship_approved": {"type": "boolean"}
    },
    "required": ["blocking_issues", "ship_approved"]
  }' > review-result.json
```

---

<!-- layout: 2-column -->
# Session Management Commands and Workflows

### Resuming, Forking, and Reconciling Changes
- **`claude --continue` (`-c`)**: resumes the most recent conversation in the current directory
- **`claude --resume <session-name>` (`-r`)**: resumes a specific named investigation session across workdays
- **`claude --resume <id> --fork-session`** (Agent SDK: `fork_session=True`): branches a new session ID from a shared investigation baseline so you can test two competing refactors without polluting each other
- **External file edits**: resuming a session does **not** automatically re-read files modified outside Claude Code overnight; tell the agent which files changed so it re-reads them

### Managing Context Health Mid-Session
- **`/compact`**: summarizes verbose tool history into a concise state summary when the context window fills up during long debugging sessions (you can pass focus instructions: `/compact keep all database migration filenames`)
- **`/clear`**: resets conversation history completely when switching to an unrelated task so old file context does not bleed over

---

<!-- layout: navigation -->
# Chapter 4

- Core Components and Configuration
- Session Management and Execution Modes
- **Error Handling and Recovery**
- Trace Analysis and Problem Isolation

---

<!-- layout: card-layout -->
# Identifying API and Tool Error Types

### Client and Validation Errors (`400`, `401`, `403`, `404`, `413`)
- **`400 invalid_request_error`**: malformed JSON, missing `tool_result` after `tool_use`, modified `thinking` block signature, a legacy parameter the model rejects (`budget_tokens`, `temperature`), or prompt exceeds context window. Fix the request; do **not** retry.
- **`401` / `403`**: invalid key or missing workspace/IAM permission (fail fast and alert). **`413`**: payload exceeds request size limit.

### Rate Limits and Overloads (`429`, `500`, `529`)
- **`429 rate_limit_error`**: request or token quota exceeded; honor the `retry-after` header.
- **`529 overloaded_error`** / **`500 api_error`**: transient server-side capacity or internal failure; retry with exponential backoff and jitter.

### HTTP 200 Stop States (`max_tokens`, `refusal`)
- `refusal` and `max_tokens` arrive as HTTP `200 OK`, so exception handlers never catch them; always inspect `response.stop_reason` before reading `content`.

### Tool Execution Errors (`is_error: true`)
- When a tool fails (invalid customer ID, database timeout), catch the exception in your tool handler and return a `tool_result` with `is_error: true` (MCP: `isError: true`) so Claude can self-correct or explain the failure.

---

# Code: Catching Typed SDK Exceptions in Order

- **Catch specific subclasses before `APIStatusError` or `APIError`**: never blind-retry `BadRequestError` (`400`) or `AuthenticationError` (`401`)

```python
import anthropic

try:
    response = await client.messages.create(
        model="claude-sonnet-5-5", max_tokens=4096, messages=messages
    )
except anthropic.BadRequestError as err:          # HTTP 400: bug in our request payload
    logger.error("Non-retryable 400 payload bug: %s", err.message)
    raise
except anthropic.AuthenticationError as err:      # HTTP 401: invalid/expired API key
    alert_oncall_secops(err)
    raise
except anthropic.RateLimitError as err:           # HTTP 429: after SDK retries exhausted
    retry_after = err.response.headers.get("retry-after", "30")
    raise BackpressureError(f"Rate limited; back off for {retry_after}s") from err
except anthropic.APIStatusError as err:           # HTTP 500 / 529 transient server errors
    logger.warning("Server error %s after SDK retries", err.status_code)
    return fallback_response()
```

---

<!-- layout: navigation -->
# Chapter 4

- Core Components and Configuration
- Session Management and Execution Modes
- Error Handling and Recovery
- **Trace Analysis and Problem Isolation**

---

<!-- layout: stacked -->
# Isolating Integration Bugs vs Model Output Failures

- **Inspect the full execution trace first**: before rewriting prompts or switching models, examine the exact HTTP request payload, `messages` history, `tool_use` arguments, `tool_result` outputs, and `stop_reason`
- **Integration-layer bug**: the model received corrupted, incomplete, or stale inputs, or produced valid tool arguments that your handler code crashed on, or its output was truncated by `stop_reason == "max_tokens"`
- **Model-layer failure**: the model received clean, complete context and tool results, but misrouted a tool, ignored an instruction, or misstated a figure from the payload

![Trace Analysis Problem Isolation](images/ch04-trace-problem-isolation.svg)

---

<!-- layout: 2-column -->
# Worked Trace Investigation: Two Production Failures

### Case A: Exception During Tool Execution
- **Trace**:
  - `assistant` emits `tool_use`: `get_customer({"customer_id": "CUST-409", "include_archived": null})`
  - Schema allows `"include_archived": ["boolean", "null"]`
  - Application crashes with `TypeError: bool() argument...` inside `customer_service.py`
- **Diagnosis**: **Integration-layer bug**. Claude produced schema-valid JSON; the Python tool handler failed to handle `None` for an optional field. Rewriting the prompt is the wrong fix.

### Case B: Wrong Figure in Final Summary
- **Trace**:
  - `assistant` calls `get_invoice({"id": "INV-88"})`
  - `user` returns `tool_result` with `{"subtotal": 820.00, "tax": 65.60, "total": 885.60}`
  - `assistant` replies (`stop_reason: "end_turn"`): `"Your total invoice amount is $820.00."`
- **Diagnosis**: **Model-layer synthesis failure**. The integration layer returned clean data, and Claude picked `subtotal` instead of `total`. Fix with clearer field names/prompt rules or post-output validation.

---

# Trace Diagnostic Checklist for Developers

| Observed Failure in Production | What to Check in the Trace | Root Cause Layer and Fix |
| :--- | :--- | :--- |
| **`JSONDecodeError` when parsing structured output** | Check `response.stop_reason` | If `stop_reason == "max_tokens"`, **Integration Layer**: raise `max_tokens` and use `output_config.format` |
| **API returns `400` on turn 3 of a parallel tool loop** | Count `tool_use` blocks in turn 2 vs `tool_result` blocks in turn 3 | **Integration Layer**: return a matching `tool_result` for **every** `tool_use_id` in one `user` message |
| **Agent claims no orders exist when database timed out** | Check raw `tool_result` payload sent back to Claude | **Integration Layer**: tool caught the timeout and returned `[]` instead of `is_error: true` |
| **Agent picks `get_customer` when given an order number** | Compare `tools` array descriptions in the request | **Model / Tool Config Layer**: expand tool descriptions with explicit ID formats (`ORD-NNNNN`) and boundaries |

---

<!-- layout: 2-column -->
# Evals and Production Quality Monitoring

### Offline Evals (Before You Ship)
- Build a golden dataset from real traffic, edge cases, and past production bugs
- Run **deterministic code graders first** (JSON schema valid, exact ID match, tool name match), then **LLM-as-judge** with an explicit written rubric for tone and completeness
- Gate CI/CD pull requests on the eval suite: prompt, model, or tool changes must not regress the baseline

### Online Monitoring and A/B Isolation (After You Ship)
- Trace every request with OpenTelemetry: prompt version, model ID, `stop_reason`, latency, and all four `usage` token counters
- When production quality shifts after a release, isolate the cause with a **controlled A/B test** that changes **one variable at a time** (prompt vs model vs retrieval pipeline)

---

# Code: Two-Stage Evaluation Harness (Code Grader + LLM-as-Judge)

- **Combine fast deterministic assertions with rubric-scored LLM judging**:

```python
async def evaluate_case(case: GoldenCase, output: SupportReply) -> dict:
    # Stage 1: Deterministic code checks (fast, zero-variance)
    cited_correct_order = case.expected_order_id in output.cited_order_ids
    refund_within_policy = output.refund_usd <= case.max_allowed_refund_usd

    # Stage 2: Rubric-based LLM-as-judge for subjective policy and tone compliance
    judge = await client.messages.parse(
        model="claude-sonnet-5-5",
        max_tokens=512,
        output_format=RubricScore,
        system="Grade the assistant reply strictly against the provided 3-point rubric.",
        messages=[{"role": "user", "content": f"<rubric>{RUBRIC}</rubric><reply>{output.body}</reply>"}],
    )
    return {
        "deterministic_pass": cited_correct_order and refund_within_policy,
        "rubric_score": judge.parsed_output.score,
    }
```

---

# Domains 3 and 4 Quiz: Claude Code, Testing, and Debugging

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-13-ccdv-f-realistic-instructor.json)

---

# Lab 2: API Integration, Claude Code, and Debugging

**Time:** 45 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-developer-foundations-exam-prep-workshop/labs/lab-02-api-integration-claude-code-and-debugging)

---

<!-- layout: 2-column -->
# What You Learned

### Claude Code Configuration and Modes
- Authored `CLAUDE.md` files, path-scoped `.claude/rules/`, `SKILL.md` workflows, and custom subagents
- Enforced tool permissions and settings precedence in `.claude/settings.json`
- Operated interactive, Plan, headless (`claude -p`), streaming, and `auto` modes

### Error Recovery and Trace Isolation
- Handled typed SDK exceptions, rate limits, and `stop_reason` edge cases
- Isolated integration-layer bugs from model-output failures using execution traces and evals

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
