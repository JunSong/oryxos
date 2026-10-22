# Quickstart: 验证「对接 LLM」

**Spec**: [spec.md](./spec.md) | **Contracts**: [provider-service.md](./contracts/provider-service.md)、[provider-config.md](./contracts/provider-config.md)

本特性没有终端用户入口（Demo 一要等 US-2 完成），验证全部通过测试完成。

## 前置条件

- JDK 21、Maven 3.9+
- 可选：`DEEPSEEK_API_KEY`、`KIMI_API_KEY`（只有真实冒烟测试需要）

## 场景 1：CI 必跑集（无需任何 key）

```bash
mvn -pl oryxos-core,oryxos-provider,oryxos-storage,oryxos-boot -am verify
```

期望：全部通过。覆盖：

- 契约 C1~C9（桩服务集成测试）
- 启动校验各条规则
- ArchUnit 架构约束
- `llm_calls` 迁移与写入

## 场景 2：启动校验（SC-003）

```bash
unset KIMI_API_KEY
DEEPSEEK_API_KEY=dummy java -jar oryxos-boot/target/oryxos-boot-*.jar
```

期望：10 秒内退出，退出码非 0，日志里出现
`provider "kimi" 需要环境变量 KIMI_API_KEY，但它未设置或为空`，且不出现 `dummy`。

## 场景 3：真实双 Provider 冒烟（SC-001）

```bash
export DEEPSEEK_API_KEY=...  KIMI_API_KEY=...
mvn -pl oryxos-provider -Dtest='*RealProviderSmokeIT' verify
```

期望：deepseek、kimi 各返回一次非空回复和 token 用量。测试代码只改 `providerName` 一个参数。

## 场景 4：审计落库（SC-002）

场景 1 或 3 跑完后：

```bash
sqlite3 .oryxos/oryxos.db \
  "select provider, model, total_tokens, duration_ms, session_id from llm_calls order by id desc limit 5;"
```

期望：每次调用一行。失败调用的 token 列为空、`duration_ms` 有值。

## 场景 5：凭证零泄漏（SC-004）

```bash
rg -n "$DEEPSEEK_API_KEY|$KIMI_API_KEY" logs/ target/surefire-reports/ target/failsafe-reports/ || echo "OK: 0 hits"
sqlite3 .oryxos/oryxos.db ".dump llm_calls" | rg -c "$DEEPSEEK_API_KEY" || echo "OK: 0 hits"
```

## 场景 6：附加开销（SC-006）

`ProviderOverheadIT` 用固定 0ms 延迟的桩服务连续调用 1000 次，统计平台侧耗时（总耗时减去桩响应耗时），断言 P99 ≤ 50ms。
