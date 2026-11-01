<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 6: Prompt and Context Engineering

---

# Chapter 6: Objectives

- Prevent context drift with pruning, compaction, and subagents
- Structure prompts with XML tags, sanitization, and few-shots
- Validate structured outputs and verify confident claims

---

# Domain 6: Prompt and Context Engineering

| Task # | Task Statement |
| :--- | :--- |
| **6.1** | Context Engineering: Context window management, preventing context drift and bloat (tool output pruning, compaction), and subagent context isolation |
| **6.2** | Prompt Engineering: Instruction clarity, few-shot examples, system vs user placement, output constraints, iterative refinement, and input sanitization |
| **6.3** | Output Handling: Structured output patterns, response validation, defensive parsing, and skepticism toward confident output |

---

<!-- layout: navigation -->
# Chapter 6

- **Context Engineering**
- Prompt Engineering
- Output Handling

---

<!-- layout: card-layout -->
# Preventing Context Drift and Bloat

### 1. Tool Output Pruning
- Raw API or database responses often contain dozens of irrelevant fields that rapidly consume tokens and dilute model attention.
- Use a `PostToolUse` hook or tool wrapper to strip responses down to only the fields required for the task before appending to `messages`.

### 2. Context Compaction and Pinned Facts
- Progressive summarization of long conversations can lose exact numbers, dates, and IDs.
- Maintain a separate, persistent `<case_facts>` block outside summarized history so critical identifiers never degrade.

### 3. Context Isolation via Subagents
- When a step requires reading 20 files or scraping verbose web pages, delegate it to an isolated subagent (the `Agent` tool, formerly `Task`, or a `context: fork` Skill).
- Only the subagent’s structured summary returns to the parent agent’s context window.

### 4. Position-Aware Context Ordering
- Put long documents at the top of the prompt and the question and instructions at the end; wrap each document in XML tags with source metadata so Claude can find and cite it.

---

<!-- layout: 3-column -->
# API-Native Context Management

### Context Editing
- Clears stale tool results (`clear_tool_uses_20250919`) or old thinking blocks from the request before Claude sees it
- Best for long tool loops where old results no longer matter
- Removes content; does not summarize

### Server-Side Compaction
- The API summarizes earlier turns automatically when the conversation nears a token threshold
- Always append the full `response.content` so compaction blocks are preserved
- Best for very long conversations

### Memory Tool
- `memory_20250818`: Claude reads and writes files in a memory directory that your app stores
- Keeps durable facts outside the context window across sessions
- Best for agents that must remember across sessions

<!-- below-columns -->

> [!NOTE]
> Pruning and compaction can be built yourself (hooks, `<case_facts>`) or with these API features.

---

<!-- layout: navigation -->
# Chapter 6

- Context Engineering
- **Prompt Engineering**
- Output Handling

---

<!-- layout: 2-column -->
# System vs User Placement and Structural Clarity

### Top-Level `system` Parameter
- Place persistent role definitions, universal behavioral constraints, tone rules, and static reference policies in `system`
- Keeps static instructions at the front of the payload so they can be reused across turns with **Prompt Caching** (no timestamps or per-user values here)

### `user` Message Content
- Place per-request task inputs, retrieved RAG chunks wrapped in XML tags (`<retrieved_context>`), and the specific user question in `messages`
- Sanitize and wrap untrusted user inputs before interpolating them into prompt templates

<!-- below-columns -->

> [!TIP]
> When prose instructions produce inconsistent formatting or edge-case decisions, add 2 to 4 targeted **few-shot examples** demonstrating the exact desired input-to-output transformation.

---

# Structuring Prompts with XML Tags and Input Sanitization

- **Separate instructions, examples, and untrusted data** with unambiguous XML delimiters:

```xml
<instructions>
Extract all security vulnerabilities from the provided pull request diff.
Flag an issue ONLY when code introduces an exploitable flaw; skip style preferences.
</instructions>
<example>
  <diff>+ query = f"SELECT * FROM users WHERE name = '{user_input}'"</diff>
  <finding severity="critical" category="sql_injection">Use parameterized queries.</finding>
</example>
<untrusted_pr_diff>
{{sanitized_git_diff}}
</untrusted_pr_diff>
```

---

<!-- layout: navigation -->
# Chapter 6

- Context Engineering
- Prompt Engineering
- **Output Handling**

---

<!-- layout: stacked -->
# Defensive Output Handling and Validation

- **Enforce structure at generation time**: use **structured outputs** (`output_config.format` with a JSON Schema, or `client.messages.parse()` with a Pydantic model) and `strict: true` on tools; forced `tool_choice` and prefill are legacy and rejected by the newest models
- **Parse defensively at runtime**: check `stop_reason` first (`max_tokens` truncation, `refusal`), then validate business rules with Pydantic or Zod: a schema-valid answer can still be wrong
- **Maintain skepticism toward confident output**: LLMs state hallucinated numbers or miscalculated sums with high linguistic confidence; verify cross-field math (`calculated_total == stated_total`) and source citations in code

![Defensive Output Handling Pipeline](images/ch06-defensive-output-handling.svg)

---

# Defensive Parsing and Validation-Retry Example

- **Structured output plus semantic validation**: the schema guarantees shape; code checks meaning

```python
response = await client.messages.parse(
    model="claude-sonnet-5-5", max_tokens=16000,
    output_format=InvoiceExtraction,  # Pydantic model -> JSON Schema
    messages=[{"role": "user", "content": build_prompt(doc)}])
if response.stop_reason in ("max_tokens", "refusal"):
    raise IncompleteOutputError(response.stop_reason)

invoice = response.parsed_output  # schema-valid, but not necessarily correct
line_sum = sum(item.amount for item in invoice.line_items)
if line_sum != invoice.stated_total:
    feedback = f"Items sum to {line_sum}, not {invoice.stated_total}"
    return await retry_with_feedback(doc, invoice, feedback)
```

---

# Domain 6 Quiz: Prompt and Context Engineering

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-15-ccdv-f-realistic-instructor.json)

---

# Lab 3: Model Optimization and Context Engineering

**Time:** 45 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-developer-foundations-exam-prep-workshop/labs/lab-03-model-optimization-and-context-engineering)

---

<!-- layout: 2-column -->
# What You Learned

### Context and Prompt Engineering
- Prevented context drift with pruning, compaction, and subagents
- Structured prompts with XML tags, sanitization, and few-shots

### Defensive Output Handling
- Enforced structured output with `output_config.format` and strict tools
- Validated structured outputs and verified confident claims

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
