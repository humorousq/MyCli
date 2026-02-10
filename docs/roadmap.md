## MyCli 路线图（Roadmap）

本路线图聚焦于「复刻 Claude Code 核心编辑循环」这一目标，围绕 **Spec → Context → Prompt → Patch → Git** 的闭环，按阶段推进。

> 当前阶段只在文档中记录路线图，代码实现会在文档稳定后逐步进行。

---

### 阶段 0：愿景、用例与路线图（当前阶段）

- 明确项目愿景与目标用户（见 `docs/vision.md`）。
- 梳理并记录 3–5 个典型用例故事（见 `docs/spec/examples.md` / `docs/cli/workflows.md`）：
  - 重构现有模块；
  - 为服务加一个小功能/API；
  - 为已有代码补测试；
  - 批量重命名函数/组件；
  - （可选）结构化重写部分文档。
- 搭建文档结构：Spec、CLI、架构、示例等。

**完成标志**：不看任何代码，只看文档，就能说清楚「MyCli 要做什么，不做什么」以及接下来的实施路线。

---

### 阶段 1：CLI 形状与命令契约（文档优先）

- 在 `docs/cli/commands.md` 中，设计并固定核心命令：
  - `mycli init`
  - `mycli plan -s spec.yml`
  - `mycli apply -p plan.json`
  - `mycli run -s spec.yml`
- 约定每个命令的：
  - 输入参数；
  - 输出结构（stdout / 文件）；
  - 退出码语义；
  - 典型用例场景。
- 在 `docs/cli/workflows.md` 中描述 1–2 条端到端工作流。

**完成标志**：把 CLI 当作一个「对外 API」，在实现前就已经完全通过文档约定好行为。

---

### 阶段 2：Spec 模型设计（Spec Coding 的核心）

- 在 `docs/spec/overview.md` 中解释：
  - 什么是 Spec Coding；
  - 为什么要用 Spec 而不是自由聊天；
  - 自然语言需求是如何被整理成 Spec 的。
- 在 `docs/spec/schema.md` 中定义 Spec 字段：
  - `meta`、`goal`、`scope`、`constraints`、`context`、`steps/tasks` 等；
  - 说明每个字段在整体工作流中的作用。
- 在 `docs/spec/examples.md` 中，为每个典型用例给出至少一个完整 Spec 示例。

**完成标志**：看到一份 Spec，就能大致判断 MyCli 会做什么以及不会做什么。

---

### 阶段 3：上下文收集与 Git 交互设计（文档 & 原型）

- 在 `docs/architecture/overview.md` 中，用图和文字描述：
  - `SpecFile`、`ContextCollector`、`GitRepo`、`PromptBuilder` 等模块之间的数据流；
  - 如何从 `spec.scope` / `spec.context` 推导需要读取的文件。
- 在 `docs/architecture/modules.md` 中，为 `ContextCollector` 与 Git 相关逻辑写清职责：
  - 只负责「读」和「选择片段」，不直接改写文件；
  - 只支持本地 Git 仓库；
  - 配合「新文件/改动完成后尽快 commit」的工作习惯。

（实现阶段可以先做简单版：按路径读取完整文件，再慢慢演进到更智能的片段选择。）

---

### 阶段 4：PromptBuilder & ModelProvider 抽象

- 在 `docs/architecture/model-providers.md` 中：
  - 定义与具体厂商无关的 `ModelProvider` 接口（伪代码和文字解释）；
  - 写出未来可支持的 Provider（Claude、OpenAI 等）和配置方式。
- 在 `docs/architecture/modules.md` 中，为 `PromptBuilder` 设计说明：
  - 如何把 Spec 字段与上下文组织成稳定的 Prompt 模板；
  - 如何保持 Prompt 结构清晰、易于调优。

**完成标志**：即使还没具体实现，也已经知道「要写怎样的 Model 调用层」以及「Prompt 的结构与演进方向」。

---

### 阶段 5：Patch 结构与 Git 策略

- 在 `docs/architecture/modules.md` 中定义：
  - `PatchPlanner`：将模型输出转换为结构化 patch（文件 + hunks）；
  - `PatchApplier`：
    - dry-run：只展示 diff；
    - apply：真正改写文件，并生成 `git diff` 供确认；
    - commit 策略：如何围绕一次 Spec/一次 run 生成原子 commit。
- 在 `docs/cli/workflows.md` 中补充：
  - `mycli plan` 产生的 plan/patch 文件格式；
  - `mycli apply` 读取 plan/patch 并驱动 Git 的流程。

**完成标志**：在纸面上已经清楚地知道每一步会如何影响工作区和 Git 历史。

---

### 阶段 6：端到端示例与未来演进

- 在 `examples/README.md` 中列出示例场景及结构；
- 为一个或多个示例项目设计完整的：初始状态 → Spec → 预期改动 故事；
- 在 README 中补充「从 0 到 1」教程，指向具体示例。

**完成标志**：任何新读者只看文档和示例，就能理解整个 Spec Coding 流程，并知道如何在自己项目中尝试类似模式。

---

### 更远期的可能方向（待定）

以下方向暂时不会纳入 MVP，但可在未来演进中探索：

- 更智能的上下文选择（语义搜索、代码索引）；
- 模块级或函数级的影响分析；
- 与 IDE 或 GitHub Apps 的集成；
- 更丰富的回滚/审计 UI。

