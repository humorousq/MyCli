## MyCli 模块与领域对象划分

本文件从工程视角，列出 Client / Server 两侧主要模块，以及几个关键领域对象的职责与边界。

---

### Client 侧模块

#### `CliEntry`

- **职责**
  - 作为二进制入口，解析命令行参数；
  - 决定是进入 REPL，还是执行一次性命令（如 `run -s spec.yml`）。
- **典型子命令**
  - `mycli chat` / `mycli repl`
  - `mycli run -s spec.yml`

#### `CommandParser`

- **职责**
  - 解析 REPL 会话内的输入行，区分：
    - 前缀指令（如 `:spec`、`:scope`、`:plan`、`:apply` 等）；
    - 自然语言需求（走 SpecBuilder + LLM）。
- **行为示例**
  - `:spec`：查看/编辑当前 Spec；
  - `:scope services/user/`：设置/更新当前会话的 scope；
  - `:plan`：请求 Server 侧生成 plan；
  - `:apply`：请求 Server 侧应用最近的 patch。

#### `ReplSession`

- **职责**
  - 维护会话级状态：
    - 当前 Project（路径、Git 状态）；
    - 当前 Spec（可能来自文件或自然语言转换）；
    - 最近一次 Plan / Patch 集合；
    - 历史指令记录。
  - 将解析后的“意图”转发至 Server 侧服务，并收集返回结果。

#### `ViewRenderer`

- **职责**
  - 把来自 Server 的返回值渲染为终端输出：
    - 计划摘要（Plan）；
    - diff/patch 预览；
    - 错误与提示信息。
  - 尽量保持输出结构清晰，可滚动阅读，可复制粘贴。

---

### Server 侧模块

#### `ProjectService`

- **职责**
  - 统一管理「当前项目」的视图：
    - 确认路径下是否为 Git 仓库；
    - 提供文件枚举、读取、写入接口；
    - 暴露 Git 状态（当前分支、是否干净、未跟踪文件等）。
- **典型接口**
  - `get_project(root_path) -> Project`
  - `read_file(path) -> string`
  - `write_file(path, content)`（后续可附带安全检查）

#### `SpecService`

- **职责**
  - 管理 Spec 的整个生命周期：
    - 解析 Spec 文件（YAML/JSON/Markdown frontmatter）；
    - 校验字段完整性和基本约束；
    - 从自然语言生成或补全 Spec（通过 LLM 实现的 `SpecBuilder`）。
- **典型字段**
  - `meta`：版本、作者、日期等；
  - `goal`：本次修改的目标描述；
  - `scope`：涉及目录/文件/模块范围；
  - `constraints`：不允许触达的模块、性能/安全限制等；
  - `context`：补充业务/技术背景信息；
  - `tasks`/`steps`：可选的分步说明。

#### `ContextService`

- **职责**
  - 根据 `Spec.scope` / `Spec.context` 决定需要读取的文件集合；
  - 组装模型需要的上下文视图（文件路径 + 内容/片段）。
- **实现策略（可演进）**
  - v1：按路径读取完整文件；
  - v2：在文件级别上做截断（只取头尾/特定范围）；
  - v3（未来）：基于索引或语义搜索进行片段选择。

#### `PlanService`

- **职责**
  - 将 `Spec` 与 `Context` 转化为「可执行的变更计划」：
    - 调用 `PromptBuilder` 构造 Prompt；
    - 通过 `ModelGateway` 调用模型；
    - 将模型输出解析为 `Plan`（高层说明）+ `PatchDraft`（待进一步结构化的补丁草案）。

#### `PatchService`

- **职责**
  - 将 `PatchDraft` 转换为结构化补丁 `Patch`：
    - 按文件划分；
    - 提取每个文件的 hunks（增删改的行范围）。
  - 提供 dry-run 与 apply 两种模式：
    - dry-run：仅生成并展示 `git diff`，不改写文件；
    - apply：在确认后改写文件并调用 Git 创建 commit。

#### `ModelGateway`

- **职责**
  - 充当 LLM Provider 的统一网关：
    - 接收 `PromptBuilder` 产生的 Prompt；
    - 管理 Provider 配置（API Key、模型名、超时等）；
    - 统一处理重试、错误翻译和日志记录。
- **扩展性**
  - 初始可以支持单一 Provider（如 Claude 或 OpenAI）；
  - 后续可通过配置切换或增加多 Provider 策略。

#### `GitAdapter`

- **职责**
  - 与 Git 工具交互：
    - 计算当前工作区与 HEAD 的 diff；
    - 在 `PatchService` 应用补丁后，展示新的 diff；
    - 按约定策略创建 commit（如「每个 Spec/任务一个原子 commit」）。

---

### 关键领域对象（草案）

> 具体类型定义可以根据你选择的语言写在代码里，这里只描述语义。

- **`Project`**
  - 描述一个 Git 项目（根路径、当前分支、状态等）。
- **`Spec`**
  - 对「这次要做什么」的结构化描述，是整个系统的事实来源。
- **`Context`**
  - 为本次任务收集到的代码与配置上下文，是 Prompt 的一部分。
- **`Plan`**
  - 模型或规则产出的「变更方案」摘要，包括要改哪些文件、做哪些动作。
- **`Patch`**
  - 可直接应用到文件系统/Git 的补丁集，是 Plan 的具体化。
- **`Session`**
  - 一次 REPL 会话的状态集合（当前 Project、当前 Spec、最近 Plan/Patch、历史记录等）。

这些领域对象会贯穿 Client 与 Server 的交互，是后续代码设计中需要优先落地的类型。

