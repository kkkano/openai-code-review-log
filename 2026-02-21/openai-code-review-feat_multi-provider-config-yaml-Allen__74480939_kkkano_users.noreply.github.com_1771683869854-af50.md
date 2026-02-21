## 1) Summary
本次提交是一个大型重构，旨在将 `openai-code-review` 项目从单一的 ChatGLM 提供商升级为支持多提供商（OpenAI-compatible）的代码审查系统。核心改动包括：
*   **新增配置系统**：引入 YAML 配置文件（`.github/code-review.yml`）和环境变量覆盖机制，支持动态切换 LLM 提供商（如 `openai-compatible` 或 `chatglm`）。
*   **新增 OpenAI 兼容客户端**：实现了 `OpenAICompatible` 类，用于调用符合 OpenAI Chat Completions 协议的 API（例如 DeepSeek）。
*   **增强工作流**：更新了 GitHub Actions 工作流（`main-maven-jar.yml`, `main-remote-jar.yml`），升级 Actions 版本，移除冗余步骤，并注入新的配置环境变量。
*   **修复稳定性问题**：
    *   修复了 Git 日志文件名中因分支名、作者名包含非法字符（如 `/`, `<>`）导致的问题。
    *   将远程 JAR 工作流改为从当前分支源码构建 SDK，确保代码一致性。
    *   在 Shaded JAR 中补齐了 YAML 解析的依赖链（`jackson-dataformat-yaml`, `snakeyaml`）。
*   **改进用户体验**：
    *   微信通知改为条件触发（仅在配置完整时发送），避免因缺失配置导致整体失败。
    *   新增了大量文档（`docs/` 目录），包括配置说明、快速入门指南和可复用的工作流模板。
    *   更新了 README，清晰地说明了多提供商支持和项目绑定方法。

## 2) Risks
*   **Critical**: 无。
*   **High**:
    *   **依赖管理风险**：`openai-code-review-sdk/pom.xml` 中新增了 `jackson-dataformat-yaml` 依赖，但对应的 `snakeyaml` 是传递依赖。在 `maven-shade-plugin` 的 `<includes>` 中显式包含了 `org.yaml:snakeyaml:`，这确保了它被打包。然而，如果未来 `jackson-dataformat-yaml` 版本升级且改变了传递依赖，可能导致运行时 `ClassNotFoundException`。**建议**：在 `pom.xml` 的 `<dependencies>` 中也显式声明 `snakeyaml` 依赖，并指定版本。
    *   **配置加载错误处理**：`ReviewConfigLoader.loadFromFile()` 中，如果配置文件不存在，会静默返回一个 `new ReviewConfig()`。这可能导致用户因拼写错误或路径错误而意外使用默认配置（`chatglm`），而非预期的 `openai-compatible`，从而引发混淆。**建议**：当 `REVIEW_CONFIG_FILE` 被显式设置但文件不存在时，记录一个 `WARN` 级别的日志。
*   **Medium**:
    *   **HTTP 客户端灵活性**：`OpenAICompatible` 使用基础的 `HttpURLConnection`。它缺乏连接池、重试机制和更灵活的超时设置。在高频调用或网络不稳定的环境下，可能影响可靠性。**建议**：考虑引入一个轻量级 HTTP 客户端（如 OkHttp）或至少为连接和读取设置合理的超时。
    *   **分支名解析逻辑不一致**：在两个工作流文件中，获取 `BRANCH_NAME` 的逻辑略有不同。`main-maven-jar.yml` 使用 `${GITHUB_REF#refs/heads/}`，而 `main-remote-jar.yml` 使用 `${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}`。后者能更好地处理 Pull Request 事件。**建议**：统一使用 `main-remote-jar.yml` 中的逻辑，以提高在 PR 场景下的准确性。
    *   **硬编码的默认模型**：`OpenAiCodeReviewService` 构造函数的默认模型仍为 `"glm-4-flash"`，这与新的多提供商导向不符。虽然新的构造函数重载允许传入 `reviewModel`，但旧的构造函数仍可能被无意中使用。**风险较低**，因为主入口 `OpenAiCodeReview.main` 已使用新的配置系统。
*   **Low**:
    *   **代码重复**：两个 GitHub Actions 工作流文件（`main-maven-jar.yml` 和 `main-remote-jar.yml`）在步骤上高度相似，存在重复。这增加了维护成本，例如需要同时更新两处以修改环境变量。**建议**：考虑使用 GitHub Actions 的 Reusable Workflows 或 Composite Actions 来抽取公共部分。
    *   **`.gitignore` 范围过窄**：新增的 `.gitignore` 文件只忽略了 `**/target/`。对于 Java/Maven 项目，通常还会忽略 `**/.idea/`, `**/*.iml`, `**/.classpath`, `**/.project` 等 IDE 和构建产生的文件。**建议**：补充常见的 Java 项目忽略条目。

## 3) Actionable Fixes
1.  **显式声明 `snakeyaml` 依赖**：
    *   文件：`openai-code-review-sdk/pom.xml`
    *   操作：在 `<dependencies>` 部分添加：
        ```xml
        <dependency>
            <groupId>org.yaml</groupId>
            <artifactId>snakeyaml</artifactId>
            <version>1.33</version> <!-- 使用与 jackson-dataformat-yaml 兼容的版本 -->
        </dependency>
        ```
    *   理由：避免因传递依赖变更导致 Shaded JAR 运行时缺少类。

2.  **增强配置加载的日志和错误提示**：
    *   文件：`openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/types/config/ReviewConfigLoader.java`
    *   操作：修改 `loadFromFile` 方法，当通过 `REVIEW_CONFIG_FILE` 环境变量指定了路径但文件不存在时，记录警告日志。
        ```java
        private static ReviewConfig loadFromFile() {
            String configPath = getenv("REVIEW_CONFIG_FILE", DEFAULT_CONFIG_FILE);
            File file = new File(configPath);
            if (!file.exists()) {
                LoggerFactory.getLogger(ReviewConfigLoader.class).warn("Review config file not found at: {}. Using defaults.", configPath);
                return new ReviewConfig();
            }
            // ... 原有解析代码
        }
        ```

3.  **为 `OpenAICompatible` 客户端设置超时**：
    *   文件：`openai-code-review-sdk/src/main/java/plus/gaga/middleware/sdk/infrastructure/openai/impl/OpenAICompatible.java`
    *   操作：在 `HttpURLConnection` 设置请求属性后，添加连接和读取超时。
        ```java
        connection.setConnectTimeout(30000); // 30秒连接超时
        connection.setReadTimeout(60000);    // 60秒读取超时
        ```

4.  **统一工作流中的分支名获取逻辑**：
    *   文件：`.github/workflows/main-maven-jar.yml`
    *   操作：将 `Get branch name` 步骤中的命令改为与 `main-remote-jar.yml` 一致：
        ```yaml
        - name: Get branch name
          run: echo "BRANCH_NAME=${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}" >> $GITHUB_ENV
        ```

5.  **完善 `.gitignore` 文件**：
    *   文件：`.gitignore`
    *   操作：添加常见的 Java/Maven/IDE 忽略模式，例如：
        ```
        # IDE
        .idea/
        *.iml
        .classpath
        .project
        .settings/
        *.swp
        *.swo

        # Build
        target/
        **/target/
        **/build/
        *.jar
        *.war
        *.ear

        # Logs
        *.log
        logs/

        # System
        .DS_Store
        Thumbs.db
        ```

## 4) Tests to Add
1.  **单元测试：`ReviewConfigLoader`**：
    *   **场景1**：测试从有效的 YAML 文件加载配置，并验证字段映射正确。
    *   **场景2**：测试环境变量覆盖优先级（例如，`OPENAI_APIHOST` 应覆盖 YAML 中的 `apiHost`）。
    *   **场景3**：测试当配置文件不存在时，是否返回默认配置对象。
    *   **场景4**：测试 `validate` 方法，当 `apiHost` 或 `apiKey` 为空时是否抛出预期异常。

2.  **单元测试：`GitCommand.safe()` 方法**：
    *   **场景1**：输入包含 `/`、`<>`、空格等字符的字符串，验证输出是否被替换为下划线且长度受限。
    *   **场景2**：输入 `null` 或空字符串，验证是否返回 `"unknown"`。
    *   **场景3**：输入纯合法字符的长字符串，验证是否被正确截断。

3.  **集成测试：多提供商端到端测试**：
    *   **目标**：确保 `OpenAICompatible` 和 `ChatGLM` 客户端都能成功调用其对应的模拟 API（使用 WireMock 或类似工具）并解析响应。
    *   **步骤**：启动一个模拟的 OpenAI-compatible API 服务器，返回固定的审查结果，然后运行 `OpenAiCodeReviewService` 的核心流程，验证能否收到模拟的评审建议并触发后续（模拟的）日志记录步骤。

4.  **测试：微信通知条件逻辑**：
    *   **场景1**：当所有 `WEIXIN_*` 环境变量都设置时，`isWeixinConfigured()` 应返回 `true`，并且 `pushMessage` 方法应被调用。
    *   **场景2**：当任一 `WEIXIN_*` 环境变量缺失或为空时，`isWeixinConfigured()` 应返回 `false`，并且 `pushMessage` 方法不应被调用，同时应有相应的 `WARN` 日志输出。可以通过 Mock `Logger` 并验证其调用来测试。