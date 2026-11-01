<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 9: Final Practice Exam

---

# Chapter 9: Objectives

- Complete a timed CCDV-F practice exam simulation
- Review core decision rules across all eight exam domains
- Solidify your final exam-day pacing strategy

---

<!-- layout: navigation -->
# Chapter 9

- **Practice Exam**
- Exam Review and Questions and Answers

---

# Final Practice Exam

- [Open Practice Exam](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-19-ccdv-f-realistic-instructor.json)

---

<!-- layout: navigation -->
# Chapter 9

- Practice Exam
- **Exam Review and Questions and Answers**

---

# Rapid Review: Domains 1 and 2 Cheat Sheet

| Domain / Scenario | Correct Developer Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D1: Workflow vs Agent** | Workflows for fixed sequences; agents for open-ended tasks | Unconstrained agents for predictable 3-step pipelines |
| **D1: Agent Frameworks** | `PydanticAI` for type-safe schemas; `LangGraph` for stateful cyclic graphs; `Strands` for model-driven loops | Writing fragile regex parsers instead of structured validation |
| **D1: Self-Hosted vs Managed** | Agent SDK or Tool Runner for private systems; Managed Agents when Anthropic should host | Assuming a hosted sandbox reaches private VPC services |
| **D2: 10,000 Docs Overnight (Cost Focus)** | **Message Batches API** (24-hour window, 50% off, `custom_id`) | Parallel synchronous requests or lower `max_tokens` |
| **D2: Production Model Stability** | Pin model ID, SDK, and API version; version-control prompts, `CLAUDE.md`, plugins | Upgrading without an eval gate; ignoring retirement dates |
| **D2: Shared Team Configuration** | Project `.claude/settings.json` and `CLAUDE.md` in git; plugins for config shared across many repos | Putting team rules in `~/.claude/CLAUDE.md` or settings in `settings.local.json` |

---

# Rapid Review: Domains 3 to 5 Cheat Sheet

| Domain / Scenario | Correct Developer Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D3: Claude Code in CI/CD** | Run non-interactively with `claude -p` (`--print`) + `--output-format json` + `--json-schema` | Omitting `-p` (hangs waiting for input) |
| **D3: Fewer Prompts, Safely** | `auto` mode (classifier reviews actions) plus deny rules; `bypassPermissions` only in an isolated container | Using `bypassPermissions` on a developer laptop |
| **D4: Trace Analysis Isolation** | Read raw traces first: **integration bugs** (truncation, swallowed errors) vs **model failures** | Rewriting prompts when `max_tokens` truncated the JSON |
| **D5: Model Selection and Thinking** | Haiku for routing, Sonnet for coding/agents, Opus for deep reasoning; adaptive thinking + `effort` | Blind downsizing; `budget_tokens` on a current model |
| **D5: Model Upgrade Broke Production** | Check for API-breaking changes (prefill, forced `tool_choice`, sampling params) and behavior shifts; rerun evals before switching | Rewriting prompts before reading the 400 error or the eval diff |
| **D5: Cost and Token Management** | Track `usage`; `cache_control` on static tools, system, and history prefix | Timestamps at the top of the system prompt (breaks cache) |

---

# Rapid Review: Domains 6 to 8 Cheat Sheet

| Domain / Scenario | Correct Developer Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D6: Context and Output Handling** | Prune or clear old tool results, compact, isolate noisy work in subagents; use structured outputs, then check business rules | Trusting confident LLM numbers because the JSON is schema-valid |
| **D7: Untrusted Web Page Prompt Injection** | Treat page content as **untrusted input** (XML-isolated), separate from trusted instructions, and enforce **guardrails/hooks** | Raising temperature, polite system-prompt warnings, or switching to a larger model |
| **D7: Destructive Actions** | `PreToolUse` hook (exit `2` or `permissionDecision: "deny"`) layered with deny rules and least privilege | Relying on a system-prompt rule, or a hook denylist alone |
| **D8: Reusable Internal Inventory API** | Build an **MCP Server** exposing inventory tools so multiple Claude apps can connect and share it | Hard-coding API logic into prompts, pasting live data in context, or assuming built-in tools reach internal APIs |
| **D8: Skill vs MCP** | Skill for reusable know-how and procedures; MCP for live access to an external system | Building an MCP server just to deliver instructions |

---

<!-- layout: 2-column -->
# Final Exam-Day Checklist

### Before Launching Pearson VUE
- Confirm your government-issued photo ID matches your registration name exactly
- Clear your workspace of notes, phones, smart watches, and secondary monitors
- Remember the passing score is **720 on a 100 to 1,000 scale** based on your total scaled score

### Deconstructing Every Developer Question
- Check **latency vs cost constraints**: is this interactive (`Messages API` + streaming + caching) or overnight bulk (`Message Batches API`)?
- Check **trust boundaries**: is external data isolated from system instructions and guarded by deterministic hooks?
- Check **reusability scope**: is this single-app (custom tool), repo-local (`CLAUDE.md` / Skill), or multi-app reusable (**MCP Server**)?

---

# What You Learned

- Completed a timed CCDV-F practice exam simulation
- Reviewed core decision rules across all eight exam domains
- Solidified your final exam-day pacing strategy

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
