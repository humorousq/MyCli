## 模型网关与 Provider 抽象（ModelGateway）

MyCli 中与大模型交互的逻辑被集中在 `ModelGateway` 模块中，目的是：\n+
- 让上层业务（PlanService、SpecBuilder 等）不感知具体厂商与 API 细节；\n+
- 保证未来可以方便地切换或扩展 Provider；\n+
- 统一处理超时、重试、错误翻译与日志。

---

### 抽象接口（伪代码示意）

以下是一个与实现语言无关的接口草案，仅用于说明职责：

```text
interface ModelGateway {
  GeneratePlanAndPatchesResult generatePlanAndPatches(Spec spec, Context context);

  Spec refineSpecWithNaturalLanguage(Project project, string userCommand, Spec? baseSpec);
}
```

- **`generatePlanAndPatches`**
  - 输入：当前的 `Spec` 与 `Context`；
  - 输出：包含自然语言计划说明与补丁草案的结果对象：
    - `planText`: 面向用户的高层计划说明（可用于展示）；
    - `patchDrafts[]`: 每个文件级别的补丁草案（后续由 `PatchService` 结构化为 `Patch`）。
- **`refineSpecWithNaturalLanguage`**
  - 输入：当前 `Project` 上下文、用户的一句/多句自然语言指令，以及一个可选的 base Spec；
  - 输出：一个新的或被补全/修正后的 `Spec` 对象（供 `SpecService` 持久化或直接使用）。

具体的类型命名和字段可以在实现时根据语言做调整，但职责建议保持一致。

---

### Provider 配置与选择

`ModelGateway` 通过配置文件或环境变量来选择具体 Provider，例如：

- 环境变量示例：
  - `MYCLI_MODEL_PROVIDER=claude` / `openai` / `mock`；
  - `MYCLI_CLAUDE_API_KEY=...`；
  - `MYCLI_OPENAI_API_KEY=...`。
- 配置文件示例（伪 JSON/YAML）：

```yaml
provider: claude
model: claude-3.5-sonnet
timeout_ms: 60000
max_tokens: 4096
```

在实现中可以：\n+
- 根据配置实例化对应 Provider 客户端；\n+
- 将通用的 `generatePlanAndPatches` / `refineSpecWithNaturalLanguage` 调用分派给具体 Provider 实现。

---

### Provider 实现策略

#### 1. MockProvider（推荐先实现）

- 用于开发早期和测试：\n+
  - 不调用真实 LLM，只返回固定或基于简单规则构造的结果；\n+
  - 方便在无网络/无 Key 情况下开发 Core Engine 与 CLI。

#### 2. RealProvider（如 Claude / OpenAI）

- 封装真实 API 调用：\n+
  - 拼装提示词（Prompt），传入模型；\n+
  - 接收输出并解析为 `planText` + `patchDrafts` 或新的 `Spec`；\n+
  - 对异常、超时、部分响应等情况做统一处理。

---

### 与 PromptBuilder 的关系

`PromptBuilder` 与 `ModelGateway` 的职责划分建议如下：\n+
- `PromptBuilder`：\n+
  - 只负责「如何和模型说话」——即把 `Spec` + `Context` 组织成 Prompt 文本或结构化请求；\n+
  - 可以针对不同任务（生成 plan/patch、补全 Spec 等）提供不同模板；\n+
  - 在 debug 模式下，可以将 Prompt 输出到日志/终端。\n+
- `ModelGateway`：\n+
  - 只负责「怎么调用模型」——即选择 Provider、模型参数、调用方式、重试与错误处理；\n+
  - 不参与 Prompt 内容本身的设计。

这种划分有利于：\n+
- Prompt 调优时只需关注 `PromptBuilder`；\n+
- 更换 Provider 时只需修改 `ModelGateway` 对应实现。

