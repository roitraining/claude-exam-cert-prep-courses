<!-- course-title: Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Architect - Foundations: Hands-On Exam Prep Workshop

# Chapter 5: Prompt Engineering and Structured Output

---

# Chapter 5: Objectives

- Write explicit criteria and targeted few-shot examples
- Enforce structured JSON outputs using `tool_use` schemas
- Build validation-retry loops for extraction quality
- Design Message Batches workflows and multi-pass reviews

---

# Domain 4: Prompt Engineering and Structured Output

| Task # | Task Statement |
| :--- | :--- |
| **4.1** | Design prompts with explicit criteria to improve precision and reduce false positives |
| **4.2** | Apply few-shot prompting to improve output consistency and quality |
| **4.3** | Enforce structured output using tool use and JSON schemas |
| **4.4** | Implement validation, retry, and feedback loops for extraction quality |
| **4.5** | Design efficient batch processing strategies |
| **4.6** | Design multi-instance and multi-pass review architectures |

---

<!-- layout: navigation -->
# Chapter 5

- **Explicit Criteria and Few-Shot Prompting**
- Structured Output and JSON Schemas
- Validation and Retry Loops
- Batch Processing and Multi-Pass Reviews

---

<!-- layout: 2-column -->
# Explicit Criteria Beat Vague Instructions

### Vague (Low Precision)
```text
Review the comments in this diff.
Be conservative and only report
high-confidence findings.
```
- “Conservative” and “high-confidence” mean nothing measurable
- Models calibrate self-reported confidence poorly on subtle review rules
- False positives stay roughly where they were

### Explicit (High Precision)
```text
Flag a comment ONLY when the behavior it
describes contradicts what the code does,
for example: "returns None on failure"
above code that raises ValueError.

Do NOT flag: stylistic wording, missing
comments, or comments that are incomplete
but not wrong.
```
- States exactly what to report and what to skip

---

<!-- layout: 2-column -->
# False Positives Erode Trust: Disable Noisy Categories

### The Damage
- One category (say, “naming suggestions”) produces 40% false positives
- Developers learn to skim past the bot’s comments
- They start ignoring the **accurate** security findings too, so one noisy category damages the whole system

### The Response
- **Temporarily disable** the high false-positive category in production
- Refine its prompt with explicit criteria and examples on a test set
- Re-enable it only when precision meets your bar
- Keep high-precision categories (injection, auth bypass) running the whole time

<!-- below-columns -->

> [!TIP]
> “Lower the overall confidence threshold” or “tell the model to be more careful” are distractors. The exam answer isolates the noisy category and fixes it with explicit criteria.

---

# Defining Severity with Code Examples

- **Severity labels are only consistent when each level has a concrete example**: “critical” means different things to different reviewers, including the model

| Severity | Definition in the Prompt | Example Shown to the Model |
| :--- | :--- | :--- |
| `critical` | Exploitable now, or causes data loss | `cursor.execute(f"SELECT * FROM users WHERE id={uid}")` |
| `error` | Wrong behavior in a reachable path | `if amount > 0 or amount < 100:` (should be `and`) |
| `warning` | Risky but not currently wrong | `requests.get(url)` with no `timeout=` argument |

- **Put the examples in the prompt**, next to the definitions, so the model anchors on them

---

<!-- layout: 2-column -->
# Few-Shot for Ambiguous Cases: Show the Reasoning

### Why Examples with Reasoning
- Rules cover the clear cases; the hard cases sit between two plausible choices
- 2 to 4 examples that explain **why** one choice beat the alternative teach the judgment, not just the label
- The model then generalizes to edge cases no rule anticipated

### Example
```text
Request: "Cancel my order and refund me."
Order status: shipped.
Choice: lookup_order, then explain the
return process. NOT process_refund.
Why: a shipped order cannot be cancelled;
the refund depends on the return, so
issuing it now would skip the policy.
```

---

<!-- layout: 2-column -->
# Few-Shot for Output Format Consistency

### The Problem
- Prose says “report location, issue, severity, and fix”
- Outputs vary: some use line ranges, some file names only, some merge issue and fix into one sentence
- Downstream parsing breaks on every variation

### Show the Exact Shape
```text
Example finding:
location: api/orders.py:88
issue: SQL query built with an f-string
severity: critical
suggested_fix: Use a parameterized query
  with cursor.execute(sql, (uid,))
```
- One or two examples like this fix formatting faster than more prose rules

---

<!-- layout: 2-column -->
# Few-Shot to Separate Acceptable Patterns from Real Issues

### The Problem
- The repo intentionally uses `except Exception: log_and_reraise()` in its job runner
- The reviewer flags it as “overly broad exception handling” in every PR
- Developers dismiss the same finding dozens of times

### Pair a Real Bug with an Accepted Idiom
```text
FLAG (real issue):
  except Exception:
      pass        # error silently lost

DO NOT FLAG (team convention):
  except Exception:
      log_and_reraise()  # logged and raised
```
- The contrast teaches the boundary, not just the keyword

---

<!-- layout: 2-column -->
# Few-Shot for Varied Document Structures

### The Problem
- Research papers cite sources inline (“Smith 2024”) in some files and in a bibliography in others
- Measurements appear in tables, in prose, or informally (“about two kilos”)
- With one layout in the examples, the model returns empty or invented fields for the others

### Cover the Variety
```text
Example A (table): "| Mass | 2.1 kg |"
  -> mass_kg: 2.1
Example B (prose): "weighed roughly 2 kg"
  -> mass_kg: 2.0, approximate: true
Example C (absent): no mass given
  -> mass_kg: null
```

---

<!-- layout: navigation -->
# Chapter 5

- Explicit Criteria and Few-Shot Prompting
- **Structured Output and JSON Schemas**
- Validation and Retry Loops
- Batch Processing and Multi-Pass Reviews

---

# Extraction Through a Tool Schema

- **Define the output as a tool, then read the tool call’s input**: the API returns the arguments as parsed JSON, so there is no text to parse and **no JSON syntax errors**
- **Strict mode**: add `"strict": true` so the input is also guaranteed to match the schema, not just parse as JSON

```python
extract_invoice = {
    "name": "extract_invoice",
    "description": "Record the invoice fields exactly as written in the document.",
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor": {"type": "string"},
            "invoice_number": {"type": "string"},
            "stated_total": {"type": "number"},
        },
        "required": ["vendor", "invoice_number", "stated_total"],
        "additionalProperties": False,
    },
}

response = client.messages.create(model="claude-haiku-5-5", max_tokens=1024,
                                  tools=[extract_invoice], messages=messages)
invoice = next(b.input for b in response.content if b.type == "tool_use")  # already a dict
```

> [!TIP]
> “Ask for JSON in the prompt and parse the text with a regex” is the classic distractor. The tool schema is the reliable mechanism.

---

<!-- layout: 2-column -->
# Syntax Errors vs Semantic Errors

### Syntax Errors: Solved by the Schema
- Missing braces, trailing commas, unquoted keys
- A string where a number belongs
- A required field left out
- `tool_use` with a schema (and strict mode) eliminates these

### Semantic Errors: Not Solved by the Schema
- Line items that do not add up to the stated total
- A ship-to address placed in the bill-to field
- A date that is valid ISO 8601 but is the wrong date
- These need **validation code** or a review step after extraction

<!-- below-columns -->

> [!IMPORTANT]
> A valid schema proves the data has the right **shape**, not that it is **right**. The next demo shows a schema-perfect extraction with an impossible total.

---

# Demo: Valid JSON, Wrong Total

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_08_semantic_check.py`**

**Step 2. Paste this code**

```python
# demo_08_semantic_check.py - the schema is valid, but the numbers disagree
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

INVOICE = """INVOICE 7781 - Harbor Office Supply
3 x Standing desk      @ $200.00
2 x Ergonomic chair    @ $320.00
TOTAL DUE: $1,450.00"""

tool = {
    "name": "record_invoice",
    "description": "Record an invoice exactly as written.",
    "input_schema": {
        "type": "object",
        "properties": {
            "line_items": {"type": "array", "items": {
                "type": "object",
                "properties": {
                    "description": {"type": "string"},
                    "quantity": {"type": "integer"},
                    "unit_price": {"type": "number"},
                },
                "required": ["description", "quantity", "unit_price"],
            }},
            "stated_total": {"type": "number"},
        },
        "required": ["line_items", "stated_total"],
    },
}

response = client.messages.create(
    model="claude-haiku-5-5",
    max_tokens=1028,
    output_config={"effort": "low"},
    tools=[tool],
    messages=[{"role": "user", "content": f"Call record_invoice for this invoice.\n\n{INVOICE}"}],
)
data = next(b for b in response.content if b.type == "tool_use").input

calculated = sum(i["quantity"] * i["unit_price"] for i in data["line_items"])
print("Extracted:", data)
print(f"stated_total:     ${data['stated_total']:,.2f}")
print(f"calculated_total: ${calculated:,.2f}")
print("conflict_detected:", abs(calculated - data["stated_total"]) > 0.01)
```

**Step 3. Run it**

```bash
python demo_08_semantic_check.py
```

**Step 4. Examine the output**

- The extracted data matches the schema: two line items and a `stated_total` of `1450.0`
- The line items add up to `$1,240.00`
- The last line prints `conflict_detected: True`

> [!NOTE]
> The schema guaranteed the shape of the data, not that it makes sense. Extracting the stated total and calculating your own lets code catch the mismatch and send the document to review.

---

<!-- layout: 3-column -->
# Choosing `tool_choice` for Extraction

### `"auto"`
- Claude may answer in text instead of calling the extraction tool
- Fine for chat; risky for a pipeline that expects structured output every time

### `"any"`
- Claude **must** call one of the provided tools, and it chooses which
- Use when several schemas exist (`extract_invoice`, `extract_receipt`, `extract_contract`) and the document type is unknown up front

### Forced Tool
- `{"type": "tool", "name": "extract_invoice"}`
- Claude must call **that** tool
- Use when the document type is known, or when one extraction must run before any enrichment

<!-- below-columns -->

> [!NOTE]
> On the exam, `"any"` and forced selection are correct answers. In current releases, they return a 400 error on Opus 5.5, Sonnet 5.5, Fable 5.1, and Mythos 5.1; use Claude Haiku 5.5 or Opus 5, or rely on `"auto"` with a system prompt that requires the tool call.

---

<!-- layout: 2-column -->
# Nullable Fields for Absent Data

### Required String: Forced Fabrication
```json
"po_number": {"type": "string"}
```
- The document has no PO number, but the schema demands a string
- The model fills it with something plausible: `"N/A"`, `"PO-0000"`, or an invented number
- Downstream systems treat the placeholder as real data

### Nullable: An Honest Answer
```json
"po_number": {
  "type": ["string", "null"],
  "description": "PO number if printed on the document; otherwise null"
}
```
- The model can return `null` when the value is absent
- Code checks for `null` and routes the record for follow-up

---

# Demo: A Required Field Invents a Value

In `ccarf-labs`, with the virtual environment still active:

**Step 1. Create `demo_09_nullable.py`**

**Step 2. Paste this code**

```python
# demo_09_nullable.py - a required field invents a value; a nullable field does not
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

DOCUMENT = """Purchase order from Lakeside Dental.
Ship 40 boxes of nitrile gloves to 12 Harbor Rd, Portland.
Deliver by Nov 20. Contact: Priya Shah."""

def extract(po_type):
    tool = {
        "name": "record_order",
        "description": "Record the fields of a purchase order.",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer": {"type": "string"},
                "po_number": {"type": po_type, "description": "The purchase order number."},
            },
            "required": ["customer", "po_number"],
        },
    }
    response = client.messages.create(
        model="claude-haiku-5-5",
        max_tokens=1028,
        output_config={"effort": "low"},
        system="You are a batch extraction job. Always call record_order. Never ask questions.",
        tools=[tool],
        messages=[{"role": "user",
                   "content": f"Call record_order for this document.\n\n{DOCUMENT}"}],
    )
    calls = [b for b in response.content if b.type == "tool_use"]
    if not calls:
        return "(no tool call) " + "".join(b.text for b in response.content if b.type == "text")
    return calls[0].input.get("po_number")

print("po_number as string:         ", repr(extract("string")))
print("po_number as string or null: ", repr(extract(["string", "null"])))
```

**Step 3. Run it**

```bash
python demo_09_nullable.py
```

**Step 4. Examine the output**

- The document has no PO number
- With `"string"`, Claude should fill the field anyway, typically with a placeholder such as `'NOT_PROVIDED'`
- With `["string", "null"]`, Claude should return `None`

> [!NOTE]
> A placeholder string looks like real data to every system downstream. Making the field nullable gives Claude an honest way to say the value is missing, and lets your code test for `None`.

---

<!-- layout: 2-column -->
# Enums with `unclear` and `other` Plus Detail

### `"unclear"` for Ambiguous Text
```json
"contract_type": {
  "enum": ["fixed_price", "time_and_materials",
           "retainer", "unclear"]
}
```
- Without `"unclear"`, an ambiguous clause is forced into the nearest category
- With it, ambiguous documents are labeled honestly and can be routed to review

### `"other"` Plus Detail for Novel Cases
```json
"payment_method": {
  "enum": ["card", "ach", "wire", "check", "other"]
},
"payment_method_other_detail": {
  "type": ["string", "null"]
}
```
- New cases (“crypto escrow”) are captured in the detail field instead of being mislabeled
- Reviewing the detail values tells you when to add a new enum value

---

<!-- layout: 2-column -->
# Format Normalization Rules in the Prompt

### Why the Schema Is Not Enough
- `{"type": "string"}` accepts `"Oct 30, 2026"`, `"30/10/26"`, and `"2026-10-30"` equally
- The schema checks types, not your team’s conventions
- Without rules, downstream code has to handle every format the source documents used

### Add Explicit Rules
```text
Normalization rules:
- Dates: YYYY-MM-DD. "30/10/26" in a UK
  document is 2026-10-30.
- Money: number in the document currency,
  no symbols. "$1,450.00" -> 1450.00
- Units: kilograms. "2,200 lb" -> 997.9
- If a value cannot be normalized
  confidently, return null.
```

---

# Extraction Tool Schema Example

- **Combining nullable fields, `"unclear"` enums, and `"other"` + detail strings**:

```json
{
  "name": "extract_contract_terms",
  "description": "Extracts key commercial terms from a vendor contract. Return null for any field not explicitly stated in the document.",
  "input_schema": {
    "type": "object",
    "properties": {
      "effective_date": { "type": ["string", "null"], "description": "ISO 8601 YYYY-MM-DD or null if unstated" },
      "governing_law": {
        "type": "string",
        "enum": ["delaware", "new_york", "california", "uk", "unclear", "other"]
      },
      "governing_law_other_detail": { "type": ["string", "null"] },
      "stated_total": { "type": ["number", "null"] },
      "calculated_total": { "type": ["number", "null"] },
      "conflict_detected": { "type": "boolean" }
    },
    "required": ["effective_date", "governing_law", "stated_total", "calculated_total", "conflict_detected"]
  }
}
```

---

<!-- layout: navigation -->
# Chapter 5

- Explicit Criteria and Few-Shot Prompting
- Structured Output and JSON Schemas
- **Validation and Retry Loops**
- Batch Processing and Multi-Pass Reviews

---

<!-- layout: title-image -->
# Validation-Retry Feedback Loops

![Validation and Retry Feedback Loop](images/ch05-validation-retry-loop.svg)

---

# Retrying with Specific Error Feedback

- **A retry works when Claude can see exactly what went wrong**: send the **original document**, the **failed extraction**, and the **exact validation error**, not just “try again”

```python
try:
    Invoice.model_validate(extracted)
except ValidationError as err:
    messages += [
        {"role": "assistant", "content": response.content},  # the failed extraction call
        {"role": "user", "content": [{
            "type": "tool_result",
            "tool_use_id": call.id,
            "is_error": True,
            "content": (
                f"Validation failed:\n{err}\n\n"
                "Re-read the invoice above and call extract_invoice again. "
                "Fix only the fields named in the error."
            ),
        }]},
    ]
    response = client.messages.create(model="claude-haiku-5-5", max_tokens=1024,
                                      tools=[extract_invoice], messages=messages)
```

> [!NOTE]
> The original document stays in `messages`, so the retry can check the source. Cap retries at 2 or 3, then route the record to human review.

---

<!-- layout: 2-column -->
# Self-Correction Fields

### The Pattern
- Extract the total **as printed** (`stated_total`)
- Also have Claude compute the total from the line items (`calculated_total`)
- Add a `conflict_detected` boolean, and verify it in code
- A mismatch becomes a data signal, not a silent error

### Example Output
```json
{
  "line_items": [
    {"qty": 3, "unit_price": 200.00},
    {"qty": 2, "unit_price": 320.00}
  ],
  "stated_total": 1450.00,
  "calculated_total": 1240.00,
  "conflict_detected": true
}
```
- Conflicts route to review instead of flowing into the ledger

---

<!-- layout: 2-column -->
# `detected_pattern` for False-Positive Analysis

### The Pattern
- Each review finding names the code construct that triggered it
- When developers dismiss findings, you can group dismissals by pattern
- The patterns with high dismissal rates are where your criteria or few-shot examples need work

### Example Finding
```json
{
  "location": "jobs/runner.py:42",
  "issue": "Broad exception handler",
  "severity": "warning",
  "detected_pattern": "except_exception_reraise",
  "suggested_fix": "Catch specific exceptions"
}
```
- Dismissal data: `except_exception_reraise` dismissed 31 of 33 times, so add an “acceptable pattern” example

---

<!-- layout: 2-column -->
# When Retries Succeed vs When Retries Fail

### Retryable Errors (Will Succeed)
- Format mismatches (such as returning `"10/30/2026"` instead of `"2026-10-30"`)
- Structural output errors or misplaced field values
- Arithmetic sum mismatches where the line items exist in the source text

### Non-Retryable Situations (Futile)
- The required information is **completely absent from the source document** (or lives in an external appendix that was never provided)
- Retrying when data is absent wastes tokens and pressures the model into hallucinating a value

---

<!-- layout: navigation -->
# Chapter 5

- Explicit Criteria and Few-Shot Prompting
- Structured Output and JSON Schemas
- Validation and Retry Loops
- **Batch Processing and Multi-Pass Reviews**

---

<!-- layout: 2-column -->
# Synchronous API vs Message Batches API

### Synchronous Messages API
- Immediate real-time response with full multi-turn `tool_use` loop support
- Required for **blocking workflows** where a human or CI pipeline is actively waiting (such as pre-merge PR checks or live support chat)

### Message Batches API
- **50% cost savings** with an asynchronous processing window of **up to 24 hours** (no guaranteed latency SLA)
- Correlates requests and responses via `custom_id`
- **Does NOT support multi-turn tool calling** within a single batch request
- Ideal for overnight technical debt reports, weekly audits, and bulk document extraction

---

# Batch Lifecycle: Create, Poll, Retrieve

- **A batch is asynchronous**: submit every request at once, poll until processing ends, then match each result to its request by `custom_id`

```python
batch = client.messages.batches.create(requests=[
    {"custom_id": f"doc-{doc.id}",
     "params": {"model": "claude-haiku-5-5", "max_tokens": 1024,
                "tools": [extract_invoice],
                "tool_choice": {"type": "tool", "name": "extract_invoice"},
                "messages": [{"role": "user", "content": doc.text}]}}
    for doc in documents
])

while client.messages.batches.retrieve(batch.id).processing_status != "ended":
    time.sleep(60)

failed = []
for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        save(entry.custom_id, entry.result.message)
    else:  # "errored", "canceled", or "expired"
        failed.append(entry.custom_id)
```

> [!NOTE]
> Results can come back in any order, which is why `custom_id` is required. Each request is a single turn: if Claude asks for a tool, the batch cannot run that tool and continue.

---

# Batch SLA Math

- **Work backward from the SLA**: a document can wait in your queue, then spend up to **24 hours** in the batch, and the total must fit inside the SLA

| Quantity | Value |
| :--- | :--- |
| Business SLA (document arrives to result ready) | 30 hours |
| Maximum batch processing window | 24 hours |
| Longest a document can wait before submission | 30 - 24 = **6 hours** |
| Submitting every 6 hours | Worst case exactly 30 hours: no margin |
| Submitting every **4 hours** | Worst case 4 + 24 = 28 hours: 2 hours of margin |

- **Exam pattern**: the answer is the submission interval that keeps the worst case inside the SLA; “submit once a day” and “switch to synchronous calls” are the usual distractors

---

<!-- layout: 2-column -->
# Resubmitting Failures by `custom_id`

### Results Excerpt
```json
{"custom_id": "doc-1041",
 "result": {"type": "succeeded", "...": "..."}}
{"custom_id": "doc-1042",
 "result": {"type": "errored",
  "error": {"type": "invalid_request_error",
   "message": "prompt is too long"}}}
{"custom_id": "doc-1043",
 "result": {"type": "expired"}}
```

### Recovery Rules
- Resubmit **only** the failed `custom_id` values; never rerun the whole batch
- Fix the cause before resubmitting: **chunk** `doc-1042` into sections that fit the context window
- Resubmit `doc-1043` unchanged; it expired without being processed
- Merge chunk results back under the original document ID

---

<!-- layout: 2-column -->
# Refine Prompts on a Sample Before Batching

### Batching an Untested Prompt
- 10,000 documents run overnight
- In the morning, 18% have a misread date format
- Fixing the prompt means paying for, and waiting for, a second full batch

### Sample First
1. Pick 50 to 100 documents that cover every layout and edge case
2. Run them synchronously and check the results against expected values
3. Fix the prompt and schema until the sample passes
4. Submit the full batch with the refined prompt

---

<!-- layout: 2-column -->
# Why Self-Review Fails: Use an Independent Instance

### Same-Session Self-Review
- The model that wrote the extraction or code still holds the reasoning that produced it
- Asked to check its work, it tends to re-justify its choices
- Errors rooted in a wrong assumption survive the review

### Independent Instance
- A second Claude call with **fresh context** receives only the artifact and the review criteria
- It evaluates what is there, not what was intended
- Use it for code review, extraction verification, and any check where the generator could be biased

---

<!-- layout: stacked -->
# Multi-Pass Review: Per-File Then Integration

- **One pass over 14 files dilutes attention**: comments become superficial, some files get skipped, and findings contradict each other
- **Pass 1: per-file local analysis**: each file reviewed on its own for bugs inside that file
- **Pass 2: cross-file integration**: one pass over the interfaces and data flow between modules, using the per-file summaries

![Multi-Pass Code Review Architecture](images/ch05-multi-pass-review.svg)

---

<!-- layout: 2-column -->
# Confidence-Scored Verification Pass

### The Pattern
- The reviewer reports a confidence level with each finding
- A verification pass, or your thresholds, decides what happens to each finding
- High confidence: post the comment directly
- Low confidence: route to a human reviewer instead of posting it

### Calibrate, Do Not Trust Raw Scores
- Raw self-reported confidence is poorly calibrated
- Measure it against findings that humans labeled correct or incorrect
- Set the threshold where posted findings meet your precision target

<!-- below-columns -->

> [!NOTE]
> This routes **review attention** for findings. It is not an escalation trigger for live customers; the exam treats self-reported confidence as an unreliable escalation signal.

---

# Domain 4 Quiz: Prompt Engineering and Structured Output

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-30-14-50-ccar-f-realistic-instructor.json)

---

# Lab 3: Structured Data Extraction and Validation Pipelines

**Time:** 60 minutes

- [Open Lab](https://labv.roitraining.com/?lab=https://github.com/roitraining/claude-exam-cert-prep-courses/tree/main/claude-certified-architect-foundations-exam-prep-workshop/labs/lab-03-structured-data-extraction-and-validation-pipelines)

---

# What You Learned

- Wrote explicit criteria and targeted few-shot examples
- Enforced structured JSON outputs using `tool_use` schemas
- Built validation-retry loops for extraction quality
- Designed Message Batches workflows and multi-pass reviews

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
