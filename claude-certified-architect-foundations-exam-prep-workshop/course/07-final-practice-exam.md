<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 7: Final Practice Exam

---

# Chapter 7: Objectives

- Complete a timed scenario-based practice exam
- Review core decision rules across all five exam domains
- Solidify your final exam-day pacing strategy

---

<!-- layout: navigation -->
# Chapter 7

- **Practice Exam**
- Exam Review and Questions and Answers

---

# Final Practice Exam

- [Open Practice Exam](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-15-01-ccar-f-complex-instructor.json)

---

<!-- layout: navigation -->
# Chapter 7

- Practice Exam
- **Exam Review and Questions and Answers**

---

# Rapid Review: Domains 1 and 2 Cheat Sheet

| Exam Situation | Correct Architectural Decision | Common Trap to Avoid |
| :--- | :--- | :--- |
| **Loop termination** | Continue on `stop_reason == "tool_use"`; stop on `"end_turn"` | Parsing natural language or stopping when assistant text appears |
| **Mandatory tool order or refund cap** | Enforce deterministically via programmatic gate or SDK hook | Adding stronger system prompt warnings or few-shot examples |
| **Subagent missing upstream findings** | Pass findings explicitly in the `Task` prompt (isolated context) | Assuming subagents inherit the coordinator’s conversation history |
| **Similar tools misrouted (`get_customer` vs `lookup_order`)** | Expand tool descriptions with input formats, examples, and boundaries | Building an external keyword router or adding 8 few-shot examples first |
| **Synthesis agent needs 85% simple fact checks** | Grant a scoped `verify_fact` tool; route 15% complex cases via coordinator | Giving the synthesis agent all 18 web search tools |
| **Subagent hits a timeout** | Retry transient errors locally; return structured error context + partial results | Returning an empty array `[]` as success or crashing the whole pipeline |
| **Subagent fails mid-workflow** | Recover locally, then propagate structured context; continue with annotated coverage gaps | Terminating the whole workflow or silently suppressing the error |

---

# Rapid Review: Domains 3, 4, and 5 Cheat Sheet

| Exam Situation | Correct Architectural Decision | Common Trap to Avoid |
| :--- | :--- | :--- |
| **Test files scattered across repo need same rules** | `.claude/rules/*.md` with YAML frontmatter `paths: ["**/*.test.tsx"]` | Subdirectory `CLAUDE.md` in every folder or `~/.claude/CLAUDE.md` |
| **Skill produces verbose exploratory output** | Configure `context: fork` and `allowed-tools` in `SKILL.md` frontmatter | Running verbose discovery directly in the main conversation session |
| **Claude Code hangs in CI/CD pipeline** | Run non-interactively with `claude -p` (`--print`) + `--json-schema` | Using `CLAUDE_HEADLESS=true`, `--batch`, or `< /dev/null` |
| **14-file PR review has superficial/contradictory comments** | Split into per-file local passes + a separate cross-file integration pass | Switching to a larger context window in a single pass |
| **Model fabricates missing document fields** | Make schema fields nullable (`["string", "null"]`) + `"unclear"` enums | Retrying when information is absent from the source document |
| **Two credible sources have conflicting stats** | Preserve both values with source URLs and publication dates | Picking one number arbitrarily or averaging them silently |

---

<!-- layout: 2-column -->
# Final Exam-Day Checklist

### Before the Exam
- Verify your government-issued photo ID matches your Anthropic Partner Academy / Pearson VUE registration name exactly
- Prepare a clean workspace with no secondary monitors, notes, phones, or smart watches
- Review the 6 exam scenarios and 5 domain blueprints one final time

### During the Exam
- Read the scenario header first: note which tools exist and what metric is failing
- Ask: **“Does this require 100% deterministic compliance (hook/gate) or better model judgment (descriptions/criteria/few-shot)?”**
- Eliminate any option that relies on out-of-scope features (fine-tuning, custom ML classifiers, or cloud infrastructure hosting)

---

# What You Learned

- Completed a timed scenario-based practice exam
- Reviewed core decision rules across all five exam domains
- Solidified your final exam-day pacing strategy

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
