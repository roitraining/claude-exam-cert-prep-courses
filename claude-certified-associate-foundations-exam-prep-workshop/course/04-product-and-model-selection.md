<!-- course-title: Claude Certified Associate - Foundations: Hands-On Exam Prep Workshop -->
<!-- course-theme: roi-alt-theme -->

<!-- layout: title -->
![ROI Logo](images/roi-logo-with-name.png)

Claude Certified Associate - Foundations: Hands-On Exam Prep Workshop

# Chapter 4: Product and Model Selection

---

# Chapter 4: Objectives

- Select the right Claude product feature across Projects, research mode, chat, and Artifacts
- Differentiate between Claude Haiku, Sonnet, and Opus
- Align model selection with task cost, speed, and quality requirements
- Manage context window limits and memory by knowing when to restart, summarize, or persist

---

# Domain 3: Product and Model Selection

| Objective | Exam Focus |
| :--- | :--- |
| **3.1** | Select appropriate Claude product features (Projects, research mode, chat, Artifacts) |
| **3.2** | Differentiate between Claude model types (Haiku, Sonnet, Opus) |
| **3.3** | Align model selection with task requirements (cost, speed, quality) |
| **3.4** | Understand and manage context limitations and memory considerations (when to restart, summarize, or persist) |

---

<!-- layout: navigation -->
# Chapter 4

- **Claude Product Features**
- Haiku, Sonnet, and Opus
- Context and Memory

---

<!-- layout: card-layout -->
# Matching Features to the Job

### Standard Chat
- Best for one-off questions, quick drafting, ad-hoc brainstorming, and tasks that do not require persistent background files across future sessions.

### Claude Projects
- Persistent workspace combining **custom system instructions** and **uploaded knowledge files** shared across every chat inside the Project.
- Best for ongoing initiatives, client accounts, or team workflows that reuse the same style guide, templates, and reference docs.

### Research Mode
- Conducts multi-step, in-depth exploration across web and connected workspace sources to compile comprehensive, cited reports.
- Best when a task requires investigating multiple sources and synthesizing broad evidence rather than answering from a single file.

### Artifacts
- Renders standalone documents, interactive visuals, dashboards, and templates in a dedicated side panel for live editing and reuse.

---

<!-- layout: navigation -->
# Chapter 4

- Claude Product Features
- **Haiku, Sonnet, and Opus**
- Context and Memory

---

<!-- layout: 3-column -->
# Comparing the Claude Model Family

### Claude Haiku
- **Speed and cost leader**: fastest response time and lowest cost
- **Best for**: high-volume, straightforward tasks such as short customer-reply drafts, quick categorization, simple formatting, and routine summaries
- **Trade-off**: not designed for deep multi-step reasoning or subtle nuance

### Claude Sonnet
- **Balanced daily driver**: strong reasoning, writing quality, and instruction-following at fast interactive speeds
- **Best for**: most day-to-day professional workflows, document analysis, report drafting, project planning, and Artifact creation
- **Default choice** for general knowledge work

### Claude Opus
- **Deepest reasoning and synthesis**: highest capability on complex, ambiguous, or multi-constraint problems
- **Best for**: intricate strategic analysis, reconciling complex policy trade-offs, and high-stakes synthesis across dense material
- **Trade-off**: slower response speed and higher cost

---

<!-- layout: stacked -->
# Balancing Cost, Speed, and Quality

- **Match the model to the task requirements**: always balance **quality**, **speed (latency)**, and **cost**
- **Official exam pattern**: when an associate needs to generate a **high volume of short customer-reply drafts** where speed and cost matter more than deep reasoning, choose a **faster, lower-cost model (Haiku)**
- **Avoid extremes**: always using the highest-cost model (Opus) for routine tasks wastes budget and slows turnaround, while using the smallest model for complex strategic synthesis risks shallow results

![Claude Feature and Model Selection Matrix](images/ch04-feature-and-model-matrix.svg)

---

<!-- layout: navigation -->
# Chapter 4

- Claude Product Features
- Haiku, Sonnet, and Opus
- **Context and Memory**

---

<!-- layout: 2-column -->
# Understanding Context Window Limits

### How the Context Window Works
- Every chat has a finite **context window** holding your uploaded files, conversation history, and Claude’s responses
- As a single conversation grows very long or mixes unrelated topics, earlier details can get diluted and responses may slow down or drift

### Signs of Context Saturation
- Claude forgets constraints established 25 turns ago
- Instructions from an earlier task bleed into a brand-new topic (for example, applying yesterday’s marketing tone to today’s finance memo)
- You hit conversation length limits after pasting multiple large documents into one thread

---

<!-- layout: 3-column -->
# When to Restart, Summarize, or Persist

### Restart (Start a New Chat)
- You are **switching to an unrelated topic or client**
- The current chat contains false starts, outdated drafts, or rejected directions that are confusing new responses
- Keeps context clean, fast, and focused

### Summarize and Handoff
- A single multi-stage task has grown very long, and you want to continue in a fresh chat without losing progress
- Ask Claude to produce a concise **handoff summary** of decisions, verified facts, and open items, then paste that summary into a new chat

### Persist (Projects / Memory)
- Background documents, brand guidelines, and role instructions are needed **repeatedly across many sessions**
- Store them in a **Claude Project** (or workspace Memory) so every new chat starts with the right context automatically

---

# Do Now: Clean Session Handoff

**Time:** 1–2 minutes

- Open the same Claude chat you used in the Chapter 2 or Chapter 3 Do Now
- Paste this **Session Handoff** prompt to compress the thread for a fresh session:

```text
We are wrapping up this working session and moving to a fresh chat
to keep our context window clean. Summarize our work so far into a
compact "Session Handoff Block" (under 120 words) with three headings:
1. Approved Decisions and Facts
2. Constraints and Style Rules to Keep
3. Next Immediate Task
```

- Copy the generated handoff block, open a **new chat**, and see how cleanly you can continue from that compact summary

---

<!-- layout: 2-column -->
# Do Now Debrief: Exam Takeaways

### Why This Works in Practice
- Instead of dragging 30 turns of rough drafts and rejected edits into the next phase of work, a 120-word handoff block preserves 100% of your verified decisions with minimal context overhead
- Unrelated topics always get a clean restart; recurring team standards go into a **Project**

### Key Domain 3 Exam Takeaways
- Pick **Haiku** for high-volume, speed/cost-sensitive routine drafts; **Sonnet** for everyday professional work; **Opus** for deep complex reasoning
- **Restart** when changing topics, **summarize** when continuing a long thread in a fresh chat, and **persist** in a **Project** when instructions and files apply across many sessions

---

# Quiz 1: Domains 1 to 3 Review

- [Open Quiz](https://claude-certification-exam-prep-1077635956541.us-east4.run.app/questions?file=public/2026-10-31-13-37-ccao-f-realistic-instructor.json)

---

# What You Learned

- Selected the right Claude product feature across Projects, research mode, chat, and Artifacts
- Differentiated between Claude Haiku, Sonnet, and Opus
- Aligned model selection with task cost, speed, and quality requirements
- Managed context window limits and memory by knowing when to restart, summarize, or persist

---

<!-- layout: stacked -->
# Questions and Answers

![Questions and Answers](images/qa.png)
