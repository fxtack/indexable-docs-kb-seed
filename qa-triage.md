# Indexable Docs KB Seed · Goal 4: kb-qa-triage

> **执行方式**：本文档由 `/goal` 驱动执行。核心动作是**批并行**地对 kb-build-qa.md 中的每个 QA 问题分派一个隔离的 sub-agent，用 explore-kb 尝试回答，最终把 kb-build-qa.md 重组为 4 档分诊结果。
> **前提**：Goal 3 (kb-infra-setup) 必须已完成——explore-kb / commander skill 都已安装，KB 内容完整可检索。
> **位置**：Goal 4 是种子包完整构建周期的**最后一环**（Goal 1 → Goal 2 → Goal 3 → **Goal 4**）。默认执行——Goal 3 GOAL_MET 之后主 Agent 应直接启动本 Goal。仅当人类**显式**要求跳过、或本 Goal 触发出工条件时才允许绕过（详见 §出工条件（GOAL_ABANDONED））。
>
> **元认知参考**：本任务文档回答"怎么做"。若需要理解**"为什么要做 Goal 4"**——jsonl 生成阶段是局部视角、Goal 3 完成后 KB 已全局互联、此时值得做一次全局回答尝试来减少人类回答负担——请先读 `.think-tank/indexable-docs-kb-seed/DESIGN-planning.md` §6。

---

## Goal 定义

**goal_id**: `kb-qa-triage`
**goal_name**: 全局视角 QA 分诊，把 kb-build-qa 分为 4 档
**seed_pack_version**: `<从 .think-tank/indexable-docs-kb-seed/VERSION 读取>`
**depends_on_goal**: `kb-infra-setup`
**position**: `KB 构建周期的最后一环（Goal 1-4 的第 4 步）`
**default_behavior**: `Goal 3 完成后主 Agent 默认启动本 Goal，不询问人类是否执行；仅当人类显式要求跳过时才绕过`

---

## Goal 条件（GOAL_MET）

以下条件同时满足即达成：

1. `pass/kb-build-qa-pre-goal4.md` 已备份（Goal 2 Stage 5 的原始版本）
2. `pass/kb-build-qa.md` 已重组为 4 档结构（含 fully_answered / partially_answered / cannot_answer_needs_human / should_not_be_a_question 分组）
3. `pass/kb-doc-patches.md` 已生成（含所有 sub-agent 产出的 KB 修改建议，若无建议则为空文件带说明头）
4. 所有 kb-build-qa 中的 QA 条目均已被 sub-agent 分诊（含 abandoned 项——重试上限失败的问题也需明确标注）
5. 本文档 Progress Log 中所有 Stage 状态为 ✅

达成后主 Agent 输出：

```
GOAL_MET: kb-qa-triage 完成。种子包完整构建周期 (Goal 1-4) 已收官。
  原始 QA 问题：{N} 项
  分诊结果：
    fully_answered           : {A} 项（KB 已能回答，可选审阅）
    partially_answered       : {B} 项（部分覆盖，人类补充缺口即可）
    cannot_answer_needs_human: {C} 项（必须人类回答）
    should_not_be_a_question : {D} 项（噪音，已剔除）
  KB patch 建议：{P} 项（详见 pass/kb-doc-patches.md）
  人类回答负担从 {N} 项减少到 {B+C} 项（减少 {(A+D)/N * 100}%）

  KB 构建流程到此结束，接下来是人类回答环节：
  1. review pass/kb-doc-patches.md 决定是否 apply（或运行 /maintain-kb --apply-patches）
  2. 回答 partially_answered + cannot_answer_needs_human 项，Agent 根据回答更新对应 KB 文档
  3. （可选）阅读 knowledge-base/knowledge-extract-qa-guide.md，启动 post-build QA 补充隐性知识
```

---

## 出工条件（GOAL_ABANDONED）

以下任一条件触发即降级出工（**注意**：这是**降级出口**，不是常规路径。Goal 4 是默认标准流程，只有在真的无法推进时才走此分支）：

- 累计轮次 > 30
- Sub-agent 累计失败率 > 30% 且重试无进展
- 关键前置文件缺失（kb-build-qa.md 不存在、explore-kb skill 未安装）
- 人类在启动 Goal 4 后**显式**要求跳过或终止（例如 QA 问题总数极少不值得走全流程）

终止时主 Agent 输出：

```
GOAL_ABANDONED: {原因}
  Goal 4 未完成。可基于 pass/kb-build-qa-pre-goal4.md 直接进入人工回答阶段（该文件是 Stage 10.1 的原始 kb-build-qa.md 备份，若 Stage 10.1 未执行则直接用 pass/kb-build-qa.md）。
  若因资源不足暂时降级，可在后续任意时刻重新 /goal .think-tank/indexable-docs-kb-seed/qa-triage.md 重启 Goal 4。
```

---

## 0. 前置检查

主 Agent 执行：

- 验证 `pass/kb-build-qa.md` 存在（Goal 2 Stage 5 产出）
- 验证 `.claude/skills/explore-kb/SKILL.md` 存在（Goal 3 Stage 7 迁移）
- 验证 `.claude/skills/{{PROJECT_SLUG}}-commander/SKILL.md` 存在（可选，若装了 commander 可辅助分诊）
- 读取 `.think-tank/indexable-docs-kb-seed/VERSION` 获得 `seed_pack_version`

任一必选缺失即终止：`GOAL_ABANDONED: 前置条件不满足，缺失 {file}`。

---

## Stage 10: 备份 + 解析 QA 问题清单

### 10.1 备份 kb-build-qa.md

```bash
cp .think-tank/knowledge-base/pass/kb-build-qa.md \
   .think-tank/knowledge-base/pass/kb-build-qa-pre-goal4.md
```

**用途**：留作审计。若 Goal 4 判定有争议，人类可对照 pre-goal4 版本 review。

### 10.2 解析 kb-build-qa.md 提取问题清单

主 Agent 直接执行（不需 sub-agent）：

- 读 `pass/kb-build-qa.md`
- 解析 Critical / Medium / Minor 三个章节下的所有 QA 条目
- 为每个条目分配唯一 `entry_id`（若原文档已有则沿用；若无则用 `<doc_id>-q<index>` 生成）
- 提取字段：`entry_id`, `question`, `location`, `severity`, `type`, `code_reference`（若有）

**产出**：主 Agent 会话内存中的 QA 问题清单，供 Stage 11 分派。

### 10.3 Stage 10 收尾

- 验证备份文件存在
- 记录 QA 问题总数到 Journal
- 更新 Progress Log：`Stage 10: ✅`

---

## Stage 11: 分诊 sub-agent 批并行分派

### 11.1 分批策略

**并发上限**：每批 6 个 sub-agent。选择 6 而不是 Goal 2 的 8 的原因——每个 sub-agent 会多次调用 explore-kb（也会拉起 sub-agent），实际资源占用更高。

**保留上下文隔离**：每个 sub-agent fresh context 分派，天然与其他 sub-agent 隔离。你要的"回答结束销毁"效果由 sub-agent 天然生命周期保证。

**分批推进**：

```
batches = chunk(qa_entries, size=6)
for batch in batches:
  Journal 追加 dispatch 条目（含批次编号 + 每个 sub-agent 的 entry_id）
  spawn_sub_agents_parallel(batch)  # fresh context each
  wait_all()
  收集每个 sub-agent 的输出（按 schemas/qa-triage-output.schema.json 校验）
  写入主 Agent 会话内存的 triage_results 累积清单
  Journal 追加 batch_completed 条目（含本批 verdict 分布统计）
```

### 11.2 每个 sub-agent 的 prompt 参数

从 `qa_entries[i]` 提取后填入：

- `QA_ENTRY_ID`：条目唯一 id
- `QA_QUESTION`：问题原文
- `QA_LOCATION`：问题所在文档章节
- `QA_SEVERITY`：critical / medium / minor
- `QA_TYPE`：问题类型
- `QA_CODE_REFERENCE`：关联代码符号（若有）
- `EXPLORE_KB_AVAILABLE`：`true`

Prompt 模板：`.think-tank/indexable-docs-kb-seed/prompts/stage-qa-triage-subagent.md`

### 11.3 失败处理

- Sub-agent 失败（超时 / 无效输出 / API 错误）：主 Agent 重试最多 2 次
- 2 次仍失败：标记该 entry 为 `abandoned`，在 kb-build-qa.md 中归入 `cannot_answer_needs_human` 分组（保守：无法分诊的问题默认交给人类）
- Sub-agent 返回 `confidence: low` 且 `verdict: fully_answered`：见 §11.4 交叉验证

### 11.4 交叉验证（针对 fully_answered + low confidence 的抽检）

**为什么需要**：sub-agent 可能"过度自信"——把本该问人类的问题报成 fully_answered，让人类漏掉真正需要回答的问题。

**策略**：
- 收集所有 `verdict == "fully_answered"` 的 entry
- 若 `confidence == "low"`，**必须**做交叉验证：抽样重跑一个 fresh sub-agent，若两次判定不一致，降级为 `partially_answered`（保守）
- `confidence == "medium"`：**可选**做交叉验证（若成本允许）
- `confidence == "high"`：不做交叉验证

**交叉验证由主 Agent 触发**：对需验证的 entry，追加一个新的 sub-agent 分派（当前批次内即可），Journal 记录 `{action: cross_verify, entry_id: <id>, first_verdict: fully_answered, second_verdict: <...>}`。

### 11.5 Stage 11 收尾

- 验证所有 QA 条目都有分诊结果（含 abandoned）
- 更新 Progress Log：`Stage 11: ✅`（含每档统计）

---

## Stage 12: 重组 kb-build-qa.md + 生成 kb-doc-patches.md

主 Agent 直接执行（不需 sub-agent）。

### 12.1 重组 kb-build-qa.md

按 sub-agent 返回的 verdict 分组，生成新版 kb-build-qa.md。**保留** pre-goal4 版本的 Summary / Quality Warning 章节（那些是 Stage 5 的 Review 分析），追加 Goal 4 分诊结果。

新结构：

```markdown
# {{PROJECT_NAME}} KB Build QA Report（Goal 4 Triaged）

Generated at: <ISO 8601>
Seed pack version: {{SEED_PACK_VERSION}}
Review rounds: <继承>（<继承>）
Goal 4 triaged at: <ISO 8601>

## Triage Summary

| Verdict | Count | 后续动作 |
|---------|-------|----------|
| fully_answered | {A} | KB 已能回答，可选人类审阅（若想确认 AI 答案质量） |
| partially_answered | {B} | 人类补充 kb_gaps 中列出的缺口即可 |
| cannot_answer_needs_human | {C} | **必须**人类回答 |
| should_not_be_a_question | {D} | 已剔除，无需处理 |
| **人类必须处理** | **{B+C}** | |

## Fully Answered by KB（{A} 项）

以下问题 KB 已能回答。人类可选审阅（附 AI 答案供参考）。

### [{entry_id}] {问题简述}
- **原问题位置**：{QA_LOCATION}
- **AI 答案**：{ai_answer}
- **KB 依据**：
  - [{path}]({path}) §{section}
- **Confidence**：{confidence}
{若有 suggested_kb_patch，则追加：}
- **KB patch 建议**：见 `pass/kb-doc-patches.md#patch-{entry_id}`

## Partially Answered（{B} 项，需人类补缺口）

### [{entry_id}] {问题简述}
- **原问题位置**：{QA_LOCATION}
- **AI 已回答部分**：{ai_answer}
- **KB 缺什么**（需人类补充）：
  - {kb_gap_1}
  - {kb_gap_2}
- **KB 依据**：[{path}]({path}) §{section}

## Cannot Answer, Needs Human（{C} 项，必须人类回答）

### [{entry_id}] {问题简述}
- **原问题位置**：{QA_LOCATION}
- **为什么 KB 不覆盖**：
  - {kb_gap_1}
  - {kb_gap_2}
- **建议问答形式**：{按 knowledge-extract-qa-guide.md 引导人类回答}

## Should Not Be a Question（{D} 项，已剔除）

以下问题在全局视角下判定为不成立，已从人类回答队列剔除。

### [{entry_id}] {问题简述}
- **原问题位置**：{QA_LOCATION}
- **剔除理由**：{should_not_be_reason}

## Quality Warning（若软收敛，继承自 Stage 5）

{原样保留}

## 回流指引

**人类审阅完 fully_answered 后**（可选）：
- 若同意 AI 答案：无需操作
- 若质疑 AI 答案：手动重新分诊该条目（编辑 kb-build-qa.md 移到 cannot_answer_needs_human 分组）

**人类回答 partially_answered / cannot_answer_needs_human 后**，Agent 需：
1. 打开对应的 knowledge-base/*.md 文档
2. 根据人类回答修订正文（保留符号名锚定）
3. 在 kb-build-qa.md 中把该条目标记 resolved

**KB patch 应用**：查看 `pass/kb-doc-patches.md`，选定要 apply 的 patches，运行 `/maintain-kb --apply-patches`（若已实现）或手动 apply。
```

### 12.2 生成 kb-doc-patches.md

汇总所有 sub-agent 返回的 `suggested_kb_patch`：

```markdown
# {{PROJECT_NAME}} KB Documentation Patches（Goal 4 建议）

Generated at: <ISO 8601>
Total patches: {P}

**说明**：本文件由 Goal 4 sub-agent 在分诊过程中产出，每个 patch 是对 KB 文档的建议修改（追加 / 插入 / 修订）。**未经人类批准前不应 apply**。批准后可运行 `/maintain-kb --apply-patches` 落地，或手动 apply。

---

## Patch #{entry_id}

- **来源问题**：[{entry_id}] {问题简述}
- **目标文件**：{target_file}
- **目标章节**：{target_section}
- **Patch 类型**：{append | insert | revise}
- **理由**：{rationale}

### 建议内容

```markdown
{content}
```

---

（更多 patch...）
```

若无 patch 建议：文件仍生成，仅含头部说明 + "Total patches: 0"。

### 12.3 Stage 12 收尾

- 验证 kb-build-qa.md 已按 4 档重组
- 验证 kb-doc-patches.md 已生成
- 验证 kb-build-qa-pre-goal4.md 备份仍在
- 更新 Progress Log：`Stage 12: ✅`
- Journal 追加 `action: goal_completed`

---

## Progress Log

<!-- 唯一权威状态源。每 Stage / Batch 完成时重写对应行。 -->

| Stage | 状态 | 开始时间 | 完成时间 | 耗时 | 备注 |
|-------|------|---------|---------|------|------|
| 10. 备份 + 解析 | ⬜ | | | | |
| 11. 分诊 sub-agent 批并行 | ⬜ | | | | |
| 12. 重组 kb-build-qa + 生成 patches | ⬜ | | | | |

**Stage 11 batch 明细**：

| Batch | 状态 | Entry 数 | Fully | Partial | Human | NotAQ | 完成时间 |
|-------|------|---------|-------|---------|-------|-------|---------|
{{STAGE11_BATCH_PROGRESS_ROWS}}

**Goal 状态**：⬜ pending

---

## 恢复协议

见 `primer.md` 的恢复协议章节。Goal 4 的 Journal 路径：`pass/kb-seed-journal-kb-qa-triage.md`。

由于 Stage 10（备份 + 解析）与 Stage 12（重组）都是主 Agent 幂等动作，Stage 11 的 sub-agent 也是无状态的（每次 fresh），Goal 4 高度可恢复：从 Journal 中最后一个成功 batch 之后继续即可。

---

## 与人类回答阶段的关系

Goal 4 是 **种子包完整构建周期的最后一环**（Goal 1-4 的第 4 步），**默认执行**——Goal 3 GOAL_MET 之后主 Agent 无需询问人类，直接启动本 Goal。

**Goal 4 与人类回答阶段的分工**：

- **Goal 4 做的**：用 KB 全局视角尝试自动回答 QA 问题，把无法回答的问题精准圈出来
- **人类回答阶段做的**：回答 Goal 4 圈出的 `partially_answered` + `cannot_answer_needs_human` 项
- **Goal 4 不能替代人类**——它是**过滤器**（把 KB 已能回答的问题从队列剔除），不是回答者

**降级出口（罕见）**：

以下情形允许主 Agent 跳过 Goal 4，直接把 `pass/kb-build-qa.md` 交给人类回答：

1. **技术阻塞**：`.claude/skills/explore-kb/SKILL.md` 未安装（Goal 3 Stage 7 应已迁移，若缺失说明 Goal 3 未完成，属于依赖失败）
2. **人类显式跳过**：人类在 Goal 3 完成或 Goal 4 启动后，**明确用自然语言**（如"跳过 Goal 4"/"直接进入人工回答"/"这次不做分诊"）要求跳过
3. **本 Goal 触发 GOAL_ABANDONED**（见 §出工条件）

**默认不询问**：仅当 QA 问题总数极小（`kb-build-qa.md` 中 Critical + Medium 总数 < 5）时，主 Agent **可选**在启动 Goal 4 前用 `AskUserQuestion` 询问是否跳过；其他情况下直接启动，不打断执行流。

**跳过后如何继续**：跳过 Goal 4 后主 Agent 应输出：

```
Goal 4 已跳过（原因：{显式跳过 | explore-kb 缺失 | 其他}）
种子包构建周期到 Goal 3 结束。
  人类可直接基于 pass/kb-build-qa.md 回答 QA 问题
  若后续想补跑分诊，随时可 /goal .think-tank/indexable-docs-kb-seed/qa-triage.md 重启 Goal 4
```
