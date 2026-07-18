# 后端开发指南（BACKEND_Development）

## 分析快照

- 分支：`main`；HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean；子模块：无
- 分析范围：`openhands/app_server/`（核心）、`openhands/server/`（shim）、`openhands/db/`、`openhands/analytics/`、`enterprise/server/`
- 未覆盖范围：外部 `openhands-sdk/agent-server/tools` 内部实现

## 证据分类

- Evidence / Inference / Unknown，标注于关键结论。

## 核心结论

> [Evidence] 后端 = FastAPI V1 App Server（`openhands.app_server.app:app`），端口 3000。入口 `uvicorn openhands.server.listen:app`（`server/listen.py` 为 shim）。
> Agent 循环不在后端进程内执行；后端是**编排层**：启动沙箱容器、转发消息、接收 webhook 回灌事件。

---

## 1. 后端入口 / 初始化

| 项 | 内容 | Evidence |
| -- | -- | -- |
| ASGI app | `openhands/app_server/app.py:54`（`FastAPI(...)`） | `app.py:54-60` |
| uvicorn 目标 | `openhands.server.listen:app`（shim → `app_server.app:app`） | `server/listen.py:6`、`Makefile:262` |
| lifespan | `combine_lifespans(mcp_app.lifespan, app_lifespan)` | `app.py:36-51` |
| 启动迁移 | `OssAppLifespanService.run_alembic()` → `alembic upgrade head` | `oss_app_lifespan_service.py:23-38` |
| 路由挂载 | `app.include_router(v1_router.router)`（`/api/v1`）+ health（`/alive` 等）+ `/mcp` + (可选) SPA `/` | `app.py:71-79` |
| 中间件 | LocalhostCORS、CacheControl、RateLimit(10 req/s, InMemory) | `app.py:81-86` |

### 启动顺序

1. `init_tavily_proxy()`（在 app 创建前，挂 Tavily MCP 代理）；
2. 构建 `mcp_app = mcp_server.http_app(path='/mcp', stateless_http=True)`；
3. 解析 lifespan service（`get_app_lifespan_service()`，SaaS 时嗅探 `'saas'`）；
4. `FastAPI(routes=[Mount('/mcp', mcp_app)])`；
5. include routers；
6. add middlewares。

> [Evidence] `get_global_config()` 首次在某 router 模块 import 时（首个 `depends_*()` 调用）惰性构造，闭包捕获单例。

## 2. 路由注册（`v1_router.py:24-37`）

| 前缀 | router | 文件 |
| -- | -- | -- |
| `/api/v1/conversation/{id}/events` | event_router | `event/event_router.py:18` |
| `/api/v1/app-conversations` | app_conversation_router | `app_conversation/app_conversation_router.py:107` |
| `/api/v1/conversations/{id}/pending-messages` | pending_message_router | `pending_messages/pending_message_router.py:24` |
| `/api/v1/sandboxes` | sandbox_router | `sandbox/sandbox_router.py:32` |
| `/api/v1/sandbox-specs` | sandbox_spec_router | `sandbox/sandbox_spec_router.py:21` |
| `/api/v1/settings` | settings_router | `settings/settings_router.py:95` |
| `/api/v1/secrets` | secrets_router | `secrets/secrets_router.py:35` |
| `/api/v1/users` | user_router | `user/user_router.py:15` |
| `/api/v1/skills` | skills_router | `user/skills_router.py:29` |
| `/api/v1/webhooks` | webhook_router | `event_callback/webhook_router.py:77` |
| `/api/v1/web-client` | web_client_router | `web_client/web_client_router.py:6` |
| `/api/v1/git` | git_router | `git/git_router.py:41` |
| `/api/v1/config` | config_router | `config_api/config_router.py:20` |
| `/mcp` | FastMCP | `mcp/mcp_router.py:43` |

> [Evidence] 无 `@app.websocket`；实时事件经前端直连 agent-server WS。`python-socketio` 已弃用（`shared.py:20`）。

## 3. 配置加载

- `ServerConfig`（`server_config.py:9-56`）：类选择 + `app_mode` + PostHog/client_id/feature flags；`load_server_config()` env 填充；import 时即解析（`shared.py:14`）。
- `AppServerConfig`（`config.py:191-237`）：Pydantic；`config_from_env()` 用外部 `from_env(AppServerConfig,'OH')`；`get_global_config()` 单例。
- SaaS 覆盖：`OPENHANDS_CONFIG_CLS=...SaaSServerConfig` → `get_impl` 动态导入（`import_utils.py:43-78`）。

## 4. 日志初始化

- OSS：`app_server/utils/logger.py`。
- SaaS：`enterprise/server/logger.py`（JSON logger，`saas` logger）。

## 5. 服务注册 / DI（核心模式）

> [Evidence] DI 基元：`Injector[T]`（`services/injector.py:12`），`InjectorState = starlette.State`。每个请求获得一个 `InjectorState`，injector 在其上读写属性（`DB_SESSION_ATTR`、`USER_CONTEXT_ATTR`...），保证同请求内复用 `AsyncSession`/`UserContext`。
> FastAPI 依赖 `depends_*()`（`config.py:527+`）闭包捕获 `get_global_config()`，返回 injector。

## 6. 主要 handler 模式

```
入口（router 函数）
→ 参数（Pydantic 模型，如 StartAppConversationRequest）
→ 参数验证（Pydantic + dependencies=get_dependencies()）
→ 调用服务（app_conversation_service.*）
→ 数据访问（ORM 模型旁的 sql_*_service / store）
→ 事务边界（DbSessionInjector.inject：成功则 commit；keep_open 例外）
→ 错误类型（OpenHandsError 子类 → 由 exception_handler 转 JSON）
→ 调用方（前端 REST / agent-server webhook）
→ Evidence
```

### 典型 handler

- `start_app_conversation`：`app_conversation_router.py:364-411`；调 `app_conversation_service.start_app_conversation()`，返回首个状态，剩余 `asyncio.create_task(_consume_remaining)`；`set_db_session_keep_open(state,True)`（`:376`）。
- `send_message`：`app_conversation_router.py:441-591`；thin proxy 转发到 `{agent_server}/api/conversations/{id}/events`，附 `X-Session-API-Key`。
- `on_event`（webhook 入站）：`webhook_router.py:468-544`；持久化事件、更新统计、后台跑 callback processor（`:583-627`）。
- `validate_session_key`：`sandbox/session_auth.py:37-100`；按 SHA-256 查、拒非 RUNNING。

## 7. 数据访问层 / 数据库

- ORM：SQLAlchemy 2.0 声明式，`Base = DeclarativeBase`（`utils/sql_utils.py:10-19`）。
- 模型分散在各 service 旁：`StoredConversationMetadata`（`sql_app_conversation_info_service.py:91`）、`StoredAppConversationStartTask`、`StoredEventCallback`/`StoredEventCallbackResult`（`sql_event_callback_service.py:48,76`）、`StoredRemoteSandbox`（`remote_sandbox_service.py:81-100`）。
- 自定义类型：`UtcDateTime`、`StoredSecretStr`（JWE 透明加解密，`sql_utils.py:44-70`）、`create_json_type_decorator`。
- 会话管理：`DbSessionInjector`（`db_session_injector.py:29`），支持 asyncpg（异步）/ pg8000（同步，Alembic）/ SQLite（aiosqlite 回退）/ GCP Cloud SQL（`google.cloud.sql.connector`）。env：`DB_HOST/PORT/NAME/USER/PASS/SSL_MODE/POOL_SIZE/MAX_OVERFLOW/GCP_DB_INSTANCE`。
- 迁移：OSS 14 条（`app_lifespan/alembic/versions/001-014`），启动自动 upgrade；SaaS 137 条（`enterprise/migrations/versions/`）。

## 8. 数据模型 / Schema

- OSS 持久化表：`conversation_metadata`、`app_conversation_start_task`、`event_callback`、`event_callback_result`、`v1_remote_sandbox`（含 `session_api_key_hash`）+ 历史 migration 表（recaptcha log 等）。
- 用户设置：`Settings`（Pydantic，`settings_models.py`），OSS 存文件（`FileSettingsStore`），SaaS 存 DB（`SaasSettingsStore`）。

## 9. 事务 / 并发

- 事务边界：`DbSessionInjector.inject`（成功 commit，除非 `db_session_keep_open`）。
- 并发：async + asyncio.create_task；后台任务持有跨请求 DB session（keep_open）。
- SaaS 限流：Redis-backed `RateLimiter`（`enterprise/server/rate_limit.py`）。
- Alembic 并发：enterprise `env.py:142` 用 `pg_advisory_lock` 防竞态。

## 10. 缓存 / 队列 / 后台任务

- 队列：`pending_messages`（WS 断线兜底，DB 表）；SaaS `maintenance_task`（通用后台任务表 + `OrgBudgetMaintenanceProcessor`）。
- 后台脚本（SaaS CronJob）：`run_maintenance_tasks.py`、`run_budget_maintenance.py`、`sync/*.py`（resend_keycloak 等，独立进程，不被 server import）。
- 缓存：主要依赖前端 TanStack Query；后端无显式缓存层（除 in-memory rate limiter）。

## 11. 鉴权 / 权限

- OSS：`DefaultUserAuth`（无认证）；`SESSION_API_KEY` 共享密钥（`utils/dependencies.py:9`）；`get_dependencies()`（`:13-32`）为受保护路由门。
- SaaS：`SaasUserAuth`（cookie/API-key 双模式）；Keycloak；`Permission` + `require_permission`（`authorization.py:48`）；`EFFECTIVE_ORG_ID`（`X-Org-Id`）。
- 每 sandbox：`session_api_key`（`session_auth.py`）。
- JWT：`JwtService`（JWS HS256 + JWE dir/A256GCM + Fernet 回退 + 多 key 轮换，`jwt_service.py:30,42-52`）。

## 12. 错误处理

- `OpenHandsError` 层次（`errors.py`）；`AuthenticationError`→401（`app.py:63-68`）。
- SaaS：`NoCredentialsError`/`ExpiredError`→401（`saas_server.py:227-238`）。

## 13. 外部服务 / 文件系统访问

- 文件存储：Local/Memory/S3/GCS（`file_store/`）。
- 事件存储：FS/S3/GCS（`config.py:307-327` 按 `SHARED_EVENT_STORAGE_PROVIDER`/`FILE_STORE` 选）。
- git provider：`integrations/{github,gitlab,bitbucket,bitbucket_data_center,azure_devops,forgejo}/`。
- Stripe（SaaS）：`enterprise/integrations/stripe_service.py`。
- Keycloak（SaaS）：`enterprise/server/auth/keycloak_manager.py`。
- MCP：`mcp/mcp_router.py`（含 Tavily 代理）。

## 14. 安全边界

（见 SOURCE_ARCHITECTURE §10；此处从略，要点：OSS 无认证、SaaS RBAC、列加密、session_api_key 仅 RUNNING 有效、JWT 算法锁定。）

## 15. 扩展方式 / 测试方式 / 调试入口

- 扩展：`OPENHANDS_CONFIG_CLS` + `get_impl`；`include_router`；MCP 工具；技能。
- 测试：`poetry run pytest ./tests/unit`（OSS）/ `poetry run --project=enterprise pytest ./enterprise/tests/unit`（SaaS）；详见 `测试与CI.md`。
- 调试：`make start-backend`（`--reload`）；`make run`（前后端并行）；`make docker-dev`（dev 容器）；`make kind`（本地 K8s + mirrord）。

## 16. 已知限制 / 后端技术债务

- 三套配置系统；`server/*` shim；`python-socketio` 僵尸依赖。
- DB session keep_open 跨请求。
- 外部 SDK 版本强耦合。
- ORM 模型散落、无集中 repository 层。
- mypy SDK 版本漂移。

---

## 已确认事实

- 后端为 FastAPI 编排层；agent 循环外部化。
- DI = Injector[T] + InjectorState；事务边界在 DbSessionInjector。
- OSS 14 + SaaS 137 迁移。

## 合理推断

- 后端"薄"是有意为之：把重计算下放到每沙箱 agent-server，便于水平扩展与隔离。

## Unknown 与待验证事项

- 外部 agent-server 的 `/api/conversations`/WS 协议内部行为。
- reCAPTCHA router 是否在 SaaS 镜像挂载。

## 批判性评估

- Injector DI 模型清晰但隐式（基于 Starlette State 属性名约定）。
- keep_open 与 monkey-patch 路由覆盖是脆弱点。

## 建设性改善建议

- [Recommendation] keep_open 场景显式标注并加超时/连接池监控。优先级：中。
- [Recommendation] 路由覆盖改用 `include_router` 优先级或显式替换 API，避免 monkey-patch。优先级：低。

## 主要证据索引

- `openhands/app_server/app.py:30-86`
- `openhands/app_server/v1_router.py:24-37`
- `openhands/app_server/config.py:191-237,240-420,423-433,527+`
- `openhands/app_server/services/injector.py:12`、`db_session_injector.py:29-341`、`jwt_service.py:27-307`
- `openhands/app_server/utils/sql_utils.py:10-70`、`import_utils.py:43-78`、`dependencies.py:9-32`
- `openhands/app_server/app_conversation/app_conversation_router.py:107,364-591`、`live_status_app_conversation_service.py:264,539`
- `openhands/app_server/event_callback/webhook_router.py:77,348-627`
- `openhands/app_server/sandbox/docker_sandbox_service.py:86`、`session_auth.py:37-139`、`sandbox_spec_service.py:17,119-138`
- `openhands/app_server/app_lifespan/oss_app_lifespan_service.py:12-38`、`alembic/env.py`
- `enterprise/server/auth/authorization.py:48`、`enterprise/server/rate_limit.py`
