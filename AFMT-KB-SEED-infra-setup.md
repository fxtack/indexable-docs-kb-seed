# AFMT-KB-SEED · Goal 3: kb-infra-setup

> **执行方式**：本文档由 `/goal` 驱动执行。全部为文件系统操作（复制、创建、写入），主 Agent 可自主完成大部分工作；仅 KB README、postmortem 汇总需分派 sub-agent。
> **前提**：Goal 2 (kb-docs-bake) 必须已完成。
>
> **元认知参考**：本任务文档回答"怎么做"。若需要理解**"为什么这样做"**——如为什么 Goal 3 单列一 Goal、为什么不再有独立的 Commander 内容构建 Stage（产品视角边界字段已由 Stage B 直接产出并写入 index.yaml）、两种 QA 机制的分工——请先读 `.think-tank/afmt-kb-seed/DESIGN-planning.md`（尤其 §5）。

---

## Goal 定义

**goal_id**: `kb-infra-setup`
**goal_name**: KB 运行基础设施搭建 + Skill 迁移
**seed_pack_version**: `<从 .think-tank/afmt-kb-seed/VERSION 读取>`
**depends_on_goal**: `kb-docs-bake`

---

## Goal 条件（GOAL_MET）

以下所有必选项完整存在时达成：

**元规范文件**：
- `.think-tank/knowledge-base/.meta-knowledge-base-guide.md`
- `.think-tank/knowledge-base/.meta-maintain-beacon.yaml`
- `.think-tank/knowledge-base/.meta-explore-issues.jsonl`
- `.think-tank/knowledge-base/index.yaml`（含 modules[].{responsibility, capability_boundary, communicates_with} 产品视角字段）
- `.think-tank/knowledge-base/README.md`
- `.think-tank/knowledge-base/knowledge-extract-qa-guide.md`
- `.think-tank/knowledge-base/works/.meta-work-guide.md`
- `.think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md`
- `.think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md`

**Skill**：
- `.claude/skills/explore-kb/SKILL.md`（必装）
- `.claude/skills/maintain-kb/SKILL.md`（必装）
- `.claude/skills/{{PROJECT_SLUG}}-commander/SKILL.md`（**必装**——种子包内含 Commander skill 模板，Stage 7 自动执行模板化迁移。这是 KB 产品级决策入口，Goal 4 kb-qa-triage 也依赖 Commander 辅助分诊）

**目录**：
- `pass/` / `works/` / `develop/` / `troubleshooting/incidents/`

**收尾产物**：
- `pass/kb-seed-postmortem.md`
- `pass/kb-seed-summary.md`

达成后主 Agent 输出：

```
GOAL_MET: kb-infra-setup 完成。
  基础设施就绪 / Skill 已安装（explore-kb, maintain-kb, {{PROJECT_SLUG}}-commander）
  产品视角边界字段：<N> 个模块（Goal 2 Stage B 已产出并写入 index.yaml.modules[]）
  Postmortem: pass/kb-seed-postmortem.md
  Summary: pass/kb-seed-summary.md
  KB 基础设施已就绪，但 AFMT-KB-SEED **完整构建周期尚未收官**。

  **下一步：启动 Goal 4 (kb-qa-triage) 完成 QA 全局分诊**
    /goal .think-tank/afmt-kb-seed/AFMT-KB-SEED-qa-triage.md

  Goal 4 会用 KB 全局视角尝试回答 kb-build-qa.md 中的每个 QA 问题，把人类必须回答的问题精准圈出来（通常能减少 30-60% 的人类回答负担）。
  除非人类显式跳过或 explore-kb 未安装，主 Agent 应默认启动 Goal 4。跳过路径见 AFMT-KB-SEED-qa-triage.md §与人类回答阶段的关系。
```

---

## 0. 前置检查

主 Agent 执行：

- 验证 `.think-tank/knowledge-base/index.yaml` 存在（Goal 1 产出）
- 验证 `index.yaml.modules[]` 中至少 1 项含 `responsibility` 字段（Goal 1 Stage B 应已产出产品视角字段）
- 验证 `.think-tank/knowledge-base/pass/kb-build-qa.md` 存在（Goal 2 产出）
- 验证 `.think-tank/afmt-kb-seed/skills/` 目录存在（种子包应含 explore-kb / maintain-kb / commander 三个 skill）
- 验证 `.think-tank/afmt-kb-seed/skills/commander/SKILL.md` 存在（种子包应含 Commander skill 模板。若缺失，说明种子包不完整，应重新复制种子包）
- 读取 `.think-tank/afmt-kb-seed/VERSION` 获得 `seed_pack_version`
- 读取 `index.yaml.meta.project_name` 获得 `{{PROJECT_NAME}}`
- 从 `{{PROJECT_NAME}}` 派生 `{{PROJECT_SLUG}}`：小写化 + 特殊字符替换为连字符，如 `Redis` → `redis`

任一必选缺失即终止：`GOAL_ABANDONED: 前置条件不满足，缺失 {file}`。

---

## Stage 6: 目录与基础设施文件创建

### 6.1 补齐目录

主 Agent 直接执行（idempotent，若已存在则跳过）：

```
mkdir -p .think-tank/knowledge-base/pass
mkdir -p .think-tank/knowledge-base/works
mkdir -p .think-tank/knowledge-base/develop
mkdir -p .think-tank/knowledge-base/troubleshooting/incidents
mkdir -p .claude/skills
```

**目录约定**：不创建 `.think-tank/commander/` 目录（产品视角边界已内嵌于 `knowledge-base/index.yaml`）；不创建 `knowledge-base/decisions/` 目录（设计权衡通过 `works/` 或 git history 追溯）。

### 6.2 验证 Goal 1 已创建的配置文件

- `.think-tank/knowledge-base/.meta-maintain-beacon.yaml`：验证存在。若丢失则重建：`commit = git rev-parse HEAD`
- `.think-tank/knowledge-base/.meta-explore-issues.jsonl`：验证存在。若丢失则重建为空文件
- `.think-tank/knowledge-base/index.yaml`：验证 modules[] 中至少一项含 `responsibility` 字段（必填，Goal 1 Stage B 应已产出）

### 6.3 生成 KB README（分派 1 个 sub-agent）

分派 sub-agent，prompt：

```
为 {{PROJECT_NAME}} 工程的 .think-tank/knowledge-base/ 目录编写 README.md。

输入：
- .think-tank/knowledge-base/index.yaml（了解 KB 全貌 + 产品视角边界字段）
- .think-tank/knowledge-base/.meta-knowledge-base-guide.md（了解构建规范）
- CLAUDE.md（了解工程背景）
- 模板骨架：.think-tank/afmt-kb-seed/templates/kb-readme.md.template

输出 .think-tank/knowledge-base/README.md，需包含：
1. 本知识库的定位与使用场景（1-2 段）
2. 目录结构速览（表格）
3. 快速检索指南（如何通过 index.yaml 找文档；index.yaml.modules[] 已内嵌产品视角边界）
4. 文档类型说明（architecture / modules / flows / troubleshooting / develop / works）
5. 维护方式（maintain-kb skill 简介与调用方式）
6. Commander skill 使用简介（产品级决策入口）
7. 贡献指南（如何新增/修改知识文档）

约束：
- 面向 AI Agent 和人类开发者
- 控制在 100 行以内
- 使用中文
- 不重复 .meta-knowledge-base-guide.md 的详细规范，仅做导航级概述
- 用符号名锚定代码，不用行号
```

### 6.4 Stage 6 收尾

- 验证目录 4 项 + KB README 1 项存在
- 更新 Progress Log：`Stage 6: ✅`

---

## Stage 7: Skill 迁移

### 7.1 复制 explore-kb / maintain-kb（去耦合版）

主 Agent 直接执行（纯文件复制）：

```bash
cp -r .think-tank/afmt-kb-seed/skills/explore-kb .claude/skills/
cp -r .think-tank/afmt-kb-seed/skills/maintain-kb .claude/skills/
```

### 7.2 复制 knowledge-extract-qa-guide

```bash
cp .think-tank/afmt-kb-seed/knowledge-extract-qa-guide.md \
   .think-tank/knowledge-base/knowledge-extract-qa-guide.md
```

### 7.3 迁移 Commander skill（模板化复制）

主 Agent 直接执行（Commander 为必装 skill——种子包内含其模板。模板已去 module_header 化，引用路径指向 `index.yaml.modules[]` 产品边界字段。模板分两层：**模板层**——`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` / `{{MODULE_LIST}}` / `{{FLOW_LIST}}` 四个占位符，迁移时必须替换；**示例层**——取自 Redis 8.10.2 的教学示例，文件头标有「示例说明」，迁移时**不需要**改写）：

**步骤 1：复制 Commander skill 到目标位置**

```bash
cp -r .think-tank/afmt-kb-seed/skills/commander \
      .claude/skills/{{PROJECT_SLUG}}-commander
```

**步骤 2：替换模板占位符**

主 Agent 读取 `index.yaml.meta.project_name` 获得 `{{PROJECT_NAME}}`（如 `MyProject`），从 `{{PROJECT_NAME}}` 派生 `{{PROJECT_SLUG}}`（小写化 + 特殊字符替换为连字符，如 `myproject`）；再从 `index.yaml.modules[].id` 逗号连接得到 `{{MODULE_LIST}}`、从 `index.yaml.flows[].id` 逗号连接得到 `{{FLOW_LIST}}`。然后对 `.claude/skills/{{PROJECT_SLUG}}-commander/` 下的所有 `.md` 文件执行字符串替换：

| 替换 | 源 | 目标 |
|------|---|------|
| `{{PROJECT_NAME}}` | 模板层占位符 | `index.yaml.meta.project_name` 的值 |
| `{{PROJECT_SLUG}}` | 模板层占位符 | 从 project_name 派生的 slug |
| `{{MODULE_LIST}}` | 模板层占位符 | `index.yaml.modules[].id` 逗号连接 |
| `{{FLOW_LIST}}` | 模板层占位符 | `index.yaml.flows[].id` 逗号连接 |

（`{{PROJECT_SLUG}}-commander` 形式的 skill 名由 `{{PROJECT_SLUG}}` 替换一并覆盖，无需单独一行。）

**步骤 3：区分模板层与示例层**

Commander 模板分两层，迁移时只处理模板层，**不逐个重写示例**：

1. **模板层**：确认 4 个占位符已替换完毕——`grep -rc '{{' .claude/skills/{{PROJECT_SLUG}}-commander/` 各文件计数应全为 0
2. **示例层**：确认文件头的「示例说明」存在（`SKILL.md` 与 `references/` 下 5 个文件各 1 行），该行声明示例仅作教学用途、不属于模板变量
3. **去示例化（仅当人类显式要求）**：可**整目录删除** `references/examples/`（模板未含该目录时，手工删除各文件中的示例小节）。**禁止逐句改写示例内容**——示例已标明是示例，而不是待填模板

**步骤 4：目录重命名**

主 Agent 将目录从 `skills/commander/` 重命名为 `skills/{{PROJECT_SLUG}}-commander/` 后，确认 SKILL.md frontmatter 中 `name:` 字段已是 `{{PROJECT_SLUG}}-commander`。

**步骤 5：验证**

- 验证 `.claude/skills/{{PROJECT_SLUG}}-commander/SKILL.md` 存在
- 验证占位符残留 0：`grep -rc '{{' .claude/skills/{{PROJECT_SLUG}}-commander/` 各文件计数均为 0（`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` / `{{MODULE_LIST}}` / `{{FLOW_LIST}}` 全部替换完毕）
- 验证 SKILL.md description 中 modules / flows 列表已替换为本工程实际模块与流程（不再含示例工程以外的工程名；示例工程名子串允许存在，例如 `Redis`）
- 验证各文件头部的「示例说明」行仍在（示例层未被误删或逐句改写）

### 7.4 Skill 说明

**explore-kb**：知识库探索检索。用户调用 `/explore-kb <question>` 时激活，通过 `index.yaml` 三级降级检索。已去耦合（零产品名引用）。

**maintain-kb**：知识库一致性维护。基于 `.meta-maintain-beacon.yaml` 和 `.meta-explore-issues.jsonl` 消费信箱，做增量或全量一致性检查与修复。已去耦合。

**{{PROJECT_SLUG}}-commander**：产品级决策与咨询。用户涉及跨模块决策、架构评审、需求拆解、问题排障、运维决策时激活。依赖 `index.yaml.modules[]` 中已内嵌的产品视角边界字段（`responsibility` / `capability_boundary` / `communicates_with`），不再需要独立的 module_header 文件——KB 成为单一事实来源。

**两种 QA 机制的区分**（详见 planning §5.4）：

| | Build-time QA（Goal 2） | Post-build QA（knowledge-extract-qa-guide） |
|------|--------------------------|---------------------------------------------|
| **阶段** | KB 构建过程中 | KB 构建完成后 |
| **问题来源** | Sub-agent 读源码发现的代码歧义 | 阅读完整 KB 后识别的知识盲区（测试策略/性能基准/事故经验等） |
| **产出** | `pass/{doc_id}.qa.jsonl` → `pass/kb-build-qa.md` | 各层 `qa.md`（Markdown） |
| **回答者** | 人类开发者（代码歧义） | 人类团队（隐性知识） |

### 7.5 Stage 7 收尾

- 验证 explore-kb / maintain-kb / commander 三个 skill 均已就位
- 验证 knowledge-extract-qa-guide 已迁移
- 更新 Progress Log：`Stage 7: ✅`

---

## Stage 8: 完整性检查 + 收尾

### 8.1 完整性检查清单

主 Agent 对照清单逐项验证。任一必选项缺失即失败，触发 GOAL_ABANDONED。

**必选文件**：

```
[ ] .think-tank/knowledge-base/.meta-knowledge-base-guide.md
[ ] .think-tank/knowledge-base/.meta-maintain-beacon.yaml
[ ] .think-tank/knowledge-base/.meta-explore-issues.jsonl
[ ] .think-tank/knowledge-base/index.yaml（含 meta.excluded_paths + modules[].responsibility 等产品视角字段）
[ ] .think-tank/knowledge-base/README.md
[ ] .think-tank/knowledge-base/knowledge-extract-qa-guide.md
[ ] .think-tank/knowledge-base/works/.meta-work-guide.md
[ ] .think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md
[ ] .think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md
[ ] .claude/skills/explore-kb/SKILL.md
[ ] .claude/skills/maintain-kb/SKILL.md
[ ] .claude/skills/{{PROJECT_SLUG}}-commander/SKILL.md（必装）
[ ] .claude/skills/{{PROJECT_SLUG}}-commander/references/{workflow,answer-strategy,routing-rules,context-budget,knowledge-writeback}.md
```

**必选目录**：

```
[ ] .think-tank/knowledge-base/pass/
[ ] .think-tank/knowledge-base/works/
[ ] .think-tank/knowledge-base/develop/
[ ] .think-tank/knowledge-base/troubleshooting/incidents/
[ ] .think-tank/knowledge-base/troubleshooting/runbooks/
[ ] .think-tank/knowledge-base/architecture/
[ ] .think-tank/knowledge-base/modules/
[ ] .think-tank/knowledge-base/flows/
```

**内容一致性检查**：
- `index.yaml.meta.excluded_paths` 存在（即使为空数组也需存在）
- `modules[].covers_code` 中的所有路径都不落在 `excluded_paths` 下
- **产品字段覆盖**：`index.yaml.modules[]` 中含 `responsibility` 字段的条目 ≥ 全部 modules 数量的 50%（Goal 1 Stage B 应已上浮大部分模块的产品视角字段。少于 50% 说明 Stage B 有质量问题，记入 postmortem 但不阻塞 Goal 完成）

### 8.2 生成 postmortem（分派 1 个 sub-agent）

分派 sub-agent，prompt：

```
汇总 AFMT-KB-SEED 本次构建的完整过程记录。

输入：
- .think-tank/knowledge-base/pass/kb-seed-journal-kb-primer.md
- .think-tank/knowledge-base/pass/kb-seed-journal-kb-docs-bake.md
- .think-tank/knowledge-base/pass/kb-seed-journal-kb-infra-setup.md
- .think-tank/knowledge-base/pass/kb-build-qa.md（若存在）
- .think-tank/knowledge-base/index.yaml（用于统计产品视角字段覆盖率）

输出：.think-tank/knowledge-base/pass/kb-seed-postmortem.md

包含章节：
## 1. 执行概览
  - 各 Goal 起止时间 / 耗时 / 状态
  - Sub-agent 总数 / 失败数 / 重试率
  - 硬收敛还是软收敛
  - 产品视角字段覆盖率（modules 中含 responsibility 的数量 / 总数）

## 2. 异常统计
  - 按 error_type 分组：agent_timeout / api_error / invalid_output / file_write_error
  - 每类的次数、涉及的 sub-agent、常见触发原因

## 3. 关键决策
  - 从 Journal 中 action=decision 的条目
  - 例：cycle_broken / soft_convergence / batch_reorder / excluded_paths_finalized

## 4. 反复出现的问题
  - Review 多轮中重复出现的问题模式（对应文档 / 问题类型）
  - Stage B 产出产品视角字段时反复出现的困难（如某类模块的 responsibility 难以从技术描述剥离）

## 5. 流程优化建议
  - 基于本次构建观察，给种子包维护者的改进建议
```

### 8.3 生成 summary（主 Agent 直接执行）

主 Agent 生成 `.think-tank/knowledge-base/pass/kb-seed-summary.md`：

```markdown
# AFMT-KB-SEED 构建完成摘要 · {{project_name}}

**构建时间**：{start_time} → {end_time}（耗时 {duration}）
**种子包版本**：{seed_pack_version}
**工程规模**：{project_scale}

## Goal 完成状态
| Goal | 状态 | 耗时 |
|------|------|------|
| kb-primer | ✅ | {duration_1} |
| kb-docs-bake | ✅ / ⚠️ soft_convergence | {duration_2} |
| kb-infra-setup | ✅ | {duration_3} |

## 产出统计
- 架构文档：{N_A} 份
- 模块文档：{N_M} 份（其中 {N_M_HEADER} 份含产品视角边界字段）
- 流程文档：{N_F} 份
- 故障文档：symptom-index / faq / known-bugs 骨架就绪
- Skill：explore-kb / maintain-kb / {{PROJECT_SLUG}}-commander 已安装

## QA 问题统计
- Critical: open={O_c} / resolved={R_c}
- Medium: open={O_m} / resolved={R_m}
- Minor: open={O_min} / resolved={R_min}

## 下一步行动

**AFMT-KB-SEED 完整构建周期是 Goal 1 → Goal 2 → Goal 3 → Goal 4，本 Goal 完成后应默认继续 Goal 4**：

1. **[标准流程] 启动 Goal 4 (kb-qa-triage)**：`/goal .think-tank/afmt-kb-seed/AFMT-KB-SEED-qa-triage.md`
   - Goal 4 用 KB 全局视角尝试回答 kb-build-qa.md 中每个 QA 问题，输出 4 档分诊 + KB patch 建议
   - 通常能减少 30-60% 的人类回答负担
2. Goal 4 完成后：人类 review `pass/kb-doc-patches.md` 决定是否 apply；然后回答 partially_answered + cannot_answer_needs_human 项
3. Agent 根据人类回答更新对应知识文档，并在 kb-build-qa.md 中标记 resolved
4. （可选）阅读 knowledge-extract-qa-guide.md，启动 post-build QA 补充隐性知识
5. 若 index.yaml.modules[] 有模块缺 responsibility 字段，可运行 /maintain-kb 上浮补齐
6. 后续代码变更时使用 /maintain-kb 增量维护 KB 一致性

**跳过 Goal 4 的场景**（罕见，仅在下列情况允许）：
- 人类显式要求跳过
- explore-kb skill 未安装（依赖失败）
- Goal 4 累计触发 GOAL_ABANDONED
详见 `AFMT-KB-SEED-qa-triage.md §与人类回答阶段的关系`。
```

### 8.4 Stage 8 收尾

- 验证 postmortem 和 summary 存在
- 更新 Progress Log：`Stage 8: ✅`
- Journal 追加 `action: goal_completed`
- **输出 GOAL_MET 后立即启动 Goal 4**（除非满足 §Goal 条件中列出的降级出口）——主 Agent 不应在此处停顿等待人类确认。Goal 3 与 Goal 4 之间没有需要人类 review 的产物，无需询问

---

## Progress Log

| Stage | 状态 | 开始时间 | 完成时间 | 耗时 | 备注 |
|-------|------|---------|---------|------|------|
| 6. 目录与基础设施 | ⬜ | | | | |
| 7. Skill 迁移 | ⬜ | | | | |
| 8. 完整性检查 + 收尾 | ⬜ | | | | |

**Goal 状态**：⬜ pending

---

## 恢复协议

见 `AFMT-KB-SEED-primer.md` 的恢复协议章节。Goal 3 的 Journal 路径：`pass/kb-seed-journal-kb-infra-setup.md`。

由于 Goal 3 大部分是幂等文件操作，恢复时可直接重跑各 Stage（跳过已完成项由 mkdir/cp 的 idempotent 特性保证）。

---

## 结构与兼容性说明

- **Stage 编号**：本 Goal 只有 Stage 6 / 7 / 8 三个。原「Commander 内容构建」Stage 已移除（产品视角字段由 Goal 1 Stage B 直接产出并写入 `index.yaml.modules[]`），原「完整性检查 + 收尾」因此重编号为 Stage 8。
- **两条版本线**：种子包版本见 `VERSION`；KB 结构版本见 `index.yaml.meta.kb_schema_version`（当前 `2.0`，表示 `modules[]` 含产品视角字段）。两者独立演进。
- **旧 KB 兼容性**：按 `kb_schema_version: 1.x` 构建的 KB 需要迁移（`/maintain-kb --migrate-to-v2`，尚未实现）；用本种子包新建的 KB 直接是 2.0 结构。
