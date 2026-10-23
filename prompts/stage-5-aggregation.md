# Stage 5 Aggregation Prompt

> 服务于 Goal 2 Stage 5（QA 聚合）。主 Agent 通常直接执行；文档数量大时可选 1 个 sub-agent。
> 本文档同时作为**主 Agent 的执行指令**和（如需）sub-agent 的 prompt 模板。

---

## 任务

扫描 `.think-tank/knowledge-base/pass/*.qa.jsonl`，合并、去重、按 `entry_id` 取最后版本，输出：

- `.think-tank/knowledge-base/pass/kb-build-qa.md`：**只收 `knowledge_gap`**；
- 已解决的 `doc_defect` 只做摘要，不作为人类问题；
- 未关闭的 `doc_defect` 只能出现在 `Quality Warning`，且仅在 warning 模式下允许。

**不要读取或合并** `.think-tank/knowledge-base/pass/build-findings/discovered-issues.jsonl`：它是构建期代码/流程问题的独立账本，不进入 QA。

---

## 输入

- 所有 `.think-tank/knowledge-base/pass/{doc_id}.qa.jsonl`
- `PROJECT_NAME`
- `SEED_PACK_VERSION`
- `REVIEW_ROUNDS_ACTUAL`
- `CONVERGENCE_STATUS`：`hard` / `warning`
- `.think-tank/knowledge-base/pass/kb-review-ledger.md`（用于交叉检查）

---

## 处理逻辑

### 1. 读入并按 entry_id 取最后版本

每个 jsonl 每行一个 JSON 对象。同一个 `entry_id` 可能出现多次（追加式更新），**最后一行是该条目的当前状态**。

- 不要按行数统计；
- 不要删除旧行；
- 若同一 `entry_id` 的 `round` 倒退，记 `invalid_entry_round` 并按最后一行处理；
- 若最后一行缺少 `fixed_in_round`，回退到同 `entry_id` 最近一条有值的行。

### 2. 状态合法性校验

对每个条目的最后版本：

- `category` 必须是 `doc_defect` / `knowledge_gap`；
- `status` 必须是 `open` / `resolved` / `knowledge_gap` / `duplicate`；
- 若 `status in (resolved, knowledge_gap, duplicate)` 且 `category == doc_defect`：
  - 必须存在 `verified_in_round`；
  - 必须满足 `verified_in_round > fixed_in_round`（若有）或 `verified_in_round > round`（若没有）；
  - 不满足 → 记 `invalid_self_close`，按 `open` 处理。
- `knowledge_gap` 类别可以直接是 `status: knowledge_gap`；
- `duplicate` 必须有 `duplicate_of`。

### 3. 收敛门禁

按 `CONVERGENCE_STATUS` 分两种：

**`hard`**：

- 若存在 `category == doc_defect`、`severity in (critical, medium)`、`status == open` 的条目 → **中止 Stage 5**，回到 Stage 4；
- 若 ledger 中 `next_coverage` 非空 → 中止 Stage 5，回到 Stage 4；
- 全部通过才继续。

**`warning`**：

- 允许存在未关闭 critical/medium `doc_defect`，但只能写入 `Quality Warning`；
- 不得把它们混进 `knowledge_gap` 的 Critical/Medium/Minor 章节；
- 不得声称硬收敛。

### 4. 分组

- `knowledge_gap`：按 severity 分组（critical / medium / minor），按维度分类；
- `doc_defect`：按 status 分组，只生成摘要：
  - `resolved`：已修复的缺陷数量和 entry_id 列表；
  - `open`：仅 warning 模式下进入 `Quality Warning`。

### 5. 输出 `kb-build-qa.md`

结构：

```markdown
# {PROJECT_NAME} KB Build QA Report

**Generated at**: <ISO 8601>
**Seed pack version**: {SEED_PACK_VERSION}
**Review rounds**: {REVIEW_ROUNDS_ACTUAL} ({CONVERGENCE_STATUS})

---

## Summary

| 类别 | Open | Resolved | Total |
|------|------|----------|-------|
| Knowledge gaps · Critical | {O_c} | {R_c} | {T_c} |
| Knowledge gaps · Medium | {O_m} | {R_m} | {T_m} |
| Knowledge gaps · Minor | {O_min} | {R_min} | {T_min} |
| Document defects | {O_d} | {R_d} | {T_d} |

> 本报告的 Critical / Medium / Minor 章节**只包含 knowledge_gap**（代码读不出来的设计意图）。
> 文档缺陷已在 Stage 4 修复；未关闭项只在 warning 模式的 Quality Warning 中出现。

**By dimension**（open knowledge gaps only）:
- Architecture: {arch_open}
- Module: {mod_open}
- Flow: {flow_open}
- Troubleshooting: {ts_open}

---

## Critical Issues（knowledge gaps，需人类回答）

### [{entry_id}] {问题简述}
- **File**: `{doc_path}` {location}
- **Detected in round**: {round}
- **Category**: knowledge_gap
- **Type**: {type}
- **Code reference**: `{code_reference}`
- **Question**: {完整问题描述}

---

## Medium Issues（knowledge gaps）

（结构同上）

---

## Minor Issues（knowledge gaps，可延后）

- [{entry_id}] {doc_path} {location} — {question} ({code_reference})

---

## Resolved Document Defects（已修复，不需人类回答）

- 总数：{R_d}
- entry_id：{list}
- 说明：这些是 `doc_defect`，已由后续轮次 fresh reviewer 确认；明细保留在原始 jsonl / ledger 中。

---

## Quality Warning（仅 warning 模式）

以下 `doc_defect` 在轮次/attempts 上限后仍未关闭，需要人类决策：

### [{entry_id}] {问题简述}
- **File**: `{doc_path}` {location}
- **Severity**: {severity}
- **Status**: open
- **Round history**: r1={...}, r2={...}
- **已尝试的修复**: {action_taken}
- **建议**: 接受现状 / 追加定向修复 / 标记 draft

---

## 回流指引

**人类回答 knowledge_gap 后，Agent 应执行**：
1. 打开对应的 `knowledge-base/*.md` 文档；
2. 根据回答修订正文（保留符号名锚定）；
3. 在 `{doc_id}.qa.jsonl` 追加一行同 `entry_id` 的新版本：`status: "resolved"`，`resolution: "fixed"`；
4. 若涉及 `index.yaml` 字段，按 `index-impact` 规则同步。

**人类回答后是否需要复查**：knowledge_gap 的答案回填属于 post-build 内容变更；建议之后运行 `/maintain-kb` 做一次一致性检查。
```

### 6. 清理原始 jsonl

前置校验：

1. `kb-build-qa.md` 已生成且非空；
2. `kb-build-qa.md` 中 knowledge_gap 条目数 = 所有 jsonl 按 `entry_id` 去重后的 knowledge_gap 数；
3. 各 severity 计数一致；
4. 无散落 `*.qa.jsonl`（§5.1 扫描已通过）；
5. `kb-review-ledger.md` 存在，且 `next_coverage` 为空（hard 模式）或已记录 warning（warning 模式）。

执行：

```bash
find .think-tank/knowledge-base/pass/ -maxdepth 1 -name '*.qa.jsonl' -delete
```

Journal：

```json
{"action": "decision", "decision": "qa_jsonl_cleanup", "deleted_count": N, "kept_files": ["kb-build-qa.md", "kb-review-ledger.md"]}
```

若校验失败：不删除任何文件，输出 `GOAL_MET_WITH_WARNING`，记录原因。

---

## 输出（若作为 sub-agent 返回）

```json
{
  "output_file": ".think-tank/knowledge-base/pass/kb-build-qa.md",
  "knowledge_gap_open_critical": 0,
  "knowledge_gap_open_medium": 0,
  "knowledge_gap_open_minor": 0,
  "knowledge_gap_resolved": 0,
  "doc_defect_resolved": 0,
  "doc_defect_open_warning": 0,
  "quality_warning_items": 0,
  "notes": ""
}
```

---

## 硬约束

- 不修改任何 `knowledge-base/` 下的文档（回流是后续步骤）；
- 不删除任何 jsonl 条目（保留原始记录用于追溯）——但允许在合并成功且校验通过后由主 Agent 删除整份 jsonl 文件；
- 不推断人类回答；
- 不把 `doc_defect` 混进 Knowledge Gaps 章节；
- 不把 `discovered_issues` 混进本报告；
- 不声称硬收敛，除非 `CONVERGENCE_STATUS=hard` 且所有校验通过。
