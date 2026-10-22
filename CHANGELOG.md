# Indexable Docs KB Seed Changelog

> 本文件记录种子包的版本变更与兼容性语义。每个版本一个小节，固定子节：问题 / 变更 / 兼容性 / 影响；破坏性变更可另加「迁移」小节。
> 历史版本不在本文件追踪：本仓库自 `0.1.0` 起独立维护。
>
> **版本线说明**：正文与代码注释中出现的 `v1.x` / `v2.0.x` 是种子包并入本仓库**之前**的结构演进编号（用于解释设计沿革，例如「产品视角字段内嵌于 `index.yaml.modules[]`」是在那一代引入的），**不属于本仓库版本线**。本仓库版本以 `VERSION` 文件与 git tag 为准。

## 0.2.2 — 2026-10-21

### 问题
种子包只有中文 README，对非中文读者与以英文为主的 Agent 生态不友好；而 README 是唯一面向使用者的入口文档。

### 变更
- **新增英文 `README.md`（默认版本）**：与中文版全文对应，含维度演进说明、两类知识来源、文件清单、示例层/模板层边界、四个 Goal 速览、版本契约、迁移指引。
- **新增构建流水线流程图**（Mermaid）：Goal 1 → 人工 review → Goal 2 → Goal 3 → Goal 4 → 人工回答，虚线标注产物交接。英文版与中文版各用对应语言的标签，结构保持一致（两份文件内的图各有一条同步提示注释）。
- **中文版迁移为 `README-ZH.md`**：内容与 0.2.1 一致，仅增加语言切换链接。
- 两份 README 顶部互加语言切换（`English` / `简体中文`）。
- `DESIGN-planning.md` §10 对 README 的引用改为同时指向两份文件。

### 兼容性
- 无。文件路径引用不变（`README.md` 仍存在，只是语言变了）；无协议、schema、任务流程变更。

### 影响
- 默认阅读入口 `README.md` 现为英文；中文读者请走 `README-ZH.md`。

## 0.2.1 — 2026-10-21

### 问题
0.2.0 的 README 把「维度可演进」表述为「增删维度属协议变更，须同步调整 Stage B 分派、schema、bake 模板与 `GOAL_MET` 判定」。这句话在事实层没错，但读起来是在说「协议不支持扩展，别指望」，与实际设计意图相反——维度集合确实会演进，只是扩展动作为**更新种子包**，而非使用方在运行期开启的开关。

### 变更
- **README 概念段重写**，分三层表达：
  1. 当前协议按四维实现（Stage B 四个 sub-agent、`index.yaml` 四段、schema `dimension` 固定枚举）；
  2. 「可演进」指种子包层的演进——新增维度 = 更新任务/模板/schema，必要时补一个 Stage；
  3. 区分**两类知识来源**：从源码推导（AFMT 四维，可自动化）与构建后收纳（`works/` 一类；未来的会议纪要维度属此类，代码里读不出来，靠流程写入）。
- 明确「包名不绑定维度，正是为了让这类演进不产生改名成本」，与 0.2.0 的改名动机对齐。
- 同步修正本文件 0.2.0 条目中的概念表述。

### 兼容性
- 无。纯文档变更，协议、schema、任务流程与目录结构均不变。

### 影响
- 使用方式不变；仅 README 与 CHANGELOG 的表述更准确。

## 0.2.0 — 2026-10-21

### 问题
包名 `AFMT-KB-SEED` 把知识库维度缩写（Architecture / Flows / Modules / Troubleshooting）焊进了名称。但随着工程类型、知识类型与结构化保存需求的变化，维度已被大范围拓宽，用固定的四维缩写指代整个种子包不再准确；且 GitHub 仓库名早已是 `indexable-docs-kb-seed`，包名与之不一致。

### 变更（**破坏性**）
- **包名**：`AFMT-KB-SEED` → `Indexable Docs KB Seed`
- **目录名**：`.think-tank/afmt-kb-seed/` → `.think-tank/indexable-docs-kb-seed/`
- **任务文档文件名**（去掉重复的包名前缀，路径已含包名）：
  - `AFMT-KB-SEED-primer.md` → `primer.md`
  - `AFMT-KB-SEED-bake.md.template` → `bake.md.template`（实例化产物 `bake-{project}.md`）
  - `AFMT-KB-SEED-infra-setup.md` → `infra-setup.md`
  - `AFMT-KB-SEED-qa-triage.md` → `qa-triage.md`
- **schema `$id`**：`urn:afmt-kb-seed:schema:*` → `urn:indexable-docs-kb:schema:*`
- **概念表述**：`AFMT` 保留为"**默认四维**"的简称。明确「维度集合可演进」指的是**种子包层的演进**（新增维度 = 更新任务/模板/schema/Stage），而非运行时自定义；并区分两类知识来源——从源码推导（AFMT 四维）与构建后收纳（`works/` 一类，未来的会议纪要等维度属此类）。

### 兼容性
- **对已构建的 KB**：内容与结构不受影响；但种子包目录名与任务文档文件名变化，**旧路径将失效**。
- **对协议**：Goal 1-4 的流程、`goal_id`、schema 字段均未变化，`index.yaml` 结构不变。
- **历史记录**：首次提交（`5fa4f1f`）与 `v0.1.0` tag 的信息中保留旧名，不做历史重写。

### 迁移（旧包 → 新包）

```bash
# 目标工程内，重命名种子包目录
mv .think-tank/afmt-kb-seed .think-tank/indexable-docs-kb-seed
# 若已实例化 Goal 2 任务文档，同步改名
mv .think-tank/indexable-docs-kb-seed/AFMT-KB-SEED-bake-<project>.md \
   .think-tank/indexable-docs-kb-seed/bake-<project>.md
```

旧 KB 的 `index.yaml.meta.seed_pack_version` 保留原值（记录构建时用的版本）；改写它会丢失溯源信息，且 `/maintain-kb` 只做 KB↔代码一致性，不会改写该字段。升级包版本请人工修改该字段并自行承担溯源损失。

### 影响
- 新工程的接入命令：`.think-tank/indexable-docs-kb-seed/primer.md`
- 本仓库自身的引用已全量更新；`skills/` 未受影响（名称不含旧包名）。

## 0.1.0 — 2026-10-21

### 基线
- 本仓库以 `0.1.0` 为首个版本，历史变更不在本文件追踪（结构沿革见文首说明）。
- 种子包版本契约：`VERSION` 为唯一版本源，发布时同步打 git tag。

### 变更
- 仓库结构定型：Goal 1-4 任务文档 + `prompts/` + `schemas/` + `templates/` + `skills/`。
- **示例层与模板层分离**：模板层只含 `{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` / `{{MODULE_LIST}}` / `{{FLOW_LIST}}` 占位符；示例层取自 Redis 8.10.2（公共领域工程）。
- Goal 3 Stage 7 的 Commander 迁移改为“占位符替换 + 示例可整体替换”两步。

### 兼容性
- 无接口变化；对已构建的 KB 无影响。

### 影响
- 消费者用法不变：复制种子包到目标工程 `.think-tank/indexable-docs-kb-seed/` 后按 Goal 1 → 4 执行。
