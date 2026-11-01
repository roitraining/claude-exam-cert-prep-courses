<!-- course-title: Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Developer - Foundations: Hands-On Exam Prep Workshop

# Chapter 1: Claude Certified Developer - Foundations Overview

---

# Chapter 1: Objectives

- Review the CCDV-F exam format, scoring, and domain weights
- Evaluate the Minimally Qualified Candidate developer profile
- Analyze canonical CCDV-F sample question patterns
- Apply an engineering-focused exam-day strategy

---

<!-- layout: navigation -->
# Chapter 1

- **Understanding the Exam**
- Preparing to Pass the Exam

---

<!-- layout: card-layout -->
# Exam Details at a Glance (CCDV-F)

### Format and Time Limit
- The CCDV-F exam contains 53 multiple-choice and multiple-response items completed within 120 minutes.
- Each multiple-response item states explicitly how many answer choices to select.

### Scoring Standard
- Criterion-referenced assessment reported on a scaled range from 100 to 1,000 with a passing cut score of 720.
- Your pass or fail status is determined by your total scaled score, accompanied by domain-level percentage breakdowns.

### Exam Fee and Validity
- The exam fee is $125 USD, and the credential is valid for 12 months from the date awarded.
- On-time renewal requires completing a free, non-proctored assessment on the Anthropic Partner Academy.

### Delivery Environment
- Delivered via Pearson VUE (online proctored or test center) under strict non-disclosure and clear-workspace rules.

---

<!-- layout: stacked -->
# Exam Blueprint Across Eight Domains

- **Domain 2: Applications and Integration (33.1%)** is by far the largest section, accounting for roughly one-third of all scored items
- **Domain 5: Model Selection and Optimization (16.8%)** and **Domain 1: Agents and Workflows (14.7%)** together represent nearly another third
- **Prompt and Context Engineering (11.0%)**, **Tools and MCPs (10.6%)**, **Security and Safety (8.1%)**, **Claude Code (3.1%)**, and **Eval, Testing, and Debugging (2.6%)** complete the blueprint

![CCDV-F Exam Domain Weights](images/ch01-ccdvf-domain-weights.svg)

---

<!-- layout: 2-column -->
# Minimally Qualified Candidate (MQC) Profile

### Software Engineering Profile
- 1 to 5 years of software engineering experience with proficiency in **Python** and/or **TypeScript**
- At least 6 months of hands-on experience building with Claude or comparable LLM systems
- Fluency with REST APIs, JSON schemas, asynchronous programming, version control, and CLI tools

### Core Developer Competencies
- Building custom agent loops and workflows with the **Claude Agent SDK** and agentic frameworks (**PydanticAI**, **LangGraph**, **Strands**)
- Integrating the **Messages API** and **Message Batches API** across direct and cloud-provider SDKs
- Operating **Claude Code** (`CLAUDE.md`, `settings.json`, Rules, Skills, Commands, headless mode)
- Implementing custom tools, **MCP servers** (`stdio` and Streamable HTTP), prompt caching, and security hooks

---

<!-- layout: navigation -->
# Chapter 1

- Understanding the Exam
- **Preparing to Pass the Exam**

---

<!-- layout: card-layout -->
# Three Canonical CCDV-F Sample Question Patterns

### 1. Realtime vs Message Batches API (Domain 2)
- **Scenario**: Process 10,000 documents overnight for a non-urgent morning analytics report with cost as the primary concern.
- **Correct Choice**: Use the **Message Batches API**, which processes large asynchronous workloads within a 24-hour window at a 50% cost reduction.

### 2. Defending Against Prompt Injection (Domain 7)
- **Scenario**: An agent summarizes user-submitted web pages; one page contains hidden text instructing the model to reveal its system prompt.
- **Correct Choice**: Treat retrieved page content as **untrusted input**, keep it separate from trusted instructions, and use **guardrails or hooks** so injected instructions cannot trigger sensitive actions.

### 3. Reusable Internal Tooling with MCP (Domain 8)
- **Scenario**: Expose an internal inventory REST API so multiple Claude applications can share it and maintain it independently.
- **Correct Choice**: Build an **MCP server** that exposes the inventory operations as tools so multiple Claude applications can connect to it.

---

<!-- layout: 2-column -->
# Eliminating Developer Exam Distractors

### Common Distractor Traps
- **Sending 10,000 synchronous requests in parallel** or **lowering `max_tokens`** when an overnight batch workload can use the Message Batches API
- **Raising model temperature**, **switching to a larger model**, or **asking users politely in the system prompt** to mitigate prompt injection
- **Hard-coding API logic into system prompts**, **pasting live inventory into every context window**, or assuming **built-in tools can reach arbitrary internal REST APIs**
- Blaming the model when an **integration layer** bug (bad parser, retired model ID, or truncated payload) caused the failure

### Winning Developer Habits
- Match **synchronous vs batch APIs** to latency tolerance and cost goals
- Enforce **trust boundaries** (XML tags + hooks + least-privilege tools) around all untrusted external content
- Choose the right customization primitive: **Built-in Tools** vs **Custom Tools** vs **Skills** vs **MCP Servers**
- Inspect execution traces to isolate **integration-layer bugs** from **model output errors**

---

# Exam-Day Pacing and Logistics

- **Generous time per question**: 53 items in 120 minutes gives you roughly **2 minutes 15 seconds per item**
- **Allocate extra focus to Domain 2 (33.1%)**: expect roughly 17 to 18 questions on API mechanics, application design, SDLC integration, and configuration management
- **Retake policy**: 14 days after the 1st failed attempt, 30 days after the 2nd, and 90 days after the 3rd (up to 4 attempts in a rolling 12-month period)

---

# What You Learned

- Reviewed the CCDV-F exam format, scoring, and domain weights
- Evaluated the Minimally Qualified Candidate developer profile
- Analyzed canonical CCDV-F sample question patterns
- Applied an engineering-focused exam-day strategy

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
