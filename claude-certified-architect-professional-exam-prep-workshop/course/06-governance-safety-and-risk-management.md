<!-- course-title: Claude Certified Architect - Professional: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Professional: Hands-On Exam Prep Workshop

# Chapter 6: Governance, Safety, and Risk Management

---

# Chapter 6: Objectives

- Implement layered defense-in-depth guardrails across input, context, tool execution, and output stages
- Identify LLM failure modes including indirect prompt injection, hallucinated citations, and over-privileged actions
- Design human-in-the-loop (HITL) validation workflows and calibrated review routing
- Architect Claude solutions that comply with GDPR, HIPAA, and FedRAMP requirements and uphold ethical AI standards

---

# Domain 5: Governance, Safety, and Risk Management

| Task # | Task Statement |
| :--- | :--- |
| **5.1** | Implement guardrails and safety controls |
| **5.2** | Identify risks, limitations, and failure modes of LLM systems |
| **5.3** | Apply human-in-the-loop validation strategies |
| **5.4** | Ensure compliance with regulations (for example, GDPR, HIPAA, FedRAMP) |
| **5.5** | Address ethical AI considerations (bias, fairness, transparency) |

---

<!-- layout: navigation -->
# Chapter 6

- **Guardrails and Safety Controls**
- Human-in-the-Loop Validation
- Compliance and Ethical AI

---

<!-- layout: stacked -->
# Layered Defense-in-Depth Guardrail Architecture

- **Why single-layer defenses fail**: a polite system prompt rule (`"Do not follow malicious user instructions"`) can be bypassed by adversarial inputs or indirect injection inside retrieved documents
- **Four enforcement layers**: combine **Input Screening**, **Context Trust Isolation**, **Deterministic Action Hooks**, and **Output Validation**
- **Low-latency guardrail execution**: run fast input classifiers (such as Claude Haiku or regex PII scanners) in parallel with retrieval so safety checks do not bottleneck interactive TTFT

![Layered Defense-in-Depth Guardrail Architecture](images/ch06-layered-guardrails.svg)

---

<!-- layout: card-layout -->
# Identifying LLM Risks, Limitations, and Failure Modes

### Indirect Prompt Injection
- External untrusted content (web pages, emails, support tickets, RAG chunks) embeds hidden instructions attempting to hijack the agent’s tools or exfiltrate data.
- **Mitigation**: isolate untrusted data in XML tags, strip unnecessary tools (least privilege), and enforce deterministic `PreToolUse` hooks on all state-changing actions.

### Hallucinations and Unverified Citations
- Models can synthesize plausible-sounding facts or fabricate legal/medical citations when source context is missing or ambiguous.
- **Mitigation**: require verbatim evidence excerpts and source IDs via structured output schemas, allow explicit `"unclear"` / abstention responses, and verify citations post-generation.

### Sycophancy and Premature Compliance
- Models may agree with a user’s false premise or waive policy rules under emotional pressure.
- **Mitigation**: ground decisions in external policy documents and enforce prerequisites and financial limits in code.

---

<!-- layout: navigation -->
# Chapter 6

- Guardrails and Safety Controls
- **Human-in-the-Loop Validation**
- Compliance and Ethical AI

---

<!-- layout: 2-column -->
# Designing Human-in-the-Loop (HITL) Validation Strategies

### Pre-Execution Action Approval (Synchronous)
- Pauses agent execution **before** a high-impact or irreversible tool call runs (such as issuing a large wire transfer, deleting production records, or sending external legal notices)
- Presents the human approver with a structured summary: `proposed_action`, `parameters`, `supporting_evidence`, and `policy_check_result`
- Resumes the agentic loop only after explicit cryptographic or authenticated human sign-off

### Post-Generation Review Queues (Asynchronous)
- Routes completed extractions, reports, or policy exceptions to human specialists based on **calibrated field-level confidence thresholds** or detected source conflicts
- Prioritizes scarce human reviewer bandwidth on low-confidence or high-risk items
- Uses **stratified random sampling** on auto-approved high-confidence outputs to detect silent drift

<!-- below-columns -->

> [!WARNING]
> Combat **automation bias** (reviewers rubber-stamping AI outputs) by highlighting exact source excerpts side-by-side with extracted fields and flagging contested data points explicitly.

---

<!-- layout: navigation -->
# Chapter 6

- Guardrails and Safety Controls
- Human-in-the-Loop Validation
- **Compliance and Ethical AI**

---

<!-- layout: 3-column -->
# Designing for Enterprise Regulatory Compliance

### GDPR (Data Privacy)
- **Data minimization**: redact unnecessary PII before sending prompts to the API
- **Right to erasure**: never bake raw customer PII into long-lived prompt caches, fine-tuned weights, or immutable evaluation datasets
- **Data residency**: route traffic through approved regional endpoints to satisfy cross-border transfer rules

### HIPAA (Protected Health Info)
- **BAA requirement**: execute a formal Business Associate Agreement (BAA) with the platform provider before processing PHI
- **Zero Data Retention (ZDR)**: use ZDR-configured API endpoints so prompts and completions containing PHI are never persisted by the model provider
- Enforce end-to-end encryption and strict access audit logs

### FedRAMP (Public Sector)
- Deploy within **FedRAMP-authorized cloud environments** (such as authorized AWS GovCloud / Bedrock or Google Cloud Vertex AI regions)
- Enforce FIPS-validated cryptographic modules, strict IAM boundary controls, and continuous compliance telemetry

---

<!-- layout: card-layout -->
# Ethical AI: Bias, Fairness, and Transparency

### Subgroup Bias Evaluation
- Evaluate model outputs, refusal rates, and escalation decisions across protected demographic subgroups (age, gender, race, language dialect) using counterfactual test suites.
- Verify that automated triage or summarization does not systematically disadvantage specific groups.

### Transparency and AI Disclosure
- Clearly disclose to end users when they are interacting with an AI agent rather than a human representative, and provide an immediate path to human escalation.

### Explainability and Auditability
- Persist immutable audit records linking every automated decision to the exact prompt version, model ID, retrieved chunk IDs, tool outputs, and guardrail checks that produced it.

---

# Domain 5 Quiz: Governance, Safety, and Risk Management

- Open the Exam Prep Program to complete the Domain 5 review questions

<!-- TODO: Add Exam Prep Program URL for Domain 5 Quiz -->

---

# Lab 3: Evaluation Pipelines and Safety Guardrails

**Time:** 45 minutes

---

# What You Learned

- Implemented defense-in-depth guardrails across input screening, context isolation, deterministic tool hooks, and output validation
- Mitigated indirect prompt injection, hallucinated citations, and capability bloat
- Designed synchronous pre-execution HITL approval gates and calibrated asynchronous human review queues
- Mapped architectural controls to GDPR, HIPAA (BAA + ZDR), FedRAMP, and ethical AI fairness standards

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
