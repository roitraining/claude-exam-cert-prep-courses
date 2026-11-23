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

# Final Practice Exam

- [Open Practice Exam](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-11-57-ccar-p-realistic-instructor.json)

---

<!-- layout: navigation -->
# Chapter 9

- Practice Exam
- **Exam Review and Questions and Answers**

---

# Rapid Review: Domains 1 and 2 Cheat Sheet

| Domain / Scenario | Correct Enterprise Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D1: Pattern Selection** | Augmented LLM or deterministic Workflow when the steps are known; Agent only when the next step depends on what was just found | Unconstrained agents or multi-agent hierarchies for fixed, regulated sequences |
| **D1: Multi-Agent Design** | Use subagents for parallel breadth, context isolation, or separated permissions; contracts and budgets per worker | Multi-agent for tightly coupled work, which multiplies tokens and loses shared context |
| **D1: Business Value** | Optimize cost per resolved case and resolution rate, tied to SLAs | Optimizing raw cost per token while escalations dominate the unit economics |
| **D2: 8K Static Policy + Short User Turn** | Place static prompt and policy **first** and enable **Prompt Caching**; dynamic content after the cached prefix | Truncating policy to 1K tokens, downsizing blindly, or moving policy to few-shot |
| **D2: Model Tier and Effort** | Haiku for high-volume routing and screening, Sonnet for core agents, Opus for deep synthesis; set **effort** per route and measure | Opus at maximum effort for ticket triage, or the smallest model regardless of task fit |
| **D2: Many Procedures, Few Used per Request** | Package them as **Skills** (progressive disclosure) | Truncating procedures, or loading all of them into every request |

---

# Rapid Review: Domains 3 and 4 Cheat Sheet

| Domain / Scenario | Correct Enterprise Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D3: Capability Bloat / Extra Tools** | **Remove unneeded tools** from the agent config; use **tool search** when a large catalog is unavoidable | Adding audit logs or confirmation prompts while keeping the tools attached |
| **D3: Authentication** | Propagate the user’s identity with scoped, short-lived tokens; no shared superuser accounts; no token passthrough | A system prompt telling the agent to access only the user’s own data |
| **D3: Protocol Selection** | **MCP** for tools shared across apps; **Direct API/CLI** for one tight integration; **A2A** for independently owned agents across boundaries | A2A for in-app subagents, or duplicating API wrappers in every app |
| **D3: Exact Codes Missed by Retrieval** | **Hybrid retrieval**: BM25 fused with vectors, plus metadata filters | Raising top-K or switching to a bigger model |
| **D4: Confident Wrong RAG Answers After Doc Refresh** | Investigate **retrieval and indexing** first (broken re-index, stale chunks, mismatched embeddings) | Blaming silent model weight changes, temperature, or shrinking context windows |
| **D4: Is the New Prompt Better?** | Larger stratified eval set, several trials per case, one variable changed, then a canary | Declaring a winner from a few cases or a single run |

---

# Rapid Review: Domains 5 to 7 Cheat Sheet

| Domain / Scenario | Correct Enterprise Decision | Common Exam Distractor |
| :--- | :--- | :--- |
| **D5: Control Strength** | **Preventive** controls (remove the tool, ACL filter, code gate) over detective (logs) and compensating (confirmations) | “Add logging and review it weekly” as the primary fix |
| **D5: Indirect Prompt Injection and Exfiltration** | Break a leg of the lethal trifecta: scope data, isolate untrusted content, remove or gate outbound actions | A system prompt asking the model to ignore instructions in documents |
| **D5: Regulatory Compliance (GDPR / HIPAA / FedRAMP)** | GDPR: minimization and erasure across logs, datasets, and indexes; HIPAA: BAA plus ZDR-eligible features; FedRAMP: authorized cloud environments | Assuming every feature and model is ZDR- or BAA-eligible |
| **D6: Communicating Accuracy SLAs** | Measurable success criteria, quality SLOs on an agreed eval set, deterministic gates and HITL for costly errors | Promising 100% deterministic LLM outputs |
| **D6: Lifecycle Governance** | ADRs, pinned model IDs, golden eval gates in CI, rollback runbooks, and a model migration playbook | Moving production to a new model without side-by-side evaluation |
| **D7: Team Claude Code Standardization** | Committed project settings and `CLAUDE.md`; **managed settings** for policy users cannot override; `claude -p` in CI | Team rules in `CLAUDE.md` alone or `~/.claude/` files; omitting `-p` in CI |

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
