---
name: maintain-kb
description: >
  知识库一致性维护。通过增量/全量两种模式系统性检查 KB 与代码的一致性，
  输出结构化检查报告到 pass 目录，支持问答式逐条讨论审批（含 sub-agent 交叉核实），
  并在人工确认后修复不一致项。增量模式基于 git diff 自动定位受影响的条目；
  全量模式分层遍历全部 KB 条目。支持 beacon 文件追踪维护历史。
user-invocable: true
---

# maintain-kb — 知识库一致性维护

> 本 skill 提供 KB 与代码一致性的**系统性检查与修复**能力。默认增量模式，支持全量模式。运行时自动从 KB 元文档自发现产品名、索引结构、条目规模等环境信息，复制到任何遵循 `.think-tank/knowledge-base/` 约定的仓库即可使用。

---

## 1. 维护模式概览

| 模式 | 触发 | 适用场景 |
|------|------|---------|
| 增量（默认） | `/maintain-kb` | 日常维护，代码有小批量提交后同步 KB |
| 全量 | `/maintain-kb full` | 首次建立 beacon、怀疑大面积过时、重大版本发布前 |

**Beacon 文件**（`.think-tank/knowledge-base/.meta-maintain-beacon.yaml`）记录上次维护的 commit hash 和统计。增量模式依赖 beacon 计算 git diff 范围。

---

## 2. 前置检查（两种模式共用）

**硬约束**：执行前必须 `git status --porcelain`。非空 → **拒绝执行**，提示用户先 commit 或 stash。

### 2.1 KB 环境自发现

确认工作区干净后，读取 KB 元数据以了解当前仓库环境：

1. 确认 KB 根目录 `.think-tank/knowledge-base/` 存在。若不存在 → 终止执行，提示「当前仓库未检测到知识库（`.think-tank/knowledge-base/` 目录不存在），maintain-kb 依赖 KB 索引进行条目定位」。

2. 读取 `.think-tank/knowledge-base/index.yaml`，提取以下信息：
   - `project.name` → **产品名**（用于报告标题和日志标识）
   - 统计 `modules`、`flows`、`troubleshooting`、`architecture` 各段落下条目数 → **各层条目数**（用于全量模式范围评估和摘要输出）
   - 检查 `covers_code` / `covers_commands` 字段是否存在于索引条目中且至少有一条非空值 → **增量模式可用性**。若不存在或全为空，增量模式不可用，后续自动降级为全量模式
   - 检查 `depends_on` / `involves_modules` / `refs` 字段分别是否存在于索引条目中 → **传递展开可用字段**。存在的用于二级展开，不存在的静默跳过

3. 检查文件系统：
   - `.agents/skills/explore-kb/SKILL.md` 是否存在 → **peer skill 可用性**。若不存在，JSONL 信箱协作机制不可用（旧布局可回退检查 `.claude/skills/explore-kb/SKILL.md`）
   - `.think-tank/knowledge-base/.meta-explore-issues.jsonl` 是否存在 → **JSONL 信箱可用性**（需 peer skill 可用为前提）

4. 输出自发现摘要，示例格式：
   > **KB 环境自发现**：产品 `{project.name}`，条目数 模块 N1 / 流程 N2 / 排查 N3 / 架构 N4，增量模式{可用/不可用}，JSONL 协作{可用/不可用}。

### 2.2 Beacon 状态判定

```
读取 .meta-maintain-beacon.yaml
  ├── beacon 存在且 commit 在历史中？→ 按所选模式执行
  ├── beacon 不存在？→ 降级为全量模式（首次运行）
  ├── beacon commit 不在历史中？→ 尝试 git fetch --deepen=100 恢复
  │   ├── 恢复成功 → 正常增量
  │   └── 仍不可达 → 降级为全量模式
  ├── beacon commit == HEAD 且 explore_issues 为空？→ 跳过（无变更）
  └── beacon commit == HEAD 但 explore_issues 非空？→ 仅检查 explore_issues 涉及的条目
```

---

## 3. 增量维护流程

> 增量模式依赖 `covers_code` / `covers_commands` 字段将 git diff 文件路径映射到 KB 条目。若 §2.1 自发现判定增量模式不可用（索引中无这些字段或全为空），则自动降级为全量模式，并提示：「索引中未检测到 covers_code / covers_commands 映射，增量模式不可用，降级为全量模式。」

增量模式可用时，按以下流程执行：

```
git diff beacon.commit..HEAD --name-only
  → 匹配 index.yaml covers_code / covers_commands
  → 直接受影响条目集合 A

两级展开（不递归）：
  Level 1: 直接 covers_* 命中
  Level 2: 依赖字段反向查找——使用 §2.1 自发现中确认存在的传递展开字段（如 depends_on / involves_modules / refs），查找包含 A 中条目 id 的条目
  → 间接影响条目集合 B

（若 peer skill 可用且 JSONL 信箱存在）读取 .meta-explore-issues.jsonl → 强制追加集合 C

A ∪ B ∪ C = 待检查条目集合
```

**边界**：
- 新增未匹配文件 → 报告标注「未关联 KB 的新代码」
- 删除文件 → 关联条目生成 `defer / code_deleted`
- index.yaml 自身变更 → 优先检查映射关系
- 待检查集合为空 → 结束，无需维护

---

## 4. 全量维护流程

分层逐批检查（排除 works/ 和 pass/，条目数由 §2.1 自发现统计）：

| 优先级 | 层级 | 检查策略 |
|--------|------|---------|
| P0 | `modules/` | API 签名、数据模型、业务流程、文件清单 |
| P1 | `flows/` | 调用链匹配 |
| P1 | `troubleshooting/` | 行号漂移、已知陷阱 |
| P2 | `architecture/` | 技术栈版本号等硬事实 |

执行前输出实际检查范围：各层条目数（来自自发现统计）、预计总条目数。

---

## 5. 检查阶段（只读）

**绝不写入任何 KB 文档或工程代码**。仅读取 KB 文档和代码，比对差异。检查报告输出到 pass/ 目录（不入索引，属于外部材料）。

逐条比对：读 KB 条目 → 通过 covers_code 定位代码 → 比对函数签名、数据结构、API 接口、IPC 协议、数据流、文件清单、行号引用、交叉引用。

**发现分级**：`outdated`（KB 与代码不一致）/ `missing`（代码有但 KB 未覆盖）/ `drift`（细节漂移）

**发现方向**：
- `update_kb`：代码为准，AI 能可靠判断如何更新 → 确认阶段由人类审批
- `defer`：AI 无法可靠判断如何更新 → 推迟，由人类决策

defer 触发条件：constraint_conflict / trap_triggered / architectural_change / info_incomplete / scale_too_large / code_deleted

**检查报告格式**（每条发现）：

```yaml
- id: finding-<NNN>
  severity: outdated | missing | drift
  direction: update_kb | defer
  defer_reason: <enum>           # 仅 direction=defer
  confidence: 0.0-1.0
  impact: high | medium | low
  kb_entry: <entry-id>
  section: "<§>"
  kb_claims: "<KB 描述>"
  code_reality: "<代码实际>"
  code_ref: "<文件:行号>"
  suggestion: "<建议修复>"
```

### 5.1 输出检查报告到 pass 目录

检查阶段结束后，**先将完整的结构化报告写入 pass 目录**，再进行人工确认。

- 若 `pass/` 目录不存在 → 创建后写入
- 文件命名：`pass/maintain-kb-<mode>-check-<YYYYMMDD>-<seq>.md`，seq 为当日序号（1/2/3...）
- 报告内容包含：
  - 摘要（检查条目数 / CLEAN 数 / 发现数 / update_kb / defer / missing）
  - CLEAN 条目列表（附简要验证说明）
  - 逐条发现详情（按 impact 降序，含 severity / direction / confidence / kb_claims / code_reality / suggestion）
  - 补充观察（已验证正确的关键声明，防止后续误报）
  - 审批待办区（供后续填写）

---

## 6. 确认阶段（人工介入）

### 6.1 呈现摘要并询问讨论方式

报告写入 pass 后，向人类呈现检查摘要，然后**主动询问**：

> 是否以问答讨论方式逐条展开？你也可以回复：
> - **全部批准** — 一键批准所有 update_kb 项（defer 项仍需单独讨论）
> - **只讨论高影响** — 仅逐条讨论 high impact，其余全部批准
> - **只讨论第一个** — 先讨论第一条，之后再看

根据人类的回复选择对应路径：

| 人类回复 | 行为 |
|---------|------|
| "全部批准" | 所有 update_kb 项标记为「批准」，defer 项逐一询问（维持推迟 / 重分类） |
| "只讨论高影响" | high impact 项逐条讨论，medium/low 全部批准，defer 逐一询问 |
| "只讨论第一个" | 从最高影响项开始逐条，每完成一条询问是否继续下一条 |
| 无特别指示（默认） | 按 impact 从高到低逐条展开讨论 |

### 6.2 逐条讨论格式

每条发现的讨论以简洁表格呈现核心信息：

```
## F-XX · <entry-id> · <一句话描述>

| 属性 | 内容 |
|------|------|
| **KB 章节** | <section> |
| **KB 声称** | <kb_claims> |
| **代码实际** | <code_reality> |
| **建议** | <suggestion> |

批准？
```

### 6.3 讨论过程中的 sub-agent 核实

讨论期间，当发现的问题涉及代码细节且置信度不是 100% 时，**主动提示人类可以派遣 sub-agent 核实**：

> 该发现涉及代码细节，如需确认可以派遣 sub-agent 核实实际情况。

人类可回复"派遣 sub agent 核实"或"派 agent 确认"等。此时启动一个独立的 Explore agent 读取相关代码或 KB 文件，返回核实结论。

核实结果可能：
- **确认发现正确** → 继续讨论该项的批准/否决
- **发现原始判断有误** → 撤回该项，更新 pass 文档标记为 ⊗ 撤回
- **发现部分偏差** → 更新建议内容后继续讨论

### 6.4 发现的结果状态

每项发现在讨论结束后必须落入以下状态之一：

| 状态 | 含义 | 后续行为 |
|------|------|---------|
| **批准** | 人类同意按建议修复 | 进入更新阶段执行 |
| **否决** | 人类不同意修复 | 不执行，写入 beacon 记录 |
| **挂起** | 暂时搁置，需要更多信息或等待条件成熟 | 不执行，写入 beacon deferred；下次维护时重新评估 |
| **待确认** | 已派遣 sub-agent 核实中 | 核实完成后重分类为批准/否决/挂起 |

### 6.5 讨论期间更新 pass 文档

每完成一批讨论（如一个 impact 级别讨论完毕、或人类表示"先更新文档"），**同步更新 pass 文档**：

- 在对应发现条目后追加状态标注：`**状态**: ✅ 批准 / ❌ 否决 / ⏸️ 挂起 / 🔍 待确认`
- 如有补充信息（如人类提供的上下文、sub-agent 核实结论），以备注形式追加
- 文档顶部的状态行同步更新（如 `已讨论 12/20，批准 10，否决 1，挂起 1`）
- 如果讨论中新发现了值得研发关注的代码疑点，追加到文档末尾的「待研发核实问题」表格

---

## 7. 更新阶段（写入）

### 7.1 进入条件

全部发现状态明确（无「待确认」残留）后，**主动提示人类**：

> 全部检查结果状态已明确（批准 X / 否决 Y / 挂起 Z），是否开始知识库维护（执行修复）？

人类确认后进入更新阶段。

### 7.2 执行修复

**仅处理状态为「批准」的发现**。被否决或挂起的写入 beacon。

逐项修复：定位 KB 段落 → 按代码更新描述 → 更新行号引用（或按人类决策移除行号改用函数名定位）→ 更新 updated 日期

**不做**：不处理未批准项、不自行判断 direction、不创建未经确认的新条目、**不修改任何工程代码**。

### 7.3 修复完成

修复完成后向人类汇报修改摘要（修改了哪些文件、各文件修复项数），等待人类确认 commit。

---

## 8. 收尾

1. 更新 beacon：当前 HEAD commit + 本次维护统计 + deferred_items（含挂起项）
2. （若 JSONL 信箱可用）truncate `.meta-explore-issues.jsonl`
3. 输出维护摘要：检查数 / 发现数 / 批准数 / 修复数 / 否决数 / 挂起数
4. 如有 deferred 或挂起项，提醒用户下次维护时重新评估
5. 建议 commit message 模板：`docs: maintain-kb <incremental|full> check (<N> entries, <M> fixed)`

---

## 9. 边界与注意事项

- **代码全程只读**：maintain-kb 只修改 KB 文档，绝不修改任何工程代码
- **works/ 不在维护范围**：历史设计档案，多版本共存，以当前代码验证会产生误报
- **pass/ 不在维护范围**：外部只读材料，不入索引
- **与 explore-kb 协作**（若 peer skill 可用）：explore-kb 发现的 KB 不一致通过 `.meta-explore-issues.jsonl` 传递，maintain-kb 消费后 truncate
- **首次运行**（beacon 不存在）→ 自动降级为全量模式
- **受影响的 KB 条目 >20** → 建议分批或降级为按层级检查
