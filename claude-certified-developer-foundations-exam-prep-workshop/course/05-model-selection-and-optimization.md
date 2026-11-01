<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 5: Model Selection and Optimization

---

# Chapter 5: Objectives

- Apply tokenization, sampling, adaptive thinking, and effort
- Integrate client SDKs with REST, SSE streaming, and WebSockets
- Select models and manage breaking changes across releases
- Optimize token costs with Prompt Caching checkpoints

---

# Domain 5: Model Selection and Optimization

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
- Claude processes text in sub-word **tokens** (roughly 3 to 4 English characters); code, JSON keys, and images also consume tokens. Newer models use a newer tokenizer (up to about 35% more tokens for the same text): measure with the `count_tokens` endpoint.
- The context window (1M tokens on current models) covers **input + thinking + output (`max_tokens`)**.

### Next-Token Generation and Non-Determinism
- LLMs generate responses autoregressively one token at a time based on probability distributions.
- The same request can return different outputs, even at the lowest-variance settings: design for variance with schema validation, evals, and retries instead of expecting identical responses.

### Sampling Parameters (`temperature`, `top_p`, `top_k`)
- Concept: sampling picks each token from a probability distribution; lower `temperature` concentrates on likely tokens, higher adds diversity.
- Current API: the newest models reject non-default sampling values, and API version `2026-10-21` rejects them on every model. Steer with prompts, examples, structured outputs, and effort instead.

### Zero-Shot, Single-Shot, and Multi-Shot
- **Zero-shot** relies on instructions alone; **single-shot** provides 1 format example; **multi-shot (few-shot)** provides 2 to 5 examples to calibrate edge-case judgment.

---

<!-- layout: 2-column -->
# Extended Thinking, Adaptive Thinking, Effort, and Fast Mode

### Thinking: Concept vs Current API
- **Extended thinking** (concept): Claude reasons in `thinking` blocks before answering or calling a tool
- **Adaptive thinking** (`thinking: {"type": "adaptive"}`): Claude decides when and how much to think; it is the only mode on current models (Claude 4.6 and later)
- Fixed `budget_tokens` budgets are legacy: removed (HTTP 400) on the Claude 5 family; still used by Haiku 4.5

### Effort and Fast Mode
- **Effort** (`output_config: {"effort": ...}`): `low`, `medium`, `high`, `xhigh`, `max`; trades thoroughness for tokens and latency (default `high`; `medium` on Opus 5.5 and Haiku 5.5)
- Use `low` for subagents and simple routes, `high` or `xhigh` for hard coding and agentic work
- **Fast mode** (research preview, Claude API only, Opus only): same model at up to 2.5x output speed for premium pricing

---

<!-- layout: navigation -->
# Chapter 5

- LLM Fundamentals
- **Technical Fundamentals**
- Model Selection and Trade-Offs
- Cost and Token Management

---

# Integrating with Official Client SDKs

- **Use the official SDKs** (`anthropic`, `@anthropic-ai/sdk`) over raw HTTP: connection pooling, automatic retries on `429`/`5xx`, typed models, and stream helpers

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
- Persistent, full-duplex bi-directional connection between client applications and your backend agent server
- Ideal when your agent harness needs to push live intermediate status updates, accept mid-flight user cancellations, or coordinate interactive client-side tool approvals

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

### Claude Opus (`claude-opus-5-5`)
- **Highest capability**: complex architecture, subtle root-causing, long-horizon agents
- $4 / $20 per million input / output tokens
- Planning, coordinator synthesis, hard reasoning (Fable 5.1 sits above, at higher cost)

### Claude Sonnet (`claude-sonnet-5-5`)
- **Balance** of coding, agentic tool use, and speed
- $2 / $10 per million tokens
- Strong default for production coding and agents

### Claude Haiku (`claude-haiku-5-5`)
- **Fastest and lowest cost**: $0.10 / $0.50 per million tokens (prompts up to 100K)
- Classification, routing, guardrails, extraction, subagents
- Supports adaptive thinking; weaker on ambiguous multi-hop reasoning

---

<!-- layout: 2-column -->
# Managing Breaking Changes Across Model Releases

### API-Breaking Changes (HTTP 400)
- `thinking.budget_tokens` removed: use adaptive thinking plus effort
- Assistant-message **prefill** removed: use structured outputs
- Forced `tool_choice` (`any` / `tool`) removed on the newest models: use `auto` with `strict: true` tools or structured outputs
- `temperature` / `top_p` / `top_k` rejected on current models

### Behavior Changes (Same Schema)
- New tokenizer changes token counts and cost
- New `refusal` stop reason from safety classifiers (HTTP 200)
- **Upgrade workflow**: pin the model ID, run evals on the new model, compare tool selection, schema compliance, latency, and cost, then shift traffic

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
- **Cache checkpoints**: up to 4 `cache_control: {"type": "ephemeral"}` breakpoints on the stable prefix (render order `tools` -> `system` -> `messages`), or one top-level `cache_control` that moves forward automatically as the conversation grows
- **Advancing the conversation checkpoint**: in long agent loops, caching earlier turns means you pay full price only for the newest turn

![Prompt Caching Checkpoints and Token Accounting](images/ch05-token-caching-checkpoints.svg)

---

# Prompt Caching Rules That Show Up in Scenarios

| Rule | Detail | Exam Implication |
| :--- | :--- | :--- |
| **Exact prefix match** | Any byte change invalidates everything after it | A timestamp at the top of `system` kills the cache |
| **TTL** | 5 minutes by default (refreshed on each hit); 1 hour optional | Use 1 hour only when requests on the same prefix are 5 to 60 minutes apart |
| **Pricing** | Writes 1.25x base input (2x for 1-hour); reads about 0.1x | Pays off from the second request on a 5-minute cache |
| **Minimum length** | 512 to 4,096 tokens depending on model | Short prompts silently do not cache |
| **Scope** | Caches are per model and per organization | Switching models mid-conversation starts a new cache |
| **Verification** | `cache_read_input_tokens` stays 0 | A silent invalidator (dynamic text, unsorted JSON, changing tool list) |

---

# Domain 5 Quiz: Model Selection and Optimization

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-14-ccdv-f-realistic-instructor.json)

---

# What You Learned

- Applied tokenization, sampling, adaptive thinking, and effort
- Integrated client SDKs with REST, SSE streaming, and WebSockets
- Selected models and managed breaking changes across releases
- Optimized token costs with Prompt Caching checkpoints

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
