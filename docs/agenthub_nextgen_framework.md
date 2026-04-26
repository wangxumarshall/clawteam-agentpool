# AgentHub：面向下一代生产级多智能体框架的技术方案

## 0. 摘要

AgentPool 已经证明了一个关键方向：多智能体系统的核心问题不是“如何再封装一个 LLM 调用”，而是如何把 PydanticAI、Claude Code、Codex、OpenCode、ACP agent、AG-UI agent、MCP 工具、A2A 对等智能体、OpenAI-compatible API 等异构能力，统一纳入一个可配置、可编排、可观测、可恢复的运行时。

面向下一代生产环境提出的多智能体框架设想，AgentHub继承 AgentPool 的多协议异构 agent 编排能力，但进一步把系统目标从“统一接入和路由”提升为“自治、持久、可治理的智能体操作系统层”。

AgentHub 的核心定位是： 一个以统一智能体抽象为内核，以协议网关为外壳，以蜂群自治为执行形态，以持久化状态和记忆计算为底座的生产级多智能体框架。

它面向的不是单次问答或简单工作流，而是企业中持续数小时、数天乃至长期在线的复杂智能体应用：

- 软件研发自动化。
- 企业知识研究与审计。
- 数据分析与报表生成。
- DevOps 与云资源自治运维。
- 安全合规巡检。
- 客服、销售、运营等多渠道业务协同。
- 跨组织、跨供应商、跨模型的 agent-to-agent 网络。

## 1. 业务场景

### 1.1 企业级软件研发自治

典型需求：

- 从需求文档、Issue、PR 评论中提取任务。
- 自动拆解为架构设计、代码修改、测试生成、安全扫描、文档更新等子任务。
- 调度 Claude Code 进行深度架构理解，调度 Codex 进行快速实现和测试修复，调度 OpenCode 操作远程沙箱和 LSP 代码环境。
- 在 ACP IDE、OpenCode TUI、AG-UI 工作台中同步展示进度和等待人工审批。
- 通过 MCP 调用内部 Git、CI、制品库、Jira、Wiki、监控系统。
- 任务中断后可从 checkpoint 继续执行。

AgentHub 在该场景中承担研发操作系统角色是：统一 agent、工具、沙箱、权限、状态和协作拓扑。

### 1.2 长周期研究与决策支持

典型需求：

- 围绕一个复杂主题进行多轮 research。
- 多个 research agent 并行检索外部资料、阅读内部知识库、分析历史报告。
- 结果进入审稿、反驳、补证、再生成循环。
- 人类专家可以在 AG-UI 前端或 IDE 中随时插入反馈。
- 研究结论必须保留证据链、版本、可信度和引用来源。

AgentHub 不只是调度 agent 写报告，而是维护一个可追溯的研究状态机。

### 1.3 企业业务流程自动化

典型需求：

- 处理采购审批、合同审阅、财务对账、客户支持升级等跨系统流程。
- 调用 ERP、CRM、数据库、邮件、IM、工单系统等工具。
- 不同环节有不同权限、审计、审批和 SLA 要求。
- 某些步骤可完全自动化，某些步骤必须 human-in-the-loop。

AgentHub 需要把 agent 工作流与企业 BPM、权限、合规、审计系统融合。

### 1.4 自治运维与云资源治理

典型需求：

- 监控告警触发诊断 agent。
- agent 自动读取日志、指标、trace、配置和部署记录。
- 初级修复由低成本模型处理，复杂根因分析委托给高推理模型。
- 高风险操作进入审批链。
- 多个 worker 在隔离沙箱中模拟修复方案，最终由 coordinator 合并。

AgentHub 在这里是持续运行的自治集群控制面。

### 1.5 多渠道智能服务

典型需求：

- 同一个企业智能体能力需要同时暴露给 Web、移动端、Slack、Telegram、Email、OpenAI API、A2A peer、MCP client。
- 不同渠道有不同消息格式、权限模型和上下文长度。
- 用户从一个渠道开始任务，可以在另一个渠道继续。

AgentHub 通过协议网关与统一 session/memory 模型解决渠道割裂。

## 2. 问题挑战

### 2.1 异构协议碎片化

现有 agent 生态的协议分工越来越清晰，但也越来越复杂：

| 协议或接口 | 主要能力 | 典型使用位置 |
| --- | --- | --- |
| MCP | 工具、资源、prompt 标准化 | 工具生态和企业系统接入 |
| ACP | IDE/客户端和 agent 的双向交互 | 本地开发环境、文件系统、终端 |
| A2A | agent-to-agent 发现、任务委托和协作 | 跨框架、跨组织、多 agent 网络 |
| AG-UI | 前端 UI 和 agent 后端的流式事件 | Web 工作台、人类反馈、状态可视化 |
| OpenCode | TUI/Desktop、代码 session、permission、LSP/文件操作 | 软件工程和远程开发环境 |
| OpenAI API | 兼容传统模型客户端 | 低迁移成本集成 |

问题在于，这些协议不只是字段不同，而是底层语义不同。它们分别关注工具、客户端、前端、对等协作、代码环境和模型兼容。没有统一网关，业务系统会迅速陷入适配器爆炸。

### 2.2 Agent 类型和运行时差异巨大

一个真实企业系统可能同时使用：

- PydanticAI native agent：类型安全、结构化输出、Python 内嵌。
- Claude Code：深度代码理解和复杂重构。
- Codex：快速代码编辑、测试生成、终端任务。
- OpenCode：开放模型、TUI/Desktop、远程文件和 LSP。
- Goose 或其他 ACP agent：外部本地 agent 能力。
- AG-UI remote agent：前端交互型 agent 服务。
- A2A peer：外部组织或供应商 agent。

这些 agent 的生命周期、上下文模型、工具机制、权限模型、流式事件和错误处理都不一致。简单的“统一 prompt”无法解决运行时差异。

### 2.3 长时任务的失败概率会指数级累积

长时任务可能包含数百个步骤：

- 读取资料。
- 工具调用。
- 代码修改。
- 测试运行。
- 结果审阅。
- 人工审批。
- 子任务回滚。
- 二次修复。

单步失败率即便很低，链路整体失败率也会快速上升。没有强类型校验、状态持久化、可恢复执行和多轮自修正循环，长时任务很难进入生产环境。

### 2.4 并发 agent 会产生资源冲突

多 agent 同时操作同一个代码库、数据库或远程环境时，常见风险包括：

- 同文件并发修改。
- 工具调用覆盖。
- Git 分支冲突。
- 终端状态污染。
- 临时文件和缓存互相影响。
- 一个 worker 的错误上下文污染其他 worker。

生产级框架必须提供物理级或逻辑级隔离，例如 git worktree、容器、远程沙箱、权限域和会话空间。

### 2.5 记忆和状态不是同一件事

企业 agent 系统通常混淆三类数据：

- 执行状态：当前工作流走到哪一步，哪些 task 已完成，哪些资源锁已持有。
- 对话历史：用户、agent、工具调用的消息序列。
- 业务记忆：长期可复用的知识、偏好、经验、证据链和决策结果。

AgentHub 必须区分并融合这三者。执行状态需要强一致和可恢复；对话历史需要可检索和可压缩；业务记忆需要可计算、可演化和可治理。

## 3. 现有技术方案分析

### 3.1 AgentPool：多协议异构 agent 编排枢纽

AgentPool 的核心贡献是把异构 agent 纳入统一 pool：

- YAML-first 配置。
- `MessageNode` 统一抽象。
- `AgentPool` 注册表和生命周期管理。
- Native、Claude Code、Codex、ACP、AG-UI、file agents 等多类型支持。
- ACP、MCP、AG-UI、OpenCode、OpenAI API、A2A 等多协议暴露。
- Team 支持 parallel 和 sequential 组合。
- MCPManager 与 ToolManagerBridge 区分 inbound MCP 和 outbound tool bridge。

AgentPool 已经解决了“如何让不同 agent 能够在同一系统中被发现、调用、组合和暴露”的问题。

但从下一代生产需求看，AgentPool 仍有可演进空间：

- wrapped agents 对 pool-level/team-level MCP provider 的可见性不一致。
- AG-UI agent 的 MCP provider 接入尚未完全统一。
- A2A 能力存在，但需要 CLI、配置和治理层面的一等化。
- message、mention、resource、artifact 跨协议统一模型还需要进一步产品化。
- 长时任务的 durable execution、checkpoint、memory compute、swarm governance 仍可上升为一等内核能力。

### 3.2 PydanticAI方案：强类型与持久化执行

PydanticAI 的方向强调：

- typed agent。
- structured output。
- dependency injection。
- graph / workflow support。
- durable execution。
- model retry 和 schema validation。

这类方案适合作为 AgentHub 的认知和控制流内核，因为生产系统不能完全依赖 LLM 自觉遵守文本协议。强类型 schema 是把概率生成驯化为工程实体的关键。

但 PydanticAI 本身更偏 agent runtime 和类型安全，并不天然解决多协议网关、异构 coding agent、OpenCode TUI、ACP IDE、A2A 跨组织协作等生态问题。

### 3.3 Swarm 类方案：动态集群和物理隔离

Swarm 类方案强调：

- Leader-worker 拓扑。
- 动态任务拆解。
- worker 按需生成。
- git worktree 或容器隔离。
- inbox、broadcast、kanban 等 agent 间协调机制。
- 空闲 worker 自动认领任务。

这类方案解决的是“当任务变大时，如何让 agent 像团队一样工作”。

但如果缺少统一协议网关和持久化内核，swarm 容易变成一组并发进程，难以治理、审计和恢复。

### 3.4 Temporal / Prefect / DBOS 类方案：持久化工作流

Durable execution 系统解决：

- 任务中断恢复。
- 确定性 replay。
- activity retry。
- 长时间等待人工审批。
- 外部 API 调用去重。
- 分布式状态一致性。

它们为长时 agent 任务提供了可靠执行底座，但本身不理解 agent、tool、LLM、MCP、AG-UI、ACP 等语义。

### 3.5 AgentHub 的位置

AgentHub 应该把上述能力组合成一个统一框架：

- 以 AgentPool 为多协议和异构 agent 接入基础。
- 以 PydanticAI 类类型系统作为认知执行内核。
- 以 Swarm 类机制作为动态集群执行形态。
- 以 Temporal/Prefect/DBOS 类 durable engine 作为长时任务可靠性底座。
- 以 MCP/ACP/A2A/AG-UI/OpenCode/OpenAI API 作为协议生态外壳。

## 4. AgentHub 技术方案总览

AgentHub 的设计目标是形成一个“智能体应用操作系统层”：

- 向上承载业务应用。
- 向下管理模型、工具、沙箱、存储、记忆和协议。
- 向内统一 agent 抽象和任务状态机。
- 向外暴露多协议 gateway。

### 4.1 Agent 应用复杂度五层

AgentHub 将智能体应用复杂度划分为五个递进层次。不同层级使用不同控制机制，避免简单场景过度设计，也避免复杂场景缺乏治理。

| 层级 | 名称 | 架构特征 | AgentHub 支撑能力 |
| --- | --- | --- | --- |
| L1 | 单 Agent 任务 | 单轮或短链路工具调用 | typed agent、tool schema、structured output、basic memory |
| L2 | Agent 委托 | 主 agent 将子任务委托给专门 agent，结果返回主流程 | subagent tool、capability routing、result schema validation |
| L3 | 编程式交接 | 控制权从一个 agent 转交给另一个 agent，形成明确状态机 | handoff protocol、task envelope、state transition guard |
| L4 | 图式多 Agent 工作流 | DAG、循环、自修正、并行分支、人工审批 | durable graph、checkpoint、event bus、policy gate |
| L5 | 自治 Agent 集群 | 动态拓扑、swarm、长期记忆、资源隔离、自组织治理 | autonomous topology、worktree/container sandbox、memory compute、cluster governance |

这五层不是五套框架，而是同一内核的不同控制强度。

### 4.2 统一抽象：`AgentUnit`

AgentPool 的 `MessageNode` 很适合统一“可处理消息的节点”。AgentHub 在此基础上引入更完整的 `AgentUnit` 抽象。

`AgentUnit` 表示一个可治理、可调度、可审计的智能体执行单元，包含：

- identity：名称、类型、版本、owner、tenant。
- capability：能力声明、工具边界、协议支持、成本模型。
- runtime：执行后端，例如 native、claude_code、codex、opencode、acp、agui、a2a。
- context：输入上下文、资源引用、权限域、会话信息。
- memory_scope：可读写的记忆空间。
- state_policy：checkpoint、retry、timeout、compaction 策略。
- tool_policy：可用工具、审批策略、危险操作拦截。
- protocol_adapters：ACP、MCP、A2A、AG-UI、OpenCode、OpenAI API 转换器。
- observability：trace、metric、event、cost、token、artifact。

简化后的概念模型：

```python
class AgentUnit:
    identity: AgentIdentity
    capabilities: list[Capability]
    runtime: RuntimeAdapter
    memory_scope: MemoryScope
    tool_policy: ToolPolicy
    state_policy: StatePolicy
    protocol_adapters: list[ProtocolAdapter]
```

这样，AgentHub 不只知道“这个节点能处理消息”，还知道“这个 agent 能做什么、能访问什么、如何恢复、如何治理、如何暴露给外部协议”。

### 4.3 统一任务载体：`TaskEnvelope`

跨协议、跨 agent、跨长时任务时，不能直接传原始 message。AgentHub 应引入统一任务载体 `TaskEnvelope`：

- task_id。
- parent_task_id。
- goal。
- constraints。
- input artifacts。
- required capabilities。
- protocol origin。
- memory references。
- permission context。
- expected output schema。
- budget limits。
- deadline / SLA。
- current state。
- evidence chain。

所有协议入口都会被转换成 `TaskEnvelope`，所有 agent 输出也会回写到 `TaskEnvelope` 的状态机中。

### 4.4 统一能力模型：`Capability Graph`

AgentHub 不应只按 agent 名称路由，而应按能力路由。

能力图包含：

- agent capabilities：代码重构、测试生成、检索、推理、UI 反馈。
- tool capabilities：read_file、edit_file、run_tests、query_db、search_docs。
- protocol capabilities：是否支持 ACP、MCP、A2A、AG-UI、OpenCode。
- environment capabilities：本地、Docker、SSH、Kubernetes、E2B、浏览器。
- cost and latency profile：成本、速度、上下文长度、可靠性。
- trust and permission profile：租户、密级、审计等级。

路由器根据任务要求选择最合适的 agent 或 team，而不是硬编码 `claude`、`codex`、`opencode`。

## 5. 总体架构设计

AgentHub 采用六层架构：

```text
┌─────────────────────────────────────────────────────────────┐
│ Application Layer                                            │
│ IDE / Web Console / OpenAI API / A2A Peer / Business Apps     │
├─────────────────────────────────────────────────────────────┤
│ Protocol Gateway Layer                                       │
│ ACP / MCP / A2A / AG-UI / OpenCode / OpenAI API / Webhooks    │
├─────────────────────────────────────────────────────────────┤
│ Orchestration Kernel                                         │
│ TaskEnvelope / AgentUnit / Capability Graph / Policy Engine   │
├─────────────────────────────────────────────────────────────┤
│ Swarm Runtime Layer                                          │
│ Teams / Handoff / DAG / Autonomous Topology / Sandboxes       │
├─────────────────────────────────────────────────────────────┤
│ Memory & Durable State Layer                                 │
│ Event Sourcing / Checkpoint / Vector Memory / Evidence Graph  │
├─────────────────────────────────────────────────────────────┤
│ Infrastructure & Observability Layer                         │
│ Model Providers / Tool Runtime / Storage / Tracing / Cost     │
└─────────────────────────────────────────────────────────────┘
```

### 5.1 Protocol Gateway Layer

协议网关负责把所有外部入口归一化为 `TaskEnvelope`，并把内部事件投射回不同协议。

核心职责：

- ACP gateway：IDE 会话、文件系统、终端、权限确认。
- MCP gateway：工具/资源/prompt inbound；AgentHub 能力 outbound。
- A2A gateway：agent card、task delegation、peer discovery。
- AG-UI gateway：前端状态流、工具事件、人类审批、生成式 UI。
- OpenCode gateway：TUI/Desktop session、message parts、question、permission、MCP 管理、LSP/文件事件。
- OpenAI API gateway：chat/completions/responses 兼容。
- Webhook gateway：CI、监控、工单、IM 事件触发。

网关不是简单 proxy，而是协议语义中间件：

- 做鉴权。
- 做租户隔离。
- 做任务归一化。
- 做 artifact 引用转换。
- 做流式事件转换。
- 做权限和审批注入。
- 做审计记录。

### 5.2 Orchestration Kernel

编排内核是 AgentHub 的控制中心。

核心模块：

- Task Manager：创建、暂停、恢复、取消、重试任务。
- Agent Registry：管理 `AgentUnit`。
- Capability Router：按能力、成本、权限、状态选择 agent。
- Policy Engine：权限、预算、工具审批、数据边界、合规约束。
- State Machine：管理 task 状态转换。
- Event Bus：发布任务、工具、agent、协议、存储事件。
- Artifact Manager：管理文件、patch、报告、截图、日志、结果数据。

状态流转示例：

```text
created -> planned -> assigned -> running -> waiting_review
        -> revising -> validating -> completed
        -> failed | cancelled | paused
```

每个状态转换都应可验证、可审计、可恢复。

### 5.3 Swarm Runtime Layer

Swarm Runtime 负责执行复杂任务。

支持四种执行拓扑：

| 拓扑 | 使用场景 |
| --- | --- |
| Chain | 顺序流程，例如研究 -> 审阅 -> 报告 |
| Parallel Team | 并行调研、并行实现、并行验证 |
| DAG Workflow | 有依赖的复杂业务流程 |
| Autonomous Swarm | 动态生成 worker、自组织任务认领、长期运行 |

关键能力：

- Leader-worker 模式。
- Planner-executor-critic 模式。
- 多轮自修正循环。
- worker 动态生成和回收。
- git worktree / container / remote sandbox 隔离。
- inbox / broadcast / kanban 协作协议。
- resource lock 和 merge coordinator。
- idle worker rebalancing。

### 5.4 Memory & Durable State Layer

AgentHub 把状态、历史和记忆分层处理。

| 类型 | 作用 | 技术形态 |
| --- | --- | --- |
| Execution State | 保证任务可靠执行 | event sourcing、checkpoint、workflow DB |
| Conversation History | 支撑会话恢复和审计 | message store、session store、compaction |
| Semantic Memory | 支撑长期知识复用 | vector index、knowledge graph、summary memory |
| Evidence Graph | 支撑研究和合规证明 | artifact links、citations、decision records |
| Operational Memory | 支撑系统自优化 | tool success rate、agent score、cost history |

记忆计算不是简单 RAG。AgentHub 应支持：

- 记忆写入策略。
- 记忆压缩。
- 证据可信度。
- 记忆遗忘。
- 版本化记忆。
- tenant-scoped memory。
- task-scoped scratch memory。
- agent personal memory。
- organization shared memory。

### 5.5 Infrastructure & Observability Layer

生产级多 agent 系统必须默认可观测。

AgentHub 应记录：

- 每个 task 的状态机事件。
- 每个 agent 的输入、输出、工具调用、token、成本。
- 每次协议转换。
- 每个 artifact 的来源和用途。
- 每次 human approval。
- 每次 retry 和 auto-fix。
- 每个 worker 的资源占用。
- 每个模型的成功率、延迟、失败类型。

这些数据同时服务于：

- 调试。
- 审计。
- 成本控制。
- 路由优化。
- agent 评分。
- 记忆演化。
- SLA 分析。

## 6. 核心功能设计

### 6.1 多协议融合与网关中间件

AgentHub 的协议网关应提供统一中间件链：

```text
auth -> tenant -> quota -> policy -> normalize -> route
     -> execute -> stream_convert -> audit -> persist
```

每个协议适配器只负责协议语义转换，不直接持有业务逻辑。业务逻辑进入统一内核处理。

典型转换：

- ACP session message -> `TaskEnvelope`。
- AG-UI user action -> `HumanFeedbackEvent`。
- MCP tool call -> `ToolInvocation`。
- A2A task request -> `ExternalDelegationTask`。
- OpenCode message part -> `ArtifactReference`。
- OpenAI chat completion -> `CompatTaskEnvelope`。

### 6.2 更优雅的统一抽象

AgentHub 采用五个核心对象：

| 抽象 | 含义 |
| --- | --- |
| `AgentUnit` | 可调度、可治理、可暴露的智能体单元 |
| `TaskEnvelope` | 跨协议统一任务载体 |
| `Capability` | agent、tool、environment 的能力声明 |
| `Artifact` | 文件、patch、报告、日志、图像、结果等可追踪产物 |
| `MemoryCell` | 可计算、可版本化、可治理的记忆单元 |

这些抽象让 AgentHub 避免把系统建在脆弱的文本消息和字符串路由上。

### 6.3 多轮自修正循环

AgentHub 内建自修正循环，而不是让每个 agent 自己在 prompt 中猜测。

标准循环：

```text
plan -> execute -> validate -> diagnose -> repair -> revalidate -> summarize
```

每一步都需要：

- 明确输入输出 schema。
- 明确失败类型。
- 明确最大重试次数。
- 明确预算限制。
- 明确升级策略。

失败类型示例：

- schema_error。
- tool_error。
- permission_denied。
- test_failure。
- compile_error。
- missing_context。
- conflicting_edits。
- budget_exceeded。
- human_rejected。

修复策略示例：

- schema_error：交给原模型带 schema guidance 重试。
- test_failure：交给 Codex 快速修复。
- architectural_conflict：交给 Claude Code 深度分析。
- missing_context：触发 MCP/knowledge retrieval。
- conflicting_edits：交给 merge coordinator。
- human_rejected：进入重新规划状态。

### 6.4 蜂群与自治集群拓扑

AgentHub 的 swarm 不应只是并行执行，而应具备自治拓扑治理。

核心机制：

- Dynamic Planner：根据任务复杂度生成 worker 拓扑。
- Worker Archetype：用模板定义角色、能力、权限和默认工具。
- Sandbox Allocator：为 worker 分配 worktree、container、remote env。
- Task Board：维护可认领任务、阻塞任务、完成任务。
- Inbox/Broadcast：支持点对点和全局通信。
- Merge Coordinator：合并 worker 成果，解决冲突。
- Swarm Evaluator：评估 worker 产出质量并调整拓扑。
- Rebalancer：根据负载、失败率、成本动态迁移任务。

拓扑可以从静态 YAML 演进为动态策略：

```yaml
swarm_policy:
  mode: autonomous
  max_workers: 12
  isolation: git_worktree
  spawn_when:
    - task_parallelizable
    - estimated_duration_gt: 15m
  merge_strategy: reviewed_patch
  idle_strategy: pull_from_board
```

### 6.5 持久化存储与记忆计算融合

AgentHub 应把 durable execution 和 memory compute 合并为一个统一数据平面。

执行数据进入 event log：

- task created。
- plan generated。
- worker spawned。
- tool called。
- artifact produced。
- validation failed。
- repair attempted。
- human approved。
- task completed。

语义数据进入 memory graph：

- 关键事实。
- 业务规则。
- 设计决策。
- agent 经验。
- 工具使用效果。
- 失败模式。
- 人类偏好。

二者之间通过 evidence link 连接：

```text
MemoryCell -> derived_from -> Artifact
Artifact -> produced_by -> AgentUnit
AgentUnit -> executed_in -> TaskEnvelope
TaskEnvelope -> recorded_in -> EventLog
```

这样，AgentHub 不只是“记得内容”，还知道这些内容从哪里来、由谁生成、是否被验证、在哪些任务中复用过。

### 6.6 权限、安全与治理

生产级 AgentHub 必须默认零信任：

- agent 只获得最小工具权限。
- 高危工具默认需要审批。
- 不同租户 memory 隔离。
- 机密数据进入 policy guard。
- 外部 A2A peer 必须鉴权和能力限制。
- MCP server 接入需要 allowlist 和审计。
- 代码执行进入 sandbox。
- 所有 artifact 带 lineage。

权限对象应至少覆盖：

- user。
- tenant。
- agent。
- tool。
- resource。
- protocol origin。
- environment。
- task risk level。

### 6.7 统一开发体验

AgentHub 的配置不应退化为不可维护的巨型 YAML。建议采用三层配置：

- `agenthub.yml`：全局协议、存储、模型、权限、observability。
- `agents/*.yml`：agent unit 和 capability 定义。
- `tasks/*.yml`：业务任务模板、swarm policy、output schema。

同时支持代码方式定义高级工作流：

```python
async with AgentHub("agenthub.yml") as hub:
    task = await hub.tasks.create(
        goal="审计支付模块并生成重构方案",
        required_capabilities=["code_analysis", "architecture_review", "test_generation"],
    )
    result = await hub.run(task)
```

原则是：静态拓扑用配置，复杂控制流用 typed code，所有运行时状态都进 durable state。

## 7. 方案设计：关键业务流

### 7.1 长时研发任务全链路

阶段一：入口归一化

- 用户从 AG-UI 工作台发起任务。
- Protocol Gateway 生成 `TaskEnvelope`。
- Policy Engine 检查权限、预算、风险等级。
- Durable State 创建 workflow instance。

阶段二：规划与拆解

- Coordinator 分析目标。
- Capability Router 选择 Claude Code、Codex、OpenCode、PydanticAI reviewer。
- Swarm Runtime 创建 worker 拓扑。
- Sandbox Allocator 分配 worktree 和容器。

阶段三：并行执行

- Claude Code 负责架构理解。
- Codex 负责快速实现和测试。
- OpenCode 负责远程文件/LSP 精确编辑。
- PydanticAI reviewer 负责 schema 校验和结构化报告。
- MCP 工具提供 Git、CI、Wiki、Issue、数据库能力。

阶段四：验证与自修正

- 测试失败进入 diagnose。
- 根据失败类型路由给对应 agent。
- 修复结果重新验证。
- 超过预算或风险阈值则等待人类确认。

阶段五：人工审批与交付

- AG-UI 推送 diff、测试报告、风险评估。
- 用户审批后，OpenCode/Codex 创建 PR。
- AgentHub 写入 evidence graph 和 memory。
- 任务关闭，worker sandbox 回收。

### 7.2 企业研究任务全链路

- Research coordinator 创建问题树。
- 多个 research worker 并行检索内外部资料。
- Critic agent 对结论进行反驳。
- Evidence agent 检查引用和证据。
- Human expert 在 AG-UI 前端批注。
- Coordinator 生成第二版、第三版报告。
- 最终报告和证据链进入 Memory Graph。

### 7.3 自治运维任务全链路

- 监控 webhook 触发 incident task。
- AgentHub 拉取日志、指标、trace、最近部署。
- Planner 判断是否可自动修复。
- 低风险修复自动执行。
- 高风险操作等待审批。
- 结果同步到工单、知识库和长期记忆。

## 8. 与 AgentPool 的关系

AgentHub 不应推翻 AgentPool，而应把 AgentPool 升级为下一代核心子系统：

| AgentPool 能力 | AgentHub 演进 |
| --- | --- |
| YAML-first agent config | 多层配置 + typed task template |
| MessageNode | AgentUnit + TaskEnvelope + Capability |
| AgentPool registry | Agent Registry + Capability Graph |
| Team parallel/sequential | DAG + swarm + autonomous topology |
| MCPManager / ToolBridge | 双向 MCP data plane + governance |
| serve-acp/agui/mcp/opencode/api | Protocol Gateway middleware |
| Storage/history | Durable state + memory compute + evidence graph |
| Event handlers | Unified event bus + replayable event sourcing |

可以把 AgentPool 看作 AgentHub 的互操作核心，把 AgentHub 看作 AgentPool 在生产级长时任务、自治集群和记忆计算方向上的系统化扩展。

## 9. 预期效果

### 9.1 业务效果

- 缩短复杂研发任务周期。
- 降低多 agent 应用从 PoC 到生产的改造成本。
- 统一企业内不同 agent、工具、前端和业务系统接入方式。
- 让长时 research、coding、ops、audit 任务可恢复、可追溯、可审批。
- 支持跨团队、跨组织、跨供应商的 agent 协作。

### 9.2 技术效果

- 减少协议适配器和胶水代码。
- 提升 agent 调度的类型安全。
- 提升长时任务成功率。
- 降低并行 worker 资源冲突。
- 统一 MCP inbound/outbound 工具治理。
- 让 A2A、ACP、AG-UI、OpenCode、OpenAI API 共享同一任务状态和记忆底座。
- 通过 event sourcing 和 checkpoint 支持故障恢复。

### 9.3 治理效果

- 每个任务有完整 lineage。
- 每次工具调用可审计。
- 每个高危动作可审批。
- 每个 artifact 可追踪来源。
- 每个 agent 的成本、质量、延迟可量化。
- 记忆可版本化、可隔离、可遗忘。

## 10. 分阶段建设路线

### Phase 1：AgentPool 生产化增强

- 补齐 A2A CLI 和配置一等支持。
- 统一 AG-UI agent 的 MCP provider 接入。
- 明确并统一 wrapped agents 的 pool-level/team-level MCP 可见性。
- 建立跨协议 message/resource/artifact 模型。
- 建立协议兼容测试矩阵。

### Phase 2：AgentHub Kernel

- 引入 `TaskEnvelope`。
- 引入 `AgentUnit`。
- 引入 Capability Graph。
- 引入 Policy Engine。
- 引入统一 Artifact Manager。
- 将 research loop、coding loop、review loop 做成 task templates。

### Phase 3：Durable Runtime

- 引入 event sourcing。
- 引入 checkpoint/replay。
- 接入 Temporal/Prefect/DBOS 类后端。
- 支持 pause/resume/cancel/retry。
- 支持 long-running human approval。

### Phase 4：Swarm Runtime

- 支持 worker archetype。
- 支持 dynamic spawn。
- 支持 worktree/container sandbox。
- 支持 task board、inbox、broadcast。
- 支持 merge coordinator。
- 支持 swarm evaluator。

### Phase 5：Memory Compute

- 建立 MemoryCell。
- 建立 Evidence Graph。
- 支持 memory compaction。
- 支持 task/agent/org 多级记忆。
- 用运行数据优化 agent routing。
- 建立长期自治任务能力。

## 11. 结论

AgentPool 解决的是多协议、多 agent 的统一接入和编排问题。AgentHub进一步解决生产级智能体应用的长期运行问题：如何让 agent 集群具备可靠状态、自治拓扑、可治理工具、持续记忆、人工反馈和跨协议互操作能力。

- 统一抽象。
- 融合协议生态。
- 支撑长时任务。
- 组织自治蜂群。
- 能把状态和记忆变成可计算资产。
- 能满足企业安全、审计、成本和可靠性要求。

AgentHub 的目标就是在 AgentPool 的基础上，把多 agent 编排从“协议桥接层”推进到“智能体操作系统层”。
