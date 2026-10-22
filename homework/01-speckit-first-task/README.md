# 作业：用 Spec Kit 拆解 OryxOS 第一个任务

**题目**：使用当前的四份技术文档，用 spec kit 完成第一个任务的拆解。

**结论**：第一个任务是 **US-1 对接 LLM（Provider 抽象）**（AiProgrammingGuide §1）。走完 constitution → specify →（clarify）→ plan → tasks，最终拆成 **47 个任务、4 个 user story、7 个阶段**，MVP 是「单家模型对话」。

## 目录

```text
homework/01-speckit-first-task/
├── README.md                 # 本文件：作业说明与产物索引
├── PROCESS.md                # 执行过程记录（每步命令、输入、决策、校验结果、与现有 001 的对比）
├── logs/                     # 各步脚本的原始输出
│   ├── 00-env.log            # Spec Kit 版本与环境
│   ├── 01-specify.log        # 模板解析、特性目录登记
│   ├── 02-plan.log           # setup-plan.sh
│   ├── 03-tasks.log          # setup-tasks.sh
│   ├── 04-validate.log       # tasks.md 格式与完整性校验
│   ├── 05-restore.log        # feature.json 还原
│   └── feature.json.before   # 执行前的 .specify/feature.json 备份
└── specs/001-llm-provider/   # Spec Kit 产物（特性目录）
    ├── spec.md               # /speckit-specify：4 个 user story、14 FR、7 SC
    ├── checklists/requirements.md
    ├── plan.md               # /speckit-plan：技术上下文、Constitution Check、模块与文件结构
    ├── research.md           # Phase 0：10 条技术决策 + 文档口径差异裁决
    ├── data-model.md         # Phase 1：值对象 + llm_calls 表
    ├── contracts/
    │   ├── provider-service.md
    │   └── provider-config.md
    ├── quickstart.md         # Phase 1：6 个验证场景
    └── tasks.md              # /speckit-tasks：T001~T047
```

## 阅读顺序

1. `PROCESS.md`：了解每一步怎么做的、为什么这么做
2. `specs/001-llm-provider/tasks.md`：最终拆解结果
3. 需要追溯依据时再看 `spec.md` → `plan.md` → `research.md`
