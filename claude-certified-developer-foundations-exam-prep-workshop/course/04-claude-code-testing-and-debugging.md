<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 4: Claude Code, Testing, and Debugging

---

# Chapter 4: Objectives

- Configure Claude Code rules, skills, agents, memory, and permissions
- Operate interactive, Plan, headless, streaming, and auto modes
- Handle API and tool errors with resilient recovery strategies
- Analyze execution traces and evals to isolate root causes

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

# Configuring `settings.json` and Permissions

- **Precedence (highest first)**: managed -> CLI flags -> local (`.claude/settings.local.json`) -> project (`.claude/settings.json`) -> user (`~/.claude/settings.json`)
- **Rules**: `deny` beats `ask` beats `allow`; a narrower allow cannot override a broad deny (also Domain 2 configuration)

```json
{
  "permissions": {
    "allow": ["Bash(npm run test *)", "Bash(git status)", "Read(src/**)"],
    "deny": ["Read(./.env*)", "Read(./secrets/**)", "Bash(curl *)"]
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
- **Plan Mode** reads and researches without editing source files: use it before complex multi-file refactors

### Headless and Streaming
- `claude -p "..."` (`--print`) runs one prompt non-interactively and exits: the CI/CD form
- `--output-format json` (add `--json-schema` for validated structured output) or `stream-json` for real-time events to a harness

### Permission Modes
- `default`, `acceptEdits`, `plan`, `dontAsk`
- **`auto`**: a classifier reviews each action instead of prompting you; deny and ask rules still apply
- **`bypassPermissions`**: skips prompts; only in isolated containers or VMs

---

<!-- layout: 2-column -->
# Session Management Commands and Workflows

### Resuming and Branching Sessions
- **`claude --continue` (`-c`)**: resumes the most recent conversation in the current directory
- **`claude --resume <session-name>` (`-r`)**: resumes a specific named investigation session across workdays
- **`claude --resume <id> --fork-session`** (Agent SDK: `fork_session=True`): branches from a shared analysis baseline to test two competing implementation strategies independently

### Managing Context Health Mid-Session
- **`/compact`**: summarizes verbose tool history into a concise state summary when the context window fills up during long debugging sessions
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
- **`400 invalid_request_error`**: malformed request, missing `tool_result` after `tool_use`, a parameter the model no longer supports, or too many tokens. Fix the request; do **not** blind-retry.
- **`401` / `403`**: invalid key or missing permission (fail fast and alert). **`413`**: request too large.

### Rate Limits and Overloads (`429`, `500`, `529`)
- **`429 rate_limit_error`**: quota exceeded; honor `retry-after`. **`529 overloaded_error`** / **`500 api_error`**: transient; back off with jitter.
- The SDKs retry these twice by default and raise typed exceptions (`RateLimitError`, `APIStatusError`): catch most-specific first.

### Not Errors: Check `stop_reason`
- `refusal` and `max_tokens` arrive as HTTP 200, so check `stop_reason` before reading `content`.
- Tool failures go back as a `tool_result` with `is_error: true` (MCP: `isError`); schema failures go back as a bounded self-correction retry.

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
- **Integration-layer bug**: the model received corrupted, incomplete, or stale inputs (or its output was truncated by `stop_reason == "max_tokens"`)
- **Model-layer failure**: the model received clean, complete context and instructions, but misrouted a tool or hallucinated an absent field

![Trace Analysis Problem Isolation](images/ch04-trace-problem-isolation.svg)

---

# Trace Diagnostic Checklist for Developers

| Observed Failure in Production | What to Check in the Trace | Root Cause Layer and Fix |
| :--- | :--- | :--- |
| **`JSONDecodeError` when parsing structured output** | Check `response.stop_reason` | If `stop_reason == "max_tokens"`, **Integration Layer**: increase `max_tokens` and use `tool_use` schema |
| **API returns `400` on turn 3 of an agent loop** | Check message ordering after assistant `tool_use` | **Integration Layer**: ensure every `tool_use_id` has a matching `tool_result` in the very next user message |
| **Agent claims no orders exist when database timed out** | Check raw `tool_result` payload sent back to Claude | **Integration Layer**: tool caught the timeout and returned `[]` instead of a `tool_result` with `is_error: true` |
| **Agent picks `get_customer` when given an order number** | Check `tools` array descriptions in the request | **Model / Tool Config Layer**: expand tool descriptions with explicit input formats and boundaries |

---

<!-- layout: 2-column -->
# Evals and Production Quality Monitoring

### Offline Evals (Before You Ship)
- Build a golden dataset from real traffic, including edge cases and past failures
- Grade with code checks first (schema valid, exact match), then an LLM-as-judge with a written rubric
- Gate CI on the eval suite: prompt, model, or tool changes must not regress the score

### Online Monitoring (After You Ship)
- Trace every request: inputs, tool calls, `stop_reason`, latency, and `usage` (OpenTelemetry works with the SDKs)
- Alert on shifts in refusal rate, schema-failure rate, cache hit rate, and cost per completed task
- Feed production failures back into the golden dataset

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
- Configured Claude Code rules, skills, agents, memory, and permissions
- Operated interactive, Plan, headless, streaming, and auto modes

### Error Recovery and Trace Isolation
- Handled API and tool errors with resilient recovery strategies
- Analyzed execution traces and evals to isolate root causes

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
