# APP (Agent Pipeline Protocol) Design Principles

> This document covers the design philosophy and architectural thinking behind APP. For the technical specification, see [README (English)](README.md) | [README (Chinese)](README_zh.md).

[中文文档](agent-pipeline-protocol-design-principles_zh.md)

---

## Introduction: From Skill to APP

The mainstream AI Skill architecture today is essentially "prompt-driven intent description" — a SKILL.md tells the LLM "what to do," and how the model interprets and executes it depends on the reasoning path at that moment. This works well for creative, exploratory tasks, but exposes a fundamental problem in enterprise applications: **non-determinism and instability**.

The same Skill connected to different models may produce completely different outputs; the same model may produce different results across two runs. If we believe that Skill-driven LLMs are the foundational framework for next-generation software, then the current Skill architecture clearly cannot deliver stable, predictable, deterministic output.

**APP (Agent Pipeline Protocol)** is the response to this problem. The name captures three core dimensions: **Agent** (AI agent-driven), **Pipeline** (execution paths hardened through practice), **Protocol** (validated execution procedures).

> In the traditional world you build apps purely with code; in the new world you build them with a mix of natural language and code. APP is still an APP — what changes is how you build it.

---

## I. Core Metaphor: AI's Muscle Memory

The most intuitive way to understand APP is to draw an analogy with human **muscle memory**.

When someone learns to play piano, every keystroke initially requires conscious thought — reading the score, identifying notes, controlling fingers. Playing is slow, error-prone, and mentally exhausting. But through repeated practice, common finger patterns become muscle memory — fingers move automatically, the brain barely participates. Playing becomes fast, stable, and frees up mental attention for higher-level concerns (emotional expression, improvisation).

**Skill is brain mode**: flexible, able to handle unexpected situations, but consuming significant resources (tokens), slow, with potentially different execution paths each time.

**APP's Pipeline is muscle memory**: can only handle practiced, fixed movements, but executes extremely fast, barely consumes "thinking" resources, and produces highly stable output.

This metaphor reveals the most essential relationship between APP and Skill: **APP is not a replacement for Skill, but its evolved form.** An APP gradually accumulates Pipelines through use (forming muscle memory), evolving from "everything requires conscious thought" to "common actions happen automatically, the brain only handles new situations."

### Why This Metaphor Matters

It resolves a core tension in architecture design: **Should we choose the determinism of static Pipelines, or the flexibility of LLM dynamic orchestration?**

The answer: you don't have to choose. You get both.

Humans work the same way — practiced movements use muscle memory, new situations engage the brain. It's not either/or, but two systems working in concert. APP's architecture follows the same principle: high-frequency paths are hardened into Pipelines, unknown paths fall back to Skill mode's dynamic orchestration.

---

## II. Problem Diagnosis: Sources of Non-determinism

Non-determinism in the current Skill architecture comes from multiple compounding layers:

* **Model version differences**: Different model versions (or even iterations of the same model) may interpret and respond to the same prompt differently.
* **Inference randomness**: The same model under different temperature settings, or even with identical parameters on different inference paths, produces random fluctuations in output.
* **Context sensitivity**: Changes in the context window content affect attention distribution, causing output drift.
* **Instruction ambiguity**: Natural language instructions in SKILL.md inherently admit multiple reasonable interpretations.

These errors may be acceptable in single tasks, but in compound agentic workflows they **amplify exponentially** — output deviation from one step becomes input deviation for the next, accumulating progressively.

**Core diagnosis**: The problem is not that LLMs are non-deterministic, but that the current architecture **fails to isolate non-determinism**. Traditional software also has non-deterministic components (network requests, user input, random numbers), but software engineering has an entire methodology for isolating and handling these: type systems, contracts, idempotency design, retry mechanisms, test coverage. The current Skill bets every layer on the LLM's single inference pass, with no such isolation layer.

---

## III. APP's Design Philosophy: Assign Capabilities by Task Nature

Before diving into APP's technical design, a key question needs clarification: **In an APP, which work should go to code, and which to the LLM?**

### The Criterion: Task Nature, Not "Who Is Stronger"

Return to the muscle memory metaphor. A pianist playing a scale practiced a thousand times — fingers move automatically, brain barely participates — this is muscle memory's domain. But when improvising a melody never played before, full concentration is required — this is the brain's domain. The basis for division is not "can the brain play scales?" (of course it can), but **whether the action is fixed and repetitive, or requires understanding and creativity.**

Work in an APP follows the same logic:

**Fixed, repetitive work → Pipeline code (muscle memory)**: Data collection, statistical calculation, format validation, report skeleton generation, JSON assembly — the logic is identical every time, requires no "understanding," only precise execution. Given to code: zero errors, zero token consumption. **Core principle**: Anything Pipeline code can deterministically complete, the LLM should not redo probabilistically. If a function can calculate the compliance rate, don't let the LLM "calculate" it; if an API can fetch data, don't let the LLM "guess" it.

**Work requiring understanding and judgment → LLM (brain)**: Analyzing what a piece of code does, judging whether a commit follows conventions, understanding a user's ambiguous intent, choosing the most reasonable next step from multiple paths — the core of this work is semantic understanding, cannot be enumerated with if-else, and naturally belongs to the LLM.

### Not "LLM Only Orchestrates"

A common misconception reduces the LLM's role to "orchestrator" — only deciding which function to call, not participating in actual work. This is too narrow. The LLM takes on far more than orchestration in an APP:

```
✅ Semantic analysis: What does this code do?                    → LLM
✅ Judgment: Does this commit follow conventions? Which path?    → LLM
✅ Content generation: Write a summary and analysis for a report → LLM
✅ Intent understanding: What date range is "last week"?         → LLM
❌ Existing functionality: Don't redo what code already does     → Code
❌ Precise calculation: Calculate compliance rates, rank tables  → Code
❌ Structure assembly: Build report skeletons, format output     → Code
❌ Data retrieval: Call APIs, query databases, read files        → Code
```

**The LLM is both orchestrator and executor — it just executes tasks requiring "understanding," not tasks requiring "precision."** Code is the mirror image: it executes deterministic work but doesn't participate in decisions requiring semantic understanding.

### Core Principle: Match Capabilities to Task Nature

This means the APP design question is not "should the LLM participate in execution," but rather **find the right executor for each task — understanding goes to LLM, precision goes to code — and establish clear contracts between the two, ensuring LLM output passes programmatic validation before entering downstream flows.**

---

## IV. Dual-Mode Architecture and Engineering Implementation

### The Constraint Spectrum

How do you ensure the LLM knows which functions to call and in what order? Existing practice offers three approaches with different constraint strengths:

**Weakest constraint (tool descriptions)**: Give the LLM only tool definitions and descriptions; the LLM decides call order entirely on its own. Works for simple tasks, but complex flows may miss steps or get order wrong.

**Medium constraint (SKILL.md)**: Write step-by-step procedures in the prompt; the LLM follows along. Essentially a Pipeline written in natural language. Usually effective, but the LLM may deviate from instructions.

**Strongest constraint (static Pipeline)**: Define execution order in YAML or code, driven by deterministic programs. Most reliable, but can only handle predefined flows, unable to handle unexpected situations.

APP's key insight is: **You don't need to pick a fixed position on this spectrum.** An APP should have both modes simultaneously:

### Pipeline Mode (Muscle Memory)

For known task paths, Pipelines are pre-written. These Pipelines are fully deterministic — code controls the flow, the LLM is only called at specific steps for semantic analysis or content generation, with its output validated by Schema. Like muscle memory that doesn't require brain participation — actions complete automatically, fast and stable.

### Dynamic Orchestration Mode (Brain Thinking)

When a user's request doesn't match any predefined Pipeline, the system falls back to Skill mode — the LLM autonomously decides which tools to call and in what order. Flexible but consumes more tokens, output stability decreases.

### Routing Layer: Intent Recognition and Mode Switching

When a user request arrives, the routing layer's job is to understand user intent and attempt to match it to an existing Pipeline. This is essentially a classification task (choose one from N+1 options), and a lightweight, fast small model is more than sufficient.

Pipeline hit → enter muscle-memory mode, driven by deterministic code for fast execution; miss → fall back to brain mode, driven by LLM for flexible handling. Routing failure consequences are safe: a missed Pipeline that should have hit just costs extra tokens without affecting results; a Pipeline that shouldn't have hit fails due to input mismatch and falls back to brain mode.

### Day 0: Generate Pipelines Immediately

**Most Skills' main paths are already clear at design time.** When you write a Skill for code review, the path "fetch diff → analyze per-file → summarize report" doesn't require a hundred runs to discover — the Skill author knows from Day 1. A typical Skill has 90% of its invocations following these known fixed paths, with only 10% being unexpected situations requiring LLM flexibility.

Since main paths are known, there's no reason to wait. **Day 0 should codify known paths as Pipelines.** Pipeline code is generated by the LLM; developers only need to describe "what this path does." Generation cost is a one-time few hundred tokens — while every invocation of that Pipeline saves the LLM's re-orchestration and reasoning tokens. If the APP will be called thousands of times, this ROI is obvious.

Day 0 does three things:

**1. Extract what code should do from the Skill.** Look at what the LLM is doing in the existing Skill; implement the fixed, repetitive work (data fetching, calculation, formatting, report skeletons) in code. The LLM retains only the understanding and judgment parts.

**2. Add validation to LLM outputs.** Define Schemas for every LLM output, add structural and content validation, retry with error messages on failure.

**3. Codify known paths into Pipelines.** Change main path execution from "LLM decides every time" to "code defines the flow, LLM only called at specific steps." Add a routing layer for intent matching.

### Runtime: Continuous Evolution

After Day 0, the APP can stably and efficiently handle 90% of requests. The remaining 10% fall back to Skill mode for flexible handling.

Evolution happens in that 10%: if a type of "unexpected" request appears repeatedly, it's transitioning from "new situation" to "common operation" — time to add a new Pipeline. The APP accumulates more Pipelines over time, handling an ever-wider range of scenarios automatically, like a person's muscle memory library continuously expanding.

---

## V. Validation: Making LLM Output Reliable

The code parts don't need worrying about — functions compute correctly by definition. What truly needs validation is the LLM-driven segments.

### Two Lines of Validation

**Structural validation**: Check whether the LLM output's format is compliant — required fields present, data types correct, enum values within range. This is pure JSON/Schema validation, nearly zero cost, and should be enabled by default for all LLM outputs.

**Content validation**: Check whether the LLM output's values make business sense — 50 students can't produce 200 legs, analyzing 50 commits can't yield 53 results, compliance rate can't exceed 100%. Essentially a few business assertions in code, catching obvious absurdities from LLM hallucination or calculation errors.

### Validation Failure: Retry, Then Error

When validation fails, handling is straightforward: **inject the error message into the prompt and have the LLM regenerate**. Set a maximum retry count (typically 2-3), exhaust retries then error, transparently returning failure information to the caller.

As for more complex exception strategies — degraded output, circuit breaking, human review — these are decisions for specific business scenarios and should not be mandated at the protocol level. APP provides validation mechanisms and retry capabilities; the business layer decides what to do after failure based on its own needs.

---

## VI. Vision: Self-Evolving APPs

The APP described above relies on developer observation and manual additions for Pipeline accumulation. But the APP architecture naturally possesses a more far-reaching possibility: **runtime self-optimization** — the APP observes its own behavior patterns and automatically compiles high-frequency reasoning paths into Pipelines.

This concept has a classic counterpart in computer science: **JIT (Just-In-Time) compilation**.

A JIT compiler works as follows: the program runs in interpreted mode, recording hot code paths at runtime. When a code segment is repeatedly executed, it's automatically compiled to machine code. From then on, reaching that code executes the compiled version directly — faster, with consistent results. APP's self-optimization follows exactly the same logic:

1. **Record call stacks**: When the APP runs in Skill mode, record the LLM's reasoning path each time — which tools were called, in what order, with what parameters
2. **Identify hot paths**: Periodically analyze these call records, finding patterns where "the LLM reasoned extensively, but the execution path ended up the same every time"
3. **Auto-generate Pipelines**: Compile these hot paths into Pipeline code, register them with the routing layer

From then on, the same request hits the new Pipeline through the routing layer, going directly to muscle-memory mode — skipping LLM orchestration, faster and cheaper.

This self-optimization mechanism inherits several important JIT properties: **only optimizes hot paths** — infrequent unexpected requests continue in Skill mode with no extra cost; **transparent to users** — results are identical before and after optimization, just faster and cheaper in tokens; **safe rollback** — if an auto-generated Pipeline fails due to input mismatch, the system automatically falls back to Skill mode for reprocessing.

If this vision is realized, APP will become a **self-evolving software form**: it can not only become more "skilled" through developers manually adding Pipelines, but can also autonomously observe and optimize during runtime, continuously compiling probabilistic reasoning paths into deterministic execution code. The more an APP is used, the faster, more stable, and more resource-efficient it becomes — truly achieving the ultimate form of the "muscle memory" metaphor.

---

*This document serves as the systematic thinking behind the APP (Agent Pipeline Protocol) design framework.*

*Date: 2026-04-08*
*License: MIT*
