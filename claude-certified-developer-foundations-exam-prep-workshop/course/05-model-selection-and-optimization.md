<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 5: Model Selection and Optimization

---

# Chapter 5: Objectives

- Apply tokenization, `count_tokens`, non-determinism, adaptive thinking, effort, and fast mode
- Integrate client SDKs with REST, SSE streaming, and full-duplex WebSockets
- Select and route across Opus, Sonnet, and Haiku tiers and manage breaking model upgrades
- Model workload costs and configure Prompt Caching checkpoints and TTLs

---

# Domain 5: Model Selection and Optimization (16.8% of Exam)

| Task # | Task Statement |
| :--- | :--- |
| **5.1** | LLM Fundamentals: Tokens, context windows, sampling, non-determinism, model options (fast mode, extended/adaptive thinking, effort levels), and prompting |
| **5.2** | Technical Fundamentals: Integrating with client SDKs that wrap REST APIs and WebSockets |
| **5.3** | Model Selection and Trade-Offs: Opus vs Sonnet vs Haiku use cases, quality/latency/cost trade-offs, and breaking behavior changes across releases |
| **5.4** | Cost and Token Management: Token usage tracking, cost modeling, Prompt Caching, and cache checkpointing |

---

<!-- layout: navigation -->
# Chapter 5

- **LLM Fundamentals**
- Technical Fundamentals
- Model Selection and Trade-Offs
- Cost and Token Management

---

<!-- layout: card-layout -->
# Core LLM Mechanics for Developers

### Tokenization and Context Windows
- Claude processes text in sub-word **tokens** (roughly 3 to 4 English characters); code, JSON keys, tool schemas, and images also consume tokens.
- Current models use a newer tokenizer (introduced with Opus 4.7) that produces up to ~35% more tokens for the same text than older generations: always measure with `client.messages.count_tokens()`.
- The **1M token context window** bounds **input + thinking + output (`max_tokens`)** within a single request.

### Token Generation and Non-Determinism
- LLMs generate responses autoregressively one token at a time by sampling from probability distributions.
- Identical requests can return slightly different wording across calls: design production systems around structured outputs, schema validation, and eval suites rather than expecting byte-identical strings.

### Sampling Parameters 
- Concept: sampling parameters shape the probability distribution (`temperature` scales logit sharpness; `top_p` and `top_k` truncate the tail).
- Current API rule: the Claude 5 model family and API version `2026-10-21` **reject non-default sampling values with HTTP 400**. Steer output consistency with prompts, few-shot examples, structured outputs, and `effort` instead.

### Zero-Shot, Single-Shot, and Multi-Shot
- **Zero-shot** relies on instructions alone; **single-shot** provides 1 format example; **multi-shot (few-shot)** provides 2 to 5 labeled examples to calibrate ambiguous edge-case judgment.

---

# Measuring Exact Token Budgets with `count_tokens()`

- **Preflight token accounting**: call `client.messages.count_tokens()` with the exact same `model`, `system`, `tools`, `thinking`, and `messages` before sending a large request to verify it fits your context budget or meets the 512-token cache minimum

```python
token_estimate = await client.messages.count_tokens(
    model="claude-sonnet-5-5",
    system=SYSTEM_POLICY,
    tools=REGISTERED_TOOLS,
    thinking={"type": "adaptive"},
    messages=conversation_history,
)

print(f"Preflight input tokens: {token_estimate.input_tokens}")
if token_estimate.input_tokens > 850_000:
    conversation_history = await compact_history(conversation_history)
```

---

# Extended Thinking and Adaptive Thinking

- **Extended thinking** (concept): Claude reasons step by step in `thinking` blocks before answering or calling a tool
- **Adaptive thinking** (`thinking: {"type": "adaptive"}`): Claude dynamically decides when and how much to think based on task complexity; it is always on for Opus 5.5 / Fable 5.1 and the standard mode across the Claude 5 family
- Fixed `thinking.budget_tokens` is legacy: rejected (HTTP 400) on the Claude 5 family; still used only on older models such as Haiku 4.5
- **`between_tools` mode** (`thinking: {"type": "between_tools"}`): supported on models like Sonnet 5.5 to skip up-front thinking on turn 1 while still thinking between tool calls

---

# Effort Levels and Fast Mode

- **Effort** (`output_config: {"effort": ...}`): `low`, `medium`, `high`, `xhigh`, `max`
- Default effort is `high` on Sonnet 5.5 and Fable 5.1, and `medium` on Opus 5.5 and Haiku 5.5
- Lower effort (`low` or `medium`) cuts thinking token spend and latency on simple queries. Higher effort (`high`, `xhigh`, or `max`) deepens reasoning on hard debugging or architecture tasks
- **Fast mode** (research preview, Claude API only, Opus only): runs the **exact same Opus model weights** at up to **2.5x faster output speed** for premium pricing (does not switch to a smaller model)

---

# Choosing the Right `effort` Level by Workload

| `output_config.effort` | Thinking Token Spend and Latency | Best-Fit Developer Workloads |
| :--- | :--- | :--- |
| **`"low"`** | Minimal or zero thinking tokens; lowest latency | High-volume classification, routing, simple FAQ answers, lightweight extraction subagents |
| **`"medium"`** | Balanced reasoning for multi-step tasks (default on Opus 5.5 and Haiku 5.5) | Standard tool-use turns, doc summarization, routine code edits where speed matters |
| **`"high"`** | Thorough step-by-step reasoning (default on Sonnet 5.5 and Fable 5.1) | Complex coding, multi-file debugging, root-cause trace analysis, synthesis across tools |
| **`"xhigh"` / `"max"`** | Maximum reasoning depth; highest latency and output token consumption | Hardest architectural migrations, subtle security vulnerability audits, ambiguous legal/financial proofs |

> [!TIP]
> You do not need to switch models for every query: a single `claude-sonnet-5-5` endpoint can serve fast syntax answers at `effort: "low"` and deep multi-step debugging at `effort: "high"` or `"xhigh"`.

---

<!-- layout: navigation -->
# Chapter 5

- LLM Fundamentals
- **Technical Fundamentals**
- Model Selection and Trade-Offs
- Cost and Token Management

---

# Integrating with Official Client SDKs

- **Use the official SDKs** (`anthropic` in Python, `@anthropic-ai/sdk` in TypeScript) over raw HTTP: connection pooling, automatic retries on `429`/`5xx`, typed models, and stream helpers

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ maxRetries: 3, timeout: 60_000 }); // timeout in ms

const message = await client.messages.create({
  model: "claude-sonnet-5-5",
  max_tokens: 16000,
  thinking: { type: "adaptive" },
  output_config: { effort: "medium" },
  messages: [{ role: "user", content: "Design an idempotent retry key schema." }],
});
```

---

<!-- layout: 2-column -->
# REST + SSE Streaming vs WebSockets

### REST + Server-Sent Events (SSE)
- Unidirectional server-to-client stream over standard HTTP (`POST /v1/messages` with `stream: true`)
- Firewall-friendly, stateless per turn, and natively supported by the Claude Messages API SDKs
- Ideal for standard chat UIs, code generation streaming, and single-turn tool calls

### WebSockets in Agent Architectures
- Persistent, full-duplex bi-directional connection between client applications (browser/IDE) and **your backend agent harness**
- Ideal when users need to push mid-flight tool cancellations, respond to interactive tool-approval prompts, and receive live intermediate status updates while your backend calls the Claude API via the SDK

<!-- below-columns -->

> [!IMPORTANT]
> Never call the Claude API directly from a browser over SSE or WebSockets with a bundled API key. Terminate the client’s WebSocket or HTTP connection on **your authenticated backend**, which holds the API credentials server-side.

---

<!-- layout: navigation -->
# Chapter 5

- LLM Fundamentals
- Technical Fundamentals
- **Model Selection and Trade-Offs**
- Cost and Token Management

---

<!-- layout: 3-column -->
# Selecting Between Opus, Sonnet, and Haiku

### Claude Opus
- **Deep reasoning and long-horizon coding**: complex architecture, subtle root-causing, multi-agent coordinator synthesis
- **$4 / $20** per million input / output tokens (1M context, 128K max output; default effort `medium`)
- **Fable 5.1** (`claude-fable-5-1`, $10 / $50) sits above Opus 5.5 for the hardest reasoning tasks

### Claude Sonnet
- **Best balance of speed and intelligence**: production coding, multi-step tool-use agents, customer-facing workflows
- **$2 / $10** per million input / output tokens (1M context, 128K max output; default effort `high`)
- Strong default workhorse for interactive agents

### Claude Haiku
- **Fastest and lowest cost**: **$0.10 / $0.50** per million tokens (for prompts up to 100K; 1M context, 128K max output; default effort `medium`)
- High-volume classification, ticket routing, input guardrails, entity extraction, and lightweight subagents
- Supports adaptive thinking; trade-off is shallower multi-hop reasoning on ambiguous problems

---

# Tiered Model Routing and Cascade Pattern

- **Optimize cost and latency across mixed traffic**: use **Haiku 5.5** at `low` effort to classify or handle simple FAQ queries in milliseconds, and route complex debugging or multi-step policy disputes to **Sonnet 5.5** or **Opus 5.5**

```python
class TriageDecision(BaseModel):
    complexity: Literal["simple_faq", "standard_agent", "deep_escalation"]
    direct_answer: str | None = None

async def route_and_answer(user_query: str) -> str:
    triage = (await client.messages.parse(
        model="claude-haiku-5-5", max_tokens=512,
        output_config={"effort": "low"},
        output_format=TriageDecision,
        messages=[{"role": "user", "content": user_query}],
    )).parsed_output
    if triage.complexity == "simple_faq" and triage.direct_answer:
        return triage.direct_answer  # 80% of traffic resolved at $0.10 / MTok
    target_model = "claude-opus-5-5" if triage.complexity == "deep_escalation" else "claude-sonnet-5-5"
    return await run_specialist_agent(model=target_model, query=user_query)
```

---

<!-- layout: 2-column -->
# Managing Breaking Changes Across Model Releases

### Legacy Code That Fails with HTTP 400
```python
# Fails on claude-sonnet-5-5 (API 2026-10-21)
resp = await client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=4096,
    temperature=0.2,                  # 400: rejected
    thinking={
        "type": "enabled",
        "budget_tokens": 4096,        # 400: rejected
    },
    tool_choice={"type": "any"},      # 400: rejected
    messages=[
        {"role": "user", "content": prompt},
        {"role": "assistant", "content": "{"}, # 400: no prefill
    ],
)
```

### Migrated Claude 5 Code
```python
# Succeeds on claude-sonnet-5-5
resp = await client.messages.parse(
    model="claude-sonnet-5-5",
    max_tokens=4096,
    thinking={"type": "adaptive"},
    output_config={"effort": "medium"},
    output_format=ExtractedReport,    # replaces prefill
    messages=[
        {"role": "user", "content": prompt},
    ],
)
# Also verify: new tokenizer (+~35% tokens)
# and check resp.stop_reason == "refusal"
```

---

# Querying Model Capabilities Programmatically

- **Avoid hardcoding assumptions across model generations**: query the **Models API** (`client.models.list()`) to inspect `max_input_tokens`, `max_tokens`, model `line`, and supported `thinking` or `server_tools` capabilities at startup

```python
models_page = await client.models.list()
for m in models_page.data:
    print(
        m.id,
        "line=", m.line,
        "max_in=", m.max_input_tokens,
        "can_disable_thinking=", m.capabilities.thinking.types.disabled.supported,
        "between_tools=", m.capabilities.thinking.types.between_tools.supported,
    )
```

---

<!-- layout: navigation -->
# Chapter 5

- LLM Fundamentals
- Technical Fundamentals
- Model Selection and Trade-Offs
- **Cost and Token Management**

---

<!-- layout: stacked -->
# Token Budgeting and Cache Checkpointing

- **Track all four token counters** in `response.usage`: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`
- **Two ways to cache**:
  - **Automatic caching**: set top-level `cache_control: {"type": "ephemeral"}` on the request; the API automatically moves the breakpoint forward as a multi-turn conversation grows
  - **Explicit cache breakpoints**: place up to **4** `cache_control` markers on individual `tools`, `system`, or `messages` blocks (evaluated in exact order `tools` -> `system` -> `messages`)

![Prompt Caching Checkpoints and Token Accounting](images/ch05-token-caching-checkpoints.svg)

---

# Explicit 1-Hour Prefix Cache + Automatic Turn Caching

- **Combine a 1-hour TTL on static tools/system with a 5-minute rolling cache on conversation turns**: remember that `1h` cache entries must appear **before** `5m` entries

```python
response = await client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=4096,
    cache_control={"type": "ephemeral"},  # 5m rolling breakpoint on latest conversation turn
    tools=[
        {**TOOL_SCHEMA_A},
        {**TOOL_SCHEMA_B, "cache_control": {"type": "ephemeral", "ttl": "1h"}},  # Bp 1: tools (1h)
    ],
    system=[{
        "type": "text",
        "text": STATIC_COMPANY_POLICY_20K_TOKENS,
        "cache_control": {"type": "ephemeral", "ttl": "1h"},  # Bp 2: system policy (1h)
    }],
    messages=conversation_history,
)
```

---

# Prompt Caching Rules That Show Up in Scenarios

| Rule | Detail | Exam Implication |
| :--- | :--- | :--- |
| **Render order and prefix match** | Prefix renders in exact order: **`tools` -> `system` -> `messages`** | Reordering keys in `tools` or putting a timestamp at the top of `system` invalidates everything after it |
| **TTL options** | **5 minutes** by default (refreshed on every hit); **1 hour** (`"ttl": "1h"`) optional | Use 5-min TTL when turns arrive every few minutes; use 1-hour TTL when calls are 5 to 60 minutes apart |
| **TTL ordering** | When mixing TTLs in one request, **1-hour breakpoints must appear before 5-minute breakpoints** | Put `1h` on static `tools`/`system` and `5m` on growing `messages` history |
| **Pricing multipliers** | 5m write = **1.25x** base input; 1h write = **2.0x**; cache read = **0.1x** (0.05x on Opus 5.5 / Sonnet 5.5; 0.025x on Fable 5.1) | Pays off from the second request on a 5-minute cache |
| **Minimum length** | **512 tokens** on Claude 5 / 5.5 models (1,024 to 4,096 on older 4.x models) | Prompts shorter than the minimum silently do not cache (`cache_creation_input_tokens == 0`, no error) |
| **Isolation scope** | Caches are isolated per **model** and per **organization/workspace** | Switching from Sonnet to Haiku mid-session starts a separate cache |

---

# Demo: A Cache Miss, Then a Cache Hit

In `ccdvf-labs`, with the virtual environment still active:

**Step 1. Create `demo_04_cache.py`**

**Step 2. Paste this code**

```python
# demo_04_cache.py - same prompt twice, so the second call can read the cache
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

# Long enough to pass the 512-token cache minimum on Claude 5.
policy = "\n".join(
    f"Rule {i}: refunds over 50 dollars need a manager when the order is older than 14 days."
    for i in range(1, 40)
)

def call_once(label):
    response = client.messages.create(
        model="claude-haiku-5-5",
        max_tokens=64,
        system=[{
            "type": "text",
            "text": policy,
            "cache_control": {"type": "ephemeral"},
        }],
        messages=[{
            "role": "user",
            "content": "In one sentence, when does a refund need a manager?",
        }],
    )
    usage = response.usage
    print(label)
    print("  cache_creation_input_tokens:", usage.cache_creation_input_tokens)
    print("  cache_read_input_tokens:", usage.cache_read_input_tokens)

call_once("Call 1")
call_once("Call 2")
```

**Step 3. Run it once. The script sends both calls back to back.**

```bash
python demo_04_cache.py
```

**Step 4. Examine the output**

- Call 1 has `cache_creation_input_tokens` above 0 and `cache_read_input_tokens` at 0
- Call 2 has `cache_read_input_tokens` above 0

> [!NOTE]
> A cache hit is a usage field. `cache_read_input_tokens` stays 0 when the prefix differs by even one character, which is why a timestamp at the top of the system prompt breaks caching.

---

# Worked Cost: Caching a Live Support Agent

- **Live support agent on `claude-sonnet-5-5`** ($2 / MTok input, $10 / MTok output): 10,000 requests/day sharing an identical **10,000-token** system and tool prefix, plus **500** unique user input tokens and **300** output tokens per call
- **Uncached input**: 10,000 × 10,500 = 105 MTok × $2 = $210/day
- **Cached** (99.9% hit rate at 0.05×, which is $0.10/MTok on Sonnet 5.5):
  - Prefix reads: 10,000 × 10,000 = 100 MTok × $0.10 = $10/day
  - Unique input: 10,000 × 500 = 5 MTok × $2 = $10/day
- **Input cost drops from $210/day to about $20/day (90% reduction)**

---

# Worked Cost: Batching a Nightly Analytics Job

- **Nightly analytics job on `claude-sonnet-5-5`**: 20,000 archived transcripts/night, **2,000** input tokens and **500** output tokens each
- **Realtime**: (40 MTok × $2) + (10 MTok × $10) = $80 + $100 = $180/night
- **Message Batches API (50% off)**: **$90/night**, and the discount stacks with prompt caching when the transcripts share a policy prefix

---

# Domain 5 Quiz: Model Selection and Optimization

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-14-ccdv-f-realistic-instructor.json)

---

# What You Learned

- Applied tokenization, `count_tokens`, non-determinism, adaptive thinking, effort, and fast mode
- Integrated client SDKs with REST, SSE streaming, and full-duplex WebSockets
- Selected and routed across Opus, Sonnet, and Haiku tiers and managed breaking model upgrades
- Modeled workload costs and configured Prompt Caching checkpoints and TTLs

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
