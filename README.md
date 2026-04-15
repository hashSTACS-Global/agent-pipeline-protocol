# APP (Agent Pipeline Protocol) v0.4

> **This is a technical specification for AI coding tools** (Claude Code, ChatGPT Codex, Cursor, etc.) to follow when developing APPs — the pipeline-driven evolution of LLM Skills.

---

## 1. 核心概念

**APP** 是 Skill 的进化形态。一个 Skill 完全由 LLM 动态驱动；一个 APP 将大部分已知路径固化为 Pipeline（确定性代码驱动），只有未知路径退化为 Skill 模式（LLM 动态编排）。

一个 APP 由三部分组成：

- **SKILL.md** — 大脑模式，**仅当**没有 Pipeline 匹配时，LLM 按 SKILL.md 的指令灵活处理。SKILL.md **不承担** Runner 的调度职责
- **Pipeline** — 肌肉记忆模式，确定性代码控制流程，LLM 只在 llm 类型的步骤被调用
- **Pipeline Runner** — 独立程序，负责路由、调度、数据传递。Runner **不是** LLM，也不是 SKILL.md 里的文字指令

路由层由 Pipeline Runner 自动构建——启动时扫描所有 `pipeline.yaml`，收集其中的 `description` + `triggers`，构建路由表用于意图分类。

### 构造 Pipeline 与析构 Pipeline

借鉴 C++ 构造函数/析构函数的思想，Runner 在执行任何业务 Pipeline 时，**强制**在前后各执行一条特殊 Pipeline：

- **构造 Pipeline（constructor）** — 业务 Pipeline 第一个 step 执行前运行。用于配置检查、环境初始化、git pull 等前置工作。如果构造 Pipeline 失败，业务 Pipeline 不执行，Runner 直接返回错误
- **析构 Pipeline（destructor）** — 业务 Pipeline 最后一个 step 执行后运行（**无论成功或失败**，类似 `finally`）。用于状态清理、日志记录、临时文件清理

构造和析构 Pipeline 是**框架级保证**，不依赖 LLM 判断，不依赖业务 step 自觉调用。

---

## 2. 目录结构

```
my-app/
├── SKILL.md                          # 大脑模式指令（纯 fallback，不含 Runner 逻辑）
├── pipelines/
│   ├── _constructor/                 # 构造 Pipeline（可选，Runner 自动识别）
│   │   ├── pipeline.yaml
│   │   └── steps/
│   │       └── check_config.py
│   ├── _destructor/                  # 析构 Pipeline（可选，Runner 自动识别）
│   │   ├── pipeline.yaml
│   │   └── steps/
│   │       └── cleanup.py
│   ├── <pipeline-name>/              # 业务 Pipeline
│   │   ├── pipeline.yaml             # Pipeline 定义：步骤声明 + 路由信息
│   │   ├── schemas/                  # LLM 输出的 JSON Schema
│   │   │   └── <step-name>.json
│   │   └── steps/                    # 步骤脚本（任意语言）
│   │       ├── <step-name>.py
│   │       ├── <step-name>.sh
│   │       └── validate_<step>.py    # 内容验证脚本（可选）
│   └── <pipeline-name>/
│       └── ...
└── tools/                            # 共享工具（现有 Skill 工具目录，不变）
```

**约定**：
- 每条 Pipeline 一个目录，目录名即 Pipeline 名
- `_constructor/` 和 `_destructor/` 是保留目录名（以 `_` 开头），Runner 自动识别，不纳入路由表
- `pipeline.yaml` 是每条 Pipeline 的**单一信息来源**，包含步骤定义和路由信息（description + triggers）
- `schemas/` 存放 LLM 步骤的 JSON Schema，文件名和步骤名对应
- `steps/` 存放步骤脚本，语言不限
- `tools/` 是共享的工具目录，Pipeline 和 Skill 模式都可以使用
- 无需集中索引文件——Runner 启动时自动扫描 `pipelines/*/pipeline.yaml` 构建路由表（跳过 `_` 开头的目录）

---

## 3. pipeline.yaml — Pipeline 定义

`pipeline.yaml` 是每条 Pipeline 的**单一信息来源**，既定义执行步骤，也包含路由所需的信息。Runner 启动时自动扫描所有 `pipelines/*/pipeline.yaml`，收集 `description` + `triggers` 构建路由表——无需手动维护集中索引文件。

```yaml
name: code-review
description: "标准代码审查：拿 diff、逐文件分析、生成报告"
triggers:                                 # 路由参考示例（可选）
  - "review this PR"
  - "代码审查"
  - "看看这个 diff"

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
      分析以下代码变更，逐文件识别潜在问题：

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

**字段说明**：

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | Pipeline 名称，和目录名一致 |
| `description` | 是 | Pipeline 功能描述，同时供路由模型理解意图 |
| `triggers` | 否 | 触发示例，帮助路由模型匹配（不是精确关键词，是参考示例） |
| `input` | 否 | 输入参数定义（名称: 类型） |
| `steps` | 是 | 步骤列表，按顺序执行 |
| `output` | 否 | 指定哪个步骤的输出作为 Pipeline 最终输出，默认最后一步 |

**路由机制**：Pipeline Runner 将用户请求和所有 Pipeline 的 `description` + `triggers` 一起发给路由模型（默认 `lite` 档位），路由模型做分类判断——命中哪条 Pipeline，或者都不命中退化为 Skill 模式（SKILL.md 动态编排）。

---

## 4. 步骤类型

### 4.1 code 步骤

确定性代码，由 Pipeline Runner 起子进程执行。

```yaml
- name: fetch_diff
  type: code
  command: "bash steps/fetch_diff.sh"
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 步骤名称，在 Pipeline 内唯一 |
| `type` | 是 | 固定为 `code` |
| `command` | 是 | 执行命令，可以是任意语言的脚本 |

**数据传递约定**：

Runner 通过 stdin 向脚本传入 JSON：

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

- `input` — Pipeline 的原始输入参数
- `steps` — 所有已完成的前序步骤的输出

脚本通过 stdout 返回 JSON：

```json
{
  "output": { "..." : "..." }
}
```

**唯一约定：stdin JSON in, stdout JSON out。** 脚本用什么语言实现无所谓——Python、Shell、Node.js、Java（`java -jar`）、Go 编译后的二进制——只要遵守这个约定。

**错误处理**：脚本以非零退出码退出表示执行失败，Runner 终止 Pipeline 并返回错误信息。

### 4.2 llm 步骤

调用 LLM，由 Pipeline Runner 直接处理（不起子进程）。

```yaml
- name: analyze_files
  type: llm
  model: standard
  prompt: |
    分析以下代码变更，逐文件识别潜在问题：

    {{fetch_diff.output}}
  schema: schemas/analyze_files.json
  validate: steps/validate_analysis.py
  retry: 3
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 步骤名称 |
| `type` | 是 | 固定为 `llm` |
| `model` | 否 | 模型档位（见第 5 节），默认 `standard` |
| `prompt` | 是 | Prompt 模板，支持 `{{step_name.output}}` 引用前序步骤的输出 |
| `schema` | 否 | JSON Schema 文件路径，用于结构验证 |
| `validate` | 否 | 内容验证脚本路径 |
| `retry` | 否 | 验证失败时最大重试次数，默认 `2` |

**执行流程**：

```
1. 渲染 prompt 模板（用前序步骤的输出替换 {{}} 变量）
2. 调用 LLM API（附带 schema 做结构化输出）
3. 结构验证（JSON Schema）
   → 失败 → 带错误信息重试
4. 内容验证（调验证脚本，如果配置了 validate）
   → 失败 → 带错误信息重试
5. 都通过 → 输出结果，进入下一步
6. 重试耗尽 → 报错，终止 Pipeline
```

**Prompt 模板语法**：

- `{{step_name.output}}` — 引用指定步骤的完整输出（JSON 序列化为字符串）
- `{{step_name.output.field}}` — 引用指定步骤输出的某个字段
- `{{input.param}}` — 引用 Pipeline 的输入参数

### 4.3 内容验证脚本

内容验证脚本遵循和 code 步骤相同的 stdin/stdout 约定。

Runner 通过 stdin 传入：

```json
{
  "output": { "..." : "..." },
  "input": { "..." : "..." },
  "steps": { "..." : "..." }
}
```

- `output` — 当前 LLM 步骤的输出（待验证）
- `input` — Pipeline 的原始输入
- `steps` — 前序步骤的输出（可用于交叉验证）

验证通过：退出码 `0`，stdout 输出 `{"valid": true}`

验证失败：退出码非零，stdout 输出错误信息（会被注入重试 prompt）：

```json
{
  "valid": false,
  "errors": [
    "分析了 12 个文件，但 diff 只有 8 个",
    "严重问题 #3 缺少具体描述"
  ]
}
```

验证脚本是可选的。不配置 `validate` 字段时，只做 JSON Schema 结构验证。

---

## 5. 模型档位

APP 不绑定具体的模型名称。Pipeline 中通过档位声明所需的模型能力，运行时负责将档位映射到实际模型。

| 档位 | 用途 | 典型场景 |
|------|------|----------|
| `lite` | 轻量快速，擅长分类和简单提取 | 路由判断、参数提取、格式转换 |
| `standard` | 平衡能力和成本，通用 | 语义分析、内容生成、代码理解 |
| `reasoning` | 深度推理，带扩展思考 | 复杂判断、合规审查、安全分析 |

**默认值**：llm 步骤不指定 `model` 时，默认使用 `standard`。

**运行时映射示例**（非规范内容，由运行时自行配置）：

```yaml
# 运行时配置示例（不属于 APP 规范）
model_mapping:
  lite: claude-haiku-4-5
  standard: claude-sonnet-4-6
  reasoning: claude-opus-4-6
```

---

## 6. Pipeline Runner 职责

Pipeline Runner 是 APP 的运行时，**必须是一个独立程序**（不是 SKILL.md 里的文字指令让 LLM 逐步执行）。LLM 只在 llm 类型的步骤被调用时介入，其余时间 Runner 自主运行。

### 6.1 核心职责

1. **发现** — 扫描 `pipelines/*/pipeline.yaml`（跳过 `_` 开头的目录），自动收集所有业务 Pipeline 的定义和路由信息（description + triggers），构建路由表
2. **路由** — 将用户请求和路由表中所有 Pipeline 的 description + triggers 发给路由模型（默认 `lite` 档位），做意图分类。命中 Pipeline → 执行对应 Pipeline；未命中 → 退化为 Skill 模式（SKILL.md 动态编排）
3. **执行** — 按 `pipeline.yaml` 的步骤顺序依次执行：code 步骤起子进程，llm 步骤调 LLM API
4. **数据传递** — 维护步骤间的数据上下文，每步的输出存入上下文供后续步骤引用
5. **验证与重试** — 对 llm 步骤做结构验证（JSON Schema）+ 内容验证（验证脚本），失败则带错误信息重试

### 6.2 构造/析构执行流程

Runner 在执行每条业务 Pipeline 时，遵循以下固定流程：

```
1. 执行构造 Pipeline（_constructor/）
   → 失败 → 终止，不执行业务 Pipeline，返回构造错误
   → 成功 → 继续
2. 执行业务 Pipeline
   → 成功或失败，都进入下一步
3. 执行析构 Pipeline（_destructor/）
   → 无论业务 Pipeline 成功或失败，析构始终执行（类似 finally）
   → 析构失败不覆盖业务 Pipeline 的错误，两个错误都返回
```

**构造 Pipeline 的典型用途**：
- 检查配置文件完整性
- 初始化环境变量
- 执行 `git pull` 同步最新数据
- 检查依赖工具是否可用

**析构 Pipeline 的典型用途**：
- 清理临时文件
- 记录执行日志
- 释放锁

**如果 APP 没有 `_constructor/` 或 `_destructor/` 目录**，Runner 跳过对应阶段，直接执行业务 Pipeline。两者都是可选的。

### 6.3 为什么 Runner 必须是独立程序

如果让 LLM 充当 Runner（读 SKILL.md 里的执行指令逐步调度），会导致：
- 每个 code step 多一轮 LLM tool call（5-10 秒延迟）
- 数据传递依赖 LLM 正确拼接 JSON（可能出错）
- 构造/析构无法保证执行（LLM 可能跳过）
- 验证/重试机制无法可靠实现

Runner 作为独立程序，所有确定性逻辑（路由、步骤调度、数据传递、构造/析构）在代码中保证执行，LLM 只在需要语义理解的 llm step 被调用。

Pipeline Runner 本身的实现语言不限，但推荐 Python（生态成熟，方便调用 LLM API 和做 JSON Schema 验证）。

---

## 7. Skill → APP 自动转化

给定一个现有的 Skill（SKILL.md + tools/），AI 编程工具按以下流程自动转化为 APP：

### Step 1：分析 Skill

- 读 SKILL.md，理解 Skill 的意图和功能
- 读 tools/ 目录，了解可用的工具
- 识别主路径：哪些执行流程是固定的、会被反复调用的
- 识别灵活部分：哪些情况需要 LLM 临场判断

### Step 2：为每条主路径生成 Pipeline

对每条识别出的主路径：

1. **拆分步骤**：哪些是确定性工作（code 步骤），哪些需要语义理解（llm 步骤）
2. **code 步骤**：生成可执行脚本，遵循 stdin JSON in / stdout JSON out 约定
3. **llm 步骤**：
   - 编写 prompt 模板
   - 定义 JSON Schema（结构验证）
   - 如有必要，生成内容验证脚本
   - 选择合适的模型档位（lite / standard / reasoning）
4. **组装 pipeline.yaml**——包含步骤定义、description 和 triggers（路由信息与步骤定义在同一个文件中，无需额外索引）

### Step 3：保留 SKILL.md

- 原始 SKILL.md 保持不变，作为未命中 Pipeline 时的 fallback
- 确保 tools/ 目录中的工具在 Skill 模式下仍可正常使用

### 转化输出

```
输入：                           输出：
SKILL.md                  →     SKILL.md（不变）
tools/                    →     tools/（不变）
                                pipelines/
                                  ├── _constructor/     （如有横切关注点）
                                  ├── _destructor/      （如有清理需求）
                                  ├── <pipeline-1>/
                                  │   ├── pipeline.yaml
                                  │   ├── schemas/
                                  │   └── steps/
                                  └── <pipeline-2>/
                                      └── ...
```

---

## 8. 设计约定总结

1. **pipeline.yaml 是单一信息来源** — 每条 Pipeline 的步骤定义和路由信息（description + triggers）都在同一个文件中，无需集中索引，消除重复登记和一致性风险
2. **Pipeline 是声明，不是实现** — YAML 定义流程，脚本实现步骤，语言不限
3. **stdin JSON in, stdout JSON out** — 所有脚本（code 步骤和验证脚本）遵循统一的数据传递约定
4. **结构验证 + 内容验证** — JSON Schema 做格式检查，验证脚本做业务检查，两道防线
5. **模型档位而非模型名称** — APP 声明需要的能力等级（lite / standard / reasoning），运行时映射到具体模型
6. **SKILL.md 是纯 fallback** — 只在没有 Pipeline 匹配时使用，不承担 Runner 的调度职责
7. **Runner 是独立程序** — 所有确定性逻辑（路由、调度、数据传递、构造/析构）由代码保证，LLM 只在 llm step 被调用
8. **构造/析构是框架级保证** — 横切关注点（配置检查、环境初始化、清理）不依赖 LLM 判断或业务 step 自觉调用，由 Runner 强制执行
9. **Pipeline 的边际成本极低** — 由 LLM 生成，Day 0 就可以把已知路径全部固化

---

*Version: v0.4*
*Date: 2026-04-16*
*License: MIT*
