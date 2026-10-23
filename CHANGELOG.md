# Indexable Docs KB Seed Changelog

> 本文件记录种子包的版本变更与兼容性语义。每个版本一个小节，固定子节：问题 / 变更 / 兼容性 / 影响；破坏性变更可另加「迁移」小节。
> 历史版本不在本文件追踪：本仓库自 `0.1.0` 起独立维护。
>
> **版本线说明**：正文与代码注释中出现的 `v1.x` / `v2.0.x` 是种子包并入本仓库**之前**的结构演进编号（用于解释设计沿革，例如「产品视角字段内嵌于 `index.yaml.modules[]`」是在那一代引入的），**不属于本仓库版本线**。本仓库版本以 `VERSION` 文件与 git tag 为准。

## 0.2.4 — 2026-10-23

### 问题
构建 KB 时可能发现目标工程代码/流程本身有问题，但此前没有独立记录通道：这类发现要么混进 `kb-build-qa.md`，要么被构建 Agent 顺手确认/修复，干扰“知识库构建只关注构建本身”的边界；同时 `troubleshooting/known-bugs.md` 是已确认知识，未确认条目不能直接写入。

### 变更
- **新增构建期问题账本**：`pass/build-findings/discovered-issues.jsonl`，由主 Agent 唯一写入；Stage 3 / Stage 4 / Goal 4 sub-agent 通过返回值 `discovered_issues` 上报。
- **新增 `schemas/discovered-issue.schema.json`**：定义 `status: recorded / confirmed / rejected / needs_more_info / duplicate / promoted` 生命周期；构建期只写 `recorded`。
- **构建期硬边界**：不修改目标工程源码/配置/测试；不为确认问题运行构建/测试/复现；不写进 QA、不计入收敛、不在构建期请人类处理。
- **Goal 3 / Goal 4 收官生成人类可读快照**：`pass/build-findings/kb-build-discovered-issues.md`，并在最终 GOAL_MET / 跳过 Goal 4 的路径中提示。
- **新增构建后复核任务文档** `verify-build-findings.md`（**不是 skill**）：独立上下文复核每条发现，允许跑测试/复现；确认后才提升到 `troubleshooting/known-bugs.md`、`troubleshooting/incidents/` 或 `works/`。构建收官时复制到 `pass/build-findings/`。
- **明确 `pass/` 语义**：从“外部输入只读中转”扩展为“构建期产物 + 外部输入中转（不入索引）”；未确认的 build-findings 不作为 KB 知识使用。
- README / README-ZH / DESIGN-planning / explore-kb / KB README 模板同步更新。

### 兼容性
- 不改变 KB 结构、`index.yaml` 或 QA 分诊协议。
- 旧构建若没有 `pass/build-findings/discovered-issues.jsonl`，Goal 3/4 会创建空文件并记录 `build_findings_missing`，不阻塞。
- `troubleshooting/` 仍然只收已确认知识；未确认条目留在 `pass/build-findings/`。

### 影响
- 构建期发现不再污染 QA 收敛与 Goal 4 分诊，也不会被误写进 `known-bugs.md`。
- 人类在构建收官时看到一份集中、带证据、明确标注“未确认”的问题记录，并可选择执行独立复核任务。

## 0.2.3 — 2026-10-23

### 问题
Skill 迁移目标硬编码为 `.claude/skills/`，只有 Claude Code 能发现；pi / OpenCode 等 Agent 读不到同一份 skill。部分工程已把 skill 放到 `.agents/skills/` 并用 `.claude` 软链兼容，但种子包没有把这套做法固化，导致每次构建都要人工改路径。

### 变更
- **Goal 3 Stage 7 的 skill 目标目录改为 `.agents/skills/`**：Agent Skills 中立位置，pi / OpenCode 可直接发现；Claude Code 通过兼容层读取同一份文件。
- **新增 `.claude` 兼容层步骤**：`.claude` 不存在时整目录软链到 `.agents`；`.claude` 已存在时只建立 `.claude/skills -> ../.agents/skills`，保留 `.claude` 下的其他配置；不支持软链的环境允许复制 fallback，并记录 `claude_skills_copied` 与漂移风险。
- 同步更新 Goal 3 / Goal 4 / `maintain-kb` / 两份 README / `DESIGN-planning.md` 中所有 `.claude/skills` 引用。
- 若目标 Agent 只认 `.codex/skills`（部分 Codex 版本），可按同样原则建立 `.codex/skills -> ../.agents/skills` 兼容软链；`.agents/skills/` 仍是唯一真实源。
- **Commander 模板的 `description` 不再内联 modules/flows 清单**：改为「完整清单见 `index.yaml` 的 `modules[]` 与 `flows[]`」，避免占位符展开后超过 1024 字符；`{{MODULE_LIST}}` / `{{FLOW_LIST}}` 仅在旧版模板中作为可选替换项保留。
- **Stage 7 增加 `description` 长度校验**：所有 `SKILL.md` 的 `description` 必须 ≤ 1024 字符（Agent Skills 上限）。DeepInfer 实跑发现 commander 的 description 因占位符展开到 1120 字符，会被客户端直接拒绝加载。
- README 前置条件从「Claude Code 环境」放宽为「支持 `/goal` 的 Agent 环境（如 Claude Code、pi）」。

### 兼容性
- 新建 KB 默认使用 `.agents/skills/`；旧 KB 的 `.claude/skills/` 布局仍可通过 `.claude` 软链或手动迁移继续使用。
- 任务文档中的路径变化只影响 Goal 3 的迁移目标，不改变已生成 KB 的内容、`index.yaml` 结构或 QA 协议。

### 影响
- pi / OpenCode 可直接发现 skill；Claude Code 通过软链读取同一份文件，不再维护第二份可编辑副本。
- 已有 `.claude` 配置的工程不会被覆盖；只有在 `.claude` 不存在或为空时才使用整目录软链。

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
