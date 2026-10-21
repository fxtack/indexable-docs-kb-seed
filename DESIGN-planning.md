# AFMT-KB-SEED 任务规划方案

> 本文档描述如何**规划**以 `/goal` 驱动的 AFMT-KB-SEED（Architecture/Flows/Modules/Troubleshooting Knowledge Base Seed）任务。
> 它不产出 KB，而是产出一套**可分发到其他工程的、结构化的任务描述文档**，目标工程的 Agent 读取后能自主构建等价的知识库体系。
>
> 本文档属于"规划的规划"——回答的问题不是"种子任务怎么执行"，而是"种子任务文档应该怎么设计，要覆盖哪些维度，采用什么结构，以及这套文档如何被 Goal/Agent/Sub-agent 体系驱动执行"。
>
> **当前状态**：v8-ready（含 v2.0.0 破坏性重构 + v2.0.2 决策机制精简 + v2.0.3 Goal 4 定位调整 + v2.0.4 Commander 模板化迁移恢复）。v1.1.0 引入 Commander skill；v1.1.1 泛化 prompt 示例；v1.1.2 新增第三方源码排除机制；v1.2.0 新增 Goal 4 QA 分诊 + Stage 5 清理 jsonl；**v2.0.0 破坏性重构**——**消除 `.think-tank/commander/` 目录**，产品视角边界字段（`responsibility` / `capability_boundary` / `communicates_with`）直接内嵌于 `index.yaml.modules[]`，Goal 1 Stage B2 直接产出，Goal 3 Stage 8「Commander 内容构建」完全移除。KB 成为单一事实来源。**v2.0.2**：移除历史决策记录机制（`decisions/` 目录）。**v2.0.3**：Goal 4 从可选改为默认标准流程。**v2.0.4**：Commander skill 重新打包进种子包（`skills/commander/`），模板已去 module_header 化和去 decisions 化，Goal 3 Stage 7 恢复完整的模板迁移流程。

---

## 1. 问题定义

### 1.1 输入与产出

```
输入：一套已建成的同构知识库（结构 + 规范 + Skill + 行为模式 + 实际构建流程经验），本种子包的示例取自 Redis 8.10.2
产出：一套 AFMT-KB-SEED 任务文档，包含：
  - AFMT-KB-SEED-primer.md（Goal 1 任务文档：理解工程 → 产出实例化执行计划）
  - AFMT-KB-SEED-bake.md.template（Goal 2 任务文档模板：确定性执行 → 产出完整 KB）
  - AFMT-KB-SEED-infra-setup.md（Goal 3 任务文档：基础设施搭建 + Skill 迁移）
  - 配套的模板文件、Prompt 模板、Schema 定义
```

### 1.2 核心约束：Goal 驱动多 Agent 编排的确定性要求

这是驱动整个方案设计的最关键约束。

`/goal` 模式下的主 Agent 行为模型：

```
每轮循环：
  读任务文档 → 按文档规定的规则做决策 → 分派 sub-agent → 等待结果 → 输出摘要 → 下一轮
```

**能做**（确定性决策）：
- "如果 Stage N 的所有 sub-agent 已完成，进入 Stage N+1"
- "如果 Review 轮次 < 5 且不满足收敛标准，再发起一轮 Review"
- "如果 sub-agent 失败，重试最多 2 次"

**不能做**（需要根据 sub-agent 返回值动态决策）：
- "sub-agent 返回了模块列表，让我根据列表的**具体内容**动态决定下一批 sub-agent 的数量、参数和分派策略"
- "sub-agent 发现意外的架构分层，让我**调整**文档生成策略"

这意味着：当主 Agent 决定分派下一批 sub-agent 时，它必须**提前知道**分派多少个、每个的参数是什么。但这些参数（模块数量、流程数量、依赖拓扑、文档清单）恰恰是需要通过分析才能知道的。

**这就是鸡生蛋蛋生鸡问题**：

```
需要分析工程才能知道有多少文档要生成
    ↓
但 Goal 要求主 Agent 提前知道要分派多少 sub-agent
    ↓
但主 Agent 不能亲自分析（上下文污染）
    ↓
也不能根据 sub-agent 的返回值动态决定（Goal 约束）
```

### 1.3 解决方案：多 Goal 模型

```
Goal 1: kb-primer（分析 + 规划）
  输入：通用方法论文档 + 目标工程源码
  过程：理解工程 → 生成元规范 → 识别模块/流程/架构文档清单 → 规划依赖拓扑
  输出：一份完全实例化的 Goal 2 任务文档

Goal 2: kb-docs-bake（确定性执行）
  输入：Goal 1 产出的实例化任务文档
  过程：按文档中的确定性指令逐 Stage 执行
        分派 sub-agent → 文档内容生成 → 多轮 Review → 收敛 → QA 聚合
  输出：完整 KB + kb-build-qa.md

Goal 3: kb-infra-setup（基础设施搭建）
  输入：Goal 2 产出的完整 KB
  过程：目录创建 → Skill 迁移 → 元文档移植 → 收尾（postmortem + summary）
  输出：就绪的 KB 运行环境
```

---

## 2. 门禁检查（Pre-flight）

任何 Goal 启动之前，主 Agent 必须先通过以下两项检查。两项检查**不通过即终止**，不进入任何 Goal。

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
含义：AFMT-KB-SEED 任务文档预期放置在工程根目录下执行
行为：不满足时终止执行，输出：
  "当前目录不是 git 仓库根目录（未检测到 .git）。
   请将 AFMT-KB-SEED 任务文档放置到工程根目录下，然后重新执行。"
```

**通过后**：主 Agent 可正式进入 Goal 1（kb-primer）。

---

## 3. Goal 1: kb-primer（分析 + 规划）

### 3.1 定位

Goal 1 的任务不是构建 KB，而是**为构建 KB 生成一份精确的执行计划**。这份执行计划本身将成为 Goal 2 的任务文档。

### 3.2 Goal 1 的 Stage 设计

```
Stage A: 工程理解 → 元规范推导
  └── Sub-agents 分维度理解工程，生成 .meta-knowledge-base-guide.md 等元规范

Stage B: 文档清单规划
  └── Sub-agents 识别 A/F/M/T 各维度应有哪些文档，产出文档清单 + 简短描述

Stage C: 实例化 Goal 2 任务文档
  └── 主 Agent 基于 Stage A+B 的产出，填充 Goal 2 任务文档模板的所有参数
  └── 产出：一份参数完整的 AFMT-KB-SEED-bake-{project}.md
```

### 3.3 Stage A: 工程理解 + 元规范推导

**不做集中化的"工程理解"步骤**。每个 Stage A 的 sub-agent 各自读 CLAUDE.md / README / 目录结构建立自己的理解。不需要一个独立的 Stage 0 来生产"工程画像报告"再传递给后面的 sub-agent——那样反而增加了主 Agent 的中转负担。

**Sub-agent 分派**：

```
Sub-agent A1（架构维度）：
  - 任务：理解工程的系统分层、技术栈、核心组件
  - 输入：CLAUDE.md / README / 顶层目录 / 关键配置文件
  - 产出 A1-1：architecture/ 维度的初步观察（分层方式、语言分布、部署形态）
  - 产出 A1-2：基于观察推导 .meta-knowledge-base-guide.md 中 architecture/ 相关的适配参数
    （如：是否有内核态/用户态分层需要体现在模板中、技术栈跨度多大影响模板章节）
  - 产出 A1-3：**架构维度规模标签**（small / medium / large）+ 判定依据（语言数、分层数、部署形态复杂度）

Sub-agent A2（模块维度）：
  - 任务：理解工程的模块划分方式、模块间依赖的复杂度
  - 输入：CLAUDE.md / 源码目录结构 / include/import 关系
  - 产出 A2-1：modules/ 维度的初步观察（模块粒度、耦合度、是否有明显的层级）
  - 产出 A2-2：基于观察推导 .meta-knowledge-base-guide.md 中 modules/ 相关的适配参数
    （如：模块文档的 depends_on 策略、数据模型章节的粒度建议）
  - 产出 A2-3：**模块维度规模标签**（small / medium / large）+ 判定依据（模块数、跨模块引用数、耦合度）

Sub-agent A3（流程/排查维度）：
  - 任务：理解工程的运行时特征（入口点、关键路径、日志/错误处理体系）
  - 输入：CLAUDE.md / 入口文件 / 日志配置 / 错误处理代码
  - 产出 A3-1：flows/ 和 troubleshooting/ 维度的初步观察
  - 产出 A3-2：基于观察推导 .meta-knowledge-base-guide.md 中 flows/ 和 troubleshooting/ 相关的适配参数
  - 产出 A3-3：**流程维度规模标签**（small / medium / large）+ 判定依据（入口点数、关键路径条数、错误处理体系复杂度）
```

**主 Agent 汇总 Stage A**：
- 收集 A1/A2/A3 的适配参数
- **工程规模判定**：基于 A1/A2/A3 的观察，从多个维度综合判断规模。任一维度触发即认定该级别（"最大者胜"，保守优先），避免单一维度失真（如 Go/Rust 工程或微服务架构用源文件数会判断偏小）：

| 规模 | 判定条件（**任一满足**即认定） | 后续策略调整 |
|------|-------------------------------|-------------|
| **小型** | 模块 ≤ 3 **且** 总源文件 < 50 **且** 语言 ≤ 2 种 **且** 分层 ≤ 2 层 **且** 模块耦合度低（跨模块引用 < 20 处） | Stage 3 不分批（全部一次性分派），Stage 4 只做 2 轮 Review |
| **中型** | 模块 4-15 **或** 源文件 50-300 **或** 语言 3-4 种 **或** 分层 3 层 **或** 中等耦合 | 按依赖拓扑分批，每批 ≤ 8 个 sub-agent（标准流程） |
| **大型** | 模块 > 15 **或** 源文件 > 300 **或** 语言 ≥ 5 种 **或** 分层 ≥ 4 层 **或** 高耦合（跨模块引用密集，无清晰边界） | 按依赖拓扑分批，每批 ≤ 6 个 sub-agent（控制并发），Review 最多 5 轮 |

**判定规则**：
- A1/A2/A3 每个 sub-agent 从自己的维度视角各自给出规模标签（small / medium / large）
- 主 Agent 取**最大值**（保守优先，不用中位数）——理由：低估规模会导致 Stage 3 分批不足、Review 轮次不够、并发过高，纠错成本远高于高估
- 规模的判定**必须在 Stage A 完成时就确定**，因为 Goal 1 的 Stage C 需要据此调整 Goal 2 的模板参数
- 分派一个专门的 sub-agent 来基于适配参数生成元规范文件：
  - `.meta-knowledge-base-guide.md`
  - `.meta-work-guide.md`
  - `.meta-runbook-guide.md`
  - `.meta-incident-guide.md`
- 验证：检查所有文件存在 + 内容自洽
- 创建 KB 目录骨架

### 3.4 Stage B: 文档清单规划

**目标**：决定 A/F/M/T 四个维度各有哪些文档。每个文档一句话描述 + frontmatter 占位。

**Sub-agent 分派**：

```
分派 4 个 sub-agent（各负责一个维度）：

Sub-agent B1（Architecture）：
  - 输入：.meta-knowledge-base-guide.md + 目标工程源码
  - 任务：读源码（高层级）→ 决定 architecture/ 下应有哪些文档
  - 输出：每个文档的 id / title / path / description / keywords / covers_code
  - 为每个文档创建骨架文件（frontmatter + 一级标题 + blockquote 描述 + 空章节标题）

Sub-agent B2（Modules）：
  - 输入：同上
  - 任务：深度学习源码 → 识别模块边界 → 按依赖拓扑排序 → **产出产品视角边界字段**（v2.0 新增）
  - 输出：id / title / path / description / keywords / covers_code / depends_on + **v2.0 新增产品视角字段** responsibility / capability_boundary / communicates_with / system_privilege / lifecycle
  - 为每个模块创建骨架文件（frontmatter + 一级标题 + blockquote + 空章节标题）
  - **v2.0 关键变化**：原 v1.1 由 Goal 3 Stage 8 独立分派 sub-agent 从 modules/ 上浮生成的产品视角字段，现在直接由 Stage B2 sub-agent 在识别模块边界的同时产出。理由：Stage B2 本来就在读源码识别模块，顺便产出产品边界的边际成本很低，避免了 Goal 3 再跑一次上浮 sub-agent 的重复工作。

Sub-agent B3（Flows）：
  - 输入：同上 + B2 产出的模块文档清单
  - 任务：识别跨模块的端到端关键业务流程
  - 输出：同上 + involves_modules 字段
  - 为每个流程创建骨架文件

Sub-agent B4（Troubleshooting）：
  - 输入：同上 + B1/B2/B3 产出
  - 任务：基于模块和流程知识，识别可能的故障域
  - 输出：symptom-index.yaml 骨架 + FAQ 骨架 + known-bugs 骨架
```

**主 Agent 汇总 Stage B**：
- 收集 4 个 sub-agent 的输出
- 检查跨维度冲突（两个维度规划了重叠的文档）
- 生成 `index.yaml`（含所有条目的完整 frontmatter 字段 + 种子包版本元信息）
- 写入 `.meta-maintain-beacon.yaml`（commit = HEAD）
- 写入空 `.meta-explore-issues.jsonl`

**index.yaml 顶部必须包含以下版本契约字段**（种子包演进时用于对照兼容性）：

```yaml
# index.yaml 顶部
meta:
  seed_pack_version: "1.0.0"       # 生成本 KB 的 AFMT-KB-SEED 种子包版本
  seed_pack_build: "YYYY-MM-DD"    # 种子包发布日期
  kb_schema_version: "2.0"         # KB 结构 schema 版本（v2.0 起 modules[] 含产品视角字段）
  built_at: "YYYY-MM-DDTHH:MM:SS+TZ"  # 本次构建时间
  built_project_scale: "medium"    # 构建时判定的工程规模
  project_name: "{{PROJECT_NAME}}"  # 例："Redis"
entries:
  # ... 各文档条目（v5 已有的内容）
```

**版本契约的用途**：
- **升级 KB 时**：新种子包（v1.1.0）发布后，用户运行 `/maintain-kb` 或后续增量工具时，工具读取 `seed_pack_version` 判断当前 KB 与新种子包的差距——是仅需追加新章节（minor 版本差距）、还是需要重新生成部分文档（major 版本差距）、还是可以完全跳过（版本一致）
- **兼容性对照**：种子包每次发布需附带 `CHANGELOG.md`，标注各版本间的破坏性变更（如某字段重命名），升级工具据此决定是否需要迁移
- **审计**：多个工程使用同一种子包时，可通过 `seed_pack_version` 快速统计升级覆盖率

### 3.5 Stage C: 实例化 Goal 2 任务文档

**目标**：将 Stage A+B 的所有产出转化为 Goal 2 的**完全确定的执行指令**。

**这是 Goal 1 的核心价值产出**。主 Agent（或分派一个 sub-agent）基于 Goal 2 任务文档模板，将以下参数填充进去：

需要确定的参数：

| 参数类别 | 参数 | 来源 |
|---------|------|------|
| 文档清单 | `architecture_docs`: [{id, path, covers_code}] | Stage B - B1 |
| 文档清单 | `module_docs`: [{id, path, covers_code, depends_on}] | Stage B - B2 |
| 文档清单 | `flow_docs`: [{id, path, covers_code, involves_modules}] | Stage B - B3 |
| 分批策略 | Stage 3 的分批方案：每批包含哪些文档 id | 主 Agent 根据 depends_on 拓扑 + Stage A 规模判定计算 |
| 并发数 | 每批最多 N 个 sub-agent | 小型：不限 / 中型：≤8 / 大型：≤6 |
| Review 范围 | Stage 4 每轮需要 Review 的文档清单（= 全部 A/F/M 文档） | Stage B |
| Review 轮次上限 | 小型：最多 2 轮 / 中型：最多 5 轮 / 大型：最多 5 轮 | Stage A 规模判定 |
| 收敛标准 | critical: 0, medium: 小型 <3 / 中型 <5 / 大型 <5 | Stage A 规模判定 |
| QA jsonl 路径 | `pass/{doc_id}.qa.jsonl` | 硬编码 |
| 出工条件 | 最多 50 轮，Review 最多 5 轮 | 硬编码 |

**分批策略：依赖环路处理**

主 Agent 用 `depends_on` 做拓扑排序时，可能遇到循环依赖（module A 依赖 B，B 又依赖 A——现实工程中常见于紧耦合的双向服务、循环回调、header 互相 include）。种子包必须处理这种情况，不能让 Stage C 卡住：

1. **检测**：主 Agent 用标准 DFS 拓扑排序，遇到 back edge 即认定存在环
2. **降级顺序**（依次尝试）：
   - **顺序 1：入度最少打破环**——找出环中入度（被依赖数）最少的节点，将其 `depends_on` 清空，重新拓扑排序。理由：入度少的节点作为"起点"，对下游影响最小
   - **顺序 2：按目录分批**——若步骤 1 仍无法解环（多个环嵌套或深度耦合），放弃拓扑分批，改用目录路径作为分批依据（`src/kernel/*` 一批、`src/service/*` 一批）
   - **顺序 3：单批全量**——若目录结构也无清晰边界（如大型单体工程），降级为单批全量分派（仅适用于小型工程；中型/大型工程走到此步时需人类介入）
3. **告警**：任何环的检测和处理决策都必须记入 Journal（`action: decision, decision: cycle_broken, cycle: [A→B→A], resolution: minimum_indegree, node_cleared: B`），供后续 Review 阶段特别关注被清空 `depends_on` 的节点是否在其文档中体现了循环依赖的真实关系

**成本估算**

Goal 1 完成时必须在 Goal 2 任务文档头部生成"预计成本"章节，让用户在启动 Goal 2 前有决策依据。

**估算公式**（基于文档数和 Review 轮次）：

```
N_a = architecture_docs 数量
N_m = module_docs 数量
N_f = flow_docs 数量
N_total = N_a + N_m + N_f
R = review_max_rounds（2 / 5 / 5 for small/medium/large）

Stage 3 (文档生成) sub-agent 数：N_total
Stage 4 (Review) sub-agent 数：N_total × R（最坏情况，若提前收敛会少）
Stage 5 (聚合) sub-agent 数：1 或 0（可主 Agent 完成）
Goal 1 sub-agent 数：约 8-10（Stage A 3 个 + Stage B 4 个 + 元规范生成 1-2 个）
Goal 3 sub-agent 数：1-2（KB README + 可选 postmortem）

sub-agent 总数：N_total × (1 + R) + ~12
```

**token 估算参考**（每个 sub-agent 平均值，经验值，需按实际工程校准，未来需持续更新）：

| Sub-agent 类型 | 输入 tokens | 输出 tokens | 备注 |
|---------------|-------------|-------------|------|
| Stage A 观察 | 30-50K | 3-8K | 读顶层文件 + 输出适配参数 |
| Stage B 规划 | 50-80K | 5-15K | 读源码目录 + 输出文档清单和骨架 |
| Stage 3 文档生成 | 60-120K | 8-20K | 读 covers_code + 生成完整文档 |
| Stage 4 Review | 80-150K | 3-10K | 读文档 + 读 covers_code + 输出发现 |

**耗时估算**：

| 规模 | Goal 1 | Goal 2 | Goal 3 | 总耗时（并发充分） |
|------|--------|--------|--------|---------------------|
| 小型（N_total ≈ 5，R=2） | 15-25min | 20-40min | 5-10min | ~45min - 1.5h |
| 中型（N_total ≈ 15，R=5） | 25-40min | 1.5-3h | 5-10min | ~2-4h |
| 大型（N_total ≈ 30+，R=5） | 40-60min | 3-6h | 10-15min | ~4-8h |

**Goal 2 任务文档头部生成模板**：

```markdown
## 预计成本

- **工程规模**：medium（12 modules, 3 languages, 4 layers）
- **文档总数**：N_total = 18 (A=3, M=12, F=3)
- **Sub-agent 总数（估算）**：N_total × (1 + R) + 12 = 18 × 6 + 12 = **120 个**（最坏情况；若 Review 提前收敛可减少 30-50%）
- **Token 估算**：输入 ~9M-15M，输出 ~800K-1.5M（含所有 sub-agent）
- **预计耗时**：2-4 小时（并发充分时）
- **前置资源要求**：API 配额 ≥ 15M tokens；建议在 2h 无中断的时间窗口启动

**中止条件**（用户可随时中止）：
- 达到 50 轮出工上限
- Sub-agent 累计失败 > 20% 且重试无进展
- 用户手动 Stop
```

用户在读到这段后可以判断：是否值得启动、何时启动、是否需要先申请更高 API 配额。这也是作为成本基线的参考点。

**输出**：`AFMT-KB-SEED-bake-{project}.md`——一份参数完全的、可确定性执行的任务文档。

**Goal 1 完成条件**：
- 4 份元规范文件存在
- 所有文档骨架文件存在（frontmatter + 描述 + 空章节）
- `index.yaml` 完整（所有条目 + `meta.seed_pack_version` 等版本契约字段 + `meta.excluded_paths`）
- Goal 2 任务文档已生成（参数全部填充 + 头部含"预计成本"章节 + `excluded_paths` 段落）
- Agent 输出：`GOAL_MET: kb-primer 完成。模块 N 个，流程 M 个，架构 K 个。规模：{scale}。已排除路径 E 项。预计 Goal 2 sub-agent 数 X 个 / 耗时 Y 小时。Goal 2 任务文档已就绪。`

---

### 3.6 第三方源码排除机制（v1.1.2 新增，横切主题）

**问题背景**：
现实工程常见把第三方源码 commit 到仓库内（Go vendor、C/C++ ext/、node_modules 等），或存在 git submodule 引用的子仓。这些代码**不是本工程的职责**，不应进入 KB：
- 生成 module_header 会让 Commander 误以为它是本工程责任模块；
- 生成 modules/ 文档会浪费 token 记录第三方实现细节，且这些内容会随着依赖升级而失效；
- Review 阶段会因为无法理解第三方设计意图而产生大量误报；
- maintain-kb 增量检查会因第三方代码更新而频繁误触发。

**解决方案：三段式排除机制**

```
Stage A1 识别         → 主 Agent 汇总 + 人类 review → 持久化到 index.yaml.meta.excluded_paths
        │                        │                                      │
        │                        │                                      ├─► Stage B 消费（不把被排除路径作为模块）
        │                        │                                      ├─► Stage 3 消费（不读入被排除路径的源码）
        │                        │                                      ├─► Goal 3 Stage 8.2 消费（不为被排除模块上浮 module_header）
        │                        │                                      └─► /maintain-kb 消费（不因被排除路径的 diff 触发检查）
        │                        │
        │                        └── 软提醒（不阻断），列出高置信度 + 需 review 项
        │
        └── 基于 .gitignore / .gitmodules / 包管理器配置 / 常见目录名启发式识别
```

**识别策略**（Stage A1 sub-agent 的判定原则，详见 `prompts/stage-a-subagent.md`）：

| 类别 | 例子 | 置信度 |
|------|------|--------|
| **必须排除** | 包管理器依赖（`node_modules/` `vendor/` `.venv/`）/ 构建产物（`build/` `dist/` `target/`）/ 集中式第三方源码（`ext/` `third_party/` `deps/`）/ submodule | high |
| **可能排除** | 大规模测试 fixture / 示例代码 / 归档目录 / 生成的 stub 代码 | medium |
| **不能排除** | 主源码目录 / 测试代码 / 内部工具 / 集成层代码（哪怕调用第三方） | — |

**核心判定原则**：
- 若某目录属于本工程责任范畴，**不排除**——KB 应记录"本工程如何使用第三方"，但不记录"第三方本身如何实现"；
- 遇到不确定的目录，**倾向于不排除**并在 `excluded_paths[i].needs_human_review: true` 标注，由人类 review 决定；
- 排除决策必须写入 Journal（`{action: decision, decision: excluded_paths_finalized, ...}`）。

**人类介入点**：

Stage A.2.1 是**软提醒式**的介入点：主 Agent 展示识别到的清单，人类可增删修改，但**不主动阻断**执行。这个设计的权衡：
- **软提醒 vs 硬阻断**：软提醒让流程保持顺畅，人类若没关注也不会失去继续执行的能力；代价是人类可能错过某个错误分类，但由于识别策略保守（默认倾向"不排除"），错误方向多为"少排除"而非"多排除"。
- **软提醒 vs 完全静默**：完全静默会让 vendor/ 悄悄进入 KB 而被人类察觉不到，直到看到 module_header 中出现莫名其妙的模块。软提醒把这个决策显式化。

**持久化到 index.yaml.meta**：

`index.yaml.meta.excluded_paths` 是**运行时权威源**。所有下游消费者（Stage 3 / Stage 4 / Goal 3 Stage 8 / maintain-kb）都从这里读取。为什么放在 meta 而不是单独文件：
- `index.yaml` 本身是 KB 的入口，其他 sub-agent 都会读，从 meta 段读取零额外成本；
- 后续 maintain-kb / commander 增量维护时可持续对照；
- 版本契约（`seed_pack_version` 等）也在 meta，`excluded_paths` 与之同源便于追溯。

**边界与已知限制**：
- 目前 Stage A1 只**识别一次**，若工程后续新增 vendor 目录，需要人工在 index.yaml.meta 中追加，或调用 `/maintain-kb --refresh-excluded-paths`（未来能力，v1.1.2 尚未实现）；
- 若某模块的 `covers_code` **既包含**本工程代码**又包含**第三方代码（罕见情况），Stage B 应按文件级别过滤，不整模块排除。目前实现是路径级别精确匹配，若源码结构是 `src/module/{self,third-party}/*` 混合，可能出现漏过滤，需人类事后检查。

---

## 4. Goal 2: kb-docs-bake（确定性执行）

### 4.1 定位

Goal 2 拿到 Goal 1 产出的实例化任务文档后，它面对的是一个**所有参数已确定的执行计划**。每一轮的行为是确定的——主 Agent 不需要根据 sub-agent 的返回内容做动态决策，只需要检查完成状态然后推进到下一批/下一 Stage。

### 4.2 Goal 2 的 Stage 设计

```
Stage 3: 文档内容生成（Round 0）
  └── 按 Goal 1 确定的分批方案，逐批分派 sub-agent
  └── 每个 sub-agent 生成一份文档 + 输出自己的 QA jsonl

Stage 4: 多轮 Review（Round 1, 2, ... N）
  └── 每轮对全部文档分派 Review sub-agent
  └── 每轮后统计问题数量，判断收敛
  └── 每轮独立 sub-agent（上下文隔离）

Stage 5: QA 聚合
  └── 扫描所有 per-document QA jsonl → kb-build-qa.md
```

### 4.3 Stage 3: 文档内容生成

**确定性特征**：文档数量、每个文档的 covers_code、依赖关系——全部在 Goal 1 的产出中确定。主 Agent 不需要做任何分析或判断，只需要按分批方案逐批分派。

```
每批的行为：
  for doc in batch:
    spawn sub-agent(
      prompt: "生成文档 {doc.id}，
               封面代码：{doc.covers_code}，
               已有骨架：{doc.path}（含章节标题），
               依赖文档（已完成，可引用）：{dep_docs}，
               以源代码为唯一事实来源。
               疑问输出到 pass/{doc.id}.qa.jsonl。"
    )
  wait_all()
  验证：每个文档的必填章节是否有内容
  更新 index.yaml（补充 keywords、描述）
```

**Sub-agent 的 prompt 构成**：
- 文档生成指令（通用，来自种子包）
- `doc.id` / `doc.path` / `doc.covers_code` / `dep_docs`（实例化参数，来自 Goal 1）
- QA jsonl 写入规范（通用，来自种子包）

### 4.4 Stage 4: 多轮 Review

**确定性特征**：Review 的文档范围、每轮的 Review sub-agent 数量——全部确定。主 Agent 只需要：分派 → 收集统计 → 判断收敛 → 决定是否再来一轮。

```
每轮的行为：
  for doc in all_docs:  // 全部 A/F/M 文档
    spawn sub-agent(
      prompt: "审查 {doc.id}：
               文档路径：{doc.path}，
               封面代码：{doc.covers_code}，
               以源代码为唯一事实来源。
               逐条验证文档声明。
               修正发现的错误。
               不一致项写入 pass/{doc.id}.qa.jsonl（追加）。"
    )
  wait_all()
  统计：critical N / medium M / minor K
  判断：critical == 0 && medium < 收敛阈值（见 Goal 1 实例化参数中的 convergence_criteria）→ 硬收敛，进入 Stage 5
        否则 → 下一轮（最多 review_max_rounds 轮）
  若达到 review_max_rounds 仍未硬收敛 → **软收敛兜底**：
    - 记录未解决项到 kb-build-qa.md 的 "quality warning" 章节
    - 每一项标注：文档 id、问题严重度、Review 轮次历史、每轮变化趋势（趋近于零 / 反复出现 / 越来越多）
    - Journal 追加 decision 条目：`{action: decision, decision: soft_convergence, unresolved: {critical: X, medium: Y}}`
    - 主 Agent 输出 `GOAL_MET_WITH_WARNING: kb-docs-bake 达到轮次上限，X 项 critical / Y 项 medium 未收敛，已列入 quality warning 供人类决策`
    - 允许人类介入：追加一轮定向 Review、放宽阈值接受现状、或标记特定文档为 draft 状态延后处理
  更新 index.yaml（如需）
```

**关键约束**：
- 每轮的 Review sub-agent 是全新的（不继承上一轮上下文）
- 同一个文档的不同轮次 Review sub-agent 追加到同一个 `{doc.id}.qa.jsonl`
- 主 Agent 不读 Review sub-agent 的详细发现（上下文污染），只读统计数字
- new QA 条目追加，已有 QA 条目可以被新轮次标记 resolved

#### 4.4.1 严重度分级锚定示例

**问题**：如果不明确定义 critical / medium / minor 的边界，不同 sub-agent 会主观漂移——同一个问题可能被 A 判为 critical，被 B 判为 minor，导致收敛阈值失去意义。

**解决**：种子包的 `prompts/stage-4-subagent.md` 必须内嵌以下锚定示例。Review sub-agent 按示例套用，不允许自由裁量：

**Critical**（会导致读者形成错误认知，或让基于文档的决策直接失败）：
- 声明与代码矛盾：文档说"使用 RocksDB 存储"，代码实际用 SQLite
- 关键接口签名错误：文档标注函数 `Foo(int, string)`，代码实际是 `Foo(string, int)`
- 数据流方向错误：文档说 A → B，代码实际 B → A
- 关键前提缺失：文档描述某功能但未提及"仅 Debug 构建生效"这类必要前提
- 事实性错误：涉及安全、并发、事务边界的错误描述

**Medium**（信息不完整或误导性，但不会引发直接错误决策）：
- 章节缺失但非核心：模块文档缺"已知陷阱"章节
- 描述过时：文档基于旧版本代码，当前代码已重构但主流程仍能对应
- 关键路径的次要分支未描述：主流程正确，某个 error path 被遗漏
- 术语不一致：同一概念在不同章节用不同名字
- 缺少必要的交叉引用：涉及跨模块流程但未指向 flows/ 文档

**Minor**（不影响正确性和可读性的次要瑕疵）：
- 拼写、标点、排版
- 章节顺序建议、格式建议
- 可选章节的缺失（如"未来优化"）
- Frontmatter 中 keywords 覆盖不全（不影响 index.yaml 检索命中）

**判定原则**（当 sub-agent 举棋不定时）：
- "读者按文档做决策会失败吗？" → yes = critical
- "读者会形成不完整或轻微误导的认知吗？" → yes = medium
- "只影响阅读体验，不影响理解吗？" → yes = minor
- 就高不就低（拿不准时选更严重的级别）

### 4.5 Stage 5: QA 聚合

**确定性特征**：QA jsonl 文件列表确定（= 文档清单），聚合逻辑固定。

```
行为：
  扫描所有 pass/{doc.id}.qa.jsonl
  合并 → 去重 → 按严重度分组 → 按维度分类
  标注 resolved vs open
  输出 kb-build-qa.md
```

**Goal 2 完成条件**：
- 所有文档完整（N 份）
- Review 收敛（0 critical + medium < 收敛阈值）
- `kb-build-qa.md` 已生成
- Agent 输出：`GOAL_MET: kb-docs-bake 完成。文档 N 份，QA 问题 M 项（含已 resolved K 项）。请人类回答 kb-build-qa.md 中的问题。`

---

## 5. Goal 3: kb-infra-setup（基础设施 + Skill + Commander 内容构建）

### 5.1 定位

Goal 3 在 `kb-docs-bake` 完成之后执行。此时 KB 文档内容已经生成、Review 收敛、QA 已聚合。但 KB 的**运行基础设施**尚未就绪——Skill 没有安装、目录不完整、没有 README 作为入口文档、维护信标和 explore-issues 等基础设施文件也缺失、Commander 决策中枢的 module_header 内容也未构建。

Goal 3 的任务大部分是**文件系统操作**（复制、创建、写入），主 Agent 可自主完成。仅少数需要判断或语义提炼的任务需要分派 sub-agent（KB README 编写、Commander 内容构建的 module_header 上浮、Commander README 编写、postmortem 汇总）。

### 5.2 Goal 3 的 Stage 设计

```
Stage 6: 目录与基础设施文件创建
  └── 创建 pass/、works/、develop/、troubleshooting/incidents/、commander/history/decisions/
  └── 验证 .meta-maintain-beacon.yaml、.meta-explore-issues.jsonl（Goal 1 已产出）
  └── 生成 KB README（1 个 sub-agent）

Stage 7: Skill 迁移
  └── 复制 explore-kb / maintain-kb skill（去耦合版）
  └── 复制 knowledge-extract-qa-guide.md
  └── 边复制边模板化，将 commander skill 安装为 {{PROJECT_SLUG}}-commander

Stage 8: Commander 内容构建（本 Goal 的核心新增能力）
  └── 复制 commander 元文档（module_header_meta.md / TEMPLATE.md / decisions/index.yaml）
  └── **module_header 上浮**：为 modules/ 中的每个模块，分派 sub-agent 从技术视角文档上浮生成产品视角的 module_header_<Name>.md
  └── 生成 Commander README（1 个 sub-agent，含目录速览表 + 模块通讯关系图 + 核心流程时序图 + 盲区清单）

Stage 9: 完整性检查 + 收尾
  └── 对照清单逐项验证所有必选文件与目录
  └── 检查 module_header 覆盖率（阈值：≤ 10% 缺失允许）
  └── 生成 postmortem + summary
```

### 5.3 Stage 6: 目录与基础设施文件创建

**目标**：补齐 KB 运行所需的基础设施。

**不需要 sub-agent**：这些是纯文件操作，主 Agent 可以直接执行。

操作清单：

| 操作 | 文件/目录 | 说明 |
|------|----------|------|
| 创建目录 | `pass/` | 如不存在。用于存放 Q&A jsonl、maintain-kb 检查报告、外部输入材料 |
| 创建目录 | `works/` | 如不存在。用于存放 Work 文档。需包含 `.meta-work-guide.md`（已在 Goal 1 中生成） |
| 创建目录 | `develop/` | 如不存在。用于存放研发规范文档（初始为空，后续按需创建） |
| 创建目录 | `troubleshooting/incidents/` | 如不存在 |
| 验证配置文件 | `.meta-maintain-beacon.yaml` | 验证存在（通常已在 Goal 1 Stage B 中创建）。若丢失则重新创建，commit = HEAD |
| 验证配置文件 | `.meta-explore-issues.jsonl` | 验证存在（通常已在 Goal 1 Stage B 中创建）。若丢失则重建空文件 |
| 创建入口文档 | `README.md` | **需分派 sub-agent** 编写 |

**KB README 的编写**（需 sub-agent）：

```
Sub-agent 任务：
  "为目标工程的 .think-tank/knowledge-base/ 目录编写 README.md。
   
   输入：
   - index.yaml（了解 KB 全貌）
   - .meta-knowledge-base-guide.md（了解构建规范）
   - CLAUDE.md（了解工程背景）
   
   输出 README.md，需包含：
   1. 本知识库的定位与使用场景（1-2 段）
   2. 目录结构速览（表格）
   3. 快速检索指南（如何通过 index.yaml 找文档）
   4. 文档类型说明（architecture / modules / flows / troubleshooting 各是什么）
   5. 维护方式（maintain-kb skill 的简介与调用方式）
   6. 贡献指南（如何新增/修改知识文档）
   
   约束：
   - 面向 AI Agent 和人类开发者
   - 控制在 100 行以内
   - 使用中文
   - 不重复 .meta-knowledge-base-guide.md 的详细规范，仅做导航级概述"
```

### 5.4 Stage 7: Skill 迁移

**目标**：将可复用的 meta-skill 和关键元文档从种子包复制到目标工程。

**大部分不需要 sub-agent**：explore-kb / maintain-kb / knowledge-extract-qa-guide 是纯文件复制。Commander skill 是模板化的，需要边复制边替换占位符（主 Agent 自主完成，无需 sub-agent）。

迁移清单：

| 源（种子包中） | 目标 | 操作 | 说明 |
|---------------|------|------|------|
| `skills/explore-kb/` | `.claude/skills/explore-kb/` | 复制目录 | 已去耦合，零产品名引用 |
| `skills/maintain-kb/` | `.claude/skills/maintain-kb/` | 复制目录 | 已去耦合，通过 index.yaml 自发现环境 |
| `skills/commander/` | `.claude/skills/{{PROJECT_SLUG}}-commander/` | 边复制边模板化 | 6 份文件（SKILL.md + 5 references），替换 `{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` 等占位符；`{ADAPT}` 标记保留 |
| `knowledge-extract-qa-guide.md` | `.think-tank/knowledge-base/knowledge-extract-qa-guide.md` | 复制文件 | 通用文档，直接可用 |

**关于 Commander skill**：
- Commander skill 本身是**协议**，负责回答策略、路由规则、上下文预算、知识回写等运行时行为。协议部分是通用的，通过模板化后即可迁移到任意工程；
- Commander 依赖 `.think-tank/commander/` 目录下的 module_header 体系作为**内容**——这部分内容由 Stage 8 从 `knowledge-base/modules/` 上浮生成；
- Skill 命名规则：`{{PROJECT_SLUG}}-commander`。`{{PROJECT_SLUG}}` 从 `project_name` 派生（小写化 + 特殊字符替换为连字符），即 `{{PROJECT_NAME}} → {{PROJECT_SLUG}}`（例：`Redis` → `redis`，对应 skill 名 `redis-commander`）；
- 迁移完成后，用户可通过 `/{{PROJECT_SLUG}}-commander` 或自然语言触发（skill description 中含产品名与主要模块名）激活。

**关于 knowledge-extract-qa-guide.md**：
- 这是 KB 生态中的关键元文档
- 定义了 AI Agent 如何通过结构化 QA 工作流从人类团队提取隐性知识（非源码知识）并回流到知识库
- 该文档应放在 `.think-tank/knowledge-base/` 根目录下，与 `.meta-knowledge-base-guide.md` 同级

**重要：两种 QA 机制的区分**

本方案中存在两种 QA 机制，服务于不同的阶段和目的，请勿混淆：

| | Build-time QA（Goal 2） | Post-build QA（knowledge-extract-qa-guide） |
|------|--------------------------|---------------------------------------------|
| **阶段** | KB 构建过程中 | KB 构建完成后 |
| **问题来源** | Sub-agent 阅读源码时发现的代码歧义、逻辑矛盾、无法从代码推断的设计决策 | AI 阅读完整 KB 后识别的知识盲区（测试策略、性能基准、部署拓扑、事故经验等——这些是无法从代码中获取的隐性知识） |
| **产出格式** | `pass/{doc_id}.qa.jsonl`（每行一个发现，JSON 格式） | `architecture/qa.md`、`modules/qa.md` 等（结构化问答，Markdown 格式） |
| **聚合后** | `pass/kb-build-qa.md`（Goal 2 Stage 5 产出） | 各层 `qa.md` 独立维护 |
| **回答者** | 人类开发者（代码相关的歧义需要人确认） | 人类团队（隐性知识只有人知道） |
| **回流方式** | 人类回答 kb-build-qa.md → Agent 更新对应知识文档 | 遵循 knowledge-extract-qa-guide.md §5 回流流程 |

`knowledge-extract-qa-guide.md` 同时替代了 `qa-template.md` 的作用——它内嵌了 QA 问题条目的格式规范（§4），因此不再需要独立的 `qa-template.md`。

### 5.4.1 ~~Stage 8: Commander 内容构建~~（v1.1 引入，v2.0 已移除）

> **v2.0 状态**：本 Stage **已完全移除**。原设计将产品视角边界字段（`responsibility` / `capability_boundary` / `communicates_with`）产出为独立的 `commander/module_header_<Name>.md` 文件，v2.0 起改为**直接由 Goal 1 Stage B 产出并写入 `index.yaml.modules[]`**（详见 §3.2 字段规范）。此变更消除了 KB 与 commander 之间的双份维护，让 KB 成为单一事实来源。
>
> 本节保留作为 v1.x 历史记录。v2.0 起：
> - `.think-tank/commander/` 目录不再存在；
> - `stage-commander-header-gen.md` sub-agent prompt 已废除；
> - Goal 3 Stage 8 直接改为完整性检查 + 收尾（原 Stage 9 归位为 Stage 8）；
> - 产品视角字段的产出、可信度自检、更新时机均由 `.meta-knowledge-base-guide.md §3.2/3.3/3.4` 规范。

#### v1.x 原始设计（供历史查阅）

**目标**：为目标工程构建 `.think-tank/commander/` 目录下完整的产品视角决策中枢——含 module_header_meta.md 规范文档、历史决策模板、以及为 `knowledge-base/modules/` 中每个模块生成一份 module_header_<Name>.md。

**核心动作：module_header 上浮**——从技术视角的详尽文档（`knowledge-base/modules/<id>.md`）中，抽出产品视角的宏观信息（`commander/module_header_<Name>.md`）。

**为什么用"上浮"而不是"手写"或"照搬"**：
- **手写**：目标工程可能有 20+ 个模块，逐个手写 module_header 是重复劳动；
- **照搬**：module_header 天然与产品名/技术栈耦合，无法直接跨工程复制；
- **上浮**：Goal 2 已经生成了详尽的 modules 文档，其中蕴含了所有 module_header 所需的信息，只是**表达层次**不同。

**上浮转换规则**（v2.0 起转移到 §3.2 的字段填写要点，同一套规则由 Goal 1 Stage B 直接应用）：

| 源文档中的表述 | 上浮后的表述 |
|--------------|-----------|
| "使用 RocksDB 存储路径映射" | "维护路径与文件 ID 的双向映射" |
| "通过内部协议接收上层指令" | "通过控制通道接收上层管理指令" |
| "底层事件上报" | "接收底层事件并驱动处理" |
| "spdlog 记录到 XX 日志" | （不入产品视角字段——这是实现细节） |

**判定标准**：若一句表述包含技术专有名词（框架名 / 协议名 / API 签名 / 数据结构名 / 内部术语），必须转换为产品语言。

**Commander module_header 的覆盖率阈值**：
- v1.x：期望 100% / 允许 ≤ 10% 缺失（large）/ > 10% 缺失阻塞
- v2.0：产品视角字段由 Stage B 直接产出，Stage 8 完整性检查中作为软检查（≥ 50% 即通过，不阻塞 Goal 完成，但记入 postmortem 供后续 /maintain-kb 上浮补齐）

### 5.5 Stage 8: 元文档移植完整性检查 + 收尾（v2.0 重编号，原 Stage 9）

**目标**：确保所有元文档已从种子包移植到目标工程，无遗漏。

**不需要 sub-agent**：主 Agent 可以对照清单逐项检查。

检查清单：

```
必选项（缺一不可）：
  [ ] .think-tank/knowledge-base/.meta-knowledge-base-guide.md
  [ ] .think-tank/knowledge-base/.meta-maintain-beacon.yaml（验证存在，Goal 1 已创建）
  [ ] .think-tank/knowledge-base/.meta-explore-issues.jsonl（验证存在，Goal 1 已创建）
  [ ] .think-tank/knowledge-base/index.yaml
  [ ] .think-tank/knowledge-base/README.md
  [ ] .think-tank/knowledge-base/knowledge-extract-qa-guide.md
  [ ] .think-tank/knowledge-base/works/.meta-work-guide.md
  [ ] .think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md
  [ ] .think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md
  [ ] .think-tank/knowledge-base/troubleshooting/.meta-ts-output-template.md（如有）
  [ ] .think-tank/commander/module_header_meta.md
  [ ] .think-tank/commander/README.md
  [ ] .think-tank/commander/history/decisions/TEMPLATE.md
  [ ] .think-tank/commander/history/decisions/index.yaml
  [ ] .claude/skills/explore-kb/SKILL.md
  [ ] .claude/skills/maintain-kb/SKILL.md
  [ ] .claude/skills/{{PROJECT_SLUG}}-commander/SKILL.md
  [ ] .claude/skills/{{PROJECT_SLUG}}-commander/references/{workflow,answer-strategy,routing-rules,context-budget,knowledge-writeback}.md

module_header 覆盖率：
  [ ] modules/ 中每个模块对应 module_header_<Name>.md 存在（阈值 ≥ 90%）

目录项：
  [ ] pass/ 存在
  [ ] works/ 存在
  [ ] develop/ 存在
  [ ] troubleshooting/incidents/ 存在
  [ ] commander/history/decisions/ 存在

收尾：
  [ ] pass/kb-seed-postmortem.md 已生成（聚合三个 Goal 的异常信号和统计）
  [ ] pass/kb-seed-summary.md 已生成（人类可读的构建完成证书）
```

**postmortem 生成**（由主 Agent 或分派一个 sub-agent）：读取三个 Journal 文件，聚合异常信号。包含：各 Goal 耗时、异常统计、重试率、最常见错误类型、关键决策记录（含 module_header 上浮失败项）、流程优化建议。

**summary 生成**（由主 Agent）：一页纸的构建完成摘要。包含：三个 Goal 完成状态、总耗时、文档数量（A/F/M/T）、QA 问题统计、Commander module_header 覆盖率、关键决策、下一步行动。

**Goal 3 完成条件**：
- 所有必选项文件存在
- 所有目录项存在（含 develop/、commander/history/decisions/）
- KB README 已生成
- explore-kb / maintain-kb / {{PROJECT_SLUG}}-commander 三个 skill 已安装
- knowledge-extract-qa-guide.md 已移植
- Commander 元文档（module_header_meta.md / README.md / decisions TEMPLATE + index）齐备
- module_header 覆盖率 ≥ 90%
- `pass/kb-seed-postmortem.md` 已生成
- `pass/kb-seed-summary.md` 已生成
- Agent 输出：`GOAL_MET: kb-infra-setup 完成。基础设施就绪，Skill 已安装（含 {{PROJECT_SLUG}}-commander），module_header 覆盖 N/M。Postmortem 和 Summary 已生成。`

---

## 5.6 Goal 4: kb-qa-triage（全局视角 QA 分诊，v1.2.0 新增）

### 5.6.1 定位

Goal 4 是**可选**的，在 Goal 3 完成后执行。

**核心动机**：Goal 2 Stage 4 Review 阶段产出的 kb-build-qa.md 是**局部视角**的产物——每个 Review sub-agent 只看到自己负责的一份文档 + 对应 covers_code，无法从全局 KB 视角判断"这个问题是否其实已经能被 KB 回答"。等到 Goal 3 结束、KB 完整互联（含 module_header / flows / troubleshooting / knowledge-base-guide），实际上很多局部视角提出的问题此时已经能被 KB 回答；剩下的才是真正需要人类介入的隐性知识。

Goal 4 就是补上这个"全局视角回答尝试"环节。用 explore-kb 尝试回答每个 QA 问题，把结果分为 4 档：

- **fully_answered**：KB 已能明确回答，人类不必再答
- **partially_answered**：KB 部分覆盖，人类补充 kb_gaps 中的缺口即可
- **cannot_answer_needs_human**：KB 完全不覆盖，属于隐性知识（测试策略/性能基准/事故经验等），必须人类回答
- **should_not_be_a_question**：噪音问题（描述模糊/代码已改/重复问题），从人类队列剔除

**价值**：显著减少流入人类回答阶段的问题数量。同时对 fully_answered / partially_answered 问题，sub-agent 可能生成 `suggested_kb_patch`——把答案回填到 KB，让下次同类问题能被 explore-kb 直接命中。

### 5.6.2 Goal 4 的 Stage 设计

```
Stage 10: 备份 + 解析 QA 问题清单
  └── 备份 kb-build-qa.md 为 kb-build-qa-pre-goal4.md
  └── 主 Agent 直接解析 QA 条目清单

Stage 11: 分诊 sub-agent 批并行分派
  └── 每批 6 个 sub-agent，fresh context 保证隔离
  └── 每个 sub-agent 用 explore-kb 尝试回答 + 判定 4 档
  └── 对 low confidence 的 fully_answered 做交叉验证

Stage 12: 重组 kb-build-qa.md + 生成 kb-doc-patches.md
  └── 按 4 档重组 kb-build-qa.md
  └── 汇总所有 suggested_kb_patch 到 pass/kb-doc-patches.md（人类批准后 apply）
```

### 5.6.3 关键设计决策

**为什么用 explore-kb 而不是让 sub-agent 直接读 KB 文档**：

- 复用 explore-kb 已有的三级降级检索、路由规则、上下文预算
- KB 若在 Goal 4 期间被其他 Agent 修改，explore-kb 会读到最新版
- 未来 explore-kb 演进（向量检索、语义索引），Goal 4 自动受益

**批并行 vs 完全串行**：

- **串行**：完全上下文隔离但速度慢（30 问题 × 30-60s = 15-30 分钟）
- **批并行 + fresh context**：每个 sub-agent fresh 分派天然隔离；每批 6 个并发；批间同步。速度快 5-10 倍，且保留隔离性

**交叉验证的设计**：

- 只对 `fully_answered` + `confidence: low` 的判定做交叉验证
- 目的：防止"过度自信"——把本该问人类的问题误报为 fully_answered
- 成本可控：交叉验证的 entry 数占总数很小比例

**为什么不自动 apply KB patch**：

- 自动 apply 有风险：sub-agent 说"完全回答"实际错，把错误答案写进 KB 会污染
- 因此选择**生成 patch 建议 + 人类批准 + /maintain-kb apply**
- 这与 seed 既有的"Goal 只生成建议，maintain-kb 系统性落地"分工一致

**为什么标记为 optional**：

- 若 QA 问题总数很少（<10 项），Goal 4 的过滤价值不明显
- 若 explore-kb 因某些原因不稳定，可跳过 Goal 4 直接人工回答（人类总能兜底）
- Goal 4 是**优化**不是**必需**——KB 没有它也能进入人类回答阶段

### 5.6.4 Goal 4 完成条件

- `pass/kb-build-qa-pre-goal4.md` 备份存在
- `pass/kb-build-qa.md` 重组为 4 档结构
- `pass/kb-doc-patches.md` 生成（含所有 sub-agent 的 patch 建议，若无则为空文件带说明头）
- 所有 QA 条目均已分诊（含 abandoned——重试上限失败的项，保守归入 cannot_answer_needs_human）
- Progress Log 中 Stage 10 / 11 / 12 均为 ✅
- Agent 输出：`GOAL_MET: kb-qa-triage 完成。原始 N 项 → fully_answered A / partially_answered B / cannot_answer_needs_human C / should_not_be_a_question D。人类回答负担减少 (A+D)/N * 100%。`

### 5.6.5 边界与已知限制

- **可能过度自信**：sub-agent 有把 partially_answered 判为 fully_answered 的风险；对冲手段是交叉验证 + 人类审阅 fully_answered 分组的选项
- **依赖 explore-kb 质量**：若 explore-kb 检索能力不足（如关键词覆盖不全），Goal 4 会把本该 partially_answered 的问题判为 cannot_answer_needs_human，导致人类回答负担减少不明显。这是可以通过完善 KB 关键词/module_header 自然改善的
- **首次运行建议人工抽查判定分布**（尤其 fully_answered 分组与交叉验证结果）

---

## 6. Goal 之间的数据契约

Goal 1 产出 → Goal 2 消费 → Goal 3 消费：

示例：Redis 8.10.2

```yaml
# Goal 2 任务文档中的确定性参数（由 Goal 1 填充）

seed_pack_version: "1.0.0"        # 引用的种子包版本（必须与最终 index.yaml.meta.seed_pack_version 一致）
project_name: "目标工程名"
kb_root: ".think-tank/knowledge-base"

architecture_docs:
  - id: system-overview
    path: architecture/system-overview.md
    covers_code: ["README.md", "CLAUDE.md"]
    depends_on: []

module_docs:
  - id: replication
    path: modules/replication.md
    covers_code: ["src/replication.c", "src/sync.c"]
    depends_on: []
  - id: keyspace-and-expiration
    path: modules/keyspace-and-expiration.md
    covers_code: ["src/db.c", "src/expire.c"]
    depends_on: []
  - id: rdb-persistence
    path: modules/rdb-persistence.md
    covers_code: ["src/rdb.c", "src/rdb.h"]
    depends_on: ["keyspace-and-expiration"]
  - id: cluster
    path: modules/cluster.md
    covers_code: ["src/cluster.c", "src/cluster_legacy.c"]
    depends_on: ["replication"]
  - id: observability-and-introspection
    path: modules/observability-and-introspection.md
    covers_code: ["src/latency.c", "src/latency.h"]
    depends_on: ["replication"]
  # ... Goal 1 填充全部

flow_docs:
  - id: replication-sync-flow
    path: flows/replication-sync-flow.md
    covers_code: ["src/replication.c", "src/rdb.c"]
    involves_modules: ["replication", "rdb-persistence"]
  - id: command-execution-flow
    path: flows/command-execution-flow.md
    covers_code: ["src/server.c", "src/networking.c"]
    involves_modules: ["keyspace-and-expiration", "observability-and-introspection"]
  # ... Goal 1 填充全部

stage3_batches:
  - batch: 1
    docs: ["keyspace-and-expiration", "replication"]  # 无依赖模块
  - batch: 2
    docs: ["rdb-persistence", "cluster"]  # 依赖 batch 1
  - batch: 3
    docs: ["observability-and-introspection"]
  # ... Goal 1 计算全部

review_docs:  # Stage 4 需要 Review 的全部文档 id 列表
  - "system-overview"
  - "tech-stack"
  - "data-model-relations"
  - "replication"
  # ... 全部 A/F/M 文档

convergence_criteria:
  critical_max: 0
  medium_max: 4          # 小型工程为 2

project_scale: medium     # small | medium | large（来自 Goal 1 Stage A 的规模判定）
max_turns: 50             # 小型工程为 30
review_max_rounds: 5      # 小型工程为 2
```

---

## 7. 跨轮次状态管理

### 7.1 状态持久化

```
每个 Goal 有独立的任务文档，Progress Log 写在自己的任务文档中：

Goal 1 (AFMT-KB-SEED-primer.md):
  ## Progress Log
  | 时间 | Stage | 动作 | 结果 |
  |------|-------|------|------|
  | T0 | A | 分派 A1/A2/A3 | ✅ |
  | T1 | A | A1/A2/A3 返回，分派元规范生成 sub-agent | ⏳ |
  ...

Goal 2 (AFMT-KB-SEED-bake-{project}.md):
  ## Progress Log
  | 时间 | Stage | 动作 | 结果 |
  |------|-------|------|------|
  | T0 | 3 | 分派 Batch 1（3 个 sub-agent） | ✅ |
  | T1 | 3 | 分派 Batch 2（2 个 sub-agent） | ⏳ |
  ...
```

### 7.2 状态恢复

每轮开始时的恢复协议（三个 Goal 通用）：
1. 读自己的任务文档 Progress Log
2. 验证文件系统（Log 声称完成但文件不存在 → 回退状态）
3. 找到第一个 status ≠ ✅ 的 Stage/Batch → 从那里继续

---

## 8. 并发安全

### 8.1 QA jsonl 隔离策略

```
每个文档独立的 QA jsonl：
  pass/{doc.id}.qa.jsonl

写入者：
  Stage 3: 一个 sub-agent（生成该文档的 sub-agent）
  Stage 4: 多轮 Review sub-agent（每轮一个，追加写入同一文件）

冲突处理：
  同一文档同时只有一个 sub-agent 在写（Stage 3 生成 → Stage 4 Round 1 Review → Round 2 Review...是串行的）
  不同文档的 sub-agent 并行，各写各的 jsonl → 无冲突

聚合：
  Stage 5 由主 Agent 或单一 sub-agent 读取所有 jsonl 文件 → 合并
```

### 8.2 文档写入隔离

```
每个 sub-agent 只写一个文档 → 天然隔离
index.yaml 仅主 Agent 写入（每个 Stage 结束时统一更新）→ 无并发问题
```

---

## 9. 流程可观测性：Journal 日志体系

### 9.1 设计目标

AFMT-KB-SEED 的三个 Goal 可能跨越数小时甚至更久（取决于工程规模）。主 Agent 和 sub-agent 可能因各种原因中断（session 超时、API 限流、人为停止）。整个可观测性体系（Progress Log + Journal）需要同时服务三个目标：

| 目标 | 读者 | 承载者 | 核心需求 |
|------|------|--------|---------|
| **人类可读** | 人类开发者 | 任务文档 Progress Log | 一眼看出当前进度、哪个 Stage 在跑、已耗时多久 |
| **可重拭（Resumable）** | 新 Agent | Progress Log + Journal Entries + 文件系统 | 读取后能在**不依赖原对话上下文**的情况下重建状态，找到中断点，继续推进 |
| **可追溯（Traceable）** | 人类 + Agent | Journal Entries | 记录所有问题、决策、重试、异常，支持事后复盘和流程优化 |

**权威状态源**：任务文档 Progress Log。Journal 只做事件流水，不聚合状态（消除 v5 版本 State Summary 与 Progress Log 双写导致的漂移风险）。

### 9.2 文件布局

```
.think-tank/knowledge-base/pass/
├── kb-seed-journal-kb-primer.md        # Goal 1 的 Journal
├── kb-seed-journal-kb-docs-bake.md     # Goal 2 的 Journal
└── kb-seed-journal-kb-infra-setup.md   # Goal 3 的 Journal
```

每个 Goal 一个独立 Journal 文件。原因：
- 不同 Goal 可能在不同时间段执行（中间有人工介入窗口）
- 单个文件不会无限增长
- Goal 之间的输入输出关系由任务文档本身定义，Journal 不需要跨 Goal 引用

### 9.3 Journal 文件结构

每个 Journal 文件由两部分组成：**结构化条目列表（Entries）** 和 **恢复指引尾部（Resume Guide）**。

**为什么没有 State Summary**：早期版本设计了 Journal 头部的 State Summary 块（当前进度快照），但它与任务文档的 Progress Log 存在信息冗余，且两者需要同步维护，容易漂移。v6 起明确职责分离：

- **任务文档 Progress Log** = 唯一权威状态源（Haiku 用它判断 Goal 条件、新 Agent 用它恢复的第一站）
- **Journal Entries** = 追加式流水，记录每个 sub-agent 事件的详情（用于事后分析、恢复时的详细上下文）

新 Agent 恢复时的第一步永远是读任务文档 Progress Log 定位到"当前 Stage/Batch"，再读 Journal Entries 拿到该 Stage/Batch 内的 sub-agent 明细。

```
# Journal: kb-primer

<!-- 权威状态请查阅任务文档 AFMT-KB-SEED-primer.md 的 Progress Log 章节。
     本 Journal 仅追加事件流水，不维护聚合状态。 -->

## Entries

### [YYYY-MM-DDTHH:MM:SS] TURN-1 · Stage A · 分派工程理解 sub-agents
- **action**: dispatch
- **stage**: A
- **sub_agents**:
  - `kb-primer-A1` (架构维度): spawned, agent_id=`a1b2c3d4...`
  - `kb-primer-A2` (模块维度): spawned, agent_id=`e5f6g7h8...`
  - `kb-primer-A3` (流程/排查维度): spawned, agent_id=`i9j0k1l2...`
- **expected_output**: 三份维度观察报告 + 适配参数建议 + 规模标签
- **status**: pending

### [YYYY-MM-DDTHH:MM:SS] TURN-1 · Stage A · A1 完成
- **action**: result_collected
- **sub_agent**: kb-primer-A1 (agent_id=`a1b2c3d4...`)
- **result**: success
- **artifacts**: 适配参数已收集（architecture 维度）
- **scale_label**: medium（判定依据：3 种语言 + 4 层分层 + 中等部署复杂度）
- **notes**: 识别到工程有 4 层分层（接入 / 调度 / 存储 / 运维），建议 .meta-knowledge-base-guide.md 增加 system_privilege 字段

### [YYYY-MM-DDTHH:MM:SS] TURN-1 · Stage A · A2 完成
- **action**: result_collected
- **sub_agent**: kb-primer-A2 (agent_id=`e5f6g7h8...`)
- **result**: success
- **artifacts**: 适配参数已收集（modules 维度）
- **scale_label**: large（判定依据：12 个模块 + 中高耦合，跨模块引用密集）
- **notes**: 依赖关系复杂，建议 modules 文档增加 depends_on 拓扑图

### [YYYY-MM-DDTHH:MM:SS] TURN-1 · Stage A · A3 完成 + Stage A 收尾
- **action**: stage_completed
- **stage**: A
- **sub_agents_completed**: 3/3
- **sub_agents_failed**: 0
- **duration**: 8min
- **final_scale**: large（取 A1=medium / A2=large / A3=medium 的最大值）
- **artifacts_produced**:
  - A1 适配参数（architecture）
  - A2 适配参数（modules）
  - A3 适配参数（flows/troubleshooting）
- **next_stage**: B - 文档清单规划

### [YYYY-MM-DDTHH:MM:SS] TURN-2 · Stage B · 分派文档清单规划 sub-agents
- **action**: dispatch
- **stage**: B
- **sub_agents**:
  - `kb-primer-B1` (Architecture): spawned, agent_id=`m3n4o5p6...`
  - `kb-primer-B2` (Modules): spawned, agent_id=`q7r8s9t0...`
  - `kb-primer-B3` (Flows): spawned, agent_id=`u1v2w3x4...`
  - `kb-primer-B4` (Troubleshooting): spawned, agent_id=`y5z6a7b8...`
- **expected_output**: 四份文档清单 + 骨架文件
- **status**: pending

### [YYYY-MM-DDTHH:MM:SS] TURN-2 · Stage B · B2 失败
- **action**: error
- **sub_agent**: kb-primer-B2 (agent_id=`q7r8s9t0...`)
- **error_type**: agent_timeout
- **error_detail**: "sub-agent exceeded 300s timeout while scanning src/service/ directory"
- **decision**: retry（第一次重试）
- **retry_count**: 1/2

### [YYYY-MM-DDTHH:MM:SS] TURN-2 · Stage B · B2 重试中
- **action**: retry_dispatched
- **sub_agent**: kb-primer-B2-retry1 (agent_id=`c9d0e1f2...`)
- **adjusted_prompt**: 缩小扫描范围到 src/service/core/（排除测试和工具目录）
- **status**: pending

---

## Resume Guide
<!-- 本节是给新 Agent 的恢复指南。格式固定，不随进度变化。 -->

### 恢复协议

如果你是被唤醒继续执行本 Goal 的新 Agent，请按以下步骤恢复：

1. **读取任务文档的 Progress Log**（**唯一权威状态源**）——了解 Goal 整体进度：哪些 Stage 已完成、当前处于哪个 Stage/Batch
2. **读取本 Journal 的 Entries**——获取当前 Stage/Batch 内 sub-agent 级别的明细
3. **对于每个 pending 的 sub-agent**：
   - 如果条目中有 `agent_id` → 说明已分派但结果未收集。直接重新分派（session 大概率已崩溃，agent_id 不可追溯），使用相同 prompt，写入新 agent_id
   - 如果条目中无 `agent_id` → 说明尚未分派。重新构造 prompt 并分派
4. **对于每个 failed 且未达到重试上限的 sub-agent**：按原 prompt 重试（最多 2 次）
5. **对于每个 failed 且已超重试上限的 sub-agent**：将其标记为 `abandoned`，继续推进
6. **每完成一个批次/Stage 后**：更新任务文档 Progress Log（权威状态）+ 追加本 Journal 条目（详情）
7. **不要依赖原对话上下文**——任务文档 Progress Log + Journal Entries + 文件系统 = 所有需要的信息
```

### 9.4 条目的语义标记

Journal 条目使用 `action` 字段标记语义类型。主 Agent 在写入 Journal 时使用对应的 action 值：

| action | 含义 | 触发时机 |
|--------|------|---------|
| `dispatch` | 分派了 sub-agent(s) | 主 Agent 决定分派一批 sub-agent |
| `result_collected` | 收集到 sub-agent 的返回结果 | sub-agent 完成，主 Agent 读取结果后 |
| `retry_dispatched` | 重试了一个失败的 sub-agent | 失败后决定重试 |
| `retry_exhausted` | 重试次数耗尽，放弃该 sub-agent | 重试 2 次仍失败 |
| `error` | 发生了异常 | sub-agent 失败、文件写入失败等任何异常 |
| `decision` | 主 Agent 做出了一个需要记录的选择 | 如"降级为手动模式""跳过该模块" |
| `stage_completed` | 一个 Stage 完成 | Stage 全部子任务完成、验证通过 |
| `goal_completed` | Goal 完成 | 所有 Stage 完成，GOAL_MET 条件满足 |
| `goal_failed` | Goal 失败 | 超过最大轮次或全部 Stage 阻塞 |

### 9.5 条目的必需字段与可选字段

每个条目必须包含以下字段（确保新 Agent 能可靠解析）：

| 字段 | 必需 | 说明 |
|------|------|------|
| `timestamp` | ✅ | ISO 8601 格式，条目标题的一部分 |
| `turn_id` | ✅ | `TURN-N`，N 为当前 Goal 的轮次计数 |
| `action` | ✅ | 语义标记（见 §9.4） |
| `status` | ✅ | `pending` / `success` / `failed` / `abandoned` |

可选字段（根据 action 类型）：

| 字段 | 适用 action | 说明 |
|------|-----------|------|
| `sub_agents` | dispatch | 分派的 sub-agent 列表（名称 + agent_id） |
| `sub_agent` | result_collected, error, retry_dispatched | 单个 sub-agent 的名称和 id |
| `artifacts` | result_collected, stage_completed | 产出的文件/数据 |
| `error_type` | error | 错误分类（agent_timeout / api_error / invalid_output / file_write_error） |
| `error_detail` | error | 错误详情 |
| `decision` | decision, error | 主 Agent 的决策说明 |
| `retry_count` | retry_dispatched, retry_exhausted | 重试次数 |
| `adjusted_prompt` | retry_dispatched | 重试时的 prompt 调整 |
| `duration` | stage_completed, goal_completed | 耗时 |
| `next_stage` | stage_completed | 下一 Stage 的名称 |

### 9.6 进度可重拭（Resumability）的设计

整个体系的可重拭性依赖三个要素的配合：

```
任务文档 Progress Log（AFMT-KB-SEED-*.md）
    ↑ 权威状态源：哪个 Stage/Batch 处于什么状态（每 Stage 完成时更新）
Journal Entries（pass/kb-seed-journal-*.md）
    ↑ 事件流水：每个 sub-agent 的分派、完成、重试、错误（追加式）
文件系统（.think-tank/knowledge-base/ 下的实际文件）
    ↑ 事实核对：Journal 声称完成的，文件是否真的存在
```

**恢复流程**（新 Agent 视角）：

```
1. 读取任务文档 Progress Log → 找到 Goal 1/2/3 哪个是第一个 status ≠ ✅ 的
2. 定位到该 Goal 的 current_stage 和 current_batch（Progress Log 中标注）
3. 读取该 Goal 的 Journal Entries 中当前 Stage/Batch 相关的条目
4. 扫描这些条目，构建"待处理清单"：
   a. 所有 status=pending 且无 agent_id 的 → 从未分派，需要分派
   b. 所有 status=pending 且有 agent_id 的 → 保守处理，视为结果丢失，重新分派（使用相同 prompt，写入新 agent_id）
   c. 所有 status=failed 且 retry_count < 2 的 → 需要重试
   d. 所有 status=failed 且 retry_count >= 2 的 → 标记 abandoned，跳过
5. 验证文件系统：
   - 对于已完成的 sub-agent，检查其声明的 artifact 文件是否真实存在
   - 不存在 → 降级该 sub-agent 为 failed，重新分派
6. 从待处理清单的第一项开始执行
7. 每完成一个操作 → 追加 Journal 条目；每完成一个 Stage/Batch → 更新任务文档 Progress Log
```

**关键约束**：
- 新 Agent **不得依赖原对话上下文**。任务文档 Progress Log + Journal Entries + 文件系统 = 所有需要的信息
- 恢复时不对 agent_id 做活跃性检查（需要恢复的场景大概率是整个 session 崩溃，所有 agent_id 均已失效）。对任何 pending 且带 agent_id 的条目，直接重新分派。最多造成少量重复工作——idempotency 由文件写入覆盖保证（第二次写入覆盖第一次的结果）
- 主 Agent 在恢复时发现 Journal 中有未完成的工作 → 继续推进。发现所有工作已完成 → 验证产物完整性后输出 GOAL_MET

### 9.7 可追溯性：问题与决策记录

Journal 不仅记录进度，还记录过程中的"异常信号"。这些信号在事后可以聚合分析，用于流程优化。

**需要记录的异常类型**：

| 异常类型 | 记录内容 | 事后用途 |
|---------|---------|---------|
| Sub-agent 超时 | 哪个 sub-agent、扫描哪个目录时超时 | 调整 sub-agent 的扫描范围或 timeout 设置 |
| Sub-agent 返回无效输出 | 哪个 sub-agent、输出与 schema 的差异 | 优化 prompt 模板中的 schema 说明 |
| 重试后成功 | 原 prompt vs 调整后的 prompt | 识别 prompt 的常见缺陷，固化调整策略 |
| 模块依赖图与预期不符 | Stage B 发现的依赖关系与 Stage A 推断的差异 | 优化 Stage A 的依赖推断方法 |
| Review 发现反复出现的问题 | 哪个文档、什么问题在多轮 Review 中反复出现 | 优化 Stage 3 文档生成的 prompt，在源头避免 |
| 人类介入的决策点 | 为什么需要人类决策、决策内容 | 识别可以自动化的决策点 |

在 Journal 中，这些信息以 `decision` 或 `error` action 记录，包含 `error_detail` 和 `decision` 字段。

**事后分析**：
- Goal 3 Stage 8 收尾时，主 Agent（或一个独立的 sub-agent）读取三个 Journal 文件，聚合异常信号 → 输出 `pass/kb-seed-postmortem.md`
- 同时输出 `pass/kb-seed-summary.md`（构建完成摘要，一页纸）
- postmortem 包含：各 Goal 耗时、异常统计、重试率、最耗时的 Stage、最常见的错误类型、流程优化建议
- summary 包含：三个 Goal 完成状态、文档数量（A/F/M/T）、QA 问题统计、关键决策、下一步行动

### 9.8 Journal 写入的并发安全

只有主 Agent 写入 Journal 文件。Sub-agent 不接触 Journal。

但主 Agent 在一轮中可能同时收集多个 sub-agent 的结果。写入顺序：
1. 按 sub-agent 完成顺序逐个追加条目（每个 sub-agent 一个 `result_collected` 条目）
2. 所有 sub-agent 收集完毕后，追加一个 `batch_completed` 或 `stage_completed` 条目
3. 最后更新 State Summary

一轮中 Journal 被多次写入（追加模式），每次写入是原子的（Edit 工具的单次调用）。不要求一轮中的 Journal 写入在一个事务中——如果在一轮中间崩溃，Journal 会留下部分 sub-agent 完成 + 部分 pending 的记录，这正是恢复协议设计为能够处理的情况。

### 9.9 任务文档 Progress Log 的更新时机

Journal Entries 由主 Agent 在每个 sub-agent 事件时**追加**（不重写）。任务文档 Progress Log 作为唯一权威状态源，主 Agent 在以下时机**重写**对应 Stage/Batch 的状态行：

```
每个 Stage/Batch 完成时：标记为 ✅，写入耗时和产出摘要
每个 Stage/Batch 开始时：标记为 ⏳，写入分派时间
遇到 error 且决定重试时：不改 Stage 状态（仍为 ⏳），Journal 追加 error 条目
Goal 完成/失败时：更新 Goal 顶层状态
```

Progress Log 的粒度是 Stage/Batch 级别（"Stage B 已完成 ✅"），不含 sub-agent 明细——明细在 Journal Entries 中。这确保两处不会有重复维护：Progress Log 只管"某个 Stage 到没到"，Journal 只管"过程里发生了什么"。

### 9.10 日志示例：一个完整的 Stage 执行

以下展示一个 Stage 从开始到结束的完整 Journal 片段，包含设计中的所有要素：

```
### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · 分派文档生成 sub-agents
- **action**: dispatch
- **stage**: 3
- **batch**: 1/3
- **sub_agents**:
  - `doc-gen-core-runtime`: spawned, agent_id=`f1a2b3c4...`
  - `doc-gen-state-store`: spawned, agent_id=`d5e6f7g8...`
  - `doc-gen-event-bridge`: spawned, agent_id=`h9i0j1k2...`
- **expected_output**: 三份完整的模块文档 + 三份 QA jsonl
- **status**: pending

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · doc-gen-core-runtime 完成
- **action**: result_collected
- **sub_agent**: doc-gen-core-runtime (agent_id=`f1a2b3c4...`)
- **result**: success
- **artifacts**:
  - `modules/core-runtime.md` (生成完毕)
  - `pass/core-runtime.qa.jsonl` (3 条问题，2 medium + 1 minor)
- **notes**: §5 设计决策章节有一条 critical 级别的不确定声明（某个平台相关字段的来源），已写入 qa jsonl

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · doc-gen-state-store 完成
- **action**: result_collected
- **sub_agent**: doc-gen-state-store (agent_id=`d5e6f7g8...`)
- **result**: success
- **artifacts**:
  - `modules/state-store.md` (生成完毕)
  - `pass/state-store.qa.jsonl` (0 条问题)
- **notes**: 无歧义，文档完整

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · doc-gen-event-bridge 超时
- **action**: error
- **sub_agent**: doc-gen-event-bridge (agent_id=`h9i0j1k2...`)
- **error_type**: agent_timeout
- **error_detail**: "sub-agent exceeded 300s timeout while generating event-bridge.md. Likely too many source files to read."
- **decision**: retry with narrowed scope（排除 `<file>.<ext>`，该文件的细节在 state-store 中已覆盖）
- **retry_count**: 1/2

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · doc-gen-event-bridge 重试中
- **action**: retry_dispatched
- **sub_agent**: doc-gen-event-bridge-retry1 (agent_id=`l3m4n5o6...`)
- **adjusted_prompt**: "跳过 `<file>.<ext>` 的详细分析，引用 state-store.md 中已有的描述"
- **status**: pending

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 · doc-gen-event-bridge 重试成功
- **action**: result_collected
- **sub_agent**: doc-gen-event-bridge-retry1 (agent_id=`l3m4n5o6...`)
- **result**: success
- **artifacts**:
  - `modules/event-bridge.md` (生成完毕)
  - `pass/event-bridge.qa.jsonl` (1 条问题，1 medium)
- **notes**: 重试成功。排除 `<file>.<ext>` 后仍在 300s 内完成了文档

### [YYYY-MM-DDTHH:MM:SS] TURN-4 · Stage 3 Batch 1 完成
- **action**: stage_completed
- **stage**: 3
- **batch**: 1/3
- **sub_agents_completed**: 3/3（含 1 次重试）
- **sub_agents_failed**: 0
- **duration**: 23min
- **errors**: 1（event-bridge 首次超时，重试后恢复）
- **next**: Batch 2/3（replication, cluster）
```

### 9.11 Journal Entries 与任务文档 Progress Log 的关系

v6 起明确职责分离，消除 v5 版本中两者维护重叠的问题：

| | 任务文档 Progress Log | Journal Entries |
|------|-------------|--------|
| **位置** | 任务文档末尾（`AFMT-KB-SEED-*.md`） | `pass/kb-seed-journal-*.md` |
| **角色** | **唯一权威状态源** | 事件流水（详情） |
| **粒度** | Stage/Batch 级别（"Stage B 完成 ✅"） | Sub-agent 级别（每个 sub-agent 的分派、结果、重试） |
| **用途** | Goal 条件评估（Haiku 读此判断进度）+ 新 Agent 恢复的第一站 | 事后复盘 + 恢复时的详细上下文 |
| **谁写** | 主 Agent 每 Stage/Batch 完成时**重写**对应状态行 | 主 Agent 每个 sub-agent 事件时**追加**条目 |
| **格式** | 简洁表格 | 结构化 YAML-like 条目 |
| **同步风险** | 无（权威源，无需与其他数据源同步） | 无（追加式流水，不聚合状态） |

---

## 10. 种子包的内容

AFMT-KB-SEED 作为可分发物，包含以下文件：

| 文件 | 类型 | 说明 |
|------|------|------|
| `VERSION` | 版本文件 | 单行文本，内容为种子包版本号（如 `1.0.0`）。Goal 1 Stage B 读取此文件填入 `index.yaml.meta.seed_pack_version` |
| `CHANGELOG.md` | 版本变更记录 | 记录各版本间的破坏性变更、迁移指南。供后续升级工具对照 |
| `AFMT-KB-SEED-primer.md` | Goal 1 任务文档模板 | 需要被 `/goal` 驱动执行。文档中有 `{ADAPT}` 标记表示需根据目标工程调整的段落 |
| `AFMT-KB-SEED-bake.md.template` | Goal 2 任务文档模板 | 不直接执行。由 Goal 1 的 Stage C 填充实例化参数后生成 `-bake-{project}.md`，再被 Goal 2 执行 |
| `AFMT-KB-SEED-infra-setup.md` | Goal 3 任务文档 | Skill 迁移、基础设施搭建、元文档移植的执行指令。不模板化，直接分发（目标工程 KB 结构已就绪） |
| `templates/meta-knowledge-base-guide.md.template` | 元规范模板 | 含 `{ADAPT}` 标记，由 Stage A 的 sub-agent 适配 |
| `templates/meta-work-guide.md` | 通用模板 | 直接可用（Work 文档规范是通用的） |
| `templates/meta-runbook-guide.md` | 通用模板 | 直接可用 |
| `templates/meta-incident-guide.md` | 通用模板 | 直接可用 |
| `schemas/stage-b-output.schema.json` | Schema | Stage B sub-agent 输出格式 |
| `schemas/qa-jsonl-entry.schema.json` | Schema | QA jsonl 每行的格式 |
| `schemas/goal2-task-doc.schema.yaml` | Schema | Goal 2 任务文档的参数结构 |
| `prompts/stage-a-subagent.md` | Prompt 模板 | Stage A 的 sub-agent prompt |
| `prompts/stage-b-subagent.md` | Prompt 模板 | Stage B 的 sub-agent prompt |
| `prompts/stage-3-subagent.md` | Prompt 模板 | Goal 2 Stage 3 文档生成 sub-agent prompt |
| `prompts/stage-4-subagent.md` | Prompt 模板 | Goal 2 Stage 4 Review sub-agent prompt |
| `prompts/stage-5-aggregation.md` | Prompt 模板 | Goal 2 Stage 5 QA 聚合指令 |
| `skills/explore-kb/` | Skill | 已去耦合，直接复制。Goal 3 Stage 7 中迁移 |
| `skills/maintain-kb/` | Skill | 已去耦合，直接复制。Goal 3 Stage 7 中迁移 |
| `skills/commander/` | Skill（模板化） | v1.1 新增。含 SKILL.md + 5 references。文件中使用 `{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` / `{{MODULE_LIST_INLINE}}` 等占位符。Goal 3 Stage 7 迁移时边复制边替换占位符（`{ADAPT}` 标记保留） |
| `templates/commander/module_header_meta.md` | 通用规范 | v1.1 新增。module_header 字段规范与构建指令。Goal 3 Stage 8 中复制到 `.think-tank/commander/` |
| `templates/commander/README.md.template` | 元文档模板 | v1.1 新增。含 `{ADAPT}` 标记。Goal 3 Stage 8.3 由 sub-agent 填充生成实际 README |
| `templates/commander/history/decisions/TEMPLATE.md` | 通用模板 | v1.1 新增。历史决策记录模板 + 目录规范，直接可用 |
| `templates/commander/history/decisions/index.yaml.template` | 索引模板 | v1.1 新增。含 `{{PROJECT_NAME}}` / `{{TODAY}}` 占位符。Goal 3 Stage 8.1 中替换后写入 |
| `prompts/stage-commander-header-gen.md` | Prompt 模板 | v1.1 新增。Goal 3 Stage 8.2 从 `knowledge-base/modules/<id>.md` 上浮生成 `commander/module_header_<Name>.md` 的 sub-agent prompt，内嵌上浮转换规则 |
| `knowledge-extract-qa-guide.md` | 关键元文档 | 通用文档，定义 QA 提取工作流（内嵌格式规范，替代 qa-template.md）。Goal 3 中迁移 |

---

## 11. 模板中的 `{ADAPT}` 标记机制

种子包中的模板文件使用 `{ADAPT: ...}` 标记表示需要根据目标工程适配的段落：

```markdown
<!-- 示例：.meta-knowledge-base-guide.md 模板片段 -->

## 2.1 目录结构

```
.think-tank/knowledge-base/
├── architecture/
│   ├── system-overview.md
{ADAPT: 如果工程有特殊的分层（如内核态/用户态/前端/后端），在此补充 architecture/ 下的额外文档}
│   └── ...
```

### 2.2 层级结构

{ADAPT: 如果工程不是"内核态+用户态服务+CLI"这种经典分层，
       而是微服务/纯前端/数据管道等其他形态，需要调整层级描述和职责表格}
```

Stage A 的 sub-agent 需要识别这些 `{ADAPT}` 标记，根据工程特征填充具体内容，或标注"本工程不适用"。

---

## 12. 总结：多 Goal 模型

| 维度 | 单 Goal（v2） | 多 Goal（v5） |
|------|-------------|-------------|
| **确定性** | Stage 3 不知道文档数量，需动态决策 | Goal 2 参数完全确定；Goal 3 纯文件操作 |
| **职责分离** | 分析 + 执行 + 基础设施耦合 | Goal 1 分析规划 → Goal 2 文档生成 → Goal 3 基础设施 |
| **可恢复性** | 中断后从中间状态恢复困难 | 每个 Goal 独立，前一 Goal 的产出是后一 Goal 的输入 |
| **可分发性** | 通用模板（接收方自行适配） | 通用模板 + Goal 1 运行后得实例化模板 |
| **人类介入** | Goal 跑完才出 QA | Goal 1 后可确认清单（建议非强制）→ Goal 2 完成后回答 QA → Goal 3 无需介入 |
| **上下文污染** | 多阶段异质任务混杂 | 每个 Goal 的主 Agent 只处理一类任务 |

---

## 13. 未解决的问题

| 编号 | 问题 | 状态 |
|------|------|------|
| Q1 | Goal 1 的 Stage C 中，"实例化 Goal 2 任务文档"应该由主 Agent 做还是分派 sub-agent？ | 建议：参数 <50 项则主 Agent 做 |
| Q2 | Goal 1 和 Goal 2 之间：人类是否需要 Review 文档清单再启动 Goal 2？ | 建议是。快速扫一眼确认。v6 新增"预计成本"章节后，此确认可与成本预审合并 |
| ~~Q3~~ | ~~Commander skill 的模板化——场景 D 依赖 symptom-index + Runbook~~ | **v1.1 已解决**：Commander skill 已重构为"协议 + 内容"分离——协议（SKILL.md + 5 references）随种子包分发，内容（module_header）由 Goal 3 Stage 8 从 modules/ 上浮生成。场景 D 依赖的 symptom-index + Runbook 已由 Goal 1 Stage B4 构建，Commander 直接引用 |
| Q4 | 是否需要"验证 Goal"——在已知工程上试跑以验证种子包的完整性？ | **v6 起明确为强制前置**。以任一真实工程作为验证目标，未通过验证不发布 |
| ~~Q5~~ | ~~工程规模判定的阈值是否合理？~~ | **v6 已扩展**：多维度取最大值（模块数/源文件/语言/分层/耦合度）。仍需 Q4 自跑校准 |
| Q6 | postmortem 和 summary 应该放在 pass/ 还是 KB 根目录？ | 建议 pass/，与 Journal 同级，方便人类在一个目录下看到完整构建记录 |
| Q7 | 成本估算公式的经验值（每 sub-agent token 数、耗时）需要多少个工程校准才可信？ | **v6 新增**。建议至少 3 个不同规模的工程，v6-ready 版本发布前完成 |
| Q8 | 严重度分级示例覆盖是否足够？边界情况（跨维度问题、组合性问题）如何处理？ | **v6 新增**。建议在 Q4 自跑中记录 sub-agent 的判定分歧，作为 v6.1 的补充示例来源 |
