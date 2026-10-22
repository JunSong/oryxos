# Implementation Plan: 对接 LLM（Provider 抽象）

**Branch**: `homework/speckit-us1-decomposition` | **Date**: 2026-10-22 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `homework/01-speckit-first-task/specs/001-llm-provider/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

做一层薄的 `ProviderService`。上层（后续 `ReActLoop`）传 provider 名 + 消息 + 工具描述，拿回文本、工具调用请求和 token 用量。各家协议差异交给 Spring AI（Alibaba 生态）的 `ChatModel` 吸收。OryxOS 自己负责四件事：

1. 按配置**显式**建 `provider name → ChatModel` 映射表，不扫描容器 Bean（宪法 III）。
2. 只调 `chatModel.call(new Prompt(...))`，关掉内部工具执行，把 tool call 原样交回（宪法 I/II）。
3. 启动期校验配置：凭证只收环境变量名，缺失即拒启（宪法 VI、技术方案 §8.8）。
4. 每次调用在 `finally` 里写一条 `llm_calls` 审计，经 core 契约由 storage 模块落 SQLite（宪法 V）。

## Technical Context

**Language/Version**: Java 21（虚拟线程）

**Primary Dependencies**: Spring Boot 3.x；Spring AI 1.0.x + Spring AI Alibaba BOM（只用 `ChatModel` 与 OpenAI 兼容 connector）；Spring Data JPA；Flyway；SnakeYAML（随 Boot）。具体小版本在 T001 用当时的 BOM 核实（见 research R1）。

**Storage**: SQLite（`.oryxos/oryxos.db`），只建 `llm_calls` 一张表，Flyway 迁移，`ddl-auto=none`

**Testing**: JUnit 5 + AssertJ + Mockito；OkHttp `MockWebServer` 做 OpenAI 兼容桩服务；ArchUnit 守宪法边界；真实 key 冒烟测试用 `@EnabledIfEnvironmentVariable` 门控

**Target Platform**: Linux / macOS 服务器，单 fat JAR

**Project Type**: Maven 多模块后端库 + Spring Boot 装配（无对外 HTTP 端点，本特性只提供进程内服务）

**Performance Goals**: 平台附加开销 P99 ≤ 50ms（SC-006，桩模型下测量）

**Constraints**: 全同步阻塞（宪法 VII）；不做 fallback / hedge / 流式；凭证零明文（SC-004）；默认调用超时 60s

**Scale/Scope**: 核心阶段 2 家 Provider（DeepSeek、Kimi），设计上不限数量；单节点 ≥100 并发调用不互相阻塞

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | 原则 | 本特性的落点 | Phase 0 前 | Phase 1 后 |
|---|------|--------------|-----------|-----------|
| I | 自实现 ReAct，禁 `ChatClient` 自动执行 | `ProviderService` 只做单次调用，不循环、不执行工具；ArchUnit 规则禁止引用 `ChatClient` | ✅ | ✅ |
| II | Spring AI 只做协议转换 + schema | 只用 `ChatModel.call`；`internalToolExecutionEnabled(false)`；交给模型的 `ToolCallback` 是「执行即抛异常」的守卫实现；排除 OpenAI / DashScope 等自动装配 | ✅ | ✅ |
| III | Provider 显式映射 | `ProviderRegistry` 由配置列表逐条构建 `Map<String, ChatModel>`；ArchUnit 禁止 `getBeansOfType(ChatModel)` | ✅ | ✅ |
| IV | 一个目录 = 一个 Agent | 不涉及（本特性不读 `AGENT.md`，`Profile` 只留 provider 引用字段） | N/A | N/A |
| V | 审计 day one 落库 | `LlmCallRecorder` 契约在 core，`JpaLlmCallRecorder` 在 storage，成功失败都写 | ✅ | ✅ |
| VI | 安全地基、凭证走环境变量 | 配置只接受 `api-key-env`（变量名），校验器拒绝不像变量名的值；`ProviderConfig.toString()` 脱敏 | ✅ | ✅ |
| VII | 同步 + 虚拟线程 | 全同步；`spring.threads.virtual.enabled=true`；不引入 Reactor 返回类型 | ✅ | ✅ |
| VIII | 实例无状态、状态外置；不靠 Hibernate 迁移 | 注册表启动期构建、只读；审计落 SQLite；Flyway `V1__llm_calls.sql` | ✅ | ✅ |
| 技术栈 | 9 模块骨架、跨模块契约放 core | 建 9 模块骨架；契约（`ProviderService`、值对象、`LlmCallRecorder`）都在 core | ✅ | ✅ |

**结论**：无未论证的违例，允许进入 Phase 0。唯一需要说明的取舍见 Complexity Tracking。

## Project Structure

### Documentation (this feature)

```text
homework/01-speckit-first-task/specs/001-llm-provider/
├── spec.md              # /speckit-specify 输出
├── checklists/
│   └── requirements.md  # spec 质量检查
├── plan.md              # 本文件
├── research.md          # Phase 0
├── data-model.md        # Phase 1
├── contracts/
│   ├── provider-service.md   # 进程内 Java 契约
│   └── provider-config.md    # application.yaml 配置契约
├── quickstart.md        # Phase 1 验证指南
└── tasks.md             # /speckit-tasks 输出
```

### Source Code (repository root)

```text
pom.xml                                   # 父 POM：Java 21、Boot 3.x、Spring AI / Alibaba BOM、9 个 <module>
oryxos-core/src/main/java/io/oryxos/core/
├── provider/
│   ├── ProviderService.java              # 统一调用入口（契约）
│   ├── ChatRequest.java / ChatResult.java
│   ├── ChatMessage.java                  # sealed：System / User / Assistant / ToolResult
│   ├── ToolSpec.java / ToolCallRequest.java / TokenUsage.java
│   └── ProviderNotFoundException.java / ProviderCallException.java / ProviderConfigException.java
├── tool/
│   └── OryxTool.java / ToolResult.java   # 工具统一抽象（只立契约，US-4 特性实现）
├── profile/
│   └── Profile.java / ProviderRef.java   # 最小数据结构，只含 provider 引用
└── audit/
    └── LlmCallRecord.java / LlmCallRecorder.java
oryxos-provider/src/main/java/io/oryxos/provider/
├── ProviderProperties.java               # @ConfigurationProperties("oryxos.providers")
├── ProviderConfigValidator.java          # 启动期校验：重名、变量名格式、变量缺失/为空
├── ProviderRegistry.java                 # 显式 Map<String, ChatModel>
├── OpenAiCompatibleChatModelFactory.java # 按单条配置构建 ChatModel（base URL、key、超时）
├── MessageMapper.java                    # ChatMessage ↔ Spring AI Message
├── ToolSpecAdapter.java                  # ToolSpec → ToolCallback（执行即抛异常的守卫）
├── SpringAiProviderService.java          # ProviderService 实现 + 审计计时
└── ProviderAutoConfiguration.java
oryxos-storage/src/main/
├── java/io/oryxos/storage/audit/
│   ├── LlmCallEntity.java / LlmCallJpaRepository.java
│   └── JpaLlmCallRecorder.java
└── resources/db/migration/sqlite/V1__llm_calls.sql
oryxos-boot/src/main/
├── java/io/oryxos/boot/OryxosApplication.java
└── resources/application.yaml           # 虚拟线程、数据源、排除自动装配、providers 示例
oryxos-memory/ oryxos-tool/ oryxos-channel-cli/ oryxos-web/ oryxos-cli/   # 仅空骨架 pom

各模块 src/test/java/... 对应测试；oryxos-provider 下放 MockWebServer 集成测试与 ArchUnit 规则
```

**Structure Decision**: 按技术方案 §10 与编程指南 §4.1 一次建齐 9 模块骨架。本特性写代码的模块是 core / provider / storage / boot，其余 5 个只有空 pom，保证依赖方向从第一天就固定：provider、storage 依赖 core，boot 聚合。

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| 编程指南 §4.1 只列 core / provider / boot，本计划额外动了 `oryxos-storage` | 宪法 V 要求 `llm_calls` day one 落库，而审计写入只能落在持久化模块 | 「先写日志，第四周再落库」直接违反宪法 V 的 MUST NOT；把 JPA 放进 provider 会让能力模块耦合存储，违反依赖倒置 |
