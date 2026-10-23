# Indexable Docs KB Seed · Goal 1: kb-primer

> **执行方式**：本文档由 `/goal` 驱动执行。主 Agent 逐 Stage 推进，分派 sub-agent 完成具体任务。
> **禁止**：主 Agent 亲自读源码分析。所有源码分析工作必须通过 sub-agent 完成。
>
> **元认知参考**：本任务文档回答"怎么做"（step-by-step 指令）。若需要理解**"为什么这样做"**——如为什么规模判定要取最大值、为什么要 Journal + Progress Log 分离、依赖环路降级为什么按入度最少优先——请先读 `DESIGN-planning.md`（v8-ready）。遇到边缘情况需要合理判断时，"why" 的理解比死板套用指令更重要。

---

## Goal 定义

**goal_id**: `kb-primer`
**goal_name**: 知识库构建的分析与规划阶段
**seed_pack_version**: `<从 .think-tank/indexable-docs-kb-seed/VERSION 读取>`

**核心产出**：一份参数完全实例化的 Goal 2 任务文档 `bake-{project}.md`，包含所有 sub-agent 分派参数，可被 `/goal` 直接执行。

---

## Goal 条件（GOAL_MET）

以下所有条件同时满足时，本 Goal 达成：

1. 4 份元规范文件已存在且内容自洽：
   - `.think-tank/knowledge-base/.meta-knowledge-base-guide.md`
   - `.think-tank/knowledge-base/works/.meta-work-guide.md`
   - `.think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md`
   - `.think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md`
2. 所有文档骨架文件已存在（frontmatter + 描述 blockquote + 空章节标题）
3. `.think-tank/knowledge-base/index.yaml` 完整（含所有 A/F/M/T 条目 + `meta.seed_pack_version` 等版本契约字段）
4. `.think-tank/knowledge-base/.meta-maintain-beacon.yaml` 已创建（commit = HEAD）
5. `.think-tank/knowledge-base/.meta-explore-issues.jsonl` 已创建（空文件）
6. `.think-tank/knowledge-base/pass/build-findings/discovered-issues.jsonl` 已创建（可为空文件）
7. `bake-{project}.md` 已生成，包含：
   - 所有确定性参数（文档清单、分批策略、并发数、Review 轮次、收敛标准）
   - 头部含"预计成本"章节（sub-agent 数、token 估算、耗时估算）
8. 本文档的 Progress Log 中，Stage A/B/C 三行状态均为 ✅

达成后主 Agent 输出：

```
GOAL_MET: kb-primer 完成。
  规模：{scale}
  文档：架构 {K} 份 / 模块 {N} 份 / 流程 {M} 份 / 故障 {T} 项
  已排除路径：{E} 项（vendor / 第三方 / 生成产物 / submodule，详见 index.yaml.meta.excluded_paths）
  预计 Goal 2：{X} 个 sub-agent / {Y} 小时
  Goal 2 任务文档：bake-{project}.md 已就绪

  种子包完整构建周期为 Goal 1 → Goal 2 → Goal 3 → Goal 4。当前 Goal 1 完成，请人类 review 文档清单和预计成本，确认后启动 Goal 2；后续 Goal 2 → 3 → 4 会连续推进（除非人类显式跳过 Goal 4）。
```

---

## 出工条件（GOAL_ABANDONED）

以下任一条件触发即终止本 Goal：

- 累计轮次 > 50
- 单个 sub-agent 连续 3 次失败仍无进展
- 门禁检查失败（见 §0）

终止时主 Agent 输出：`GOAL_ABANDONED: {原因}`，并在 Journal 追加 `action: goal_failed` 条目。

---

## 0. 门禁检查（Pre-flight）

**必须在进入 Stage A 之前执行。任一检查失败即终止，不进入任何 Stage。**

### 检查 1：KB 是否已存在

```
条件：.think-tank/knowledge-base/index.yaml 存在
含义：目标工程已构建过知识库
行为：终止执行，输出：
  "工程已有知识库（.think-tank/knowledge-base/index.yaml 存在），无需重新构建。
   如需更新现有知识库，请使用 /maintain-kb 进行增量或全量维护。
   如需彻底重建，请先删除现有 .think-tank/knowledge-base/ 目录。"
```

### 检查 2：是否为 git 仓库根目录

```
条件：.git 目录存在于当前工作目录
含义：任务文档须以工程根目录为工作目录被 `/goal` 驱动（文档内相对路径基于工程根解析）
行为：不满足时终止执行，输出：
  "当前目录不是 git 仓库根目录（未检测到 .git）。
   请先 cd 到含 .git 的工程根目录，再执行 /goal .think-tank/indexable-docs-kb-seed/primer.md"
```

### 检查 3：种子包完整性

```
条件：以下路径全部存在
  .think-tank/indexable-docs-kb-seed/VERSION
  .think-tank/indexable-docs-kb-seed/prompts/stage-a-subagent.md
  .think-tank/indexable-docs-kb-seed/prompts/stage-b-subagent.md
  .think-tank/indexable-docs-kb-seed/schemas/stage-b-output.schema.json
  .think-tank/indexable-docs-kb-seed/schemas/qa-jsonl-entry.schema.json
  .think-tank/indexable-docs-kb-seed/schemas/index-impact.schema.json
  .think-tank/indexable-docs-kb-seed/schemas/discovered-issue.schema.json
  .think-tank/indexable-docs-kb-seed/templates/meta-knowledge-base-guide.md.template
  .think-tank/indexable-docs-kb-seed/templates/kb-review-ledger.md.template
  .think-tank/indexable-docs-kb-seed/verify-build-findings.md
行为：任一缺失即终止，输出：
  "种子包不完整。缺失文件：{list}。请重新完整复制种子包。"
```

**通过后**：在 Journal 追加 `action: preflight_passed` 条目，正式进入 Stage A。

---

## Stage A: 工程理解 + 元规范推导

### 目标

三个维度并行观察工程，各自输出适配参数与规模标签。主 Agent 汇总后判定工程规模、生成元规范文件、创建 KB 目录骨架。

### A.1 分派工程理解 sub-agents（并行 3 个）

分派三个 sub-agent，prompt 模板来自 `.think-tank/indexable-docs-kb-seed/prompts/stage-a-subagent.md`，通过参数区分维度：

| Sub-agent | 维度 | 关键输入 | 产出 |
|-----------|------|---------|------|
| `kb-primer-A1` | architecture | CLAUDE.md, README, 顶层目录, 关键配置, `.gitignore`, `.gitmodules`, 包管理器配置 | 适配参数 + 规模标签 + **excluded_paths 清单** |
| `kb-primer-A2` | modules | CLAUDE.md, 源码目录结构, include/import 关系 | 适配参数 + 规模标签（依据：模块数/跨模块引用/耦合度） |
| `kb-primer-A3` | flows-troubleshooting | CLAUDE.md, 入口文件, 日志配置, 错误处理代码 | 适配参数 + 规模标签（依据：入口点/关键路径/错误处理复杂度） |

**分派后**：Journal 追加 `action: dispatch` 条目，含 3 个 sub-agent 的 name + agent_id。

### A.2 收集结果 + 判定工程规模 + 汇总 excluded_paths

三个 sub-agent 全部返回后：

1. 收集每个 sub-agent 的规模标签（small / medium / large）
2. **取最大值**（保守优先，不用中位数）——理由：低估规模会导致 Stage 3 分批不足、Review 轮次不够、并发过高，纠错成本远高于高估
3. 判定结果写入 Journal `action: stage_completed` 条目的 `final_scale` 字段
4. **收集 A1 的 `excluded_paths`**：汇总到主 Agent 的会话记忆中，后续 Stage B/C 都会使用；规模判定时**仅统计非排除路径下的源文件**
5. **构建期问题记录**：确保 `.think-tank/knowledge-base/pass/build-findings/` 与空 `discovered-issues.jsonl` 存在，收集 A1/A2/A3 输出顶层的 `discovered_issues`，分配 `DISC-xxxx`，追加到该文件；只记录，不确认、不修复、不阻塞

**规模判定表**（任一维度触发即认定该级别）：

| 规模 | 判定条件 | 后续策略 |
|------|---------|---------|
| **small** | 模块 ≤ 3 **且** 源文件 < 50 **且** 语言 ≤ 2 **且** 分层 ≤ 2 **且** 低耦合 | Stage 3 不分批，Stage 4 最多 2 轮 Review |
| **medium** | 模块 4-15 **或** 源文件 50-300 **或** 语言 3-4 **或** 分层 3 **或** 中耦合 | Stage 3 按拓扑分批 ≤ 8/批，Stage 4 最多 5 轮 |
| **large** | 模块 > 15 **或** 源文件 > 300 **或** 语言 ≥ 5 **或** 分层 ≥ 4 **或** 高耦合 | Stage 3 按拓扑分批 ≤ 6/批，Stage 4 最多 5 轮 |

### A.2.1 excluded_paths 人类 review 检查点（软提醒）

汇总 excluded_paths 后，主 Agent 向人类**软提醒**（不阻断执行），格式如下：

```
📋 Stage A 识别的排除路径清单（第三方源码 / 生成产物 / submodule）：

高置信度（推荐排除）：
  • vendor/         第三方 Go 依赖，从 go.mod dependencies 观察
  • ext/            C/C++ 第三方库（boost/、cjson/ 等子目录）

需要 review（需要你确认是否排除）：
  • examples/       示例代码，非本工程核心（不确定）
  • docs/generated/ 疑似 Swagger 生成产物（不确定）

如需调整（增删任何路径），请在启动 Stage B 前告诉我；否则将按此清单继续执行。
```

**行为规则**：
- 使用 `AskUserQuestion` 或直接在回答中列出清单，让人类可随时干预但**不主动阻断**；
- 若用户明确回应"确认/继续/OK"，将 `needs_human_review` 全部改为 `false`，正式进入 Stage B；
- 若用户提出调整（增删、修改），主 Agent 更新会话记忆中的 excluded_paths，然后再进入 Stage B；
- 若用户未回应，等待默认约 5 轮 turn 后自动进入 Stage B（清单保持 A1 的判定）。

**注意**：本节所有"排除"决策必须记入 Journal，格式：`{action: decision, decision: excluded_paths_finalized, count: N, high_certainty: M, human_reviewed: <true|false>}`。

### A.3 生成元规范文件

分派一个元规范生成 sub-agent（prompt 见 `prompts/stage-a-subagent.md` 的 "meta-generation" 章节）：

- 输入：A1/A2/A3 的适配参数 + 工程规模 + `templates/meta-knowledge-base-guide.md.template`
- 任务：识别模板中所有 `{ADAPT: ...}` 标记，根据工程特征填充具体内容或标注"本工程不适用"
- 产出：
  - `.think-tank/knowledge-base/.meta-knowledge-base-guide.md`（含工程适配的目录结构、分层描述、术语定义）
  - `.think-tank/knowledge-base/works/.meta-work-guide.md`（复制自 `templates/meta-work-guide.md`，直接可用）
  - `.think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md`（复制自模板，直接可用）
  - `.think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md`（复制自模板，直接可用）

### A.4 创建 KB 目录骨架

主 Agent 直接执行（不需 sub-agent）：

```
.think-tank/knowledge-base/
├── architecture/            # 空目录
├── modules/                 # 空目录
├── flows/                   # 空目录
├── troubleshooting/
│   ├── runbooks/            # 空目录
│   └── incidents/           # 空目录
├── works/                   # 已含 .meta-work-guide.md
├── develop/                 # 空目录
└── pass/                    # 空目录
    └── build-findings/      # 构建期问题账本（Goal 2/3/4 继续写）
        └── discovered-issues.jsonl  # 空文件
```

创建 `pass/build-findings/` 与空 `discovered-issues.jsonl`；字段规范见 `schemas/discovered-issue.schema.json`。Goal 1 只记录，不确认、不修复。

### A.5 Stage A 收尾

- 验证 4 份元规范文件存在
- 更新本文档 Progress Log：`Stage A: ✅`
- Journal 追加 `action: stage_completed`

---

## Stage B: 文档清单规划

### 目标

分派 4 个 sub-agent 各负责一个维度（A/F/M/T），产出文档清单 + 骨架文件 + 每份文档的 frontmatter。

### B.1 分派文档清单规划 sub-agents（并行 4 个）

Prompt 模板来自 `prompts/stage-b-subagent.md`，通过 `dimension` 参数区分。**每个 sub-agent 都必须收到 `EXCLUDED_PATHS` 参数**（Stage A.2 汇总后的清单，且已经过人类 review 检查点）：

| Sub-agent | 维度 | 关键输入 | 产出 |
|-----------|------|---------|------|
| `kb-primer-B1` | architecture | .meta-knowledge-base-guide.md + 高层源码 + `EXCLUDED_PATHS` | architecture/ 下所有文档的 {id, title, path, description, keywords, covers_code} + 骨架文件 |
| `kb-primer-B2` | modules | 深度扫描源码目录 + include/import + `EXCLUDED_PATHS` | modules/ 下所有文档 + `depends_on` 字段 + 骨架文件（自动跳过 EXCLUDED_PATHS 下的目录） |
| `kb-primer-B3` | flows | B1/B2 产出 + 入口点分析 + `EXCLUDED_PATHS` | flows/ 下所有文档 + `involves_modules` 字段 + 骨架文件 |
| `kb-primer-B4` | troubleshooting | B1/B2/B3 产出 + 错误处理代码 + `EXCLUDED_PATHS` | symptom-index.yaml + faq.md + known-bugs.md 骨架 |

**输出格式**：每个 sub-agent 必须按 `schemas/stage-b-output.schema.json` 输出结构化数据。

**并发上限**：按 §A.2 的规模判定，small 不限 / medium ≤ 8 / large ≤ 6。Stage B 只有 4 个 sub-agent，任何规模都能一次性并发。

### B.2 收集结果 + 冲突检查

主 Agent 收集 4 份输出后：

1. **跨维度冲突检查**：两个维度是否规划了重叠的文档 id 或 path？重叠即失败，重新分派受影响的 sub-agent
2. **依赖完整性检查**：`depends_on` / `involves_modules` 中引用的所有 id 必须在文档清单中存在
3. **依赖环路检测**（module_docs）：
   - 用 DFS 拓扑排序，遇到 back edge 即认定存在环
   - **降级顺序**：
     1. 入度最少打破环：找出环中入度最少的节点，清空其 `depends_on`
     2. 按目录分批：若步骤 1 无法解环，改用目录路径作为分批依据
     3. 单批全量：若目录也无清晰边界，降级为单批全量分派（仅 small 工程；中/大工程走到此步需人类介入）
   - 环处理决策必须写入 Journal：`action: decision, decision: cycle_broken, cycle: [A→B→A], resolution: {strategy}, node_cleared: {node}`
4. **excluded_paths 交叉验证**：扫描所有 sub-agent 产出的 `covers_code`，若发现任何路径落在 `EXCLUDED_PATHS` 下（Stage B sub-agent 违反约束的情况），记入 Journal `{action: error, error_type: covers_code_hits_excluded, path: <...>, module: <id>}`，主 Agent 直接从对应模块的 covers_code 中剔除该路径，若剔除后该模块的 covers_code 为空则整模块从清单剔除
5. **构建期问题记录**：收集 4 份输出顶层的 `discovered_issues`，分配 `DISC-xxxx`，追加到 `.think-tank/knowledge-base/pass/build-findings/discovered-issues.jsonl`；只记录，不确认、不修复、不阻塞

### B.3 生成 index.yaml + beacon + explore-issues

主 Agent 直接执行：

**index.yaml** 顶部必须包含版本契约字段：

```yaml
meta:
  seed_pack_version: "<读自 .think-tank/indexable-docs-kb-seed/VERSION>"
  seed_pack_build: "<种子包发布日期，从 CHANGELOG.md 首个版本节读取>"
  kb_schema_version: "1.0"
  built_at: "<当前 ISO 8601 时间戳>"
  built_project_scale: "<small|medium|large>"
  project_name: "<工程名>"
  excluded_paths:                    # v1.1.2 新增，Stage A.2 汇总的清单
    - path: "vendor/"
      reason: "Go vendor 依赖"
      certainty: high
      needs_human_review: false
    - path: "ext/"
      reason: "C/C++ 第三方库"
      certainty: high
      needs_human_review: false
    # ... 更多条目

architecture:
  - id: ...
    title: ...
    path: ...
    description: ...
    keywords: [...]
    covers_code: [...]
modules:
  - id: ...
    depends_on: [...]
    ...
flows:
  - id: ...
    involves_modules: [...]
    ...
```

**为什么 excluded_paths 要持久化到 `index.yaml.meta`**：
- Stage 3 / Stage 4 sub-agent 从 index.yaml 就能读到；
- Goal 3 Stage 8 module_header 上浮时可读；
- 后续 `/maintain-kb` 增量检查时同样使用（若代码变更命中排除路径，maintain-kb 自动跳过而不误报）；
- 未来种子包升级到 v1.2 时可对照该字段做兼容处理。

**.meta-maintain-beacon.yaml**：

```yaml
last_maintain_commit: "<`git rev-parse HEAD` 的输出>"
last_maintain_at: "<当前 ISO 8601 时间戳>"
last_maintain_mode: "seed_init"
```

**.meta-explore-issues.jsonl**：创建空文件。

### B.4 Stage B 收尾

- 验证 index.yaml、beacon、explore-issues 三个文件存在
- 验证所有骨架文件存在（架构 + 模块 + 流程 + 故障索引）
- 更新本文档 Progress Log：`Stage B: ✅`
- Journal 追加 `action: stage_completed`

---

## Stage C: 实例化 Goal 2 任务文档

### 目标

将 Stage A + B 的所有产出转化为 Goal 2 的**完全确定的执行指令**。

### C.1 主 Agent 汇总所有参数

主 Agent 汇集以下参数（不需 sub-agent，参数量 < 50 项时主 Agent 直接完成——见 planning §13 Q1）：

| 参数类别 | 参数 | 来源 |
|---------|------|------|
| 版本 | `seed_pack_version` | `VERSION` 文件 |
| 项目 | `project_name` | `kb.meta.project_name` |
| 项目 | `project_scale` | Stage A.2 判定 |
| 排除路径 | `excluded_paths` | Stage A.2 汇总 + 人类 review 后确定 |
| 文档清单 | `architecture_docs` | Stage B - B1 |
| 文档清单 | `module_docs`（含 `depends_on`） | Stage B - B2 |
| 文档清单 | `flow_docs`（含 `involves_modules`） | Stage B - B3 |
| 文档清单 | `troubleshooting_setup` | Stage B - B4 |
| 分批策略 | `stage3_batches` | 主 Agent 根据 `depends_on` 拓扑 + `project_scale` 计算 |
| 并发数 | `batch_concurrency` | small: 无限制 / medium: 8 / large: 6 |
| Review 范围 | `review_docs` | 全部 A/F/M 文档 id 列表；Round 1 全量，Round 2+ 按 coverage 定向 |
| Review 轮次 | `review_max_rounds` | small: 2 / medium: 5 / large: 5（全局安全上限） |
| 定向轮次 | `targeted_rounds_per_doc` | 3（单文档一个未收敛周期内的上限） |
| 收敛标准 | `convergence_criteria.critical_max` | 0 |
| 收敛标准 | `convergence_criteria.medium_max` | 0 |
| 收敛标准 | `convergence_criteria.require_next_coverage_empty` | true |
| 账本 | `ledger.*` | `.think-tank/knowledge-base/pass/kb-review-ledger.md` 等 |
| QA 路径 | `qa_jsonl_pattern` | `.think-tank/knowledge-base/pass/{doc_id}.qa.jsonl`（硬编码，v2.0.1 起严格要求完整路径） |
| 出工条件 | `max_turns` | small: 30 / medium/large: 50 |

参数结构详见 `schemas/goal2-task-doc.schema.yaml`。

### C.2 计算 Stage 3 分批方案

主 Agent 基于 `module_docs.depends_on` 做拓扑排序：

```
batches = []
remaining = all_module_docs
while remaining:
  # 找出所有 depends_on 已全部在已完成集合中的模块
  ready = [m for m in remaining if all(d in completed for d in m.depends_on)]
  if not ready:
    # 遇到环，触发 §B.2 的降级流程
    ...
  # 按并发上限拆批
  batch_size = min(len(ready), batch_concurrency)
  batches.append(ready[:batch_size])
  completed.update(ready[:batch_size])
  remaining -= ready[:batch_size]
```

**architecture 和 flows 文档如何入批**：
- architecture 文档一般依赖较少，作为 batch 1 的一部分
- flows 文档依赖 modules（通过 `involves_modules`），放到所有涉及模块完成后的批次

### C.3 计算成本估算

主 Agent 计算以下值并写入 Goal 2 任务文档头部：

```
N_a = architecture 文档数
N_m = module 文档数
N_f = flow 文档数
N_total = N_a + N_m + N_f
R = review_max_rounds

sub_agent_stage3 = N_total
sub_agent_stage4_round1 = N_total
sub_agent_stage4_targeted_expected ≈ N_total × 0.75   # 经验值，按 coverage 收缩
sub_agent_stage4_max = N_total × R                    # 全局上限
sub_agent_total_expected = N_total + sub_agent_stage4_round1 + sub_agent_stage4_targeted_expected + 12
sub_agent_total_upper = N_total + sub_agent_stage4_max + 12   # +12 是 Goal 1 + Goal 3 的开销

token_input_est_upper = (N_total + sub_agent_stage4_max) × 100K
token_output_est_upper = (N_total + sub_agent_stage4_max) × 10K

耗时估算（按并发充分）：
  small (N<8, R=2):  45min - 1.5h
  medium (N~15, R=5): 2-4h
  large (N>25, R=5):  4-8h

占位符映射：
  SUBAGENT_TOTAL = sub_agent_total_upper
  SUBAGENT_STAGE4 = sub_agent_stage4_max
  TARGETED_ROUNDS_PER_DOC = targeted_rounds_per_doc（默认 3）
```

### C.4 填充 Goal 2 任务文档模板

主 Agent 读取 `bake.md.template`（位于种子包根，即 `.think-tank/indexable-docs-kb-seed/bake.md.template`），逐一替换所有 `{{PLACEHOLDER}}`，生成：

**输出路径**：`.think-tank/indexable-docs-kb-seed/bake-{project_name}.md`

（放在种子包目录下的原因：Goal 2 是"种子包的实例化产物"，与 primer/infra-setup 同级便于人类查看和 review）

### C.5 Stage C 收尾

- 验证 `bake-{project_name}.md` 存在且所有 `{{PLACEHOLDER}}` 已替换（grep `{{` 应无匹配）
- 更新本文档 Progress Log：`Stage C: ✅`
- Journal 追加 `action: goal_completed`

---

## Progress Log

<!-- 唯一权威状态源。每 Stage 完成时重写对应行。sub-agent 明细见 pass/kb-seed-journal-kb-primer.md -->

| Stage | 状态 | 开始时间 | 完成时间 | 耗时 | 备注 |
|-------|------|---------|---------|------|------|
| 0. 门禁 | ⬜ | | | | |
| A. 工程理解 | ⬜ | | | | |
| B. 文档清单 | ⬜ | | | | |
| C. 实例化 Goal 2 | ⬜ | | | | |

**Goal 状态**：⬜ pending → ⏳ in_progress → ✅ completed / ❌ failed

**当前状态**：⬜ pending

---

## 恢复协议

如果你是被唤醒继续执行本 Goal 的新 Agent：

1. **读取本文档 Progress Log**（唯一权威状态源）→ 找到最后一个 status ≠ ✅ 的 Stage
2. **读取 `pass/kb-seed-journal-kb-primer.md` 的 Entries**：找到该 Stage 内 sub-agent 的分派/完成/失败明细
3. **对每个 pending 的 sub-agent**：
   - 若条目中有 agent_id：session 大概率已崩溃，直接按相同 prompt 重新分派，写入新 agent_id
   - 若无 agent_id：从未分派，重新构造 prompt 并分派
4. **对每个 failed 的 sub-agent**：retry_count < 2 则重试，≥ 2 则标记 abandoned 跳过
5. **验证文件系统**：Journal 声称已产出的文件必须真实存在，否则降级为 failed 重新分派
6. **禁止依赖原对话上下文**：Progress Log + Journal + 文件系统 = 全部信息来源

---

## 附录：Journal 写入规范

**Journal 文件路径**：`.think-tank/knowledge-base/pass/kb-seed-journal-kb-primer.md`

**格式**：追加式流水，不聚合状态。每个 sub-agent 事件写一条 entry。字段规范见 planning §9.4、§9.5。

**action 类型**：`dispatch` / `result_collected` / `retry_dispatched` / `retry_exhausted` / `error` / `decision` / `stage_completed` / `goal_completed` / `goal_failed`
