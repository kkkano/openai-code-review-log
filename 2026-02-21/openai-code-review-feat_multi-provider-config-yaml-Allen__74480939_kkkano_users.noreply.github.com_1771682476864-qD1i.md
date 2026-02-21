## 1) Summary
本次提交主要引入了一个**多提供商 OpenAI 兼容 API 的配置系统**，并重构了代码审查 SDK 以支持通过 YAML 配置文件或环境变量动态选择 LLM 提供商（如 OpenAI、ChatGLM 或任何兼容 OpenAI API 的第三方服务）。核心变更包括：
- **新增配置文件** (`.github/code-review.yml`)：支持定义提供商、API 端点、模型和提示模板。
- **重构 SDK 核心**：引入了 `ReviewConfig` 和 `ReviewConfigLoader` 类来集中管理配置，并新增了 `OpenAICompatible` 客户端以支持通用 OpenAI 协议。
- **更新 GitHub Actions 工作流**：升级了 Actions 版本，简化了步骤，并集成了新的配置环境变量。
- **增强健壮性**：改进了错误处理、文件名安全处理和条件性微信通知。
- **更新文档**：新增了配置说明和 DeepSeek 快速入门指南。

## 2) Risks

### Critical (0)
无。

### High (1)
*   **配置加载与环境变量覆盖逻辑的复杂性** (`ReviewConfigLoader.java`)：
    *   **风险**：配置加载顺序（文件 -> 环境变量）和覆盖逻辑（如 `firstNonBlank`）可能引入非预期的行为。如果环境变量名错误或优先级混乱，可能导致运行时使用错误的 API 密钥或端点。
    *   **原因**：复杂的配置源（文件、多个可能的环境变量）增加了调试和维护的难度。

### Medium (2)
*   **硬编码的默认值和回退逻辑** (`OpenAiCodeReviewService.java`, `ReviewConfigLoader.java`)：
    *   **风险**：在 `OpenAiCodeReviewService` 中硬编码了默认模型 (`glm-4-flash`) 和提示模板。在 `ReviewConfigLoader` 中，默认的 `provider` 是 `"chatglm"`，而代码中 `OpenAiCodeReview.buildOpenAiClient` 的默认分支是 `openai-compatible`。这种不一致可能导致新用户在没有配置时，SDK 行为与预期不符。
    *   **原因**：默认值分散且可能冲突，缺乏统一的默认配置策略。
*   **Git 操作中的潜在资源泄漏与异常处理** (`GitCommand.java`)：
    *   **风险**：`Git` 对象在使用后没有在 `finally` 块中显式调用 `close()`。虽然 JGit 可能在某些情况下自动管理，但在长期运行或高频率调用的服务中，显式关闭是最佳实践。此外，`diff()` 方法中的 `Process` 流没有在 finally 中确保关闭。
    *   **原因**：资源管理不够严格，可能在极端情况下导致文件句柄或内存泄漏。

### Low (2)
*   **新增依赖未进行充分的兼容性测试** (`pom.xml`)：
    *   **风险**：新增了 `jackson-dataformat-yaml` 和间接引入的 `snakeyaml` 依赖。虽然已加入 shade 插件，但未在多样本环境中测试其与现有依赖（如特定版本的 fastjson2、guava）的兼容性。
    *   **原因**：依赖变更可能引入潜在的类冲突或运行时行为变化。
*   **文件名安全处理可能过于严格** (`GitCommand.java`):
    *   **风险**：`safe()` 方法将所有非字母数字、点、下划线、短横线的字符替换为下划线，并截断至80字符。虽然这避免了文件系统问题，但可能使文件名可读性下降（例如，邮箱地址中的 `@` 和 `+` 会被替换）。
    *   **原因**：文件名生成策略可能影响日志的可追溯性。

## 3) Actionable Fixes

1.  **统一并明确默认配置**：
    *   **修改文件**：`ReviewConfig.java` 和 `OpenAiCodeReviewService.java`。
    *   **具体操作**：将 `ReviewConfig` 中的默认 `provider` 改为 `"openai-compatible"`，以匹配 `OpenAiCodeReview.buildOpenAiClient` 中的默认分支。考虑将 `OpenAiCodeReviewService` 中的默认提示模板移至 `ReviewConfig` 作为默认值，确保单一事实来源。
    *   **验证**：启动 SDK 时不提供任何配置文件和环境变量，应抛出明确的错误（要求 `apiHost` 和 `apiKey`），而不是静默使用可能不正确的默认值。

2.  **简化并加固配置加载逻辑**：
    *   **修改文件**：`ReviewConfigLoader.java`。
    *   **具体操作**：
        a. 明确环境变量覆盖优先级文档。例如，`OPENAI_APIHOST` 总是覆盖 `CHATGLM_APIHOST`，无论 `provider` 设置如何？这需要根据设计意图澄清。
        b. 在 `validate` 方法中，增加对 `provider` 值的校验，只允许 `"chatglm"` 或 `"openai-compatible"`（或 `"openai"`），并对未知值给出友好错误提示。
        c. 考虑将 `firstNonBlank` 逻辑简化，优先使用与所选 `provider` 明确对应的环境变量。
    *   **验证**：编写单元测试，覆盖不同环境变量组合下的配置加载结果。

3.  **修复资源泄漏风险**：
    *   **修改文件**：`GitCommand.java`。
    *   **具体操作**：
        a. 在 `record` 方法中，将 `Git git = Git.cloneRepository()...` 的使用包装在 try-with-resources 语句中（如果 JGit 的 `Git` 实现了 `AutoCloseable`），或在 finally 块中调用 `git.close()`。
        b. 在 `diff()` 方法中，确保 `Process` 的 `InputStream` 和 `ErrorStream` 在 try-with-resources 中打开，或确保在 finally 中关闭。
    *   **验证**：在代码审查后，运行工具并监控进程资源使用情况，确保没有明显的泄漏。

4.  **增强错误信息的可读性**：
    *   **修改文件**：`OpenAiCodeReview.java` (`getEnv` 方法)。
    *   **具体操作**：当环境变量缺失时，抛出的异常信息可以更友好，例如：`"Required environment variable 'XXX' is not set. Please check your GitHub Actions secrets or workflow configuration."`
    *   **验证**：在 CI/CD 流水线中故意移除一个必要 Secret，观察错误输出是否清晰指引用户解决问题。

## 4) Tests to Add

1.  **单元测试：`ReviewConfigLoader` 测试套件**：
    *   **测试点**：
        *   从 YAML 文件正确加载所有字段。
        *   环境变量 `REVIEW_PROVIDER`, `REVIEW_MODEL`, `OPENAI_APIHOST` 等正确覆盖文件配置。
        *   `firstNonBlank` 逻辑在各种输入组合（包括 null、空字符串、有效值）下的行为符合预期。
        *   缺少 `apiHost` 或 `apiKey` 时，`validate` 方法抛出包含明确字段名的异常。
        *   提供未知的 `provider` 值时，应有相应的错误处理（可在 `buildOpenAiClient` 或 `validate` 中增加）。
    *   **位置**：在 `test` 目录下创建 `ReviewConfigLoaderTest.java`。

2.  **单元测试：`OpenAICompatible` 客户端**：
    *   **测试点**：
        *   使用 Mock 或 WireMock 模拟 HTTP 连接，测试其对标准 OpenAI 响应格式（包括错误格式，如 401、429）的解析和异常传播。
        *   验证 `authScheme` 参数正确应用于 `Authorization` 请求头（默认为 "Bearer"，但也支持自定义）。
    *   **位置**：在 `test` 目录下创建 `OpenAICompatibleTest.java`。

3.  **集成测试：端到端配置与执行流程**：
    *   **测试点**：
        *   模拟一个完整的代码审查流程，使用一个最小化的、可预测的 `git diff` 和 Mock 的 LLM 响应，验证从配置加载、diff 获取、AI 调用到结果记录（可 Mock Git 推送）的整个链条是否畅通。
        *   测试当微信环境变量不全时，`isWeixinConfigured()` 逻辑是否正确跳过通知，并记录警告日志。
    *   **位置**：可以作为一个集成测试类，例如 `OpenAiCodeReviewServiceIntegrationTest.java`。

4.  **安全与健壮性测试：`GitCommand.safe()` 方法**：
    *   **测试点**：
        *   输入包含特殊字符（`/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`, `@`, `+` 等）、超长字符串、空字符串、null 值时的输出。
        *   验证生成的字符串是否确实在常见文件系统（如 Linux ext4, Windows NTFS）上是安全的。
    *   **位置**：在 `test` 目录下创建 `GitCommandTest.java`。