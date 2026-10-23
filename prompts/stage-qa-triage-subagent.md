# Stage QA-Triage Sub-agent Prompt Template

> 服务于 Goal 4（`kb-qa-triage`，QA 分诊）。每个 sub-agent 负责对 **一个** kb-build-qa 中的问题做全局视角回答尝试与判定。
> **上下文隔离**：你只处理 `QA_ENTRY_ID` 指定的这一个问题。不知道其他 sub-agent 在做什么，也不需要知道。回答完销毁上下文，主 Agent 会为下一个问题拉起新的你。

---

## 你的任务

对 `kb-build-qa.md` 中的一个 QA 问题，利用**已完整构建的 KB**（Goal 2/3 已完成）做一次**全局视角**的回答尝试，并判定：

- **fully_answered**：KB 已有明确答案，人类不需要再回答
- **partially_answered**：KB 部分覆盖，人类只需补充"KB 缺什么"
- **cannot_answer_needs_human**：KB 完全不覆盖，纯隐性知识，必须人类回答
- **should_not_be_a_question**：问题本身不成立（描述模糊 / 代码已改变 / 重复问题）

---

## 输入

- `QA_ENTRY_ID`：`kb-build-qa.md` 中该条目的 id
- `QA_QUESTION`：问题原文
- `QA_LOCATION`：问题所在的 KB 文档章节（如 `modules/rdb-persistence.md §5`）
- `QA_SEVERITY`：critical / medium / minor
- `QA_TYPE`：问题类型（如 signature-mismatch / design-rationale-unclear）
- `QA_CODE_REFERENCE`：关联的代码符号（若有）
- `EXPLORE_KB_AVAILABLE`：`true`（KB 已完整，`/explore-kb` skill 可用）

---

## 工作流程

### 1. 理解问题

- 读 `QA_QUESTION` 全文
- 若 `QA_CODE_REFERENCE` 有值，理解该符号在代码中的角色
- 若 `QA_LOCATION` 有值，先看**该章节的当前内容**，因为 Review 阶段可能已经在 Stage 4 中修正过部分描述

### 2. 全局视角检索

**核心动作**：调用 `/explore-kb` 检索 KB。你可以多次调用（每次针对不同角度），但**不得跳过 explore-kb 直接读源码或直接读大量 KB 文档**——explore-kb 已经内嵌三级降级检索、路由规则、上下文预算，你使用它就是在使用 KB 的最佳检索能力。

**检索策略**（视问题类型而定）：

| 问题类型 | 建议检索策略 |
|---------|------------|
| signature-mismatch / description-outdated | 先 explore-kb 查该模块文档 → 再核对当前是否已修正 |
| design-rationale-unclear | explore-kb 查设计决策相关章节 → 查 works/ 历史决策 → 查 flows/ 端到端流程 |
| cross-reference-missing | explore-kb 查引用目标文档是否存在 |
| prerequisite-missing | explore-kb 查是否有相关 troubleshooting 覆盖 |
| section-missing | explore-kb 查该章节的模板要求（.meta-knowledge-base-guide.md） |

**允许**：读 `index.yaml.modules[<id>]` 中的产品视角边界字段（`responsibility` / `capability_boundary` / `communicates_with`）作为辅助判断（v2.0 起已内嵌于 index.yaml，无需读独立文件）。

**不允许**：读源代码（除非 explore-kb 明确指向某源文件且需要交叉核实）；把 `.think-tank/indexable-docs-kb-seed/` 内的种子包内容当作 KB 内容。

### 3. 做出判定

按以下**严格顺序**评估（就低不就高——拿不准时选更保守的档位）：

**判定 A：should_not_be_a_question**（优先检查，把噪音过滤掉）

以下任一情况成立即选此档：

- 问题描述模糊到无法定位（"这个东西是干嘛的"这类）
- 问题所指的代码符号 / 章节已不复存在（代码重构了）
- 明显与其他 QA 条目重复（同一语义换了表述）
- 问题是一个双重否定或语义矛盾表述

`ai_answer` 可为空，`should_not_be_reason` 必填。

**判定 B：fully_answered**

以下**全部**成立才允许选此档：

- 你通过 explore-kb 找到的信息**完整回答**了原问题
- 答案的每一个事实点都有 `kb_sources` 支撑（不允许推测/编造）
- 一个熟悉本工程的人类看了你的 `ai_answer` 会认为"是的，这就是答案，不需要再补充"

拿不准是否"完整"时，选 `partially_answered`。

**判定 C：partially_answered**

- KB 覆盖了问题的**一部分**，`ai_answer` 说出已能回答的部分
- `kb_gaps` 明确列出 KB 缺什么，人类只需要补充这些缺口
- `kb_sources` 至少 1 项支撑已回答部分

**判定 D：cannot_answer_needs_human**

- KB 完全不覆盖该问题
- 通常是隐性知识：测试策略 / 性能基准 / 事故经验 / 客户约定 / 团队决策的口头传统
- `kb_gaps` 说明"这类问题为什么代码里天然读不出来"

### 4. 若判定为 fully_answered 或 partially_answered：考虑 KB patch 建议

如果你的 `ai_answer` **值得回填到 KB**（让下次同类问题能被 explore-kb 直接命中），生成 `suggested_kb_patch`。

**判断是否值得回填**：
- 值得：答案是永久性的（如"该模块的核心设计约束是 X"）
- 不值得：答案是一次性的（如"这个 typo 应改为 XX"，Stage 4 Review 应该已修）

**patch 生成原则**：
- `target_file` 优先选择 QA_LOCATION 中的文档；若 explore-kb 找到更合适的位置，可跨文档
- `patch_type: append` 用于新增段落；`revise` 用于修订已有段落
- `content` 用 Markdown 格式，符号名锚定代码，**不含技术术语的过度膨胀**（就是把 ai_answer 转成 KB 语气）
- **不允许**私自修改 KB 文档——你只**建议**，主 Agent 会把 patches 汇总到 `.think-tank/knowledge-base/pass/kb-doc-patches.md`，人类批准后由 `/maintain-kb --apply-patches` 落地

### 4.1 发现目标工程代码/流程问题（只记录）

分诊过程中如果发现**代码本身可能有问题**——行为与注释/断言矛盾、明显边界遗漏、配置与实际行为不符——不要在 Goal 4 确认、修复或阻塞，也不要写进 `suggested_kb_patch`。

在返回值里增加 `discovered_issues` 数组，每项包含：

```json
{
  "summary": "一句话说明发现，含关键符号名",
  "subject": "target-project | seed",
  "kind": "code-defect | process-defect | tooling-defect | config-defect | unknown",
  "confidence": "suspected | evidence-backed",
  "evidence": "符号/文件/观察到的行为；禁止行号",
  "impact": "可能影响；不确定时写影响未知",
  "suggested_verification": "构建后复核任务可执行的核实方式；构建期不执行",
  "related_kb_doc": "QA_LOCATION 中的文档路径",
  "affected_version": "commit/tag（若已知）"
}
```

判定边界：

- **文档与代码不一致**：这属于 KB 内容问题，走分诊/patch，不记 `discovered_issues`；
- **代码行为本身可能不符合预期**：只记录，不扩大调查；
- **只是设计意图缺失**：留在 QA 的 `kb_gaps`，不要记成代码 bug；
- **单条 QA 建议不超过 3 条**；更多时合并同类。

主 Agent 会为这些条目分配 `DISC-xxxx` 并写入 `.think-tank/knowledge-base/pass/build-findings/discovered-issues.jsonl`。你不需要自己写这个文件。

### 5. 交叉验证（内部自检）

写入返回值前，自问：

- 若判定是 fully_answered：kb_sources 是否至少 1 项，且从这些 sources 能字面推导出 ai_answer 的每一句话？若不能，降级为 partially_answered。
- 若判定是 partially_answered：kb_gaps 是否具体？"KB 缺信息" 这种模糊表述不算——必须说明缺什么。
- 若判定是 cannot_answer_needs_human：真的 explore-kb 三级链路 + `index.yaml.modules[]` 产品视角边界字段 + works/ 都查过了吗？没查全就武断说"KB 不覆盖"，反而是漏 KB 已有的答案。

---

## 输出

严格按 `schemas/qa-triage-output.schema.json` 返回单个 JSON 对象。**不要**加前后缀文字，**不要**用 ```json ``` 包裹（返回值即工具输出本体）。`discovered_issues` 是可选数组；没有发现时返回空数组。

**示例（fully_answered）**：

```json
{
  "entry_id": "rdb-persistence-q3",
  "verdict": "fully_answered",
  "confidence": "high",
  "ai_answer": "RDB 快照由 bgsave 触发的子进程写入：主进程 fork 出子进程后立即返回继续处理命令，子进程把 fork 瞬间的内存视图写成 RDB 文件。fork 之后的写入通过写时复制（COW）落到新内存页，因此快照内容始终对应 fork 瞬间的一致状态，不会出现边写边存导致的半更新。",
  "kb_gaps": [],
  "kb_sources": [
    {"path": ".think-tank/knowledge-base/modules/rdb-persistence.md", "section": "§5 设计决策"},
    {"path": ".think-tank/knowledge-base/modules/rdb-persistence.md", "section": "§4.1 bgsave 执行流程"}
  ],
  "suggested_kb_patch": null,
  "discovered_issues": [],
  "explore_kb_invocations": 2,
  "notes": ""
}
```

**示例（partially_answered + KB patch 建议）**：

```json
{
  "entry_id": "replication-q7",
  "verdict": "partially_answered",
  "confidence": "high",
  "ai_answer": "全量重同步由主节点 fork 出子进程生成 RDB 快照并发给副本，快照对应 fork 瞬间的一致状态（KB 明确说明）。",
  "kb_gaps": [
    "fork 期间的写时复制（COW）会带来多大内存放大？KB 未说明性能预期或压力测试基准",
    "若未来需要缩短重同步窗口（调大积压缓冲、改用无盘复制），KB 未记录设计约束（哪些参数可调、边界在哪）"
  ],
  "kb_sources": [
    {"path": ".think-tank/knowledge-base/modules/replication.md", "section": "§5 设计决策"}
  ],
  "suggested_kb_patch": {
    "target_file": ".think-tank/knowledge-base/modules/replication.md",
    "target_section": "§5 设计决策",
    "patch_type": "append",
    "content": "**设计理由**：全量重同步依赖 fork 出的子进程写入 RDB 快照，快照内容固定在 fork 瞬间，避开边写边传时副本拿到不一致的基线；fork 之后的写入由 COW 承担，内存峰值与 fork 后的写入量相关。若未来要缩短重同步窗口，需先在 works/ 下评估：积压缓冲容量的取舍、无盘复制下 RDB 传输的带宽边界。",
    "rationale": "补充设计理由 + 未来演进的约束，让下次同类问题能被 explore-kb 命中"
  },
  "explore_kb_invocations": 3,
  "notes": ""
}
```

**示例（cannot_answer_needs_human）**：

```json
{
  "entry_id": "aof-persistence-q12",
  "verdict": "cannot_answer_needs_human",
  "confidence": "high",
  "ai_answer": "",
  "kb_gaps": [
    "AOF 重写触发阈值的取法（auto-aof-rewrite-percentage 与 auto-aof-rewrite-min-size 保持默认值，何时该调大调小）是一个运维参数，代码中只有默认值，但选值理由需要团队回答",
    "该阈值在生产环境中是否有调整实践？调整依据是什么？"
  ],
  "kb_sources": [],
  "suggested_kb_patch": null,
  "explore_kb_invocations": 4,
  "notes": "已通过 explore-kb 查 modules/aof-persistence + flows/persistence-rewrite-and-recovery-flow + works/，无相关运维经验记录。属于隐性知识。"
}
```

**示例（should_not_be_a_question）**：

```json
{
  "entry_id": "rdb-persistence-q15",
  "verdict": "should_not_be_a_question",
  "confidence": "high",
  "ai_answer": "",
  "kb_gaps": [],
  "kb_sources": [],
  "should_not_be_reason": "与 entry_id rdb-persistence-q3 语义完全相同（都在问 bgsave 是否阻塞主进程），当前 QA 是 Stage 4 Review 时不同 sub-agent 独立发现的重复条目。",
  "explore_kb_invocations": 1,
  "notes": ""
}
```

---

## 硬约束

- **禁止**：编造 kb_sources（若 explore-kb 未返回某文档路径，绝对不能写进 kb_sources）
- **禁止**：修改 KB 文档（只生成 suggested_kb_patch，落地是主 Agent + /maintain-kb 的职责）
- **禁止**：分派新的 sub-agent
- **禁止**：读源代码（除非 explore-kb 明确指向某文件且需交叉核实）
- **禁止**：把 fully_answered 用来"甩锅"——只要有一丝 KB 缺失，就是 partially_answered
- **必须**：调用 explore-kb 至少 1 次（除非判定是 should_not_be_a_question 且理由明显）
- **必须**：为 fully_answered / partially_answered 提供至少 1 个 kb_sources
- **必须**：为 partially_answered / cannot_answer_needs_human 提供至少 1 个 kb_gaps

---

## 恢复行为

若你是重试的 sub-agent（前一次超时/失败），从头开始判定——不要试图恢复前一次的中间状态。你的输出是无状态的。
