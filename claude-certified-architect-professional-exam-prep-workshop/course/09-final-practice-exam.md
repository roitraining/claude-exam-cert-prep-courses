<!-- course-title: Claude Certified Architect - Professional: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Professional: Hands-On Exam Prep Workshop

# Chapter 9: Final Practice Exam

---

# Chapter 9: Objectives

- Complete a timed CCAR-P practice exam simulation
- Review core decision rules across all seven exam domains
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
- 63 multiple-choice and multiple-response items spanning all seven CCAR-P blueprint domains.
- Pay close attention to multiple-response prompts that specify selecting two or three options.

### Time Management
- Budget roughly 1 minute 50 seconds per item across the 120-minute session.
- Flag lengthy architectural trade-off items, record your best choice, and return during your final review pass.

### Diagnostic Focus
- Note which of the seven domains feels most challenging during the simulation so we can dissect those items in the review session.

---

# Final Practice Exam

- [Open Practice Exam](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-11-57-ccar-p-realistic-instructor.json)

---

<!-- layout: navigation -->
# Chapter 9

- Practice Exam
- **Exam Review and Questions and Answers**

---

# Rapid Review: Domains 1 to 4 Cheat Sheet

| Domain / Scenario | Correct Enterprise Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D1: Pattern Selection** | Start with Augmented LLM or deterministic Workflow; use Autonomous Agents only for open-ended paths | Using unconstrained multi-agent loops for fixed 3-step business processes |
| **D2: 8K Static Policy + Short User Turn** | Place static prompt and policy **first** and enable **Prompt Caching** | Truncating policy to 1K tokens, downsizing blindly, or moving policy to few-shot |
| **D2: Model Tiering** | Haiku for fast routing/guardrails, Sonnet for core agent/RAG, Opus for deep synthesis | Using Opus with max extended thinking for high-volume ticket triage |
| **D3: Capability Bloat / Extra Tools** | **Remove unneeded tools** (`delete_account`, `issue_refund`) from agent config | Adding detective audit logs or confirmation prompts while keeping the tools attached |
| **D3: Protocol Selection** | **MCP** for reusable cross-app tools/resources; **Direct API** for tight single-app calls; **A2A** for specialist agent delegation | Hard-coding internal API logic into system prompts across multiple apps |
| **D4: Confident Wrong RAG Answers After Doc Refresh** | Investigate **retrieval and indexing** first (broken re-index, stale chunks, mismatched embeddings) | Blaming silent model weight changes, temperature, or shrinking context windows |

---

# Rapid Review: Domains 5 to 7 Cheat Sheet

| Domain / Scenario | Correct Enterprise Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D5: Indirect Prompt Injection Defense** | Isolate untrusted text in XML tags + least-privilege tool removal + deterministic `PreToolUse` hooks | Adding a system prompt line asking users not to include malicious instructions |
| **D5: Regulatory Compliance (GDPR / HIPAA / FedRAMP)** | GDPR: PII minimization/erasure; HIPAA: BAA + Zero Data Retention (ZDR); FedRAMP: authorized cloud regions | Storing raw customer PII/PHI inside long-term prompt caches or public eval logs |
| **D6: Communicating Accuracy SLAs** | Frame LLM quality as a statistical benchmark distribution backed by deterministic safety gates and HITL fallbacks | Promising 100% deterministic LLM outputs without code gates |
| **D6: Lifecycle Governance** | Maintain ADRs, pinned model versions, golden eval gates in CI, and operational rollback runbooks | Upgrading production traffic to a new model release without running golden evals |
| **D7: Team Claude Code Standardization** | Root `CLAUDE.md` + `.claude/rules/` path globs + `.claude/commands/` + `claude -p` in CI | Putting team rules in `~/.claude/CLAUDE.md` or omitting `-p` in CI pipelines |

---

<!-- layout: 2-column -->
# Final Exam-Day Checklist

### Before Launching Pearson VUE
- Verify your government-issued ID matches your Anthropic Partner Academy registration name exactly
- Clear your desk of all notes, phones, watches, and secondary monitors
- Remember the cut score is **720 on a 100 to 1,000 scale**, determined by your total scaled score

### Deconstructing Every Question
- Identify the **primary constraint**: Latency? Cost? Security/Least Privilege? Compliance? Retrieval accuracy?
- Prefer **preventive architectural controls** (removing unneeded tools, deterministic hooks, ACL pre-filtering) over detective or prompt-only controls
- Match every failure symptom to the **exact layer** that recently changed

---

# What You Learned

- Completed a timed CCAR-P practice exam simulation
- Reviewed core decision rules across all seven exam domains
- Solidified your final exam-day pacing strategy

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
