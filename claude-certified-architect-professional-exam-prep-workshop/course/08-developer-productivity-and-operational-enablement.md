<!-- course-title: Claude Certified Architect - Professional: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Professional: Hands-On Exam Prep Workshop

# Chapter 8: Developer Productivity and Operational Enablement

---

# Chapter 8: Objectives

- Standardize Claude Code configurations across engineering teams
- Design AI-assisted workflows for refactoring and CI/CD reviews
- Triage and resolve production AI incidents systematically

---

# Domain 7: Developer Productivity and Operational Enablement

| Task # | Task Statement |
| :--- | :--- |
| **7.1** | Configure Claude tools and environments for teams (for example, Claude Code) |
| **7.2** | Improve developer workflows using AI-assisted tooling |
| **7.3** | Support debugging and operational issue resolution |

---

<!-- layout: navigation -->
# Chapter 8

- **Team Tooling Configuration**
- AI-Assisted Workflows
- Operational Debugging

---

<!-- layout: card-layout -->
# Standardizing Claude Code Across Engineering Teams

### Project-Scoped `CLAUDE.md` and `@import`
- Commit shared build commands, architectural boundaries, and testing standards in root `CLAUDE.md` (not `~/.claude/CLAUDE.md`) so every developer and CI runner inherits them.
- Use `@import` to link modular package standards cleanly.

### Conditional Path Rules (`.claude/rules/`)
- Place YAML frontmatter `paths` globs (such as `paths: ["**/*.test.ts", "src/api/**/*"]`) in `.claude/rules/*.md`.
- Loads domain conventions automatically when editing matching files anywhere in the repository without bloating unrelated sessions.

### Shared Commands and Isolated Skills
- Version-control team workflows in `.claude/commands/` and specialized tasks in `.claude/skills/<name>/SKILL.md`.
- Set `context: fork` and `allowed-tools` on verbose analysis skills so main developer sessions stay clean and safe.

### Governed Team MCP Servers (`.mcp.json`)
- Configure approved team MCP servers in project `.mcp.json` using `${ENV_VAR}` secret expansion and enterprise permission allowlists in `settings.json`.

---

<!-- layout: navigation -->
# Chapter 8

- Team Tooling Configuration
- **AI-Assisted Workflows**
- Operational Debugging

---

<!-- layout: 2-column -->
# Scaling AI-Assisted Engineering Workflows

### Interactive Architecture and Refactoring
- Use **Plan Mode** before executing multi-file migrations, microservice decompositions, or unfamiliar legacy refactoring
- Delegate broad codebase searches to the **`Explore` subagent** or maintain scratchpad files to prevent context degradation
- Use `fork_session` to compare alternative implementation designs from a shared discovery baseline

### Automated CI/CD Quality Gates
- Run headless Claude Code with `claude -p` (`--print`), `--output-format json`, and `--json-schema` for automated PR reviews
- Execute PR reviews in an **independent Claude instance** (never the same session that wrote the code)
- Split large PR reviews (such as 14+ files) into **per-file local passes** plus a **cross-file integration pass**

<!-- below-columns -->

> [!TIP]
> Protect developer trust in automated PR reviews by using explicit categorical reporting rules and temporarily disabling noisy, high false-positive categories while prompts are tuned.

---

<!-- layout: navigation -->
# Chapter 8

- Team Tooling Configuration
- AI-Assisted Workflows
- **Operational Debugging**

---

<!-- layout: card-layout -->
# Production Triage and Operational Issue Resolution

### 1. Isolate the Failing Layer First
- Inspect distributed trace spans (`trace_id`) to determine whether an incident originates in **retrieval/indexing**, **prompt/context assembly**, **model tier/latency**, or **MCP tool execution**.

### 2. Audit Recent Change Events
- Correlate sudden quality drops with recent operational events: a knowledge-base document refresh points to **retrieval/indexing**, while a sudden spike in TTFT and input cost points to a **broken prompt cache prefix**.

### 3. Inspect Structured Tool Telemetry
- Check MCP `isError` rates by `errorCategory`. A spike in `transient` errors requires backoff or circuit breaking, whereas `validation` errors indicate a schema or tool-description mismatch.

### 4. Execute Safe Rollbacks and Degradations
- Use pinned model IDs and versioned prompt registries to roll back a regressed prompt or index immediately, or route affected intents to human queues while root-causing.

---

# Operational Debugging Playbook Reference

| Production Incident Symptom | First Diagnostic Check | Immediate Remediation |
| :--- | :--- | :--- |
| **Confident wrong answers after doc update (latency unchanged)** | Check RAG re-indexing logs, chunk boundaries, and embedding alignment | Roll back to previous verified index snapshot and re-run ingestion eval |
| **TTFT doubles and input token cost spikes 4x overnight** | Check `cache_read_input_tokens` metric and diff recent system prompt commits | Move newly added dynamic per-request variables out of the static prefix |
| **Agent skips mandatory identity verification in 10% of turns** | Check whether verification relies on prompt prose instead of a code gate | Enforce a deterministic `PreToolUse` hook blocking downstream tools |
| **CI pipeline job hangs indefinitely on PR review step** | Check CLI invocation flags in CI workflow YAML | Add `-p` (`--print`) flag to run Claude Code in non-interactive mode |

---

# Domain 7 Quiz: Developer Productivity and Operational Enablement

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-11-54-ccar-p-realistic-instructor.json)

---

# Lab 4: Lifecycle Architecture Review and Team Enablement

**Time:** 60 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-architect-professional-exam-prep-workshop/labs/lab-04-lifecycle-architecture-review-and-team-enablement)

---

<!-- layout: 2-column -->
# What You Learned

### Team Enablement and CI/CD
- Standardized Claude Code configurations across engineering teams
- Designed AI-assisted workflows for refactoring and CI/CD reviews

### Production Operations
- Triaged and resolved production AI incidents systematically
- Executed safe rollbacks using pinned models and prompts

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
