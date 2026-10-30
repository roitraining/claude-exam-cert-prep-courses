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
- When a step requires reading 20 files or scraping verbose web pages, delegate that step to an isolated subagent (`Task` or `context: fork` skill).
- Only the subagent’s structured summary returns to the parent agent’s context window.

### 4. Position-Aware Context Ordering
- Mitigate the “lost in the middle” effect by placing key summaries at the top of long inputs and organizing sections with explicit headers or XML tags.

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
- Keeps static instructions at the front of the payload so they can be reused across turns with **Prompt Caching**

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

<examples>
  <example>
    <diff>+ query = f"SELECT * FROM users WHERE name = '{user_input}'"</diff>
    <finding severity="critical" category="sql_injection">Use parameterized queries.</finding>
  </example>
</examples>

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

- **Enforce structure at generation time**: use `tool_use` with a strict JSON Schema (`tool_choice: {"type": "tool", "name": "..."}`) and nullable fields for optional data
- **Parse defensively at runtime**: always check `stop_reason` first (to catch `"max_tokens"` truncation) and validate with Pydantic or Zod before passing data downstream
- **Maintain skepticism toward confident output**: LLMs state hallucinated numbers or miscalculated sums with high linguistic confidence; verify cross-field math (`calculated_total == stated_total`) and source citations in code

![Defensive Output Handling Pipeline](images/ch06-defensive-output-handling.svg)

---

# Defensive Parsing and Validation-Retry Example

- **Catching truncation and semantic validation errors** in Python:

```python
if response.stop_reason == "max_tokens":
    raise TruncatedOutputError("Response hit max_tokens ceiling before completing JSON.")

tool_block = next(b for b in response.content if b.type == "tool_use")
try:
    invoice = InvoiceExtraction.model_validate(tool_block.input)
    if invoice.calculated_total != invoice.stated_total:
        raise ValueError(f"Line item sum {invoice.calculated_total} != stated {invoice.stated_total}")
except ValidationError as err:
    # Retry with original document + failed JSON + exact validation error
    return await retry_extraction_with_feedback(doc, tool_block.input, str(err))
```

---

# Domain 6 Quiz: Prompt and Context Engineering

- Open the Exam Prep Program to complete the Domain 6 review questions

<!-- TODO: Add Exam Prep Program URL for Domain 6 Quiz -->

---

# Lab 3: Model Optimization and Context Engineering

**Time:** 45 minutes

---

<!-- layout: 2-column -->
# What You Learned

### Context and Prompt Engineering
- Prevented context drift with pruning, compaction, and subagents
- Structured prompts with XML tags, sanitization, and few-shots

### Defensive Output Handling
- Enforced structured output via `tool_use` schemas
- Validated structured outputs and verified confident claims

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
