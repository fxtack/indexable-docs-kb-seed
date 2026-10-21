# {{PROJECT_SLUG}}-commander · 回答策略参考（answer-strategy.md）

> 示例说明：本文中的具体命令、模块 id 与机制示例取自 Redis 8.10.2（公共领域工程），仅作教学示例，不属于模板变量；`{{PROJECT_NAME}}` / `{{PROJECT_SLUG}}` 为迁移时需替换的占位符。
>
> 本文是 `SKILL.md` Step 0.5 的完整参考文档，定义角色/意图推断规则、策略矩阵路由与 6 个 answer profile。
> 在执行任何 {{PROJECT_NAME}} 任务时，Step 0 加载 index.yaml 后，必须执行 Step 0.5 加载本文件并确定回答策略。

---

## 1. 角色确定

角色不通过特征词从问题文本中隐式推断（准确率低）。改为**显式识别 + 会话记忆 + 意图推导**三级优先级。

### 1.1 角色枚举

| 标识 | 角色 | 典型角色提示词 | {{PROJECT_NAME}} 语境中的常见来源 |
|------|------|--------------|-------------------------|
| `developer` | 开发者 | "从开发角度"、"从研发角度"、"从代码层面"、"从实现角度"、"作为开发者"、"以开发视角" | Redis core 的 C 研发同事、内置模块（`modules/vector-sets/`）开发者 |
| `qa` | 测试/QA | "从测试角度"、"从QA角度"、"从质量角度"、"作为测试"、"以QA视角" | Tcl 集成测试（`tests/unit`、`tests/integration`）、回归验证、长稳压测的测试同事 |
| `pm` | 产品/PM | "从产品角度"、"从PM角度"、"从业务角度"、"作为产品"、"以产品视角" | 需求评审、产品能力评估的产品同事 |
| `sre` | 运维/SRE | "从运维角度"、"从SRE角度"、"从部署角度"、"作为运维"、"以运维视角" | 线上运维、容量规划、主从与 Cluster 故障切换、持久化与备份 |
| `user` | 用户/客户 | "从用户角度"、"从客户角度"、"作为用户"、"以用户视角" | 用 redis-cli 与客户端库访问实例的应用开发者、把实例当缓存的业务方 |

### 1.2 角色确定优先级

```
1. 用户当前问题中包含角色提示词 → 提取角色，同时覆盖会话偏好
2. 无角色提示词、会话中有偏好角色 → 使用会话偏好
3. 无角色提示词、无会话偏好 → 从意图推导默认角色（见 §1.3）
```

> 角色切换规则：用户每次携带角色提示词提问时，直接覆盖已有会话偏好，不区分"临时切换"与"永久切换"。

### 1.3 意图 → 默认角色映射

当用户未明示角色且无会话偏好时，根据意图推导默认角色：

| 意图 | 默认角色 | 推导理由 |
|------|---------|---------|
| `understand` | `developer` | 理解型问题大多关注技术实现 |
| `troubleshoot` | `developer` | 排查问题通常由开发者主导（Redis 排障常需读 `INFO` / `SLOWLOG` / `LATENCY` 与日志） |
| `evaluate` | `pm` | 评估可行性偏向产品决策 |
| `implement` | `developer` | 实施开发天然是开发者视角 |
| `operate` | `sre` | 运维操作天然是运维视角（扩容、主从切换、failover、持久化重写等） |

若意图为 `none` 且无会话偏好，默认角色 = `developer`。

### 1.4 首次无偏好引导

首次会话中角色来源为"意图推导"（即 §1.2 优先级第 3 档）时，回答末尾应附加澄清引导（`profile.fallback_prompt`）。一旦用户在任何一轮携带角色提示词回复，后续不再附加引导。

---

## 2. 意图推断

### 2.1 意图枚举与特征词

| 标识 | 意图 | 特征词（含非精确匹配） |
|------|------|----------------------|
| `understand` | 理解 | "是什么"、"如何工作"、"原理"、"架构"、"为什么这样设计"、"流程"、"机制"、"如何保证"、"关系"、"分层" |
| `troubleshoot` | 排查 | "报错"、"失败"、"异常"、"卡住"、"超时"、"不工作"、"如何排查"、"定位"、"日志"、"根因"、"debug"、"看了"、"现象"、"error"、"crash"、"OOM"、"MOVED"、"主从延迟"、"抖动" |
| `evaluate` | 评估 | "是否可行"、"能不能支持"、"有没有风险"、"评估"、"可行性"、"方案对比"、"选型"、"影响"、"是否合理"、"影响范围"、"波及" |
| `implement` | 实施 | "如何实现"、"新增"、"改造"、"开发"、"接入"、"集成"、"怎么写"、"怎么改"、"怎么实现"、"改动"、"修改"、"增加"、"添加"、"拆解需求"、"任务拆解" |
| `operate` | 运维 | "如何部署"、"怎么配置"、"扩容"、"迁移"、"升级"、"上线"、"回滚"、"主从切换"、"failover"、"槽迁移"、"该不该"、"什么时候"、"操作步骤" |

### 2.2 推断规则

```
1. 对用户问题文本逐词扫描，命中 1 个特征词 → 该意图 +1 分
2. 计分完成后：
   - 多个意图得分接近（最高分与次高分差距 ≤1）→ confidence: low
   - 单一意图明显领先（最高分与次高分差距 ≥2）→ confidence: high
   - 所有意图得分 = 0 → confidence: none
3. 输出：{ intent: <最高分意图标识>, intent_confidence: high|low|none }
```

**可观测性要求**：与角色推断相同，得分必须可追溯到 §2.1 特征词表的具体特征词。不得使用 LLM 自由裁量。

### 2.3 推断示例

| 用户问题 | 得分 | 结果 |
|---------|------|------|
| "SCAN 遍历时为什么同一个 key 会返回两次？它的游标机制是如何工作的？" | understand=3（"为什么"、"机制"、"如何工作"） | intent=understand, high |
| "MEMORY USAGE 和实际占用为什么不一致，影响大吗？" | understand=1（"为什么"）+ evaluate=1（"影响"） | intent=understand, low |
| "主从同步报错 MASTERDOWN，日志里该怎么排查？" | troubleshoot=3（"报错"、"日志"、"排查"） | intent=troubleshoot, high |
| "把 maxmemory-policy 改成 allkeys-lfu 会影响哪些指标、波及哪些业务，有没有风险？" | evaluate=3（"影响"、"波及"、"风险"）+ implement=1（"改"） | intent=evaluate, high |
| "什么时候该做 failover？" | operate=2（"什么时候"、"failover"） | intent=operate, high |
| "hello" | 全 0 | intent=none, none |

---

## 3. 策略矩阵

| | understand | troubleshoot | evaluate | implement | operate |
|---|---|---|---|---|---|
| **developer** | `deep-dive` | `deep-dive` | `deep-dive` | `deep-dive` | `ops-detail` |
| **qa** | `qa-view` | `qa-view` | `qa-view` | `qa-view` | `qa-view` |
| **pm** | `product-view` | `product-view` | `product-view` | `product-view` | `product-view` |
| **sre** | `ops-detail` | `ops-detail` | `ops-detail` | `ops-detail` | `ops-detail` |
| **user** | `user-guide` | `user-guide` | `user-guide` | `user-guide` | `user-guide` |

---

## 4. 推断结果 → 策略收敛

### 4.1 角色来源判定

```
if 用户问题包含 §1.1 角色提示词:
    persona = 提取的角色
    persona_source = "explicit"        # 同时覆盖会话偏好
elif 会话中有偏好角色:
    persona = 会话偏好
    persona_source = "session"
elif intent != "none":
    persona = 意图推导默认角色(§1.3)
    persona_source = "derived"
else:
    persona = "developer"              # 最终兜底
    persona_source = "derived"
```

### 4.2 策略收敛

```
if intent_confidence == "none":
    strategy = "default"
    attach_clarification = true

elif persona_source == "derived":
    strategy = 矩阵[persona][intent]
    attach_clarification = true        # 推导角色需引导确认

else:  # explicit 或 session
    strategy = 矩阵[persona][intent]
    attach_clarification = false       # 用户已明示角色，无需引导
```

> 注：`attach_clarification=true` 时在回答末尾附加 `profile.fallback_prompt`。首次 explicit 确认后，后续不再附加。

### 4.3 硬性约束

- **策略名必须来自 §3 策略矩阵**，即以下 6 个名称之一：`deep-dive` / `ops-detail` / `qa-view` / `product-view` / `user-guide` / `default`。禁止使用任何矩阵外的策略名。
- **意图完全失败（intent=none）时，强制使用 `default` 策略**。即使问题与 {{PROJECT_NAME}} 完全无关，也必须走完流程并收敛到 `default`，不得跳过 Step 0.5 或自创策略。
- **不得因"问题超出知识范围"而拒绝回答或绕过策略系统**。`default` 策略设计上就能处理这种情况。

### 4.4 澄清引导

需要附加在回答末尾时使用（`attach_clarification == true`）：

> 如果你希望回答更有针对性，可以告诉我你的角色（开发者/测试/产品/运维/用户）和你关注的重点（理解原理/排查问题/评估方案/实施开发/运维操作）。

---

## 5. Answer Profile 定义

### 5.1 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `strategy` | string | 策略标识 |
| `description` | string | 策略适用场景的一句话描述 |
| `depth.level` | enum | `code`（代码级）/ `architecture`（架构级）/ `capability`（能力级）/ `api-surface`（API 表面级） |
| `depth.allow_module_code` | bool | Step 3 是否允许下钻到模块源码 |
| `depth.allow_grep` | bool | 是否允许代码搜索 |
| `terminology.internal_terms` | enum | `use-freely` / `explain-first` / `avoid`（内部术语：`listpack`、`intset`、`repl_backlog`、`rdbSaveBackground()` 等） |
| `terminology.component_names` | enum | `use-freely` / `necessary-only` / `avoid`（模块名：`keyspace-and-expiration`、`memory-management-and-eviction`、`replication` 等） |
| `structure.problem_breakdown_table` | bool | 是否在回答中展示 Step 1 问题切割表 |
| `structure.routing_table` | bool | 是否在回答中展示 Step 2 路由表 |
| `structure.confidence_matrix` | bool | 是否在回答中展示置信度矩阵 |
| `content.include` | list | 明确要包含的内容类别 |
| `content.exclude` | list | 明确不包含的内容类别 |
| `tone` | enum | `technical-precise` / `systematic` / `business` / `operational` / `user-friendly` |
| `use_analogy` | bool | 是否推荐使用类比解释 |
| `max_detail_level` | 1-4 | 1=一句话, 2=段落, 3=分节, 4=全量含代码片段 |
| `fallback_prompt` | string | 模糊推断时附加的澄清引导文本（空表示不需要） |

### 5.2 `deep-dive` — 开发者技术深度回答

```yaml
strategy: deep-dive
description: "面向开发者的技术深度回答，覆盖代码路径、设计决策与数据结构"

depth:
  level: code
  allow_module_code: true
  allow_grep: true

terminology:
  internal_terms: use-freely
  component_names: use-freely

structure:
  problem_breakdown_table: true
  routing_table: false
  confidence_matrix: true

content:
  include:
    - "代码符号锚点（函数名/结构体名/常量名）与设计决策"
    - "数据结构与协议细节（`robj` 的 `type`/`encoding`、`listpack`/`intset` 布局、RESP2/RESP3 帧格式、`repl_backlog` 环形缓冲）"
    - "并发模型与线程边界（主线程串行执行、iothread 只做收发与解析、bio 后台任务）"
    - "错误码传播与失败路径"
    - "性能与稳定性权衡"
    - "已知陷阱与变更影响"
  exclude: []

tone: technical-precise
use_analogy: false
max_detail_level: 4
fallback_prompt: ""
```

### 5.3 `ops-detail` — 运维/SRE 操作细节

```yaml
strategy: ops-detail
description: "面向运维/SRE 的架构级操作回答，关注部署拓扑、配置、监控与故障模式"

depth:
  level: architecture
  allow_module_code: false
  allow_grep: false

terminology:
  internal_terms: explain-first
  component_names: explain-first

structure:
  problem_breakdown_table: false
  routing_table: true
  confidence_matrix: true

content:
  include:
    - "部署形态与拓扑（单机/主从/Cluster/Sentinel、`redis.conf` 与启动参数入口）"
    - "日志与诊断入口（redis-server 日志路径、`INFO` 各分段、`SLOWLOG`、`LATENCY`、`MONITOR`）"
    - "redis-cli 命令参数与预期输出"
    - "关键持久化产物（RDB 快照、AOF 清单与分段、`INFO persistence` 的进度字段）"
    - "故障与恢复路径（全量/部分重同步、failover、Cluster 槽迁移）"
    - "故障模式与恢复步骤"
    - "关键监控指标（`used_memory`、`evicted_keys`、`keyspace_hits`/`keyspace_misses`、`master_link_status`）"
  exclude:
    - "源码级符号（`rdbSaveBackground()`、`performEvictions()` 等内部函数名）"
    - "内部数据结构（`robj` 位域布局、`listpack` 字节编码细节）"

tone: operational
use_analogy: false
max_detail_level: 3
fallback_prompt: ""
```

### 5.4 `qa-view` — QA 行为与测试视角

```yaml
strategy: qa-view
description: "面向测试/QA 的系统行为回答，覆盖输入输出契约、边界条件、测试范围与回归影响"

depth:
  level: architecture
  allow_module_code: false
  allow_grep: false

terminology:
  internal_terms: necessary-only
  component_names: necessary-only

structure:
  problem_breakdown_table: false
  routing_table: false
  confidence_matrix: true

content:
  include:
    - "系统行为契约（客户端请求 → 应答与键空间变化的可观察输出）"
    - "边界条件（空键 / 二进制安全键名 / 大 key / 空集合 / 已过期键 / 编码升降级的阈值临界点）"
    - "错误码含义与用户可见表现（`WRONGTYPE` / `OOM` / `MOVED`、`ASK` 重定向 / `LOADING` 等）"
    - "复制与持久化状态转换的可测触发点（全量同步、部分重同步、AOF 重写）"
    - "回归影响范围（命令语义变更 / 编码阈值变更 / RDB 与 AOF 格式变更）"
    - "可测性评估（是否需要多实例或 Cluster / 是否需要注入 fork 或磁盘写失败）"
    - "建议的测试策略（单模块 / 端到端 / 长稳）"
  exclude:
    - "主线程与 iothread 的调度细节"
    - "`listpack`/`intset` 的字节布局"
    - "设计决策与权衡理由"

tone: systematic
use_analogy: false
max_detail_level: 3
fallback_prompt: ""
```

### 5.5 `product-view` — 产品/PM 视角

```yaml
strategy: product-view
description: "面向产品/PM 的能力级回答，使用产品语言与类比，不涉及内部实现"

depth:
  level: capability
  allow_module_code: false
  allow_grep: false

terminology:
  internal_terms: avoid
  component_names: avoid

structure:
  problem_breakdown_table: false
  routing_table: false
  confidence_matrix: false

content:
  include:
    - "系统能力边界（支持的数据类型与命令族、单机/主从/Cluster/Sentinel 形态）"
    - "产品限制与上限（单键大小、`maxmemory` 与淘汰策略、连接数与输出缓冲上限）"
    - "适用场景与不适用场景（缓存/会话/排行榜/消息流，vs 强事务的关系型存储）"
    - "与竞品/替代方案的高层对比（如 Redis 与 Memcached、嵌入式 KV 的取舍）"
    - "使用体验（用户看到的表现，如"数据访问像本地内存一样快"）"
  exclude:
    - "主线程/I/O 线程/后台任务的线程分层"
    - "`listpack` / `intset` / `repl_backlog` 等技术术语"
    - "内部模块名（`keyspace-and-expiration` / `replication` 等）"
    - "内部实现机制（见 §8 内部实现模块可见性约束）"
    - "置信度矩阵"

tone: business
use_analogy: true
max_detail_level: 2
fallback_prompt: ""
```

### 5.6 `user-guide` — 用户/客户指南

```yaml
strategy: user-guide
description: "面向 {{PROJECT_NAME}} 用户的操作指南回答，只涉及 {{PROJECT_NAME}} 命令行客户端的命令与可见现象，不涉及内部架构"

depth:
  level: api-surface
  allow_module_code: false
  allow_grep: false

terminology:
  internal_terms: avoid
  component_names: avoid

structure:
  problem_breakdown_table: false
  routing_table: false
  confidence_matrix: false

content:
  include:
    - "redis-cli 命令的使用方式与参数说明"
    - "命令的常见输出与含义"
    - "常见错误现象与自助处理方式（如被 `MOVED` 重定向时先确认访问的节点角色）"
    - "日志文件位置（供联系管理员时提供）"
    - "使用限制与注意事项（线上不要执行 `KEYS`、大 key 优先用 `UNLINK` 删除等）"
  exclude:
    - "主线程与 I/O 线程的分工"
    - "内部数据结构"
    - "内部模块名"
    - "内部实现机制"
    - "置信度矩阵"

tone: user-friendly
use_analogy: true
max_detail_level: 2
fallback_prompt: ""
```

### 5.7 `default` — 模糊推断时的默认策略

```yaml
strategy: default
description: "推断置信度不足时的默认策略，提供架构级回答并引导用户明确身份与意图"

depth:
  level: architecture
  allow_module_code: false
  allow_grep: false

terminology:
  internal_terms: explain-first
  component_names: explain-first

structure:
  problem_breakdown_table: false
  routing_table: false
  confidence_matrix: true

content:
  include:
    - "架构级系统行为描述"
    - "关键结论与要点"
  exclude:
    - "代码符号与函数签名"

tone: technical-precise
use_analogy: false
max_detail_level: 3
fallback_prompt: "如果你希望回答更有针对性，可以告诉我你的角色（开发者/测试/产品/运维/用户）和你关注的重点（理解原理/排查问题/评估方案/实施开发/运维操作）。"
```

---

## 6. Profile 对 {{PROJECT_SLUG}}-commander 步骤的约束映射

| Step | 受约束的行为 | 约束来源 |
|------|-------------|---------|
| Step 0.5 | 确定角色/推断意图 → 查矩阵 → 加载 profile | 本文 §1~§4 |
| Step 1 | 是否产出问题切割表 | `structure.problem_breakdown_table` |
| Step 3 | 是否允许下钻源码 / grep | `depth.allow_module_code` / `depth.allow_grep` |
| Step 3 | 术语解释策略 | `terminology.internal_terms` / `terminology.component_names` |
| Step 5 | 回答包含/排除的内容类别 | `content.include` / `content.exclude` |
| Step 5 | 回答语气与类比使用 | `tone` / `use_analogy` |
| Step 5 | 回答详细程度 | `max_detail_level` |
| Step 5 | 是否展示置信度矩阵 | `structure.confidence_matrix` |
| Step 5 | 是否展示路由表 | `structure.routing_table` |
| Step 5 | 模糊推断时附加澄清引导 | `fallback_prompt` |

---

## 7. Step 0.5 执行清单

每次接到 {{PROJECT_NAME}} 问题，Step 0（加载 index.yaml）完成后，在**内部**执行以下操作（分析过程不输出到对话）：

1. 打开本文（`references/answer-strategy.md`）；
2. 按 §1.1 检测用户问题中的角色提示词：
   - 命中 → 提取角色，覆盖会话偏好，`persona_source = "explicit"`；
   - 未命中、有会话偏好 → `persona_source = "session"`；
   - 未命中、无会话偏好 → 暂不决定，等意图推断后按 §1.3 推导；
3. 按 §2 对用户问题进行意图推断，产出 `intent` + `intent_confidence` + 得分 + 命中明细（内部自检）；
4. 若 `persona` 尚未确定（无 explicit 也无 session），按 §1.3 意图→默认角色映射推导，`persona_source = "derived"`；
5. 按 §4.2 收敛规则确定 `strategy` + `attach_clarification`；
6. 按 §3 矩阵查表，加载 §5 中对应 profile；
7. 后续 Step 1~5 按 §6 约束映射执行。

**命中明细要求**（内部自检，不输出到回答）：意图推断的得分必须可追溯到 §2.1 特征词表。这是防止 LLM 自由裁量的内部检查机制。

**输出禁令**：角色来源、意图得分、策略名、命中明细等信息**严禁出现在用户可见的回答中**。仅当 `attach_clarification == true` 时，在回答末尾附加 `profile.fallback_prompt` 澄清引导文本。

> **注意**：Step 0.5 的角色/意图推断仅用于**确定回答策略**，不影响 Step 1 的问题切割和 Step 2 的路由逻辑。路由仍按原规则执行。推断置信度低不代表问题无法回答，只代表回答策略会收敛到 `default` 并附加澄清引导。

---

## 8. 内部实现模块的可见性约束

### 8.1 原则

{{PROJECT_NAME}} 的部分模块与机制属于**内部实现细节**，对上层用户完全透明——用户无需感知其存在、时机或状态。在 Step 5 归纳回答时，面向用户/产品的 profile **不得**以这些模块的存在或行为来解释系统表现。

### 8.2 内部实现清单

以下模块与机制被认定为"内部实现"：

| 内部机制 | 用户不可见的原因 |
|---------|------------------|
| 事件循环与 I/O 线程（`aeProcessEvents()` 驱动事件分发，iothread 只承担收发与解析，命令仍在主线程串行执行） | 用户只看到请求按顺序拿到应答，不需要知道字节在哪个线程被读、写、解析 |
| 值的编码与编码转换（`embstr`/`raw`/`int` 之间，以及 `listpack`/`intset`/`quicklist`/`hashtable`/`skiplist` 之间的升降级） | 用户看到的是同一数据类型与同一套命令语义；编码不进对外契约，`OBJECT ENCODING` 只是诊断信息 |
| RDB 快照的 fork 与写时复制（`rdbSaveBackground()` 起子进程，序列化完一个键就放弃其物理页以降低 COW） | 用户只看到 `BGSAVE` 返回 `Background saving started` 与 `INFO persistence` 的进度，不需要知道 fork/COW 与内存放大 |
| AOF 重写与分段清单（`BGREWRITEAOF` 整理日志时复用 RDB 写出流程，`aofLoadManifestFromDisk()` 在启动时装载清单与分段） | 用户只看到 `INFO persistence` 的 `aof_rewrite_in_progress` 与文件体积变化，不需要知道重写如何分批与切换 |
| 复制积压缓冲与部分重同步（`repl_backlog` 环形缓冲、`PSYNC` 续传判定、副本缓冲的记账口径） | 用户只看到 `master_link_status` 与 `INFO replication` 的偏移量，不需要知道积压缓冲多大、断链后走全量还是续传 |
| 过期键的清理节奏（`activeExpireCycle()` 抽样清理与访问时的惰性删除） | 用户只看到"键到点就不见了"，不需要知道到期是访问时判定还是由周期任务抽样清掉 |
| 客户端缓存失效提醒的推送时机（`trackingInvalidateKey()` 登记、`beforeSleep()` 批量发送） | 用户只看到失效 push 到达，不需要知道它是在命令末尾补发还是等下一轮事件循环 |

**用户可见的边界**：`redis-cli` / `redis-benchmark` 的命令与参数、`redis.conf` / `sentinel.conf` 的配置项、`INFO` / `SLOWLOG` / `LATENCY` / `MEMORY` 等观测命令的输出、redis-server 日志文件、RDB 与 AOF 持久化文件。

### 8.3 各 Profile 的约束

| Profile | 提及内部实现模块 | 说明 |
|---------|----------------|------|
| `user-guide` | **严禁** | 即使用户问题涉及相关行为（如"为什么删除后空间不立刻释放"），只从可见现象与 {{PROJECT_NAME}} CLI 命令层面回答 |
| `product-view` | **严禁** | 从产品能力边界回答（支持/不支持），不涉及内部技术实现 |
| `qa-view` | 可提及行为契约 | 如"键修改后失效提醒在一个事件循环内发出"，但不应展开跟踪表结构与推送路径的实现细节 |
| `ops-detail` | 可提及 | 建议标注"内部实现，用户不可见" |
| `deep-dive` | 无限制 | — |
| `default` | 遵循术语策略 | 若提及需先解释其为内部机制 |

### 8.4 误答对照示例（Redis 场景）

| 用户问题 | Profile | ❌ 错误回答 | ✅ 正确回答 |
|---------|---------|------------|-----------|
| "为什么删一个大 key，服务会卡一下？" | `user-guide` | "因为 `decrRefCount()` 走同步释放，而 `lazyfreeGetFreeEffort()` 对单个大字符串只算 1，没到 64 的阈值，所以没投到 `BIO_LAZY_FREE` 线程……" | "删除大 key 时，服务端要把这个键占用的内存一次性回收，这一步会占住主线程，其他请求要排队。建议把大 key 拆小，删除时用 `UNLINK` 让内存回收动作在后台完成。" |
| "UNLINK 和 DEL 有什么区别？" | `product-view` | "`UNLINK` 只是把任务投进 `bio` 队列，由 `lazyfreeFreeObject()` 在主线程之外调 `decrRefCount()`……" | "两者的删除效果一样，区别在回收时机：`DEL` 当场把内存还掉，`UNLINK` 先让键立刻不可见、把回收放到后台做。体量大的键用 `UNLINK` 可以少占主线程时间，但内存不会马上降。" |
| "为什么线上禁止用 KEYS？" | `qa-view` | "`keysCommand()` 直接用 `kvstore` 全量迭代，把匹配的键名全部装进应答，按游标分片扫描的 `SCAN` 路径完全没走……" | "`KEYS` 是一次性全库遍历并返回全部匹配键名（O(N)，N 为库里键数），没有分批语义，键多时会把服务占住、其他请求只能排队。回归测试应把「大量键时 `KEYS` 的耗时与阻塞」作为断言项，生产场景改用 `SCAN` 分批遍历。" |
