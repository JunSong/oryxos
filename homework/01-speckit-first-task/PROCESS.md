# 执行过程记录：用 Spec Kit 拆解 OryxOS 第一个任务

- **作业题目**：用 spec kit，基于当前四份技术文档，完成第一个任务的拆解
- **执行日期**：2026-10-22
- **分支**：`homework/speckit-us1-decomposition`（从 `main@ad5ed6b7` 切出）
- **产物目录**：`homework/01-speckit-first-task/`，只新增文件，不改仓库已有的 `specs/`、`docs/`、源码

每一步的原始命令输出都留在 `logs/` 目录，下文引用对应文件。

---

## 第 0 步：确认环境与「第一个任务」是什么

### 0.1 Spec Kit 环境

见 `logs/00-env.log`：

| 项 | 值 |
|----|----|
| Specify CLI | 0.13.3.dev0（Python 3.13.2，macOS） |
| 模板版本 | speckit_version 1.0.13 |
| 集成方式 | Claude Code skills 模式：`.claude/skills/speckit-*`（`.trae/skills/` 有同一套） |
| 扩展钩子 | 无 `.specify/extensions.yml`，各命令的 before/after 钩子按规则静默跳过 |
| 宪法 | `.specify/memory/constitution.md` v2.0.1，8 条原则 |

工作区早已 `specify init` 过，这次直接复用。

### 0.2 四份技术文档与「第一个任务」

`docs/AiProgrammingGuide.md` §3.5 列出的四份文档就是输入：

| 文档 | 这次用到的章节 | 用途 |
|------|----------------|------|
| `docs/DemandAnalysis.md`（需求） | §5.3 Provider 抽象、§5.12 配置与密钥、§8 非功能、§10 数据模型、§13 验收 | specify 的需求来源 |
| `docs/TechnicalSolution.md`（技术方案） | §1.1 七个关键决策、§1.2 技术栈、§3 对接 LLM、§8.8 ConfigLoader、§9 持久化、§10 模块、§13 实施节奏 | plan 的技术来源 |
| `docs/AiProgrammingGuide.md`（AI 编程指南） | §1 五个 user story 与依赖顺序、§3 Spec Kit 准备阶段、§4.1 US-1 拆解思路 | 确定「第一个任务」和拆分颗粒度 |
| `docs/IndustryResearch.md`（业界调研） | §4.3、§5.7（复用 Spring AI Alibaba connector，不重复造轮子）、§6.1（实例无状态） | 选型依据 |

**结论**：AiProgrammingGuide §1 把核心阶段拆成 5 个 user story，推进顺序是 US-1 → US-2 → (US-3 ∥ US-4) → US-5。「US-1 是基础，没有 LLM 调用所有 Agent 能力都跑不起来」，所以第一个任务就是 **US-1：对接 LLM（Provider 抽象）**。

### 0.3 产物隔离方案

作业要求「结果产物单独存起来」，我同时做了两层隔离：

1. **新分支** `homework/speckit-us1-decomposition`。
2. **独立目录**。Spec Kit 默认把特性写进 `specs/NNN-xxx/`，仓库里已经有 `specs/001-provider-abstraction/` 等 50 多个特性目录。为了不混进去，我利用 Spec Kit 支持的 `SPECIFY_FEATURE_DIRECTORY` 与 `.specify/feature.json` 机制，把特性目录指到 `homework/01-speckit-first-task/specs/001-llm-provider/`。
3. `.specify/feature.json` 是受版本控制的共享状态，原值 `specs/053-cost-dashboard`。开始前备份到 `logs/feature.json.before`，结束后还原（第 5 步）。

---

## 第 1 步：constitution（复用，不重写）

- **做法**：读 `.specify/memory/constitution.md`，确认 8 条原则覆盖了 AiProgrammingGuide §3.2 要求的内容，因此不重跑 `/speckit-constitution`。
- **理由**：AiProgrammingGuide §3.2 明确「constitution 写一次定下来，整个主体开发期间不改……不允许 AI agent 自己修改 constitution」。
- **和 US-1 直接相关的原则**：
  - I 自实现 ReAct
  - II Spring AI 只做协议转换与 schema
  - III Provider 显式映射
  - V 审计 day one 落库
  - VI 凭证走环境变量
  - VII 同步 + 虚拟线程
  - VIII 状态外置、不用 Hibernate 迁移

## 第 2 步：`/speckit-specify` → `spec.md` + `checklists/requirements.md`

**命令**（见 `logs/01-specify.log`）：

```bash
bash .specify/scripts/bash/resolve-template.sh spec-template > <特性目录>/spec.md
SPECIFY_FEATURE_DIRECTORY=<特性目录> bash .specify/scripts/bash/check-prerequisites.sh --json --paths-only
# 按 speckit-specify skill 第 3 步要求，把特性目录写入 .specify/feature.json
```

**输入**：DemandAnalysis §5.3（Provider 四项配置、不做 fallback、token 记录），加上 §8 与 §13 的验收要求。

**产出要点**：

- 4 个可独立测试的 user story：
  - P1 单家对话（MVP）
  - P2 多家按名路由
  - P2 调用审计落库
  - P3 工具描述透传、tool call 原样交回
- 14 条 FR、7 条可度量 SC、9 条边界情况。
- SC 只用次数、比例、时间表达，不出现框架名；需要技术细节的约束留到 plan。

**质量检查**（skill 第 8 步）：生成 `checklists/requirements.md`，16 项全部通过。第 1 轮发现 1 处实现细节泄漏（Clarifications 里写了 `chatModel.call`），改成「只用底层单次模型调用」。

## 第 3 步：clarify（按文档优先级自行裁决）

specify 过程中发现四份文档有三处口径不一致。spec kit 允许在 specify 阶段用合理默认值代替提问，最多留 3 个 `[NEEDS CLARIFICATION]` 标记。这三处都能按宪法「本宪法凌驾于其它一切实践之上」裁决，所以没有留标记，也没有打断提问，结论直接写进 `spec.md` 的 `## Clarifications`：

| 冲突 | 文档 A | 文档 B | 裁决 |
|------|--------|--------|------|
| token 用量记在哪 | 需求 §5.3「落到日志」 | 技术方案 §3.3、宪法 V「写 `llm_calls`」 | 落库 |
| 验收跑通几家 | 编程指南 §4.1「DeepSeek 或 Kimi」 | 需求 §13「DeepSeek 和 Kimi」 | 两家 |
| 调用入口 | 需求 §5.3、技术方案 §3「`ChatClient`」 | 宪法 I/II「只用 `chatModel.call`」 | `ChatModel` |

## 第 4 步：`/speckit-plan` → plan / research / data-model / contracts / quickstart

**命令**（见 `logs/02-plan.log`）：

```bash
bash .specify/scripts/bash/setup-plan.sh --json
# 输出 FEATURE_SPEC / IMPL_PLAN / FEATURE_DIR，已把 plan 模板拷进特性目录
```

**输入**：`spec.md`、宪法、技术方案 §1.1/§1.2/§3/§8.8/§9/§10、编程指南 §4.1。

**Constitution Check**：Phase 0 前后各做一次，8 条原则加技术栈约束全部 ✅，原则 IV 不涉及。只有一处偏离编程指南、需要说明，写在 Complexity Tracking：编程指南 §4.1 只列了 core / provider / boot 三个模块，但宪法 V 要求审计 day one 落库，所以加上 `oryxos-storage`。

**Phase 0 research.md**：10 条决策，每条都写了 Decision / Rationale / Alternatives。最关键的四条：

- **R3 显式映射**：不让 Spring 自动装配任何 `ChatModel`，按配置列表逐条手工构建 `Map<String, ChatModel>`。新增 provider 只改配置、不改代码。
- **R4 凭证形态**：配置项写 `api-key-env: DEEPSEEK_API_KEY`（变量名），不写 `${...}` 占位。配置里根本没有放明文的位置；报错也能直接点出缺哪个变量。对比过 Spring 占位符方案，后者报错不带 provider 名，也拦不住明文。
- **R5 不执行工具的双保险**：既关掉 `internalToolExecutionEnabled`，又让交给模型的 `ToolCallback` 一被执行就抛异常。SC-005「执行次数为 0」因此可以直接断言。
- **R8 失败语义**：关闭框架重试，外部请求只发 1 次，审计条数和实际请求一一对应。

**Phase 1**：

- `data-model.md`：7 个值对象，每个字段都带校验规则；`llm_calls` 表 9 列，与需求 §10 一致，不增不减。
- `contracts/provider-service.md`：`ProviderService` 接口、9 条行为契约 C1~C9，以及 4 条 ArchUnit 架构约束。本特性没有 HTTP 端点，契约就是进程内接口。
- `contracts/provider-config.md`：配置格式与 8 条启动校验规则及错误文案。
- `quickstart.md`：6 个验证场景。

## 第 5 步：`/speckit-tasks` → `tasks.md`

**命令**（见 `logs/03-tasks.log`）：

```bash
bash .specify/scripts/bash/setup-tasks.sh --json
# AVAILABLE_DOCS = [research.md, data-model.md, contracts/, quickstart.md]
```

**拆解结果**：47 个任务，按 user story 分 7 个阶段。

| 阶段 | 任务 | 数量 | 说明 |
|------|------|------|------|
| Phase 1 Setup | T001~T004 | 4 | 9 模块骨架、BOM 锁版本并核实 API、质量门禁、启动类 |
| Phase 2 Foundational | T005~T015 | 11 | core 契约与值对象、测试桩、ArchUnit 宪法守卫 |
| Phase 3 US1 (P1) MVP | T016~T025 | 10 | 单家对话 + 启动校验 |
| Phase 4 US2 (P2) | T026~T030 | 5 | 多家路由 + 真实双家冒烟 |
| Phase 5 US3 (P2) | T031~T036 | 6 | `llm_calls` 迁移、实体、recorder、存储装配 |
| Phase 6 US4 (P3) | T037~T041 | 5 | 工具描述适配、tool call 透传 |
| Phase 7 Polish | T042~T047 | 6 | 并发、开销、凭证泄漏、文档同步、`mvn verify`、quickstart |

和编程指南 §4.1 的四个大类对应如下：

- 环境搭建类：T001~T004
- 核心抽象类：T005~T012，包括 `OryxTool`、`Profile`、`Message`
- Provider 实现类：T019~T023、T039~T041
- 配置类：T013、T019、T025、T029

编程指南特别提醒的「不能靠扫描容器 `ChatModel` 区分 Provider」，落在 T015（ArchUnit 规则）和 T022（注册表）。

**格式校验**（见 `logs/04-validate.log`）：

- 47 条任务全部符合 `- [ ] T### [P]? [USn]? 描述 + 文件路径`，格式不合规 0 条。
- ID 从 T001 到 T047 连续。
- 无遗留模板占位符。
- 28 条标了 [P]。
- `check-prerequisites.sh --require-tasks --include-tasks` 通过。

**MVP 与合并门槛**：MVP 是 US1；但宪法 V 规定审计不能缺席，所以第一个可合并的增量是 US1 + US3。

## 第 6 步：和仓库现有 `specs/001-provider-abstraction/` 对比

仓库里有一份 2026-07 为同一主题做的拆解（20 个任务，已实施）。作业产物是按「当前」四份文档重新做的，下表只用来复盘，不改动原文件：

| 维度 | 现有 001（2026-07） | 本次作业 | 差异原因 |
|------|---------------------|----------|----------|
| user story 切法 | 路由 + 透传 / 审计 / 配置即运维（3 个） | 单家对话 / 路由 / 审计 / 工具透传（4 个） | 把 MVP 收到最小的单家对话，工具透传单独成故事，方便 US-2 直接接手 |
| Profile 加载 | 包含（`ProfileLoader` 扫 `.oryxos/profiles/*.yaml`，坏文件跳过） | 不包含，只立 `Profile` / `ProviderRef` 结构 | 当前文档已改成「一个目录 = 一个 Agent」，`.oryxos/profiles/` 已取消，加载交给 AgentLoader 特性 |
| 凭证配置 | `api-key: ${ENV}` 解析后非空校验 | `api-key-env: 变量名` | 从结构上杜绝明文，报错能点出变量名（research R4） |
| 建表 | `schema.sql` 手工脚本 | Flyway `V1__llm_calls.sql` | 宪法 VIII 与 025 之后的演进要求 Flyway |
| 失败原因落库 | 有 `success`、`error_message` 列 | 不加列，失败原因走 ERROR 日志 | 当前需求 §10 / 技术方案 §9 的 `llm_calls` 只有 9 列（YAGNI）。这是一项可讨论的取舍，见下文「待讨论」 |
| 宪法守卫 | 末尾用 grep 脚本自查 | Foundational 阶段就放 ArchUnit 规则 | 让违规在第一次 `mvn test` 时就失败 |
| 版本 API 核实 | 单列 T003「解包 jar 核实 API」 | 并入 T001 | 吸收了原做法：Spring AI 的工具开关在不同版本里名字不同（M6 叫 `proxyToolCalls`，1.0 GA 叫 `internalToolExecutionEnabled`），必须先核实再写 |

另外，仓库主干现状已经演进到 Spring Boot 4.1.1、Spring AI 2.0.1、14 个模块。作业按题目要求只以四份文档为准，当作 greenfield 拆解，所以技术栈写的是文档里的 Boot 3.x + Spring AI Alibaba。

## 第 7 步：收尾

- `.specify/feature.json` 已还原为 `specs/053-cost-dashboard`，`git diff` 为空（见 `logs/05-restore.log`）。
- 只提交 `homework/01-speckit-first-task/` 目录，不带 `.vscode/settings.json` 等无关改动。

---

## 待讨论 / 下一步

1. **`llm_calls` 要不要加 `success` / `error_message` 列**：现有实现加了，当前文档没有。如果审计要回答「失败率多少」，建议加。改法是 spec FR-010 加两个字段，再加一条 Flyway 迁移。
2. **analyze**：按编程指南 §3.4 与宪法流程，US-1 + US-2 实施后再跑 `/speckit-analyze` 做 spec 与代码的一致性检查。本次只做到 tasks。
3. **implement**：作业范围是「拆解」，没有执行 `/speckit-implement`。
