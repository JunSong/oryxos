---

description: "Task list for 001-llm-provider（US-1 对接 LLM）"
---

# Tasks: 对接 LLM（Provider 抽象）

**Input**: Design documents from `homework/01-speckit-first-task/specs/001-llm-provider/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 本特性包含测试任务。理由：spec 每个故事都定义了 Independent Test，SC-002/004/005/006 只能靠测试度量，`quickstart.md` 也以测试作为验证入口；宪法「质量门禁」要求 `mvn verify` 全绿。同一故事内先写测试、确认失败，再写实现。

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Maven 多模块，每个模块 `src/main/java/io/oryxos/<module>/...`、`src/test/java/io/oryxos/<module>/...`
- 下文简写：`core/` = `oryxos-core/src/main/java/io/oryxos/core/`，`provider/` = `oryxos-provider/src/main/java/io/oryxos/provider/`，`storage/` = `oryxos-storage/src/main/java/io/oryxos/storage/`，`boot/` = `oryxos-boot/src/main/java/io/oryxos/boot/`；测试目录把 `src/main` 换成 `src/test`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 9 模块骨架与构建门禁

- [ ] T001 创建父 POM `pom.xml`：`<java.version>21</java.version>`，parent 为 Spring Boot 3.x，`dependencyManagement` 引入 Spring AI BOM 与 Spring AI Alibaba BOM，并按两者兼容矩阵锁定小版本（research R1）。锁定后解包对应 jar，核实 `internalToolExecutionEnabled`、`ToolCallback` / `ToolDefinition`、`ToolResponseMessage`、自动装配类名在该版本的确切签名，结论回写 research.md R2 / R5；核实不到就停下报告，不要猜。声明 9 个 `<module>`：oryxos-core、oryxos-provider、oryxos-memory、oryxos-tool、oryxos-channel-cli、oryxos-web、oryxos-storage、oryxos-cli、oryxos-boot
- [ ] T002 创建 9 个模块的 `*/pom.xml`，依赖方向：core 不依赖 Spring AI；provider → core + `spring-ai-openai`；storage → core + Spring Data JPA + `sqlite-jdbc` + `hibernate-community-dialects` + Flyway；boot → 全部；memory / tool / channel-cli / web / cli 只建空骨架
- [ ] T003 [P] 在 `pom.xml` 的 `<build>` 中接入质量门禁：Spotless（Google 格式）、P3C-PMD、Checkstyle、SpotBugs + Find Security Bugs、OWASP Dependency-Check，全部绑定到 `verify`（宪法「质量门禁」）
- [ ] T004 [P] 创建启动类 `boot/OryxosApplication.java` 与 `oryxos-boot/src/main/resources/application.yaml`：`spring.threads.virtual.enabled: true`；`spring.autoconfigure.exclude` 排除 OpenAI / DashScope 的 ChatModel 自动装配（类名以 T001 锁定版本为准，见 contracts/provider-config.md）

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: core 契约、值对象与测试基础设施。所有故事都依赖它们。

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 [P] 在 `core/provider/ChatMessage.java` 定义 `sealed interface ChatMessage`，实现 `System(content)`、`User(content)`、`Assistant(content 可空, List<ToolCallRequest> toolCalls)`、`ToolResult(toolCallId, toolName, content)`，均为 record（data-model §1）
- [ ] T006 [P] 在 `core/provider/` 创建 record：`ToolSpec(name, description, inputSchemaJson)`，name 校验「匹配 `^[a-zA-Z0-9_-]{1,64}$`」；`ToolCallRequest(id, name, argumentsJson)`，「参数 JSON 原文，不做二次解析或改写」；`TokenUsage(Integer promptTokens, Integer completionTokens, Integer totalTokens)`，「服务端未返回时为 null，不编造」
- [ ] T007 在 `core/provider/ChatRequest.java`、`core/provider/ChatResult.java` 创建 record。`ChatRequest` 紧凑构造器校验「messages 非空列表；空列表直接 IllegalArgumentException」，tools 缺省空列表且「工具名在列表内唯一」，model / sessionId 可空。`ChatResult(providerName, model, text 可空, toolCalls, usage)`（依赖 T005、T006）
- [ ] T008 [P] 在 `core/provider/` 创建 `ProviderNotFoundException`（消息列出已登记名）、`ProviderCallException`（携带 providerName 与 cause，消息不含凭证）、`ProviderConfigException`（聚合多条校验错误）
- [ ] T009 在 `core/provider/ProviderService.java` 定义接口：`ChatResult chat(ChatRequest)`、`List<String> providerNames()`，Javadoc 写明「不执行任何工具，不自动发起后续轮次」（contracts/provider-service.md）
- [ ] T010 [P] 在 `core/audit/LlmCallRecord.java`、`core/audit/LlmCallRecorder.java` 定义审计值对象与契约。字段：sessionId 可空、provider、model、三个 token 可空、durationMs、createdAt（UTC Instant），与 `llm_calls` 一一对应
- [ ] T011 [P] 在 `core/tool/OryxTool.java`、`core/tool/ToolResult.java` 定义工具统一抽象：`getName()`、`getDescription()`、`getInputSchema()`、`execute(JsonNode)`，外加 `default ToolSpec toSpec()`；`ToolResult(success, content, errorMessage, retryable)`。本特性只立契约，不写实现
- [ ] T012 [P] 在 `core/profile/Profile.java`、`core/profile/ProviderRef.java` 创建最小 record：`ProviderRef(name, model 可空, temperature 可空)`、`Profile(name, ProviderRef provider)`
- [ ] T013 [P] 在 `provider/ProviderProperties.java` 创建 `@ConfigurationProperties("oryxos.providers")` 列表绑定；在 `provider/ProviderConfig.java` 定义单条配置 record：name、model、apiKeyEnv、baseUrl 可空、timeout 默认 60s。`toString()` 不输出任何凭证相关值
- [ ] T014 [P] 在 `oryxos-provider/src/test/java/io/oryxos/provider/support/OpenAiStub.java` 封装 OkHttp `MockWebServer`：可预置 chat completion 响应（纯文本 / 含 tool_calls / 无 usage / 5xx / 延迟），记录收到的请求供断言
- [ ] T015 在 `oryxos-boot/src/test/java/io/oryxos/boot/ArchitectureRulesTest.java` 用 ArchUnit 落实 contracts/provider-service.md 的四条架构约束：core 不依赖 `org.springframework.ai..`；全局不依赖 `ChatClient`；不批量获取 `ChatModel` Bean；provider 不依赖 storage

**Checkpoint**: 契约就绪，各故事可以开始

---

## Phase 3: User Story 1 - 通过统一入口完成一次模型对话 (Priority: P1) 🎯 MVP

**Goal**: 登记一家服务（deepseek），调用方传 provider 名 + 消息，拿到回复与 token 用量；配置不合法时拒启

**Independent Test**: 只配 deepseek（桩服务），发「你好」拿到非空回复与 usage；去掉 `DEEPSEEK_API_KEY` 后启动失败，报错含 provider 名与变量名

### Tests for User Story 1 ⚠️

- [ ] T016 [P] [US1] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ProviderConfigValidatorTest.java` 覆盖单条配置规则：providers 为空、model 为空、`api-key-env` 不匹配 `^[A-Z_][A-Z0-9_]*$`（输入 `sk-abc` 断言提示「疑似明文 key」）、变量不存在或为空、base-url 非 http(s) 绝对地址、timeout ≤ 0；断言多条错误一次性报出（contracts/provider-config.md）
- [ ] T017 [P] [US1] 在 `oryxos-provider/src/test/java/io/oryxos/provider/SingleProviderChatIT.java` 用 `OpenAiStub` + 内存版 `LlmCallRecorder` 验证契约 C1（回复与 usage）、C3（空消息不发请求、不写审计）、C6（5xx / 超时抛 `ProviderCallException`，桩服务只收到 1 次请求，写 1 条 token 为空的审计）
- [ ] T018 [P] [US1] 在 `oryxos-boot/src/test/java/io/oryxos/boot/StartupFailFastIT.java` 验证 SC-003：缺 `KIMI_API_KEY` 时 Spring 上下文启动失败，耗时 < 10s，异常信息含 `provider "kimi"` 与 `KIMI_API_KEY`，且不含测试用 key 值

### Implementation for User Story 1

- [ ] T019 [US1] 在 `provider/ProviderConfigValidator.java` 实现 T016 中的规则，汇总为一个 `ProviderConfigException`，错误文案与 contracts/provider-config.md 一致；只有它调用 `System.getenv`，且把 key 值只交给工厂、不存储（research R4）
- [ ] T020 [P] [US1] 在 `provider/OpenAiCompatibleChatModelFactory.java` 按单条 `ProviderConfig` + 已解析 key 构建 `ChatModel`：base URL 缺省用该服务公开地址；连接 / 读取超时取 `timeout`；Spring AI 重试次数 0（research R8）；默认 options 设 `internalToolExecutionEnabled(false)`
- [ ] T021 [P] [US1] 在 `provider/MessageMapper.java` 实现 `ChatMessage.System/User/Assistant(纯文本)` → Spring AI `SystemMessage/UserMessage/AssistantMessage` 的映射，以及 `ChatResponse` → `ChatResult` 的文本与 usage 映射（usage 缺失记 null）
- [ ] T022 [US1] 在 `provider/ProviderRegistry.java` 遍历校验后的配置，逐条调工厂，构建不可变 `Map<String, ChatModel>`（保持配置顺序）；`get(name)` 未命中时抛 `ProviderNotFoundException` 并列出已登记名（宪法 III，research R3）
- [ ] T023 [US1] 在 `provider/SpringAiProviderService.java` 实现 `ProviderService`：查注册表 → model 缺省取配置默认值 → `chatModel.call(new Prompt(...))` → 映射结果；底层异常包装为 `ProviderCallException`；`try/finally` 计时并调 `LlmCallRecorder.record`，recorder 异常捕获后打 ERROR 日志、不外抛（research R7）
- [ ] T024 [US1] 在 `provider/ProviderAutoConfiguration.java` 装配 validator → registry → service（启动期执行校验，失败即抛出使上下文启动失败），并登记到 `oryxos-provider/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- [ ] T025 [US1] 在 `oryxos-boot/src/main/resources/application.yaml` 增加 deepseek 条目：`name: deepseek`、`model: deepseek-chat`、`api-key-env: DEEPSEEK_API_KEY`

**Checkpoint**: US1 可独立验证。注意：生产装配需要一个 `LlmCallRecorder` 实现，由 US3 提供。宪法 V 要求本特性合并前 US3 必须完成，US1 阶段只在测试中使用内存版 recorder。

---

## Phase 4: User Story 2 - 多家模型服务并存时按名称精确路由 (Priority: P2)

**Goal**: deepseek 与 kimi 并存，按名路由不串台；未登记名明确报错；重名拒启

**Independent Test**: 两个桩端点分别代表两家，按名各调一次，断言请求只落到对应端点；用 `qwen` 调用得到错误且两个桩都没收到请求

### Tests for User Story 2 ⚠️

- [ ] T026 [P] [US2] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ProviderConfigValidatorTest.java` 追加：name 重复（报出「第 1、3 条」）、name 不匹配 `^[a-z][a-z0-9-]*$`（输入 `Deep Seek`）
- [ ] T027 [P] [US2] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ProviderRoutingIT.java` 用两个 `OpenAiStub` 验证：按 `deepseek` / `kimi` 调用只命中各自端点且结果不串台；契约 C2（`qwen` 抛 `ProviderNotFoundException`，消息含已登记名，两个桩请求数都为 0，不写审计）；`providerNames()` 按配置顺序返回

### Implementation for User Story 2

- [ ] T028 [US2] 在 `provider/ProviderConfigValidator.java` 加入 name 唯一性与格式规则，错误文案与 contracts/provider-config.md 一致
- [ ] T029 [US2] 在 `oryxos-boot/src/main/resources/application.yaml` 增加 kimi 条目：`name: kimi`、`model: moonshot-v1-8k`、`api-key-env: KIMI_API_KEY`、`base-url: https://api.moonshot.cn`
- [ ] T030 [US2] 在 `oryxos-provider/src/test/java/io/oryxos/provider/RealProviderSmokeIT.java` 写真实冒烟测试，分别用 `@EnabledIfEnvironmentVariable(named = "DEEPSEEK_API_KEY")` 与 `KIMI_API_KEY` 门控：两家各一次真实对话，只换 providerName 参数（SC-001）

**Checkpoint**: US1 + US2 均可独立验证

---

## Phase 5: User Story 3 - 每次模型调用留下审计记录 (Priority: P2)

**Goal**: 成功、失败调用都写 `llm_calls`，重启不丢；审计故障不影响业务

**Independent Test**: 两次成功 + 一次失败调用后查库恰好新增 3 行且字段完整；重建上下文后记录仍在

### Tests for User Story 3 ⚠️

- [ ] T031 [P] [US3] 在 `oryxos-storage/src/test/java/io/oryxos/storage/audit/JpaLlmCallRecorderIT.java` 用临时目录 SQLite 验证：写入后字段与 `LlmCallRecord` 一致；token 为 null 时落 NULL；关闭并重建上下文后记录仍在；Flyway history 显示 V1 已执行
- [ ] T032 [P] [US3] 在 `oryxos-boot/src/test/java/io/oryxos/boot/LlmCallAuditIT.java` 用桩服务 + 真实 SQLite 验证：契约 C7（sessionId=s1 的成功调用 1 行字段齐全）；C6（失败调用 1 行，token 为空，duration_ms 有值）；C8（把 recorder 换成抛异常的实现，调用照常返回且有 1 条 ERROR 日志）；SC-002（100 次成功失败混合调用，行数 = 实际外部请求数）

### Implementation for User Story 3

- [ ] T033 [P] [US3] 创建 `oryxos-storage/src/main/resources/db/migration/sqlite/V1__llm_calls.sql`：`id INTEGER PK AUTOINCREMENT`、`session_id VARCHAR(128) NULL`、`provider VARCHAR(64) NOT NULL`、`model VARCHAR(128) NOT NULL`、`prompt_tokens/completion_tokens/total_tokens INTEGER NULL`、`duration_ms BIGINT NOT NULL`、`created_at TIMESTAMP NOT NULL`；索引 `idx_llm_calls_session(session_id)`、`idx_llm_calls_created(created_at)`（data-model §2，只增不改）
- [ ] T034 [P] [US3] 在 `storage/audit/LlmCallEntity.java`、`storage/audit/LlmCallJpaRepository.java` 创建与 V1 列一一映射的 JPA 实体和 `JpaRepository`
- [ ] T035 [US3] 在 `storage/audit/JpaLlmCallRecorder.java` 实现 `LlmCallRecorder`：`LlmCallRecord` → `LlmCallEntity` → `save`，同步写入（宪法 VII）
- [ ] T036 [US3] 在 `storage/StorageAutoConfiguration.java` 与 `oryxos-boot/src/main/resources/application.yaml` 完成存储装配：数据源 `jdbc:sqlite:${oryxos.home:.oryxos}/oryxos.db`；启动前确保目录存在；`spring.jpa.hibernate.ddl-auto: none`；SQLite 方言；`spring.flyway.locations: classpath:db/migration/sqlite`（宪法 VIII）

**Checkpoint**: 生产装配完整，宪法 V 满足

---

## Phase 6: User Story 4 - 把工具描述交给模型，并把工具调用请求原样交回 (Priority: P3)

**Goal**: 工具清单透传给模型；模型返回的 tool call 原样交回；平台执行工具次数为 0

**Independent Test**: 交给模型 `get_weather` 工具，桩服务返回 tool_calls，断言 `ChatResult.toolCalls` 完整且守卫 callback 调用次数为 0

### Tests for User Story 4 ⚠️

- [ ] T037 [P] [US4] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ToolSpecAdapterTest.java` 验证：生成的 `ToolDefinition` 名称、说明、schema 与 `ToolSpec` 一致；调用 callback 的 `call(...)` 抛 `IllegalStateException("tool execution is owned by ReActLoop")`
- [ ] T038 [P] [US4] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ToolCallPassthroughIT.java` 验证：契约 C4（tool_calls 的 id / name / argumentsJson 原文交回，守卫调用 0 次，桩只收到 1 次请求）；C5（回填 `ToolResult` 后请求体里有对应 `tool_call_id` 的 tool 角色消息）；单轮多个 tool call 保序；文本与 tool call 同时返回；不给工具时请求体不含 `tools`

### Implementation for User Story 4

- [ ] T039 [P] [US4] 在 `provider/ToolSpecAdapter.java` 实现 `ToolSpec` → `ToolCallback`：只提供 `ToolDefinition`，`call(...)` 抛异常作为守卫（research R5）
- [ ] T040 [US4] 在 `provider/MessageMapper.java` 扩展：`Assistant.toolCalls` ↔ `AssistantMessage.ToolCall`；`ToolResult` → `ToolResponseMessage`（保留 toolCallId）；`ChatResponse` 中的 tool calls → `List<ToolCallRequest>`（保序、参数原文）
- [ ] T041 [US4] 在 `provider/SpringAiProviderService.java` 把 `request.tools()` 经 `ToolSpecAdapter` 放入本次调用的 options，并显式设置 `internalToolExecutionEnabled(false)`；把映射出的 tool calls 写入 `ChatResult`

**Checkpoint**: 全部故事完成，可交给 US-2（ReAct 循环）使用

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: 跨故事的非功能验证与文档同步

- [ ] T042 [P] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ConcurrencyIT.java` 用 100 个虚拟线程并发调用桩服务，验证契约 C9：结果与请求一一对应、审计 100 行、无阻塞（FR-013）
- [ ] T043 [P] 在 `oryxos-provider/src/test/java/io/oryxos/provider/ProviderOverheadIT.java` 用 0ms 桩连续调用 1000 次，断言平台侧耗时 P99 ≤ 50ms（SC-006）
- [ ] T044 [P] 在 `oryxos-boot/src/test/java/io/oryxos/boot/SecretLeakIT.java` 用 `OutputCaptureExtension` 收集启动失败、调用失败、正常调用三种路径的日志与异常消息，并 dump `llm_calls`，断言测试 key 命中 0 次（SC-004）
- [ ] T045 [P] 同步文档：`docs/TechnicalSolution.md` §3.2 / §8.8 写明「显式映射由配置列表构建」和 `api-key-env` 只收变量名；`CLAUDE.md` 的 provider 配置示例同步
- [ ] T046 在仓库根（`pom.xml`）运行 `mvn clean verify`，Spotless / P3C / Checkstyle / SpotBugs / OWASP 与全部测试全绿
- [ ] T047 按 `quickstart.md` 场景 1~6 逐条执行并记录结果

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**：无依赖。T002 依赖 T001；T003、T004 依赖 T002，二者可并行
- **Foundational (Phase 2)**：依赖 Setup，阻塞所有故事。T007 依赖 T005、T006；T009 依赖 T007、T008
- **User Stories (Phase 3~6)**：依赖 Foundational
- **Polish (Phase 7)**：依赖全部故事

### User Story Dependencies

- **US1 (P1)**：只依赖 Foundational，是 MVP
- **US2 (P2)**：依赖 US1 的 validator / registry / service（在其上加规则与多端点测试），可独立测试
- **US3 (P2)**：依赖 US1 的 service 已调用 `LlmCallRecorder`；T031、T033、T034 只依赖 Foundational，可与 US1 并行开工
- **US4 (P3)**：依赖 US1 的 `MessageMapper` 与 service；与 US2、US3 互不依赖
- **合并门槛**：US1 + US3 是宪法 V 的最小可合并集；完整交付 US-1 需要四个故事全部完成

```text
Setup ──▶ Foundational ──▶ US1 ──┬──▶ US2 ──┐
                       │         ├──▶ US4 ──┼──▶ Polish
                       └─(T031/T033/T034)──▶ US3 ──┘
```

### Within Each User Story

- 测试先写并确认失败（红），再实现（绿）
- 值对象 → 校验 / 映射 → 注册表 → 服务 → 装配
- 一个故事过 Checkpoint 再进下一个

### Parallel Opportunities

- Phase 2：T005、T006、T008、T010、T011、T012、T013、T014 互不依赖，可同时进行
- US1：T016、T017、T018 三个测试并行；T020、T021 并行
- US1 完成后，US2、US3（实现部分）、US4 可由不同人并行
- Polish：T042~T045 并行

---

## Parallel Example: User Story 1

```bash
# 先并行写好 US1 的三个测试（应全部失败）
Task: "ProviderConfigValidatorTest in oryxos-provider/src/test/java/io/oryxos/provider/ProviderConfigValidatorTest.java"
Task: "SingleProviderChatIT in oryxos-provider/src/test/java/io/oryxos/provider/SingleProviderChatIT.java"
Task: "StartupFailFastIT in oryxos-boot/src/test/java/io/oryxos/boot/StartupFailFastIT.java"

# 再并行写两个互不依赖的组件
Task: "OpenAiCompatibleChatModelFactory in oryxos-provider/src/main/java/io/oryxos/provider/OpenAiCompatibleChatModelFactory.java"
Task: "MessageMapper in oryxos-provider/src/main/java/io/oryxos/provider/MessageMapper.java"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 Setup → Phase 2 Foundational
2. Phase 3 US1 → **STOP and VALIDATE**：桩服务下单家对话、启动校验通过
3. 紧接着完成 US3，否则不满足宪法 V，不能合并

### Incremental Delivery

1. Setup + Foundational → 契约就绪
2. US1 + US3 → 单家可用且有审计（第一个可合并增量）
3. US2 → 双家并存、真实冒烟（满足需求 §13「DeepSeek 和 Kimi」）
4. US4 → 工具描述透传，交给 US-2（ReAct 循环）
5. Polish → 非功能指标与文档同步

### Parallel Team Strategy

1. 全员完成 Setup + Foundational
2. A 做 US1；B 同时做 US3 的 T031、T033、T034
3. US1 完成后：A 做 US2，B 收尾 US3，C 做 US4

---

## Notes

- 共 47 个任务：Setup 4 / Foundational 11 / US1 10 / US2 5 / US3 6 / US4 5 / Polish 6
- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- 宪法守卫：T015（ArchUnit）把原则 I/II/III 变成会失败的测试；T023 / T041 是最容易写错的地方（自动工具执行、Bean 扫描）
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
