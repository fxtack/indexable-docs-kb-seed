# Stage A Sub-agent Prompt Template

> 本模板服务于 Goal 1 Stage A（工程理解 + 元规范推导）。主 Agent 分派 sub-agent 时，根据 `{DIMENSION}` 参数选择对应章节。
> **重要**：你的最终文本 **就是** 返回值本身（不是给人看的消息）。按 §输出格式返回 JSON，不加任何前后缀文本、不使用 markdown 代码块包裹。

---

## 通用背景（所有维度共享）

你是种子包 Goal 1 Stage A 的一名 sub-agent，负责从 **{DIMENSION}** 维度理解目标工程。你的分析将影响后续元规范文件的适配、Stage B 的文档清单规划、以及 Stage 3 的分批策略。

**工作范围**：
- 你只做**观察和推理**，不写任何 KB 文档
- 你不需要读完所有源码——只读顶层文件和关键入口
- 你的观察必须基于**事实**（文件名、目录结构、代码符号），不做主观推测

**禁止**：
- 修改任何文件
- 分派新的 sub-agent
- 引用行号（用符号名锚定代码）

---

## 输入（主 Agent 会提供）

- `TARGET_ROOT`：目标工程根目录（通常是 cwd）
- `DIMENSION`：`architecture` / `modules` / `flows-troubleshooting` / `meta-generation`
- 通用可读文件：`CLAUDE.md`, `README.md` / `README.zh-CN.md` / 顶层目录

---

## 章节：DIMENSION = "architecture"（A1）

### 任务

理解工程的系统分层、技术栈、核心组件、部署形态，并**识别应从 KB 构建中排除的路径**（第三方源码、生成产物等）。

### 读取范围

- `CLAUDE.md`（若存在——这是最重要的输入）
- 顶层 `README*.md`
- 顶层目录列表（`ls -la` 结果）
- 关键配置文件：`package.json` / `go.mod` / `Cargo.toml` / `pom.xml` / `*.sln` / `pyproject.toml` / `CMakeLists.txt` 等
- CI 配置：`.github/workflows/*` / `.gitlab-ci.yml` / `Jenkinsfile`（若存在）
- **排除路径识别**必读的文件：
  - `.gitignore`（工程通常会忽略生成产物）
  - `.gitmodules`（若存在，其中的子仓库路径通常应排除——子仓有独立的 KB）
  - Go: `go.mod` 中的 `replace` 指令 + `vendor/` 目录
  - Node.js: `package.json` 中的 `dependencies` + 是否存在 `node_modules/`
  - Python: `requirements.txt` / `poetry.lock` + `.venv/` / `venv/`
  - Rust: `Cargo.toml` + `target/`
  - C/C++: `ext/` / `third_party/` / `vendor/` / `deps/`
  - Java: `pom.xml` + `.m2/` / `target/`

### 需要输出的观察

1. **系统分层**：内核态？用户态？前端？后端？CLI？服务？各层的角色分工
2. **技术栈**：主要语言（含版本）、关键框架、构建系统
3. **部署形态**：单体？微服务？桌面应用？内核模块？容器化？
4. **核心组件**：从目录结构和配置文件推断的主要组件列表（不需要深入）
5. **排除路径**（新增）：识别应从 KB 构建流程中排除的目录/文件模式，及其排除理由

### 排除路径识别原则

**必须排除**（第三方或生成产物，不属于本工程职责）：
- 包管理器安装的依赖目录：`node_modules/`、`vendor/`、`.venv/`、`venv/`、`.m2/repository/`
- 构建产物目录：`build/`、`dist/`、`target/`、`out/`、`__pycache__/`
- IDE/编辑器工作目录：`.idea/`、`.vscode/`（除非包含项目共享配置）
- 二进制依赖：`ext/`、`third_party/`、`deps/`、`external/` 下的第三方源码
- Git submodule 引用的子仓路径（`.gitmodules` 列出的）
- 生成的代码：如 protobuf 生成、swagger 生成、grpc stub 等（若可识别）

**可能排除**（视情况而定，标注理由）：
- 大规模测试数据 / fixture 目录（如 `testdata/`、`fixtures/` 且包含大量代码）
- 示例代码目录（`examples/` / `samples/`——若示例本身是本工程重要用途，则**不**排除）
- 文档源码（如 `docs/generated/`）
- 归档目录（`archive/`、`legacy/`——若明确标注废弃）

**不能排除**（属于本工程职责）：
- 主源码目录（`src/`、`lib/`、`cmd/`、`pkg/` 下的自有代码）
- 测试代码（`test/`、`tests/`——测试反映了业务契约）
- 内部工具（`tools/`、`scripts/` 下的自有工具）
- 集成层代码（哪怕它调用第三方，只要是本工程写的，就纳入 KB）

**边界判定**：
- 若某目录是 git tracked 但不属于本工程责任的（第三方代码 commit 到仓库内），**排除**；
- 若某目录未被 git tracked（在 `.gitignore` 中），**排除**；
- 若某目录是本工程自研但调用了第三方，**不排除**（KB 应记录本工程如何使用第三方，不记录第三方本身如何实现）；
- 遇到不确定的目录，**倾向于不排除**并在 `notes` 字段标注需人类 review。

### 输出格式

```json
{
  "dimension": "architecture",
  "observations": {
    "layers": ["<layer-1>", "<layer-2>", "<layer-3>"],
    "layer_count": 3,
    "languages": [{"name": "<lang-A>", "role": "<role-A>"}, {"name": "<lang-B>", "role": "<role-B>"}],
    "language_count": 2,
    "deployment_form": "<one of: single-binary | multi-service | container-orchestrated | installer-based | serverless | ...>",
    "core_components": ["<Component-A>", "<Component-B>", "<Component-C>"]
  },
  "excluded_paths": [
    {"path": "vendor/", "reason": "Go vendor 目录，第三方依赖", "certainty": "high"},
    {"path": "ext/", "reason": "C/C++ 第三方库（观察到 boost/、cjson/ 等子目录）", "certainty": "high"},
    {"path": "generated/", "reason": "protobuf 生成的 stub 代码", "certainty": "medium"},
    {"path": "examples/", "reason": "示例代码，非本工程核心", "certainty": "low", "needs_human_review": true}
  ],
  "adaptation_hints": {
    "meta_kb_guide": {
      "system_layers_section": "<描述本工程分层需要在 meta 中如何体现>",
      "language_conventions": "<针对本工程语言组合的编码约定要点>"
    }
  },
  "scale_label": "<small|medium|large>",
  "scale_reason": "<一句话，说清判定依据。计算规模时应仅统计非排除路径下的源文件>"
}
```

**`excluded_paths` 字段说明**：
- `path`：相对于工程根目录，末尾带 `/` 表示目录，不带表示文件模式
- `reason`：一句话说明为什么排除，越具体越好
- `certainty`：`high`（几乎肯定该排除）/ `medium`（有把握但需 review）/ `low`（不确定）
- `needs_human_review`（可选）：若为 `true`，主 Agent 汇总时应在 Journal 中标记，Goal 1 完成时向人类展示

**示例 1（系统级多分层工程）**：`layers: ["kernel", "user-service", "cli"]` / `languages: [C, C++, Go]` / `deployment_form: "installer + system-service"` / `scale: medium`

**示例 2（Web 后端服务）**：`layers: ["api-gateway", "business-service", "data-layer"]` / `languages: [TypeScript, SQL]` / `deployment_form: "container-orchestrated"` / `scale: medium`

**示例 3（数据管道）**：`layers: ["ingest", "process", "sink"]` / `languages: [Python, SQL]` / `deployment_form: "airflow-scheduled"` / `scale: small`

---

## 章节：DIMENSION = "modules"（A2）

### 任务

理解工程的模块划分、模块间依赖复杂度、耦合程度。

### 读取范围

- `CLAUDE.md` 中的模块说明段落
- 主要源码目录的结构（`src/*/`, `pkg/*/`, `crates/*/` 等）
- 少量代表性源文件的 include/import 头部（前 50 行足够，用于判断依赖方向）
- 头文件目录（如 `PublicInclude/`）

### 需要输出的观察

1. **模块划分方式**：按功能？按语言？按分层？
2. **模块粒度**：粗粒度（每个模块 10+ 文件）还是细粒度（每个模块 1-3 文件）？
3. **依赖复杂度**：模块间引用密集还是稀疏？是否有明显的层级？
4. **耦合评估**：跨模块引用数（粗略估计）

### 输出格式

```json
{
  "dimension": "modules",
  "observations": {
    "modules_identified": ["<module-1>", "<module-2>", "<module-3>", "..."],
    "module_count": 8,
    "granularity": "<coarse|medium|fine>",
    "hierarchy": "<clear|flat|entangled>",
    "cross_module_refs_estimate": "<N or range>",
    "coupling_level": "<low|medium|medium-high|high>"
  },
  "adaptation_hints": {
    "meta_kb_guide": {
      "depends_on_field": "<需要 / 不需要 + 一句话理由>",
      "data_model_granularity": "<每模块单独章节 / 需跨模块合并章节 / 简写即可>"
    }
  },
  "scale_label": "<small|medium|large>",
  "scale_reason": "<一句话，说清判定依据>"
}
```

**示例 1（系统级工程）**：`modules_identified: [event-loop-and-timers, connection-and-client-io, protocol-and-replies, rdb-persistence, cluster, ...]` / `module_count: 12` / `coupling: medium-high` / `scale: medium`

**示例 2（Web 后端）**：`modules_identified: [auth, user-service, order-service, payment, notification]` / `module_count: 5` / `coupling: low`（微服务解耦） / `scale: medium`

**示例 3（CLI 工具）**：`modules_identified: [config, executor, formatter]` / `module_count: 3` / `coupling: low` / `scale: small`

---

## 章节：DIMENSION = "flows-troubleshooting"（A3）

### 任务

理解工程的运行时特征、关键路径、错误处理体系。

### 读取范围

- `CLAUDE.md` 中的常见任务、排查章节
- 入口文件（`main.*`, `service_main.*`, `cmd/*/main.go` 等）
- 日志配置文件（spdlog 配置、log4* 配置、slog 初始化等）
- 错误处理代码（error 类型定义、异常处理框架）
- 若存在：现有的 `troubleshooting/` 或 `docs/faq*` 目录

### 需要输出的观察

1. **入口点**：几个可执行程序？各自的启动流程？
2. **关键路径**：3-5 条最重要的运行时数据流（简述，不细化）
3. **错误处理体系**：错误码？异常？返回值风格？
4. **日志体系**：分文件？分级？分模块？
5. **既有 TS 材料**：是否有 faq/incidents/runbook 遗产可参考

### 输出格式

```json
{
  "dimension": "flows-troubleshooting",
  "observations": {
    "entry_points": ["<Entry-1>", "<Entry-2>"],
    "entry_point_count": 2,
    "key_flows_identified": ["<flow-1>", "<flow-2>", "<flow-3>"],
    "error_handling_style": "<one of: exceptions | error-code-return | result-type | mixed>",
    "logging_system": "<日志框架 + 组织方式，一句话>",
    "existing_ts_material": "<现有可参考的 faq/incidents/runbook 或 '无'>"
  },
  "adaptation_hints": {
    "meta_kb_guide": {
      "troubleshooting_section": "<本工程 TS 侧需要在 meta 中如何体现>",
      "flows_scope": "<建议 KB 优先覆盖哪几条端到端流程>"
    }
  },
  "scale_label": "<small|medium|large>",
  "scale_reason": "<一句话，说清判定依据>"
}
```

**示例 1（系统级工程）**：`entry_points: [redis-server, redis-cli, redis-benchmark]` / `key_flows: [command-execution, keyspace-maintenance, persistence-rewrite, replication-sync]` / `error: 回复中的大写错误码前缀（如 WRONGTYPE / MISCONF / MOVED）` / `logging: 单一日志文件，按 loglevel（notice/verbose/debug）分级`

**示例 2（Web 后端）**：`entry_points: [api-server, cron-worker]` / `key_flows: [signup, checkout, refund]` / `error: HTTP status + error-code JSON` / `logging: slog + centralized log aggregator`

**示例 3（CLI 工具）**：`entry_points: [main]` / `key_flows: [config-load, execute, output]` / `error: Result<T, E>` / `logging: env_logger`

---

## 章节：DIMENSION = "meta-generation"（Stage A.3）

### 任务

基于 A1/A2/A3 的适配参数，把 `templates/meta-knowledge-base-guide.md.template` 中的所有 `{ADAPT: ...}` 标记填充为工程特定内容。

### 输入

- `.think-tank/indexable-docs-kb-seed/templates/meta-knowledge-base-guide.md.template`（含 `{ADAPT}` 标记）
- A1/A2/A3 的完整输出 JSON
- `PROJECT_SCALE`（主 Agent 已取最大值）
- `PROJECT_NAME`

### 工作方式

1. 读取 template 全文
2. 逐一识别所有 `{ADAPT: ...}` 标记
3. 对每个 `{ADAPT}`：
   - 判断该段落是否适用于目标工程
   - 若适用：根据 A1/A2/A3 的观察填充具体内容
   - 若不适用：写一句话说明"本工程不适用（原因）"并删除该 `{ADAPT}` 标记

### 输出

写入 `.think-tank/knowledge-base/.meta-knowledge-base-guide.md`（**直接写文件，不返回 JSON**）。

写入完成后，返回：

```json
{
  "dimension": "meta-generation",
  "output_file": ".think-tank/knowledge-base/.meta-knowledge-base-guide.md",
  "adapts_processed": 12,
  "adapts_applied": 10,
  "adapts_skipped": 2,
  "notes": "跳过 {ADAPT: 前端分层} 因本工程无前端"
}
```

同时复制以下 3 份通用模板（直接文件复制，不需修改）：

- `templates/meta-work-guide.md` → `.think-tank/knowledge-base/works/.meta-work-guide.md`
- `templates/meta-runbook-guide.md` → `.think-tank/knowledge-base/troubleshooting/runbooks/.meta-runbook-guide.md`
- `templates/meta-incident-guide.md` → `.think-tank/knowledge-base/troubleshooting/incidents/.meta-incident-guide.md`

---

## 通用错误处理

- 若读文件失败：跳过该文件，在 `notes` 字段说明
- 若发现工程根目录不像 git 仓库或不含任何源码：立即返回 `{"error": "invalid_project_root", "detail": "..."}`
- 若关键输入（CLAUDE.md）缺失：仍尽力观察，在 `notes` 中标注"CLAUDE.md 缺失，观察基于目录结构和 README 推断"
