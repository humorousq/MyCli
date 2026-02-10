## MyCli · Claude Code 风格终端助手（C/S 架构 + Spec 内核）

MyCli 是一个面向开发者的命令行工具，目标是 **在终端里复刻 Claude Code 的体验**：

- `mycli` 启动后进入一个对话式 REPL，会话内用自然语言 + 前缀命令和它沟通；
- 工具理解当前 Git 仓库、规划修改方案、生成 diff，并在你确认后改代码、创建 commit；
- 内部一切决策都以 **结构化 Spec** 为事实来源，但对外呈现为「对话式 + 命令式」交互。

更直白地说：这是一套 **Claude Code Terminal 版 + 可学习的后端实现**，专门给工程师阅读与复用。

---

### 从工程视角看 MyCli

- **逻辑 C/S 架构**：
  - **Client（C）**：终端 CLI + REPL
    - 负责解析命令行参数；
    - 提供交互式会话（`mycli chat` / `mycli repl`）；
    - 渲染 plan、diff、错误与提示；
    - 维护本次会话的前端状态（当前项目、当前 Spec、当前 plan 等）。
  - **Server（S）**：本地 Core Engine + LLM 网关
    - 管理项目上下文（Git 仓库、文件系统）；
    - 管理 `Spec`、`Plan`、`Patch`、`Session` 等领域对象；
    - 调用模型（Claude / OpenAI 等），构造 Prompt 并解析结果；
    - 生成并应用 patch，驱动 Git 完成 diff 与 commit。

详细架构见 `docs/architecture/overview.md` 与 `docs/architecture/modules.md`。

---

### 典型使用体验（目标形态）

> 下面是我们要逼近的终态体验，目前尚未实现，只在文档中约定。

- **方式一：对话式使用（接近 Claude Code）**
  - 在项目根目录执行：
    - `mycli chat`
  - 在 REPL 中：
    - 输入自然语言需求：`给 user 服务加一个重置密码接口`；
    - 用前缀命令控制行为：
      - `:scope services/user/` 限定修改范围；
      - `:plan` 生成此次修改的 plan 与候选 patch；
      - `:apply` 展示 diff 并在确认后应用与 commit。

- **方式二：显式 Spec 驱动（更工程化的用法）**
  - 编写 `specs/add-reset-password.yml`，描述目标、范围、约束等；
  - 运行：
    - `mycli run -s specs/add-reset-password.yml`
  - 工具会：
    - 解析 Spec；
    - 收集相关上下文；
    - 调用模型生成 plan + patch；
    - 展示 diff，询问是否应用，并按约定策略创建原子 commit。

---

### 文档导航（按角色和关注点组织）

- **整体愿景与路线**
  - `docs/vision.md`：为什么要做一个 Claude Code 风格的终端助手？目标用户是谁？
  - `docs/roadmap.md`：按工程节奏划分的阶段目标（架构定稿 → 核心骨架 → REPL → 模型接入 → 体验优化）。

- **架构与模块设计**
  - `docs/architecture/overview.md`：C/S 架构、核心数据流（Spec → Context → Plan → Patch → Git）。
  - `docs/architecture/modules.md`：Client / Server 各模块职责与接口草案。
  - `docs/architecture/model-providers.md`：模型网关（ModelGateway）与 Provider 抽象。

- **Spec 与工作流（稍后补充）**
  - 计划中的文档（名称可能微调）：
    - `docs/spec/overview.md`：Spec 在系统中的角色与哲学。
    - `docs/spec/schema.md`：Spec 字段定义（`goal`、`scope`、`constraints`、`context` 等）。
    - `docs/spec/examples.md`：典型用例对应的完整 Spec 示例。

- **CLI & REPL 使用（稍后补充）**
  - 计划中的文档：
    - `docs/cli/commands.md`：`mycli chat` / `mycli run` 等命令与参数。
    - `docs/cli/workflows.md`：从「一次自然语言/Spec」到「一个已提交的改动」的端到端流程。

- **示例项目（稍后补充）**
  - `examples/README.md`：示例列表与说明（如 basic-refactor、add-feature 等场景）。

---

### 给开发者的「从哪开始写代码」指南

如果你现在就想开写，可以按下面的顺序切入（在选择语言/技术栈后）：

- **第一步：实现 Server 核心骨架（后端角色）**
  - 定义领域对象：`Spec`、`Plan`、`Patch`、`Session`、`Project` 的类型。
  - 实现最小版本的：
    - `ProjectService`：识别当前 Git 仓库、读写文件。
    - `SpecService`：从 YAML 解析 Spec（先支持最基础字段）。
    - `ContextService`：根据 Spec.scope 读取相关文件全文。
    - `PatchService` + `GitAdapter`：能在 dry-run 下打印一个“假 patch”并展示 git diff。

- **第二步：实现 CLI & REPL 骨架（CLI 角色）**
  - 做出 `mycli chat` / `mycli run` 的基本流程。
  - 在 REPL 里先支持最小指令集：`:spec`、`:plan`、`:apply`，内部先调用 mock 的 Server。

- **第三步：接入真实模型 & 打磨 Spec（LLM 角色）**
  - 按 `docs/architecture/model-providers.md` 的约定接入 ModelGateway。
  - 用一个小示例项目跑通「真实模型 + 真实 patch」的完整链路。

这些步骤在 `docs/roadmap.md` 中也有更正式的阶段描述，你可以根据自己的时间和兴趣选择当前要戴的那顶“工程师帽子”。

## MyCli · Spec Coding Claude Code 复刻

MyCli 是一个面向开发者的命令行工具，目标是 **以 Spec Coding 的方式，复刻 Claude Code 的核心编辑循环**：

- 用户通过 **Spec 文件** 精确描述本次改动的目标、范围和约束；
- MyCli 负责读取当前 Git 仓库上下文，构造对大模型友好的 Prompt；
- 大模型产出变更计划与补丁（patch），MyCli 展示 diff，用户确认后再真正修改代码并提交。

这一项目更重要的价值在于 **学习 Claude Code 的原理与架构思想**，而不仅仅是做一个“能写代码的 CLI”。

---

### 项目状态

- 当前阶段：**只在搭建文档与规划，不写任何业务代码**。
- 目标：先把 Spec、CLI、架构、迭代路线全部在文档中说清楚，再进入实现阶段。

---

### 快速认识 MyCli（MVP 视角）

> 下述行为是目标形态，当前阶段尚未实现，只在文档中约定。

- **输入**：
  - 一个本地 Git 仓库；
  - 一份描述需求的 Spec 文件（例如 `specs/add-endpoint.yml`）。
- **核心命令**（规划中）：
  - `mycli plan -s spec.yml`：读取 Spec 与仓库上下文，调用模型生成变更计划与 patch 文件；
  - `mycli apply -p plan.json`：展示 diff，用户确认后应用 patch，并在合适的粒度上创建 git commit；
  - `mycli run -s spec.yml`：串联 `plan + apply`，形成一次完整的「Spec → 变更」闭环。
- **特性**：
  - 聚焦 **Spec 驱动**，不做随意“闲聊式 coding”；
  - 所有变更都通过 diff 呈现，可审查、可回滚；
  - 内部模块化设计，方便阅读与学习 Claude Code 风格的架构。

---

### 文档导航

- 愿景与路线
  - `docs/vision.md`：为什么要做 Spec Coding 版 Claude Code 复刻？目标用户是谁？
  - `docs/roadmap.md`：从 MVP 到后续版本的阶段性目标。
- Spec 设计
  - `docs/spec/overview.md`：什么是 MyCli 的 Spec？它解决什么问题？
  - `docs/spec/schema.md`：Spec 字段定义与语义说明。
  - `docs/spec/examples.md`：典型用例对应的完整 Spec 示例。
- CLI 使用
  - `docs/cli/commands.md`：命令行接口设计（命令、参数、返回结构）。
  - `docs/cli/workflows.md`：从 Spec 到代码改动的端到端工作流。
- 架构与实现
  - `docs/architecture/overview.md`：高层架构与数据流（Spec → Context → Prompt → Patch → Git）。
  - `docs/architecture/modules.md`：核心模块职责说明。
  - `docs/architecture/model-providers.md`：模型 Provider 抽象与配置思路。
- 示例
  - `examples/README.md`：示例场景列表及说明。

---

### 贡献与开发（规划中）

详细的开发与贡献指南会放在：

- `CONTRIBUTING.md`
- `docs/dev/testing.md`
- `docs/dev/releasing.md`

在正式写代码前，可以优先阅读上述文档，理解整个 Spec Coding 流程与架构设计。

