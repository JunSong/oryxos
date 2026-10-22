# Data Model: 对接 LLM（Provider 抽象）

**Phase**: 1 | **Date**: 2026-10-22 | **Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

本特性的实体分两类：

- 进程内值对象：放在 `oryxos-core`，全部是不可变 `record`。
- 持久化实体：只有一张 `llm_calls`。

## 1. 进程内值对象（`oryxos-core`）

### ProviderConfig（配置，启动期构建，只读）

| 字段 | 类型 | 必填 | 校验规则 | 来源 |
|------|------|------|----------|------|
| `name` | String | 是 | 非空；匹配 `^[a-z][a-z0-9-]*$`；全局唯一（重名拒启） | FR-001、US2-场景3 |
| `model` | String | 是 | 非空，作为默认模型 | FR-001 |
| `apiKeyEnv` | String | 是 | 匹配 `^[A-Z_][A-Z0-9_]*$`；对应环境变量存在且非空 | FR-005、R4 |
| `baseUrl` | URI | 否 | 若填必须是 http(s) 绝对地址；缺省用该服务公开默认地址 | FR-001、Edge |
| `timeout` | Duration | 否 | > 0，默认 60s | R8 |

解析后的 key 值只存在于 `ChatModel` 内部，**不**放进 `ProviderConfig`。`toString()` 不输出任何凭证相关值（FR-006）。

### ChatRequest（一次调用的输入）

| 字段 | 类型 | 必填 | 校验规则 |
|------|------|------|----------|
| `providerName` | String | 是 | 必须已登记，否则 `ProviderNotFoundException`（FR-004） |
| `model` | String | 否 | 缺省取 `ProviderConfig.model` |
| `messages` | List\<ChatMessage\> | 是 | 非空列表；空列表直接 `IllegalArgumentException`，不发外部调用、不写审计（Edge） |
| `tools` | List\<ToolSpec\> | 否 | 缺省空列表；工具名在列表内唯一 |
| `sessionId` | String | 否 | 透传给审计，可空 |

### ChatMessage（sealed）

| 子类型 | 字段 | 说明 |
|--------|------|------|
| `System` | `content` | 系统提示词 |
| `User` | `content` | 用户输入 |
| `Assistant` | `content`（可空）、`toolCalls`（List\<ToolCallRequest\>） | 模型回复；回填历史时使用 |
| `ToolResult` | `toolCallId`、`toolName`、`content` | 工具执行结果，`toolCallId` 必须与某条 `ToolCallRequest.id` 对应（FR-009） |

### ToolSpec（工具描述，只描述不执行）

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | String | 工具名，匹配 `^[a-zA-Z0-9_-]{1,64}$`（OpenAI 兼容协议约束） |
| `description` | String | 给模型看的用途说明 |
| `inputSchemaJson` | String | 参数 JSON Schema（object 类型） |

`OryxTool.toSpec()` 产出 `ToolSpec`，所以 ReActLoop 不用关心工具来源（技术方案决策四）。

### ToolCallRequest（模型产出）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 模型给出的调用编号，原样保留 |
| `name` | String | 工具名 |
| `argumentsJson` | String | 参数 JSON 原文，不做二次解析或改写（SC-005） |

### TokenUsage

| 字段 | 类型 | 说明 |
|------|------|------|
| `promptTokens` / `completionTokens` / `totalTokens` | Integer（可空） | 服务端未返回时为 `null`，不编造（US3-场景2） |

### ChatResult（一次调用的输出）

| 字段 | 类型 | 说明 |
|------|------|------|
| `providerName` / `model` | String | 实际使用的 provider 与模型 |
| `text` | String（可空） | 回复文本；纯工具调用轮可为空 |
| `toolCalls` | List\<ToolCallRequest\> | 可能为空，可能多条，顺序与模型输出一致 |
| `usage` | TokenUsage | 见上 |

### LlmCallRecord（审计值对象，交给 `LlmCallRecorder`）

字段与下方 `llm_calls` 表一一对应，`id` 由存储生成。

### Profile / ProviderRef（最小结构）

| 实体 | 字段 | 说明 |
|------|------|------|
| `ProviderRef` | `name`、`model`（可空）、`temperature`（可空） | Agent 对 provider 的引用 |
| `Profile` | `name`、`provider: ProviderRef` | 本特性只立结构；从 `AGENT.md` 派生由后续特性完成 |

## 2. 持久化实体：`llm_calls`（`oryxos-storage`）

字段沿用需求 §10 与技术方案 §9 的定义，不增不减：

| 列 | 类型 | 可空 | 说明 |
|----|------|------|------|
| `id` | INTEGER PK AUTOINCREMENT | 否 | 主键 |
| `session_id` | VARCHAR(128) | 是 | 无状态调用为空 |
| `provider` | VARCHAR(64) | 否 | provider 名 |
| `model` | VARCHAR(128) | 否 | 实际模型 |
| `prompt_tokens` | INTEGER | 是 | |
| `completion_tokens` | INTEGER | 是 | |
| `total_tokens` | INTEGER | 是 | |
| `duration_ms` | BIGINT | 否 | 从发出请求到拿到结果或异常 |
| `created_at` | TIMESTAMP | 否 | 调用开始时间（UTC） |

索引：`idx_llm_calls_session (session_id)`、`idx_llm_calls_created (created_at)`。为将来按会话 / 时间查询预留，本特性不提供查询接口。

迁移脚本：`oryxos-storage/src/main/resources/db/migration/sqlite/V1__llm_calls.sql`，只增不改（宪法 VIII）。

## 3. 关系与生命周期

```text
ProviderConfig (N, 启动期) ──构建──▶ ProviderRegistry: Map<name, ChatModel>（只读，进程生命周期）
ChatRequest ──▶ ProviderService.chat ──▶ ChatResult
                       │
                       └─finally─▶ LlmCallRecord ──▶ LlmCallRecorder ──▶ llm_calls（1 次调用 = 1 行）
ChatResult.toolCalls[i].id  ◀──对应──  后续请求中的 ChatMessage.ToolResult.toolCallId
```

- 注册表没有状态迁移：启动构建，运行期只读，配置变更需重启（FR-014）。
- 审计行写入后不更新、不删除。
