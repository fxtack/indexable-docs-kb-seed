# Stage 5 Aggregation Prompt

> 服务于 Goal 2 Stage 5（QA 聚合）。这一 Stage 主 Agent 通常直接执行；但若文档数量大（N_total > 20）或希望隔离上下文，可分派 1 个 sub-agent 完成。
> 本文档同时作为**主 Agent 的执行指令**和**（如需）sub-agent 的 prompt 模板**。

---

## 任务

扫描 `.think-tank/knowledge-base/pass/*.qa.jsonl`，合并、去重、分组、分类，输出 `.think-tank/knowledge-base/pass/kb-build-qa.md`。

**注意**：所有路径**必须**是完整的 `.think-tank/knowledge-base/pass/` 前缀。若发现散落在其他位置的 qa.jsonl（如 `pass/xxx.qa.jsonl` 位于工程根目录、或 `.think-tank/knowledge-base/architecture/pass/xxx.qa.jsonl`），说明 Stage 3/4 sub-agent 未遵守路径规范——此时应：
1. 记录违规位置到 Journal
2. 主 Agent 决定：小规模散落时可**收拢**（将散落文件移到规范位置后再合并）；大规模散落时应**中止本 Stage** 并触发 Stage 3/4 重跑（因为可能有更多未被发现的散落文件）

---

## 输入

- 所有 `.think-tank/knowledge-base/pass/{doc_id}.qa.jsonl` 文件（doc_id 列表来自 Goal 2 参数 `review_docs`）
- `PROJECT_NAME`
- `SEED_PACK_VERSION`
- `REVIEW_ROUNDS_ACTUAL`：实际执行了几轮 Review（1..review_max_rounds）
- `CONVERGENCE_STATUS`：`hard` / `soft`（是否达到硬收敛，或达到轮次上限进入软收敛）

---

## 处理逻辑

### 1. 读入所有 jsonl 条目

每个 jsonl 每行一个 JSON 对象。合并所有文件的所有行到统一列表。

### 2. 状态解析

对每个条目：
- 若 `status == "resolved_in_this_round"` → 状态：**resolved**
- 若有其他条目 `resolves` 指向本条目的 entry_id → 状态：**resolved**
- 否则 → 状态：**open**

### 3. 分组分类

- 按 `severity` 分组：critical / medium / minor
- 按维度分类（从 `doc_id` 推断，或查 index.yaml）：architecture / module / flow / troubleshooting

### 4. 若 `CONVERGENCE_STATUS == "soft"`，识别 Quality Warning 项

未 resolved 且 severity in (critical, medium) 的条目，额外计算 `trend`：
- 从最早到最新 round 的该 doc 内 severity 数量变化
- `converging`（每轮减少）/ `oscillating`（反复）/ `growing`（增多）

### 5. 输出 `.think-tank/knowledge-base/pass/kb-build-qa.md`

结构：

```markdown
# {PROJECT_NAME} KB Build QA Report

**Generated at**: <当前 ISO 8601 时间>
**Seed pack version**: {SEED_PACK_VERSION}
**Review rounds**: {REVIEW_ROUNDS_ACTUAL} ({CONVERGENCE_STATUS} converged)

---

## Summary

| Severity | Open | Resolved | Total |
|----------|------|----------|-------|
| Critical | {O_c} | {R_c} | {T_c} |
| Medium | {O_m} | {R_m} | {T_m} |
| Minor | {O_min} | {R_min} | {T_min} |

**By dimension**（open only）:
- Architecture: {arch_open}
- Module: {mod_open}
- Flow: {flow_open}
- Troubleshooting: {ts_open}

---

## Critical Issues（需人类紧急回答）

### [{doc_id}] {问题简述}
- **File**: `{doc_path}` §{section}
- **Detected in round**: {round}
- **Status**: open
- **Type**: {type}
- **Code reference**: `{code_reference}`
- **Question**: {完整问题描述}

（重复以上结构，每个 critical 条目一段）

---

## Medium Issues

（结构同上）

---

## Minor Issues（可延后处理）

（简化列表，一行一条）

- [{doc_id}] §{section} — {question} ({code_reference})

---

## Quality Warning（仅软收敛时）

以下项达到 Review 轮次上限（{REVIEW_ROUNDS_ACTUAL} 轮）仍未解决：

### [{doc_id}] {问题简述}
- **Severity**: {severity}
- **Trend**: {converging | oscillating | growing}
- **Round history**: r1={c:X, m:Y}, r2={c:X, m:Y}, ...
- **建议处理方式**: {oscillating → 需要人类介入决策 / growing → Stage 3 prompt 需改进 / converging → 追加一轮 Review 可能收敛}

---

## 回流指引

**人类回答 open 条目后，Agent 应执行**：
1. 打开对应的 `knowledge-base/*.md` 文档
2. 根据回答修订正文（保留符号名锚定，不引入行号）
3. 在 `.think-tank/knowledge-base/pass/{doc_id}.qa.jsonl` 中追加一行：`{"round": "post-build", "resolves": <entry_id>, "status": "resolved", "action_taken": "<how>"}`
4. 若涉及 index.yaml 的 keywords 或 description 变更，同步更新

**回流指引**：v1.2 起 kb-build-qa.md 会先经 Goal 4 (`kb-qa-triage`) 做全局视角回答尝试，Goal 4 分诊完成后 kb-build-qa.md 会分组为 4 档（fully_answered / partially_answered / cannot_answer_needs_human / should_not_be_a_question）。此处的"回流指引"由 Goal 4 阶段的产物取代，不再需要在此展开。

**Post-build QA 参考**：见 `knowledge-extract-qa-guide.md`——用于收集**代码里读不出来**的隐性知识（测试策略、性能基准、部署拓扑、事故经验）。两种 QA 机制的区分见 planning §5.4。
```

---

## 后续动作：清理原始 jsonl（v1.2.0 新增）

**由主 Agent 直接执行**（不是 sub-agent 的职责，但作为 Stage 5 的完整流程记入本文档）。

### 校验清单（必须全部通过）

1. `.think-tank/knowledge-base/pass/kb-build-qa.md` 存在且非空
2. `kb-build-qa.md` 中的 QA 条目数 ≥ 原 jsonl 汇总行数（合并未丢失）
3. 各严重度计数与 jsonl 汇总一致
4. **无散落 jsonl 检查**：`find` 扫描以下位置确认没有遗漏的 `*.qa.jsonl`：
   - 工程根目录及各级子目录（不含 `.think-tank/knowledge-base/pass/`）
   - `.think-tank/knowledge-base/` 的任何子目录（`architecture/pass/`、`modules/pass/` 等错误位置）
   - 若发现散落文件，先合并入 kb-build-qa.md 或触发 Stage 3/4 重跑，**再执行清理**

### 执行

```bash
find .think-tank/knowledge-base/pass/ -maxdepth 1 -name '*.qa.jsonl' -delete
```

**注意**：只删除 `.think-tank/knowledge-base/pass/` 目录下的 `{doc_id}.qa.jsonl` 命名模式的文件；`kb-build-qa.md` / `kb-seed-*.md` / 其他 pass/ 内容不受影响。**禁止**递归删除或删除其他位置的 `.qa.jsonl`（那可能是散落文件的证据，应先分析再决定处理方式）。

### Journal 记录

```
{action: decision, decision: qa_jsonl_cleanup, deleted_count: <N>, kept_files: [kb-build-qa.md]}
```

### 失败处理

若校验失败：不删除任何文件，Journal 追加 error，Stage 5 输出 `GOAL_MET_WITH_WARNING`，人类可手动 review。

---

## 输出（若作为 sub-agent 返回）

```json
{
  "output_file": ".think-tank/knowledge-base/pass/kb-build-qa.md",
  "total_entries": 47,
  "open_critical": 3,
  "open_medium": 8,
  "open_minor": 12,
  "resolved": 24,
  "quality_warning_items": 2,
  "notes": "..."
}
```

---

## 硬约束

- 不修改任何 knowledge-base/ 下的文档（回流是后续步骤，本 Stage 只做聚合报告）
- 不删除任何 jsonl 条目（保留原始记录用于后续追溯）——但**允许**在合并成功且校验通过后，由主 Agent（不是本 sub-agent）删除整份 jsonl 文件（v1.2.0）
- 不推断人类的回答（open 就是 open，不要脑补答案）
