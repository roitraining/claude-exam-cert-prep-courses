<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 4: Claude Code Configuration and Workflows

---

# Chapter 4: Objectives

- Configure `CLAUDE.md` hierarchies and path-scoped `.claude/rules/`
- Create custom slash commands and isolated skills
- Choose between Plan Mode and Direct Execution
- Integrate Claude Code into CI/CD pipelines

---

# Domain 3: Claude Code Configuration and Workflows

| Task # | Task Statement |
| :--- | :--- |
| **3.1** | Configure `CLAUDE.md` files with appropriate hierarchy, scoping, and modular organization |
| **3.2** | Create and configure custom slash commands and skills |
| **3.3** | Apply path-specific rules for conditional convention loading |
| **3.4** | Determine when to use plan mode vs direct execution |
| **3.5** | Apply iterative refinement techniques for progressive improvement |
| **3.6** | Integrate Claude Code into CI/CD pipelines |

---

<!-- layout: navigation -->
# Chapter 4

- **CLAUDE.md and Path Rules**
- Slash Commands and Skills
- Plan Mode and Iterative Refinement
- CI/CD Integration

---

<!-- layout: stacked -->
# The `CLAUDE.md` Configuration Hierarchy

- **User-level (`~/.claude/CLAUDE.md`)**: personal instructions applied only on your machine; **never shared via version control**
- **Project-level (`CLAUDE.md` or `.claude/CLAUDE.md`)**: committed to Git so every developer and CI job receives universal team standards
- **Subdirectory-level (`packages/billing/CLAUDE.md`)**: directory-bound instructions for a specific package or service folder

![Claude Code Configuration Hierarchy](images/ch04-claude-md-hierarchy.svg)

---

# Modular Organization and Diagnostics

- **Diagnosing missing instructions**: if a new teammate clones the repository and Claude Code ignores team conventions, check whether the author placed those conventions in `~/.claude/CLAUDE.md` instead of the project-level `CLAUDE.md`
- **Modularizing with `@import`**: reference external standards files (such as `@docs/api-standards.md` or `@docs/security-checklist.md`) inside package `CLAUDE.md` files to avoid duplicating prose
- **Inspecting loaded memory**: run the `/memory` command inside Claude Code to verify exactly which `CLAUDE.md` and `.claude/rules/` files are active in the current session

---

# Path-Scoped Rules in `.claude/rules/`

- **Conditional rule loading**: files in `.claude/rules/*.md` use YAML frontmatter `paths` globs so conventions load **only** when Claude edits matching files
- **Solves scattered file conventions**: when test files (`Button.test.tsx`) live next to components across dozens of directories, `.claude/rules/testing.md` with `paths: ["**/*.test.tsx"]` applies test conventions everywhere automatically

```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "terraform/**/*"
---
# Testing and Infrastructure Conventions
- Use React Testing Library queries by accessible role, never by test ID.
- Mock external HTTP calls using MSW fixtures from `src/test/fixtures/`.
```

---

<!-- layout: 2-column -->
# Choosing `.claude/rules/` vs `CLAUDE.md`

### Use `.claude/rules/` with `paths` Globs
- Conventions apply to file types spread across many directories (`**/*.test.tsx`, `**/*.sql`)
- Rules are specialized (such as Terraform HCL or GraphQL schemas) and would waste context tokens when editing unrelated code
- Keeps active context lean and relevant

### Use Root or Subdirectory `CLAUDE.md`
- **Root `CLAUDE.md`**: universal repo commands (build, lint, test), architecture overview, and rules that apply to every file
- **Subdirectory `CLAUDE.md`**: rules strictly confined to one folder tree (such as `services/legacy-auth/CLAUDE.md`)

---

<!-- layout: navigation -->
# Chapter 4

- CLAUDE.md and Path Rules
- **Slash Commands and Skills**
- Plan Mode and Iterative Refinement
- CI/CD Integration

---

<!-- layout: 2-column -->
# Custom Slash Commands: Project vs User Scope

### Project-Scoped (`.claude/commands/`)
- Markdown files placed in `.claude/commands/` inside the repository (for example, `.claude/commands/review.md` creates `/review`)
- Version-controlled in Git and automatically available to every developer who clones or pulls the repo
- Use for standardized team checklists, PR reviews, and release workflows

### User-Scoped (`~/.claude/commands/`)
- Stored in the developer’s home directory
- Available only to that individual user across their local projects
- Note: `.claude/config.json` with a `commands` array does **not** exist in Claude Code (common exam distractor)

---

<!-- layout: card-layout -->
# Configuring Skills with `SKILL.md` Frontmatter

### `context: fork`
- Runs the skill inside an isolated sub-agent context window.
- Prevents verbose codebase analysis, log dumps, or exploratory brainstorming from polluting the main conversation history.

### `allowed-tools`
- Restricts which tools Claude can invoke while executing the skill (for example, `allowed-tools: ["Read", "Grep", "Glob"]` or read-only inspection).
- Prevents destructive actions during diagnostic or review skills.

### `argument-hint`
- Displays parameter guidance when a developer types the skill name without required arguments (for example, `argument-hint: "<migration-name> <target-schema>"`).

### Personal Skill Variants
- Store personal skill customizations in `~/.claude/skills/` under a distinct skill name so you do not override or disrupt shared project skills in `.claude/skills/`.

---

# Example `SKILL.md` Configuration and Selection Rule

- **When to choose a Skill vs `CLAUDE.md`**:
  - **Skill (`.claude/skills/`)**: on-demand, task-specific workflows invoked for specialized jobs (security audit, migration generator)
  - **`CLAUDE.md` / `.claude/rules/`**: always-on or file-path-triggered coding standards

```markdown
---
name: security-audit
description: Audits authentication and SQL modules for OWASP vulnerabilities
context: fork
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: "<directory-or-module-path>"
---
Analyze the target module for injection risks and return a summarized report.
```

---

<!-- layout: navigation -->
# Chapter 4

- CLAUDE.md and Path Rules
- Slash Commands and Skills
- **Plan Mode and Iterative Refinement**
- CI/CD Integration

---

<!-- layout: 2-column -->
# Plan Mode vs Direct Execution

### Use Plan Mode When
- Restructuring a monolith into microservices or migrating a library across 45+ files
- Multiple valid architectural approaches exist with different infrastructure or dependency trade-offs
- You need safe, read-only codebase exploration and dependency mapping before modifying code

### Use Direct Execution When
- Fixing a well-scoped single-file bug with a clear stack trace
- Adding a straightforward validation check or date conditional to a known function
- Executing an implementation approach that was already designed and approved in Plan Mode

<!-- below-columns -->

> [!TIP]
> For multi-phase tasks with verbose discovery, delegate exploration to the **`Explore` subagent** so discovery output is summarized and does not exhaust the main conversation context window.

---

<!-- layout: card-layout -->
# Four Iterative Refinement Patterns

### 1. Concrete Input/Output Examples
- When prose instructions for data or code transformations are interpreted inconsistently, provide 2 to 3 concrete input/output examples.
- Examples eliminate ambiguity faster than adding longer prose rules.

### 2. Test-Driven Iteration
- Write the test suite covering expected behavior, edge cases (such as `null` values in migration scripts), and performance first.
- Iterate by sharing test execution failures until all tests pass.

### 3. The Interview Pattern
- Before implementing in an unfamiliar domain, instruct Claude to interview you and ask clarifying questions first.
- Surfaces hidden design considerations such as cache invalidation or partial-failure recovery early.

### 4. Interacting vs Independent Fixes
- Send **interacting issues together in a single detailed message** so the fix accounts for all constraints simultaneously.
- Fix **independent issues sequentially** one at a time.

---

<!-- layout: navigation -->
# Chapter 4

- CLAUDE.md and Path Rules
- Slash Commands and Skills
- Plan Mode and Iterative Refinement
- **CI/CD Integration**

---

# Running Claude Code in Automated CI/CD Pipelines

- **Preventing interactive hangs**: always pass the **`-p` (or `--print`)** flag so Claude Code runs non-interactively, prints to stdout, and exits cleanly
- **Enforcing structured JSON output**: combine **`--output-format json`** with **`--json-schema`** so the pipeline can parse findings deterministically and post inline PR comments

```bash
claude -p "Review the git diff for security and concurrency bugs" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"number"},"severity":{"type":"string"},"issue":{"type":"string"},"suggested_fix":{"type":"string"}},"required":["file","line","severity","issue"]}}},"required":["findings"]}'
```

> [!WARNING]
> Exam Trap: Flags like `--batch`, environment variables like `CLAUDE_HEADLESS=true`, or `< /dev/null` redirection are distractors. Always choose **`claude -p`**.

---

<!-- layout: 2-column -->
# High-Signal CI Code Reviews and Test Generation

### Independent Review Context
- A Claude session that just generated code retains its own reasoning context and is poor at questioning its own assumptions
- Always run automated PR review in an **independent Claude instance** with clean context
- On follow-up commits, pass **prior review findings** in context and instruct Claude to report only new or still-unaddressed issues

### Context-Aware Test Generation
- Include **existing test files** in context so Claude does not generate duplicate test scenarios already covered
- Document testing standards, high-value test criteria, and available mock fixtures in project `CLAUDE.md` so CI jobs inherit them automatically

---

# Domain 3 Quiz: Claude Code Configuration and Workflows

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-14-48-ccar-f-realistic-instructor.json)

---

# Lab 2: MCP Tool Design and Claude Code Configuration

**Time:** 45 minutes

---

# What You Learned

- Configured `CLAUDE.md` hierarchies and path-scoped `.claude/rules/`
- Created custom slash commands and isolated skills
- Chose between Plan Mode and Direct Execution
- Integrated Claude Code into CI/CD pipelines

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
