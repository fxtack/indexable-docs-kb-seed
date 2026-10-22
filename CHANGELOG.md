# Indexable Docs KB Seed Changelog

> 本文件记录种子包的版本变更与兼容性语义。每个版本一个小节，固定子节：问题 / 变更 / 兼容性 / 影响；破坏性变更可另加「迁移」小节。
> 历史版本不在本文件追踪：本仓库自 `0.1.0` 起独立维护。
>
> **版本线说明**：正文与代码注释中出现的 `v1.x` / `v2.0.x` 是种子包并入本仓库**之前**的结构演进编号（用于解释设计沿革，例如「产品视角字段内嵌于 `index.yaml.modules[]`」是在那一代引入的），**不属于本仓库版本线**。本仓库版本以 `VERSION` 文件与 git tag 为准。

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
- **概念表述**：`AFMT` 保留为"**默认四维**"的简称，并明确其为默认组织方式、可在 Goal 1 Stage B 规划阶段增删维度。

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
