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

<!-- layout: card-layout -->
# Practice Exam Simulation Guidelines

### Exam Format Match
- 53 multiple-choice and multiple-response items spanning all eight CCDV-F blueprint domains.
- Remember that **Domain 2 (Applications and Integration)** accounts for roughly one-third (33.1%) of the exam.

### Time Management
- Budget roughly 2 minutes 15 seconds per item across the 120-minute window.
- Flag uncertain items, record your best answer, and review flagged questions before submitting.

### Diagnostic Review
- Note any questions where two options looked plausible so we can examine the underlying blueprint rule in the review session.

---

# Final Practice Exam

- [Open Practice Exam](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-19-ccdv-f-realistic-instructor.json)

---

<!-- layout: navigation -->
# Chapter 9

- Practice Exam
- **Exam Review and Questions and Answers**

---

# Rapid Review: Domains 1 to 4 Cheat Sheet

| Domain / Scenario | Correct Developer Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D1: Workflow vs Agent** | Use deterministic Workflows for fixed sequences; use Autonomous Agents for dynamic, open-ended tasks | Using unconstrained agents for predictable 3-step pipelines |
| **D1: Agent Frameworks** | `PydanticAI` for type-safe schemas; `LangGraph` for stateful cyclic graphs; `Strands` for model-driven loops | Writing fragile regex parsers instead of structured validation |
| **D2: 10,000 Docs Overnight (Cost Focus)** | Use the **Message Batches API** (up to 24-hour window, 50% cost reduction, `custom_id`) | Sending 10,000 synchronous requests in parallel or lowering `max_tokens` |
| **D2: Production Model Stability** | Pin exact snapshot IDs (`claude-sonnet-4-20250514`) and version-control prompts/`CLAUDE.md` | Using unpinned floating model aliases in production without eval gates |
| **D3: Claude Code in CI/CD** | Run non-interactively with `claude -p` (`--print`) + `--output-format json` + `--json-schema` | Omitting `-p` (hangs waiting for stdin) or putting team rules in `~/.claude/CLAUDE.md` |
| **D4: Trace Analysis Isolation** | Inspect raw request/response traces first to separate **integration bugs** (`max_tokens` truncation, swallowed errors) from **model output failures** | Blindly rewriting prompts when `stop_reason == "max_tokens"` truncated the JSON |

---

# Rapid Review: Domains 5 to 8 Cheat Sheet

| Domain / Scenario | Correct Developer Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D5: Model Selection and Thinking** | Haiku for fast routing/classification, Sonnet for core coding/agents, Opus for deep reasoning; use adaptive/extended thinking when needed | Downsizing blindly to the smallest model regardless of task quality fit |
| **D5: Cost and Token Management** | Track `response.usage` counters and place `cache_control` checkpoints on static tools, system prompts, and history prefixes | Placing dynamic timestamps at the start of the system prompt (breaks cache) |
| **D6: Context and Output Handling** | Prune verbose tool outputs, isolate noisy work in subagents, validate with Pydantic/Zod, and check cross-field math | Trusting confident LLM numbers without schema or arithmetic verification |
| **D7: Untrusted Web Page Prompt Injection** | Treat page content as **untrusted input** (XML-isolated), separate from trusted instructions, and enforce **guardrails/hooks** | Raising temperature, polite system-prompt warnings, or switching to a larger model |
| **D8: Reusable Internal Inventory API** | Build an **MCP Server** exposing inventory tools so multiple Claude apps can connect and share it | Hard-coding API logic into prompts, pasting live data in context, or assuming built-in tools reach internal APIs |

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
