<!-- course-title: Claude Certified Architect - Professional: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Professional: Hands-On Exam Prep Workshop

# Chapter 8: Developer Productivity and Operational Enablement

---

# Chapter 8: Objectives

- Standardize and govern Claude Code configuration across engineering teams
- Design AI-assisted workflows for refactoring, CI/CD reviews, and productivity measurement
- Triage and resolve production AI incidents systematically

---

# Domain 7: Developer Productivity and Operational Enablement (7% of Exam)

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
- Version-control team workflows in `.claude/commands/` and specialized tasks in `.claude/skills/<name>/SKILL.md`; in current releases, commands and skills are merged, and both create a `/name` command.
- Set `context: fork` and `allowed-tools` on verbose analysis skills so main developer sessions stay clean and safe.

### Governed Team MCP Servers (`.mcp.json`)
- Configure approved team MCP servers in project `.mcp.json` using `${ENV_VAR}` secret expansion, and enforce which servers are allowed through settings.

---

# Settings Scopes and Precedence

| Scope | Location | Shared With | Use It For |
| :--- | :--- | :--- | :--- |
| **Managed (enterprise)** | Managed settings deployed by IT or the admin console | Everyone in the organization; users cannot override | Security policy: denied commands and files, allowed MCP servers, telemetry |
| **Command line** | Flags such as `--permission-mode` or `--allowedTools` | One session | Temporary overrides and CI runs |
| **Local project** | `.claude/settings.local.json` (not committed) | One developer in one repo | Personal experiments |
| **Shared project** | `.claude/settings.json` (committed) | Everyone who clones the repo | Team permissions, hooks, and project conventions |
| **User** | `~/.claude/settings.json` | One developer across all repos | Personal preferences |

- **Higher scopes win**, from managed at the top to user at the bottom, and a **deny** rule at any scope blocks the action even when another scope allows it

> [!IMPORTANT]
> Put team rules in committed project files, and put non-negotiable security policy in managed settings. Instructions in `CLAUDE.md` are guidance Claude usually follows; permission rules are enforced.

---

# Permissions as Team Policy

- **Three rule lists**: `allow` runs without asking, `ask` always prompts, and `deny` blocks outright; deny wins over allow
- **Scope rules narrowly**: allow the exact test and lint commands, and deny secrets and destructive commands

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test:*)",
      "Bash(npm run lint)"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./secrets/**)",
      "Bash(curl:*)",
      "Bash(rm -rf:*)"
    ]
  }
}
```

---

# Demo: A Project Deny Rule

This demo runs Claude Code, so it uses your Claude subscription. In a terminal in `ccarp-labs`:

**Step 1. Create a demo project with a fake secret**

**macOS:**

```bash
mkdir -p demo-deny/config
cd demo-deny
echo "PAYMENTS_TOKEN=demo-not-a-real-secret" > config/payments.env
```

**Windows (PowerShell):**

```powershell
mkdir demo-deny\config
cd demo-deny
Set-Content config\payments.env "PAYMENTS_TOKEN=demo-not-a-real-secret"
```

**Step 2. Ask Claude Code to read it, with no rules in place**

```bash
claude -p "Print the contents of config/payments.env"
```

**Step 3. Create `.claude/settings.json` in `demo-deny` with a deny rule**

```json
{
  "permissions": {
    "deny": ["Read(./config/payments.env)"]
  }
}
```

**Step 4. Run the same command again**

```bash
claude -p "Print the contents of config/payments.env"
```

**Step 5. Examine the output**

- Step 2 prints the fake token: reading project files is allowed by default
- Step 4 does not print it: Claude reports that it cannot read the file
- The only change was one committed settings file, which every teammate and CI runner now inherits

> [!NOTE]
> A line in `CLAUDE.md` saying “never read payments.env” is guidance. A deny rule is enforcement. For team policy, choose the control that cannot be talked around. Type `cd ..` to return to `ccarp-labs` when you finish.

---

<!-- layout: 2-column -->
# Distributing Team Configuration with Plugins

### Repository Files
- `CLAUDE.md`, `.claude/settings.json`, `.claude/skills/`, `.claude/rules/`, and `.mcp.json` committed to one repo
- Best when the configuration belongs to that project
- Every clone gets it automatically

### Plugins from a Marketplace
- A **plugin** bundles Skills, subagents, hooks, MCP servers, and commands behind one manifest
- A **marketplace** is a catalog of plugins, such as an internal Git repository your platform team maintains
- Best when the same standards must reach dozens of repositories; update once, and teams pick up the new version

<!-- below-columns -->

> [!TIP]
> Pair the two: a platform team publishes the shared review, security, and deployment plugins; each repository adds only what is specific to it, and managed settings enforce the non-negotiables.

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
- Fork a session (`--resume` with `--fork-session` on the CLI, or `fork_session` in the Agent SDK) to compare alternative designs from a shared discovery baseline

### Automated CI/CD Quality Gates
- Run headless Claude Code with `claude -p` (`--print`), `--output-format json`, and `--json-schema` for automated PR reviews
- Execute PR reviews in an **independent Claude instance** (never the same session that wrote the code)
- Split large PR reviews (such as 14+ files) into **per-file local passes** plus a **cross-file integration pass**

<!-- below-columns -->

> [!TIP]
> Protect developer trust in automated PR reviews by using explicit categorical reporting rules and temporarily disabling noisy, high false-positive categories while prompts are tuned.

---

# Code: Claude Code as a CI Review Step

- **Non-interactive by design**: `-p` prints one result and exits, so the pipeline never waits for input; the repository’s `CLAUDE.md` and settings apply in CI too

```yaml
name: ai-pr-review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash
      - name: Review the diff
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > pr.diff
          claude -p "Review pr.diff against CLAUDE.md. Report only correctness and security issues." \
            --output-format json > review.json
```

> [!NOTE]
> Anthropic also publishes a GitHub Action for Claude Code that handles setup and posts review comments. Either way, the key comes from the CI secret store, never from the repository.

---

<!-- layout: 2-column -->
# Measuring Developer Productivity Gains

### Vanity Metrics
- Lines of code generated
- Number of AI suggestions accepted
- Number of sessions or prompts per developer
- Each one rises without any improvement in what ships

### Outcome Metrics
- **Cycle time**: from first commit to production, per change type
- **Review load**: time to first review and number of review rounds
- **Quality**: escaped defects, rollback rate, and change failure rate
- **Developer experience**: survey trends and time spent on toil versus design work

<!-- below-columns -->

> [!IMPORTANT]
> Measure a baseline before rollout and compare like with like. A productivity claim without a baseline and a quality metric is not evidence.

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
- Check tool `isError` rates by the error category your tools return. A spike in `transient` errors requires backoff or circuit breaking, whereas `validation` errors indicate a schema or tool-description mismatch.

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
| **Bursts of 429 errors at peak hours** | Compare request and token rates with your rate limits per model | Retry with backoff that honors `retry-after`, smooth traffic with a queue, or request higher limits |
| **Some requests return no answer and `stop_reason` is `refusal`** | Review which inputs trigger it and whether they are legitimate | Handle `refusal` explicitly: rephrase, fall back to another model, or route to a human |
| **Developers report a team MCP server’s tools are missing** | Run `/mcp` for server status, then `claude --debug` for connection errors | Fix the server URL, credentials, or allowlist entry, then reconnect |

---

# Worked Exam Question: Keeping Secrets Out of Sessions

A platform team supports 300 developers using Claude Code across 120 repositories. Security requires that Claude Code never reads files under `secrets/` or runs `curl` in any repository, and that individual developers cannot turn this off. What is the best approach?

- **A.** Add “Never read secrets/ or run curl” to every repository’s `CLAUDE.md`
- **B.** Ask each developer to add deny rules to `~/.claude/settings.json`
- **C.** Deploy managed settings with deny rules for `Read(./secrets/**)` and `Bash(curl:*)`
- **D.** Add a `PreToolUse` hook to each repository that logs secret-file reads for weekly review

---

<!-- layout: 2-column -->
# Worked Exam Question: Reasoning

### Applying the Four Steps
- **Constraint:** organization-wide enforcement that users cannot override
- **Layer:** Claude Code configuration scope
- **Eliminate:** A is guidance, not enforcement; B is per user and can be removed; D is detective and per repository
- **Answer: C.** Managed settings sit at the top of the precedence order and apply everywhere

### Why It Matters
- Match the scope of the setting to the scope of the requirement
- Permission rules enforce; `CLAUDE.md` advises
- Logging is not a substitute for blocking

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

### Team Enablement and Workflows
- Standardized and governed Claude Code configuration across engineering teams
- Designed AI-assisted workflows for refactoring, CI/CD reviews, and productivity measurement

### Production Operations
- Triaged and resolved production AI incidents systematically
- Executed safe rollbacks using pinned models and prompts

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
