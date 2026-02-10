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

