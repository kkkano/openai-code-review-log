好的，我将以资深代码审查工程师的身份，对您提供的 `git diff` 输出进行结构化评审。

### **1) Summary**

本次提交是一个大型功能增强和重构，核心目标是**将代码审查工具从单一 ChatGLM 提供商升级为支持多提供商（OpenAI-compatible）**，并修复了多个关键稳定性问题。主要改动包括：

*   **新增配置系统**：引入了 `.github/code-review.yml` YAML 配置文件，支持定义 LLM 提供商、模型、API 端点等，并允许通过环境变量覆盖。
*   **新增 `OpenAICompatible` 客户端**：实现了兼容 OpenAI API 协议的客户端，可接入 DeepSeek 等第三方服务。
*   **重构工作流**：更新了两个 GitHub Actions 工作流文件，升级了 actions 版本，优化了构建和运行逻辑（如从当前分支构建 SDK，而非下载远程 JAR），并移除了冗余的调试输出步骤。
*   **增强文档**：新增了 `PR_BODY.md` 和多个 Markdown 文档（如 `docs/configuration.md`, `docs/deepseek-quickstart.md`），详细说明了新功能、配置方法和绑定到其他项目的步骤。
*   **关键稳定性修复**：
    *   修复了日志文件名因包含非法字符（如 `/`, `<`, `>`）导致写入失败的问题。
    *   修复了微信通知逻辑，仅在相关环境变量完整配置时才发送，避免因缺失配置导致整个流程失败。
    *   在 `pom.xml` 中补齐了 YAML 解析的依赖链（`jackson-dataformat-yaml` 和 `snakeyaml`）。
*   **代码结构优化**：将硬编码的配置逻辑重构为可配置的 `ReviewConfig` 和 `ReviewConfigLoader` 类，提高了代码的可维护性和可测试性。

### **2) Risks (Critical/High/Medium/Low)**

*   **Critical (严重)**: 无。
*   **High (高)**:
    *   **依赖冲突风险**：`pom.xml` 中新增了 `jackson-dataformat-yaml` 依赖，但 `dependency-reduced-pom.xml`（shade 插件生成）中的 `<includes>` 列表也更新了。需要确保 `snakeyaml` 依赖被正确打包进最终的可执行 JAR 中，否则在缺少该依赖的环境中运行会抛出 `ClassNotFoundException`。**已验证**：`pom.xml` 的 shade 配置已包含 `org.yaml:snakeyaml:`，风险已缓解。
    *   **环境变量覆盖逻辑**：`ReviewConfigLoader` 中，环境变量 `OPENAI_APIHOST` 和 `CHATGLM_APIHOST` 的优先级高于配置文件。如果用户无意中设置了错误的环境变量，可能导致意料之外的 API 端点调用。建议在日志中明确打印最终生效的配置来源。
*   **Medium (中)**:
    *   **分支名获取逻辑不一致**：`main-maven-jar.yml` 中使用 `${GITHUB_REF#refs/heads/}`，而 `main-remote-jar.yml` 和模板中使用 `${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}`。后者能更好地处理 Pull Request 事件，建议统一为后者以提高健壮性。
    *   **错误处理粒度**：`OpenAiCodeReview.getEnv` 方法在环境变量缺失时直接抛出 `RuntimeException` 并终止进程。对于非核心配置（如微信相关变量），可以考虑更优雅的降级处理（虽然本次已通过 `isWeixinConfigured` 修复了微信部分）。
    *   **API 调用缺乏重试机制**：`OpenAICompatible` 和 `ChatGLM` 客户端使用简单的 HTTP 调用，未实现网络波动或 API 限流（429）时的重试与退避逻辑，在不可靠的网络环境下可能失败。
*   **Low (低)**:
    *   **硬编码的默认值**：`ReviewConfig` 中 `provider` 默认值为 `”chatglm”`，而 `OpenAiCodeReview.buildOpenAiClient` 中 `provider` 为空时的默认逻辑是 `”openai-compatible”`。虽然环境变量通常会覆盖，但存在逻辑上的不一致。建议统一默认值。
    *   **示例配置中的占位符**：`docs/examples/code-review.yml` 中 `apiHost` 写的是 DeepSeek 的地址，但 `README.md` 和 `.github/code-review.yml` 中又是 OpenAI 的地址。这容易造成用户混淆，建议在示例中使用一个通用的注释说明。

### **3) Actionable Fixes**

1.  **统一分支名获取逻辑**：
    *   **文件**: `.github/workflows/main-maven-jar.yml`
    *   **修改**: 将 `BRANCH_NAME` 的赋值逻辑改为与 `main-remote-jar.yml` 一致。
    *   **代码**:
      ```yaml
      run: echo "BRANCH_NAME=${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}" >> $GITHUB_ENV
      ```

2.  **增强配置加载的日志输出**：
    *   **文件**: `openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/types/config/ReviewConfigLoader.java`
    *   **修改**: 在 `load()` 方法中，验证完成后或返回前，使用 `logger.info` 打印最终生效的核心配置（如 `provider`, `model`, `apiHost`），并注明是来自文件还是环境变量。这有助于调试。

3.  **修正默认 Provider 逻辑不一致**：
    *   **文件**: `openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/types/config/ReviewConfig.java`
    *   **修改**: 将 `provider` 字段的默认值从 `”chatglm”` 改为 `”openai-compatible”`，与 `buildOpenAiClient` 方法中的默认逻辑保持一致。
    *   **代码**:
      ```java
      private String provider = "openai-compatible";
      ```

4.  **优化示例文档**：
    *   **文件**: `docs/examples/code-review.yml`
    *   **修改**: 将 `apiHost` 的值改为一个带有注释的通用示例，避免直接指向某个具体服务商。
    *   **代码**:
      ```yaml
      apiHost: https://api.openai.com/v1/chat/completions # 替换为你的 OpenAI-compatible API 网关地址
      ```

### **4) Tests to Add**

1.  **单元测试 - `ReviewConfigLoader`**:
    *   **场景 1**: 测试当 `REVIEW_CONFIG_FILE` 指定的文件不存在时，是否返回一个包含默认值的 `ReviewConfig` 对象。
    *   **场景 2**: 测试环境变量 `OPENAI_APIHOST` 和 `OPENAI_APIKEY` 能否正确覆盖 YAML 文件中的配置。
    *   **场景 3**: 测试 `CHATGLM_APIHOST` 和 `CHATGLM_APIKEYSECRET` 的向后兼容性，确保它们也能被正确读取。
    *   **场景 4**: 测试当 `apiHost` 和 `apiKey` 均为空时，`validate` 方法是否按预期抛出异常。

2.  **单元测试 - `GitCommand.safe()` 方法**:
    *   **场景 1**: 输入包含 `/`、`<`、`>`、空格等特殊字符的字符串，验证输出是否被替换为下划线 `_`。
    *   **场景 2**: 输入超长字符串（>80字符），验证输出是否被正确截断。
    *   **场景 3**: 输入 `null` 或空字符串，验证是否返回 `”unknown”`。

3.  **集成测试 - 配置文件解析与客户端构建**:
    *   **场景**: 模拟一个完整的运行流程，给定一个包含 `provider: openai-compatible` 的 YAML 配置文件和相应的环境变量，验证 `OpenAiCodeReview.buildOpenAiClient` 方法是否能正确实例化 `OpenAICompatible` 客户端，并且其内部字段（`apiHost`, `apiKey`）与预期一致。

4.  **故障注入测试 - 微信通知降级**:
    *   **场景**: 在测试环境中，仅设置部分微信环境变量（如只设置 `WEIXIN_APPID` 和 `WEIXIN_SECRET`），验证 `isWeixinConfigured()` 方法是否返回 `false`，并且 `pushMessage` 不会被调用，同时能在日志中看到预期的警告信息。