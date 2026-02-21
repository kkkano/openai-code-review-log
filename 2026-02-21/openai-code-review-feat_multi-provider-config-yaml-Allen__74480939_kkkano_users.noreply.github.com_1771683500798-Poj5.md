## 1) Summary
本次改动是一个大型功能升级，主要目标是将代码评审系统从单一 ChatGLM 提供商扩展为支持多提供商（OpenAI-compatible）的架构。核心变更包括：
- **新增配置文件支持**：引入 `.github/code-review.yml` YAML 配置文件，用于定义提供商、API 端点、模型和提示词模板。
- **新增 OpenAI 兼容客户端**：实现 `OpenAICompatible` 类，支持与任何兼容 OpenAI API 的第三方服务（如 DeepSeek）集成。
- **重构配置加载**：新增 `ReviewConfig` 和 `ReviewConfigLoader` 类，支持从文件和环境变量（优先级：环境变量 > 配置文件）加载配置。
- **增强工作流**：更新了两个 GitHub Actions 工作流文件（`main-maven-jar.yml`, `main-remote-jar.yml`），升级了 Actions 版本、JDK 发行版，并注入了新的配置环境变量。
- **修复稳定性问题**：
    - 在 `GitCommand` 中修复了日志文件名可能包含非法字符（如 `/`, `<>`）的问题。
    - 将 `main-remote-jar.yml` 工作流从下载预构建的 JAR 改为从当前分支源码构建，确保代码一致性。
    - 在 Shaded JAR 的构建配置中补齐了 YAML 解析的依赖链（`jackson-dataformat-yaml`, `snakeyaml`）。
    - 在 `AbstractOpenAiCodeReviewService` 中，将微信通知改为仅在相关环境变量全部配置时才发送，避免因部分配置缺失导致流程失败。
- **完善文档**：新增了多个 Markdown 文档（`PR_BODY.md`, `bind-other-projects.md`, `configuration.md`, `deepseek-quickstart.md`），详细说明了新功能、配置方法和快速入门指南。
- **清理构建产物**：更新了 `.gitignore` 以忽略 `target/` 目录，并删除了 SDK 模块下的所有编译产出文件（`.class`, `.jar`）。

## 2) Risks
**Critical: 0**
**High: 1**
**Medium: 3**
**Low: 2**

### High
1.  **配置加载逻辑的健壮性**：`ReviewConfigLoader` 中的 `firstNonBlank` 方法用于合并环境变量和文件配置。如果同时设置了 `OPENAI_APIHOST` 和 `CHATGLM_APIHOST`，当前逻辑会优先使用 `OPENAI_APIHOST`。这通常是期望的行为，但缺乏明确的文档或日志说明此优先级，在调试多环境变量冲突时可能造成困惑。**建议**：在 `applyEnvOverrides` 方法中添加日志，记录最终生效的 `apiHost` 和 `apiKey` 来源。

### Medium
1.  **HTTP 客户端缺乏重试机制**：`OpenAICompatible.completions` 方法使用基本的 `HttpURLConnection`，没有设置连接超时、读取超时，也没有对网络波动或服务端临时错误（如 429、500）的重试逻辑。**建议**：至少添加合理的超时设置（例如，连接超时 10 秒，读取超时 30 秒）。对于生产环境，考虑引入带有退避策略的重试机制。
2.  **分支名获取逻辑不一致**：在 `main-remote-jar.yml` 中，获取分支名的命令变更为 `echo "BRANCH_NAME=${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}"`，这旨在更好地支持 Pull Request 事件。然而，`main-maven-jar.yml` 中仍使用旧命令 `echo "BRANCH_NAME=${GITHUB_REF#refs/heads/}"`。**建议**：统一两个工作流中的分支名获取逻辑，都使用支持 PR 的新语法，以确保在所有触发场景下行为一致。
3.  **硬编码的默认值**：`ReviewConfig` 中 `provider` 的默认值是 `"chatglm"`，而 `OpenAiCodeReview.buildOpenAiClient` 方法中 `provider` 的默认处理逻辑是 `"openai-compatible"`。虽然环境变量覆盖可以解决，但代码中的默认值不一致可能在新项目未配置时导致意外行为。**建议**：将 `ReviewConfig` 中的默认 `provider` 改为 `"openai-compatible"`，以与主逻辑保持一致，或确保默认值逻辑完全统一。

### Low
1.  **配置文件路径硬编码**：`ReviewConfigLoader` 中默认配置文件路径硬编码为 `".github/code-review.yml"`。虽然可以通过 `REVIEW_CONFIG_FILE` 环境变量覆盖，但该默认值可能与某些项目结构不符。**风险较低**，因为这是项目约定，且提供了覆盖方式。
2.  **依赖项版本管理**：`pom.xml` 中部分依赖（如 `jackson-dataformat-yaml`）的版本是硬编码的，未使用父 POM 或属性统一管理。如果未来升级其他 Jackson 组件版本，容易造成版本不一致。**建议**：将常用依赖的版本号提取到 `<properties>` 标签中统一管理。

## 3) Actionable Fixes
1.  **为配置加载添加日志**：
    - **文件**：`openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/types/config/ReviewConfigLoader.java`
    - **修改**：在 `applyEnvOverrides` 方法末尾，添加日志输出最终生效的配置项（至少包含 `provider`, `model`, `apiHost` 的来源）。例如：
      ```java
      logger.info("Review config loaded: provider={} (from {}), model={}, apiHost={} (from {})",
          config.getProvider(), config.getProviderSource(),
          config.getModel(), config.getApiHost(), config.getApiHostSource());
      ```
      （需要先在 `ReviewConfig` 中添加临时字段来记录来源，或直接在日志中说明是环境变量还是文件）。
2.  **为 HTTP 客户端添加超时设置**：
    - **文件**：`openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/infrastructure/openai/impl/OpenAICompatible.java`
    - **修改**：在 `completions` 方法中，`HttpURLConnection connection` 建立后，添加：
      ```java
      connection.setConnectTimeout(10000); // 10 seconds
      connection.setReadTimeout(30000); // 30 seconds
      ```
3.  **统一工作流中的分支名获取逻辑**：
    - **文件**：`.github/workflows/main-maven-jar.yml`
    - **修改**：将 `Get branch name` 步骤中的命令改为与 `main-remote-jar.yml` 一致：
      ```yaml
      run: echo "BRANCH_NAME=${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}" >> $GITHUB_ENV
      ```
4.  **统一默认提供商配置**：
    - **文件**：`openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/types/config/ReviewConfig.java`
    - **修改**：将 `provider` 字段的默认值从 `"chatglm"` 改为 `"openai-compatible"`，以与 `buildOpenAiClient` 方法中的默认逻辑保持一致。

## 4) Tests to Add
1.  **单元测试：`ReviewConfigLoader` 的配置加载与覆盖优先级**：
    - **场景**：创建临时 YAML 配置文件，设置一组值，然后通过环境变量设置另一组值，验证 `load()` 方法返回的配置是否正确遵循了“环境变量优先于文件”的规则。
    - **关注点**：`apiHost` 和 `apiKey` 从 `OPENAI_*` 和 `CHATGLM_*` 环境变量中正确选择。
2.  **单元测试：`GitCommand.safe()` 方法的文件名清理功能**：
    - **场景**：提供包含各种特殊字符（`/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`, ` `）以及超长字符串的输入，验证输出是否被正确替换为下划线且长度被合理截断。
    - **关注点**：确保生成的文件名在 Windows/Linux 文件系统下均合法。
3.  **集成测试：`OpenAICompatible` 客户端的错误处理**：
    - **场景**：使用 WireMock 或类似工具模拟 API 端点，返回不同的 HTTP 状态码（如 400, 401, 429, 500, 503）。验证客户端是否抛出了预期的异常，并且异常信息有助于诊断。
    - **关注点**：虽然当前没有重试，但至少应确保错误能被清晰捕获和记录。
4.  **工作流冒烟测试**：
    - **场景**：在测试仓库中，应用简化版的工作流（或使用 `act` 工具本地运行），使用一个模拟的、总是返回固定评审结果的 API 端点，验证整个流程（检出 -> 构建 -> 执行 SDK -> 写入日志）能否成功走通，且不会因微信配置缺失而失败。
    - **关注点**：验证 `isWeixinConfigured()` 逻辑在实际工作流环境中按预期工作。