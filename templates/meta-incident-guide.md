---
id: .meta-incident-guide
title: "Incident Record 编写规范与 Runbook 转换指南"
status: active
created: {{TODAY}}
updated: {{TODAY}}
---

# Incident Record 编写规范与 Runbook 转换指南

> Incident Record 是研发人员在解决一个 TS 知识库尚未覆盖的问题后所做的轻量记录，也是诊断 Runbook 的原始语料来源。本文档定义 Incident Record 的编写规范，以及 AI Agent 将其转换为结构化 Runbook 的完整流程。

---

## 1. 定位

Incident Record 位于 `troubleshooting/incidents/` 目录下，与 `runbooks/` 形成**生产-消费**关系：

```
研发人员解决问题
      │
      ▼
Incident Record（轻量，5-10 分钟）    ← 本文档定义
      │
      ▼
AI Agent 转换                        ← §7 定义
      │
      ▼
Runbook 草稿 → 人工审批 → runbooks/  ← .meta-runbook-guide.md 定义
```

**与 Runbook 的分工**：

| 维度 | Incident Record | Runbook |
|------|----------------|---------|
| 作者 | 研发人员（人类） | AI Agent（从 Incident 转换） |
| 粒度 | 一次具体的事件 | 一类症状的通用诊断程序 |
| 格式 | YAML，自由文本为主 | Markdown，结构化决策树 |
| 时效 | 解决后立即记录 | 积累到一定量后批量转换，或单篇高危时触发 |
| 写作文本量 | ~20-30 行 YAML | ~200-300 行 Markdown |
| 质量要求 | 如实记录即可 | 命令可执行、分支闭合、跳转正确 |

---

## 2. 目录结构

```
troubleshooting/
├── symptom-index.yaml
├── faq.md
├── known-bugs.md
├── incidents/                         ← Incident Record 存放目录
│   ├── .meta-incident-guide.md        ← 本文档
│   └── YYYY-MM-DD_<slug>.yaml         ← Incident Record 文件
└── runbooks/
    ├── .meta-runbook-guide.md
    └── <runbook>.md
```

**文件命名**：`YYYY-MM-DD_<slug>.yaml`，slug 为问题的简短英文描述

---

## 3. Incident Record 模板

### 3.1 完整模板

```yaml
---
incident_id: <YYYY-MM-DD-slug>     # 与文件名一致（不含 .yaml）
date: YYYY-MM-DD                   # 问题解决日期
reporter: <姓名>                    # 排查人员
severity: critical | major | minor
status: resolved | workaround | investigating
---

# 一、症状

symptom:
  description: >
    用户描述的问题现象。用用户/运维的角度写，不要写技术原因。
    可以复述用户原话。
  category: {{ADAPT: 使用本工程 symptom-index.yaml 中定义的分类枚举}}
  trigger_keywords:                # 可选：用户可能使用的自然语言关键词
    - "<关键词 1>"
    - "<关键词 2>"

# 二、诊断过程

diagnosis:
  # 按时间顺序记录关键发现。不需要完整记录每一步，只记录转折点和排除项。
  # 弯路也要记录——知道"不是这个原因"同样有价值。
  steps:
    - "<第 1 步观察或检查动作，及结果>"
    - "<第 2 步>"
    - "..."
  root_cause: >
    <一段话描述根本原因，定位到代码/配置/环境层面即可，无需长篇技术分析>
  excluded:                         # 可选：排查过的错误方向
    - "<最初怀疑的方向 A，检查后排除>"
    - "<方向 B 也排除>"

# 三、解决方案

solution:
  immediate:                        # 临时/紧急修复
    - "<临时恢复操作 1>"
    - "<临时恢复操作 2>"
  permanent:                        # 永久修复（是否需要改代码？改流程？）
    - "<永久修复动作 / 或说明为何无需代码改动>"
  commands:                         # 可直接执行的恢复命令（供 Agent 引用）
    - "<完整可执行的命令 1>"
    - "<命令 2>"

# 四、关联信息

related:
  modules: [<涉及的模块 id 列表>]           # 必须在 modules/ 下存在
  bugs: []                                  # 是否关联 known-bugs.md 中的 BUG-XXX
  existing_runbooks: []                     # 是否已有 Runbook 可部分覆盖此场景
  related_incidents: []                     # 是否与其他 Incident 有关联（同根因不同表现）
```

### 3.2 最小模板

如果时间有限，以下字段为**必填**，其他可省略或事后补充：

```yaml
---
incident_id: <YYYY-MM-DD-slug>
date: YYYY-MM-DD
reporter: <姓名>
severity: critical | major | minor
status: resolved | workaround | investigating
---

symptom:
  description: <一句话描述用户看到的症状>
  category: <从 symptom-index.yaml 定义的分类中选择>

diagnosis:
  root_cause: <一句话描述根本原因>

solution:
  commands:
    - <恢复命令>
```

### 3.3 症状类别（category 枚举）

`category` 的取值必须来自本工程 `symptom-index.yaml` 中定义的分类枚举。分类枚举在 Goal 1 Stage B4 阶段初始化，后续可扩展。

{{ADAPT: 若本工程需要在本文档中固化 category 枚举列表，请在此列出。否则删除此段——引用 symptom-index.yaml 为唯一权威源}}

### 3.4 状态（status 枚举）

| 状态 | 含义 |
|------|------|
| `resolved` | 问题已解决，有明确的根因和恢复步骤 |
| `workaround` | 有临时规避方案但无真正修复（根因未消除） |
| `investigating` | 正在排查中，Incident Record 用于记录中间发现 |

---

## 4. 编写行为指南

### 4.1 什么时候创建 Incident Record

- 解决了 TS 知识库（symptom-index + runbooks + FAQ）中**尚无记录**的问题
- 遇到已知问题但发现了**新的根因或新的恢复路径**
- 排查过程中发现了一个**值得记录的弯路**（"最初以为是 X 结果是 Y"）
- 即使最终没有代码改动（问题出在环境/配置/外部系统），也值得记录——排查路径是可复用的

### 4.2 不需要创建的情况

- 问题已被已有 Runbook 覆盖，且没有发现新信息
- 纯环境问题且无法复现的操作（如"用户重启电脑就好了"且无根因）
- 一次性的配置错误或操作失误

### 4.3 编写原则

- **如实记录，不要美化**：如果排查过程走了弯路，写下来——这比"正确的排查路径"更能帮助后续的 Agent 排除干扰
- **从用户视角描述症状**：写用户可观察的表现，不要用技术术语替换（用户不会说"某某模块的事务提交失败"）
- **根因用技术语言，但要简洁**：不需要写 200 字的技术分析，一两句能定位到代码层面即可
- **命令要可执行**：`solution.commands` 中的命令要完整、可直接复制
- **关联要明确**：模块名用代码中的实际名称，不用中文意译

---

## 5. Incident Record 的状态流转

```
investigating ──→ resolved ──→ Runbook 草稿 ──→ 审批 ──→ runbooks/ 发布
                                      │
                                      │ (workaround)
                                      ▼
                                 积累多篇 workaround 后评估
                                 是否需要建 Runbook
```

- `investigating` → `resolved` 或 `workaround`：问题排查出结论时更新
- 多篇 `resolved` Incident 积累到同一症状类别时，触发批量转换
- 单篇 `resolved` Incident 如果症状覆盖面广（severity 为 critical + 根因空间大），可单独触发转换

---

## 6. 与 index.yaml 的关系

- Incident Record **不入 `index.yaml` 索引**——它是中间产物，不作为 Agent 的检索目标
- Incident Record 仅通过 `symptom-index.yaml` 和 `runbooks/` 间接进入检索体系
- 转换完成后，原 Incident Record 保留不删，作为 Runbook 的溯源依据

---

## 7. AI Agent：Incident → Runbook 转换流程

本章定义 AI Agent 如何将一篇或多篇 Incident Record 转换为诊断 Runbook。

### 7.1 触发条件

| 触发条件 | 说明 |
|---------|------|
| 人工指令 | 用户明确要求："帮我把这个 Incident 转成 Runbook" |
| 批量积累 | 同一 `category` 下积累了 ≥3 篇 `resolved` Incident，用户要求批量处理 |
| 单篇高危 | 某篇 Incident severity 为 `critical`，且根因空间大，用户认为值得独立建 Runbook |

### 7.2 转换步骤（AI Agent 执行）

#### 步骤 1：信息提取

从 Incident Record 中提取以下信息：

| 来源字段 | 提取内容 | 目标 Runbook 位置 |
|---------|---------|-----------------|
| `symptom.description` | 用户可观察的症状描述 | §1 症状确认的标题和引导文字 |
| `symptom.category` | 症状分类 | 用于确定 Runbook 在 decision tree 中的定位 |
| `symptom.trigger_keywords` | 用户可能的自然语言表述 | `symptom-index.yaml` 的 `trigger_keywords` 字段 |
| `diagnosis.steps` | 排查过程（含弯路） | §1 症状确认 + §2 诊断决策树 |
| `diagnosis.root_cause` | 根本原因 | §2 中的根因标签 + §3 中的恢复子节标题 |
| `diagnosis.excluded` | 排查后排除的方向 | §2 中被标为"排除"的检查步骤 |
| `solution.immediate` / `solution.permanent` | 恢复操作 | §3 恢复步骤 |
| `solution.commands` | 可执行命令 | §3 中的命令代码块 |
| `related.modules` | 涉及模块 | Frontmatter `affected_modules` + §4 关联参考 |
| `related.bugs` | 关联的已知 Bug | §4 关联参考（或可选章节 §5.2 已知 Bug 关联） |
| `related.existing_runbooks` | 已有 Runbook 引用 | §2 中的跨 Runbook 引用 |

#### 步骤 2：将排查过程转换为决策树

Incident Record 的 `diagnosis.steps` 是**线性记录**（实际发生了什么的日记），需要转换为**分支决策树**（下次遇到类似问题时沿着哪个分支走）。

**关键转换规则**：

- **线性 → 分支**：将"我检查了 A，结果是 X，然后检查了 B"转换为"如果检查 A 结果是 X → 下一步检查 B；如果结果是 Y → 检查 C"
  - 如果 Incident 只记录了一条路径，AI Agent 需要**推断其他可能的分支**：如果检查 A 的结果是 Y，应该跳到什么结论？推断依据是 `diagnosis.excluded` 和已有模块文档的知识。

- **正向步骤 → 检查命令**：Incident 中的排查动作转换为 Runbook 中的精确命令（完整可执行的命令行）

- **跳跃推理 → 显示链路**：如果 Incident 的排查步骤之间存在隐含的因果跳跃，AI Agent 需要补充中间环节，使每一步的跳转条件明确

- **弯路保留**：`diagnosis.excluded` 中的排查路径应保留在 Runbook 中，标记为"常见误判"或"已验证排除"：

  ```markdown
  > ⚠️ **常见误判**：此症状容易被误判为 <方向 A> 问题。若检查 <关键指标> 正常，可排除此方向。
  ```

**当 Incident 只记录了一条路径时**，AI Agent 需要做以下补充：

1. 基于 `root_cause` 和 `diagnosis.steps`，逆向推断"如果在这一步检查出不同的结果，意味着什么"
2. 查阅 `related.modules` 对应模块文档的 §6「已知陷阱」，补充其他可能的根因分支
3. 查阅相关 `flows/` 流程文档，确认正常路径 vs 故障路径的对照
4. 若补充的分支**无法从已有 KB 知识中确认**，在 Runbook 中标注 `[待确认]`，留待人工审批时补充

#### 步骤 3：编写恢复步骤

从 `solution.commands` 和 `solution.immediate` 生成 §3 恢复步骤：

- 将每个恢复操作拆分为**有序步骤**（1. 2. 3.）
- 每步包含：操作描述 + 完整命令 + 验证方法
- 若恢复涉及数据破坏风险，标注 `> ⚠️ **警告**`
- 若存在多个可选的恢复路径（临时修复 vs 永久修复），用子节区分

#### 步骤 4：补充关联参考

自动生成 §4 关联参考表格：

1. 从 `related.modules` 查找对应模块文档路径（`modules/<module>.md`）
2. 从 `related.bugs` 引用 `known-bugs.md` 中的具体条目
3. 从 `symptom.category` 推断可能相关的流程文档（`flows/`）
4. 如果同一类别下有多个 Runbook，标注跨引用关系

#### 步骤 5：注册到 symptom-index.yaml

在 `symptom-index.yaml` 中新增症状条目：

- 从 `symptom.description` 提炼 `description` 字段（一句话）
- 从 `symptom.trigger_keywords` 填充 `trigger_keywords` 字段
- 从 `solution.commands[0]` 或诊断过程第一步提取 `quick_check` 字段
- 从 `severity` 和 `related.modules` 填充对应字段
- `runbook` 字段指向新生成的 Runbook 文件名

### 7.3 多篇 Incident 合并

当同一 `category` 下有多篇 Incident 时，AI Agent 应：

1. 先通读所有相关 Incident
2. 按症状细分（同一 category 下可能有不同的细分症状，对应不同的 Runbook 或同一 Runbook 的不同分支）
3. 提取出**多条排查路径的交集**——这些交集步骤应该放在 §1 和 §2 决策树的早期，因为多篇 Incident 都经过这些步骤
4. 将各 Incident 特有的排查路径作为决策树的分支

**合并的优先级判断**：
- 多篇 Incident 的 `root_cause` 不同，但 `diagnosis.steps` 前几步相同 → 合并为一个 Runbook，用决策树分叉
- 多篇 Incident 的 `symptom.description` 描述的完全不同的现象 → 拆分为多个 Runbook
- 多篇 Incident 的 `root_cause` 相同但 `symptom.description` 表述不同 → 合并为一个 Runbook，在 symptom-index.yaml 中注册多条路由

### 7.4 输出格式

AI Agent 转换完成后应输出：

```
## Runbook 转换结果

### 生成文件
- [x] runbooks/<runbook-id>.md
- [x] symptom-index.yaml 新增路由条目

### Runbook 概要
- **症状**：<一句话>
- **根因空间**：<N 种可能根因>
- **恢复路径**：<N 条恢复子节>

### 待人工确认
- [ ] <转换过程中 AI 无法确定的决策分支>
- [ ] <引用关系需要人工核实>
- [ ] <命令参数的完整性和正确性>

### 关联 Incident
- <incident_id>：转换中使用了该 Incident 的信息
```

### 7.5 转换后的审批

Runbook 草稿生成后需人工审批，审批要点：

1. 决策树的分支条件是否正确（特别是 AI 推断的分支）
2. 恢复命令是否完整、参数是否正确
3. 跨 Runbook 引用是否正确
4. `[待确认]` 标记项是否需要补充
5. 有无需要脱敏的信息（用户数据、内部 IP 等）

审批通过后，将 Runbook 文件中的 `[待确认]` 标记移除，状态从 `draft` 更新为正式。
