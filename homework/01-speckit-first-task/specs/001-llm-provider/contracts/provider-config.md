# Contract: Provider 配置（`application.yaml`）

**Owner**: `oryxos-provider` 的 `ProviderProperties` + `ProviderConfigValidator`

运维人员接入新模型服务，唯一需要改的就是这段配置，不需要改代码。

## 格式

```yaml
spring:
  threads:
    virtual:
      enabled: true
  autoconfigure:
    exclude:               # 宪法 II：禁止 eager 模型自动装配；具体类名以 T001 锁定的版本为准
      - org.springframework.ai.model.openai.autoconfigure.OpenAiChatAutoConfiguration
      - com.alibaba.cloud.ai.autoconfigure.dashscope.DashScopeAutoConfiguration

oryxos:
  providers:
    - name: deepseek                 # 唯一，小写字母开头，[a-z0-9-]
      model: deepseek-chat           # 默认模型
      api-key-env: DEEPSEEK_API_KEY  # 环境变量名，不是 key 本身
      # base-url: https://api.deepseek.com   # 可选，缺省用公开地址
      # timeout: 60s                         # 可选
    - name: kimi
      model: moonshot-v1-8k
      api-key-env: KIMI_API_KEY
      base-url: https://api.moonshot.cn
```

## 启动期校验（失败即拒启，退出码非 0）

| 规则 | 示例错误信息（格式约定，不含凭证） |
|------|------------------------------------|
| `providers` 至少 1 条 | `oryxos.providers 为空：至少登记一个模型服务` |
| `name` 唯一 | `provider 名重复：deepseek（第 1、3 条）` |
| `name` 格式 | `provider[2].name 非法："Deep Seek"，只允许 [a-z][a-z0-9-]*` |
| `model` 非空 | `provider "kimi" 缺少 model` |
| `api-key-env` 像变量名 | `provider "deepseek" 的 api-key-env 不是环境变量名（疑似明文 key），请改为如 DEEPSEEK_API_KEY` |
| 变量存在且非空 | `provider "kimi" 需要环境变量 KIMI_API_KEY，但它未设置或为空` |
| `base-url` 合法 | `provider "kimi" 的 base-url 不是 http(s) 绝对地址` |
| `timeout` > 0 | `provider "kimi" 的 timeout 必须大于 0` |

多条错误一次性全部报出，而不是改一个报一个。

## 兼容性

- 新增字段只能是可选项，带默认值。
- 本特性不支持热更新，改配置后重启生效（FR-014）。
