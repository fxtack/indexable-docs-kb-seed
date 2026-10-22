# Stage B Sub-agent Prompt Template

> 服务于 Goal 1 Stage B（文档清单规划）。主 Agent 分派 4 个 sub-agent，通过 `{DIMENSION}` 参数区分：B1=architecture / B2=modules / B3=flows / B4=troubleshooting。
> **输出契约**：必须按 `schemas/stage-b-output.schema.json` 返回 JSON。

---

## 通用背景

你是种子包 Goal 1 Stage B 的一名 sub-agent，负责为 **{DIMENSION}** 维度规划文档清单。你的输出决定了：
1. Goal 2 Stage 3 会分派多少个文档生成 sub-agent
2. 每个 sub-agent 的封面代码范围（`covers_code`）
3. 文档间的依赖关系（`depends_on` / `involves_modules`）

**产出双份**：
1. 结构化 JSON（返回给主 Agent，供 Stage C 使用）
2. 骨架文件（每份文档一个 `.md` 文件，含 frontmatter + 一级标题 + blockquote 描述 + 空的必填章节标题）

**禁止**：
- 在骨架文件中写入正文内容（正文是 Stage 3 的职责）
- 引用行号（用符号名锚定代码）
- 分派新的 sub-agent

---

## 通用输入

- `TARGET_ROOT`：工程根目录
- `KB_ROOT`：`.think-tank/knowledge-base/`
- `META_KB_GUIDE`：`.think-tank/knowledge-base/.meta-knowledge-base-guide.md`（已由 Stage A 生成）
- `PROJECT_SCALE`：`small` / `medium` / `large`
- `PROJECT_NAME`：工程名
- `EXCLUDED_PATHS`：Stage A1 识别的排除路径列表（第三方源码、生成产物、submodule 等）。**你必须遵守**：任何被排除路径下的文件都不得进入 `covers_code`，被排除路径下的目录也不得作为一个"模块"

## 通用硬约束：尊重 EXCLUDED_PATHS

- **禁止**将 `EXCLUDED_PATHS` 中列出的任何路径（或其子路径）写入 `covers_code`；
- **禁止**将被排除路径下的顶层子目录视为"模块"（如 `vendor/foo/` 不应被识别为模块 foo）；
- **允许**在模块文档 §7 变更影响范围或 §5 设计决策中**提及**第三方库的**接口边界**（如"本模块调用 boost::asio"）——但只描述"如何使用"，不描述"boost::asio 内部实现"；
- **允许**若某第三方在工程中有关键定制或补丁，可以在模块文档中记录**定制点**（不是整份库文档）；
- **若你观察到 `EXCLUDED_PATHS` 中的某路径实际不该排除**（例如自研代码被误分类），**不要私自纳入**——在返回值的 `notes` 字段标注 `excluded_path_review_needed: <path>` 供主 Agent 转达人类。

---

## 章节：DIMENSION = "architecture"（B1）

### 任务

规划 `architecture/` 下应有哪些文档。

### 读取范围

- `META_KB_GUIDE`（了解本工程的架构描述规范）
- `CLAUDE.md`
- 顶层配置文件（无需深入源码）

### 决策依据

参考 `META_KB_GUIDE` 的 §2 目录结构章节。对本工程适用的 architecture 文档做出规划，通常包含：

- **必选**：`system-overview.md`（系统全景）、`tech-stack.md`（技术栈）
- **常见可选**（按工程需要）：`data-model-relations.md`、`deployment-paths.md`、`security-model.md`
- **QA 收集页**：`qa.md`（架构层 post-build QA 收集页，供人类填答）

### 输出

**结构化 JSON**（按 `schemas/stage-b-output.schema.json`）：

```json
{
  "dimension": "architecture",
  "docs": [
    {
      "id": "system-overview",
      "title": "系统全景与分层架构",
      "path": "architecture/system-overview.md",
      "description": "本工程的产品定位、分层架构图、核心数据流、项目目录结构",
      "keywords": ["架构", "分层", "数据流", "目录结构"],
      "covers_code": ["README.md", "CLAUDE.md"],
      "depends_on": []
    },
    {
      "id": "tech-stack",
      "title": "技术栈与编码约定",
      ...
    }
  ]
}
```

**骨架文件**（每份文档一个）：

```markdown
---
id: system-overview
title: "系统全景与分层架构"
status: draft
created: 2026-07-10
updated: 2026-07-10
keywords: [架构, 分层, 数据流, 目录结构]
covers_code:
  - README.md
  - CLAUDE.md
---

# 系统全景与分层架构

> 本文档描述 {PROJECT_NAME} 的产品定位、分层架构图、核心数据流、项目目录结构。

## 1. 产品定位

## 2. 分层架构

## 3. 核心数据流

## 4. 项目目录结构
```

---

## 章节：DIMENSION = "modules"（B2）

### 任务

深度扫描源码目录，识别模块边界，规划 `modules/` 下的模块文档，并给出模块间依赖关系。

### 读取范围

- `META_KB_GUIDE`
- `CLAUDE.md`
- 源码目录结构（`ls -R`）
- **少量代表性源文件的 include/import 头部**——目的是判断依赖方向，不是理解具体逻辑
- 头文件目录（如有集中式头文件目录）

### 决策依据

- 每个"模块"对应一份文档（`modules/<id>.md`）
- 模块粒度：一个模块 = 一个有清晰职责边界的功能单元（不是一个类，也不是整个子系统）
- `covers_code` 必须精确到具体文件路径，不用通配符（`modules/<id>.md` 会成为 Stage 3 sub-agent 的输入）
- `depends_on`：本模块依赖哪些其他模块的输出（数据或接口层面），基于 include/import 关系推断

### 依赖关系规则

- **允许**：DAG（有向无环图）
- **不允许**：环（A → B → A）——如果观察到环，仍如实标注，主 Agent 会在 Stage C 处理环
- **粒度**：只标注**强依赖**（本模块的核心功能依赖对方的输出），不标注偶尔的工具函数引用

### 输出

**结构化 JSON**（示意，实际输出全部模块清单。v2.0 起 modules 条目必须包含产品视角边界字段）：

```json
{
  "dimension": "modules",
  "docs": [
    {
      "id": "<module-id-1>",
      "title": "<Module 1 标题>",
      "path": "modules/<module-id-1>.md",
      "description": "<模块职责一句话>",
      "keywords": ["<关键词 1>", "<关键词 2>", "<关键词 3>"],
      "covers_code": [
        "<path/to/source-1>",
        "<path/to/source-2>"
      ],
      "depends_on": [],
      "responsibility": "<一到两句话，产品视角职责。必须剥离所有技术实现细节>",
      "capability_boundary": {
        "does": [
          "<该模块承诺提供的能力 1>",
          "<能力 2>"
        ],
        "does_not": [
          {"task": "<该模块不做的事 1>", "owner": "<负责该事的其他模块 id>"},
          {"task": "<该模块不做的事 2>", "owner": "<...>"}
        ]
      },
      "communicates_with": [
        {"module": "<对端模块 id>", "direction": "<调用|被调用|双向>", "purpose": "<通讯目的一句话>"}
      ],
      "system_privilege": "<权限级别，视工程而定，可选>",
      "lifecycle": "<触发方式与生命周期，视工程而定，可选>"
    },
    {
      "id": "<module-id-2>",
      "title": "<Module 2 标题>",
      "path": "modules/<module-id-2>.md",
      "description": "<模块职责一句话>",
      "keywords": ["<...>"],
      "covers_code": ["<...>"],
      "depends_on": ["<module-id-1>"],
      "responsibility": "<...>",
      "capability_boundary": {"does": [...], "does_not": [...]},
      "communicates_with": [...]
    }
  ],
  "circular_dependencies_observed": []
}
```

**格式要点**：
- `id` 使用 kebab-case（如 `rdb-persistence` / `order-service` / `data-ingestor`）
- `covers_code` 精确到具体文件路径，不用通配符
- `depends_on` 只标注**强依赖**（本模块核心功能依赖对方输出），基于 include/import 关系推断
- 若观察到循环依赖，在 `circular_dependencies_observed` 中列出（如 `[{"cycle": ["a", "b", "a"]}]`）——主 Agent 会在 Stage C 处理

**产品视角字段的填写要点**（v2.0 起为必填）：

- `responsibility`：**必须**剥离所有技术专有名词（框架名 / 协议名 / API 签名 / 数据结构名 / 内部术语）。示例：
  - ❌ "使用 RocksDB 存储路径映射"
  - ✅ "维护路径与文件 ID 的双向映射"
  - ❌ "通过集群总线接收集群消息"
  - ✅ "通过控制通道接收上层管理指令"
- `capability_boundary.does_not` 的每一项**必须**通过 `owner` 字段标注真正负责该 task 的模块 id
- `communicates_with` 每项三要素完整：`module`（对端 id） / `direction`（`调用` / `被调用` / `双向`） / `purpose`（一句话说明为什么通讯）
- `system_privilege` / `lifecycle` 视工程性质而定，无相关概念时可省略

**判定标准**：若一句表述包含技术专有名词，必须转换为产品语言。转换后仍不确定的，可以在句末追加 `[待确认]` 标记，主 Agent 收集时会归入 Journal。

**骨架文件**（每份文档）——章节骨架必须涵盖以下必填项（参照 `META_KB_GUIDE` §模块文档规范）：

```markdown
---
id: <module-id>
title: "..."
status: draft
created: <date>
updated: <date>
keywords: [...]
covers_code: [...]
depends_on: [...]
---

# <title>

> <description>

## 1. 模块职责
## 2. 数据模型
## 3. 关键 API
## 4. 业务流程
## 5. 设计决策与权衡
## 6. 已知陷阱
## 7. 变更影响评估
```

---

## 章节：DIMENSION = "flows"（B3）

### 任务

规划 `flows/` 下的跨模块端到端流程文档。

### 读取范围

- `META_KB_GUIDE`
- `CLAUDE.md`
- **B2 sub-agent 的输出**（模块清单，主 Agent 会传给你）
- 入口文件（`main.*` 等）
- Stage A3 观察到的关键流程列表

### 决策依据

- 每条**跨模块的端到端业务流程**对应一份文档
- 流程文档是**导航图**，不重复模块文档内容——通过 `involves_modules` 串联
- `involves_modules` 中的每个 id 必须在 B2 的 module docs 中存在

### 输出

**结构化 JSON**（示意）：

```json
{
  "dimension": "flows",
  "docs": [
    {
      "id": "<flow-id>",
      "title": "<Flow 标题>",
      "path": "flows/<flow-id>.md",
      "description": "<一句话概括流程起点/终点/核心价值>",
      "keywords": ["<关键词 1>", "<关键词 2>", "端到端"],
      "covers_code": [
        "<path/to/source-1>",
        "<path/to/source-2>",
        "<path/to/source-3>"
      ],
      "involves_modules": ["<module-id-1>", "<module-id-2>", "<module-id-3>"]
    }
  ]
}
```

**flow 命名建议**（视工程领域而定）：
- 系统级：`<verb>-<object>-flow.md`（如 `command-execution-flow.md`、`replication-sync-flow.md`）
- Web 应用：`<user-action>-flow.md`（如 `signup-flow.md`、`checkout-flow.md`）
- 数据管道：`<source>-to-<sink>-flow.md`（如 `ingest-to-warehouse-flow.md`）

**骨架文件**：

```markdown
---
id: <flow-id>
title: "..."
status: draft
created: <date>
updated: <date>
keywords: [...]
covers_code: [...]
involves_modules: [...]
---

# <title>

> <description>

## 1. 触发条件
## 2. 端到端时序
## 3. 涉及模块的分工
## 4. 异常路径
## 5. 相关模块文档
```

---

## 章节：DIMENSION = "troubleshooting"（B4）

### 任务

基于模块和流程知识，规划故障域索引和 FAQ 骨架。

### 读取范围

- `META_KB_GUIDE`
- B1/B2/B3 的输出
- Stage A3 观察到的错误处理体系
- CLAUDE.md 中的"问题排查"段落
- 若存在：现有的 log/error 相关文档或代码

### 输出

**结构化 JSON**（结构与前 3 个维度不同，因 troubleshooting 是"设施"而非"文档清单"）：

```json
{
  "dimension": "troubleshooting",
  "setup": {
    "symptom_index_file": "troubleshooting/symptom-index.yaml",
    "faq_file": "troubleshooting/faq.md",
    "known_bugs_file": "troubleshooting/known-bugs.md",
    "runbooks_dir": "troubleshooting/runbooks/",
    "incidents_dir": "troubleshooting/incidents/"
  },
  "initial_symptoms_identified": [
    {"id": "replica-data-lagging", "trigger_keywords": ["从库读到旧数据", "主从延迟"], "runbook": "TBD"},
    {"id": "persistence-latency-spike", "trigger_keywords": ["bgsave 期间卡顿", "每秒延迟尖峰"], "runbook": "TBD"}
  ],
  "notes": "identified 3-5 symptom candidates; runbook 待 Stage 3 或后续人工补齐"
}
```

**骨架文件**：

- `troubleshooting/symptom-index.yaml`：YAML 文件，写入结构化的空骨架 + 已识别的候选症状
- `troubleshooting/faq.md`：Markdown 骨架，含 frontmatter + 一级标题 + blockquote + 空章节 "常见问题（待补充）"
- `troubleshooting/known-bugs.md`：Markdown 骨架
- `troubleshooting/runbooks/` 和 `troubleshooting/incidents/`：确保目录存在

---

## 通用错误处理

- 若无法识别任何模块（如源码目录空、只有配置文件）：返回 `{"error": "no_modules_identified", "detail": "..."}`，主 Agent 会决定是否终止
- 若 B3 收到的 B2 输出中模块数为 0：不生成任何 flow 文档，返回空 `docs` 数组，`notes` 说明原因
- 骨架文件写入失败：返回 `{"error": "file_write_error", "path": "...", "detail": "..."}`

---

## 输出规则总结

1. 返回值 = 单个 JSON 对象（不加前后缀文本、不用 ```json 包裹）
2. JSON 必须符合 `schemas/stage-b-output.schema.json`
3. 所有 path 使用相对于 `KB_ROOT` 的路径
4. 所有 id 使用 kebab-case（`system-overview`、`rdb-persistence`）
