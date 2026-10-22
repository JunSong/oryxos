# Research: 对接 LLM（Provider 抽象）

**Phase**: 0 | **Date**: 2026-10-22 | **Plan**: [plan.md](./plan.md)

Technical Context 里没有遗留 `NEEDS CLARIFICATION`。下面记录的是需要拍板的技术选择，以及四份文档之间的口径差异如何处理。每条都按 Decision / Rationale / Alternatives 写。

---

## R1. 依赖与版本基线

- **Decision**: Java 21 + Spring Boot 3.x + Spring AI 1.0.x（经 Spring AI Alibaba BOM 管理版本）。DeepSeek、Kimi 都走 OpenAI 兼容 connector（`spring-ai-openai`），DashScope（通义）connector 本特性不引入。小版本在 T001 按当时 BOM 的兼容矩阵锁定。
- **Rationale**: 技术方案 §1.2 和宪法「技术栈」一节锁定 Boot 3.x + Spring AI Alibaba。需求 §8.4 把 OpenAI 兼容协议定为事实标准；DeepSeek 和 Kimi 都提供该协议，一个 connector 就能覆盖两家，符合 KISS。
- **Alternatives**:
  - 每家各用一个专用 connector：多一套依赖与配置，对两家都兼容 OpenAI 的场景没有收益。
  - LangChain4j：与宪法 II 指定的 Spring AI 相冲突。
- **备注**: 当前仓库主干已经演进到 Boot 4.1.1 / Spring AI 2.0.1 / `spring-ai-openai`，模块也扩到 14 个。本作业按任务要求只以四份文档为输入，当作 greenfield 拆解，不跟随仓库现状（对比见 `PROCESS.md` 第 6 节）。

## R2. 调用方式：`ChatModel.call` 而不是 `ChatClient`

- **Decision**: `SpringAiProviderService` 只调用 `chatModel.call(new Prompt(messages, options))`，options 设置 `internalToolExecutionEnabled(false)`。
- **Rationale**: 宪法 I/II 明确禁止 `ChatClient` 的自动工具执行，否则工具会被 Spring AI 和 `ReActLoop` 各执行一次。需求 §5.3、技术方案 §3 提到「基于 `ChatClient`」，属于早期措辞，按宪法优先级以 `ChatModel` 为准（spec Clarifications 第 3 条）。
- **Alternatives**: 用 `ChatClient` 但不注册工具：一旦后续特性注册工具就会踩坑，边界不清，不选。

## R3. 显式映射如何构建

- **Decision**: 不让 Spring 自动装配任何 `ChatModel` Bean。`ProviderRegistry` 启动时遍历 `oryxos.providers` 列表，用 `OpenAiCompatibleChatModelFactory` 为每条配置**手工**构建一个 `ChatModel`，放进不可变的 `Map<String, ChatModel>`。同时在 `application.yaml` 排除 OpenAI / DashScope 的自动装配。
- **Rationale**:
  - 宪法 III：映射关系完全来自配置里的 provider 名，不存在「类型相同分不清」的问题。
  - 宪法 II：排除自动装配，避免没有 key 时启动被阻断。
  - 技术方案 §3.2 把具体做法留到研发阶段决定（`@Qualifier` 还是自维护配置表），这里选自维护配置表，因为新增 provider 只改配置、不改代码。
- **Alternatives**:
  - 每家一个 `@Bean` + `@Qualifier("deepseek")`：新增 provider 要改 Java 代码，违背「配置即接入」。
  - `getBeansOfType(ChatModel.class)` 按 Bean 名推断：宪法 III 明令禁止。

## R4. 凭证配置形态

- **Decision**: 配置项写 `api-key-env: DEEPSEEK_API_KEY`（环境变量**名**），由 `ProviderConfigValidator` 在启动时读 `System.getenv` 解析。以下三种情况都拒绝启动，并在一条错误信息中给出 provider 名与变量名：
  - 值不匹配 `^[A-Z_][A-Z0-9_]*$`（比如有人把 `sk-...` 明文填进来）
  - 变量不存在
  - 变量值为空
- **Rationale**:
  - FR-005 / FR-006 / SC-004：配置里根本没有可以放明文的字段，零明文是结构性保证，不靠人自觉。
  - 报错能精确指出变量名，满足 SC-003「一条错误信息即可定位」。
  - 技术方案 §8.8 要求 `ConfigLoader` 做必填项与格式校验，这里由 provider 模块的校验器承担 provider 部分。
- **Alternatives**:
  - `api-key: ${DEEPSEEK_API_KEY}`（Spring 占位符）：变量缺失时 Spring 报的是通用的「Could not resolve placeholder」，不带 provider 名；而且同一字段也能直接填明文，无法拦截。
  - `${DEEPSEEK_API_KEY:}` 加非空校验：能拦缺失，但报错拿不到变量名，也拦不住明文。
- **影响**: Agent `AGENT.md` frontmatter 里的 `${ENV_VAR}` 占位语法不受影响，那是 Profile 加载的事，由后续特性处理。

## R5. 工具描述转换与「不执行」的双保险

- **Decision**: `ToolSpecAdapter` 把 `ToolSpec(name, description, inputSchemaJson)` 转成 Spring AI 的 `ToolCallback`，只提供 `ToolDefinition`；它的 `call(...)` 直接抛 `IllegalStateException("tool execution is owned by ReActLoop")`。同时 options 设 `internalToolExecutionEnabled(false)`。
- **Rationale**: 第一层（关闭内部执行）是官方开关，第二层（守卫 callback）防止开关因版本变化失效时工具被悄悄执行。测试断言守卫从未被触发，可以直接度量 SC-005。
- **Alternatives**: 只依赖开关：版本升级后行为一旦变化不会被发现，不选。

## R6. 消息映射

- **Decision**: core 定义 `sealed interface ChatMessage`，实现为 `System`、`User`、`Assistant(text, toolCalls)`、`ToolResult(toolCallId, toolName, content)`。`MessageMapper` 负责与 Spring AI 的 `SystemMessage`、`UserMessage`、`AssistantMessage`、`ToolResponseMessage` 双向转换。
- **Rationale**: 上层（ReActLoop、Session 序列化）只依赖 core 的值对象，不感知 Spring AI 类型。将来换掉底层框架，只改 provider 模块。
- **Alternatives**: 上层直接用 Spring AI 的 `Message`：core 会依赖 Spring AI，违反依赖倒置，也让 Session 的 JSON 序列化被第三方类型绑住。

## R7. 审计写入位置与失败处理

- **Decision**: `SpringAiProviderService.chat()` 用 `try/finally` 计时，在 `finally` 中构造 `LlmCallRecord` 交给 `LlmCallRecorder.record()`。recorder 抛出的异常被捕获并打 ERROR 日志，不向上抛。存储实现 `JpaLlmCallRecorder` 在 storage 模块，用 Flyway `V1__llm_calls.sql` 建表。
- **Rationale**:
  - 宪法 V：成功、失败都写。
  - FR-011：审计失败不改变业务结果，但必须可告警。
  - 宪法 VIII：不依赖 Hibernate 自动建表。
  - core 只定义契约，provider 不依赖 storage，两者都只依赖 core。
- **Alternatives**:
  - 异步写审计：违反宪法 VII，而且进程崩溃时会丢记录。
  - 审计失败直接让调用失败：一次存储抖动会让所有 Agent 停摆，代价过高。
- **字段**: 沿用需求 §10 / 技术方案 §9 的 `llm_calls` 九个字段，不额外加列（YAGNI）。失败原因不单独建列，靠 ERROR 日志承载；扩展阶段如需按失败统计，再用 Flyway 加列。

## R8. 超时与失败语义

- **Decision**: 每条 provider 配置可设 `timeout`（默认 60s），由工厂注入底层 HTTP 客户端的连接 / 读取超时。任何底层异常统一包装成 `ProviderCallException(providerName, cause)`；Spring AI 自带的重试设为 0 次。
- **Rationale**: FR-012 要求立即失败、不做 fallback；需求 §8.2「Provider 故障时核心阶段直接报错」。关掉框架重试，失败才「立即」可见，审计里的耗时也不会被重试放大。
- **Alternatives**: 保留默认重试：与 FR-012 的「立即返回」冲突，并会让一次逻辑调用对应多次外部请求，审计条数对不上（SC-002）。

## R9. 测试策略

- **Decision**:
  - 单元测试：校验器、映射器、适配器、注册表。
  - 集成测试：用 `MockWebServer` 起两个 OpenAI 兼容桩端点，验证路由、tool call 透传、失败语义、审计条数。
  - 冒烟测试：`@EnabledIfEnvironmentVariable(named = "DEEPSEEK_API_KEY")` / `KIMI_API_KEY` 门控，打真实服务（SC-001）。
  - 架构测试：ArchUnit 禁止依赖 `ChatClient`、禁止 `getBeansOfType(ChatModel)`、禁止 core 依赖 `org.springframework.ai`。
- **Rationale**: 路由正确性必须看到请求落在哪个端点，桩服务最直接。真实 key 测试不能进 CI 必跑集；ArchUnit 把宪法 I/II/III 变成会失败的测试。
- **Alternatives**: WireMock：功能更多但更重。这里只需要记录请求、返回固定 JSON，`MockWebServer` 足够。

## R10. 文档口径差异汇总

| 差异点 | 文档 A | 文档 B | 采用 | 依据 |
|--------|--------|--------|------|------|
| token 记录位置 | 需求 §5.3「落到日志」 | 技术方案 §3.3、宪法 V「写 `llm_calls`」 | 落库 | 宪法优先 |
| 验收 Provider 数 | 编程指南 §4.1「DeepSeek 或 Kimi」 | 需求 §13「DeepSeek 和 Kimi」 | 两家 | 验收标准更严且能证明路由 |
| 调用入口 | 需求 §5.3 / 技术方案 §3「`ChatClient`」 | 宪法 I/II「`chatModel.call`」 | `ChatModel` | 宪法优先 |
| 涉及模块 | 编程指南 §4.1：core / provider / boot | 宪法 V 推出需要 storage | 加 storage | 见 plan Complexity Tracking |
| 业界依据 | IndustryResearch §4.3：复用 Spring AI Alibaba connector，不重复造轮子 | — | 与 R1 一致 | — |
