# {{PROJECT_SLUG}}-commander · 知识缺口反馈与回写（knowledge-writeback.md）

> 示例说明：本文中的具体命令、模块 id 与机制示例取自 Redis 8.10.2（公共领域工程），仅作教学示例，不属于模板变量；`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` 为迁移时需替换的占位符。

> 本文规定：当你在使用知识库的过程中发现**缺口、过期、矛盾、TODO**时，应该如何反馈，让知识库随使用持续生长。

---

## 1. 触发场景

凡符合下表任一情形，应在 Step 5 输出「知识回写建议」段落：

| 场景 | 典型迹象 |
|------|---------|
| **路由失败** | Step 2 阶段 1+2+3 都未命中，走兜底全仓 grep |
| **路由命中但内容缺失** | `index.yaml` 有条目但对应 md 内容明显残缺 |
| **跨模块 flow 缺失** | 涉及多模块的问题找不到对应 `flows/<id>.md`，需要拼接多份模块文档来回答 |
| **KB 与代码矛盾** | knowledge-base 描述与实际代码行为不一致（用符号名锚定的对比可反证） |
| **产品边界字段缺失** | `index.yaml.modules[<id>]` 中缺少 `responsibility` / `capability_boundary` / `communicates_with` 字段（v2.0 迁移期间尚未上浮的模块，或 modules/ 未包含的模块） |
| **产品边界字段可信度低** | `responsibility` 出现技术名词、`capability_boundary.does_not` 未标注 owner、`communicates_with` 缺方向/目的 |
| **TS 症状未覆盖** | 排障走到第四级"逆向追踪"才解决，说明该症状缺 Runbook |
| **重复信息冲突** | 同一概念在两处 KB 描述不同（违反「单一事实来源」） |

---

## 2. 反馈原则

1. **只提议，不擅自批量修改**。除非用户明确说「帮我补一下」，否则不要直接 `edit_file` 知识库文件。
2. **KB 与代码矛盾例外**：这类问题应**立即静默追加**到 `.meta-explore-issues.jsonl` 信箱（不需要用户确认），供后续 `/maintain-kb` 消费。原因：这是 {{PROJECT_NAME}} 已建立的成熟机制（见 §5）。
3. **建议必须可执行**：指明应改哪个文件、加什么字段、写什么内容，而非空泛地说"建议补充"。
4. **遵循构建规范**：所有提议的 KB 修改必须符合 `.think-tank/knowledge-base/.meta-knowledge-base-guide.md`。引用规范中的具体节号（含 §3.2 产品视角字段规范）。

---

## 3. {{PROJECT_NAME}} 的三种回写目标

{{PROJECT_NAME}} 有 `.meta-explore-issues.jsonl` 信箱机制。这让你在发现 KB 与代码不一致时有一条"零成本、自动流转"的回写通道。

| 目标 | 使用场景 | 回写方式 |
|------|---------|---------|
| **`.meta-explore-issues.jsonl`** | KB 与代码不一致（有代码可反证） | **立即静默追加**，无需用户确认 |
| **`index.yaml.modules[<id>]` 产品边界字段** | 边界字段缺失或可信度低 | 在 Step 5 提议，用户确认后再由 /maintain-kb 上浮补齐 |
| **`knowledge-base/*.md` 文档** | 缺失、过期、矛盾等结构性缺陷 | 在 Step 5 提议，用户确认后再动手 |
| **`knowledge-base/works/*.md`** | 本次 Commander 分析本身是一次值得存档的设计权衡/决策 | 在 Step 6 询问用户是否记为 Work（含决策汇总章节） |

---

## 4. 反馈条目模板

### 4.1 通用模板（用于 Step 5 提议）

每条建议作为一段独立块，使用以下统一模板：

```markdown
### 知识回写建议 #N

- **类型**：路由失败 / 内容缺失 / flow 缺失 / KB-代码矛盾 / 产品边界字段缺失 / 产品边界字段可信度低 / TS 症状未覆盖 / 重复冲突
- **触发场景**：（一句话说明本次任务中怎么发现的）
- **目标文件**：`.think-tank/knowledge-base/<path>`
- **建议内容**：
  - 字段：……
  - 值：……
  - 引用的构建规范：`.meta-knowledge-base-guide.md §X.X`（§3.2 是 index.yaml 产品视角字段规范）
- **优先级**：高 / 中 / 低（高 = 影响多次后续路由；中 = 影响特定问题域；低 = 完整性优化）
```

### 4.2 `.meta-explore-issues.jsonl` 追加格式（KB-代码矛盾专用）

信箱位置：`.think-tank/knowledge-base/.meta-explore-issues.jsonl`

每行一个 JSON 对象（严格 JSONL，一行一个，不用 `[ ... ]` 数组包裹）：

```json
{"kb_entry":"<entry-id>","section":"<§X>","note":"<差异简述>","timestamp":"<ISO 8601>"}
```

字段说明：

| 字段 | 说明 | 示例 |
|------|------|------|
| `kb_entry` | 对应 `index.yaml` 中的 `id` | `keyspace-and-expiration`（不是完整路径） |
| `section` | 文档内的章节位置 | `§4.4 复制与持久化` |
| `note` | 差异简述（代码事实 vs KB 描述） | `文档说装载期由 rdbLoadKeyValuePair() 跳过过期键，代码无此符号，跳过发生在 rdbLoadRioWithLoadingCtxInternal() 的装载循环里` |
| `timestamp` | 追加时间 ISO 8601 | `YYYY-MM-DDTHH:MM:SS+TZ` |

**追加行为**：使用 Edit / Write 追加到文件末尾（保留已有行），**不覆盖**。若文件不存在，可直接创建。

**后续消费**：`/maintain-kb` 运行时会读取信箱、验证差异、修复文档，然后 truncate 该文件。你不需要关心消费——只负责追加。

---

## 5. 各场景的具体模板

### 5.1 路由失败 → 提议补充 `index.yaml` + 建立文档

```markdown
### 知识回写建议 #1

- **类型**：路由失败
- **触发场景**：用户问「`maxmemory-policy` 设成 `allkeys-lfu` 之后淘汰行为为什么还是像 LRU」，`index.yaml.modules[memory-management-and-eviction]` 的 `keywords` 只有「内存上限 / 淘汰策略」这类上义词，没有 `maxmemory-policy`、`allkeys-lfu`、`volatile-ttl` 等具体取值，Step 2 阶段 1+2+3 均未命中，走兜底全仓 grep `evict.c`
- **目标文件 1**：`.think-tank/knowledge-base/index.yaml`
- **建议内容 1**：在 `modules.memory-management-and-eviction` 的 `keywords` 中新增 `maxmemory-policy, allkeys-lfu, volatile-lru, volatile-ttl, noeviction, maxmemory-samples`
- **目标文件 2**：`.think-tank/knowledge-base/modules/memory-management-and-eviction.md`
- **建议内容 2**：在 §4.3「淘汰候选的选择：采样、池与偏差」开头补一张策略取值对照表（策略 → 采样源 `db->keys` / `db->expires` → 是否进候选池 → 失败路径是 `EVICT_FAIL` 还是 `EVICT_RUNNING`），把目前散落在 §4.3 / §4.4 / §6 的策略语义收到一处
- **引用规范**：`.meta-knowledge-base-guide.md §4.1`（模块文档章节规范：§3 核心业务流程须最详细）
- **优先级**：高（`maxmemory-policy` 是内存类问题的首要路由词，多次路由可能命中）
```

### 5.2 flow 缺失 → 提议新增

```markdown
### 知识回写建议 #2

- **类型**：flow 缺失
- **触发场景**：用户问「主从切换全过程」，index.yaml 里 `flows/cluster-failover-and-slot-migration-flow.md` 只覆盖集群模式下的 PFAIL→FAIL 判定与槽归属搬迁，`flows/sentinel-monitoring-and-failover-flow.md` 只覆盖哨兵巡检、客观下线与领导者选举，复制拓扑下「谁把副本提成主、旧主怎么降级」没有对应的 flow 文档，需要拼接 replication / cluster / sentinel 三份模块文档加两份 flow 才能回答
- **目标文件**：`.think-tank/knowledge-base/flows/replication-failover-flow.md`（新建）
- **建议内容**：按 `.meta-knowledge-base-guide.md §5` 流程文档模板，覆盖阶段：
  1. 切换入口发起（引用 modules/sentinel.md、modules/replication.md）：`SENTINEL FAILOVER` 或哨兵自动触发，或运维直接 `REPLICAOF NO ONE` / `FAILOVER`
  2. 角色变更（引用 modules/replication.md）：`replicationUnsetMaster()` 让副本转主、旧主转副本，`INFO replication` 的 `role` 与 `master_link_status` 翻转
  3. 拓扑收敛与客户端重定向（引用 modules/cluster.md §4.4、modules/connection-and-client-io.md）：集群拓扑由槽归属搬迁承接口径，复制拓扑只需客户端重连到新主
- **同时**在 `index.yaml.flows` 段落注册新条目，`involves_modules: [replication, sentinel, cluster, connection-and-client-io, configuration]`；并在上述两份既有 flow 的「故障与异常场景」表各加一行转介本 flow，明确「哨兵负责判定与组织切换 / 集群负责槽归属搬迁 / 复制负责角色变更」的边界
- **引用规范**：`.meta-knowledge-base-guide.md §5, §3.1`
- **优先级**：高
```

### 5.3 KB 与代码矛盾 → **静默追加到信箱**（不需要用户确认）

**行为**：使用 Edit 或 Write 直接向 `.meta-explore-issues.jsonl` 追加。示例：

```json
{"kb_entry":"keyspace-and-expiration","section":"§4.4 复制与持久化：为什么传播的是 PEXPIREAT 与 DEL","note":"文档称 RDB 装载期由 rdbLoadKeyValuePair() 就地丢弃已过期键，但代码中无此符号：装载路径是 rdbLoadRioWithLoadingCtxInternal() 循环调用 rdbLoadObject()，过期键跳过与 rdb_last_load_keys_expired++ 发生在 rdb.c:4998 的装载循环内","timestamp":"YYYY-MM-DDTHH:MM:SS+TZ"}
```

**回答中的提示**（可选，一句话）：

> 已发现 `keyspace-and-expiration.md §4.4` 与代码不一致（RDB 装载期跳过过期键的函数名），已静默追加到 `.meta-explore-issues.jsonl` 信箱，下次运行 `/maintain-kb` 会自动修复。

### 5.4 产品边界字段缺失 → 提议补齐

```markdown
### 知识回写建议 #3

- **类型**：产品边界字段缺失
- **触发场景**：本次分析要判断「槽内数据落盘」与「键到期判定」各自归谁，但 `index.yaml.modules[cluster].capability_boundary.does_not` 只有 replication / sentinel 两项，未覆盖这两类行为，影响链推理从 `cluster` 出发到这里断掉（阻塞路由阶段 2 的边界判断）
- **目标文件**：`.think-tank/knowledge-base/index.yaml`
- **建议内容**：在 `modules.cluster` 条目的 `capability_boundary.does_not` 下追加两条，从 `modules/cluster.md` §1.2 职责边界、§7 变更影响评估上浮，参考 `.meta-knowledge-base-guide.md §3.2` 的字段规范（每条 `does_not` 必须标注 owner）：
  - `{task: "把槽内数据写入持久化文件", owner: rdb-persistence}`
  - `{task: "判定键是否到期并在到期后清除", owner: keyspace-and-expiration}`
- **引用规范**：`.meta-knowledge-base-guide.md §3.2`
- **优先级**：中（`capability_boundary` 是影响链分析的入口，缺项会让归属判断落到人工 grep）
```

### 5.5 产品边界字段可信度低 → 提议修订

```markdown
### 知识回写建议 #4

- **类型**：产品边界字段可信度低
- **触发场景**：`index.yaml.modules[cluster].communicates_with` 只有 3 条，与对端条目不对称：`modules.observability-and-introspection.communicates_with` 已声明「cluster / 被调用 / 提供分片维度的统计与节点信息」，`cluster` 侧没有回指；`cluster-node-timeout`、`cluster-require-full-coverage` 等设置也说明与 `configuration` 有边却未列出。按 §3.3 自检「三要素完整」通过、「边完整」不通过，从 `cluster` 出发的影响链会漏边
- **目标文件**：`.think-tank/knowledge-base/index.yaml`
- **建议内容**：为 `modules.cluster.communicates_with` 补齐对端条目，例如：
  - `{module: observability-and-introspection, direction: 调用, purpose: 汇总分片与槽位统计以供运行信息查询}`
  - `{module: configuration, direction: 调用, purpose: 读取集群超时与全覆盖率等设置}`
  - 若对端已声明而本条目缺失，逐条补齐
- **引用规范**：`.meta-knowledge-base-guide.md §3.2, §3.3`（可信度自检）
- **优先级**：中
```

### 5.6 TS 症状未覆盖 → 提议创建 Runbook

```markdown
### 知识回写建议 #5

- **类型**：TS 症状未覆盖
- **触发场景**：用户报告「`BACKUP SEAL` 之后 `appendonlydir` 里多了一个 INCR 文件，怀疑备份把线上 AOF 写坏了」，`troubleshooting/symptom-index.yaml` 现有 12 条症状（write-fails-read-succeeds … command-latency-high）没有一条命中该现象，TS 四级检索链走到第四级逆向追踪 `aof.c` 才定位到 `openNewIncrAofForAppend()` 的刻意轮转
- **目标文件 1**：`.think-tank/knowledge-base/troubleshooting/incidents/2026-10-21_backup-seal-rotates-aof.yaml`（新建）
- **建议内容 1**：按 `troubleshooting/incidents/.meta-incident-guide.md §3.1` 完整模板记录本次排查过程（症状、诊断步骤、根因、解决方案、关联模块），证据层取 server-log + metrics（`INFO persistence` 的 `backup_in_progress`）
- **目标文件 2**（后续）：先在 `troubleshooting/symptom-index.yaml` 登记症状 `backup-seal-rotates-aof`（`deploy_topo: any`，`likely_modules: [aof-persistence, server-lifecycle-and-dispatch, configuration]`），再由 AI Agent 从多篇同类 Incident 转换为 `runbooks/<id>.md`
- **引用规范**：`troubleshooting/incidents/.meta-incident-guide.md §3, §7`、`.meta-knowledge-base-guide.md §11.1`
- **优先级**：高（症状驱动的排障路径没记录，下次同类问题还会走第四级）
```

---

## 6. 回写建议的"克制原则"

- **每次回答最多列 3 条**回写建议，避免淹没主回答；
- 多于 3 条时，按优先级排序，剩余的合并为「其他低优先级建议」一笔带过；
- 如果用户**没有要求**关心知识库建设，可以**省略**回写建议段落（但 **KB-代码矛盾**类必须走信箱静默追加，是零成本自动机制不受此限）；
- 永远不要**预测性**地建议未来可能需要的条目 —— 只针对**本次任务实际触发**的缺口提建议。
- **信箱追加不算入 3 条上限**，因为它是零打扰、异步消费的机制。

---

## 7. 用户授权后的执行规则

如果用户明确说「帮我把这些建议落地」/「直接补一下」：

1. **逐条确认**：把建议条目复述一遍，让用户对范围有数；
2. **小批量提交**：一次最多落地 2~3 条，避免大改难审；
3. **严格遵循构建规范**：
   - 修改 `knowledge-base/` 下的文档 → 引用 `.meta-knowledge-base-guide.md` 具体节号；
   - 修改 `index.yaml.modules[<id>]` 的产品视角字段 → 引用 `.meta-knowledge-base-guide.md §3.2, §3.3` 字段规范与可信度自检；
4. **改完后建议运行自检**：
   - 检查 `index.yaml` 是否有新增条目的 `keywords`、`description`、`covers_code`；
   - 检查 `modules[<id>]` 的 `capability_boundary.does_not` 每条是否有 `owner`；`communicates_with` 每条是否三要素完整；
   - 建议用户运行 `/maintain-kb` 做一次增量检查确认无残留问题。
5. **不要越界写代码**：Commander 的回写职责仅限 KB 目录，不改工程源码。

---

## 8. 回写质量自检表

提议条目前内部走一遍：

| 检查项 | 通过条件 |
|-------|---------|
| 是否符合"单一事实来源"原则 | 同一信息没有在多处重复描述 |
| 是否引用了构建规范具体节号 | 提议必须有规范依据 |
| 优先级是否合理 | 高优先级仅给"影响多次路由 / 关乎正确性"的项 |
| 是否只针对本次任务触发的缺口 | 不做预测性建议 |
| KB-代码矛盾是否走了信箱通道 | 这类问题应静默追加而非提议 |
| 是否只提议不擅自修改（除信箱外） | 除非用户授权 |
| 引用的文件路径是否真实存在 | 目标文件位置准确 |

---

## 9. 与 explore-kb / maintain-kb 的分工

{{PROJECT_NAME}} 有一套完整的 KB 维护三方协作机制：

| 角色 | 职责 | 与本 skill 的关系 |
|------|------|------------------|
| `{{PROJECT_SLUG}}-commander`（本 skill） | 使用 KB 做产品级决策，发现 KB 缺口时提议或追加信箱 | 生产者：向信箱写入 KB-代码不一致 |
| `explore-kb` | 使用 KB 检索答案，发现 KB 与代码不一致时静默追加信箱 | 平级生产者：同样向信箱写入 |
| `maintain-kb` | 消费信箱 + git diff + 全量遍历，系统性修复 KB | 消费者：读取信箱后 truncate |

**你的定位**：Commander 是**决策使用者**，不是维护者。你的回写职责是"发现并记录"，不是"批量修复"。系统性修复交给 `/maintain-kb`。
