# 技术栈与依赖分析（TECH_STACK）

## 分析快照

- 分支：`main`
- HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean（无未提交修改）
- 子模块状态：无子模块
- 分析范围：根 `pyproject.toml`、`poetry.lock`/`uv.lock`、`frontend/package.json`、`openhands-ui/package.json`、`enterprise/pyproject.toml`、`Makefile`、`containers/`、`config.template.toml`、源码 import
- 未覆盖范围：`poetry.lock`/`uv.lock` 未逐行阅读（大型锁文件，仅抽查关键版本）；外部 PyPI 包（`openhands-sdk` 等）的内部依赖未展开

## 证据分类

- Evidence：来自 `pyproject.toml`、`package.json`、源码 import、`Makefile`、`Dockerfile` 的直接证据
- Inference：由多处证据推导
- Unknown：仓库内无法确认（多与外部依赖有关）

## 核心结论

> [Evidence] 本仓库是一个**多语言 monorepo**，但**核心 Agent 运行时不在本仓库内**。
> 三个外部 PyPI 包（`openhands-sdk==1.36.0`、`openhands-agent-server==1.36.0`、`openhands-tools==1.36.0`，均来自外部仓库 `OpenHands/software-agent-sdk`）通过命名空间包机制与本仓库 `openhands/` 包合并。
> 证据：`openhands/__init__.py:1-9`（`pkgutil.extend_path`）；`pyproject.toml:61-63`（依赖声明）。

> [Evidence] 本仓库实际拥有的是 **V1 App Server**（`openhands/app_server/`）+ **React 前端**（`frontend/`）+ **共享 UI 库**（`openhands-ui/`）+ **SaaS/企业层**（`enterprise/`）。
> 证据：`openhands/app_server/app.py:54`、`frontend/package.json:1-3`、`enterprise/saas_server.py:76`。

> [Evidence] 根 `README.md` 描述的产品是 **"OpenHands Agent Canvas"**，其代码正向其他仓库迁移（README L49-54），与本仓库当前的 V1 实现并非同一产品形态。

---

## 1. 编程语言与用途

| 语言 | 用途 | Evidence |
| -- | -- | -- |
| Python 3.12–3.13 | 后端 V1 App Server、SaaS server、migration、后台任务、测试 | `pyproject.toml:13-17`（`requires-python = ">=3.12,<3.14"`）；容器 `python:3.13.7-slim-trixie`（`containers/app/Dockerfile`） |
| TypeScript / React 19 | 前端 SPA、共享 UI 库 | `frontend/package.json`（react 19.2.4）、`openhands-ui/package.json` |
| Shell / Makefile | 构建、启动、dev 容器入口 | `Makefile`、`containers/app/entrypoint.sh`、`containers/dev/dev.sh`、`build.sh` |
| YAML | CI/CD、Kubernetes 清单 | `.github/workflows/*.yml`、`kind/manifests/*.yaml` |
| TOML | 配置模板（遗留）、pyproject | `config.template.toml`、`pyproject.toml` |

> [Evidence] 同时存在 `poetry.lock`（1.05 MB）与 `uv.lock`（946 KB）两套锁文件，`pyproject.toml` 同时声明 PEP 621 `[project]` 与遗留 `[tool.poetry]` 两套依赖块。CI 实际使用 `poetry install`（`Makefile:150-186`、`.github/workflows/py-tests.yml`），故 `[tool.poetry.*]` 为权威来源。
> 证据：`pyproject.toml:7-110`（`[project]`）与 `pyproject.toml:142-259`（`[tool.poetry]`）。
> [Inference] 仓库正处于 poetry → uv 的迁移过渡期，两套元数据并存可能产生漂移。

---

## 2. 后端框架与关键依赖（Python）

来源：`pyproject.toml:21-96`（`[project.dependencies`）、`pyproject.toml:251-253`（外部 SDK 包）。

### 2.1 框架与服务

| 依赖 | 实际用途 | 初始化位置 | 调用位置 | 关键路径 | Evidence |
| -- | -- | -- | -- | -- | -- |
| `fastapi` / `starlette==1.3.1` | ASGI Web 框架，构建 V1 App Server | `openhands/app_server/app.py:54` | 全部 router | 是 | `app.py:10-13,54` |
| `uvicorn` | ASGI 服务器 | 容器 CMD / Makefile | `containers/app/Dockerfile:105`、`Makefile:262` | 是 | 同上 |
| `python-socketio==5.14.0` | **遗留**：`shared.py:20` 注明 "socketio is no longer used" | 未初始化 | 无服务端创建 | 否（仅声明） | `openhands/app_server/shared.py:20`；`pyproject.toml:87` |
| `fastmcp>=3.2,<4` | FastMCP 服务，挂在 `/mcp`，暴露 `create_pr`/`create_mr` 等工具 | `app.py:33`（`mcp_server.http_app(path='/mcp')`） | `mcp/mcp_router.py` | 是 | `app.py:33,59` |
| `sse-starlette==3.3.4` | SSE 流式响应 | event/sandbox 路由 | router 层 | 是 | `pyproject.toml:80` |

### 2.2 数据库与持久化

| 依赖 | 实际用途 | 初始化位置 | 是否进入运行时 | Evidence |
| -- | -- | -- | -- | -- |
| `sqlalchemy[asyncio]>=2.0.40` | ORM（2.0 声明式），`Base = DeclarativeBase` | `openhands/app_server/utils/sql_utils.py:10-19` | 是 | 同上 |
| `asyncpg>=0.30` | PostgreSQL 异步驱动（主路径） | `db_session_injector.py:193-213` | 是 | `services/db_session_injector.py` |
| `pg8000==1.31.5` | PostgreSQL 同步驱动（Alembic / Cloud SQL） | `db_session_injector.py:240-258`、`enterprise/migrations/env.py:44-81` | 是（migration 与 sync 路径） | 同上 |
| `aiosqlite`（间接） | SQLite 回退（本地无 DB_HOST 时） | `db_session_injector.py:202` | 是（本地回退） | 同上 |
| `redis==6.4.0` | SaaS 限流、缓存（OSS 未使用） | `enterprise/server/rate_limit.py`、`enterprise/storage/redis.py` | 仅 SaaS | `pyproject.toml:71` |
| `alembic`（迁移） | OSS 14 条迁移 + Enterprise 137 条迁移 | `app_lifespan/alembic/`、`enterprise/migrations/` | 启动时（OSS `run_alembic_on_startup`） | `oss_app_lifespan_service.py:23-38` |

> [Evidence] `docker-compose.yml` 仅定义单个 `openhands` 服务，**没有 db / redis / runtime 服务**。Postgres/Redis 由外部提供或本地用 SQLite/文件存储回退。
> 证据：`docker-compose.yml`。

### 2.3 LLM 与 Agent（核心，但实现外部）

| 依赖 | 实际用途 | 是否进入运行时 | Evidence |
| -- | -- | -- | -- |
| `litellm==1.84.1` | 多 LLM 路由（"Bring your own model"） | 是（被外部 SDK 使用，并在 `config_api` 暴露模型搜索；SaaS 通过 LiteLLM proxy） | `pyproject.toml:55,163`；`config_api/llm_model_service.py`；`enterprise/server/verified_models/` |
| `openai==2.33.0` / `anthropic[vertex]` / `google-genai` / `google-cloud-aiplatform` | LLM provider 客户端 | 是（经 litellm / SDK） | `pyproject.toml:34,46-49` |
| **`openhands-sdk==1.36.0`** | Agent 核心：`Agent`、`LLM`、`Event`、`AgentContext`、`Skill`、`Tool`、condenser、secret、hook 等 | 是（关键运行路径，但**源码不在本仓库**） | `pyproject.toml:62,251`；`openhands/__init__.py:1-9` |
| **`openhands-agent-server==1.36.0`** | 每 sandbox HTTP agent-server 容器内的服务；提供 `ConversationInfo`、`StartConversationRequest`、`SendMessageRequest`、`EventPage`、`env_parser.from_env` 等线类型 | 是（app-server 代理目标） | `pyproject.toml:61,252` |
| **`openhands-tools==1.36.0`** | 工具实现（browsing/editor/ipython/cmd/think/planning 等），`get_default_tools`、`get_planning_tools` | 是（conversation 构建期） | `pyproject.toml:63,253` |

> [Evidence] Agent 运行时以**预构建 Docker 镜像** `ghcr.io/openhands/agent-server:<version>-python` 形式提供，由外部 `software-agent-sdk` 仓库的 CI 构建。
> 证据：`openhands/app_server/sandbox/sandbox_spec_service.py:17`（`_DEFAULT_REPOSITORY = 'ghcr.io/openhands/agent-server'`）、`:119-138`（`get_agent_server_image()`）；`.agents/skills/update-sdk/references/docker-image-locations.md:9-13`。

### 2.4 沙箱与运行时编排

| 依赖 | 实际用途 | Evidence |
| -- | -- | -- |
| `docker==7.1.0` | 通过 Docker SDK 启动 sibling 容器（每会话一个 agent-server 容器） | `pyproject.toml:36`；`openhands/app_server/sandbox/docker_sandbox_service.py:86` |
| `kubernetes>=33.1` | K8s 客户端（remote sandbox provider 路径） | `pyproject.toml:51` |
| `pexpect==4.9.0` / `libtmux>=0.46.2` | 进程型沙箱的终端/进程控制 | `pyproject.toml:50,72` |

> [Evidence] 三种 sandbox 后端，由 `RUNTIME` env 选择：`docker`（默认，`DockerSandboxServiceInjector`）、`local|process`（`ProcessSandboxServiceInjector`，无 Docker，用于"用 OpenHands 跑 OpenHands"）、`remote`（`RemoteSandboxServiceInjector`，OpenHands Cloud）。
> 证据：`openhands/app_server/config.py:332-394`。

### 2.5 鉴权、加密、OAuth

| 依赖 | 实际用途 | Evidence |
| -- | -- | -- |
| `pyjwt==2.13.0` / `joserfc>=1.0.0` / `authlib>=1.6.12` | JWT（JWS/JWE）、OAuth | `services/jwt_service.py:30`；`pyproject.toml:44,77,80` |
| `bcrypt` / `passlib`（间接） | 密码哈希（SaaS） | `enterprise/server/auth/` |
| Keycloak（外部服务） | SaaS 身份提供者 | `enterprise/server/auth/keycloak_manager.py`；`enterprise/allhands-realm-github-provider.json.tmpl` |
| Stripe（`stripe`） | SaaS 计费 | `enterprise/server/routes/billing.py`；`enterprise/integrations/stripe_service.py` |

### 2.6 文件、文档、代码分析

`pypdf`、`python-docx`、`python-pptx`、`html2text`、`tree-sitter-language-pack`、`gitpython==3.1.50`、`whatthepatch==1.7`、`pygithub>=2.5`、`pylatexenc==2.10`、`playwright==1.58.0`、`browsergym-core==0.13.3`。多用于 agent 工具/浏览能力，实际实现在 `openhands-tools`。

### 2.7 可观测性

`opentelemetry-api==1.39.1`、`opentelemetry-exporter-otlp-otlp-grpc`、`lmnr>=0.7.20`、`posthog`（前端 `posthog-js`）、Datadog（`enterprise/Dockerfile:5-7` 打标签，`ddtrace` 在 `enterprise/pyproject.toml:39`）。

> [Evidence] OSS `openhands/analytics/` 用 PostHog，带"consent gate"（无同意不上报 PII）。
> 证据：`openhands/analytics/analytics_service.py`、`analytics_context.py`。

### 2.8 MCP（Model Context Protocol）

| 依赖 | 实际用途 | Evidence |
| -- | -- | -- |
| `mcp>=1.25` / `fastmcp` | MCP 服务端（`/mcp` 暴露 PR/MR 工具）与 MCP 客户端配置 | `openhands/app_server/mcp/mcp_router.py:19,33`；`settings/settings_models.py:429-583` |

---

## 3. 前端技术栈（`frontend/`）

来源：`frontend/package.json`。

| 类别 | 依赖 | Evidence |
| -- | -- | -- |
| 框架 | `react@19.2.4`、`react-dom@19.2.4`、`react-router@7.17.0`（SSR 关闭，`react-router.config.ts:31-35`） | `package.json:31-32,34` |
| 构建 | `vite@7.3.5`、`@tailwindcss/vite@4.1.18` | `package.json:33,6` |
| 状态 | `zustand@5.0.13`（17 个 store）、`@tanstack/react-query@5.90.20` | `package.json:35,5` |
| UI | `@heroui/react@2.8.8`、`tailwind-merge`、`clsx`、`class-variance-authority`、`lucide-react`、`react-icons` | `package.json:4,9,38,24,23` |
| 编辑器/终端 | `@monaco-editor/react@4.7.0`、`monaco-editor`、`@xterm/xterm@6`、`@xterm/addon-fit` | `package.json:7,25,8,6` |
| 实时通信 | 原生 WebSocket（主路径，`/sockets/events/<id>`）+ `socket.io-client@4.8.3`（仅后台会话订阅） | `frontend/src/utils/websocket-url.ts:78`；`frontend/src/context/conversation-subscriptions-provider.tsx:263` |
| HTTP | `axios@1.16.0` | `package.json:13` |
| Markdown | `react-markdown`、`remark-gfm`、`remark-breaks`、`react-syntax-highlighter` | `package.json` |
| i18n | `i18next@25.10.10`、`react-i18next`、`i18next-http-backend`、`i18next-browser-languagedetector`（15 种语言） | `frontend/src/i18n/index.ts:24-51` |
| 动画 | `framer-motion@12.38.0` | `package.json` |
| 测试 | `vitest`（unit，283 文件）、`playwright`（E2E，仅 chromium 在 CI 跑） | `vite.config.ts:107-116`、`playwright.config.ts`、`.github/workflows/fe-unit-tests.yml`、`fe-e2e-tests.yml:42` |
| Mock | MSW（`VITE_MOCK_API`） | `frontend/src/mocks/handlers.ts` |

> [Evidence] 前端**不依赖** `@openhands/ui`（`rg "@openhands/ui"` 在 frontend 下 0 命中；`package.json` 无该依赖）。前端有自己的 `src/ui/` 与重复的 token 定义。
> 证据：`frontend/src/tailwind.css:13-25`、`frontend/src/index.css:4-17`、`tsconfig.json:39-43`（`#/ui/*` 别名指向本地）。

---

## 4. 共享 UI 库（`openhands-ui/`）

| 项 | 值 | Evidence |
| -- | -- | -- |
| 包名 | `@openhands/ui`（`1.0.0-beta.9`），发布到 npm | `openhands-ui/package.json:1-95` |
| 构建 | Bun + Vite（lib 模式）+ Storybook 9 + vitest | `openhands-ui/vitest.config.ts`、`.storybook/` |
| 导出 | `Button/Checkbox/Chip/.../Typography`，`exports["./styles"]` → `tokens.css` | `openhands-ui/index.ts:1-19`、`package.json:21-27` |
| Design Token | `tokens.css`（6 色族 × 13 阶 + 排版） | `openhands-ui/tokens.css:1-176` |

> [Evidence] `openhands-ui` 与主前端**未打通**：主前端不 import 它。这是独立的"抽取并标准化组件库"尝试，尚未接入主应用。`openhands-ui` 的 CI（`ui-build.yml`）只 `bun run build`，**不跑测试**（无 `test` 脚本，`vitest.setup.ts` 缺失）。
> 证据：`.github/workflows/ui-build.yml:30-34`；`openhands-ui/package.json:80-86`。

---

## 5. 企业层技术栈（`enterprise/`）

- 独立 Poetry 项目（`enterprise/pyproject.toml`），以 path 依赖 `openhands-ai = { path = "../", develop = true }` 依赖 OSS 包（`enterprise/pyproject.toml:24`）。
- 许可证：**Polyform Free Trial**（与 OSS 的 MIT 不同）；`enterprise/README.md:2-3`。
- 关键能力：Keycloak SSO、Stripe 计费、组织/RBAC、8 个 git provider 集成（GitHub/GitLab/Bitbucket/Bitbucket DC/Azure DevOps/Jira/Jira DC/Slack）、RFC 8628 Device Flow、Redis 限流、维护任务队列、137 条 Alembic 迁移。
- 入口：`enterprise/saas_server.py`，CMD `uvicorn saas_server:app --host 0.0.0.0 --port 3000`（`enterprise/Dockerfile:56`）。

---

## 6. 构建、包管理、容器、部署

| 项 | 工具 | Evidence |
| -- | -- | -- |
| Python 包管理 | Poetry 2.3.4（CI 用）、UV（过渡） | `pyproject.toml:39`、`Makefile:150-186` |
| 前端包管理 | npm（`package-lock.json`） | `frontend/package.json` |
| UI 库包管理 | Bun（`bun.lock`） | `openhands-ui/package.json` |
| 容器 | Docker（多阶段：node 构建前端 → python 构建后端 → `openhands-app`） | `containers/app/Dockerfile` |
| Compose | 单服务 `openhands`，端口 3000，挂 docker.sock | `docker-compose.yml` |
| K8s（本地） | kind + mirrord（仅本地开发） | `kind/`、`Makefile:214-246` |
| K8s（生产） | 外部仓库 `OpenHands/deploy` 的 Helm chart | `.agents/skills/cross-repo-testing/SKILL.md:22` |
| CI/CD | GitHub Actions（26 个 workflow） | `.github/workflows/` |
| 发布 | release-please（两条线：`gui` / `cloud`）、PyPI 发布、GHCR 多架构镜像 | `release-please-config*.json`、`pypi-release.yml`、`ghcr-build.yml` |

> [Evidence] 镜像仅 COPY `./skills` 与 `./openhands`（`containers/app/Dockerfile:89`），不含 `openhands/runtime`、`evaluation`（本就不存在）。CMD 为 `uvicorn openhands.server.listen:app`（`openhands/server/listen.py` 是已废弃的 re-export shim，指向 `openhands.app_server.app:app`）。

---

## 7. 关键依赖的状态分类汇总

| 类别 | 说明 |
| -- | -- |
| 声明且源码实际 import 且运行时初始化 | fastapi、uvicorn、sqlalchemy、asyncpg/pg8000、litellm、docker、fastmcp/mcp、i18next、zustand、axios 等 |
| 声明且源码 import，但实现在外部包 | `openhands-sdk`/`openhands-agent-server`/`openhands-tools`（命名空间合并，运行时关键） |
| 仅配置/声明，OSS 运行时未启用 | `redis`（仅 SaaS）、`python-socketio`（已弃用，`shared.py:20`） |
| 仅在测试中使用 | `pytest`、`pytest-asyncio`、`pytest-xdist`、MSW、playwright、vitest |
| 可能已失效/僵尸配置 | `flake8`（`pyproject.toml` 声明，无任何 hook/CI 调用）、`black`/`autopep8`（`[tool.*]` 块存在但未被调用，已被 ruff format 取代） |

---

## 已确认事实

- 多语言 monorepo；核心 Agent 运行时为外部 PyPI 包 + Docker 镜像。
- 后端 FastAPI，端口 3000；前端 React 19 SPA。
- `openhands-ui` 独立、未接入主前端。
- poetry/uv 双轨；PEP 621 与 `[tool.poetry]` 双声明。
- compose 单服务、无 db/redis 托管服务。

## 合理推断

- 仓库处于"V0 → V1 + 外部 SDK 抽取"的活跃迁移期，遗留 shim（`openhands/server/*`）、双重锁文件、双重依赖声明、过期 README 均指向同一结论。
- `python-socketio`、`flake8`、`black`、`autopep8` 已成为无效/僵尸配置。

## Unknown 与待验证事项

- [Unknown] 外部包（`openhands-sdk` 等）的具体内部依赖与实现，无法在本仓库验证。
- [Unknown] 生产环境（app.all-hands.dev）实际启用了哪些 SaaS 路由（部分路由由 env 条件挂载）。
- [Unknown] `openhands-ui` 是否已被任意外部消费者实际使用。

## 批判性评估

- 三套配置系统并存（`ServerConfig` 类选择 / `AppServerConfig` Pydantic / 散落 `os.getenv`），认知负担大。
- 三个 Design Token 来源（`tailwind.css`、`index.css`、`openhands-ui/tokens.css`）彼此独立、前端未消费 UI 库，存在重复与漂移。
- README 与实际产品形态不符，对新贡献者误导性强。

## 建设性改善建议

- [Recommendation] 将 `openhands/server/*` shim 删除，把 `Makefile`/`Dockerfile` 的 uvicorn 目标改为 `openhands.app_server.app:app`。优先级：中；难度：低。
- [Recommendation] 收敛配置系统为单一 `AppServerConfig`，迁移散落 `os.getenv`。优先级：中；难度：中。
- [Recommendation] 统一 Design Token：让前端消费 `@openhands/ui` 的 `tokens.css`，或反向收敛。优先级：低；难度：中。
- [Recommendation] 清理僵尸配置（`flake8`/`black`/`autopep8`/`python-socketio`）。优先级：低；难度：低。
- [Recommendation] README 与实际仓库对齐（端口、包名、入口），或在显著位置标注本仓库为 V1 App Server。优先级：高；难度：低。

## 主要证据索引

- `openhands/__init__.py:1-9`
- `pyproject.toml:21-96,142-259,251-253`
- `openhands/app_server/app.py:54-86`
- `containers/app/Dockerfile:89,105`
- `docker-compose.yml`
- `frontend/package.json`
- `openhands-ui/package.json`、`openhands-ui/tokens.css:1-176`
- `enterprise/pyproject.toml:24`、`enterprise/Dockerfile:56`
- `.github/workflows/`（26 个文件）
- `Makefile:150-308`
