---
name: explore-kb
description: >
  探索和检索知识库 (.think-tank/knowledge-base/)。
  当需要定位模块文档、理解系统架构、查阅设计决策、按 .meta-work-guide.md 进行任务规划、
  或使用 troubleshooting 文档排查问题时使用本 skill。
user-invocable: true
---

# explore-kb — 知识库探索

> 本 skill 提供知识库的**深度操作指导**。基础使用（读 index.yaml → 读匹配文档）已是默认行为，无需本 skill。

---

## 1. 状态感知：了解 kb

知识库位于 `.think-tank/knowledge-base/`，七层结构：

| 目录 | 职责 | 何时查阅 |
|------|------|---------|
| `architecture/` | 系统全景、技术栈、领域模型 | 首次接触项目、需要全局认知 |
| `modules/` | 各模块详尽知识（**权威源**） | 需要了解单个模块的数据模型、API、业务流程、已知陷阱 |
| `flows/` | 跨模块端到端流程 | 需要理解数据在模块间如何流转 |
| `troubleshooting/` | 高频问题排查 | 遇到异常现象需要定位 |
| `develop/` | 研发行为规范 | 特定操作（如驱动开发、错误处理）有规范约束时 |
| `works/` | 研发任务中枢 | 接到研发任务需创建设计文档、查阅既有决策 |
| `pass/` | 构建期产物 + 外部输入中转 | 只读参考，**不入 index.yaml**；构建 QA / Journal / build-findings 与外部输入都放这里，消化后转化为 works/ 或代码 |

检索入口只有一个：**`.think-tank/knowledge-base/index.yaml`**。始终从这里开始。

---

## 2. 检索：三级降级匹配

在 `index.yaml` 中按以下优先级匹配，上一级命中即停止：

```
Level 1 — 精确匹配
  ├── 问题含源代码文件路径？→ 匹配 covers_code
  └── 问题含 CLI 命令名？   → 匹配 covers_commands

Level 2 — 关键词匹配
  └── 提取问题关键词 → 匹配 keywords / tags，取交集最多者

Level 3 — 泛化搜索
  ├── 匹配 depends_on（前置依赖通常也有相关信息）
  ├── 匹配 involves_modules（流程文档可提供更多上下文）
  └── 匹配 refs（guide/排查文档引用的模块文档）

全部失败 → 浏览 architecture/ 建立全局认知 → 直接读代码
```

命中后扩展阅读：检查匹配条目的 `depends_on`、`involves_modules`、`refs` 字段，按需补充上下文。

---

## 3. 规划：Work 文档驱动的研发流程

接到研发任务（功能开发、优化、修复、重构、测试等）时，按以下流程推进：

1. 检索 kb 了解相关模块现状，核实问题或确认影响范围
2. 与用户进行需求与设计讨论，掌握任务全貌
3. 读 `works/.meta-work-guide.md` 了解 Work 文档规范
4. 在 `works/` 下创建 Work 文档（`status: draft`），按模板填充原始需求、现状分析、设计方案、决策汇总等章节
5. 围绕决策汇总章节与用户 Q&A，澄清所有设计不明确点
6. 决策锁定后调整设计方案与 Phase 计划，确保前后一致
7. 逐 Phase 实施，每个 Phase 完成验收后标记 `✅ 已完成`
8. 全部完成后更新 `status` 为 `implemented`

详细规范见 `works/.meta-work-guide.md` 和 `.meta-knowledge-base-guide.md` §7.1。

---

## 4. 排查：troubleshooting 文档使用

```
Bug / 异常报告
  → 提取症状关键词（如「attach 失败」「缓存未命中」「.error 文件」）
  → 读 index.yaml，按 keywords 匹配 troubleshooting/ 文档
  → 若命中：读取排查文档，按步骤定位
  → 若未命中：
      → 匹配相关模块文档（covers_code / keywords）
      → 查对应模块文档 §6「已知陷阱」
      → 仍未找到 → 直接分析代码
```

---

## 5. 边界与注意事项

- **kb 是辅助，代码是唯一事实来源**。1-2 个文件的局部细节应直接读代码。发现文档与代码不一致时以代码为准，并提示用户 kb 需更新。
- **不重复代码内容**：模块文档记录设计决策、业务流程、已知陷阱——这些是代码里读不出来的东西。引用代码位置即可，不逐行复述。
- **pass/ 是构建期产物与外部输入的中转区**：只读参考，不入 index.yaml；构建 QA / Journal / build-findings 与外部输入都放这里。消化后转化为 works/ 或代码；`build-findings` 中未确认的条目不要当作 KB 知识使用。
- **与 CLAUDE.md 的关系**：CLAUDE.md 声明了 kb 基础使用行为（场景→目录映射、两层配合）。本 skill 提供深度操作（三级检索、规划流程、排查流程），不重复 CLAUDE.md 已声明的内容。
- **发现 KB 与代码不一致时**：(1) 立即以代码为事实源响应当前任务（零延迟）；(2) 静默追加一行 JSONL 到 `.think-tank/knowledge-base/.meta-explore-issues.jsonl`，格式：`{"kb_entry":"<entry-id>","section":"<§>","note":"<差异简述>","timestamp":"<ISO8601>"}`；(3) 建议用户后续调用 `/maintain-kb` 进行系统性修复。
- **kb 一致性维护**：调用 `/maintain-kb` skill 可进行增量/全量两种模式的 KB 与代码一致性检查与修复。本 skill 负责检索与发现问题，maintain-kb 负责系统性诊断与修复。
