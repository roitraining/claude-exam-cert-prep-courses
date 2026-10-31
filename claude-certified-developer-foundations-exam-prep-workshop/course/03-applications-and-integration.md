<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 3: Applications and Integration

---

# Chapter 3: Objectives

- Map functional and infrastructure requirements across the SDLC
- Integrate the Messages API, streaming, and Message Batches API
- Apply async I/O, resilient retries, and refactoring patterns
- Manage session boundaries and pinned model configurations

---

# Domain 2: Applications and Integration

| Task # | Task Statement |
| :--- | :--- |
| **2.1** | Understanding Requirements: Functional and infrastructure requirements based on business and solution architecture |
| **2.2** | Systems Life Cycle: Concepts and frameworks to develop, implement, operate, and maintain IT systems |
| **2.3** | Claude API Mechanics: Messages, tools, streaming, vision, thinking, caching, third-party vendors, and Realtime vs Batch APIs |
| **2.4** | Software Engineering Foundations: REST APIs, JSON, async programming, version control, SDLC, code review, and refactoring |
| **2.5** | Claude Application Design: Instruction interpretation across interfaces, content boundaries, schema design, and session hygiene |
| **2.6** | Configuration Management: `CLAUDE.md`, `settings.json`, model version pinning, prompt versioning, and plugin dependencies |

---

<!-- layout: navigation -->
# Chapter 3

- **Requirements and Systems Life Cycle**
- Claude API Mechanics
- Software Engineering Foundations
- Application Design and Configuration

---

<!-- layout: card-layout -->
# Translating Requirements Across the Systems Life Cycle

### Functional vs Infrastructure Requirements
- **Functional requirements** define what the AI application must accomplish: supported input formats (text, images, PDFs), tool integrations, output schemas, and escalation rules.
- **Infrastructure requirements** define latency SLAs (TTFT vs end-to-end), concurrency/rate-limit capacity, VPC egress rules, data residency, and secret storage.

### Systems Life Cycle (SDLC) Phases
- **Planning and Discovery**: define success metrics and build the initial golden evaluation dataset.
- **Development and Integration**: implement stateless API handlers, typed schemas, and deterministic hooks.
- **Testing and Deployment**: gate CI/CD merges on automated evaluations and roll out via canary/shadow stages.
- **Operations and Maintenance**: monitor token usage, cache hit rates, latency p95, and model deprecations.

---

<!-- layout: navigation -->
# Chapter 3

- Requirements and Systems Life Cycle
- **Claude API Mechanics**
- Software Engineering Foundations
- Application Design and Configuration

---

<!-- layout: stacked -->
# Claude Messages API Mechanics

- **Stateless API contract**: the Messages API does not store conversation state on the server; your application must pass the required conversation history in the `messages` array on every request
- **Top-level `system` parameter**: pass system instructions in the top-level `system` field rather than as a `"system"` role inside `messages`
- **Multi-format content blocks**: a single message’s `content` can contain an array of typed blocks (`text`, `image` base64/URL, `document` PDF, `tool_use`, `tool_result`, `thinking`)

![Claude Messages API Mechanics](images/ch03-messages-api-mechanics.svg)

---

# Streaming, Vision, and Multi-Turn `tool_result` Rules

- **Server-Sent Events (SSE) streaming**: use `client.messages.stream(...)` to receive incremental events (`message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`) for low perceived latency
- **Strict `tool_use` / `tool_result` pairing**: whenever an `"assistant"` message contains one or more `tool_use` blocks, the **immediately following `"user"` message** must contain the matching `tool_result` blocks (referenced by `tool_use_id`) before any new user text

```python
async with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a production log analyzer.",
    messages=[{"role": "user", "content": "Summarize this stack trace."}],
) as stream:
    async for text in stream.text_stream:
        print(text, end="", flush=True)
```

---

<!-- layout: 3-column -->
# Invoking Claude via First-Party and Cloud Vendors

### Anthropic Direct API
- Uses `Anthropic()` or `AsyncAnthropic()` from the `anthropic` SDK
- Authenticates via `ANTHROPIC_API_KEY`
- Fastest access to newly released platform features and headers

### Amazon Bedrock
- Uses `AnthropicBedrock()` client wrapper or AWS `boto3` Bedrock Runtime
- Authenticates via **AWS IAM roles / SigV4** credentials (`AWS_REGION`)
- Ideal when enterprise workloads and VPC controls already live in AWS

### Google Cloud Vertex AI
- Uses `AnthropicVertex(project_id=..., region=...)` client wrapper
- Authenticates via **Google Cloud Application Default Credentials (ADC) / IAM**
- Keeps traffic within GCP project boundaries and billing

---

<!-- layout: 2-column -->
# Realtime Messages API vs Message Batches API

### Realtime Messages API
- Synchronous request/response (or SSE stream) with immediate execution
- Supports interactive multi-turn `tool_use` loops
- **Use when**: a user or blocking CI pipeline is waiting on the result right now

### Message Batches API
- Asynchronous processing of up to 100,000 requests per batch; completes within **24 hours** at a **50% cost discount**
- Uses `custom_id` on each item to correlate results (since batch results may return in any order)
- **Use when**: running overnight analytics across 10,000 documents, bulk evaluations, or non-urgent backfills

<!-- below-columns -->

> [!IMPORTANT]
> Exam Rule: Sending 10,000 requests in parallel via the synchronous Messages API or lowering `max_tokens` does **not** unlock the 50% batch discount. Always select the **Message Batches API** for latency-tolerant bulk jobs.

---

<!-- layout: navigation -->
# Chapter 3

- Requirements and Systems Life Cycle
- Claude API Mechanics
- **Software Engineering Foundations**
- Application Design and Configuration

---

<!-- layout: card-layout -->
# Software Engineering Foundations for AI Apps

### Asynchronous Programming (`AsyncAnthropic`)
- LLM calls are network I/O bound (often 1 to 15 seconds).
- Use `AsyncAnthropic` with `asyncio.gather` (or `Promise.all` in TypeScript) with bounded semaphores so concurrent requests never block the event loop.

### Resilient REST Integration
- Configure connection timeouts, idempotency, and exponential backoff with jitter for transient `429` (rate limit) and `529` (overloaded) responses.
- Respect the `retry-after` HTTP response header when present.

### Small-Scale vs Large-Scale Refactoring
- Use direct targeted edits (`Edit` tool) for localized single-file refactoring.
- Use Plan Mode, dependency mapping, and incremental per-module verification for large-scale cross-repository refactoring.

### SDLC and Code Review Integration
- Treat prompts, tool schemas, and evaluation datasets as version-controlled code artifacts reviewed in pull requests alongside application code.

---

<!-- layout: navigation -->
# Chapter 3

- Requirements and Systems Life Cycle
- Claude API Mechanics
- Software Engineering Foundations
- **Application Design and Configuration**

---

<!-- layout: card-layout -->
# How Claude Interprets Instructions Across Interfaces

### Claude Code (CLI / IDE)
- Automatically loads the `CLAUDE.md` hierarchy, `.claude/rules/` path globs, `settings.json`, and local/project MCP servers.
- Operates directly on the local workspace with built-in file and bash tools.

### Claude Desktop and claude.ai
- **Claude Desktop** connects to local MCP servers configured in `claude_desktop_config.json`.
- **claude.ai** uses Project instructions, uploaded knowledge files, and Artifacts within a managed web session.

### API and Client SDKs
- Completely **stateless** and zero-magic: Claude only sees the exact `system` string, `tools` list, and `messages` array you send in that HTTP request.
- Does **not** automatically read `CLAUDE.md` unless your code explicitly loads and injects it.

---

<!-- layout: 2-column -->
# Session Hygiene and Configuration Management

### Content Boundaries and Session Hygiene
- Isolate per-user conversation histories and tool execution state so Tenant A’s data never leaks into Tenant B’s session
- Wrap external retrieved data or user uploads inside explicit XML delimiters (`<user_document>`) to maintain strict content boundaries
- Sanitize and validate all model outputs against strict JSON schemas before persisting

### Configuration Management and Version Pinning
- **Pin exact model versions** in production (for example, `claude-sonnet-4-20250514` rather than `-latest` aliases) so silent model updates cannot alter production behavior unexpectedly
- Version-control `CLAUDE.md`, `settings.json`, prompt templates, and plugin dependencies so any deployment can be reproduced or rolled back cleanly

---

# Domain 2 Quiz: Applications and Integration

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-12-11-ccdv-f-realistic-instructor.json)

---

# What You Learned

- Mapped functional and infrastructure requirements across the SDLC
- Integrated the Messages API, streaming, and Message Batches API
- Applied async I/O, resilient retries, and refactoring patterns
- Managed session boundaries and pinned model configurations

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
