## MyCli 路线图（Roadmap）

本路线图从一个完整的软件项目视角出发，围绕 MyCli 的 C/S 架构与核心链路：

> 自然语言/Spec → Context → Plan → Patch → Git

按「阶段 → 角色 → 产出」的方式拆解迭代步骤。

---

### 阶段 1：架构与领域模型定稿（主要是架构/后端角色）

- **目标**
  - 用文档和类型定义，把系统的「说法」统一起来。
- **主要工作**
  - 在 `docs/architecture/overview.md` 中：
    - 固定 C/S 架构图（Client CLI/REPL + Server Core Engine + ModelGateway + GitAdapter）。
  - 在 `docs/architecture/modules.md` 中：
    - 定义核心服务：`ProjectService`、`SpecService`、`ContextService`、`PlanService`、`PatchService`、`ModelGateway`、`GitAdapter` 的职责；
    - 给出领域对象：`Spec`、`Plan`、`Patch`、`Session`、`Project` 的字段/类型草案。
  - 明确 Client ↔ Server 之间的调用接口（函数签名或未来可演进为 HTTP/gRPC 的契约）。
- **完成标志**
  - 不写任何代码，只看 README + vision + architecture 文档，就能画出自己的架构图和类型草图。

---

### 阶段 2：Server 核心骨架（后端 / Core Engine 角色）

- **目标**
  - 做出最小可运行的 Core Engine：不接真实模型，先用 mock 把链路打通。
- **主要工作**
  - 实现 `ProjectService`：
    - 校验当前目录是否 Git 仓库；
    - 提供读/写文件接口；
    - 提供「当前分支、工作区是否干净」等状态查询接口。
  - 实现 `SpecService`：
    - 基于 YAML/JSON 解析最小 Spec（先支持 `goal`、`scope`、`constraints`、`context` 等基础字段）；
    - 做基础校验并返回内部 `Spec` 对象。
  - 实现 `ContextService`：
    - 按 `Spec.scope` 列表，读取相关文件全文（暂不做精细片段选择）；
    - 将结果以结构化形式返回。
  - 实现 `PatchService` + `GitAdapter` 最小版本：
    - 先用假 patch（如追加一行注释）验证 diff 打印与文件改写流程；
    - 支持 dry-run（只打印 diff）和 apply（真改文件），但可以暂不自动 commit。
- **完成标志**
  - 能调用一组 Server API，在一个示例仓库中完成一次「假 patch」的应用和 diff 展示。

---

### 阶段 3：CLI & REPL 骨架（CLI & 交互角色）

- **目标**
  - 搭出和 Claude Code 类似的终端交互壳子：`mycli chat` / `mycli run`。
- **主要工作**
  - 在 CLI 层：
    - 实现 `mycli chat` / `mycli repl`：进入 REPL 会话；
    - 实现 `mycli run -s spec.yml`：一次性执行「读取 Spec → 调 Server → 打印 diff」流程。
  - 在 REPL 内：
    - 实现基础指令解析（`CommandParser`）：
      - `:spec`：查看当前 Spec；
      - `:plan`：生成 plan（此时仍可用 mock plan）；
      - `:apply`：调用 `PatchService` 应用 patch；
      - 其他文本按「自然语言意图」处理，先简单 echo 或打日志。
  - 在输出渲染层（`ViewRenderer`）：
    - 提供基础的 plan/diff/错误信息渲染。
- **完成标志**
  - 在一个示例仓库中，用户可以：
    - 运行 `mycli chat`，敲入 `:plan` / `:apply` 等指令；
    - 看到清晰的打印信息和（即便是假的）diff。

---

### 阶段 4：接入真实模型 & 打磨 Spec（LLM 集成 + 后端角色）

- **目标**
  - 用真实 LLM 替换 mock，让 MyCli 真正具备「理解需求 → 生成 patch」的能力。
- **主要工作**
  - 在 `docs/architecture/model-providers.md` 中：
    - 固定 `ModelGateway` 接口：如 `generatePlanAndPatches(spec, context) -> Plan + Patch[]`；
    - 约定 Provider 配置方式（环境变量 / 配置文件）。
  - 实现 `ModelGateway`：
    - 支持至少一个 Provider（如 Claude 或 OpenAI）；
    - 处理超时、重试、错误信息回传。
  - 实现 `PromptBuilder`：
    - 将 `Spec` + `Context` 组织成稳定 Prompt 模板；
    - 支持 debug 模式下打印 Prompt 便于调优。
  - 改造 `PlanService`：
    - 从调用 mock 改为调用真实 `ModelGateway`；
    - 把模型输出解析成结构化 `Plan` 与 `Patch`。
- **完成标志**
  - 在一个小型真实项目上，按照 examples 中的示例：
    - 写一份 Spec 或在 `mycli chat` 中下达一个简单需求；
    - 工具能产出有意义的 plan 与 patch，并安全地在本地应用。

---

### 阶段 5：体验优化、Git 策略与测试（CLI + 测试角色）

- **目标**
  - 让 MyCli 从「能用」变成「敢用」「好用」。
- **主要工作**
  - Git 策略：
    - 固化「每个 Spec / 任务对应一个原子 commit」的策略；
    - 约定 commit message 格式（含 Spec 名称/ID、简要说明）。
  - REPL 体验：
    - 改进错误提示、确认提示、diff 高亮和分页；
    - 增加常见快捷命令（如 `:help`、`:history` 等）。
  - 测试与质量：
    - 为 `Spec → Context → Plan → Patch → Git` 全链路增加集成测试；
    - 为关键模块（SpecService、ContextService、PatchService、ModelGateway）增加单元测试；
    - 为 CLI/REPL 流程设计端到端测试脚本。
- **完成标志**
  - 你愿意在自己日常的小需求上实际使用 MyCli，而不是只把它当 Demo。

---

### 阶段 6：示例、文档与未来演进（文档 + 产品化角色）

- **目标**
  - 让其他工程师可以轻松理解和上手 MyCli。
- **主要工作**
  - 示例：
    - 在 `examples/README.md` 中列出示例场景（basic-refactor、add-feature、add-tests 等）；
    - 为每个示例写清：初始状态 → 使用的 Spec/指令 → 预期改动。
  - 文档：
    - 在 README 中补充「从 0 到 1 教程」，指向某个具体 example；
    - 在 `docs/cli/workflows.md` 中，用故事形式走一遍完整流程。
  - 未来方向（可选）：
    - 更智能的上下文选择（索引、语义搜索）；
    - 更丰富的任务管理与多轮会话支持；
    - 与 IDE、GitHub Apps 或自建平台的集成。
- **完成标志**
  - 一个第一次接触 MyCli 的工程师，按 README 操作一次，就能在本地跑通一个完整示例并大致读懂架构。

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

