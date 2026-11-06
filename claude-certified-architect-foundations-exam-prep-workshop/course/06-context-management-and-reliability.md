<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 6: Context Management and Reliability

---

# Chapter 6: Objectives

- Preserve critical context across long sessions and codebases
- Design reliable escalation rules and error propagation
- Calibrate confidence scores for human review routing
- Preserve source provenance across multi-agent synthesis

---

# Domain 5: Context Management and Reliability

| Task # | Task Statement |
| :--- | :--- |
| **5.1** | Manage conversation context to preserve critical information across long interactions |
| **5.2** | Design effective escalation and ambiguity resolution patterns |
| **5.3** | Implement error propagation strategies across multi-agent systems |
| **5.4** | Manage context effectively in large codebase exploration |
| **5.5** | Design human review workflows and confidence calibration |
| **5.6** | Preserve information provenance and handle uncertainty in multi-source synthesis |

---

<!-- layout: navigation -->
# Chapter 6

- **Context Preservation and Codebase Exploration**
- Escalation and Error Propagation
- Human Review and Information Provenance

---

<!-- layout: card-layout -->
# Three Context Degradation Failure Modes

### 1. Progressive Summarization Loss
- Rolling conversation summaries condense exact numbers, percentages, dates, and customer-stated expectations into vague prose.
- Later turns lose precision and contradict earlier commitments.

### 2. The “Lost in the Middle” Effect
- Models process information at the very beginning and very end of long inputs reliably, but frequently overlook details buried in the middle.

### 3. Tool Result Accumulation Bloat
- Raw MCP tool calls (such as an order lookup returning 40+ database columns when only 5 matter) rapidly fill the context window with low-value noise.

---

<!-- layout: 2-column -->
# Persistent Case Facts Blocks

### The Pattern
- Extract transactional facts into a structured block: IDs, exact amounts, dates, statuses, and what the customer was promised
- Inject the block at the **top of every prompt**, outside the conversation history
- **Never summarize it**: update individual fields when facts change
- Summarize the conversation freely; the facts survive every compression

### Example Block
```text
<case_facts>
customer_id: C-48213 (verified)
order_id: ORD-58213
issue: cracked water tank
promised: replacement tank on ORD-58290
refund_amount: $847.50
refund_due_by: Nov 14
open_questions: none
</case_facts>
```

---

# Demo: A Summary That Drops the Numbers

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_10_case_facts.py`**

**Step 2. Paste this code**

```python
# demo_10_case_facts.py - summaries drop facts; a case facts block keeps them
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

TRANSCRIPT = """Customer: My espresso machine arrived with a cracked water tank. Order ORD-58213.
Agent: Sorry about that. I see you paid $899.00 for it.
Customer: I want to keep it if you can send a new tank, but I want money back too.
Agent: I can send a replacement tank on order ORD-58290 and refund $847.50 to your card.
Customer: What about the $51.50 shipping charge?
Agent: Shipping is not refundable on partial refunds, so that stays.
Customer: Fine. When will the refund show up?
Agent: The refund will post by Nov 14."""

CASE_FACTS = """<case_facts>
order_id: ORD-58213
replacement_order: ORD-58290
refund_amount: $847.50
shipping_refunded: no ($51.50 kept)
refund_due_by: Nov 14
</case_facts>"""

QUESTION = "What exact amount are we refunding, on which order, and by what date?"

def ask(content):
    response = client.messages.create(
        model="claude-haiku-5-5",
        max_tokens=1028,
        output_config={"effort": "low"},
        messages=[{"role": "user", "content": content}],
    )
    return "".join(b.text for b in response.content if b.type == "text")

summary = ask("Summarize this support chat for a shift handoff in 12 words or fewer. "
              f"Reply with only the summary.\n\n{TRANSCRIPT}")
print("Shift 1 summary:", summary)
summary = ask(f"Shorten these handoff notes to 8 words or fewer. Reply with only the notes.\n\n{summary}")
print("Shift 2 summary:", summary, "\n")

print("Summary only:\n", ask(f"Handoff notes: {summary}\n\n{QUESTION}"), "\n")
print("Summary plus case facts:\n", ask(f"{CASE_FACTS}\n\nHandoff notes: {summary}\n\n{QUESTION}"))
```

**Step 3. Run it**

```bash
python demo_10_case_facts.py
```

**Step 4. Examine the output**

- Each shift shortens the notes, and the original order `ORD-58213` should drop out
- With the summary only, Claude should be unsure of the order or name the replacement order `ORD-58290` instead
- With the case facts block, Claude should answer `$847.50` on `ORD-58213` by Nov 14

> [!NOTE]
> Each summary looked reasonable, but two rounds of compression lost the one ID the refund depends on. Keeping transactional facts in a separate block that is never summarized means every turn starts with the exact values.

---

<!-- layout: 2-column -->
# Trimming Verbose Tool Outputs

### Raw `lookup_order` Result
```json
{
  "order_id": "ORD-58213",
  "status": "delivered",
  "total": 899.00,
  "delivered_at": "2026-11-02",
  "return_eligible": true,
  "warehouse_audit": [ ...120 entries... ],
  "pick_list_ids": [ ... ],
  "carrier_raw_events": [ ... ],
  "...": "35 more internal fields"
}
```
- Over 40 fields, and only 5 matter to support

### Trimmed Before It Enters History
```json
{
  "order_id": "ORD-58213",
  "status": "delivered",
  "total": 899.00,
  "delivered_at": "2026-11-02",
  "return_eligible": true
}
```
- Trim in a `PostToolUse` hook or in the tool itself, so every call stays small
- Ten lookups now cost a few hundred tokens instead of filling the context window

---

<!-- layout: 2-column -->
# Position-Aware Input Ordering

### Lost in the Middle
- Models use information at the **start** and **end** of a long input reliably
- Details buried in the middle of 30 concatenated subagent reports are often overlooked
- The synthesis misses a key finding that was present all along

### Order the Input Deliberately
```markdown
## Key Findings (read first)
1. Music: 38% of labels use AI mastering
2. Film: 2 of 5 studios paused AI scripts
3. Gap: no data on games

## Detailed Evidence
Music: ...
Film: ...
```
- Summary of key findings at the **top**, with detailed evidence under explicit section headers

---

<!-- layout: 2-column -->
# Metadata-Rich Upstream Handoffs

### Raw Reasoning Handoff
- The search subagent returns 3,000 tokens of its own reasoning: what it tried, what it read, what it thought
- Synthesis has to dig the facts out of the narrative
- Citations and dates are often lost along the way

### Structured Handoff
```json
{
  "subtopic": "music",
  "findings": [{
    "claim": "38% of labels use AI mastering",
    "source_url": "https://example.org/music-2026",
    "published": "2026-03-14",
    "relevance": 0.9
  }],
  "gaps": ["no data on independent artists"]
}
```
- When downstream context budgets are tight, require facts, citations, and relevance, not reasoning chains

---

<!-- layout: 2-column -->
# Recognizing Context Degradation in Long Sessions

### The Symptom
- Early in the session: “`RefundService.apply_credit()` calls `LedgerClient.post()` in `ledger/client.py`”
- Two hours later: “refunds typically go through a service layer that updates the ledger”
- Exact names and paths have turned into vague “typical patterns”

### The Cause
- Verbose search results and file dumps fill the context window
- Earlier precise findings are pushed into the middle or compressed away
- The model falls back on general knowledge instead of what it discovered

<!-- below-columns -->
> [!TIP]
> When an exam scenario describes answers drifting from specific to generic over a long session, the answer is a context management remedy (the next slides), not a larger model or a longer system prompt.

---

<!-- layout: 2-column -->
# Scratchpad Files for Key Findings

### The Pattern
- The agent writes each **verified** finding to a file as soon as it confirms it
- It reads the file back when it needs those facts, instead of relying on history
- The file survives `/compact`, subagent boundaries, and new sessions

### Example `findings.md`
```markdown
# Refund flow: verified findings
- Entry: api/refunds.py handle_refund()
- Calls RefundService.apply_credit()
  (services/refund.py:41)
- Ledger write: LedgerClient.post()
  (ledger/client.py:88)
- Open: who calls reverse_credit()?
```

---

<!-- layout: 2-column -->
# Phase Summaries and Subagent Delegation

### Delegate Verbose Searches
- Spawn a subagent for “trace every caller of `reverse_credit()`”
- It reads dozens of files in **its own** context
- Only a short summary comes back to the main agent

### Carry Phases Forward with Summaries
- End each exploration phase with a structured summary of what was verified
- Inject that summary into the next phase’s subagent prompts
- Each phase starts with the facts it needs, not the raw history of earlier phases

---

<!-- layout: 2-column -->
# Crash-Recovery Manifests

### The Problem
- A coordinator has finished 6 of 10 research subtasks when the process crashes
- Restarting from scratch repeats all 6, and the in-memory state is gone
- Resuming a session does not help if the state was never written down

### The Manifest
```json
{
  "goal": "AI impact on creative industries",
  "subtasks": {
    "music": {"status": "done",
              "result": "out/music.json"},
    "film": {"status": "done",
             "result": "out/film.json"},
    "games": {"status": "in_progress"},
    "writing": {"status": "pending"}
  }
}
```
- On restart, the coordinator loads the manifest, reuses finished results, and continues from `games`

---

<!-- layout: 2-column -->
# `/compact` and When to Use It

### What It Does
- Replaces the conversation history with a condensed summary, freeing context space
- You can steer it: `/compact keep all file paths and function names we verified`
- Available in interactive Claude Code sessions

### When to Use It
- A long interactive exploration session is filling up with old search output
- The current task can continue from a summary of what happened so far
- **Not a fix for exact facts**: compaction is summarization, so keep IDs, amounts, and verified findings in a scratchpad or case facts block

---

<!-- layout: navigation -->
# Chapter 6

- Context Preservation and Codebase Exploration
- **Escalation and Error Propagation**
- Human Review and Information Provenance

---

<!-- layout: title-image -->
# Escalation Decision Matrix

![Escalation Decision Matrix](images/ch06-escalation-decision-tree.svg)

---

<!-- layout: 2-column -->
# Explicit Escalation Criteria with Few-Shot Examples

### Vague Rule
```text
Escalate complex or sensitive cases
to a human.
```
- “Complex” and “sensitive” are judged differently on every conversation
- The agent escalates simple cases and resolves ones it should not

### Explicit Criteria Plus Examples
```text
Resolve yourself: damaged item with a
photo, under $500, within 30 days.
Escalate: refund over $500, legal
threats, policy not covering the request,
or a direct request for a human.

Example: "Box arrived crushed, photo
attached, $64 lamp" -> resolve, send
replacement.
Example: "Price-match Competitor X"
-> escalate: policy covers own-site only.
```

---

<!-- layout: 2-column -->
# Honor Explicit Requests; Acknowledge Frustration First

### Explicit Request for a Human
- “I want to talk to a person.”
- **Escalate immediately**, with a structured handoff
- Do not investigate first, and do not try to talk the customer out of it

### Frustration Without a Request
- “This is the third time I’ve asked where my order is!”
- The issue (tracking) is within the agent’s ability
- **Acknowledge the frustration, then offer to resolve it now**
- Escalate if the customer then says they want a human

<!-- below-columns -->
> [!WARNING]
> Frustration is not the same as complexity. Escalating every upset customer overloads human agents with problems the agent could have solved in one turn.

---

<!-- layout: 2-column -->
# Policy Gaps Trigger Escalation

### The Situation
- Customer: “Competitor X sells this for $40 less. Will you match it?”
- The policy document covers **own-site** price adjustments within 14 days
- It says nothing about competitor prices

### The Rule
- When policy is **silent**, the agent must not invent a policy in either direction
- Granting the match sets an unauthorized precedent; refusing it may contradict an unwritten business rule
- Escalate with the policy gap stated in the handoff: “Request: competitor price match. Policy covers own-site only.”

---

<!-- layout: 2-column -->
# Multiple Matches: Ask for Identifiers, Never Guess

### The Situation
```json
get_customer(name="John Smith") ->
[
  {"id": "C-1182", "city": "Austin"},
  {"id": "C-7730", "city": "Denver"},
  {"id": "C-9014", "city": "Austin"}
]
```
- Three accounts match the name

### The Rule
- **Ask for another identifier**: email, phone, or an order ID
- Do not pick the most recent account, the first result, or the one that “seems likely”
- A wrong guess exposes one customer’s data to another and can refund the wrong account

---

<!-- layout: 2-column -->
# Unreliable Escalation Proxies vs Reliable Triggers

### Unreliable Proxies (Exam Traps)
- **Sentiment analysis scores**: customer frustration does not correlate with technical case complexity; a frustrated customer with a simple tracking question can be helped immediately
- **Self-reported confidence scores (1 to 10)**: uncalibrated LLM confidence scores are frequently overconfident on policy edge cases
- **External ML ticket classifiers**: over-engineered when prompt criteria have not been defined

### Reliable Escalation Triggers
- **Explicit customer request** for a human representative
- **Policy gaps or silence**: when a customer asks for competitor price matching and your policy document only covers own-site price adjustments
- **Policy threshold violations**: refunds exceeding autonomous limits (enforced via hook)
- **Inability to make progress** after clarification attempts

---

# Structured Error Context from Subagents

- **When a subagent cannot recover, it reports what failed, what it tried, and what it still has**: the coordinator needs all of that to choose its next step

| Field | Purpose |
| :--- | :--- |
| `failureType` | Timeout, rate limit, or permission: decides whether a retry elsewhere makes sense |
| `attemptedQuery` | The exact parameters tried, so the coordinator does not repeat them |
| `partialResults` | Data collected before the failure, so the work is not lost |
| `suggestedAlternatives` | Fallback sources or modified queries the coordinator can try |

```json
{
  "status": "partial_failure",
  "failureType": "UPSTREAM_TIMEOUT",
  "attemptedQuery": "EU AI Act compliance penalties 2026",
  "partialResults": [{ "claim": "Tier 1 fines up to 7% global turnover", "source": "eu-summary.pdf" }],
  "suggestedAlternatives": ["Query internal regulatory database MCP resource"]
}
```

---

<!-- layout: 2-column -->
# Local Recovery Before Propagation

### Recover Locally
- **Transient** failures (timeouts, rate limits) are retried inside the subagent with backoff
- **Validation** failures are fixed and retried inside the subagent
- The coordinator never hears about failures the subagent fixed itself

### Propagate Upward
- Failures the subagent cannot fix: permission errors, a source that stays down after retries, a business-rule block
- Send structured error context (previous slide), not a bare “failed”
- The coordinator decides: try an alternative source, re-scope, or continue with a gap

<!-- below-columns -->
> [!NOTE]
> Handling each failure at the lowest level that can fix it keeps the coordinator’s context free of retry noise and keeps decisions where the information is.

---

<!-- layout: 2-column -->
# Two Anti-Patterns: Silent Suppression and Whole-Workflow Termination

### Silent Suppression
- The subagent catches a timeout and returns `[]` as a success
- The coordinator concludes that no data exists
- The final report states “no regulatory penalties found,” which is false and has no warning attached

### Whole-Workflow Termination
- One of five research subagents fails, and the coordinator aborts everything
- Four subagents’ completed work is thrown away
- The user gets nothing, when a report with one marked gap would have been useful

<!-- below-columns -->
> [!IMPORTANT]
> The correct path is between the two: recover locally where possible, propagate structured context when not, and **continue with partial results that are clearly annotated**.

---

<!-- layout: 2-column -->
# Coverage Annotations in Synthesis Output

### The Instruction to Synthesis
```text
Some sources were unavailable. For each
subtopic, state whether coverage is
complete, partial, or missing, and why.
Do not fill gaps with general knowledge.
```

### Example Report Section
```markdown
## Coverage
| Subtopic | Coverage | Note |
| --- | --- | --- |
| Music | Complete | 4 sources |
| Film | Partial | Regulatory DB timed out |
| Games | Missing | No sources found |

## Findings
...
```
- Readers can tell a real “no evidence” from a gap caused by a failed source

---

<!-- layout: navigation -->
# Chapter 6

- Context Preservation and Codebase Exploration
- Escalation and Error Propagation
- **Human Review and Information Provenance**

---

# Aggregate Accuracy Hides Segment Failures

- **A single accuracy number averages away the failures that matter**: 97% overall can coexist with a segment that is wrong most of the time

| Segment | Documents | Field | Accuracy |
| :--- | :--- | :--- | :--- |
| Standard vendor invoices | 9,200 | All fields | 99.6% |
| Scanned handwritten invoices | 450 | `stated_total` | 88.0% |
| Vendor B multi-page layout | 350 | `line_items` | **40.0%** |
| **All documents** | **10,000** | **All fields** | **97.0%** |

- **Segment by document type and by field before automating**: Vendor B’s line items need human review even though the overall number looks excellent

> [!WARNING]
> “Accuracy is 97%, so remove human review for high-confidence extractions” is the exam trap. Automate only the segments whose own accuracy meets the bar.

---

<!-- layout: 2-column -->
# Calibrating Field-Level Confidence with a Labeled Set

### Field-Level, Not Document-Level
- The model outputs a confidence for **each** extracted field
- One uncertain `stated_total` should send that field to review, not the whole document
- Contradictory source values also route to review

### Calibrate the Threshold
1. Build a labeled validation set where humans recorded the correct value of each field
2. Run extraction and record the model’s confidence for each field
3. Measure actual accuracy at each confidence level, for example “fields at 0.9 or higher are 99.4% correct”
4. Auto-accept above the threshold that meets your target; route the rest to humans

---

<!-- layout: 2-column -->
# Stratified Random Sampling of High-Confidence Output

### Why Sample Accepted Output
- Auto-accepted extractions are never reviewed, so new errors there are invisible
- A new vendor layout or a changed form can break extraction while confidence stays high
- Ongoing sampling measures the real error rate in production

### What “Stratified” Means
- Divide accepted output into groups (strata), such as document type or vendor, then sample randomly **within each group**
- A pure random sample of 200 from 10,000 documents might include only 7 Vendor B documents
- **Example plan**: 30 per document type per week, plus 30 from any vendor first seen this month

---

<!-- layout: 2-column -->
# Claim-Source Mappings Through Every Handoff

### Where Attribution Is Lost
- A subagent summarizes its findings into a prose paragraph
- Synthesis merges five paragraphs into one report
- The final claims have no URLs, no page numbers, and no way to be checked

### Keep Claims Structured End to End
```json
{
  "claim": "Tier 1 fines reach 7% of global turnover",
  "evidence_excerpt": "fines of up to 7% of total worldwide annual turnover",
  "source_url": null,
  "document_name": "eu-ai-act-summary.pdf",
  "page_number": 14
}
```
- Every subagent emits this shape, and synthesis preserves and merges the mappings instead of rewriting them

---

<!-- layout: 2-column -->
# Temporal Context: Publication and Collection Dates

### Without Dates
- Source A: “12% of studios use AI scripting tools”
- Source B: “41% of studios use AI scripting tools”
- Synthesis reports a contradiction between the sources

### With Dates
- Source A: published 2023, survey collected 2022
- Source B: published 2026, survey collected 2025
- This is a **trend**, not a contradiction: adoption grew over three years
- Require every finding to carry its publication or data collection date

---

<!-- layout: 2-column -->
# Conflicting Statistics: Annotate, Do Not Resolve

### Wrong Ways to Handle a Conflict
- Pick the number from the “more reputable” source without saying so
- Average the two values (`(31% + 44%) / 2 = 37.5%`), a number neither source reported
- Drop the statistic entirely

### Annotated Output
```markdown
AI-assisted mastering adoption among
major labels in 2025 is contested:
- 31% (Industry Survey A, n=40 labels,
  self-reported, published 2026-01)
- 44% (Analyst Report B, includes
  pilots, published 2026-02)
The gap likely reflects whether pilot
programs are counted.
```
- Both values, both sources, and the methodological difference

---

<!-- layout: 3-column -->
# Rendering by Content Type

### Financial Figures: Tables
- Revenue by quarter, prices, fines, budgets
- Readers compare values side by side

### News and Developments: Prose
- What happened, why it matters, what changed
- Narrative keeps cause and effect clear

### Technical Findings: Structured Lists
- Steps, requirements, specifications
- Readers scan and check items off

<!-- below-columns -->
> [!TIP]
> Forcing every finding into one uniform format, such as all bullets or all tables, hurts readability. Synthesis should choose the format for each content type.

---

# Domain 5 Quiz: Context Management and Reliability

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-14-51-ccar-f-realistic-instructor.json)

---

# Lab 4: Context Preservation, Provenance, and Escalation Workflows

**Time:** 60 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-architect-foundations-exam-prep-workshop/labs/lab-04-context-preservation-provenance-and-escalation-workflows)

---

# What You Learned

- Preserved critical context across long sessions and codebases
- Designed reliable escalation rules and error propagation
- Calibrated confidence scores for human review routing
- Preserved source provenance across multi-agent synthesis

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
