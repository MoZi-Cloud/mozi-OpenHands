# OpenHands 多 Agent 协作工作流可行性分析

> **分析对象**：`/home/prome/agent3/OpenHands`（OpenHands app-server 仓库）
> **依赖版本**：`openhands-sdk` / `openhands-agent-server` / `openhands-tools` == `1.36.0`
> **分析日期**：2026-07-18
> **勘察范围**：本仓库源码（已忽略 `node_modules`、`.venv`、`frontend/node_modules` 等依赖目录）

---

## 0. 需求复述

目标：在 OpenHands 上实现**多 Agent 协作工作流**，典型流程为：

```
Agent 1 (Claude Code / cc) 写代码
   → Agent 2 (Codex) 读取代码并给出复核意见
   → cc 读取复核意见后修改代码
   → Codex 二次复核
   → cc 二次修改
   → （循环 3 次）
   → Agent 3 (openclaw) 自动化测试
```

核心问题：

1. OpenHands **能否实现**该需求（含适当变通）？
2. 若不能，**二次开发工作量**有多大？

---

## 1. 关键前提：本仓库不是 Agent 内核

本仓库是 OpenHands 的 **编排 / UI 层（app-server，FastAPI）**。真正的 agent 执行内核位于外部 PyPI 包：

| 包 | 提供内容 |
|---|---|
| `openhands-sdk` | `Agent`、`EventStream`、`Tool`、`LLM`、`LocalWorkspace`、`ConversationSettings`、`ACPAgent`、sub-agent |
| `openhands-agent-server` | sandbox 容器内运行的 HTTP server（`ConversationInfo`、`SendMessageRequest`、`StartConversationRequest` 等） |
| `openhands-tools` | 内置工具集合 |

部署拓扑：

```
app-server (本仓库, FastAPI, 主进程)
    │  HTTP + Webhook
    ▼
agent-server (sandbox 容器内, ghcr.io/openhands/agent-server:1.36.0-python)
    │  ACP JSON-RPC over stdio
    ▼
claude-code / codex / gemini-cli 子进程
```

**含义**：本仓库**全栈实现了 ACP 接入与编排 API**，但不包含 agent 内部 step loop 的源码。`from openhands.sdk import ...` 全部解析到 site-packages，本仓库内 grep 不到这些类。这一点决定了"二次开发"的边界——改编排在本仓库，改协议/内核要去 `software-agent-sdk` 仓库。

证据：`openhands/__init__.py:4`（namespace package）、`pyproject.toml:61-63, 251-253`（外部依赖声明）。

---

## 2. 核心结论

**需求完全可实现，且绝大多数基础设施已具备。** 缺的不是"agent 能力"，而是一个**编排器（orchestrator）**去把这条流水线串起来。该编排器**本仓库没有内置**，但所有拼接原料都现成。

- 最轻路径（外部 Python 脚本 + 现有 REST API）：**约 1–2 人天，不动 OpenHands 一行代码**。

---

## 3. 四块拼图逐一对照

### ① 接入多个 Agent（cc / codex / openclaw）→ ✅ 已具备（ACP）

本仓库**全栈实现了 ACP（Agent-Client Protocol）**，三大第三方 agent 的 provider 注册表、凭据字段、启动命令均现成：

| Agent | ACP 启动命令 | 凭据字段 |
|---|---|---|
| Claude Code | `npx -y @agentclientprotocol/claude-agent-acp` | `CLAUDE_CODE_OAUTH_TOKEN` |
| Codex | `npx -y @openai/codex-acp` | `CODEX_AUTH_JSON`（`~/.codex/auth.json`） |
| Gemini CLI | `npx -y @google/gemini-cli --acp` | GCP 系列（`GOOGLE_APPLICATION_CREDENTIALS_JSON` 等） |

- ACP 会话启动入口：`openhands/app_server/app_conversation/live_status_app_conversation_service.py:2136`（`_build_acp_start_conversation_request`）
- Settings 里 `agent_kind`（`openhands` / `acp`）双 variant 切换：`openhands/app_server/settings/settings_models.py:795-906`
- ⚠️ **仅创建会话时可选 agent kind，运行中不能切换**（`webhook_router.py:384-425`）
- ⚠️ **openclaw 不在预置 provider 里**。接入方式需二选一：
  - ACP "custom preset" fallback；或
  - 作为 `agent_kind=openhands` 的自定义 agent（`config.template.toml` 的 `[agent.<name>] classpath`，见 `config.template.toml:141-144`）

### ② "cc 写代码 → codex 读同一份代码复核" → ✅ 已具备（共享 workspace）

最关键、也最易误判的点。本仓库没有"agent A 的内存输出直接喂给 agent B"的原语，但**文件系统级共享天然成立**：

- `SandboxGroupingStrategy`（`openhands/app_server/settings/settings_models.py:605-633`）提供 5 种分组策略：

  | 策略 | 行为 |
  |---|---|
  | `NO_GROUPING`（默认） | 每个 conversation 独占 sandbox |
  | `GROUP_BY_NEWEST` | 新会话复用最近 sandbox |
  | `LEAST_RECENTLY_USED` | 复用最久未用 sandbox |
  | `FEWEST_CONVERSATIONS` | 复用会话数最少的 sandbox |
  | `ADD_TO_ANY` | 加入任意可用 sandbox |

- 用非 `NO_GROUPING` 策略时，**多个 conversation 共享同一个 sandbox（同一个 Docker 容器 / 同一个 `PROJECTS_PATH` 仓库目录）**。

- 于是：conversation#1 跑 cc（写代码到仓库）→ conversation#2 跑 codex（读同一仓库根目录复核）→ **codex 直接看到 cc 的改动**，完全可行。
- 更稳的做法：用 **git 当传递介质**（cc commit → codex pull / 看 diff）。

### ③ "复核意见"喂给下一个 Agent → ✅ 原料齐备，缺约 30 行胶水代码

- agent 文本回复以 `MessageEvent` 落盘为 JSON：
  `{persistence_dir}/{user_id}/v1_conversations/{conversation_id.hex}/{event_id.hex}.json`
  （`openhands/app_server/event/event_service_base.py:32-84`、`conversation_paths.py:12-35`）
- 三条可读渠道：
  1. REST `GET /api/v1/conversation/{id}/events/search`
  2. REST `GET /api/v1/app-conversations/{id}/download`（zip 导出整段轨迹）
  3. 直接读文件系统（任何使用同版本 `openhands-sdk` 的 Python 进程都能 `Event.model_validate_json`）
- 注入下一个 agent 的标准入口：`AppConversationStartRequest.initial_message`（`openhands/app_server/app_conversation/app_conversation_models.py:231`），可塞任意 `list[TextContent | ImageContent]`。
- **会话血缘模型已就位**：`parent_conversation_id` / `sub_conversation_ids`（`app_conversation_models.py:129-130, 248`），cc→codex→cc 的"链"在数据模型里已可表达，webhook 更新中血缘保留（`webhook_router.py:424`）。

### ④ 循环 3 次 + 最后跑测试 → ✅ 纯编排逻辑

循环次数、第 N 轮、切换"写代码 / 复核 / 测试"角色——均属**编排器职责**，本仓库无需为此改代码。

---

## 4. 缺什么？——只有编排器（workflow engine）

三个独立勘察 agent 一致确认：**全仓库搜不到 `workflow` / `pipeline` / `orchestrator` 的运行时概念**（命中均为 CI workflow、prompt 文案、UI 路由）。具体缺失：

1. **没有内置"agent handoff / 串联"原语**：`parent_conversation_id` 是被动元数据，没有"会话 STOP → 自动起下一个会话并注入 summary"的自动化。
2. **`event_callback` 子系统只内置一个 processor**（`SetTitleCallbackProcessor`，给会话起标题）；它设计用途是"通知外部系统"，不是"触发本系统下一个 agent"。入口：`openhands/app_server/event_callback/webhook_router.py:348-545`。
3. **send-message 是异步的**：`AppSendMessageResponse`（`app_conversation_models.py:418-430`）只返回 `success` + `sandbox_status`，**不含回复正文**——编排器必须轮询 events 或订阅 webhook 拿结果。
4. 企业版的 `enterprise/server/services/automation_event_service.py` 只把 GitHub/GitLab webhook 转发到**外部** automation 服务，且是 trigger → 单会话 1:1，**不能定义 agent 间流水线**。

---

## 5. 现有 REST API（编排脚本可直接调用）

路由聚合点：`openhands/app_server/v1_router.py:24-37`（统一前缀 `/api/v1`）。

| 操作 | Method + Path | 位置 |
|---|---|---|
| 创建会话（异步任务） | `POST /api/v1/app-conversations` | `app_conversation_router.py:364` |
| 流式创建 | `POST /api/v1/app-conversations/stream-start` | `app_conversation_router.py:1021` |
| 查询创建任务状态 | `GET /api/v1/app-conversations/start-tasks/search` | `app_conversation_router.py:1036` |
| 发送 follow-up 消息 | `POST /api/v1/app-conversations/{id}/send-message` | `app_conversation_router.py:430-591` |
| 列出 / 搜索会话 | `GET /api/v1/app-conversations/search` | `app_conversation_router.py:223` |
| 列出事件（消息 / observation / state） | `GET /api/v1/conversation/{id}/events/search` | `event_router.py:29` |
| 读 workspace 文件 | `GET /api/v1/app-conversations/{id}/file?file_path=...` | `app_conversation_router.py:1118` |
| 下载整段轨迹 zip | `GET /api/v1/app-conversations/{id}/download` | `app_conversation_router.py:1590` |
| 切换 LLM profile | `POST /api/v1/app-conversations/{id}/switch_profile` | `app_conversation_router.py:620` |
| 切换 ACP 模型 | `POST /api/v1/app-conversations/{id}/switch_acp_model` | `app_conversation_router.py:773` |
| Webhook（agent→app 状态） | `POST /api/v1/webhooks/conversations` | `webhook_router.py:348` |
| Webhook（agent→app 事件） | `POST /api/v1/webhooks/events/{id}` | `webhook_router.py:468` |

---

## 6. 二次开发工作量评估（三条路径，由轻到重）

### 路径 A：外部 Python 编排脚本 + REST（推荐 ⭐）

**工作量：约 1–2 人天**

写一个独立 Python 脚本，用 `httpx` 调本仓库现有 REST API：

```
启动 cc 会话
  → 轮询 /events/search 直到 execution_status=STOP/IDLE
  → 抽取 MessageEvent（复核意见）+ 读 workspace 文件 / git diff
  → 拼成 initial_message + parent_conversation_id=<上一会话>
  → 启动 codex 会话（同一 shared sandbox）
  → ……循环 3 次
  → 启动 openclaw 测试会话
```

- **零改动 app-server**，纯客户端
- 原料全部现成；唯一要写的是"等会话结束 + 抽回复 + 拼消息"的胶水（约 30–80 行）
- 优点：多进程隔离、有持久化与会话血缘、可用 Docker sandbox
- 缺点：要先启动 app-server；轮询有延迟

### 路径 B：用 Hooks / Pending Messages 做半自动触发（中等）

**工作量：约 2–4 人天**

利用现成的 **Hooks 系统**（事件类型 `pre_tool_use` / `post_tool_use` / `session_start` / `session_end` / `stop`，可挂 shell command；加载于 `live_status_app_conversation_service.py:1868-1886`、`app_conversation_router.py:1535-1542`）+ **Pending Messages 队列**（`openhands/app_server/pending_messages/`），把"会话结束 → 触发下一个 agent"做成回调。比纯外部脚本更内嵌，但仍需自建一个编排状态机。

### 路径 C：在 app-server 内建 workflow 引擎（最重，产品化）

**工作量：约 2–3 人周**

给 app-server 加一层 `workflow` 概念：

- DAG / 有向图定义（节点 = agent + prompt 模板，边 = 数据传递）
- 基于 `event_callback` / webhook 的状态推进
- 持久化 + UI 可视化编辑器 + 团队共享

这才是"OpenHands 产品级多 agent 工作流"。

> ⚠️ 若要改 **ACP 协议本身**或 **sub-agent 调度内核**，需动外部 `software-agent-sdk` 仓库（本机未克隆，源在 https://github.com/OpenHands/software-agent-sdk），那是另一个量级的工作。

---

## 7. 已具备 / 缺失 总览

### ✅ 已具备（本仓库）

- 完整 REST API：创建会话、发消息、读 events、读文件、下载轨迹、切换 LLM/profile、切换 ACP 模型
- Webhook 反向通道：会话状态、事件批量推送
- Event 持久化三路：JSON 文件 + SQL 元数据 + workspace 文件，**跨 agent 可读**
- 会话血缘模型 `parent_conversation_id` / `sub_conversation_ids`
- ACP 多 agent 支持：claude-code / codex / gemini-cli，provider 凭据字段与冲突检测齐备
- 共享 workspace（5 种 sandbox 分组策略）
- Hooks 系统（`pre_tool_use` / `post_tool_use` / `session_start` / `session_end` / `stop`）
- Pending messages 队列（会话未就绪时消息先排队）
- Skills / Microagents（基于 `KeywordTrigger` / `TaskTrigger` 的动态上下文注入）

### ❌ 缺失（需自行实现）

- 内置 "agent handoff / 串联" 原语（`parent_conversation_id` 仅被动元数据）
- workflow / pipeline / orchestrator 引擎（全仓库无运行时实现）
- 跨 conversation 的自动 context 桥接（"拉 events + 拼字符串 + 塞进 `initial_message`"需自写）
- conversation 运行中切换 agent kind
- headless 模式范例脚本（仓库内无 `examples/`）
- openclaw 预置 provider（需自行适配）

---

## 8. 关键文件索引

| 文件 | 关键锚点 | 作用 |
|---|---|---|
| `openhands/app_server/v1_router.py` | `:24-37` | 全部 REST 路由聚合点 |
| `openhands/app_server/app_conversation/app_conversation_router.py` | `:364-591`, `:1118`, `:1590` | 创建会话 / 发消息 / 读文件 / 下载轨迹 |
| `openhands/app_server/app_conversation/app_conversation_models.py` | `:129`, `:231`, `:248`, `:418` | 血缘 / `initial_message` / 响应模型 |
| `openhands/app_server/app_conversation/live_status_app_conversation_service.py` | `:2136`, `:1829-1839`, `:1868-1886` | ACP 启动 / sub-agent / hooks |
| `openhands/app_server/event_callback/webhook_router.py` | `:348-545` | agent→app webhook、event 持久化 |
| `openhands/app_server/event/event_service_base.py` | `:32-84` | Event 文件路径契约（跨 agent 读取） |
| `openhands/app_server/settings/settings_models.py` | `:605-633`, `:795-906` | sandbox 分组策略 / agent_kind 切换 |
| `openhands/app_server/sandbox/sandbox_spec_service.py` | `:120-138` | agent-server 镜像解析 |
| `openhands/app_server/web_client/default_web_client_config_injector.py` | `:254-270` | ACP provider 注册表透传 |
| `frontend/src/constants/acp-provider-secrets.ts` | 全文 | claude-code/codex/gemini 凭据字段 |

---

## 9. 建议

1. **优先走路径 A**（1–2 人天，外部脚本 + REST）做 PoC，验证 cc↔codex 循环 + openclaw 测试整条链。
2. **先确认 openclaw 的接入方式**（它是否提供 ACP 接口？还是作为 openhands 自定义 agent classpath？），这决定测试那一环要不要额外半天适配。
3. PoC 跑通后，若需要产品化（UI 编排、可复用、团队共享），再投入路径 C（2–3 人周）。
