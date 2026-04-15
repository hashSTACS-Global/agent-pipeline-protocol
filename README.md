# APP (Agent Pipeline Protocol) v0.4

> **This is a technical specification for AI coding tools** (Claude Code, ChatGPT Codex, Cursor, etc.) to follow when developing APPs — the pipeline-driven evolution of LLM Skills.

[中文文档](README_zh.md)

---

## 1. Core Concepts

**APP** is the evolution of a Skill. A Skill is fully driven by an LLM dynamically; an APP codifies most known execution paths into Pipelines (deterministic code), falling back to Skill mode (LLM dynamic orchestration) only for unknown paths.

An APP consists of three parts:

- **SKILL.md** — Brain mode. Used **only when** no Pipeline matches. SKILL.md **does not** serve as the Runner's dispatch logic
- **Pipeline** — Muscle-memory mode. Deterministic code controls the flow; the LLM is only invoked during `llm`-type steps
- **Pipeline Runner** — A standalone program responsible for routing, scheduling, and data passing. The Runner is **not** an LLM, nor text instructions inside SKILL.md

The routing layer is built automatically by the Pipeline Runner — on startup it scans all `pipeline.yaml` files, collects `description` + `triggers`, and builds a routing table for intent classification.

### Constructor and Destructor Pipelines

Inspired by C++ constructors/destructors, the Runner **enforces** execution of special Pipelines before and after every business Pipeline:

- **Constructor Pipeline** — Runs before the first step of the business Pipeline. Used for config checks, environment initialization, git pull, etc. If the constructor fails, the business Pipeline does not execute; the Runner returns an error immediately
- **Destructor Pipeline** — Runs after the last step of the business Pipeline (**regardless of success or failure**, like `finally`). Used for state cleanup, logging, temp file removal

Constructor and destructor Pipelines are a **framework-level guarantee** — they do not rely on LLM judgment or business steps remembering to call them.

---

## 2. Directory Structure

```
my-app/
├── SKILL.md                          # Brain-mode instructions (pure fallback, no Runner logic)
├── pipelines/
│   ├── _constructor/                 # Constructor Pipeline (optional, auto-detected by Runner)
│   │   ├── pipeline.yaml
│   │   └── steps/
│   │       └── check_config.py
│   ├── _destructor/                  # Destructor Pipeline (optional, auto-detected by Runner)
│   │   ├── pipeline.yaml
│   │   └── steps/
│   │       └── cleanup.py
│   ├── <pipeline-name>/              # Business Pipeline
│   │   ├── pipeline.yaml             # Pipeline definition: step declarations + routing info
│   │   ├── schemas/                  # JSON Schemas for LLM step outputs
│   │   │   └── <step-name>.json
│   │   └── steps/                    # Step scripts (any language)
│   │       ├── <step-name>.py
│   │       ├── <step-name>.sh
│   │       └── validate_<step>.py    # Content validation script (optional)
│   └── <pipeline-name>/
│       └── ...
└── tools/                            # Shared utilities (used by both Pipelines and Skill mode)
```

**Conventions**:
- One directory per Pipeline; directory name = Pipeline name
- `_constructor/` and `_destructor/` are reserved directory names (prefixed with `_`); the Runner auto-detects them and excludes them from the routing table
- `pipeline.yaml` is the **single source of truth** for each Pipeline, containing both step definitions and routing info (description + triggers)
- `schemas/` holds JSON Schemas for LLM steps; filenames correspond to step names
- `steps/` holds step scripts in any language
- `tools/` is a shared utility directory, available to both Pipeline and Skill mode
- No centralized index file needed — the Runner auto-scans `pipelines/*/pipeline.yaml` to build the routing table (skipping `_`-prefixed directories)

---

## 3. pipeline.yaml — Pipeline Definition

`pipeline.yaml` is the **single source of truth** for each Pipeline, defining both execution steps and routing information. The Runner auto-scans all `pipelines/*/pipeline.yaml` on startup, collecting `description` + `triggers` to build the routing table — no manual index file required.

```yaml
name: code-review
description: "Standard code review: fetch diff, analyze per-file, generate report"
triggers:
  - "review this PR"
  - "code review"
  - "check this diff"

input:
  repo: string
  pr_number: integer

steps:
  - name: fetch_diff
    type: code
    command: "bash steps/fetch_diff.sh"

  - name: analyze_files
    type: llm
    model: standard
    prompt: |
      Analyze the following code changes, identify potential issues per file:

      {{fetch_diff.output}}
    schema: schemas/analyze_files.json
    validate: steps/validate_analysis.py
    retry: 3

  - name: calc_stats
    type: code
    command: "python3 steps/calc_stats.py"

  - name: gen_report
    type: code
    command: "python3 steps/gen_report.py"

output: gen_report
```

**Field Reference**:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Pipeline name, must match directory name |
| `description` | Yes | Pipeline description, also used by the routing model for intent matching |
| `triggers` | No | Example trigger phrases to help the routing model match (reference examples, not exact keywords) |
| `input` | No | Input parameter definitions (name: type) |
| `steps` | Yes | Ordered list of steps to execute |
| `output` | No | Which step's output serves as the Pipeline's final output; defaults to the last step |

**Routing mechanism**: The Pipeline Runner sends the user's request along with all Pipelines' `description` + `triggers` to the routing model (default: `lite` tier) for intent classification. Match → execute the Pipeline; no match → fall back to Skill mode (SKILL.md dynamic orchestration).

---

## 4. Step Types

### 4.1 Code Steps

Deterministic code, executed by the Pipeline Runner as a subprocess.

```yaml
- name: fetch_diff
  type: code
  command: "bash steps/fetch_diff.sh"
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Step name, unique within the Pipeline |
| `type` | Yes | Must be `code` |
| `command` | Yes | Command to execute; can be a script in any language |

**Data Passing Convention**:

The Runner passes JSON to the script via stdin:

```json
{
  "input": {
    "repo": "owner/repo",
    "pr_number": 42
  },
  "steps": {
    "fetch_diff": {
      "output": { "files": ["..."], "diff": "..." }
    }
  }
}
```

- `input` — The Pipeline's original input parameters
- `steps` — Outputs from all completed preceding steps

The script returns JSON via stdout:

```json
{
  "output": { "..." : "..." }
}
```

**The only contract: stdin JSON in, stdout JSON out.** The implementation language doesn't matter — Python, Shell, Node.js, Java (`java -jar`), compiled Go binary — as long as this contract is respected.

**Error handling**: A non-zero exit code signals failure; the Runner terminates the Pipeline and returns the error.

### 4.2 LLM Steps

LLM invocation, handled directly by the Pipeline Runner (no subprocess).

```yaml
- name: analyze_files
  type: llm
  model: standard
  prompt: |
    Analyze the following code changes, identify potential issues per file:

    {{fetch_diff.output}}
  schema: schemas/analyze_files.json
  validate: steps/validate_analysis.py
  retry: 3
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Step name |
| `type` | Yes | Must be `llm` |
| `model` | No | Model tier (see Section 5); defaults to `standard` |
| `prompt` | Yes | Prompt template; supports `{{step_name.output}}` to reference preceding step outputs |
| `schema` | No | JSON Schema file path for structural validation |
| `validate` | No | Content validation script path |
| `retry` | No | Max retries on validation failure; defaults to `2` |

**Execution flow**:

```
1. Render prompt template (replace {{}} variables with preceding step outputs)
2. Call LLM API (with schema for structured output)
3. Structural validation (JSON Schema)
   → Fail → retry with error message
4. Content validation (run validation script, if configured)
   → Fail → retry with error message
5. All pass → output result, proceed to next step
6. Retries exhausted → error, terminate Pipeline
```

**Prompt Template Syntax**:

- `{{step_name.output}}` — Full output of the specified step (JSON-serialized to string)
- `{{step_name.output.field}}` — A specific field from the specified step's output
- `{{input.param}}` — A Pipeline input parameter

### 4.3 Content Validation Scripts

Content validation scripts follow the same stdin/stdout convention as code steps.

The Runner passes via stdin:

```json
{
  "output": { "..." : "..." },
  "input": { "..." : "..." },
  "steps": { "..." : "..." }
}
```

- `output` — The current LLM step's output (to be validated)
- `input` — The Pipeline's original input
- `steps` — Preceding step outputs (for cross-validation)

Pass: exit code `0`, stdout `{"valid": true}`

Fail: non-zero exit code, stdout with error details (injected into the retry prompt):

```json
{
  "valid": false,
  "errors": [
    "Analyzed 12 files but the diff only has 8",
    "Critical issue #3 lacks a specific description"
  ]
}
```

Validation scripts are optional. Without a `validate` field, only JSON Schema structural validation is performed.

---

## 5. Model Tiers

APPs do not bind to specific model names. Pipelines declare the required capability level via tiers; the runtime maps tiers to actual models.

| Tier | Purpose | Typical Use Cases |
|------|---------|-------------------|
| `lite` | Lightweight and fast; good at classification and simple extraction | Routing, parameter extraction, format conversion |
| `standard` | Balanced capability and cost; general purpose | Semantic analysis, content generation, code understanding |
| `reasoning` | Deep reasoning with extended thinking | Complex judgment, compliance review, security analysis |

**Default**: LLM steps without a `model` field default to `standard`.

**Runtime mapping example** (not part of the spec; configured by the runtime):

```yaml
# Runtime config example (not part of the APP spec)
model_mapping:
  lite: claude-haiku-4-5
  standard: claude-sonnet-4-6
  reasoning: claude-opus-4-6
```

---

## 6. Pipeline Runner Responsibilities

The Pipeline Runner is the APP's runtime. It **must be a standalone program** (not text instructions in SKILL.md for the LLM to follow step-by-step). The LLM is only involved when an `llm`-type step is reached; the Runner operates autonomously the rest of the time.

### 6.1 Core Responsibilities

1. **Discovery** — Scan `pipelines/*/pipeline.yaml` (skip `_`-prefixed directories), collect all business Pipeline definitions and routing info (description + triggers), build the routing table
2. **Routing** — Send the user's request along with all Pipelines' description + triggers to the routing model (default: `lite` tier) for intent classification. Match → execute the Pipeline; no match → fall back to Skill mode (SKILL.md)
3. **Execution** — Execute steps sequentially per `pipeline.yaml`: subprocess for code steps, LLM API call for llm steps
4. **Data Passing** — Maintain inter-step data context; each step's output is stored and available to subsequent steps
5. **Validation & Retry** — For llm steps: structural validation (JSON Schema) + content validation (script), retry with error messages on failure

### 6.2 Constructor/Destructor Execution Flow

The Runner follows this fixed flow for every business Pipeline execution:

```
1. Execute Constructor Pipeline (_constructor/)
   → Failure → abort, do not execute business Pipeline, return constructor error
   → Success → continue
2. Execute Business Pipeline
   → Success or failure, always proceed to next step
3. Execute Destructor Pipeline (_destructor/)
   → Always executes regardless of business Pipeline outcome (like finally)
   → Destructor failure does not mask business Pipeline errors; both are returned
```

**Typical constructor uses**:
- Check config file completeness
- Initialize environment variables
- Run `git pull` to sync latest data
- Verify required tools are available

**Typical destructor uses**:
- Clean up temporary files
- Record execution logs
- Release locks

**If the APP has no `_constructor/` or `_destructor/` directory**, the Runner skips the corresponding phase and executes the business Pipeline directly. Both are optional.

### 6.3 Why the Runner Must Be a Standalone Program

If the LLM acts as the Runner (reading execution instructions from SKILL.md and dispatching step-by-step), the following problems arise:
- Each code step adds an extra LLM tool call round-trip (5-10 second latency)
- Data passing depends on the LLM correctly assembling JSON (error-prone)
- Constructor/destructor execution cannot be guaranteed (LLM may skip them)
- Validation/retry mechanisms cannot be reliably implemented

With the Runner as a standalone program, all deterministic logic (routing, step scheduling, data passing, constructor/destructor) is guaranteed by code. The LLM is only invoked when semantic understanding is needed in an llm step.

The Runner's implementation language is not specified, but Python is recommended (mature ecosystem, convenient for LLM API calls and JSON Schema validation).

---

## 7. Skill → APP Conversion

Given an existing Skill (SKILL.md + tools/), AI coding tools follow this process to convert it into an APP:

### Step 1: Analyze the Skill

- Read SKILL.md to understand the Skill's intent and capabilities
- Read the tools/ directory to understand available tools
- Identify main paths: which execution flows are fixed and repeatedly invoked
- Identify flexible parts: which situations require LLM judgment

### Step 2: Generate a Pipeline for Each Main Path

For each identified main path:

1. **Split into steps**: which parts are deterministic work (code steps), which need semantic understanding (llm steps)
2. **Code steps**: generate executable scripts following the stdin JSON in / stdout JSON out convention
3. **LLM steps**:
   - Write prompt templates
   - Define JSON Schemas (structural validation)
   - Generate content validation scripts if needed
   - Choose appropriate model tier (lite / standard / reasoning)
4. **Assemble pipeline.yaml** — containing step definitions, description, and triggers (routing info and step definitions in one file, no extra index needed)

### Step 3: Preserve SKILL.md

- The original SKILL.md remains unchanged as the fallback for unmatched requests
- Ensure tools/ directory utilities still work in Skill mode

### Conversion Output

```
Input:                            Output:
SKILL.md                  →      SKILL.md (unchanged)
tools/                    →      tools/ (unchanged)
                                 pipelines/
                                   ├── _constructor/     (if cross-cutting concerns exist)
                                   ├── _destructor/      (if cleanup is needed)
                                   ├── <pipeline-1>/
                                   │   ├── pipeline.yaml
                                   │   ├── schemas/
                                   │   └── steps/
                                   └── <pipeline-2>/
                                       └── ...
```

---

## 8. Design Principles Summary

1. **pipeline.yaml is the single source of truth** — Step definitions and routing info (description + triggers) live in one file per Pipeline; no centralized index, eliminating duplication and consistency risks
2. **Pipelines are declarations, not implementations** — YAML defines the flow; scripts implement the steps; language-agnostic
3. **stdin JSON in, stdout JSON out** — All scripts (code steps and validation scripts) follow a unified data passing convention
4. **Structural + content validation** — JSON Schema for format checks; validation scripts for business logic checks; two lines of defense
5. **Model tiers, not model names** — APPs declare capability levels (lite / standard / reasoning); the runtime maps to actual models
6. **SKILL.md is pure fallback** — Used only when no Pipeline matches; does not serve as the Runner's dispatch logic
7. **Runner is a standalone program** — All deterministic logic (routing, scheduling, data passing, constructor/destructor) is guaranteed by code; the LLM is only invoked during llm steps
8. **Constructor/destructor is a framework-level guarantee** — Cross-cutting concerns (config checks, environment init, cleanup) do not rely on LLM judgment or business steps calling them; enforced by the Runner
9. **Marginal cost of Pipelines is near-zero** — Generated by LLMs; all known paths can be codified on Day 0

---

*Version: v0.4*
*Date: 2026-04-16*
*License: MIT*
