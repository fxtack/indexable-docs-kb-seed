# {{PROJECT_SLUG}} · 工作流详解（workflow.md）

> 示例说明：本文中的具体命令、模块 id 与机制示例取自 Redis 8.10.2（公共领域工程），仅作教学示例，不属于模板变量；`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` 为迁移时需替换的占位符。

> 本文是 `SKILL.md` §1 中 6 步流程的展开版，配 5 意图对应的输出模板与完整示例。
> 在执行任何 {{PROJECT_NAME}} 任务时按需查阅。

---

## Step 0 · 前置加载

### 操作清单

1. `read_file .think-tank/knowledge-base/index.yaml`（必读，v2.0 起产品视角边界字段已内嵌于 modules[] 条目中）；
2. 抽取用户问题中的所有名词 / 缩写 / 别名 / 命令名 / 文件路径；
3. 对每一个词，在 `index.yaml` 各段（`architecture` / `modules` / `flows` / `troubleshooting` / `develop` / `works`）的 `keywords` / `covers_code` / `covers_commands` 中模糊匹配；
4. 若关键词未命中任何条目，标记为「未覆盖关键词」，后续推理中谨慎使用；
5. **不要**尝试查找或加载 `.think-tank/commander/` 下的任何文件（v2.0 已移除该目录，产品边界信息全部内嵌于 index.yaml）。

### 产出模板（内部，可在回答中折叠展示）

```text
[前置加载]
- 用户词「从库」              → 命中 modules.replication keywords（主从 / 复制）（index.yaml，已覆盖）
- 用户词「读到旧值」          → 命中 troubleshooting/symptom-index.yaml 的 replica-data-lagging trigger_keywords（已覆盖，runbook: TBD）
- 用户词「SLOWLOG」           → 命中 modules.observability-and-introspection covers_commands（已覆盖）
- 用户词「K8s Operator 的漂移策略」→ 未覆盖关键词（index.yaml 无对应条目）
```

### 失败兜底

- 不要因为关键词未覆盖就拒绝回答，标记为「未覆盖」继续推理即可。

---

## Step 0.5 · 回答策略分析

详细规则见 `answer-strategy.md`。本节仅强调执行要点：

### 执行清单（内部执行，不输出）

1. 打开 `answer-strategy.md`；
2. 检测用户问题中的角色提示词 → 确定 persona + persona_source；
3. 意图特征词打分 → 确定 intent + intent_confidence；
4. 按 §4.2 收敛规则确定 strategy + attach_clarification；
5. 加载对应 profile（§5 中 6 个之一）；
6. 后续 Step 1~5 按 §6 约束映射执行。

### 内部产出（不外露）

```text
[策略分析]
- persona: developer / persona_source: derived
- intent: troubleshoot / confidence: high（命中 "报错" +1, "日志" +1, "排查" +1）
- strategy: deep-dive
- attach_clarification: true
- profile 加载：answer-strategy.md §5.2
```

**输出禁令**：以上信息严禁出现在最终回答中。

---

## Step 1 · 问题切割

### 切割原则

- **每个子问题对应一个最小可路由的语义单元**，不要太粗（路由不到具体模块）也不要太细（无意义碎片）；
- 切割粒度参考：「能否在 `index.yaml.modules[]` 中找到归属（通过 keywords 或产品边界字段）」；
- 跨多模块的端到端问题**单独切出来**，归类为 cross-module。

### 强制产出表（受 `profile.structure.problem_breakdown_table` 约束）

| # | 子问题 | 命中关键词 | 候选负责模块 | 期望产出 | 是否跨模块 |
|---|--------|-----------|-------------|---------|-----------|
| 1 | ... | ... | modules.xxx | 结论 / 数据 / 调用链 / 排查路径 | N |
| 2 | ... | ... | cross-module | ... | Y |

**字段说明**：
- *期望产出*：必须是可验证的产物（一段结论 / 一个调用链 / 一个数据结构 / 一个排查步骤列表），不允许写"了解一下"这种含糊词；
- *是否跨模块*：标 Y 的子问题在 Step 4 必须做一致性检查。

### 产出表的用途

- 后续每个 Step 都对照此表逐行收敛；
- 避免遗漏子问题；
- 便于在 Step 5 归纳时按行汇总。

**注意**：若 `profile.structure.problem_breakdown_table == false`（`ops-detail` / `qa-view` / `product-view` / `user-guide` / `default`），本表仍在**内部**执行（确保路由不丢），只是不输出到回答中。

---

## Step 2 · 路由

详细规则见 `routing-rules.md`。这里仅强调产出格式：

| # | 子问题 | 路由结果 | 路由来源 | 命中条目 |
|---|--------|---------|---------|---------|
| 1 | ... | modules.replication | index.yaml keywords | id=replication |
| 2 | ... | flows.replication-sync-flow | index.yaml keywords + involves_modules | id=replication-sync-flow |
| 3 | ... | TS 四级：symptom-index → runbooks/xxx.md | trigger_keywords="从库读不到刚写的数据" | runbook_id=replica-data-lagging（当前 runbook: TBD，落到第四级逆向追踪） |

每个子问题必须有明确的**路由结果 + 来源**，便于回溯与修订。

**注意**：路由表默认不外露，受 `profile.structure.routing_table` 约束。只在 `ops-detail` 等策略下才输出到回答。

---

## Step 3 · 深度下钻

### 下钻顺序（每个模块独立执行）

1. 从 `index.yaml.modules[<id>]` 中取产品视角边界字段（`responsibility` / `capability_boundary` / `communicates_with`）——**已在 Step 0 加载，无需额外 read_file**；
2. 判断当前 profile 的 `depth.level` 是否允许继续深入：
   - `capability` / `api-surface` → 止于 index.yaml 产品边界字段；
   - `architecture` → 允许读 `knowledge-base/modules/<id>.md` 和 `flows/<id>.md`；
   - `code` → 允许在满足 `SKILL.md §3` 全部触发条件时读源码；
3. 若允许，`read_file` 命中的 1~2 个具体文档（不是全部！）；
4. 若仍未解答且 profile 允许，进入「读代码」判定（见 SKILL.md §3）。

### 下钻产出（每个模块一份小结）

```text
[模块 replication 下钻小结]
- 已读文档：index.yaml（含 modules[replication] 产品边界字段）, modules/replication.md, flows/replication-sync-flow.md
- 关键事实：
  1. 副本侧 `server.masterhost` 非空后，角色与链路状态由 `server.repl_state`（`REPL_STATE_*`）表达；不在 `CONNECTED` 时仍可服务，取决于 `replica-serve-stale-data`
  2. 变更传播的唯一入口是 `replicationFeedSlaves()`，多副本共享 `repl_backlog` 的同一批字节块；可续传窗口见 §2.2
  3. `WAIT` 的 GETACK 由 `beforeSleep()` 里的 `sendGetackToReplicas()` 发布，与命令路径的传播共用同一条偏移轴
- 残留疑点：副本 ACK 由主线程还是 I/O 线程发出，取决于 master client 归属（`master_client_io_thread`），文档只到“归属决定者”这一层
```

### 严禁

- 一次 `read_file` 多个模块的详细文档"以防万一"；
- 在 knowledge-base 内做全文 `grep_search` 而不先看 index；
- 把模块内部术语（`repl_backlog` 的共享块与引用计数、`CLIENT_MASTER` 的只读豁免、`beforeSleep()` 的刷出时机）未做解释就抛回给非研发用户。

---

## Step 4 · 跨模块一致性检查

### 触发条件

当 Step 1 产出表中存在「跨模块 = Y」的行，或 Step 3 多个模块的小结涉及同一数据 / 流程时。

### 检查动作

1. 优先读 `flows/<id>.md`（若存在），它给出端到端骨架；
2. 对每个跨模块子问题，比对各模块小结中：
   - **数据格式 / 结构**是否一致（字段名、类型、序列化格式；如 `INFO replication` 的 `slaveN:offset` 与 `CLIENT LIST` 的 `omem` 是否被当成同一个量）；
   - **调用方向 / 协议**是否一致（谁调谁、同步 / 异步；如变更方向是主 → 从的字节流，而 ACK 方向是从 → 主）；
   - **错误码族与判定位置**是否对齐（`-MASTERDOWN` / `-READONLY` / `-MISCONF` 各自落在 `processCommand()` 的哪一段）；
   - **偏移语义**是否一致（`master_repl_offset`、`repl_ack_off`、`psync_offset` 一律遵循“下一个待发字节”的 +1 约定）；
3. 若发现矛盾，**不要私自调和**，而是：
   - 在最终回答中显式列出冲突；
   - 按 `knowledge-writeback.md` 提议补充 flow 文档或修订冲突方。

### {{PROJECT_NAME}} 常见的一致性陷阱

- **replication ↔ connection-and-client-io**：副本连接的输出缓冲与 backlog 共享同一批字节块，`INFO replication` 的 `slaveN:offset` 与 `CLIENT LIST TYPE replica` 的 `obl`/`omem` 是同一件事的两个视角；ACK 时间戳在 I/O 线程上先写 `io_repl_ack_time`，回到主线程才提升为 `repl_ack_time`；
- **keyspace-and-expiration ↔ replication**：过期从不作为事件被复制，链上只有 `PEXPIREAT` 与 `DEL`/`UNLINK`；副本读到逻辑上已过期的键返回“不存在”但不删键，于是它的 `DBSIZE`/`INFO keyspace` 与可见性会不一致；
- **observability-and-introspection ↔ replication**：`WAIT`/`WAITAOF` 在 index.yaml 的 `covers_commands` 里列在 keyspace-and-expiration，而 ACK 计数与阻塞解除的机制在 `modules/replication.md`，引用时以模块文档为准，不按 `covers_commands` 反推实现归属。

---

## Step 5 · 产品视角归纳

### 必含内容（受 profile 约束）

1. **回答归纳**：对照 Step 1 产出表逐行回答用户问题，再做总收口。语气 / 详细程度 / 术语 / 类比使用受 profile 相应字段约束。

2. **置信度声明**（受 `profile.structure.confidence_matrix` 约束）：

   | 维度 | 说明 |
   |------|------|
   | 主要依据 | KB（具体到文件） / 代码（具体到 `<file>: <func>`） / 推理 |
   | 降低置信度的因素 | index.yaml 产品边界字段缺失 / KB 与代码冲突 / 仅基于 README 推断 / 用户描述歧义 |
   | 总体置信度 | 高 / 中 / 低 |

3. **知识回写建议**（可选，最多 3 条）：
   - 路由 TODO 项可补全的；
   - 术语遗漏；
   - flow 缺失；
   - KB-代码矛盾（这类应静默追加信箱，不占 3 条名额）；
   - 产品边界字段缺失或可信度低。
   按 `knowledge-writeback.md` 模板列出，**不要擅自批量改文件**，除非用户明确授权。

---

## Step 6 · 分析报告 / 决策记录生成（按需触发）

### 硬性规则

**必须先询问，得到人类明确确认后才能生成文件。严禁在未确认的情况下主动创建持久化文件。**

### 触发条件

用户的问题属于**需求分析、方案设计、跨模块变更、架构评审、排障新场景**等需要后续实施或存档的场景时。

> 若用户问题属于简单问答（如"xxx 是什么"、"xxx 在哪里实现"），无需触发 Step 6。

### 三个方向 → 三种产物

| 意图 | 触发场景示例 | 产物类型 | 目标位置 | 模板 |
|------|-----------|---------|---------|------|
| **implement** | "帮我拆一下这个功能的任务" | Work 文档 | `.think-tank/knowledge-base/works/<slug>.md` | `works/.meta-work-guide.md` |
| **evaluate** | "评估这个跨模块改动" / "方案 A vs B"（含架构决策/技术选型/协议变更） | Work 文档（在 Work 中承载决策汇总，v2.0.2 起独立 decisions/ 目录已废除） | 同上 | 同上 |
| **troubleshoot**（新症状） | 用户报告 TS 未覆盖的症状且已解决 | Incident Record | `.think-tank/knowledge-base/troubleshooting/incidents/YYYY-MM-DD_<slug>.yaml` | `incidents/.meta-incident-guide.md` |

### 询问 → 确认 → 生成 流程

1. Step 5 回答完成后，使用 `AskUserQuestion` 询问："是否需要将本次分析记录为 XXX？"（XXX 替换为具体产物类型）；
2. **等待用户明确回复肯定答复**（"是"/"需要"/"记录一下"等）；
3. 用户确认后，才执行生成动作；
4. 若用户回复否定或未回应，**禁止生成**。

### 生成动作要点

- **Work 文档**：先读 `works/.meta-work-guide.md` §4 必选章节，按模板填写。frontmatter 中 `status: draft`。同时在 `index.yaml.works` 段落新增条目。若涉及架构决策/技术选型/跨模块协议变更，在 Work 文档的"决策汇总"章节承载，无需另开单独的决策文档（v2.0.2 起 `decisions/` 目录已废除）；
- **Incident Record**：按 `incidents/.meta-incident-guide.md` §3.1 完整模板，YAML 格式，记录症状、诊断步骤（含弯路）、根因、恢复方案、关联模块。

---

## 5 意图对应的输出格式

Step 5 的回答归纳，按用户意图有不同的推荐结构。**注意**：以下模板受 profile 约束——`product-view` / `user-guide` 应大幅精简，`deep-dive` 可完整展开。

### understand（理解型问题）

```markdown
## <一句话概括>

<段落 1：从产品视角描述现象/机制/原理>

<段落 2：涉及的模块或数据流（若 profile 允许暴露模块名）>

### 关键点
- 要点 1（用符号名锚定代码）
- 要点 2
- 要点 3

### 相关文档（可选）
- [<title>](.think-tank/knowledge-base/modules/<id>.md)
- [<title>](.think-tank/knowledge-base/flows/<id>.md)
```

### troubleshoot（排障型问题）

**优先走 TS Runbook 决策树**。若命中 Runbook，直接引用 Runbook 结构：

```markdown
## 症状确认
<用户描述归入的症状类别，引用 symptom-index.yaml 中的条目>

## 排查步骤
1. **检查 X**（引用 Runbook §1.x）
   ```shell
   <具体命令>
   ```
   - 若输出 A → 跳到步骤 2
   - 若输出 B → 根因确认为 Y，进入恢复步骤

2. **检查 Z**（引用 Runbook §2.x）
   ...

## 恢复步骤
1. <操作 1>
   ```shell
   <命令>
   ```
2. <操作 2>
3. **恢复后验证**：<验证方式>

## 相关模块（若 profile 允许）
- <模块> §6 已知陷阱条目引用
```

若未命中 Runbook（走 TS 第二/三/四级），最后必须提示：

> 此症状尚未被 TS Runbook 覆盖。建议创建 Incident Record（`troubleshooting/incidents/YYYY-MM-DD_<slug>.yaml`，模板参考 `.meta-incident-guide.md`）。是否需要我帮你起草？

### evaluate（评估型问题）

```markdown
## 问题/需求摘要
<一句话>

## 直接涉及模块
- **<模块>**：为什么涉及（依据哪个 index.yaml.modules 边界字段 / knowledge-base 章节）

## 影响链扩散
- **<模块>** ← 影响来源：<传导路径>
- <若无扩散影响，明确说明"影响链在此终止">

## 各模块变更目标
| 模块 | 变更目标（做到什么，不是怎么做） | 优先级 | 备注 |
|------|-------------------------------|--------|------|
| ... | ... | P0/P1/P2 | ... |

## 实施风险
- 风险 1（例：变更涉及持久化格式，升级后旧数据需仍可读）
- 风险 2

## 决策记录（可选）
若涉及多方案：

| 维度 | 方案 A | 方案 B | Commander 评估 |
|------|--------|--------|---------------|
| 职责边界合规性 | ... | ... | ... |
| 跨模块影响范围 | ... | ... | ... |
| 升级兼容性 | ... | ... | ... |
| 实施复杂度 | ... | ... | ... |
| 与已有设计决策的冲突 | ... | ... | ... |

## 相关文档
- [<title>](...)
```

### implement（实施型问题）

```markdown
## 需求归属
- **主责模块**：<模块>，理由：index.yaml.modules[<id>].responsibility 覆盖 <某职责>
- **候选二选**：若语义模糊，列出 2-3 个候选并说明差异

## 任务拆解
| 模块 | 技术目标 | 前置依赖 | 优先级 |
|------|---------|---------|--------|
| ... | ... | ... | P0/P1 |

## 实施顺序
1. Phase 1：<前置模块> — 先做什么
2. Phase 2：<主责模块> — 主体实施
3. Phase 3：<下游模块> — 同步适配

## 风险点
- <风险 1>
- <风险 2>

## 相关文档
- 编码规范：[<title>](.think-tank/knowledge-base/develop/...)
- 类似历史 Work（含决策汇总）：[<title>](.think-tank/knowledge-base/works/...)
```

### operate（运维型问题）

```markdown
## 操作上下文
<当前部署形态（单机 / 主从 / Cluster / Sentinel）、链路是否在同步中、是否有正在进行的 AOF 重写或 RDB 保存>

## 前置检查
1. <检查 1>
   ```shell
   <命令>
   ```
   预期输出：<...>

2. <检查 2>

## 操作步骤
1. <步骤 1>
   ```shell
   <命令>
   ```
2. <步骤 2>

## 验证
1. <验证方式 1>
2. <验证方式 2>

## 回滚预案
若失败：
1. <回滚操作>
2. <回滚验证>

## 风险提示
- ⚠️ <风险 1>
- ⚠️ <风险 2>

## 日志排查路径（若操作失败）
- <log path>
- <log path>
```

---

## 完整示例（troubleshoot 场景）

（本节约 120 行为完整教学示例，迁移时可直接替换为其他工程的等价场景。）

> 用户问："我在主实例上写入返回 OK，但另一条连接（连的是从库）读同一个 key 还是旧值。怎么排查？"
> 假设：无角色提示，会话中无偏好角色；部署形态为主从。

### Step 0 前置加载

```text
[前置加载]
- 用户词「从库 / 副本」        → 命中 modules.replication keywords（复制 / 主从 / 确认进度）与 flows.replication-sync-flow keywords（主从复制 / 端到端）（已覆盖）
- 用户词「读到旧值」          → 命中 troubleshooting/symptom-index.yaml 的 replica-data-lagging trigger_keywords「从库读不到刚写的数据」（deploy_topo=replication，runbook: TBD）（已覆盖）
- 用户词「另一条连接」        → 命中 modules.connection-and-client-io keywords（客户端 / 连接）与 covers_commands（CLIENT LIST / SELECT）（已覆盖）
- 用户词「INFO / SLOWLOG」    → 命中 modules.observability-and-introspection keywords（运行指标 / 慢操作记录）与 covers_commands（INFO / SLOWLOG）（已覆盖）
- 用户词「key 过期」          → 命中 modules.keyspace-and-expiration keywords（到期时间 / 键查找）与 covers_commands（TTL / EXISTS）（已覆盖）
- 用户词「K8s Operator 的漂移策略」→ 未覆盖关键词（index.yaml 只覆盖部署形态与高可用部件，不含外部编排器行为）
```

### Step 0.5 策略分析（内部）

```text
[策略分析]
- persona: developer（intent 推导默认）
- persona_source: derived
- intent: troubleshoot / confidence: high（"旧值"+"读不到"+"怎么排查"）
- strategy: deep-dive
- attach_clarification: true（persona_source=derived）
- profile: answer-strategy.md §5.2
```

### Step 1 问题切割（内部产出，dev profile 会输出到回答）

| # | 子问题 | 关键词 | 候选 | 期望产出 | 跨模块 |
|---|--------|-------|------|---------|-------|
| 1 | 这次写入真的落在"读的那个库/键"上了吗 | 写 OK、旧值、key 过期 | modules.keyspace-and-expiration, modules.connection-and-client-io | 命令序列 + 判定结论 | N |
| 2 | "另一条连接"连的是谁、读的是哪个 db | 另一条连接、库号 | modules.connection-and-client-io | `CLIENT LIST` 证据 + 库隔离结论 | Y |
| 3 | 若读的是副本：落后多少、链路是否已经断了还在服务旧数据 | 从库、旧值 | modules.replication | 字节差 / `lag` / `master_link_status` | Y |
| 4 | 副本是"没收到"还是"收到但应用慢" | 慢、卡、积压 | modules.replication, modules.observability-and-introspection | 二选一的差分判断 | Y |

### Step 2 路由

- Q1 → modules.keyspace-and-expiration（keywords 键查找 / 到期时间；covers_commands 含 `EXISTS` / `TTL`）+ modules.connection-and-client-io（covers_commands 含 `SELECT`，`db=` 字段在 `CLIENT LIST` 输出里）；
- Q2 → modules.connection-and-client-io（keywords 客户端 / 连接，covers_commands 含 `CLIENT LIST`）；"分库是两套隔离命名空间"的权威说明在 modules/keyspace-and-expiration.md §2.2；
- Q3 → **TS 四级链第一级**：symptom-index.yaml 命中 `replica-data-lagging`（trigger_keywords「从库读不到刚写的数据」）→ 该条 `runbook: TBD`（`troubleshooting/runbooks/` 当前为空）→ 落到第二级 `faq.md`、第三级 `known-bugs.md`，仍未命中则走第四级逆向追踪 → flows/replication-sync-flow.md §2.6 + modules/replication.md §4.6 / §6；
- Q4 → flows/replication-sync-flow.md §4.4（分诊入口 B：`lag` 单调增长但链路未断）+ modules/observability-and-introspection.md §3.2（`SLOWLOG` 判定与截断）。

### Step 3 下钻

```text
[模块 replication 下钻小结]
- 已读文档：index.yaml（含 modules[replication] 产品边界字段）, modules/replication.md §4.6 / §6, flows/replication-sync-flow.md §2.6 / §4.4
- 关键事实：
  1. 副本侧 `server.masterhost` 非空后，"要不要继续用旧数据服务"由 `replica-serve-stale-data`（默认 yes）决定；
     置 no 才是回 `-MASTERDOWN`；链路断开与"读到旧值"因此可以同时成立
  2. 主侧看落后有两个量：`master_repl_offset - slaveN:offset`（字节）与 `slaveN:lag`（距上次 `REPLCONF ACK` 的秒数，
     仅 state=online 有意义）；`repl-timeout` 踢人用的是后者
  3. 可续传窗口 = `[repl_backlog_first_byte_offset, repl_backlog_first_byte_offset + repl_backlog_histlen]`；
     副本请求的偏移落在窗口外只能全量，`sync_full` +1。窗口按字节推进，`repl-backlog-size` 不等于"可断连时长"
- 残留疑点：副本发出的 `REPLCONF ACK` 由主线程还是 I/O 线程发出，取决于 master client 归属
  （`INFO replication` 的 `master_client_io_thread` / `slaveN` 的 `io-thread`），文档只到"归属决定者"这一层
```

### Step 4 一致性检查

- flows/replication-sync-flow.md §2.6 与 modules/replication.md §4.6 对 `repl-timeout` 的五条判据（副本测主三段 + 主测副本两段）一致，无冲突；
- 一处**归属需要显式标注**：`WAIT` / `WAITAOF` 在 index.yaml 的 `covers_commands` 里列在 keyspace-and-expiration，但 ACK 计数与阻塞解除的机制在 modules/replication.md §4.9。引用时以模块文档为准，不按 `covers_commands` 反推实现归属；
- "主写入成功"与"副本读到旧值"之间没有机制冲突：写入成功只证明主侧 `call()` 完成了键空间写入并把变更写进复制流，不证明任何副本已应用。

### Step 5 归纳（回答给用户，deep-dive profile）

````markdown
## 症状确认

归入 `troubleshooting/symptom-index.yaml` 的 **`replica-data-lagging`**（部署形态：主从）。"写入返回 OK" 只说明主实例完成了键空间写入并把这次变更写进复制流；副本读到旧值可以先分成三件事：写错库、链路断了仍用旧数据服务、数据到了但没应用。

## 排查步骤

1. **确认写入落在哪个库、哪个键**（keyspace-and-expiration）
   ```shell
   redis-cli -h <master> -p 6379 -n 0 EXISTS <key>
   redis-cli -h <master> -p 6379 -n 0 TTL <key>
   redis-cli -h <master> -p 6379 INFO keyspace
   ```
   - `TTL` 返回 `-2` → 键已不存在（写时带了过期且已到期，或已被 `DEL`）；`-1` 表示没有过期时间
   - `INFO keyspace` 的 `dbN:keys=` 没有跟着增长 → 写入落到了别的库，进入步骤 2 核对 `db=`

2. **确认"另一条连接"连的是谁、读的哪个库**（connection-and-client-io）
   ```shell
   redis-cli -h <节点> -p <port> CLIENT LIST
   redis-cli -h <master> -p 6379 CLIENT LIST TYPE replica
   ```
   - 应用读的那条连接看 `db=` 字段：`db=0` 与 `db=1` 是两套互不相通的命名空间，`SELECT` 不会跨实例同步
   - `CLIENT LIST TYPE replica` 列出副本链路：`flags` 含 `S`、`cmd` 通常是 `psync` / `replconf`；`obl` / `oll` / `omem` 是该连接还没写进 socket 的量
   - 若应用那条连接的 `flags` 含 `M`，说明你自己就挂在某个主上，本实例是副本

3. **在主侧量落后量**（字节 vs 秒，replication §4.6）
   ```shell
   redis-cli -h <master> -p 6379 INFO replication
   ```
   - `connected_slaves`、`slaveN:ip=..,port=..,state=online,offset=..,lag=..,io-thread=..`
   - `master_repl_offset - slaveN:offset` = 未确认的**字节数**；`slaveN:lag` = 距上次 `REPLCONF ACK` 的**秒数**
   - `repl_backlog_first_byte_offset` / `repl_backlog_histlen` 给出可续传窗口

4. **在副本侧看链路状态与两个偏移**
   ```shell
   redis-cli -h <replica> -p 6379 INFO replication
   ```
   - `master_link_status:down` + `master_last_io_seconds_ago:-1` → 连 master client 都没有，卡在建连/握手
   - `master_link_status:down` 但仍能读到旧值 → 命中 `replica-serve-stale-data yes`（默认）的语义：链路断了继续用旧数据服务；置 `no` 才会对数据访问命令回 `-MASTERDOWN`
   - `master_link_status:up` 且 `slave_read_repl_offset` 明显大于 `slave_repl_offset` → 已读入未应用，是应用侧积压

5. **区分"没收到"还是"收到但应用慢"**（observability-and-introspection §3.2）
   ```shell
   redis-cli -h <replica> -p 6379 SLOWLOG GET 10
   redis-cli -h <replica> -p 6379 INFO stats
   redis-cli -h <replica> -p 6379 INFO commandstats
   ```
   - `SLOWLOG GET` 的条目里能看到由主连接推来并执行的命令（argv 是原命令参数，不是 `PSYNC`）：有记录说明数据到了副本、卡在应用
   - 主侧 `slaveN:offset` 不动、副本 `SLOWLOG` 也没有新条目 → 数据没到，回到步骤 3 看 backlog 与主侧输出缓冲
   - `sync_full` 持续增长且 `sync_partial_err` 也在涨 → 每次重连都续不上，已进入"被踢 → 全量"循环

6. **如果这个键本来带 TTL**（keyspace-and-expiration §4.4）
   ```shell
   redis-cli -h <replica> -p 6379 TTL <key>
   ```
   - 过期从不作为事件被复制：链上只有 `PEXPIREAT`（设定绝对到期时间）与 `DEL` / `UNLINK`（清除动作）
   - 副本读到逻辑上已过期的键按"不存在"返回，但**不自己删键**；因此副本的 `DBSIZE` / `INFO keyspace` 可能还算着这个键，直到主的 `DEL` 到达
   - 写入若用 `SET <key> <val> EX <n>`，主会把相对 TTL 改写成 `PEXPIREAT` 再传播，两端的到期点因此一致

## 恢复步骤

1. 若是读流量打到了副本（读写分离）：把"写完立刻读"的读请求钉回主实例；需要跨实例确认时用
   ```shell
   redis-cli -h <master> -p 6379 WAIT 1 100
   ```
   `WAIT <numreplicas> <timeout_ms>` 只保证"在超时前有 N 个副本 ACK 过该偏移"，不保证落盘；返回小于 N 就是没等到。

2. 若副本链路已断（`master_link_status:down`）：
   ```shell
   redis-cli -h <replica> -p 6379 REPLICAOF <master-host> <master-port>
   ```
   主侧日志出现 `Unable to partial resync ... for lack of backlog` 说明偏移已掉出窗口，只能全量；此时按"峰值写速率 × 可容忍断连秒数"调大 `repl-backlog-size`。

3. 若是副本应用慢：按步骤 5 的 `SLOWLOG` / `INFO stats` 定位慢命令，不要先调 `repl-timeout`（它只改变踢人阈值，不会让副本追上）。

4. **恢复后验证**：
   ```shell
   redis-cli -h <master> -p 6379 SET <key> <value>
   redis-cli -h <master> -p 6379 WAIT 1 100
   redis-cli -h <replica> -p 6379 GET <key>
   ```
   预期副本读出刚写入的值，且 `INFO replication` 的 `slaveN:lag` 回到 0~1 秒。

## 相关模块

- `modules/replication.md` §4.6（`repl-timeout` 的五条判据）、§6（`repl-backlog-size` 不等于可断连时长）
- `flows/replication-sync-flow.md` §2.6（稳态增量与心跳）、§4.4（`lag` 增长分诊）
- `modules/keyspace-and-expiration.md` §4.4（过期为什么传播 `PEXPIREAT` 与 `DEL`）
- `modules/observability-and-introspection.md` §3.2（`SLOWLOG` 判定与截断）

## 置信度声明

| 维度 | 说明 |
|------|------|
| 主要依据 | KB（replication.md §4.6 / §6 + replication-sync-flow.md §2.6 / §4.4 + keyspace-and-expiration.md §4.4） |
| 降低置信度的因素 | `replica-data-lagging` 的 Runbook 仍是 `TBD`，未走已落地决策树；副本应用慢与主侧慢命令的分支需现场 `SLOWLOG` 佐证 |
| 总体置信度 | 中（判据明确，根因尚未由现场指标确认） |

## 知识回写建议

- **类型**：TS 症状未覆盖（`replica-data-lagging` 的 `runbook: TBD`）
- **触发场景**：本次通过 TS 第四级逆向追踪才把"写 OK / 读旧值"拆成"写错库 / 链路断仍服务 / 应用积压"三类
- **建议**：创建 Incident Record `troubleshooting/incidents/YYYY-MM-DD_replica-stale-read.yaml`，后续转换为 `runbooks/replica-data-lagging.md`
- **优先级**：高

---

如果你希望回答更有针对性，可以告诉我你的角色（开发者/测试/产品/运维/用户）和你关注的重点（理解原理/排查问题/评估方案/实施开发/运维操作）。
````

### Step 6 询问（可选）

> "是否需要我起草一份 Incident Record（YAML 格式），记录本次排查过程以便后续转换为 Runbook？"

---

## 提前收敛规则

允许在以下情形提前结束流程：

| 情形 | 在哪一步可提前收敛 |
|------|-------------------|
| 关键词未命中任何路由 | Step 0 后即可报告未覆盖 |
| 问题完全在单一模块内 且 index.yaml 产品边界字段命中明确 | Step 3 后即可回答（仍需做 Step 5 归纳） |
| 问题已被单一 flow 文档完整覆盖 | Step 3 简化为只读 flow 文档 |
| TS 症状精确命中 Runbook | Step 3 简化为只读该 Runbook |
| 简单问答（"xxx 是什么"、"xxx 在哪里实现"） | Step 5 后结束，无需触发 Step 6 |
| profile 是 `user-guide` / `product-view` 且已从 index.yaml 产品边界字段得到答案 | 不进入 knowledge-base，Step 3 提前收敛 |

提前收敛**必须在回答中说明原因**（且原因不涉及内部 Step 编号），例如：

> 本问题完全属于 {{PROJECT_NAME}} CLI 命令的使用范畴，从命令说明即可回答，未展开到内部模块细节。
