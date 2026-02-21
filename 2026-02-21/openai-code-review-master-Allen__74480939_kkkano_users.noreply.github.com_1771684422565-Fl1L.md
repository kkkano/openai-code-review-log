好的，我将以资深代码审查工程师的身份，对您提供的 `git diff` 进行结构化评审。

### 1) Summary
本次提交是一个大型功能更新，核心目标是**将代码评审系统从单一 ChatGLM 提供商升级为支持多提供商（OpenAI-compatible）**，并修复了多个稳定性问题。主要改动包括：
*   **新增配置系统**：引入 `.github/code-review.yml` YAML 配置文件，支持定义 provider、model、prompt 等，并允许通过环境变量覆盖。
*   **新增 OpenAI 兼容客户端**：添加 `OpenAICompatible` 类，支持调用任何符合 OpenAI Chat Completions 协议的 API（如 DeepSeek）。
*   **重构工作流**：更新了两个 GitHub Actions 工作流文件，升级了 actions 版本，移除了冗余步骤，并改为构建当前分支的 SDK Jar 以确保代码一致性。
*   **增强健壮性**：
    *   修复了 Git 日志文件名中因分支名、作者名包含非法字符（如 `/`, `<>`）导致的问题。
    *   微信通知改为仅在配置齐全时才发送，避免因部分配置缺失导致整个流程失败。
    *   在 Shaded Jar 中补齐了 YAML 解析的依赖链。
*   **完善文档**：新增了大量文档（`PR_BODY.md`、`README.md` 更新、`docs/` 目录下的多个文件），详细说明了新功能、配置方法和绑定到其他项目的步骤。

### 2) Risks（Critical/High/Medium/Low）
*   **Medium - 配置加载逻辑的健壮性**：`ReviewConfigLoader` 中的 `firstNonBlank` 方法逻辑正确，但环境变量覆盖顺序（`OPENAI_*` 优先于 `CHATGLM_*`）需要在文档中明确说明，避免用户混淆。如果 `apiHost` 或 `apiKey` 最终仍为空，验证会抛出异常，这是正确的。
*   **Medium - 依赖管理**：`pom.xml` 和 `dependency-reduced-pom.xml` 中新增了 `jackson-dataformat-yaml` 依赖，并确保 `snakeyaml` 被 shade 插件包含。需要确认最终生成的 fat jar 确实包含了所有必要依赖，否则在未提供这些依赖的环境中运行会失败。
*   **Low - 硬编码的默认值**：`OpenAiCodeReviewService` 中硬编码了 `DEFAULT_PROMPT_TEMPLATE` 和默认 model (`glm-4-flash`)。虽然可以通过配置覆盖，但这些默认值可能与新 provider（如 DeepSeek）的最佳实践不匹配。建议将默认 prompt 也放入 `ReviewConfig` 的默认值中。
*   **Low - 错误信息可读性**：`OpenAiCodeReview.getEnv()` 方法在环境变量缺失时抛出异常信息为 `“value is null: ” + key`。对于用户来说，可以更友好一些，例如提示该变量需要在 GitHub Secrets 中配置。
*   **Low - 工作流中的潜在不一致**：`main-remote-jar.yml` 工作流中 `REVIEW_PROVIDER` 被硬编码为 `openai-compatible`，而 `main-maven-jar.yml` 中是从 secret 读取。这可能导致两个工作流行为不一致。建议统一从 secret 读取，或在配置文件中指定。

### 3) Actionable Fixes
1.  **增强配置加载的清晰度**：在 `ReviewConfigLoader.validate()` 方法抛出的异常信息中，更明确地提示用户检查位置。例如：“API host is required. Please set it in `.github/code-review.yml` (apiHost) or via environment variable `OPENAI_APIHOST`/`CHATGLM_APIHOST`.”
2.  **移除代码中的硬编码默认值**：将 `OpenAiCodeReviewService.DEFAULT_PROMPT_TEMPLATE` 和默认 model 字符串移至 `ReviewConfig` 类中作为字段的默认值（例如 `private String model = “glm-4-flash";`）。这样所有默认值都在配置对象中集中管理。
3.  **统一工作流配置**：修改 `.github/workflows/main-remote-jar.yml`，将 `REVIEW_PROVIDER: openai-compatible` 改为 `REVIEW_PROVIDER: ${{ secrets.REVIEW_PROVIDER }}`，与 `main-maven-jar.yml` 保持一致，并通过 Secrets 管理。
4.  **补充关键环境变量检查**：在 `OpenAiCodeReview.main()` 方法开始时，可以添加一个日志输出，打印出加载后的 `ReviewConfig` 的核心内容（如 provider, model, apiHost 掩码），便于调试和确认配置加载正确。
5.  **文件名安全处理增强**：`GitCommand.safe()` 方法将非法字符替换为下划线，并截断长度。考虑将空格也替换为下划线，并确保替换后的字符串不会以 `.` 或 `-` 开头结尾（在某些系统可能有问题），可以添加一个修剪步骤。

### 4) Tests to Add
1.  **单元测试 - `ReviewConfigLoader`**：
    *   测试从 YAML 文件正常加载。
    *   测试环境变量 `OPENAI_APIHOST` 正确覆盖 YAML 中的 `apiHost`。
    *   测试环境变量 `CHATGLM_APIKEYSECRET` 在 `OPENAI_APIKEY` 不存在时被用作 `apiKey`。
    *   测试当 `apiHost` 和 `apiKey` 均为空时，`validate()` 方法抛出预期异常。
2.  **单元测试 - `GitCommand.safe()` 方法**：
    *   测试包含 `/`、`\`、`:`、`<`、`>`、`|`、`?`、`*` 等非法字符的输入。
    *   测试过长的输入被正确截断。
    *   测试空输入返回 `”unknown”`。
3.  **单元测试 - `OpenAICompatible` 客户端**：
    *   测试 `authScheme` 参数为空时默认使用 `”Bearer”`。
    *   模拟 HTTP 连接，测试请求头是否正确设置（`Authorization`, `Content-Type`, `User-Agent`）。
4.  **集成测试 - 配置与客户端构建**：
    *   测试 `OpenAiCodeReview.buildOpenAiClient()` 方法，根据不同的 `provider` 字符串（`”chatglm”`, `”openai-compatible”`, `”openai”`, 默认值）返回正确的客户端实例类型。
5.  **文档测试**：为 `docs/examples/` 目录下的示例配置文件和工作流模板添加简单的语法验证或占位符检查，确保示例在复制粘贴后不易出错。