# DeerFlow 阅读与研究指南

> 本文档面向想要深入理解 DeerFlow 项目的开发者 / 研究者，提供一条从"跑起来"到"读懂架构"再到"动手改造"的学习路径，并对核心技术与工程机制逐一做解析。它是对 `AGENTS.md`（项目纵览）与 `backend/AGENTS.md`、`frontend/AGENTS.md`（模块深度）的**阅读向导**：先读本指南建立全局认知，再按索引进入对应模块深入。

---

## 目录

1. [项目定位与全景](#1-项目定位与全景)
2. [服务拓扑与整体架构](#2-服务拓扑与整体架构)
3. [仓库地图](#3-仓库地图)
4. [学习路径（分阶段）](#4-学习路径分阶段)
5. [核心技术点解析](#5-核心技术点解析)
6. [端到端数据流走读](#6-端到端数据流走读)
7. [配置体系详解](#7-配置体系详解)
8. [测试体系](#8-测试体系)
9. [开发工作流与命令速查](#9-开发工作流与命令速查)
10. [研究专题建议](#10-研究专题建议)
11. [文档索引](#11-文档索引)

---

## 1. 项目定位与全景

**DeerFlow 是什么？** 一个基于 **LangGraph** 的 AI 超级代理（Super-Agent）系统，采用全栈架构：

- **后端**运行一个"超级代理"（Lead Agent），具备沙箱化代码执行、持久记忆、子代理委派、以及可扩展的工具集（内置工具 / MCP 工具 / 社区技能）。
- **前端**是一个 Next.js 的聊天界面，支持流式响应、Artifact 文件、Todo 列表、目标（Goal）等丰富的交互。
- **外部 IM 平台**（飞书、Slack、Telegram、Discord、钉钉、微信、企微、Buzz 等）通过 Gateway 桥接到同一个代理运行时。

一句话概括：**"一个 LangGraph 图 + 一层运行时（记忆/检查点/沙箱/子代理）+ 多入口（Web / HTTP API / IM / 嵌入式客户端 / 终端 TUI）"**。

### 核心特性一览

| 特性 | 说明 | 关键位置 |
|---|---|---|
| 超级代理 | 带完整中间件链的 LangGraph 代理 | `backend/packages/harness/deerflow/agents/` |
| 工具系统 | 内置工具 + 分组 + 授权过滤 | `backend/packages/harness/deerflow/tools/` |
| MCP 支持 | 外部 MCP 服务器接入、路由提示、自动提升 | `backend/packages/harness/deerflow/mcp/` |
| 技能（Skills） | 公共/自定义技能、延迟发现、安全扫描 | `backend/packages/harness/deerflow/skills/`、`skills/` |
| 持久记忆 | 事实记忆 + 提取/注入/过期淘汰 | `backend/packages/harness/deerflow/agents/memory/` |
| 检查点 | full / delta 两种存储模式 | `backend/packages/harness/deerflow/runtime/checkpoint_*.py` |
| 沙箱 | 隔离的代码执行环境 | `backend/packages/harness/deerflow/sandbox/` |
| 子代理 | 委派式子代理执行、进度事件 | `backend/packages/harness/deerflow/subagents/` |
| 追踪 | LangSmith / Langfuse / Monocle 三套可观测性 | `backend/packages/harness/deerflow/tracing/` |
| IM 通道 | 飞书/钉钉/Slack/Telegram/Discord/微信/企微/Buzz/GitHub | `backend/app/channels/` |
| 调度任务 | 定时后台任务（非交互式） | `backend/packages/harness/deerflow/scheduler/`、`backend/app/scheduler/` |
| 嵌入式客户端 | 无 HTTP 服务的进程内调用 `DeerFlowClient` | `backend/packages/harness/deerflow/client.py` |
| 终端 TUI | 基于 Textual 的终端界面 | `backend/packages/harness/deerflow/tui/` |

---

## 2. 服务拓扑与整体架构

一个 `make dev` / Docker 栈运行四个协作服务：

| 服务 | 端口 | 角色 |
|---|---|---|
| **Nginx** | `2026` | 统一反向代理入口——浏览器直接访问这个端口 |
| **Gateway API** | `8001` | FastAPI REST API + 内嵌的 LangGraph 兼容代理运行时 |
| **Frontend** | `3000` | Next.js Web 界面 |
| **Provisioner** | `8002` | 可选——仅当沙箱配置为 provisioner/K8s 模式时启用 |

**Nginx 是唯一对外入口**，路由规则：

```
/                     → Frontend (3000)           # 页面
/api/langgraph/*      → Gateway (8001)            # 重写为 Gateway 原生 /api/* 路由（LangGraph 兼容层）
/api/*                → Gateway (8001)            # 其余 API
```

默认仅绑定回环地址 `127.0.0.1:2026`（`BIND_HOST`/`PORT` 环境变量控制），这是刻意为之——8001 不对外暴露。

### 分层架构（后端）

```
┌─────────────────────────── app/  (FastAPI Gateway)  ───────────────────────────┐
│  gateway/  REST 路由 · 认证/授权 · 中间件 · services 编排                        │
│  channels/ IM 通道（feishu/slack/telegram/discord/dingtalk/wechat/wecom/buzz） │
│  scheduler/ 定时任务服务                                                        │
└──────────────────────────────┬─────────────────────────────────────────────────┘
                               │
┌────────────────────────── deerflow-harness 包（import: deerflow.*）───────────┐
│  agents/   Lead Agent 工厂 + 中间件链 + 线程状态 + 记忆                          │
│  runtime/  运行引擎（worker、检查点、流桥、store、事件、上下文压缩）              │
│  models/   模型工厂与多 Provider 适配                                            │
│  tools/  mcp/  skills/  subagents/  sandbox/  authz/  guardrails/              │
│  persistence/  ORM 模型 + alembic 迁移（混合引导）                               │
│  config/  配置加载与解析                                                          │
│  tracing/  可观测性   tui/ 终端界面   client.py 嵌入式客户端                      │
└──────────────────────────────┬─────────────────────────────────────────────────┘
                               │
              deerflow-extension-api 包（import: deerflow_extension_api.*）
              对外公开的扩展契约（工具/中间件/沙箱提供者的稳定 API 层）
```

**关键分界**：`deerflow-harness` 是**纯代理框架**（不依赖 FastAPI），可以被 Gateway、嵌入式客户端、TUI 三种方式复用；`app/` 是 HTTP 与 IM 的具体外壳。

---

## 3. 仓库地图

```
deer-flow/
├── Makefile                        # 根编排：驱动整套栈（dev/start/stop/docker/setup）
├── config.example.yaml             # 主配置模板 → 复制为 config.yaml（已 gitignore）
├── extensions_config.example.json  # MCP 服务器 + 技能配置模板 → extensions_config.json
├── AGENTS.md                       # 仓库总览（本指南的兄弟文档，读本指南后应通读它）
├── backend/                        # Python 后端
│   ├── Makefile                    # 后端命令（dev/gateway/test/lint/migrate-rev）
│   ├── packages/extension-api/     # deerflow-extension-api（公开扩展契约）
│   ├── packages/harness/           # deerflow-harness（import: deerflow.*，代理框架）
│   │   └── deerflow/
│   │       ├── agents/             # Lead Agent 工厂、中间件链、线程状态、记忆
│   │       ├── runtime/            # worker、检查点、流桥、事件、store、上下文压缩
│   │       ├── models/             # 模型工厂与 Provider
│   │       ├── tools/  mcp/  skills/  subagents/  sandbox/  authz/  guardrails/
│   │       ├── persistence/        # ORM + alembic 迁移
│   │       ├── config/  tracing/  tui/  uploads/  scheduler/  community/
│   │       ├── client.py           # 嵌入式客户端
│   │       └── checkpoint_patches.py  # 检查点机制补丁
│   └── app/                        # FastAPI Gateway + IM 通道
│       ├── gateway/                # routers/ auth/ services.py 等
│       └── channels/               # 各 IM 平台适配
├── frontend/                       # Next.js 前端（pnpm）
│   └── src/
│       ├── app/                    # App Router 路由（workspace/chats、agents、blog、docs）
│       ├── core/                   # 业务逻辑心脏（threads/api/agents/artifacts/mcp/skills…）
│       ├── components/             # ui/(shadcn)、ai-elements/、workspace/、landing/
│       ├── hooks/  lib/  content/  styles/  typings/
├── docker/                         # docker-compose、nginx 配置、provisioner
├── skills/                         # 代理技能：public/（已提交）、custom/（gitignore）
├── contracts/                      # 跨组件 JSON 契约（如子代理状态、技能审查）
├── scripts/                        # 根编排脚本（check/configure/doctor/serve/nginx…）
├── tests/                          # 根级测试
└── docs/                           # 跨领域文档、计划、设计笔记
```

> 提示：`config.yaml` 与 `extensions_config.json` 在仓库根目录，但已被 gitignore，可通过 Gateway API 运行时修改。第三方扩展（`plugins:` 列表）只在 `config.yaml` 中维护。

---

## 4. 学习路径（分阶段）

> 每阶段标注了"读完能做什么"与"必读文件"，建议按顺序推进。

### 阶段 0：环境准备与启动（0.5 天）

**目标**：让系统在本地跑起来，找到"入口"。

1. 通读根目录 `README.md`、`Install.md`。
2. 复制配置并启动：
   ```bash
   make setup        # 交互式安装向导（推荐新用户）
   make doctor       # 检查配置与环境
   make dev          # 热重载启动整套栈（Gateway + Frontend + Nginx）
   ```
3. 浏览器打开 `http://localhost:2026`，发起第一轮对话。
4. 熟悉"改配置→重启生效"的基本节奏：`config.example.yaml` → `config.yaml`。

**产出**：能跑通一次对话，知道四个服务各自的端口与角色。

### 阶段 1：端到端走读一次对话（1~2 天）

**目标**：跟随一条消息从浏览器到模型再回流，建立数据流全景。

1. 读第 [6 节](#6-端到端数据流走读) 的完整走读。
2. 前端：从 `frontend/src/core/threads/hooks.ts`（`useThreadStream` / `useSubmitThread`）入手，看消息如何通过 LangGraph SDK 发起。
3. 后端入口：`backend/app/gateway/routers/thread_runs.py` → `backend/app/gateway/services.py` → `backend/packages/harness/deerflow/runtime/runs/worker.py`。
4. 图执行：`backend/packages/harness/deerflow/agents/lead_agent/agent.py`（`_make_lead_agent`）。
5. 流式回传：`backend/packages/harness/deerflow/runtime/stream_bridge/`、`runtime/stream_modes.py`。

**产出**：能画出"前端 → Nginx → Gateway → 图 → 模型 → 流回前端"的完整链路图。

### 阶段 2：后端核心机制（3~5 天）

**目标**：理解超级代理的内部构造。按依赖顺序阅读第 [5 节](#5-核心技术点解析) 中：

1. **模型工厂**（5.4）：`models/factory.py` —— 一切 Agent 的地基。
2. **代理与中间件链**（5.1、5.2）：`agents/lead_agent/agent.py` + `agents/middlewares/`。
3. **工具系统**（5.3）：`tools/` + `agents/features.py`。
4. **线程状态与检查点**（5.9）：`agents/thread_state.py` + `runtime/checkpoint_state.py`。
5. **持久化与迁移**（5.10）：`persistence/`。

每读完一块，建议做一个小实验（改一个工具、加一个中间件、跑一个测试）。

**产出**：理解一个 Agent 是如何从"配置 + 模型 + 工具 + 中间件"装配出来的。

### 阶段 3：进阶机制（3~5 天）

依次深入第 [5 节](#5-核心技术点解析) 的剩余主题：

1. **子代理**（5.5）+ **技能**（5.7）：理解"委派"与"可扩展性"。
2. **MCP**（5.6）：外部工具接入协议。
3. **记忆系统**（5.8）：事实的提取、注入、淘汰。
4. **沙箱**（5.11）：代码执行的安全边界。
5. **检查点 delta 模式**（5.9 进阶）：理解存储优化的权衡。
6. **IM 通道**（5.13）+ **调度器**（5.16）：理解多入口与后台任务。

**产出**：能回答"DeerFlow 如何保证安全、如何扩展能力、如何持久化状态"。

### 阶段 4：前端深度（2~3 天）

1. 通读 `frontend/AGENTS.md`，重点看 **Data Flow** 一节。
2. 核心：`src/core/threads/hooks.ts`、`src/core/api/`、`src/core/streamdown/`（流式 Markdown 渲染）。
3. 组件：`src/components/workspace/`（消息列表、Artifact 面板、Sidecar、浏览器视图）。
4. 路由：`src/app/workspace/chats/[thread_id]/page.tsx`（聊天主页面）。

**产出**：理解前端如何消费 SSE 流、如何在服务端组件与客户端组件间划分边界。

### 阶段 5：工程实践（持续）

1. **测试体系**：读第 [8 节](#8-测试体系)，跑一遍 `make test` 与 `pnpm test`。
2. **基准测试**：`backend/scripts/benchmark/checkpoint/`（检查点通道基准）。
3. **贡献流程**：`CONTRIBUTING.md`、`RELEASING.md`；遵循 TDD 与 `ruff format`。
4. **研究专题**：按第 [10 节](#10-研究专题建议) 选题深入。

---

## 5. 核心技术点解析

> 每一小节给出：**是什么 → 核心文件 → 关键概念 → 阅读建议**。

### 5.1 超级代理（Lead Agent）

**是什么**：整个系统的心脏——一个装配好的 LangGraph 代理图，被 Gateway、嵌入式客户端、TUI 三方复用。

**核心文件**：
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py` —— `make_lead_agent()` / `_make_lead_agent()` 装配入口（已源码核验）
- `backend/packages/harness/deerflow/agents/factory.py` —— `create_deerflow_agent()` SDK 级纯参数工厂（已源码核验）
- `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` —— 提示词模板
- `backend/packages/harness/deerflow/agents/thread_state.py` —— `ThreadState` / `DeltaThreadState` 状态 schema（已源码核验）
- `backend/packages/harness/deerflow/runtime/runs/worker.py` —— `run_agent()` 后台执行入口（已源码核验）

**关键概念**：
- **状态（State）**：`ThreadState(AgentState)` 定义了图内流转的所有 channel：`sandbox`、`thread_data`、`title`、`artifacts`、`todos`、`goal`、`uploaded_files`、`viewed_images`、`promoted`、`delegations`、`skill_context`、`summary_text`（后略，`messages` 继承自 `AgentState`）。除 `messages` 外共有 **9 个带 reducer 的字段**（`THREAD_STATE_REDUCER_FIELDS`）：`sandbox`（幂等合并，冲突即抛 `ValueError`）、`artifacts`（有序去重合并）、`todos`（最后非 None 值获胜）、`goal`、`viewed_images`（空 dict 表示清除）、`promoted`（按 catalog_hash 整体替换/并集）、`delegations`（委派台账，终态不可被非终态覆盖，最多 50 条）、`skill_context`（按 path 去重、保最近读取、最多 8 条）。
- **装配（Assembly）**：`create_agent`（来自 langchain）把模型、工具、中间件、状态 schema 组装成一个可 `astream` 的图。检查点模式（full/delta）在编译前被冻结（`freeze_checkpoint_channel_mode`）；delta 模式下 `thread_state.py` 通过 `delta_messages_field()` 把 `messages` 字段替换为 `DeltaChannel(merge_message_writes, snapshot_frequency=...)`（快照频率默认 10，非默认值会动态构造 `DeltaThreadState_f{n}` schema）。
- **`messages` 的 delta 折叠器**：`merge_message_writes`（thread_state.py）是 LangGraph 私有 `_messages_delta_reducer` 的线性时间等价实现，保留公共 reducer 的完整 coercion / ID / removal / `REMOVE_ALL_MESSAGES` 语义。
- **运行解析（`_make_lead_agent`）**：从 `config.configurable` + `context` 合并出运行时配置（`_get_runtime_config`），解析顺序为"请求 > 自定义 agent 配置 > 全局默认"（`_resolve_runtime_option`，区分"未传"与"传了假值"）；模型名经 `_resolve_model_name` + `_authorize_model_name`（授权 provider 的 `model:use` 检查，拒绝时优雅回退到第一个允许的模型）双重解析。
- **追踪回调挂在图根**：`build_tracing_callbacks()` 追加到 `config["callbacks"]`，保证一次运行 = 一条 trace。图内任何 `create_chat_model(...)` 必须传 `attach_tracing=False`，否则产生重复 span（agent.py 模块注释 INVARIANT 列出 5 个现场：bootstrap agent、default agent、summarization middleware、TitleMiddleware 异步路径、skill security scanner）。
- **webhook 通道安全门**：`channel_name == "github"` 时不给 agent 装配 `update_agent` 工具（外部评论者可触发运行，不能让其自改 agent 配置）。
- **非交互模式**：`non_interactive=True` 时从工具集剔除 `ask_clarification`。

**阅读建议**：按 `agent.py` 装配顺序读：模型解析 → 状态 schema → 工具装配（`get_available_tools`）→ 中间件链（`build_middlewares`）→ 提示词 → 图编译；再对照 `thread_state.py` 的 reducer 定义。

### 5.2 中间件链（Middleware）

**是什么**：LangGraph `AgentMiddleware` 机制，用于在 agent 生命周期各阶段注入横切逻辑，是 DeerFlow 最核心的扩展点之一。

**核心文件**：`backend/packages/harness/deerflow/agents/middlewares/`（41 个文件），典型代表：
- `title_middleware.py` —— 自动生成对话标题
- `memory_middleware.py` —— 记忆写入（`after_agent` 排队）
- `todo_middleware.py` / `plan_mode` —— TodoList 计划模式（`write_todos` 工具）
- `summarization_middleware.py` —— 上下文自动压缩
- `view_image_middleware.py` —— 视觉模型图片注入
- `token_usage_middleware.py` —— token 用量统计
- `subagent_limit_middleware.py`、`loop_detection_middleware.py`、`token_budget_middleware.py` —— 运行防护（后两者暴露 `consume_stop_reason`）
- `clarification_middleware.py` —— `ask_clarification` 澄清工具（非交互模式下被移除）
- `configured_extensions.py` —— 从 `extensions_config.json` 加载用户自定义中间件
- `dynamic_context_middleware.py` —— 往首个 HumanMessage 注入 `<system-reminder>`（日期/记忆）
- `skill_activation_middleware.py`、`skill_tool_policy_middleware.py` —— `/skill-name` 激活与技能工具策略
- `system_message_coalescing_middleware.py` —— 合并所有 SystemMessage 到首位（vLLM/Qwen/Anthropic 等严格后端要求）
- `terminal_response_middleware.py`、`model_length_finish_reason_middleware.py`、`safety_finish_reason_middleware.py` —— 终端响应兜底与 stop_reason 标记

**关键概念**：
- **lead 中间件完整装配顺序（`build_middlewares`，已源码核验）**：
  1. `build_lead_runtime_middlewares`（沙箱基础设施：ThreadData → Uploads → Sandbox + DanglingToolCall + ToolErrorHandling 等）
  2. `DynamicContextMiddleware`（system-reminder 注入）
  3. `SkillActivationMiddleware` + `SkillToolPolicyMiddleware`（技能激活与工具策略）
  4. `DurableContextMiddleware`（压缩前捕获委派台账/技能上下文）
  5. `DeerFlowSummarizationMiddleware`（启用时）
  6. `TodoMiddleware`（`is_plan_mode` 时）
  7. `TokenUsageMiddleware`（`token_usage.enabled` 时）
  8. `TitleMiddleware`
  9. `MemoryMiddleware`（tool 模式下后端需要被动写入时 / 非 tool 模式时）
  10. `ViewImageMiddleware`（模型 `supports_vision` 时）
  11. `McpRoutingMiddleware` + `DeferredToolFilterMiddleware`（tool_search 延迟工具）
  12. `SystemMessageCoalescingMiddleware`
  13. `SubagentLimitMiddleware`（`subagent_enabled` 时，max_concurrent 默认 3）
  14. `LoopDetectionMiddleware` / `TokenBudgetMiddleware`（对应配置启用时）
  15. 自定义中间件 + 配置扩展中间件
  16. `TerminalResponseMiddleware` → `ModelLengthFinishReasonMiddleware` → `SafetyFinishReasonMiddleware`（可选）
  17. `ClarificationMiddleware`（**必须永远最后**）
- 中间件可以拦截 `before_model` / `after_model` / `before_agent` / `after_agent` 等节点，甚至可以贡献自己的 `state_schema` 字段（如 `viewed_images`、`summary_text`）。
- 中间件 `state_schema` 需要被"规范化"（`normalize_middleware_state_schemas`，thread_state.py）：delta 模式下把每个中间件的 `state_schema` 的 `messages` 字段适配为 `DeltaChannel`。
- `factory.py` 的 `create_deerflow_agent()` 提供**纯参数 SDK 入口**：`middleware=` 全量接管 或 `features=`/`extra_middleware=` 声明式装配（二选一）；`extra_middleware` 通过 `@Next`/`@Prev` 锚点插入链中（`_insert_extra`），支持冲突/环形检测。

**阅读建议**：先读 `title_middleware.py`（最简单），再读 `memory_middleware.py`（最复杂）。模仿写一个自己的中间件是理解全系统最快的方式。

### 5.3 工具系统

**是什么**：代理可调用的函数集合，支持内置工具、MCP 工具、社区技能工具。

**核心文件**：
- `backend/packages/harness/deerflow/tools/tools.py` —— `get_available_tools()` 工具装配主入口（已源码核验）
- `backend/packages/harness/deerflow/tools/builtins/` —— 内置工具（11 个文件，已源码核验目录）
- `backend/packages/harness/deerflow/tools/types.py` —— `Runtime` 类型别名（`ToolRuntime[dict[str, Any], ThreadState]`，已源码核验）
- `backend/packages/harness/deerflow/agents/features.py` —— 特性开关控制工具装配
- `backend/packages/harness/deerflow/authz/tool_filter.py` —— 工具级授权过滤

**关键概念**：
- **工具定义形态**：工具本质是 `langchain.tools` 的 `@tool` 装饰函数；第一个参数是 `runtime: Runtime`（LangGraph 注入的运行时上下文，含 `state` / `context` / `config`），工具签名里可声明 `InjectedToolCallId` 注入 tool_call_id。
- **`get_available_tools()` 装配顺序（已源码核验）**：
  1. 从 `config.yaml -> tools[]` 解析（`use` 类路径 + `group` 分组），`resolve_variable` 反射加载；未配置 `group` 过滤器时全部加载，否则按 `groups` 过滤。
  2. **host bash 门**：`is_host_bash_allowed()` 为 False（LocalSandboxProvider 激活）时剔除 bash 工具。
  3. 追加内置工具：`present_file_tool`、`ask_clarification_tool`、`review_skill_package`、`list_uploaded_files`；`skill_evolution.enabled` 时追加 `skill_manage_tool`；`subagent_enabled` 时追加 `task_tool`；模型 `supports_vision` 时追加 `view_image_tool`。
  4. 追加 MCP 工具（从缓存 `get_cached_mcp_tools()`，`tag_mcp_tool` 标记来源）与 ACP 工具（`invoke_acp_agent_tool`）。
  5. **按名字去重**（config > builtin > MCP > ACP），重名警告（issue #1803）。
- **工具运行时注入**：`Runtime` 类型让工具通过 `runtime.context` 读取用户身份/agent 元数据（如记忆工具的 `_resolve_scope` 用 `runtime.context["agent_name"]` + `resolve_runtime_user_id`），并跨请求/任务边界保持持久化作用域正确。
- **sync 包装**：`_ensure_sync_invocable_tool` 给纯异步工具挂 sync wrapper，供同步调用方使用。
- **授权**：`apply_tool_authorization` 在装配时过滤（`config.yaml -> authorization`）；subagent 路径在 `_build_initial_state` 里先过滤再进 `DeferredToolCatalog`。
- 文件类工具写 Artifact（`/mnt/user-data/...`），与前端 Artifact 面板联动。

**阅读建议**：先读 `tools/tools.py`（装配顺序），再读 `tools/builtins/task_tool.py`（最复杂的工具：后台子代理 + 轮询 + 事件）、`tools/builtins/present_file_tool.py`（最直观）。

### 5.4 模型工厂（Model Factory）

**是什么**：统一创建 LLM 客户端的地方，支持多家 Provider 与推理/视觉特性。

**核心文件**：`backend/packages/harness/deerflow/models/`
- `factory.py` —— `create_chat_model()` 主入口
- `claude_provider.py`、`vllm_provider.py`、`openai_codex_provider.py`、`mindie_provider.py`
- `patched_openai.py`、`patched_deepseek.py` 等 —— 针对特定厂商的补丁

**关键概念**：
- 每个模型在 `config.yaml` 的 `models[]` 中配置：`use`（Provider 类路径）、`supports_thinking`、`supports_vision`、Provider 特有字段。
- vLLM 推理模型用 `deerflow.models.vllm_provider:VllmChatModel`，Qwen 风格解析用 `chat_template_kwargs.enable_thinking`。
- 追踪回调默认挂到模型层，但图内调用必须显式 `attach_tracing=False`（见 5.1）。
- `credential_loader.py` 处理 API 密钥的加载与解析。

### 5.5 子代理（Subagents）

**是什么**：Lead Agent 委派给专门的子代理执行子任务，实现"计划-委派-汇总"的复杂任务分解。

**核心文件**：`backend/packages/harness/deerflow/subagents/`（全部已源码核验）
- `registry.py` —— 子代理注册表与配置解析（built-in → custom_agents → per-agent overrides 三层覆盖）
- `executor.py` —— `SubagentExecutor` 执行引擎（`_aexecute` / `execute` / `execute_async`）
- `config.py` —— `SubagentConfig`（dataclass）+ `resolve_subagent_model_name`
- `task_tool.py`（tools/builtins）—— 图内的 `task` 工具，发起委派 + 后台轮询 + 事件发布
- `status_contract.py` —— 前后端共享的结构化结果契约（`additional_kwargs` 元数据）
- `step_events.py`、`token_collector.py` —— 步骤事件序列化与 token 用量收集
- `builtins/general_purpose.py`、`builtins/bash_agent.py` —— 内置子代理

**关键概念**：
- **内置类型**：`general-purpose`（max_turns=150）与 `bash`（max_turns=60，仅当 host bash 允许或有 AioSandboxProvider 时暴露）。`config.yaml -> subagents.custom_agents` 可定义自定义类型（system_prompt / tools / skills / model / timeout）。
- **配置解析顺序（`get_subagent_config`，已源码核验）**：built-in → `custom_agents` → `agents[name]` 逐项覆盖（timeout/max_turns/model/skills）；全局默认只作用于内置子代理，不覆盖自定义子代理自身的默认值。
- **委派生命周期（`task_tool`，已源码核验）**：`task` 工具被模型调用 → `SubagentExecutor.execute_async()` 在 `_background_tasks` 注册 → 提交到 `_scheduler_pool`（ThreadPoolExecutor，max_workers=3）→ `_aexecute` 在**持久隔离事件循环**（`_isolated_subagent_loop`，专用守护线程）上 `agent.astream(..., stream_mode="values")` 执行 → `task_tool` 每 5s 轮询结果 → 发 `task_started`/`task_running`/`task_completed`/`task_failed`/`task_cancelled`/`task_timed_out` 事件并返回 `Command(update={messages: [ToolMessage]})`。
- **子代理禁止递归**：子代理工具集 `subagent_enabled=False`（无 `task` 工具）+ 默认 `disallowed_tools=["task"]` + 无 `list_uploaded_files`（独立 ThreadState 无法排除当前运行文件）。
- **`SubagentResult`**：终态由 `try_set_terminal` **只设置一次**（锁保护，超时/取消/执行线程竞争时首个终态获胜）；`cancel_event` 提供协作式取消（在 astream 迭代边界检查）。
- **护栏 stop_reason（#3875 Phase 2）**：token_budget / loop_detection 硬停后运行仍正常结束，`consume_stop_reason` 记录 `token_capped` / `loop_capped` / `turn_capped`（max_turns 触顶），`status_contract.py` 用 additive 字段 `subagent_stop_reason` 传递，旧前端忽略未知字段。
- **追踪**：`langfuse_session_id` 继承父线程，`langfuse_user_id` 在 `task_tool` 时捕获，trace name 为 `subagent:<name>`（名字规范化为小写+连字符）；父运行上下文（user_role/oauth/run_id/authz_attributes）整体传播给 GuardrailMiddleware。
- **契约**：`contracts/subagent_status_contract.json` 在 Python 与 TypeScript 之间钉住枚举值；`ToolMessage.additional_kwargs` 携带 `subagent_status` / `subagent_stop_reason` / `subagent_result_brief` / `subagent_result_sha256` / `subagent_model_name` / `subagent_token_usage`。

**阅读建议**：先读 `status_contract.py`（契约），再读 `task_tool.py`（委派入口）→ `executor.py`（执行引擎）→ `registry.py`（配置解析）。结合前端 `frontend/src/core/tasks/` 看同一契约在前后端如何对齐。

### 5.6 MCP 集成

**是什么**：Model Context Protocol（模型上下文协议）接入，让代理使用外部工具服务器。

**核心文件**：`backend/packages/harness/deerflow/mcp/`
- `client.py` —— MCP 客户端封装
- `session_pool.py` —— 会话池复用
- `tools.py`、`cache.py`、`oauth.py`
- `McpRoutingMiddleware`（在 agents/middlewares 中）—— 路由提示与自动提升

**关键概念**：
- 配置在 `extensions_config.json` 的 `mcpServers` 中（name → enabled/type/command/args/env/url/headers/oauth/routing/tools/timeout）。
- `routing.mode="prefer"` 会在提示词中注入 `<mcp_routing_hints>`；配合 `tool_search.enabled` 可将命中的延迟工具自动提升（`auto_promote_top_k`，默认 3）。
- `session_init_timeout`（默认 60s）与 `tool_call_timeout` 防止挂死的 MCP 服务器阻塞代理构建与调用。
- 可通过 Gateway API 或 `DeerFlowClient.update_mcp_config()` 运行时增删 MCP 服务器。

### 5.7 技能系统（Skills）

**是什么**：可插拔的"技能包"，让代理获得领域能力（与 MCP 互补）。

**核心文件**：
- `backend/packages/harness/deerflow/skills/`（31 个文件：扫描、安全审查、类型、工具）
- `skills/public/`（已提交的公共技能）、`skills/custom/`（gitignored）
- `tools/skill_manage_tool.py` —— 图内的技能管理入口
- `skills/security_scanner.py` —— 技能内容安全扫描（`scan_skill_content`）

**关键概念**：
- 配置：`extensions_config.json` 的 `skills` 部分控制启用状态；`skills.path` / `skills.container_path` 控制宿主与容器路径。
- **延迟发现**（`skills.deferred_discovery`）：开启后 `<available_skills>` 全量元数据块替换为紧凑的 `<skill_index>`（仅名字），并注册 `describe_skill` 工具按需获取元数据——节省大量 prompt token。
- 技能需通过安全扫描才能在运行时对模型可见；扫描器同样受"图根追踪"约束（`attach_tracing=False`）。
- 技能审查：`skills/public/skill-reviewer/` 是内置的只读技能质量审查器，使用 `contracts/skill_review/` 契约。

**阅读建议**：看 `skills/public/` 里一个实际技能的目录结构，理解"技能包 = 元数据 + 提示词 + 资源"的组织方式。

### 5.8 记忆系统（Memory）

**是什么**：跨会话的持久事实记忆，让代理"记得"用户信息。

**核心文件**：`backend/packages/harness/deerflow/agents/memory/`（已源码核验）
- `manager.py` —— `MemoryManager` 契约（pydantic BaseModel + 分层方法）+ 后端发现/工厂 + 宿主 hook 提供
- `tools.py` —— tool 模式下的 4 个记忆工具（`memory_search` / `memory_add` / `memory_update` / `memory_delete`）
- `summarization_hook.py` —— 压缩前记忆快照
- `backends/` —— 可插拔后端：`deermem`（默认，含提取/注入/淘汰）、`mem0`、`openviking`、`noop`
- `agents/middlewares/memory_middleware.py` —— `MemoryMiddleware`（`after_agent` 排队写）

**关键概念**：
- **后端抽象（已源码核验）**：`MemoryManager` 是 **pydantic `BaseModel` + ABC**（`ModelMetaclass` 派生自 `ABCMeta`，未实现抽象方法会在实例化时抛 `TypeError`）。方法分三层：tier-1 `add` / `get_context`（@abstractmethod 必须实现）；tier-2 `search` / `get_memory` / `clear_memory` / `import_memory` / `shutdown_flush` 等（默认 raise 或 no-op）；tier-3 可选 hook（`warm` / `reload_memory` / fact CRUD / `on_pre_compress` 等）。
- **`supports_search` 不变量**：ClassVar 标记必须与 `search()` 是否被 override 一致（`_check_invariants` 校验），`mode="tool"` 强制要求后端实现 `search()`，否则实例化即失败——防声明与实现漂移。
- **后端发现（drop-in）**：`_scan_backends()` 扫描 `backends/<name>/__init__.py` 暴露的 `MANAGER_CLASS`；`manager_class` 配置值解析顺序为"注册名 → `pkg.mod:Cls` 点路径"，**解析失败即 raise**（记忆是持久状态，静默回退到别的后端=数据完整性地雷）。
- **工厂与宿主 hook（已源码核验）**：`get_memory_manager()` 单例（双检锁，多线程安全）；工厂通过 `from_config(..., **host_hooks)` 注入 `callbacks`（`LangfuseMemoryCallbacks`，记忆 LLM 调用前合并 langfuse 元数据）、`should_keep_hidden_message`（保留带澄清响应的 `hide_from_ui` 消息）、`trace_context_manager`、`host_llm_factory`、`extraction_callback`（提取指标日志 + 高拒绝率告警）。
- **两种模式**：
  - `mode="middleware"`：`MemoryMiddleware.after_agent` 在每次 agent 完成后把原始消息交给 `manager.add()`（后端自行过滤用户输入+最终 AI 响应、检测纠正/强化、去抖入队、异步 LLM 提取）。
  - `mode="tool"`：注册 `memory_search` / `memory_add` / `memory_update` / `memory_delete` 四个工具让模型主动读写；大多后端此模式下**省略 MemoryMiddleware**，仅 `requires_passive_writes_in_tool_mode=True` 的后端（如 Deermem）保留被动写入。
- **作用域**：记忆按 `(agent_name, user_id)` 分桶；`agent_name=None` = 全局记忆。工具通过 `runtime.context["agent_name"]` + `resolve_runtime_user_id(runtime)` 解析作用域，优先于 ContextVar 回退。
- **配置**：`config.yaml -> memory`：`enabled`、`manager_class`、`mode`、`storage_path`（默认 `{runtime_home}/users/{user_id}/memory.json`，相对路径基于 runtime_home 解析）、`backend_config` 等。
- **优雅关闭**：`shutdown_flush(timeout)` 在 Gateway 关闭路径（IM 通道与调度器停止后）有界排空去抖缓冲区，必须尊重硬超时以对齐 K8s `terminationGracePeriodSeconds`。

**阅读建议**：先读 `manager.py`（契约+工厂），再看 `backends/deermem/`（默认后端：提取、置信度过滤、stale 淘汰），最后对照 `tools.py` 与 `memory_middleware.py` 理解两种模式的差异。

### 5.9 检查点与状态持久化（Checkpoint）

**是什么**：LangGraph checkpointer 机制的深度定制——线程状态如何被存储、恢复、回滚。

**核心文件**：
- `backend/packages/harness/deerflow/runtime/checkpoint_mode.py` —— 模式冻结、delta 标记注入、兼容性门
- `runtime/checkpoint_state.py` —— `CheckpointStateAccessor`（状态访问唯一咽喉）、`build_state_mutation_graph`、`RollbackPoint`
- `runtime/runs/worker.py` —— 运行 worker（回滚、线性化恢复）
- `agents/thread_state.py` —— `ThreadState` / `DeltaThreadState`
- `checkpoint_patches.py` —— 检查点机制补丁（delta 历史折叠、稳定消息 ID 等）
- `runtime/checkpoint_cache/`、`runtime/checkpointer/cached_saver.py` —— delta 历史缓存

**关键概念（重要，值得细读）**：
- **两种通道模式**：`full`（默认，每个检查点存完整 messages 列表，O(N²) 增长）vs `delta`（用 LangGraph 1.2 `DeltaChannel`，只存增量写，O(N) 增长）。由 `config.yaml -> checkpoint_channel_mode` 决定，**进程冻结、改配置需重启**。
- **delta 快照节奏**：`checkpoint_delta.snapshot_frequency`（默认 10），同样进程冻结。
- **兼容性不对称**：delta 模式进程可以透明读取 full 检查点（平滑迁移路径）；full 进程打开 delta 线程会抛 `CheckpointModeMismatchError`（HTTP 409），**fail-closed**。
- **所有线程状态访问必须经过 `CheckpointStateAccessor`**——它是绑定 graph + checkpointer + mode 的唯一咽喉，注入模式标记、运行兼容性检查、物化 delta 状态。
- **delta 模式不能 fork**：resume/regenerate 时由 worker 线性化（`_linearize_delta_checkpoint_resume`）——把请求检查点的完整状态用 `Overwrite` 写到当前 head 上，而不是 fork 出兄弟分支（避免共享父节点 pending_writes 重放污染）。
- **回滚（Rollback）**：取消运行时可回滚到运行前快照（`RollbackPoint`），full 模式 fork，delta 模式替换 head。

**为什么值得研究**：这是全项目技术难度最高、设计文档最密集的区域，体现了对 LangGraph 上游机制的深入理解与 hack。想研究"框架深度改造"的必读。

### 5.10 持久化与 Schema 迁移

**是什么**：应用表（`runs`、`threads_meta`、`feedback`、`users`、`run_events`、`channel_*`）由 alembic 管理；LangGraph 检查点表（`checkpoints` 等）在同一数据库但归 LangGraph 所有，通过 `_env_filters.py::include_object` 排除出 alembic 视野。

**核心文件**：`backend/packages/harness/deerflow/persistence/`
- `engine.py` —— `init_engine`
- `bootstrap.py` —— `bootstrap_schema`（三分支混合引导）
- `migrations/` —— alembic 环境与版本
- `migrations/_helpers.py` —— `safe_add_column` / `safe_drop_column`（幂等迁移助手）

**关键概念**：
- **混合引导（Hybrid bootstrap）**：空库 → `create_all` + `alembic stamp head`；遗留库 → `create_all`（仅基线表回填）+ stamp 0001 + upgrade；已版本化 → `alembic upgrade head`。
- **约定**：任何 ORM 模型改动必须产出一个 alembic revision（`cd backend && make migrate-rev MSG="..."`），Gateway 启动时自动 `upgrade head`。
- **并发安全**：Postgres 用 `pg_advisory_lock`；SQLite 用进程内 asyncio.Lock + 文件写锁 + `busy_timeout`（多实例建议 Postgres）。
- 迁移助手幂等，重复执行是 no-op。

### 5.11 沙箱（Sandbox）

**是什么**：隔离的代码执行环境，代理生成的代码在受限环境中运行。

**核心文件**：`backend/packages/harness/deerflow/sandbox/`（16 个文件）
- 抽象：`SandboxProvider`（acquire/release）、`Sandbox`
- 实现：本地（Docker）、远程（AIO/Provisioner/K8s）等
- 与文件上传、Artifact 路径（`/mnt/user-data/...`）联动

**关键概念**：
- `config.yaml -> sandbox.use` 指定 Provider 类路径。
- 沙箱环境中的工具（`sandbox` 相关工具）与宿主工具在授权与路径策略上不同。
- 挂载模式（`sandbox.thread_data_mounts: true`）下上传路径跳过沙箱获取与同步。

### 5.12 追踪与可观测性（Tracing）

**是什么**：三套可观测性集成——LangSmith、Langfuse、Monocle。

**核心文件**：`backend/packages/harness/deerflow/tracing/`
- `factory.py` —— `build_tracing_callbacks()` 返回回调列表
- `metadata.py` —— Langfuse trace 元数据构建
- `monocle.py` —— Monocle OTel 探针
- `backend/packages/harness/deerflow/trace_context.py` —— 请求级 trace 上下文（`X-Trace-Id`）

**关键概念**：
- **回调挂在图根**，保证一次运行一条 trace（所有节点/LLM/工具调用为子 span）。
- **Langfuse 字段映射**：`langfuse_session_id` ← thread_id、`langfuse_user_id` ← 用户、`langfuse_trace_name` ← assistant_id。
- **Monocle 与 Langfuse 不同**：它是进程级 OTel 全局探针（非回调），只在 Gateway lifespan 初始化（`import deerflow.agents` 绝不应触发追踪），默认关闭；与 Langfuse 共存已验证。
- 请求级关联：`logging.enhance.enabled` 开启时，Gateway 为每个 HTTP 请求生成/继承 `X-Trace-Id`，写入日志 `trace_id`、响应头与 Langfuse `deerflow_trace_id`。

### 5.13 IM 通道（Channels）

**是什么**：外部 IM 平台桥接——同一代理通过不同适配器对外提供服务。

**核心文件**：`backend/app/channels/`
- `base.py` —— 通道抽象基类
- `feishu.py`、`slack.py`、`telegram.py`、`discord.py`、`dingtalk.py`、`wechat.py`、`wecom.py`、`buzz.py`、`github.py`
- `manager.py`、`service.py`、`message_bus.py`、`run_policy.py`（各平台的运行策略）
- `commands.py` —— 通道内斜杠命令

**关键概念**：
- 每个平台一个适配器，实现统一接口（接收消息 → 创建线程 → 运行 → 回发）。
- 文件存储按用户隔离：`users/{user_id}/threads/{thread_id}/user-data/uploads`，IM 通道通过显式 `user_id=` kwarg 指定属主。
- 通道连接可配置（`channel_connections`），Web UI 有 IM 连接管理。

### 5.14 嵌入式客户端与 TUI

**是什么**：不启动任何 HTTP 服务的进程内使用方式。

**核心文件**：
- `backend/packages/harness/deerflow/client.py` —— `DeerFlowClient`
- `backend/packages/harness/deerflow/tui/` —— 终端 TUI（Textual）
- `backend/docs/TUI.md` —— TUI 完整指南

**关键概念**：
- `DeerFlowClient` 复用与 Gateway 相同的 `deerflow` 模块与配置文件，无 FastAPI 依赖；所有返回类型与 Gateway API 响应 schema 对齐，因此消费代码在 HTTP/嵌入式两种模式下可无缝切换。
- `chat()` 同步返回完整文本；`stream()` 订阅 `stream_mode=["values","messages","custom"]` 产出 `StreamEvent`。
- **自定义事件不变量**：生产事件必须用 `emit_custom_event` / `aemit_custom_event`，且 payload 必须带非空 `type`；`StreamWriter` 优先、回调分发尽力而为。
- TUI：`deerflow` console script 启动；纯逻辑层与 Textual UI 层分离，纯层可直接单测；TUI 会话通过 `ThreadMetaWriter` 写入 `threads_meta`，从而出现在 Web UI 侧边栏。
- **Gateway Conformance Tests** 保证 client 输出与 Gateway Pydantic 模型不漂移。

### 5.15 网关 REST API 与认证

**核心文件**：`backend/app/gateway/`
- `routers/`（24 个路由）：threads、thread_runs、runs、models、agents、mcp、skills、memory、uploads、artifacts、feedback、channels、browser、scheduled_tasks、integrations、suggestions、input_polish、github_webhooks 等
- `auth/`（认证）、`auth_middleware.py`、`authz.py`、`csrf_middleware.py`
- `services.py` —— 服务编排（构建访问器等）
- `trace_middleware.py`、`internal_auth.py`（调度器等内部调用者认证）

**关键概念**：
- 认证支持多策略（cookie/HttpOnly access_token + csrf_token），Web 登录流程在 `(auth)/`。
- `services.py` 构建 `CheckpointStateAccessor` 传给线程读取路径。
- `internal_auth` 只对内部调用者（调度器启动路径）放行 `context.non_interactive`；客户端传入的 `non_interactive` 被丢弃。
- 分页、路径安全（`path_utils.py`）、GitHub webhook（带去重）等横切逻辑都在网关层。

### 5.16 调度任务（Scheduler）

**是什么**：定时后台运行代理任务（如每日报告），非交互式。

**核心文件**：
- `backend/packages/harness/deerflow/scheduler/`
- `backend/app/scheduler/`
- `backend/app/gateway/routers/scheduled_tasks.py`
- 持久化：`persistence/scheduled_tasks/`、`persistence/scheduled_task_runs/`

**关键概念**：
- 由 `config.yaml -> scheduler.enabled` 门控；Web UI 页面在 `/workspace/scheduled-tasks`。
- 关键安全设计：**后台运行是非交互的**——lead-agent 工具集在 `context.non_interactive=true` 时排除 `ask_clarification`；该 key 只对内部认证调用者（调度启动路径）有效。
- `uq_scheduled_task_run_active` 部分唯一索引保证每任务最多一条活跃运行。

### 5.17 前端流式渲染（Streamdown）

**是什么**：前端流式 Markdown 渲染引擎——流式场景下的最大技术难点。

**核心文件**：
- `frontend/src/core/streamdown/` —— 增量词动画（`animated` / `isAnimating`）、`streamdownRenderingPlugins`
- `frontend/src/core/threads/hooks.ts` —— `useThreadStream` / `useSubmitThread`（TanStack Query + LangGraph SDK）
- `frontend/src/core/api/api-client.ts` —— SSE 重连与 gap 恢复（`stream_replay_gap` 自定义事件，最多 5 次恢复重连）

**关键概念**：
- 主线程流用 LangGraph SDK 的 `throttle: true` 合并同一宏任务的更新。
- **SSE 回放 gap 处理**：无 id 的 `gap` 控制帧清除旧重连元数据、触发内部 `stream_replay_gap` 事件、重载持久线程值并重新加入流。
- `core/messages/utils.ts` 维护 `assistant:processing` 分组：工具调用中的可见文本作为处理步骤渲染。
- **交互归属（Interaction Ownership）**：`page.tsx` 拥有 composer busy 状态、分支提交、编辑重跑、Goal 显示等；`MessageList` 拥有 human-input 卡片门控；边界划分非常严格（frontend/AGENTS.md 有专门一节）。

### 5.18 配置系统

见第 [7 节](#7-配置体系详解)。

---

## 6. 端到端数据流走读（业务层视角）

以 Web 界面发起一条消息为例。**注意：DeerFlow 业务层不定义 "planner/executor" 等显式节点**——图内是 LangGraph `create_agent` 的标准"模型 ⇄ 工具"ReAct 循环；项目的业务逻辑全部体现在 **worker 流程**与 **中间件链**上，所以本节的焦点是这两层。

### 6.1 一次请求的完整旅程

```
① 用户在 /workspace/chats/[thread_id] 输入消息
   └─ core/input-polish 可先改写草稿；core/voice-input 可语音转写
② useSubmitThread (core/threads/hooks.ts) 调用 LangGraph SDK
   └─ 请求 /api/langgraph/threads/{id}/runs/stream （经 nginx）
③ Nginx 将 /api/langgraph/* 重写为 Gateway 原生 /api/*
④ Gateway 层：认证/授权/CSRF 中间件 → routers/thread_runs.py 创建 RunRecord
   └─ services.py 构建 CheckpointStateAccessor + 图
⑤ runtime/runs/worker.py::run_agent —— 业务核心（详见 6.2）
⑥ 图内循环：模型节点 ⇄ 工具节点，业务逻辑由中间件钩子挂接（详见 6.3）
⑦ 流式回传：stream_modes.py 将 values/messages-tuple/custom 事件
   └─ SSE 推回前端（含 gap 控制帧、task_* 子代理事件）
⑧ 前端 useThreadStream 消费流
   ├─ core/streamdown 增量渲染 Markdown（词动画、代码高亮、Mermaid）
   ├─ Artifact 面板刷新（write_file 工具产物）
   ├─ 子任务卡片更新（core/tasks）
   └─ token 用量、运行时长、Goal 状态更新
⑨ 运行结束：检查点落库（checkpoints 表）+ 元数据写库（runs 表）
   └─ 标题自动生成（TitleMiddleware）落库
```

### 6.2 worker 流程（`runtime/runs/worker.py`，已源码核验）

`run_agent` 是每次请求必经的业务编排，按顺序：

| 步骤 | 动作 | 说明 |
|---|---|---|
| 1 | `wait_for_finalizing` → `try_start` | 等待前一个运行收尾、登记本运行 |
| 2 | `build_checkpointer` | 冻结检查点模式（full/delta），进程内不可变 |
| 3 | `_capture_rollback_point` | 捕获回滚点（取消时可回滚到运行前状态） |
| 4 | 构建 `CheckpointStateAccessor` | 线程状态访问唯一咽喉，注入模式标记 |
| 5 | 组装流配置 | `stream_mode`、追踪回调（LangSmith/Langfuse/Monocle） |
| 6 | 追加 `run_sentinel` | 区分"预生成标题"的运行与真实运行 |
| 7 | `agent.astream(payload, ...)` | 启动图执行（见 6.3），同时 `StreamBridge` 把事件转 SSE 推回 |
| 8 | `try_finalize` | 落库运行状态；`_resolve_run_error` 统一处理异常 |

### 6.3 图内循环的业务挂钩点（中间件链，已源码核验）

图本身是 `模型节点 ⇄ 工具节点` 的循环，直到模型不再请求工具。DeerFlow 的业务逻辑挂接在 LangGraph 的中间件钩子上（装配顺序见 5.2）：

| 钩子时机 | 主要中间件 | 业务作用 |
|---|---|---|
| 进图前 `before_agent` | `DynamicContextMiddleware` | 注入 `<memory>` 长期记忆块 + `<current_date>`（会话首轮一次） |
| 每次模型调用前 `before_model` | `DurableContextMiddleware`、`SkillToolPolicy`、`McpRouting`、`Summarization`、`SystemMessageCoalescing` 等 | 注入压缩摘要/委派台账/技能上下文、技能工具策略、MCP 路由提示、上下文压缩、系统消息合并 |
| 模型调用后 `after_model` | `DurableContext`(委派捕获)、`SubagentLimit`、`LoopDetection`、`TokenBudget`、`Clarification` 等 | 记录委派台账、子代理并发限额、死循环检测、token 预算、澄清工具 |
| 每次 agent 结束 `after_agent` | `MemoryMiddleware`、`TitleMiddleware`、`TokenUsage` | 记忆排队写入、自动标题、token 统计 |
| 工具节点内 | `ToolErrorHandling`、`DanglingToolCall`、沙箱/上传/线程数据中间件 | 工具报错兜底、悬挂工具调用清理、工作目录/上传文件注入 |

> **如何观察真实轨迹**：开启 Langfuse/LangSmith 看一条 trace（一次运行 = 一条 trace，节点/LLM/工具为子 span），或在测试里断言 `stream_mode="updates"` 的事件序列。框架内部的节点名（`agent`/`tools`）在 SSE 流中可见，但对理解业务无增量价值。

**IM 通道等价路径**：飞书消息 → `channels/feishu.py` → 同样创建线程/运行 → 结果回发飞书。**嵌入式路径**：`DeerFlowClient.stream()` 直接在进程内走 ⑤⑥ 的图执行部分。

---

## 7. 配置体系详解

### 7.1 配置文件与解析顺序

| 文件 | 用途 | 运行时可改？ |
|---|---|---|
| `config.yaml`（← `config.example.yaml`） | 主配置：模型、工具、沙箱、技能路径、记忆、子代理、调度、追踪等 | 部分字段运行时热加载，部分（如 `checkpoint_channel_mode`、`logging.enhance`）**需重启** |
| `extensions_config.json`（← `extensions_config.example.json`） | MCP 服务器 + 技能启用状态 + 中间件 | 可通过 Gateway API / `DeerFlowClient` 修改（原子替换写） |
| `plugins:`（在 `config.yaml`） | 第三方扩展导入列表 | 仅 operator 可控（刻意不放进 API 可写的文件） |

解析逻辑在 `backend/packages/harness/deerflow/config/`（44 个文件）：`app_config.py`、`agents_config.py`、`memory_config.py`、`subagents_config.py`、`tracing_config.py` 等。

### 7.2 config.yaml 关键字段速查

| 字段 | 作用 |
|---|---|
| `models[]` | LLM 配置（`use` 类路径、`supports_thinking`、`supports_vision`） |
| `tools[]` / `tool_groups[]` | 工具与分组 |
| `sandbox.use` | 沙箱 Provider 类路径 |
| `skills.path` / `skills.container_path` / `skills.deferred_discovery` | 技能路径与延迟发现 |
| `subagents.enabled` | 子代理总开关 |
| `memory` | 记忆系统全部参数（含 `staleness_*` 淘汰策略） |
| `summarization` | 上下文压缩（触发类型：tokens/messages/fraction） |
| `title` | 自动标题（`max_words`/`max_chars`/`model_name`） |
| `checkpoint_channel_mode` | 检查点存储模式（`full`/`delta`，**进程冻结**） |
| `checkpoint_delta.snapshot_frequency` | delta 快照节奏（默认 10，**进程冻结**） |
| `checkpoint_graph_cache.accessor_graph_max` | 访问器图缓存上限（默认 64，热加载） |
| `scheduler.enabled` | 调度服务开关 |
| `logging.enhance` | 请求追踪关联（`X-Trace-Id`，**需重启**） |

### 7.3 环境变量

- `LANGSMITH_TRACING` / `LANGFUSE_TRACING` / `MONOCLE_TRACING`（+ `MONOCLE_EXPORTERS`）—— 追踪开关
- `DEER_FLOW_ENV` / `ENVIRONMENT` —— trace 打标签
- `NEXT_PUBLIC_LANGGRAPH_BASE_URL` / `NEXT_PUBLIC_BACKEND_BASE_URL` —— 前端后端地址（默认走 nginx）
- `BIND_HOST` / `PORT` —— Nginx 对外绑定（默认回环 `127.0.0.1:2026`）
- `DEER_FLOW_DEV_ALLOWED_ORIGINS` —— 前端开发服务器允许的额外来源
- `DEER_FLOW_RUN_LIVE_TESTS=1` —— 启用真实 API 集成测试

---

## 8. 测试体系

### 8.1 后端（TDD 强制）

- **规则**：每个新功能/修复必须带单元测试，无例外。测试在 `backend/tests/`，命名 `test_<feature>.py`。
- 运行：
  ```bash
  cd backend && make test        # 全部离线测试
  make test-live                 # 真实 API 集成测试（需 config.yaml + 凭据，显式开启）
  PYTHONPATH=. uv run pytest tests/test_<feature>.py -v
  ```
- 测试布局参考：`test_checkpoint_mode.py`、`test_delta_channel_checkpointers.py`、`test_client.py`（含 `TestGatewayConformance`）、`test_persistence_bootstrap.py` 等。
- 轻量配置/工具模块优先纯单元测试；循环导入问题在 `tests/conftest.py` 用 `sys.modules` mock 解决。

### 8.2 前端

- 单测：`frontend/tests/unit/`（Rstest，node 环境为主；`*.dom.test.*` 用 happy-dom 跑 hooks/组件）。镜像 `src/` 目录布局。
- E2E：`frontend/tests/e2e/`（Playwright + Chromium，`page.route()` mock 后端，测试真实页面交互）。
- 命令：`pnpm test` / `pnpm test:e2e` / `pnpm check`（lint + typecheck）/ `pnpm perf:check`（路由资源预算）。

### 8.3 基准测试

- `backend/scripts/benchmark/checkpoint/bench_channels.py` + `summarize_channels.py` —— full/delta 通道对比基准（独立子进程、成对运行、正确性摘要 + 性能数据）。
- `backend/scripts/benchmark/checkpoint/bench_production.py` + `summarize_production.py` —— 生产形态基准（真实 lead-agent 图 + 真实 Gateway 路由栈）。
- 基准数据默认 1 GiB 全负载上限，超大用例需显式 `--allow-large-cases`。

---

## 9. 开发工作流与命令速查

### 9.1 根 Makefile（整套栈）

```bash
make setup        # 交互式安装向导
make doctor       # 配置与系统体检
make dev          # 热重载开发（Gateway + Frontend + Nginx）
make start        # 本地生产模式
make stop         # 停止
make up / down    # Docker 生产栈（浏览器访问 localhost:2026）
make test         # 根级测试
```

### 9.2 后端（`cd backend`）

```bash
make dev          # Gateway API 热重载（8001）
make test         # 测试套件
make lint / format   # ruff
make migrate-rev MSG="add foo column"   # 生成 alembic revision
```

### 9.3 前端（`cd frontend`）

```bash
pnpm dev          # Turbopack 开发服务器（3000）
pnpm check        # lint + typecheck（提交前必跑）
pnpm test / test:e2e
pnpm build / start
```

### 9.4 代码规范

- 后端：ruff（`ruff format --check` 由 CI 强制）、行宽 240、Python 3.12+、双引号。
- 前端：导入排序（builtin → external → internal → parent → sibling）、`@/*` 路径别名、`cn()` 拼 Tailwind 类、`ui/` 与 `ai-elements/` 为自动生成勿手改。
- **文档同步政策**：用户可见变更更新 `README.md`，架构变更更新对应 `AGENTS.md`。

---

## 10. 研究专题建议

如果你想做"深入研究"，以下是高价值的切入点：

| 方向 | 入口 | 深挖内容 |
|---|---|---|
| **LangGraph 框架深度** | `runtime/checkpoint_*.py`、`checkpoint_patches.py` | delta 通道、线性化恢复、状态变更图、回滚语义——项目如何 hack 上游机制 |
| **存储与性能** | `persistence/` + `scripts/benchmark/checkpoint/` | 混合引导、幂等迁移、并发安全、full vs delta 的存储/时延权衡 |
| **Agent 可扩展性** | `agents/middlewares/` + `skills/` + `mcp/` + `subagents/` | 中间件状态 schema 贡献、延迟技能发现、MCP 路由自动提升、子代理进度协议 |
| **安全** | `authz/`、`sandbox/`、`guardrails/`、`internal_auth.py` | 工具授权、沙箱隔离、非交互模式安全门、CSRF、上传路径校验 |
| **多模型接入** | `models/` | Provider 适配模式（vllm/claude/codex/mindie）、thinking/vision 特性、厂商补丁 |
| **可观测性** | `tracing/`、`trace_context.py` | 三套追踪并存、请求级关联、回调挂载点约束 |
| **流式前端** | `frontend/src/core/streamdown/`、`api-client.ts` | SSE gap 恢复、增量渲染动画、重连策略 |
| **IM 多通道** | `app/channels/` | 通道抽象、运行策略（run_policy）、去重（dedupe_store） |
| **端到端一致性** | `contracts/`、`TestGatewayConformance` | 前后端/多入口之间的契约对齐如何被测试锁住 |

---

## 11. 文档索引

| 文档 | 位置 | 用途 |
|---|---|---|
| `AGENTS.md` | 仓库根 | 仓库总览（本指南的权威兄弟文档） |
| `backend/AGENTS.md` | 后端 | 后端深度：harness/app 分工、代理与中间件链、沙箱、MCP、技能、记忆、IM 通道、持久化、配置、测试布局 |
| `frontend/AGENTS.md` | 前端 | 前端深度：App Router 布局、线程/流式数据流、代码风格、命令 |
| `docs/README.md` | 后端 | 后端文档索引（列出全部 `backend/docs/` 文档） |
| `docs/ARCHITECTURE.md` | 后端 | 架构细节 |
| `docs/CONFIGURATION.md` | 后端 | 配置选项 |
| `docs/API.md` | 后端 | API 参考 |
| `docs/SETUP.md` | 后端 | 安装指南 |
| `docs/FILE_UPLOAD.md` | 后端 | 文件上传特性 |
| `docs/PATH_EXAMPLES.md` | 后端 | 路径类型与用法 |
| `docs/summarization.md` | 后端 | 上下文压缩 |
| `docs/plan_mode_usage.md` | 后端 | 计划模式（TodoList） |
| `docs/STREAMING.md` | 后端 | 流式设计（Gateway 与嵌入式双路径、stream_mode 语义、去重不变量） |
| `docs/TUI.md` | 后端 | 终端工作台指南 |
| `docs/GUARDRAILS.md` | 后端 | 护栏机制 |
| `docs/MCP_SERVER.md` | 后端 | MCP 服务器接入 |
| `docs/RUN_EVENT_STREAM.md` | 后端 | 运行事件流 |
| `docs/REPLAY_E2E.md` | 后端 | 回放端到端 |
| `README.md` / `README_zh.md` | 仓库根 | 项目介绍与用法 |
| `Install.md` / `CONTRIBUTING.md` / `RELEASING.md` | 仓库根 | 安装 / 贡献 / 发版 |
| `CHANGELOG.md` / `CHANGELOG_zh.md` | 仓库根 | 变更记录 |
| `SECURITY.md` | 仓库根 | 安全策略 |

> 说明：上表中"位置"为 `backend` 的文档，其实际路径在 `backend/docs/` 下（例如 `backend/docs/ARCHITECTURE.md`）。根目录 `docs/` 只存放跨领域计划与设计笔记（`docs/plans/`、`docs/superpowers/` 等）。

---

## 附：一句话阅读顺序

> **先跑起来（阶段 0）→ 追一条消息走通全链路（阶段 1）→ 拆解后端装配（阶段 2：模型→代理→中间件→工具→状态→持久化）→ 深入进阶机制（阶段 3）→ 前端流式（阶段 4）→ 用测试与基准验证理解（阶段 5）**。遇到"这是怎么做到的"，`AGENTS.md` 系列文档 + 对应测试文件是最终权威。
