# Indexable Docs KB Seed · Post-build: verify-build-findings

> **执行方式**：本文档由 `/goal` 驱动执行。它**不属于知识库构建周期**（Goal 1-4），而是构建完成后的独立复核任务。
> **目的**：把构建期记录在 `pass/build-findings/` 的未确认发现，脱离「知识库构建」这个 Agent 任务上下文，独立核实是否属实；属实的才允许提升到 `troubleshooting/` 或 `works/`。
> **位置**：构建收官时，Goal 3/4 会把本文档复制到 `.think-tank/knowledge-base/pass/build-findings/verify-build-findings.md`。种子包目录被删除后，仍可从该副本执行。

---

## Goal 定义

**goal_id**: `kb-verify-findings`
**goal_name**: 构建期发现问题的独立复核与提升
**depends_on_goal**: `kb-qa-triage`（若 Goal 4 被跳过，则为 `kb-infra-setup`）
**seed_pack_version**: `<从 .think-tank/indexable-docs-kb-seed/VERSION 读取；若种子包已删除，从 index.yaml.meta.seed_pack_version 读取>`

---

## 核心原则

1. **独立上下文**：不要沿用知识库构建阶段的结论、QA 分类或修复记录。每个发现都从头核实。
2. **构建期只记录，复核期才定性**：`recorded` → `confirmed` / `rejected` / `needs_more_info` / `duplicate`。
3. **未确认不进 KB**：`troubleshooting/known-bugs.md` 只收 `confirmed` 且完成提升的条目。
4. **复核允许跑验证，但不修代码**：可以读代码、跑测试、复现、查 git history / issue；除非人类明确授权，不修改目标工程源码。
5. **证据可追溯**：每条结论必须附符号、命令/测试结果、版本/commit；禁止行号锚定。

---

## 输入

- `.think-tank/knowledge-base/pass/build-findings/discovered-issues.jsonl`（原始账本）
- `.think-tank/knowledge-base/pass/build-findings/kb-build-discovered-issues.md`（构建收官快照）
- `.think-tank/knowledge-base/index.yaml`（模块/流程清单与 covers_code）
- 相关 KB 文档、源码、测试、git history

若 JSONL 不存在或为空：输出 `GOAL_MET: 无构建期发现需要复核`，不创建任何 KB 条目。

---

## Goal 条件（GOAL_MET）

以下条件同时满足：

1. 所有 `status == recorded` 的条目都已有复核结果（`confirmed` / `rejected` / `needs_more_info` / `duplicate`）
2. `pass/build-findings/verification-report.md` 已生成
3. `discovered-issues.jsonl` 已回写 `verification` 与 `status`
4. 所有 `confirmed` 条目都有明确的提升决策：已提升（`promoted`）或人类明确要求暂缓并记录原因
5. Progress Log 中 V1-V4 均为 ✅

达成后输出：

```
GOAL_MET: verify-build-findings 完成。
  原始发现：N 项
  confirmed: A / rejected: B / needs_more_info: C / duplicate: D
  已提升：P 项（known-bugs X / works Y / incidents Z）
  仍留在 staging：Q 项（needs_more_info / 暂缓）
  报告：.think-tank/knowledge-base/pass/build-findings/verification-report.md
```

---

## 出工条件（GOAL_ABANDONED）

- 关键输入缺失且无法重建（JSONL 与快照都不存在）
- 累计轮次 > 30
- 复核 sub-agent 累计失败率 > 30% 且重试无进展
- 人类明确终止

输出 `GOAL_ABANDONED: {原因}`，保留原始 JSONL 和已生成的报告，不做删除。

---

## Stage V1: 盘点 + 去重

主 Agent 直接执行：

1. 读取 JSONL，按 `status` 分组：
   - 待复核：`recorded`、`needs_more_info`（若人类要求重试）
   - 已终态：`confirmed` + `promoted`、`rejected`、`duplicate`
2. 按 `subject` / `kind` / `confidence` / `doc_id` 统计
3. 去重：同一 `dedupe_key` 或高度相似 summary/evidence 的条目合并；保留最早一条为 canonical，其余标 `duplicate` 并在 `also_seen_in` 记录
4. 生成 `verification-report.md` 骨架（Summary + 待复核清单）
5. Journal 追加 `stage_completed: V1`

**Journal 路径**：`.think-tank/knowledge-base/pass/build-findings/verify-journal.md`

---

## Stage V2: 独立复核

### V2.1 分派策略

每个条目分派一个 fresh sub-agent；条目数多时按 `subject + kind` 聚合成小组（每组 ≤ 6 条），但同一 canonical 条目只能由一个 sub-agent 负责。

每个 sub-agent 收到：

- 该条目的完整 JSON 行
- `RELATED_KB_DOC`（若有）
- `COVERS_CODE` / 相关源码路径（从 `index.yaml` 查，或由主 Agent 从 evidence 推断）
- 当前代码版本/commit
- 明确指令：**独立核实，不继承构建期结论**

### V2.2 复核动作

允许：

- 读源码、测试、配置文件、git history
- 运行非破坏性测试/复现命令（在临时目录或只读方式）
- 查已有 issue / 文档 / 版本说明（若环境允许）
- 向人类提出一个具体的确认问题（仅在无法自行判定时）

禁止：

- 修改目标工程源码/配置/测试来“让它通过”
- 删除或改写原始 evidence
- 把未确认结论写成 KB 条目
- 越过 `excluded_paths` 分析第三方代码

### V2.3 判定标准

| 结果 | 判据 |
|---|---|
| `confirmed` | 可复现，或代码/测试/owner 明确证明行为不符合预期；且影响范围可说明 |
| `rejected` | 行为是刻意设计/文档写错/版本差异/第三方行为，不构成缺陷 |
| `needs_more_info` | 环境缺失、无法复现、预期行为不明确、需要 owner 决策 |
| `duplicate` | 与另一条 canonical 条目语义相同 |

对每条写入：

```json
{
  "status": "confirmed",
  "verification": {
    "verified_at": "<ISO 8601>",
    "verified_by": "<agent 名称或 human>",
    "method": "<复现命令 / 测试 / 代码核对 / owner 确认>",
    "result": "confirmed",
    "target_version": "<commit 或 tag>",
    "notes": "<结论与边界>"
  }
}
```

`rejected` / `needs_more_info` / `duplicate` 同样写入 `verification.result`，并在 `notes` 说明理由。不要删除原条目。

### V2.4 Stage V2 收尾

- 所有待复核条目都有结果
- `verification-report.md` 更新 Summary 与分组
- Journal 追加 `stage_completed: V2`

---

## Stage V3: 提升建议

主 Agent 直接执行，按结果分组：

### V3.1 `confirmed` 条目

对每条给出提升建议：

| 情况 | 目标 |
|---|---|
| 明确的代码缺陷，可用症状/根因/规避描述 | `troubleshooting/known-bugs.md` |
| 真实发生过的故障事件 | `troubleshooting/incidents/` |
| 需要设计、排期或改动较大 | `works/` 修复任务 |
| 目标工程外的流程/工具问题 | `pass/kb-seed-postmortem.md` 或种子包 issue |

`known-bugs.md` 条目至少包含：

- 症状（用户可观察）
- 根因（符号名锚定）
- 影响范围（模块 / 流程 / 版本）
- 规避方式（若有）
- 关联 KB 文档
- 来源 `DISC-xxxx`

### V3.2 `rejected` / `needs_more_info` / `duplicate`

- `rejected`：在报告中说明为什么不是缺陷；不写 KB
- `needs_more_info`：列出还缺什么信息；留在 `build-findings`，不写 KB
- `duplicate`：指向 canonical `DISC-xxxx`

### V3.3 输出

在 `verification-report.md` 增加「提升建议」章节，逐条列出目标文件与建议内容。**此时不要直接写入 `troubleshooting/`**。

报告至少包含：

```markdown
# 构建期发现复核报告

## Summary
- recorded / confirmed / rejected / needs_more_info / duplicate / promoted
- 目标工程 X 项 / 构建流程 Y 项

## Confirmed（待提升）
### [DISC-xxxx] ...
- 复核方式与证据
- 建议去向
- 建议内容

## Rejected
## Needs More Info
## Duplicate

## Apply Log
（人类批准后追加）
```

---

## Stage V4: 人类批准 + 提升

1. 主 Agent 输出提升建议，等待人类明确批准（可逐条批准或批量批准）
2. 只对已批准条目执行写入：
   - `troubleshooting/known-bugs.md`：按该文件现有格式追加
   - `troubleshooting/incidents/`：按 `.meta-incident-guide.md` 创建 Incident Record
   - `works/`：按 `.meta-work-guide.md` 创建 Work 文档
3. 写入后回写 JSONL：

```json
{
  "status": "promoted",
  "promotion": {
    "destination": "troubleshooting/known-bugs.md",
    "target_path": ".think-tank/knowledge-base/troubleshooting/known-bugs.md#BUG-xxx",
    "promoted_at": "<ISO 8601>",
    "promoted_by": "<agent 或 human>",
    "note": "<提升说明>"
  }
}
```

4. 未批准的 `confirmed` 条目保持 `confirmed`，在报告中标注「已确认，待人类决定」；不要降级为 rejected
5. 更新 `verification-report.md` 的 Apply Log
6. Journal 追加 `stage_completed: V4`

**硬约束**：未确认条目不写入 `troubleshooting/`；不把 `needs_more_info` 伪装成 `confirmed`。

---

## Stage V5: 收尾

- 验证所有条目状态合法
- 验证 `verification-report.md` 与 JSONL 一致
- 验证已提升条目在目标文件中可检索
- 更新 Progress Log
- Journal 追加 `goal_completed`

---

## Progress Log

| Stage | 状态 | 开始时间 | 完成时间 | 备注 |
|-------|------|---------|---------|------|
| V1. 盘点 + 去重 | ⬜ | | | |
| V2. 独立复核 | ⬜ | | | |
| V3. 提升建议 | ⬜ | | | |
| V4. 人类批准 + 提升 | ⬜ | | | |
| V5. 收尾 | ⬜ | | | |

**Goal 状态**：⬜ pending

---

## 恢复协议

1. 读 Progress Log，找到最后一个非 ✅ Stage
2. 读 `discovered-issues.jsonl` 与 `verification-report.md`，按 `status` 恢复
3. 已完成复核的条目不要重复分派；`recorded` 条目重新分派 fresh sub-agent
4. 若 JSONL 与报告冲突，以 JSONL 为准，重新生成报告

---

## 与知识库构建的关系

- 本文档**不是** Goal 1-4 的一部分，不改变 KB 构建协议
- 构建期只负责把发现写入 `pass/build-findings/`，不确认、不修复、不阻塞
- 本文档负责构建后的独立确认与提升；确认后才允许进入 `troubleshooting/`
- 若人类选择不执行本任务，`build-findings` 仍保留完整证据链，不影响已构建的 KB
