# {{PROJECT_SLUG}}-commander · 上下文预算与最小阅读集（context-budget.md）

> 示例说明：本文中的具体命令、模块 id 与机制示例取自 Redis 8.10.2（公共领域工程），仅作教学示例，不属于模板变量；`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` 为迁移时需替换的占位符。

> 本文展开 `SKILL.md` §2，约束你在每次任务中**读什么、不读什么、什么时候读**，避免上下文被无关文档塞满。

---

## 1. 三档阅读清单

### 🟢 必读（Must-read，每次 {{PROJECT_NAME}} 任务都加载）

- `.think-tank/knowledge-base/index.yaml`

> 仅这一个文件。它体量小、信息密度高，是路由的基础。
>
> **v2.0 起**：产品视角边界（`responsibility` / `capability_boundary` / `communicates_with`）已内嵌于 index.yaml.modules[]。Step 0 加载 index.yaml 即得所有路由信息 + 模块边界信息，无需再单独加载 module_header 文件。

### 🟡 按需读（On-demand，按路由结果加载）

- 命中的 `.think-tank/knowledge-base/modules/<id>.md`（Step 3 深度下钻时）
- 命中的 `.think-tank/knowledge-base/flows/<id>.md`（跨模块场景时）
- `.think-tank/knowledge-base/architecture/*.md`（阶段 3 拓扑反查时）
- `.think-tank/knowledge-base/troubleshooting/symptom-index.yaml`（故障排查场景第一级）
- 命中的 `.think-tank/knowledge-base/troubleshooting/runbooks/<id>.md`
- `.think-tank/knowledge-base/troubleshooting/faq.md` / `known-bugs.md`（故障排查第二/三级）

### 🔴 禁读（Forbidden，未授权不得加载）

- 一次性 `read_file` `knowledge-base/modules/` 或 `flows/` 内的全部 md；
- 在没有路由命中的前提下浏览 `src/` / `tests/` / `deps/` 等源码目录（`deps/` 是 vendored 第三方源码，KB 不覆盖，不进任何阅读路径）；
- 把整个大文件（> 500 行）`read_file` 全文（应使用 offset/limit + grep 精确定位）；
- 一开局把 `architecture/` 下所有文档都读一遍（属于全景预热反模式）。

---

## 2. 「按需读」的进入条件

| 文件类型 | 何时允许加载 |
|---------|-------------|
| `modules/<id>.md` | 需深入到数据模型/API/流程/陷阱层面，且 profile 允许（`depth.level >= architecture`） |
| `flows/<id>.md` | 子问题为跨模块类型，或路由命中 `flows.<id>` |
| `troubleshooting/symptom-index.yaml` | 用户描述故障症状（走 TS 四级链第一级） |
| `troubleshooting/runbooks/<id>.md` | symptom-index 精确匹配到该 Runbook |
| `troubleshooting/faq.md` / `known-bugs.md` | TS 四级链第二/三级 |
| `troubleshooting/incidents/*.yaml` | 需要参考历史事件时（不入索引，需明确文件名） |
| `architecture/system-overview.md` / `data-model-relations.md` | 阶段 3 拓扑反查，或用户问全景问题 |
| `architecture/tech-stack.md` | 用户问技术栈 / 编码约定 / 依赖 |
| `architecture/deployment-paths.md` | 运维场景需要路径信息 |
| 模块源码 | 满足 `SKILL.md §3` 的全部 5 条触发条件（含 profile 允许） |
| `works/*.md` | 该 Work 文档记录了当前问题相关的历史决策与设计权衡 |

---

## 3. 单次任务的上下文预算指引

> 这是**软约束**，用于自我节制；硬约束是上面的禁读清单。

| 阶段 | 推荐加载量上限 |
|------|---------------|
| Step 0（必读） | 1 个文件（index.yaml，已含所有产品边界字段） |
| Step 2（路由辅助） | 0~1 个文件（architecture 按需，通常不需要） |
| Step 3（每个模块） | modules/<id>.md 1 份 + flows/<id>.md（涉及跨模块时）1 份 |
| 源码下钻 | grep 精确符号 + read 命中片段（不超过 3 个文件 / 600 行） |
| TS 四级链单次 | symptom-index.yaml + 命中的 1 份 Runbook（若命中则不进入后续级别） |

> 经验阈值：**单次任务总加载文件数尽量控制在 8 个以内**（v2.0 起因合并了 module_header，从 10 个降至 8 个）。超过这个数量，意味着路由失败或问题过宽，应回到 Step 1 重新切割。

### 示例工程的预算参考点（Redis 8.10.2）

> 以下数字全部是示例工程 Redis 8.10.2 的实测值，只用来说明量级与读法；迁移到 {{PROJECT_NAME}} 后以本工程 KB 的实际规模为准，阅读阈值口径不变。

- **读一次 index.yaml 就能拿到全部产品边界字段**：该文件 2,231 行，`modules[]` 下 31 个模块条目各自内嵌 `responsibility` / `capability_boundary` / `communicates_with`。Step 0 一次加载即得全景 + 边界，不要再去找 module_header 文件；之后除路由外不需要再碰 index.yaml。
- **`knowledge-base/modules/` 共 31 份模块文档、合计约 1.58 万行**：最大的 `cluster.md` 734 行、`core-data-structures.md` 680 行、`object-model.md` 609 行——批量读一轮就吃掉整个预算，必须按路由只取命中的那 1 份。
- **`flows/` 只有 7 份流程文档，但单份 217–385 行**（`replication-sync-flow.md` 385 行最长）：Mermaid + 步骤表都在里面，读一份成本不低，非跨模块场景不要顺手加载。
- **`architecture/` 6 份文档、单份 308–452 行**：`system-overview.md`（308 行）是全景入口，`command-contract.md` / `data-model-relations.md` 按需，其余只在阶段 3 拓扑反查时才用。
- **源码规模**：`src/` 下 134 个 `.c` + 84 个 `.h`，合计约 21 万行；单文件最大的 `src/module.c` 16,084 行、`src/redis-cli.c` 11,533 行、`src/server.c` 8,415 行、`src/server.h` 4,893 行。"读一下 server.c"这种动作必然爆预算，只能 grep 符号后局部读。
- **命令元数据**：`src/commands/` 下 460 个文件（459 份 `*.json` + 1 份 README），是命令契约的唯一源；查单个命令的 arity / flags / 复杂度，优先读对应 JSON，不要转去翻 `src/commands*.def`（生成物）或 server.c 里的命令表。
- **`tests/` 有 231 个 `.tcl`**（测试即契约），体积 4.8M：只有在需要确认边界行为时才定点读某个 `tests/unit/*.tcl`。
- **`deps/` 是 vendored 第三方**（jemalloc、Lua、hiredis、linenoise、hdr_histogram、fpconv、tre、xxhash，约 14 万行）：KB 明确排除，不要读。
- **CLAUDE.md** 已由主 Agent 加载到上下文中，不需要再 read_file。

---

## 4. 大文件读取规则

- 文件 ≤ 200 行：可全文 `read_file`；
- 200 < 行数 ≤ 500：使用默认 limit；
- > 500 行：必须先 `grep_search` 精确定位行号 → 再 `read_file` 用 offset/limit 读取命中段落 ± 30 行；
- 永远不要为了"理解上下文"通读 > 1000 行的文件。

**示例工程已知的大文件参考（Redis 8.10.2）**：
- `.think-tank/knowledge-base/.meta-knowledge-base-guide.md` 809 行 → 用 grep 定位后局部读；
- 单份 > 500 行的 `modules/*.md`（`cluster.md` 734、`core-data-structures.md` 680、`object-model.md` 609、`observability-and-introspection.md` 608、`module-api.md` 602 行）→ 先看 frontmatter + 章节标题再局部读；
- 源码中大文件（`src/module.c` 16k、`src/redis-cli.c` 11.5k、`src/server.c` 8.4k、`src/cluster_legacy.c` 6.7k、`src/t_hash.c` 6.6k、`src/t_stream.c` 6.4k、`src/networking.c` 6.0k 行）若下钻代码 → 必须 grep + offset/limit。

---

## 5. 何时主动放弃加载更多

出现以下信号时，**立即停止加载新文件**，进入推理与归纳：

1. 已能完整回答 Step 1 产出表中所有子问题的「期望产出」；
2. 新加载的文档与已有信息高度重复，边际信息接近 0；
3. 上下文使用已接近预算上限，但仍需补充小细节 —— 此时应**降低置信度声明**，不要继续硬塞；
4. 当前 profile 是 `product-view` / `user-guide`，且已从 index.yaml 的产品边界字段获得答案 —— 不应再进入 knowledge-base/modules。

---

## 6. 当上下文不够时的优先级排序

如果面临"必须放弃一些已加载内容"的处境，按以下顺序丢弃：

```
最先丢：架构反查类文档（architecture / dependencies，已用完即可丢）
       ↓
       已读完并提炼出小结的 modules 文档（保留小结即可）
       ↓
       历史决策文件（除非贯穿整个回答）
       ↓
       flow 文档（一份就是端到端骨架，通常整段保留）
       ↓
最后丢：index.yaml（永不丢——它是产品边界字段和所有索引的宿主）
```

---

## 7. 反模式（典型上下文浪费场景）

| 反模式 | 描述 | 正确做法 |
|-------|------|---------|
| **「全景预热」** | 一上来把所有 `modules/*.md` 或 `architecture/*.md` 都读一遍 | 只读 index.yaml，按路由结果按需加载 |
| **「以防万一」** | 怕漏掉就把可能相关的 5 个模块文档都读了 | 严格遵循"单模块 1 份文档"约束，用 grep 精确定位 |
| **「代码即真相」** | 跳过 KB 直接 grep 代码 | 满足 §3 触发条件才允许下钻代码 |
| **「全文通读」** | 大文件不用 offset/limit，整个 read | 先 grep 精确行号再 read 局部 |
| **「重复加载」** | 同一文件多次 `read_file` | 加载一次后用内存中已知内容推理 |
| **「症状直读代码」** | 用户说"attach 失败"就去 grep Redis CLI 源码 | 必须先走 TS 四级检索链 |
| **「找 module_header 文件」**（v1.x 遗留习惯） | 试图 read `.think-tank/commander/module_header_*.md` | 该目录 v2.0 已删除。产品边界字段在 index.yaml.modules[<id>] 中，Step 0 已加载 |
