# Contract: `ProviderService`（进程内 Java 接口）

**Module**: `oryxos-core`（契约） / `oryxos-provider`（实现） | **Consumers**: 后续特性的 `ReActLoop`

本特性不暴露 HTTP 端点，对外契约就是下面这个进程内接口和它的值对象（字段见 [data-model.md](../data-model.md)）。

## 接口

```java
package io.oryxos.core.provider;

public interface ProviderService {

    /**
     * 同步执行一次模型调用。不执行任何工具，不自动发起后续轮次。
     *
     * @throws IllegalArgumentException   request.messages 为空
     * @throws ProviderNotFoundException  providerName 未登记（消息中列出已登记名）
     * @throws ProviderCallException      外部调用失败或超时（含 providerName 与原因，不含凭证）
     */
    ChatResult chat(ChatRequest request);

    /** 已登记的 provider 名，按配置顺序返回，供状态展示与报错使用。 */
    List<String> providerNames();
}
```

```java
package io.oryxos.core.audit;

public interface LlmCallRecorder {
    /** 实现可以抛异常；调用方负责捕获并打 ERROR 日志，不影响业务结果（FR-011）。 */
    void record(LlmCallRecord record);
}
```

## 行为契约

| # | 前置条件 | 调用 | 必须满足 | 对应需求 |
|---|----------|------|----------|----------|
| C1 | `deepseek` 已登记 | `chat(providerName=deepseek, messages=[User("你好")])` | 返回非空 `text`；请求只到达 deepseek 的 base URL | FR-002/003、US1-1 |
| C2 | 只登记 `deepseek` | `chat(providerName=qwen, ...)` | 抛 `ProviderNotFoundException`，消息含 `[deepseek]`；无任何外部请求；**不**写审计 | FR-004、SC-007 |
| C3 | 任意 | `chat(messages=[])` | 抛 `IllegalArgumentException`；无外部请求；不写审计 | Edge |
| C4 | 提供 `ToolSpec(get_weather)`，模型返回 tool call | `chat(...)` | `toolCalls` 含 `id`、`name=get_weather`、`argumentsJson` 原文；守卫 callback 调用次数 = 0 | FR-007/008、SC-005 |
| C5 | 消息末尾是 `ToolResult(toolCallId=C4.id)` | `chat(...)` | 发给服务端的请求中，该结果作为 tool 角色消息携带同一 id | FR-009、US4-2 |
| C6 | 服务端返回 5xx 或超时 | `chat(...)` | 抛 `ProviderCallException`；外部请求只发 1 次（无重试 / 无 fallback）；写 1 条审计，token 为空 | FR-012、US1-3、US3-2 |
| C7 | 任意成功调用 | `chat(sessionId="s1", ...)` | 写 1 条审计，`session_id=s1`，provider / model / token / duration 齐全 | FR-010、US3-1 |
| C8 | recorder 抛异常 | `chat(...)` | 正常返回 `ChatResult`；输出 1 条 ERROR 日志 | FR-011、US3-3 |
| C9 | 100 个虚拟线程并发调用 | `chat(...)` × 100 | 全部成功，结果与各自请求对应；审计 100 行 | FR-013、SC-002 |

## 架构约束（ArchUnit 强制）

- `io.oryxos.core..` 不得依赖 `org.springframework.ai..`。
- 任何类不得依赖 `org.springframework.ai.chat.client.ChatClient`（宪法 I/II）。
- 任何类不得调用 `ListableBeanFactory.getBeansOfType(ChatModel.class)` 或 `ObjectProvider<ChatModel>` 的批量获取（宪法 III）。
- `oryxos-provider` 不得依赖 `oryxos-storage`（两者只依赖 core）。
