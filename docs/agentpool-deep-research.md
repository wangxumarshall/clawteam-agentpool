# AgentPool：多协议异构Agent编排
> 作者：wangxu
## 摘要

AgentPool核心价值不是“再实现agent 框架”，而是把不同来源、不同运行时、不同协议的 agent 统一到一个可配置、可编排、可暴露的 orchestration hub 中。基本定位概括为：

> 用 YAML 定义 heterogeneous agents，内部统一成 `MessageNode`，外部通过 ACP、MCP、A2A、AG-UI、OpenCode、OpenAI-compatible API 等协议暴露，并支持 PydanticAI、Claude Code、Codex、OpenCode、ACP 外部 agent 等多种 agent 协作。

## 1. 问题背景

AI agent 生态正在从对话快速走向多agent、多工具、多协议、多界面协作。对于真实工程团队来说，常见需求已经不再是“调用一个模型回答问题”，而是：

- 在 IDE 中让 coding agent 读取、编辑、测试代码。
- 在 Web UI 中流式展示 agent 推理、工具调用和人工反馈。
- 让 agent 使用 MCP server 暴露的外部工具、资源和 prompt。
- 让 Claude Code、Codex、OpenCode、Goose 等专业 coding agent 参与同一个任务。
- 让 PydanticAI 这类 Python-native typed agent 承担结构化编排、路由、验证和报告生成。
- 让外部 agent 系统通过 A2A 或 OpenAI-compatible API 接入。
- 记录长时任务的多轮状态、历史、成本、工具调用和人工反馈。

如果没有统一编排层，每一种 agent 与每一种客户端之间都需要单独适配。例如：

- Claude Code 有自己的 SDK、权限模式、MCP 接入方式。
- Codex 有自己的 client、MCP 配置生命周期和 reasoning 参数。
- OpenCode 有 TUI/Desktop、HTTP API、message parts、session、permission、MCP 管理等语义。
- ACP 面向 IDE 与 agent 之间的 JSON-RPC 双向交互。
- AG-UI 面向前端 UI 的流式事件协议。
- MCP 面向工具、资源和 prompt 的标准化暴露。
- A2A 面向跨 agent 系统互操作。

AgentPool 的目标就是避免这些系统两两写胶水代码，而是把它们纳入一个统一的 typed pool。

## 2. 核心挑战

### 2.1 协议语义不一致

ACP、MCP、A2A、AG-UI、OpenCode 和 OpenAI-compatible API 并不是同一种抽象的不同 JSON 形态。它们关注的问题不同：

| 协议/接口 | 主要关注点 |
| --- | --- |
| ACP | IDE/客户端到 agent 的会话、文件系统、终端、权限确认、双向 JSON-RPC |
| MCP | 工具、资源、prompt 的标准化发现与调用 |
| A2A | agent-to-agent 互操作、agent card、任务通信 |
| AG-UI | 前端 agent UI、流式事件、人类反馈体验 |
| OpenCode | TUI/Desktop session、message parts、permission、MCP 管理和事件 |
| OpenAI API | 兼容现有 OpenAI-style 客户端、chat/completions/responses 调用 |

因此，AgentPool 不能只做字段映射，而必须在 message、tool、resource、session、event、permission 和 storage 等层面做语义桥接。

### 2.2 工具接入有两个方向

AgentPool的MCP架构里有两个方向：

- `MCPManager`：消费外部 MCP server，把外部工具、资源、prompt 引入 AgentPool。
- `ToolManagerBridge`：反向把AgentPool内部工具以 MCP server 的形式暴露出去，让 Claude Code、Codex、ACP agent 等 wrapped agents 可以调用。

这两个方向非常容易混淆：

- “AgentPool 使用外部 MCP 工具”是 inbound。
- “外部 coding agent 调用 AgentPool 内部工具”是 outbound bridge。

> `docs/mcp_architecture.md` 已经把这一点明确区分出来。

### 2.3 Native agents 与 wrapped agents 能力不对称

当前本地架构显示：

- Native PydanticAI agent 能完整使用 agent-level、pool-level、team-level 三层 MCP provider。
- Claude Code agent 通过 SDK 原生 MCP 能力和 `ToolManagerBridge` 使用内部工具。
- Codex agent 在启动时收集 MCP 配置，并通过 `ToolManagerBridge` 接入内部工具。
- ACP agent 在 session 创建时接收 MCP server 配置，并通过 `ToolManagerBridge` 接入内部工具。
- AG-UI agent 当前能使用 `ToolManager` 中的工具 schema，但 MCPManager provider 尚未完整进入 AG-UI request flow。
- Pool-level MCP provider 虽然会添加到 wrapped agents 的 `ToolManager`，但 wrapped agents 的 LLM 调用不走这个 `ToolManager`，所以实际不可见。

这说明 AgentPool 已经具备多协议集成基础，但“所有 agent 类型对 MCP 工具的统一可见性”仍是关键演进点。

### 2.4 长时任务需要状态、反馈和恢复

真实 research / coding / review 任务通常不是单轮 prompt，而是长时任务：

- 多轮调研。
- 多 agent 并行分析。
- 中间结果汇总。
- 人类反馈修订。
- 争议点复查。
- 代码或文档证据追踪。
- 任务暂停、恢复和审计。
- 成本、token、工具调用、历史记录统计。

这要求 AgentPool 不只是一个请求转发器，而要成为带状态的任务编排运行时。

## 3. AgentPool 的解决方案

### 3.1 YAML-first 配置

AgentPool 使用 `AgentsManifest` 作为根配置模型，统一定义：

- `agents`
- `file_agents`
- `teams`
- `mcp_servers`
- `storage`
- `observability`
- `responses`
- `model_variants`
- `workers`
- `jobs`
- `resources`

这让用户可以在一个 YAML 文件中描述完整多 agent 系统，而不是为每个协议和 agent 写独立启动脚本。

### 3.2 `MessageNode` 统一抽象

AgentPool 把不同处理单元统一成 `MessageNode`：

- Native PydanticAI agents。
- Claude Code agents。
- Codex agents。
- ACP agents。
- AG-UI agents。
- File agents。
- Teams。

这使得 agent 和 team 可以共享统一的处理、转发、连接、hook、stream 和 lifecycle 模型。

### 3.3 `AgentPool` 作为注册表与生命周期中心

`AgentPool` 负责：

- 读取 YAML manifest。
- 实例化 agents 和 teams。
- 管理 async lifecycle。
- 注入 shared dependencies。
- 设置连接关系。
- 管理 pool-level MCP。
- 管理 storage 和 observability。
- 对外提供统一 agent registry。

这让多 agent 系统不再是散落的对象集合，而是一个可管理的运行单元。

### 3.4 Team 表达并行和顺序协作

AgentPool 支持两类基本 team 模式：

- Sequential：`agent1 | agent2 | agent3`
- Parallel：`agent1 & agent2 & agent3`

在长时任务中，sequential 适合“研究 -> 审阅 -> 汇总”，parallel 适合“协议分析 / 代码分析 / UI 分析 / 外部资料检索”并发执行。

### 3.5 多协议 server 暴露

当前 CLI 和 server 代码显示 AgentPool 支持多种对外接口：

- `serve-acp`：面向 IDE/ACP client。
- `serve-mcp`：把 AgentPool 能力暴露为 MCP server。
- `serve-agui`：面向 AG-UI frontend。
- `serve-opencode`：面向 OpenCode TUI/Desktop。
- `serve-api`：OpenAI-compatible API。
- `A2AServer`：程序化暴露 A2A routes。

其中 A2A server 当前在代码中存在，但 CLI 主入口尚未注册 `serve-a2a` 命令，更像是程序化能力或后续 CLI 演进点。

## 4. 当前支持的 Agent 类型

| Agent 类型 | 角色定位 | 典型用途 |
| --- | --- | --- |
| PydanticAI native agent | 类型安全、结构化、Python-native 的内部 agent | 编排、验证、报告生成、工具调用 |
| Claude Code agent | Claude coding runtime 的直接集成 | 大型重构、复杂代码理解、工程操作 |
| Codex agent | Codex app/server 集成 | 高推理代码编辑、patch 生成、实现验证 |
| ACP agent | 外部 ACP provider 包装 | Goose、Claude、Codex、OpenCode 等外部 agent |
| AG-UI agent | 远程 AG-UI endpoint | 前端交互、远程 agent 服务 |
| File agent | Markdown/frontmatter 定义轻量 agent | 从 Claude/OpenCode/AgentPool 风格 agent 文件加载 |
| Team | 多 agent 组合节点 | 并行研究、顺序流水线、多角色协作 |

## 5. 长时 Research 任务设计

### 5.1 任务名称

`agentpool_ecosystem_research_loop`

### 5.2 任务目标

围绕 AgentPool、ClawTeam、PydanticAI 与多协议 agent 生态，完成一份可持续迭代的深度研究报告，覆盖：

- 问题背景。
- 问题挑战。
- AgentPool 解决方案。
- 当前实现状态。
- 协议支持矩阵。
- agent 类型协作矩阵。
- MCP inbound/outbound 工具流。
- 长时任务协作模式。
- 演进趋势和路线图。

### 5.3 参与角色

| 角色 | 类型 | 职责 |
| --- | --- | --- |
| `research_coordinator` | PydanticAI native agent | 总协调、任务拆解、结构化输出、冲突合并 |
| `protocol_architect` | PydanticAI native agent | 分析 ACP、MCP、A2A、AG-UI、OpenCode、OpenAI API 边界 |
| `claude_refactor_agent` | Claude Code agent | 深读复杂代码路径、解释架构、提出重构建议 |
| `codex_impl_agent` | Codex agent | 验证实现路径、生成配置草案、识别代码缺口 |
| `opencode_operator` | OpenCode agent/server 入口 | 模拟 TUI/Desktop session、permission、MCP 管理体验 |
| `acp_external_reviewer` | ACP agent | 代表外部 ACP provider 检查互操作 |
| `agui_feedback_agent` | AG-UI agent | 提供前端流式反馈、人类批注、UI 事件体验 |
| `a2a_peer_agent` | A2A peer | 模拟跨系统 agent-to-agent 调用 |
| `mcp_research_tools` | MCP servers | 提供 repo search、文档读取、web fetch、向量检索等工具 |

### 5.4 Round 0：任务建模

用户可以从任一入口发起任务：

- ACP IDE。
- OpenCode TUI/Desktop。
- AG-UI frontend。
- OpenAI-compatible API。
- A2A peer。
- Python API。

`research_coordinator` 首先生成研究计划：

- 研究问题列表。
- 协议覆盖清单。
- agent 类型覆盖清单。
- 本地代码证据清单。
- 外部协议资料清单。
- 输出结构。
- 验收标准。

### 5.5 Round 1：并行调研

`research_coordinator` 并行派发子任务：

- `protocol_architect` 分析协议语义和桥接边界。
- `claude_refactor_agent` 阅读 AgentPool 核心架构文件。
- `codex_impl_agent` 检查 CLI、server、config、tests 覆盖。
- `opencode_operator` 验证 OpenCode session、message parts、permission 和 MCP endpoint。
- `acp_external_reviewer` 从 ACP provider 视角评估可用性。
- `agui_feedback_agent` 从 UI streaming 和 event feedback 角度评估。
- `a2a_peer_agent` 从 agent card、route 和 task interop 角度验证。
- `mcp_research_tools` 提供外部协议文档和代码检索。

### 5.6 Round 2：证据汇总

每个 agent 返回结构化 finding：

```yaml
finding:
  area: protocol | agent_type | tool_bridge | storage | workflow | risk
  evidence:
    local_files: []
    docs: []
    commands: []
  current_state: ""
  gap: ""
  recommendation: ""
```

`research_coordinator` 合并为第一版研究报告，并写入 storage/history。

### 5.7 Round 3：人类反馈

用户可以从不同协议入口提供反馈：

- ACP IDE：对代码路径、架构判断、文件证据做批注。
- OpenCode TUI：选择继续深入某个文件、协议或失败点。
- AG-UI：在前端界面调整章节顺序、重要性、结论可信度。
- A2A：外部 agent 系统提交反驳意见或补充分析。
- MCP：注入新文档、新仓库、新资料。
- OpenAI API：外部系统提交 review comment。

### 5.8 Round 4：二次研究与冲突解决

`research_coordinator` 根据反馈重派任务：

- 协议争议交给 `protocol_architect`。
- 代码实现争议交给 `claude_refactor_agent` 和 `codex_impl_agent` 双重验证。
- UI/event 争议交给 `agui_feedback_agent` 和 `opencode_operator`。
- 外部互操作争议交给 `acp_external_reviewer` 和 `a2a_peer_agent`。

冲突解决时要求每个结论带证据来源，并区分：

- 本地代码事实。
- 本地文档事实。
- 外部协议文档事实。
- 推断或建议。

### 5.9 Round 5：最终输出

最终输出包括：

- 深度研究报告。
- 协议支持矩阵。
- agent 类型协作矩阵。
- MCP inbound/outbound 工具流图。
- 当前短板。
- 演进路线图。
- 可执行 YAML 配置样例。
- 验收测试清单。

## 6. 协议串联矩阵

| 协议/接口 | 在长时任务中的作用 |
| --- | --- |
| ACP | IDE/客户端入口；支持会话、文件、终端、权限确认；也可包装外部 ACP agent |
| MCP | 工具、资源、prompt 标准化层；外部工具进入 AgentPool，内部工具通过 bridge 给 wrapped agents |
| A2A | 把 pool 中 agent 暴露给其他 agent 系统，实现跨系统 agent-to-agent 调用 |
| AG-UI | 前端流式交互、人工反馈、可视化 research loop |
| OpenCode | TUI/Desktop 操作入口，承载 session、message parts、permission、MCP 管理 |
| OpenAI API | 兼容 OpenAI-style 客户端和已有业务系统 |
| Python API | 程序化组合、测试、嵌入式 orchestration |

## 7. 协作配置草案

下面是一个面向该长时 research 任务的概念性 YAML 草案，用于表达协作关系。实际字段需要以当前 schema 为准细化。

```yaml
name: agentpool_ecosystem_research

mcp_servers:
  - name: repo_tools
    command: uvx
    args: ["mcp-server-filesystem", "."]
  - name: docs_search
    command: uvx
    args: ["mcp-server-search"]

agents:
  research_coordinator:
    type: native
    model: openai:gpt-5-mini
    description: Coordinates long-running multi-agent research.
    tools:
      - type: subagent
      - type: resource_access
    system_prompt: |
      You coordinate a multi-round research task. Delegate work, require evidence,
      merge findings, ask for follow-up review, and produce structured reports.

  protocol_architect:
    type: native
    model: openai:gpt-5
    description: Analyzes ACP, MCP, A2A, AG-UI, OpenCode and OpenAI API boundaries.
    tools:
      - type: resource_access

  claude_refactor_agent:
    type: claude_code
    description: Reads complex code paths and explains engineering architecture.
    toolsets:
      - type: code
      - type: file_edit

  codex_impl_agent:
    type: codex
    model: gpt-5.1-codex-max
    reasoning_effort: medium
    description: Verifies implementation paths and identifies missing coverage.
    toolsets:
      - type: code
      - type: file_edit

  acp_external_reviewer:
    type: acp
    provider: goose
    description: External ACP reviewer for interoperability checks.

  agui_feedback_agent:
    type: agui
    endpoint: http://localhost:8002/agent/run
    description: Remote AG-UI feedback and frontend event reviewer.

teams:
  parallel_research_panel:
    mode: parallel
    members:
      - protocol_architect
      - claude_refactor_agent
      - codex_impl_agent
      - acp_external_reviewer
      - agui_feedback_agent

  report_pipeline:
    mode: sequential
    members:
      - parallel_research_panel
      - research_coordinator
```

## 8. 当前状态

从本地仓库看，AgentPool 已经具备较完整的多协议和多 agent 骨架：

- `AgentsManifest` 已统一 native、AG-UI、Claude Code、Codex、ACP 等 agent config。
- CLI 已注册 `serve-acp`、`serve-agui`、`serve-mcp`、`serve-api`、`serve-opencode`。
- A2A server 已存在，能够为 pool 中每个 agent 暴露独立 route 和 agent card endpoint。
- OpenCode server 已实现 session、message、permission、question、MCP 等大量兼容接口。
- ACP server/client 双角色文档已经明确：AgentPool 既可作为 ACP server 接入 IDE，也可作为 ACP client 包装外部 agent。
- MCP 架构已经区分 inbound MCPManager 和 outbound ToolManagerBridge。
- Storage、observability、history、teams、triggers、workers、jobs 等长时任务能力已经具备基础。

同时也存在清晰的演进空间：

- A2A 缺少 CLI 一等入口。
- AG-UI agent 的 MCP provider 接入尚不完整。
- Pool-level 和 team-level MCP 对 wrapped agents 的可见性不一致。
- Wrapped agents 的动态 MCP 能力差异明显：Claude Code 较强，Codex 和 ACP 更偏 session/startup 级。
- 跨协议 message mention/resource 的统一模型仍在 proposal 阶段。

## 9. 演进趋势

AgentPool 后续演进重点不应只是“增加更多 agent 类型”，而是把异构 agent 协作稳定成长期运行的工程系统。

### 9.1 协议层趋势

- MCP 会继续成为工具、资源、prompt 发现与调用的底座。
- ACP 和 OpenCode 会强化 coding / IDE / TUI 操作体验。
- AG-UI 会强化前端流式可视化、人类反馈、任务控制面板。
- A2A 会承担跨框架、跨组织、跨 runtime 的 agent-to-agent 互联。
- OpenAI-compatible API 会继续作为低迁移成本的系统集成接口。

### 9.2 运行时趋势

- PydanticAI 适合作为 typed orchestration runtime，负责结构化输入输出、验证和路由。
- Claude Code、Codex、OpenCode 更适合作为 specialized coding workers。
- MCP server registry、semantic discovery、tool permission 和 audit 将成为长时任务关键基础设施。
- Storage/history 将从“记录对话”升级为“任务恢复、证据链、成本分析、质量回放”的基础。

### 9.3 AgentPool 路线建议

短期：

- 增加 `serve-a2a` CLI。
- 明确 AG-UI agent 的 MCP provider 接入策略。
- 给 wrapped agents 的 pool-level MCP 可见性补齐一致行为或文档化限制。

中期：

- 建立统一 mention/resource/message model，覆盖 ACP、OpenCode、MCP resource、AG-UI events。
- 把 research loop、coding loop、review loop 做成可复用 task template。
- 增强 long-running task pause/resume、checkpoint、human feedback、conflict resolution。

长期：

- 将 AgentPool 定位为 multi-protocol agent operating layer。
- 支持跨 workspace、跨 remote execution environment、跨 agent vendor 的稳定协作。
- 建立协议兼容测试矩阵，持续验证 ACP/MCP/A2A/AG-UI/OpenCode 行为一致性。

## 10. 结论

AgentPool 的核心竞争力在于把 agent 生态中的分裂问题向上抽象成统一编排问题：

- 用 YAML 降低配置复杂度。
- 用 `MessageNode` 统一 agent 和 team。
- 用 `AgentPool` 管理 lifecycle、storage、MCP、依赖和连接。
- 用多协议 server 暴露同一套 agent 能力。
- 用 MCP bridge 连接内部工具和外部 coding agent。
- 用 team、storage、events 和 task 机制支撑长时协作。

在多 agent 生态继续分裂的趋势下，AgentPool 更像是一个 agent interoperability layer，而不是单纯的 agent framework。它的下一步关键是把现有多协议覆盖进一步收敛为一致、可测试、可恢复、可审计的长时任务运行时。

## 附录 A：Agent 协议详解与案例

> 注：用户问题中的 `mcoo` 结合本文上下文按 `MCP`（Model Context Protocol）理解。若后续要覆盖另一个名为 `MCOO` 的协议，应单独补充来源和边界。

### A.1 协议分层总览

ACP、MCP、A2A、AG-UI、OpenCode 和 OpenAI-compatible API 不是同一层协议。它们可以在同一个 AgentPool 系统里同时出现，但各自承担不同职责：

| 协议/接口 | 核心问题 | 典型参与方 | AgentPool 中的映射 | 不适合承担的职责 |
| --- | --- | --- | --- | --- |
| ACP | 让 IDE/编辑器与 coding agent 双向协作 | IDE client、agent subprocess | `serve-acp`、`ACPAgent`、ACP event converter | 通用工具注册、跨组织 agent discovery |
| MCP | 让 AI 应用发现并调用外部工具、资源、prompt | MCP host/client/server | `MCPManager`、`ToolManager`、`ToolManagerBridge` | 完整 agent-to-agent 任务生命周期、前端 UI |
| A2A | 让一个 agent 系统发现并调用另一个 agent 系统 | A2A client、remote agent、agent card | `A2AServer`、per-agent route、agent card route | IDE 文件/终端权限、低层工具 schema 复用 |
| AG-UI | 让前端实时渲染 agent 运行过程和状态 | Web frontend、agent backend | `serve-agui`、`AGUIServer`、`BaseAgentAGUIAdapter` | agent discovery、工具进程生命周期 |
| OpenCode | 让 OpenCode TUI/Desktop/Web 通过 HTTP 操作 session/message/tool/permission | OpenCode client、OpenCode-compatible server | `serve-opencode`、session/message/permission/MCP routes | 跨厂商标准化 agent discovery |
| OpenAI-compatible API | 让现有 OpenAI-style 客户端低迁移接入 | HTTP API client、业务系统 | `serve-api` | IDE 级文件/终端交互、丰富 UI event |

一个可运行的长时 research 例子通常是这样的：用户在 OpenCode TUI 或 ACP IDE 发起任务；AgentPool 把请求落到 `research_coordinator`；native agent 通过 MCP 调用 `repo_tools` 和 `docs_search`；需要专业代码修改时委派 Claude Code/Codex/ACP wrapped agent，并通过 `ToolManagerBridge` 让外部 agent 反向调用 AgentPool 内部工具；Web 管理台通过 AG-UI 显示 `RunStarted`、文本流、工具调用和状态 delta；外部组织的 agent 则通过 A2A agent card 发现并调用某个 AgentPool agent。

### A.2 ACP：IDE/客户端到 coding agent 的双向协议

ACP（Agent Client Protocol）面向“编辑器/IDE 作为 client、coding agent 作为 agent”的场景。官方协议使用 JSON-RPC 2.0，包含 request/response method 和 notification 两类消息。典型流程是：

1. `initialize`：client 和 agent 协商版本、能力。
2. `session/new` 或 `session/load`：创建或恢复会话。
3. `session/prompt`：用户发起一轮 prompt。
4. `session/update`：agent 持续把文本、工具调用、计划、命令更新等通知发回 client。
5. `fs/read_text_file`、`fs/write_text_file`、`terminal/create`、`session/request_permission`：agent 需要读写文件、运行终端或执行敏感操作时，请 client 承担环境访问和用户授权。

这意味着 ACP 的中心不是“工具市场”，而是“coding workbench”。IDE 掌握工作区、文件系统、终端和用户确认；agent 负责规划、推理和提出操作请求。

**案例：Zed 通过 ACP 使用 AgentPool。**

用户在 Zed 中配置：

```json
{
  "agent_servers": {
    "AgentPool": {
      "command": "agentpool",
      "args": ["serve-acp", "agents.yml"]
    }
  }
}
```

一次“重构 `mcp_architecture.md` 并修正引用”的任务流可以拆成：

1. Zed 作为 ACP client 启动 `agentpool serve-acp agents.yml`，AgentPool 成为 ACP server。
2. Zed 发送 `initialize` 和 `session/new`，AgentPool 创建内部 session，并选择默认 agent 或 team。
3. 用户发送 `session/prompt`，AgentPool 把 prompt 转成内部 `ChatMessage`，交给 `research_coordinator` 或 `claude_refactor_agent`。
4. Native agent 流式产生 `PartDeltaEvent`、`ToolCallStartEvent`、`ToolCallProgressEvent`；ACP server 的 event converter 把这些事件转换为 `session/update`。
5. 如果 agent 要读取某个绝对路径文件，ACP client 侧可以通过 `fs/read_text_file` 提供内容；如果要写文件或运行命令，IDE 可以先显示 `session/request_permission`，用户批准后再执行。
6. 当任务完成，`session/prompt` 返回 stop reason，IDE 中的对话和工具调用记录保持可追踪。

AgentPool 同时也能作为 ACP client 包装外部 ACP agent。例如 manifest 中的 `type: acp` agent 可以启动 Goose、OpenCode、Claude Code 或 Codex 的 ACP subprocess。此时 AgentPool 对上仍是一个统一 `MessageNode`，对下通过 ACP 与外部进程通信：

```text
Zed
  -> ACP
AgentPool serve-acp
  -> MessageNode / Team
AgentPool ACPAgent
  -> ACP
External coding agent subprocess
```

关键收益是：IDE 只看到一个 AgentPool server，但 AgentPool 内部可以混合 native PydanticAI agent、ACP wrapped agent 和 teams。关键边界是：ACP 适合 coding 环境交互，不负责外部工具生态的标准化发现；工具发现与调用应交给 MCP，跨系统 agent discovery 应交给 A2A。

### A.3 MCP：工具、资源和 prompt 的上下文协议

MCP（Model Context Protocol）面向“AI 应用连接外部系统”。官方架构是 host-client-server：

- Host 是 Claude、ChatGPT、IDE、AgentPool 这类承载 AI 工作流的应用。
- Client 由 host 创建，每个 client 通常维护一个到某个 MCP server 的 stateful session。
- Server 暴露专业能力，主要包括 resources、tools、prompts，也可以支持 sampling、elicitation 等扩展能力。

MCP 的语义重点是能力和上下文，而不是“另一个 agent 与我对话”。一个 MCP server 可以暴露 `search_docs(query)` 工具、`repo://README.md` resource、`review-template` prompt；host 决定是否把这些能力交给模型、何时调用、是否需要用户授权。

AgentPool 的 MCP 架构必须区分两个方向：

| 方向 | 组件 | 含义 | 例子 |
| --- | --- | --- | --- |
| Inbound | `MCPManager` + resource provider | AgentPool 消费外部 MCP server | `docs_search`、filesystem、Sentry、Context7 |
| Outbound | `ToolManagerBridge` | AgentPool 把内部工具反向暴露成 MCP server | Claude Code/Codex/ACP agent 调用 AgentPool 的 `subagent`、read、grep、worker tools |

**案例 1：native agent 使用外部 MCP 文档搜索。**

```yaml
mcp_servers:
  - name: docs_search
    command: uvx
    args: ["mcp-server-search"]

agents:
  protocol_architect:
    type: native
    model: openai:gpt-5
    tools:
      - type: resource_access
```

运行时，`MCPManager` 负责启动 `docs_search` server 并创建 provider；native `Agent` 会把 aggregating provider 加入自己的 `ToolManager`。当模型需要查 AG-UI 或 A2A 文档时，它看到的是工具 schema 和 resource 列表，不需要知道该工具背后是本地进程、远程 HTTP server 还是聚合 provider。

**案例 2：Claude Code/Codex 反向调用 AgentPool 内部工具。**

Claude Code、Codex、ACP wrapped agent 的 LLM 调用并不走 AgentPool native `ToolManager`。如果希望这些外部 runtime 使用 AgentPool 内部工具，AgentPool 需要启动 `ToolManagerBridge`，把内部工具暴露成一个 MCP server，再把这个 MCP server 的配置传给外部 agent：

```text
External coding agent
  -> MCP tool call
ToolManagerBridge HTTP MCP server
  -> AgentPool ToolManager
  -> internal tools / subagent / worker / file_edit
```

这就是 `docs/mcp_architecture.md` 中强调的 inverse 关系：`MCPManager` 是“把外部 MCP 拉进来”，`ToolManagerBridge` 是“把 AgentPool 内部工具推出去”。

MCP 的常见误区是把它当成完整 agent 协议。MCP server 通常不应该看到整个对话，也不应该直接承担多轮 task lifecycle。它适合暴露边界清晰的能力：查文档、读资源、执行受控 action、提供 prompt template。长时任务状态、agent routing 和跨 agent 任务关系应留在 AgentPool、A2A 或应用层。

### A.4 A2A：跨 agent 系统的发现、任务和互操作

A2A（Agent2Agent Protocol）面向“一个 agent 或 agent 系统调用另一个远程 agent”。它的核心概念包括：

- Agent Card：远程 agent 发布的 JSON 元数据，描述身份、能力、skills、endpoint 和认证要求。
- Message：一次通信 turn，包含 `role` 和一个或多个 `Part`。
- Task：A2A 管理的有状态工作单元，可经历 submitted、working、input-required、completed、failed 等生命周期。
- Artifact：agent 产出的文档、结构化数据或文件等结果。
- Streaming / Push Notifications：长时任务可以通过 SSE 或 webhook 推送状态和 artifact 更新。

官方 REST binding 中，常见操作包括 `POST /message:send`、`POST /message:stream`、`GET /tasks/{id}`、`POST /tasks/{id}:cancel`、`POST /tasks/{id}:subscribe`。Agent Card 的标准发现位置是 `/.well-known/agent-card.json`。

**案例：外部研究 agent 调用 AgentPool 的 `protocol_architect`。**

假设一个外部企业 agent 正在做“内部 agent 协议选型”，它不想知道 AgentPool 的 YAML、team、MCP 细节，只想发现一个能分析协议边界的远程 agent：

1. 外部 agent 读取 AgentPool 暴露的 agent card，发现 `protocol_architect` 支持 `protocol-analysis` skill，接受 `text/plain` 输入，返回 markdown 或结构化 artifact。
2. 外部 agent 发送 message：“比较 ACP、MCP、A2A 和 AG-UI 在 IDE 插件中的职责边界”。
3. AgentPool A2A server 把请求路由到 `protocol_architect` 对应的 `MessageNode`。
4. `protocol_architect` 内部可以继续调用 MCP 文档搜索、委派 Codex/Claude worker，或者访问 storage。
5. A2A response 返回一个 Task；外部 agent 可通过 streaming 或 task status 获取中间状态和最终 artifact。

AgentPool 当前的 `A2AServer` 采用 per-agent route：`/{agent_name}`、`/{agent_name}/.well-known/agent-card.json` 和 `/{agent_name}/docs`。这和单域名单 agent 的标准 well-known 模式略有不同，但适合在一个 pool 中并列暴露多个 agent。需要注意的是，当前 CLI 主入口尚未注册 `serve-a2a`，因此 A2A 更像是程序化 server 能力，后续可以补 CLI 一等入口。

A2A 与 MCP 的边界尤其重要：A2A 发现和调用“会处理任务的远程 agent”；MCP 发现和调用“可被 agent 使用的工具、资源、prompt”。如果外部系统要让 AgentPool 的 agent 产出完整分析报告，用 A2A；如果只是让 AgentPool 的 agent 查 Sentry issue 或搜索文档，用 MCP。

### A.5 AG-UI：面向前端的流式事件协议

AG-UI（Agent User Interaction Protocol）面向“前端如何实时呈现 agent”。它不是工具协议，也不是 IDE 文件系统协议，而是把 agent run 拆成前端可消费的事件流。官方事件模型包括：

- Lifecycle：`RunStarted`、`RunFinished`、`RunError`、`StepStarted`、`StepFinished`。
- Text message：`TextMessageStart`、`TextMessageContent`、`TextMessageEnd`，用于逐 token 或逐 chunk 渲染回复。
- Tool call：`ToolCallStart`、`ToolCallArgs`、`ToolCallEnd`、`ToolCallResult`，用于透明展示 agent 正在调用什么工具、参数是什么、结果是什么。
- State：`StateSnapshot`、`StateDelta`，用 snapshot + JSON Patch 同步前端状态。
- Activity / Reasoning / Custom：用于进度、可视化 activity、推理摘要或应用自定义 UI。

**案例：Web research dashboard 展示长时任务。**

用户打开一个 React/Vue 前端，选择 `research_coordinator` 并提交“研究 AgentPool 多协议架构”。流程可以是：

1. 前端向 `POST http://localhost:8002/research_coordinator` 发送 AG-UI request。
2. `AGUIServer` 通过 `BaseAgentAGUIAdapter.dispatch_request()` 调用对应 AgentPool agent。
3. 前端收到 `RunStarted` 后创建运行面板。
4. agent 生成报告时，前端按 `TextMessageContent.delta` 增量渲染正文。
5. agent 调用 `docs_search` 或 `repo_tools` 时，前端显示 `ToolCallStart`、参数 delta 和最终结果。
6. coordinator 更新计划时，前端用 `StateDelta` 更新 checklist、章节状态、置信度或证据列表。
7. 结束时收到 `RunFinished`；如果失败则收到 `RunError` 并保留可重试上下文。

AgentPool 的 `serve-agui` 把 pool 中每个 agent 暴露成独立 POST route，适合构建统一的多 agent 控制台。AG-UI 的优势是 UI 体验：用户能看到运行生命周期、工具调用、状态变化和人工反馈入口。它不应该直接承担 MCP server 管理、文件写入权限或外部 agent discovery；这些应在后端通过 MCP、ACP、OpenCode 或 A2A 完成。

### A.6 OpenCode：TUI/Desktop/Web 的 session/message/permission HTTP 接口

OpenCode 在这里更接近“应用级 agent 操作面”和 HTTP API，而不是像 ACP/MCP/A2A 那样的通用协议标准。官方 OpenCode server 会暴露 OpenAPI 3.1 endpoint，TUI 是连接 server 的 client；server API 覆盖 global event、project/path/VCS、session、message、command、file、tool、MCP、agent、TUI control、auth 等。

AgentPool 的 `serve-opencode` 目标是实现 OpenCode-compatible server，使 OpenCode TUI/Desktop 或兼容客户端可以把 AgentPool 当成后端。它比 ACP 更偏“完整应用状态”：

- Session：创建、列出、fork、abort、share、summarize、revert、diff。
- Message parts：用户输入可以包含 text、file、agent、subtask；assistant 输出可以包含 text、reasoning、tool、step、diff 等 parts。
- Permission / Question：通过 HTTP endpoint 和 SSE event 管理待确认操作与人工输入。
- MCP：支持查看、添加、连接、断开、认证 MCP server。
- Events：通过 SSE 广播 session/message/part/status 更新。
- TUI control：可通过 `/tui/*` endpoint 驱动 prompt、session selector、toast、command 等 UI 行为。

**案例：用 OpenCode TUI 操作 AgentPool research loop。**

1. 启动 `agentpool serve-opencode agents.yml --port 4096`。
2. OpenCode client 读取 `/global/health` 和 `/global/event`，建立 server 状态和 SSE 事件流。
3. 用户创建 session：`POST /session`。
4. 用户发送消息：`POST /session/{id}/message`，parts 中可以包含文本、文件引用、目标 agent 或 subtask。
5. AgentPool 根据 `request.agent` 选择 pool 内 agent；如果用户指定 `codex_impl_agent`，请求就路由到该 wrapped agent，否则走默认 agent。
6. `OpenCodeStreamAdapter` 把 AgentPool 的 `RichAgentStreamEvent` 转成 OpenCode `PartUpdatedEvent`、`SessionStatusEvent`、`SessionIdleEvent`、tool part 状态等 SSE event。
7. 如果 agent 要写文件或运行命令，server 创建 permission request；OpenCode UI 调用 `/permission/{requestID}/reply` 返回批准或拒绝。
8. 用户可以 fork session 验证另一路方案，或者用 `/session/{id}/diff` 查看本轮任务造成的文件差异。

OpenCode 与 ACP 都适合 coding agent，但切入点不同：ACP 是 IDE 与 agent subprocess 的 JSON-RPC 工作台协议；OpenCode 是面向 TUI/Desktop/Web 的 HTTP session API，强调可持久化 session、message parts、事件流、权限和 UI 控制。

### A.7 OpenAI-compatible API：低迁移成本的业务接入口

OpenAI-compatible API 不是专门的 agent 协议，而是为了让已有 OpenAI-style SDK、网关、评测系统和业务服务可以低成本调用 AgentPool。它通常以 `/v1/chat/completions`、streaming SSE、model name、messages、tools/function calling 等概念为入口。

**案例：CI 系统调用 AgentPool 做 PR 总结。**

CI bot 不需要 IDE、AG-UI 或 OpenCode 的完整交互能力，只需要向一个 HTTP endpoint 发起 chat completion：

```json
{
  "model": "research_coordinator",
  "messages": [
    {"role": "user", "content": "Summarize this PR and list risks."}
  ],
  "stream": true
}
```

AgentPool 可以把 `model` 映射到某个 agent 或 team，把 messages 转成内部 `ChatMessage`，再把输出转回 OpenAI-style chunks。这个入口适合自动化系统、已有 SDK、评测平台和简单业务集成。它的代价是语义较扁平：文件权限、终端、agent card、rich UI state、session fork/revert 等能力都需要额外约定或降级处理。

### A.8 协议选择规则

实际落地时可以按问题选择协议，而不是按“哪个更先进”选择：

- 用户在 IDE 里让 agent 修改代码、读写文件、运行命令：优先 ACP。
- agent 需要调用数据库、搜索、Sentry、Figma、文档库、内部 API：优先 MCP。
- 一个外部 agent 系统需要发现并调用 AgentPool 中的某个 agent：优先 A2A。
- Web 前端需要漂亮地显示 agent 流式过程、工具调用、计划和人工反馈：优先 AG-UI。
- 需要复用 OpenCode TUI/Desktop 的 session、message part、permission、MCP 管理体验：优先 OpenCode-compatible server。
- 已有业务服务只会调 OpenAI-style SDK：优先 OpenAI-compatible API。

AgentPool 的价值正在于这些协议可以组合：ACP/OpenCode/AG-UI/OpenAI API 是入口，A2A 是跨系统 agent 入口，MCP 是工具和资源底座，`MessageNode` 和 `AgentPool` 则把它们收敛成统一的内部编排模型。

## 参考依据

- `README.md`
- `docs/mcp_architecture.md`
- `docs/advanced/acp-integration.md`
- `docs/feature_timeline.toml`
- `src/agentpool/models/manifest.py`
- `src/agentpool_config/nodes.py`
- `src/agentpool_cli/__main__.py`
- `src/agentpool_server/a2a_server/server.py`
- `src/agentpool_server/opencode_server/ENDPOINTS.md`
- `src/agentpool_server/opencode_server/OPENCODE_UI_TOOLS_COMPLETE.md`
- PydanticAI 官方文档：https://ai.pydantic.dev/
- Agent Client Protocol 官方文档：https://agentclientprotocol.com/protocol/overview
- Model Context Protocol 官方文档：https://modelcontextprotocol.io/docs/getting-started/intro
- Model Context Protocol 架构规范：https://modelcontextprotocol.io/specification/2025-06-18/architecture
- Agent2Agent Protocol 规范：https://a2a-protocol.org/dev/specification/
- AG-UI Protocol 事件文档：https://docs.ag-ui.com/concepts/events
- OpenCode Server 文档：https://opencode.ai/docs/server/
- OpenCode MCP servers 文档：https://opencode.ai/docs/mcp-servers/
- OpenCode ACP 支持文档：https://opencode.ai/docs/acp/
- OpenAI API 参考：https://developers.openai.com/api/reference/
