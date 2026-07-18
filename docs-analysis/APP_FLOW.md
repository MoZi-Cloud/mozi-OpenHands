# 应用流程（APP_FLOW）

## 分析快照

- 分支：`main`
- HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean
- 子模块状态：无
- 分析范围：`frontend/src/routes.ts`、`routes/root-layout.tsx`、`routes/conversation.tsx`、`contexts/conversation-websocket-context.tsx`、`api/*-service.api.ts`、`openhands/app_server/app_conversation/app_conversation_router.py`、`enterprise/server/routes/auth.py`
- 未覆盖范围：外部 Agent Canvas CLI 的安装流程（在外部仓库）

## 证据分类

- Evidence：源码路由/服务直接证明
- Inference：由前端路由与后端路由推导的用户旅程
- Unknown：SaaS 生产环境的实际 OAuth/Keycloak 部署细节

## 核心结论

> [Evidence] 存在**两种应用形态**，对应两套登录流程：
> - **OSS（`app_mode=OPENHANDS`）**：无登录。`DefaultUserAuth.get_user_id()` 返回 `None`；前端 `useIsAuthed` 在 OSS 模式直接短路返回 `true`。
> - **SaaS（`app_mode=SAAS`）**：Keycloak + cookie 会话登录，受 `root-layout.tsx` 的认证门保护。

> [Evidence] 核心用户旅程（启动并对话）跨越**三个进程边界**：浏览器（前端）→ V1 App Server（FastAPI）→ 每 sandbox 的 agent-server 容器（外部镜像），前端与 agent-server 之间通过原生 WebSocket 直连。

---

## 1. 启动 / 首次运行

### 1.1 本地源码启动

- 前置：Python 3.12+、Node 22.12+、Docker（用于 sandbox）。
- [Evidence] `make run`（`Makefile:290-294`）并行启动：
  - 后端：`uvicorn openhands.server.listen:app --host 127.0.0.1 --port 3000 --reload`（`Makefile:260-262`）；
  - 前端：Vite dev server（`Makefile:265-274`）。
- 后端启动时（lifespan）运行 Alembic 迁移（`oss_app_lifespan_service.py:23-38`，`run_alembic_on_startup`）。

### 1.2 Docker 启动

- [Evidence] `docker compose up`（`Makefile:297-308`，`docker-compose.yml`）：单服务 `openhands`，端口 3000，挂载 docker.sock；默认拉取 `ghcr.io/openhands/agent-server:<v>-python` 作为 sandbox 镜像。

### 1.3 配置初始化

- [Evidence] V1 app server **不读 `config.template.toml`**，全部从环境变量经 `from_env(AppServerConfig, 'OH')`（外部 `openhands.agent_server.env_parser`）构造（`config.py:240-420`）。
- [Evidence] `Makefile:311-343` 的 `setup-config` 仅用于遗留 CLI/headless，生成最小 `config.toml`（核心运行路径不使用）。
- [Evidence] 前端 bootstrap：`PostHogWrapper` 调 `OptionService.getConfig()`（`GET /api/v1/web-client/config`）获取 `app_mode`、PostHog key、已配置的 OAuth provider、feature flags、ACP provider（`web_client/default_web_client_config_injector.py:74-100`）。

---

## 2. 登录流程

### 2.1 OSS（无登录）

```
前置条件：app_mode=OPENHANDS
→ 前端 useIsAuthed 调 POST /api/authenticate
→ 后端 AuthService（OSS）直接返回成功
→ 前端短路 authenticated=true
→ 进入主界面
```

> [Evidence] `frontend/src/hooks/query/use-is-authed.ts:7-40`（OSS 短路）；`openhands/app_server/user_auth/default_user_auth.py:24`（无认证）。

### 2.2 SaaS（Keycloak 登录）

```
前置条件：app_mode=SAAS，未登录
→ root-layout 检测未认证 → 跳转 /login?returnTo=...
→ 用户选登录方式（GitHub OAuth 等）→ 跳 Keycloak
→ 回调 → 后端设 chunked cookie（keycloak_auth，JWS 签名）
→ useAuthCallback 持久化 login_method，清理 query，导航到 returnTo
→ SetAuthCookieMiddleware 校验 TOS（accepted_tos），未接受则 TosNotAcceptedError
→ 进入主界面
```

> [Evidence] `frontend/src/routes/login.tsx:26-129`；`frontend/src/hooks/use-auth-callback.ts:11-69`；`enterprise/server/middleware.py:135-160`（TOS 校验）；`enterprise/server/auth/cookie_chunking.py`；`enterprise/server/routes/auth.py:80`。

### 2.3 无登录模式（OSS 即无登录模式）

已在 2.1 说明。

---

## 3. 创建会话 / 进入主界面

### 3.1 主界面与路由

[Evidence] `frontend/src/routes.ts:8-53`，`layout("routes/root-layout.tsx")` 下：
- `/`（home）、`/conversations/:conversationId`（运行时视图）、`/settings`（含 17 个子页）、`/accept-tos`、`/launch`、`/oauth/device/verify`。

### 3.2 启动会话（核心流程）

```
前置条件：已认证、已配置 LLM（/settings/llm-settings）
→ 用户在 / 或 /launch 填写 prompt/repo/文件 → 提交
→ 前端 POST /api/v1/app-conversations（start_request）
→ 后端 start_app_conversation()（app_conversation_router.py:364）
   返回首个状态更新，剩余状态以 asyncio.create_task 异步产出
→ LiveStatusAppConversationService 状态机推进：
   WAITING_FOR_SANDBOX → DockerSandboxService 启动容器，轮询 /alive
   PREPARING_REPOSITORY（可选克隆）
   RUNNING_SETUP_SCRIPT（运行 .openhands/setup.sh）
   SETTING_UP_GIT_HOOKS
   SETTING_UP_SKILLS（加载全局/用户/org/repo 技能）
   STARTING_CONVERSATION → 构建 StartConversationRequest → POST 到 {agent_server}/api/conversations
   READY
→ 前端拿到 conversation_url + session_api_key
→ 前端打开原生 WebSocket：ws(s)://<host>/sockets/events/<conversationId>
→ agent-server 在沙箱内运行 Agent 循环
→ 事件经 WebSocket 推送到前端 + 经 webhook 回调到 app-server（POST /api/v1/webhooks/events/{id}）
```

> [Evidence] `app_conversation_router.py:364-411`（启动）、`:441-591`（发消息，thin proxy）；`live_status_app_conversation_service.py:264,539-544`（POST 到 agent-server）；`docker_sandbox_service.py:86`；`frontend/src/utils/websocket-url.ts:78`；`openhands/app_server/event_callback/webhook_router.py:468-544`。

### 3.3 发送后续消息

```
→ 用户在聊天框输入 → 前端 POST /api/v1/app-conversations/{id}/send-message
→ 后端为 thin proxy：转发 JSON 到 {agent_server}/api/conversations/{id}/events，附 X-Session-API-Key
→ 若 WebSocket 已断开，前端改走 REST 兜底：PendingMessageService.queueMessage（/api/v1/conversations/{id}/pending-messages）
```

> [Evidence] `app_conversation_router.py:467`（docstring 明确"thin proxy"）；`frontend/src/contexts/conversation-websocket-context.tsx:878-923`（断线兜底）。

---

## 4. 设置 / 配置

[Evidence] `/settings` 路由树（`routes.ts`）含：`llm-settings`、`agent-settings`、`condenser-settings`、`verification-settings`、`mcp-settings`、`skills-settings`、`git-settings`（集成）、`secrets-settings`、`api-keys`、`app-settings`、`user-settings`；SaaS 额外：`billing`、`org-members`、`org`、`usage-monitoring`、`admin-dashboard`、`budgets`、`org-defaults`。
- 后端：`GET/PUT /api/v1/settings`（`settings/settings_router.py:95`）、`/api/v1/secrets`、`/api/v1/git`、`/api/v1/skills`。

---

## 5. git 集成与建议任务

```
→ 用户在 /settings/git-settings 连接 GitHub/GitLab/...
→ 前端调 /api/v1/git/installations、/repos、/branches
→ 后端 app_server/integrations/* 服务客户端拉取
→ /api/v1/git/suggested-tasks 返回可执行任务
→ 用户从任务发起会话（同 §3.2）
```

> [Evidence] `git/git_router.py:41`、`app_server/integrations/{github,gitlab,bitbucket,azure_devops,forgejo}/`。

---

## 6. SaaS 组织 / 计费 / 邀请

```
→ 创建/加入组织（/api/organizations/*）
→ 邀请成员（email，/api/...invitations）
→ 配置预算（org-budgets）
→ 订阅/充值（/api/billing/*，Stripe）
→ 切换组织（前端 useSelectedOrganizationStore + X-Org-Id header）
```

> [Evidence] `enterprise/server/routes/orgs.py:83`、`billing.py:30`、`org_invitations.py`；`enterprise/server/auth/org_context.py`（`EFFECTIVE_ORG_ID`）。

---

## 7. 多会话 / 后台会话切换

```
→ 用户在 home 同时有多个运行中会话
→ 活跃会话：原生 WebSocket（/sockets/events/<id>）
→ 非活跃（后台）会话：Socket.IO 订阅（/socket.io，事件名 oh_event）→ 仅显示 toast（错误/完成）
```

> [Evidence] `frontend/src/context/conversation-subscriptions-provider.tsx:206-263`。

---

## 8. 共享会话

```
→ 用户共享会话 → enterprise/sharing 生成只读链接
→ 访客访问 /shared/conversations/:id（公开路由，无需登录）
→ 前端读取 /api/shared-conversations/* 与 /api/shared-events/*
```

> [Evidence] `frontend/src/routes/shared-conversation.tsx`；`enterprise/server/sharing/shared_conversation_router.py:17`。

---

## 9. 错误提示 / 异常恢复

- 前端：`useErrorMessageStore` 全局错误 banner；`query-client-config.ts` 全局 401 失效 `["user","authenticated"]`、429 重试。
- sandbox 恢复：`useSandboxRecovery`（`websocket-provider-wrapper.tsx`）。
- WebSocket 重连：`use-websocket.ts:15`（重连感知）。
- 后端：`OpenHandsError`/`AuthError`/`SandboxError`（`app_server/errors.py`）；`AuthenticationError` → 401（`app.py:63-68`）。

## 10. 退出 / 再次启动

- 退出：`useLogout` → `AuthService.logout(appMode)`：OSS 调 `/api/unset-provider-tokens`，SaaS 调 `/api/logout`（清 cookie）。
- 再次启动：lifespan 重跑 Alembic；持久化数据在 `~/.openhands`（OSS 文件存储）或 Postgres（SaaS）。

## 11. 升级 / migration / 数据恢复

- OSS：14 条 Alembic 迁移（`app_server/app_lifespan/alembic/versions/001-014`），启动自动 `upgrade head`。
- SaaS：137 条迁移（`enterprise/migrations/versions/001-137`），CI 用 `enterprise-check-migrations.yml`（postgres:16，upgrade→downgrade→upgrade 往返）验证。
- 持久化：OSS `FileSettingsStore`/`FileSecretsStore`（`~/.openhands`）；SaaS `SaasSettingsStore`/`SaasSecretsStore`（Postgres，列加密）。

---

## 用户流程 ↔ 源码调用链对应

| 用户步骤 | 前端 | 通信 | 后端入口 | 关键服务 |
| -- | -- | -- | -- | -- |
| 启动会话 | home/launch | POST `/api/v1/app-conversations` | `app_conversation_router.start_app_conversation` | `LiveStatusAppConversationService` → DockerSandboxService → agent-server |
| 接收事件 | conversation 页 | WS `/sockets/events/<id>` | （直连 agent-server） | `ConversationWebSocketProvider` |
| 发消息 | chat input | POST `/api/v1/app-conversations/{id}/send-message` | thin proxy | 转发 agent-server |
| 设置 LLM | settings | GET/PUT `/api/v1/settings` | `settings_router` | SettingsStore |
| 登录(OSS) | — | POST `/api/authenticate` | AuthService | DefaultUserAuth |
| 登录(SaaS) | /login | OAuth → cookie | `enterprise/routes/auth.py` | SaasUserAuth/Keycloak |

---

## 已确认事实

- OSS 无登录流程；SaaS 有完整 Keycloak 登录 + TOS。
- 核心会话流程跨浏览器/App Server/agent-server 三边界，前端与 agent-server 直连 WebSocket。
- 发消息为 thin proxy；断线有 REST 兜底队列。

## 合理推断

- 后台会话用 Socket.IO、活跃会话用原生 WS，是性能/复杂度权衡。

## Unknown 与待验证事项

- [Unknown] SaaS 生产 Keycloak 部署细节（本仓库仅消费）。
- [Unknown] `agent-canvas` CLI 的本地栈启动行为（外部仓库）。

## 批判性评估

- OSS"无登录即用"降低了自托管门槛，但也意味着任何能访问 3000 端口者即"root 用户"（仅 `SESSION_API_KEY` 可选共享密钥保护）。
- 前端与 agent-server 直连绕过 app-server，导致 app-server 对实时事件无完整可见性（仅靠 webhook 回灌）。

## 建设性改善建议

- [Recommendation] OSS 默认开启 `SESSION_API_KEY` 或在文档强提示暴露风险。
- [Recommendation] 统一活跃/后台会话的实时通道（减少双协议维护成本）。优先级：低。

## 主要证据索引

- `frontend/src/routes.ts:8-53`、`routes/root-layout.tsx:70-278`、`routes/conversation.tsx:113-137`
- `frontend/src/hooks/query/use-is-authed.ts:7-40`、`hooks/use-auth-callback.ts:11-69`
- `frontend/src/contexts/conversation-websocket-context.tsx:80-1002`
- `frontend/src/utils/websocket-url.ts:78-95`
- `frontend/src/context/conversation-subscriptions-provider.tsx:206-263`
- `openhands/app_server/app_conversation/app_conversation_router.py:364-591`
- `openhands/app_server/app_conversation/live_status_app_conversation_service.py:264,539`
- `openhands/app_server/event_callback/webhook_router.py:348-627`
- `openhands/app_server/user_auth/default_user_auth.py:24`
- `enterprise/server/middleware.py:135-160`、`enterprise/server/routes/auth.py:80`
- `Makefile:260-308`
