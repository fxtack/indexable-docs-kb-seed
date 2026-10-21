# Stage 3 Sub-agent Prompt Template

> 服务于 Goal 2 Stage 3（文档内容生成）。每个 sub-agent 负责生成一份文档，写入其正文内容。
> **上下文隔离**：你不知道其他 sub-agent 在做什么，也不需要知道。只关注自己的文档。

---

## 你的任务

为 **{DOC_ID}** 生成完整的知识库文档内容，写入 `{DOC_PATH}`。

**文档已有骨架**（frontmatter + 空章节标题），你的工作是**填充正文**——不修改 frontmatter，不删除章节标题。

---

## 输入

- `DOC_ID`：文档唯一 id（如 `rdb-persistence`）
- `DOC_PATH`：文档路径（相对于工程根目录）
- `COVERS_CODE`：本文档覆盖的源代码文件列表（**唯一事实来源**）
- `DEP_DOCS`（若适用）：本文档依赖的其他已完成文档路径列表（可引用但**不重复内容**）
- `INVOLVES_MODULES`（若为 flow 文档）：本流程涉及的模块 id 列表
- `META_KB_GUIDE_PATH`：`.think-tank/knowledge-base/.meta-knowledge-base-guide.md`（了解文档规范）
- `QA_JSONL_PATH`：**完整路径** `.think-tank/knowledge-base/pass/{DOC_ID}.qa.jsonl`（把不确定的问题写这里）
- `EXCLUDED_PATHS`：全局排除路径清单（第三方源码 / 生成产物 / submodule）。**Stage B 已经过滤过 `COVERS_CODE`**——正常情况下不会包含被排除路径下的文件；但你在追求"上下文"时可能想读 `COVERS_CODE` 之外的文件，此时**必须**参照此清单主动规避

---

## 硬约束：QA jsonl 路径严格规范（v2.0.1 新增）

⚠️ **QA jsonl 文件必须写入 `.think-tank/knowledge-base/pass/` 目录下，路径必须以 `.think-tank/knowledge-base/pass/` 开头**。这是硬性约束，违反会导致 pass 目录分散、无法合并、KB 结构损坏。

**正确示例**：
- ✅ `.think-tank/knowledge-base/pass/rdb-persistence.qa.jsonl`
- ✅ 使用主 Agent 传入的 `QA_JSONL_PATH` 参数**原样**作为写入路径

**禁止示例**（都会导致文件散落）：
- ❌ `pass/rdb-persistence.qa.jsonl`（相对路径可能解析到当前 CWD）
- ❌ `.think-tank/knowledge-base/architecture/pass/rdb-persistence.qa.jsonl`（不要按文档所在目录建 pass）
- ❌ `.think-tank/knowledge-base/modules/pass/rdb-persistence.qa.jsonl`（同上）
- ❌ `~/.pass/rdb-persistence.qa.jsonl` 或任何工程外路径

**如果 `QA_JSONL_PATH` 参数为空或未提供**：不要自行推断路径。返回值 `notes` 中标注 `missing_qa_jsonl_path`，主 Agent 会重试并补齐参数。**绝不允许**用 `pass/xxx.qa.jsonl` 这种相对路径兜底。

**写入前的目录确认**：写 QA jsonl 前，先确认 `.think-tank/knowledge-base/pass/` 目录存在（该目录应由 Goal 1 或 Goal 3 已创建）。若不存在，返回值 `notes` 中标注 `pass_dir_missing`，不要自建该目录（那可能建在错误位置）。

---

## 硬约束：尊重 EXCLUDED_PATHS

- **禁止**在文档正文中记录 `EXCLUDED_PATHS` 下的实现细节。允许**提及边界**（"本模块调用 boost::asio 的异步 socket"），不描述"boost::asio 如何实现异步 socket"；
- **禁止**将 `EXCLUDED_PATHS` 下的文件写入 QA jsonl 的 `code_reference`——第三方代码的问题不是我们的 KB 要回答的；
- **允许**：若源码明显调用了第三方，在 §5 设计决策中记录"为什么选用该库"（决策理由是本工程的知识）；
- **允许**：若第三方在工程中有定制/补丁（如 `patches/xxx.diff`），在 §5 记录定制点与理由；
- **若发现 `COVERS_CODE` 中有文件路径明显属于 `EXCLUDED_PATHS`**（Stage B 分派错误），在返回值 `notes` 中标注 `unexpected_covers_code: <path>`，然后跳过该文件继续处理其余；不要私自补齐或改写 covers_code。

---

## 工作流程

### 1. 读取上下文

- 读 `META_KB_GUIDE_PATH`，确认本类型文档（module / flow / architecture）的章节规范
- 读 `DOC_PATH` 的骨架，理解需要填充哪些章节
- 读所有 `DEP_DOCS`，了解可引用的已有知识

### 2. 深度阅读 COVERS_CODE

- 逐一读完 `COVERS_CODE` 中的每个文件
- 关注：数据结构、关键函数、错误处理路径、生命周期、并发模型
- **允许**：为了理解某个符号，读 `COVERS_CODE` 之外的文件（如共享 header），但不要写入其内容
- **禁止**：把 `COVERS_CODE` 之外的文件当作本文档的核心事实来源
- **禁止**：读入 `EXCLUDED_PATHS` 下的文件——即使为了"理解上下文"

### 3. 撰写正文

**核心原则**（来自 `META_KB_GUIDE`）：
- **记录代码里读不出来的东西**：业务流程的"为什么"、设计决策的权衡、模块间隐式关联、已知陷阱
- **不做代码逐行解读**：代码是最权威的"是什么"
- **用符号名锚定代码，不用行号**：如 `` `InitializeCache()`（<file>.<ext>） `` 而非 `<file>.<ext>:123`
- **单一事实来源**：不重复 DEP_DOCS 中已有的内容，通过链接引用

**章节填充策略**：
- Module 文档：1. 职责 / 2. 数据模型 / 3. 关键 API / 4. 业务流程 / 5. 设计决策 / 6. 已知陷阱 / 7. 变更影响
- Flow 文档：1. 触发条件 / 2. 端到端时序 / 3. 涉及模块分工 / 4. 异常路径 / 5. 相关模块文档
- Architecture 文档：按 `META_KB_GUIDE` §架构文档规范

### 4. 遇到不确定时写 QA jsonl

如果你在阅读代码时遇到**无法从代码推断的问题**，追加一行到 `QA_JSONL_PATH`（每行一个 JSON 对象，符合 `schemas/qa-jsonl-entry.schema.json`）：

```json
{"round": 0, "doc_id": "<module-id>", "location": "§5 设计决策", "severity": "medium", "type": "design-rationale-unclear", "question": "<从代码推断不出的具体问题，含相关符号名>", "code_reference": "<File>.<ext>: <FunctionOrSymbol>", "status": "open"}
```

**severity 判定**（严格按此标准，不主观漂移）：

**critical**（会导致读者形成错误认知，或让基于文档的决策直接失败）：
- 声明与代码矛盾：文档说"用 RocksDB"，代码实际用 SQLite
- 关键接口签名错误
- 数据流方向错误：文档说 A → B，代码实际 B → A
- 关键前提缺失：某功能"仅 Debug 构建生效"这类必要前提未提及
- 涉及安全、并发、事务边界的错误描述

**medium**（信息不完整或误导性，但不引发直接错误决策）：
- 非核心章节缺失
- 描述过时（主流程仍能对应但细节已变）
- 关键路径的次要分支未描述
- 术语不一致
- 缺少必要的交叉引用

**minor**（不影响正确性和可读性）：
- 拼写、标点、排版
- 章节顺序、格式建议
- 可选章节缺失（如"未来优化"）
- Frontmatter 中 keywords 不影响 index.yaml 命中的补充

**判定原则**："读者按文档做决策会失败吗？"→ yes = critical。就高不就低。

### 5. 更新 index.yaml（如需）

如果你为 frontmatter 补充了新的 keywords 或调整了 description，同步更新 `.think-tank/knowledge-base/index.yaml` 中对应条目。

---

## 输出

写入 `DOC_PATH`（完整文档，含骨架 frontmatter + 你填充的正文）。

同时返回结构化摘要（简洁，供主 Agent 记入 Journal）：

```json
{
  "doc_id": "rdb-persistence",
  "doc_path": "modules/rdb-persistence.md",
  "sections_filled": ["职责", "数据模型", "关键 API", "业务流程", "设计决策", "已知陷阱", "变更影响"],
  "word_count_approx": 3500,
  "qa_entries_added": 3,
  "index_yaml_updated": true,
  "notes": "..."
}
```

---

## 硬约束

- **禁止**：分派新的 sub-agent；跨越 `COVERS_CODE` 之外做全局分析；引用行号；在 QA jsonl 里问不需要人类回答的问题（如"这个函数是干嘛的"——你自己应该读代码回答）
- **允许**：为理解上下文而读 `COVERS_CODE` 之外的文件；提问工程决策层面的问题（"这个设计是为了兼容旧版本吗？"）
- **必须**：以源代码为唯一事实来源；符号名锚定；填充所有必填章节（若确实无内容，写"待补充"并写 QA 条目）

---

## 恢复行为（若你是重试的 sub-agent）

若 `DOC_PATH` 已有部分内容（前一次尝试的产物）：
- 保留 frontmatter 和已填充的正确章节
- 补齐或修正未完成的章节
- 若发现前一次的错误内容，直接改写（无需保留错误版本）
