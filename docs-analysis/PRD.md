# 产品需求与实现状态（PRD）

## 分析快照

- 分支：`main`
- HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean
- 子模块状态：无
- 分析范围：`README.md`、`Development.md`、`AGENTS.md`、`config.template.toml`、源码（`openhands/app_server/*`、`enterprise/*`、`frontend/src/routes.ts`）、路由表
- 未覆盖范围：外部仓库（`OpenHands/agent-canvas`、`OpenHands/software-agent-sdk`、`OpenHands/automation`、`OpenHands/deploy`）的实际产品形态

## 证据分类

- Evidence：源码/路由/配置直接证明
- Inference：多处证据推导
- Unknown：仓库内无法确认

## 核心结论

> [Evidence] 本仓库的实际产品是 **OpenHands V1 App Server**：一个用 FastAPI 实现的、用于编排"每会话一个 Agent 沙箱容器"的自托管服务，配套 React 前端与可选 SaaS/企业层。
> 证据：`openhands/app_server/app.py:54`、`v1_router.py:24-37`、`sandbox/docker_sandbox_service.py:86`。

> [Evidence] 根 `README.md` 描述的产品是 **"OpenHands Agent Canvas"**（L5, L31），并明确（L49-54）该代码正在迁移：Agent/Agent Server → `OpenHands/software-agent-sdk`，Agent Canvas → `OpenHands/agent-canvas`。README 的快速开始（端口 8000、`@openhands/agent-canvas`、`ghcr.io/openhands/agent-canvas:1`）对应的是**另一个仓库**，与本仓库（端口 3000、`openhands-frontend`）不一致。

> [Evidence] 本仓库不执行 Agent 循环；Agent 运行时由外部包 `openhands-sdk`/`openhands-agent-server`/`openhands-tools`（@1.36.0）+ Docker 镜像 `ghcr.io/openhands/agent-server:1.36.0-python` 提供。

---

## 1. 项目定位（基于源码实际实现）

OpenHands 是一个**自托管的 AI 软件工程 Agent 平台**，核心能力：

1. 通过 Web UI 启动/管理多个 Agent 会话；
2. 每个会话在一个隔离的沙箱（Docker/进程/远程）中运行 Agent；
3. 支持任意 LLM（litellm）、MCP 工具、技能（skills）、git 集成；
4. 提供 OSS（单用户/本地）与 SaaS（多租户、计费、SSO、组织）两种部署形态。

## 2. 目标用户与场景

- [Inference] 目标用户：开发者与团队（自托管）、企业（SaaS）。
- 适用场景：代码生成、代码审查、依赖更新、PR 自动化、issue 拆解、报告生成（README L31、L41-42 描述）。

## 3. 输入 / 输出

- 输入：用户自然语言消息、所选仓库、LLM 配置、技能/MCP 配置、上传文件、git 集成事件。
- 输出：Agent 产生的事件流（消息、动作、观察、diff、终端输出、浏览器截图）、PR/MR、会话元数据。

---

## 4. 功能实现状态矩阵

| 功能 | 文档声明 | 源码状态 | 运行时入口 | 测试证据 | 最终判断 |
| -- | ---- | ---- | ----- | ---- | ---- |
| Web 会话管理（启动/列表/删除/发消息） | README L31 | `app_conversation_router.py`（CRUD + send-message + export） | `POST /api/v1/app-conversations` | `tests/unit/app_server/` | 已完整实现（OSS） |
| 沙箱编排（Docker） | README L39 | `DockerSandboxService` | `POST /api/v1/app-conversations` | `tests/unit/app_server/` | 已完整实现 |
| 多后端（local/remote/cloud） | README L8, L40 | `config.py:332-394` 三种 injector | env `RUNTIME` | 间接 | 已实现 |
| Agent 循环执行 | README L132 | **外部**（`openhands-agent-server` 容器） | `live_status_app_conversation_service.py:539`（POST 到 agent-server） | 无本仓库测试 | 实现在外部仓库 |
| 任意 LLM（Bring your own model） | README L43 | litellm + `config_api` 模型搜索 + `Settings.llm_*` | `GET /api/v1/config` | 有 | 已实现 |
| MCP 支持 | README / config.template `[mcp]` | `mcp/mcp_router.py`（服务端 `/mcp`）+ `settings` MCP 配置 | `/mcp`、`/api/v1/settings` | `tests/unit/mcp/` | 已实现 |
| 技能（skills） | skills/README | `skill_loader.py`、`/api/v1/skills` | conversation 启动 | 有 | 已实现 |
| git 集成（GitHub/GitLab/Bitbucket/Azure DevOps/Forgejo） | README L41 | `app_server/integrations/` + `/api/v1/git` | `/api/v1/git/*` | `tests/unit/integrations/` | 已实现（OSS 服务客户端） |
| Slack / Jira / Linear / Notion 集成 | README L41-42 | Slack/GitHub/Jira/Bitbucket 等在 `enterprise/integrations/`；Linear 仅 storage；**Notion 无源码** | enterprise webhook 路由 | `enterprise/tests/unit/` | 部分实现（Notion 缺失） |
| 计费（Stripe） | AGENTS.md | `enterprise/server/routes/billing.py` | `/api/billing/*` | `enterprise/tests/` | 已实现（仅 SaaS） |
| 组织 / RBAC / SSO（Keycloak） | AGENTS.md | `enterprise/server/auth/`、`routes/orgs.py`、137 迁移 | `/api/organizations/*` | `enterprise/tests/` | 已实现（仅 SaaS） |
| 用户登录/账户认证（OSS） | — | `DefaultUserAuth`（`get_user_id()` 返回 None，**无真实认证**） | 全部受保护路由 | 有 | OSS 无登录流程（单用户） |
| 用户登录/账户认证（SaaS） | README L8 | `SaasUserAuth` + Keycloak + cookie/JWS | `/api`、`/oauth/*` | `enterprise/tests/` | 已实现（SaaS） |
| ACP Agent（Claude Code/Codex/Gemini） | README L44 | `ACPAgentSettings`（`settings_models.py:587`） | settings 配置 | 弱 | 类型存在，实现在外部 SDK |
| 共享会话（public share） | — | `enterprise/server/sharing/` | `/api/shared-conversations/*` | `enterprise/tests/` | 已实现（SaaS） |
| 自动化（schedule/webhook） | README L41 | **外部仓库** `OpenHands/automation`（Automation Server） | 无本仓库入口 | 无 | 文档声称，实现在外部 |
| 评测/Benchmarks | Development.md L330 | **外部仓库** `OpenHands/benchmarks` | 无 | 无 | 外部 |
| Agent Canvas CLI（`agent-canvas` 命令） | README L77-86 | **外部仓库** `OpenHands/agent-canvas` | 无本仓库入口 | 无 | 外部 |
| UI 端口 8000 | README L128 | **不准确**：实际端口 **3000**（`Makefile:6`、`Dockerfile:105`、`docker-compose.yml:15`） | — | — | README 与源码冲突 |
| reCAPTCHA 日志 | config | `recaptcha_router.py` 存在但**未挂载**到 `v1_router` | 无 OSS 入口 | 无 | 仅 SaaS 使用（不可在本仓库验证） |

---

## 5. README 声明逐项核验（精简）

| 声明 | 来源 | 状态 | 证据 |
| -- | -- | ---- | -- |
| "自托管开发者控制中心" | README L5 | 部分在本仓库（V1 app server 在树；agent/agent-server 在外部） | `app_server/` |
| "运行 OpenHands/Claude Code/Codex/Gemini/任何 ACP agent" | README L8, L44 | ACP 类型在树（`ACPAgentSettings`），实现在外部 SDK | `settings_models.py:587` |
| "本地/远程/云后端" | README L8, L39 | 已实现 | `config.py:332-394` |
| "npm install -g @openhands/agent-canvas" | README L77 | 外部（不同仓库） | 本仓库前端名为 `openhands-frontend` |
| "docker pull ghcr.io/openhands/agent-canvas:1, -p 8000:8000" | README L101-106 | 外部镜像、外部端口 | 本仓库 `docker-compose.yml` 用 `openhands:latest`、端口 3000 |
| "git clone OpenHands/agent-canvas.git" | README L120 | 外部仓库 | 本仓库为 `OpenHands/OpenHands` |
| "UI at http://localhost:8000" | README L128 | **不准确**（实际 3000） | `Makefile:6` |
| "powered by OpenHands Agent Server (REST API)" | README L132 | V1 app server 在树；每沙箱 agent-server 为外部镜像 | `app.py:54`、`sandbox_spec_service.py:17` |
| "Automation Server 配套" | README L141 | 外部仓库 `OpenHands/automation` | 本仓库无 |
| "Bring your own model / any LLM" | README L43 | 已实现 | litellm |
| "集成 Slack/GitHub/Linear/Notion" | README L41-42 | Slack/GitHub 在 enterprise；Linear 仅 storage；Notion 无源码 | `enterprise/integrations/` |
| "集成 Slack/GitHub/Datadog 触发" | README L62-63 | enterprise 有 Slack/GitHub webhook；Datadog 仅镜像标签 | `enterprise/Dockerfile:5-7` |

---

## 6. 产品边界 / 非目标 / 限制

- **非目标（本仓库）**：不实现 Agent 推理循环、不实现 Automation Server、不实现 benchmarks、不发布 `agent-canvas` CLI/镜像。
- **当前限制**：
  - OSS 无多用户认证（`DefaultUserAuth` 返回 None）。
  - compose 不托管 db/redis，需外部提供或本地回退。
  - README 与实际产品/端口不一致。
  - 核心运行时强耦合外部 `software-agent-sdk` 仓库的版本（`sandbox_spec_service.py` 有版本一致性守卫）。

## 7. 数据与隐私边界

- [Evidence] OSS analytics 带 consent gate（`openhands/analytics/analytics_service.py`）。
- [Evidence] SaaS 加密敏感列：`StoredSecretStr`（JWE）、`enterprise/storage/encrypt_utils.py`（Fernet + JWT）。
- [Evidence] 每 sandbox `session_api_key` 仅在 sandbox RUNNING 时有效（`sandbox/session_auth.py:37-100`）。

---

## 已确认事实

- 本仓库 = V1 App Server + 前端 + UI 库 + SaaS 层。
- README 描述的 Agent Canvas 是迁移中的另一产品。
- Agent 运行时外部化。
- OSS 无登录流程；SaaS 有完整 Keycloak + RBAC + 计费。

## 合理推断

- Notion 集成、Automation Server、Agent Canvas CLI 均在本仓库之外，README 的相关描述对本仓库属"文档声称但源码未验证/外部"。

## Unknown 与待验证事项

- [Unknown] `agent-canvas --frontend-only/--backend-only` 的具体行为（CLI 在外部）。
- [Unknown] ACP agent（Claude Code/Codex/Gemini）在运行时是否实际可用（实现在外部 SDK）。
- [Unknown] Notion 集成是否存在于其他仓库。
- [Unknown] reCAPTCHA router 是否真的在 SaaS 镜像中被挂载。

## 批判性评估

- README 对本仓库的误导性强：端口、包名、入口、CLI 全部指向另一仓库。
- "集成 X/Y/Z" 列表混合了"在树"与"在外部仓库"，易被误读为全部已实现。
- ACP agent 类型存在但实现外部，属"类型定义不等于功能（在本仓库）实现"。

## 建设性改善建议

- [Recommendation] 在 README 顶部明确"本仓库当前为 V1 App Server"并提供准确的端口（3000）/包名/启动命令；把 Agent Canvas 的内容移至其专属仓库。
- [Recommendation] 功能矩阵中明确区分"在树实现 / 外部仓库 / 仅类型"。

## 主要证据索引

- `README.md:5,31,49-54,77-86,101-128,130-141`
- `openhands/app_server/v1_router.py:24-37`
- `openhands/app_server/app_conversation/app_conversation_router.py:107`
- `openhands/app_server/sandbox/docker_sandbox_service.py:86`
- `openhands/app_server/sandbox/sandbox_spec_service.py:17,119-138`
- `openhands/app_server/config.py:332-394`
- `openhands/app_server/settings/settings_models.py:587`
- `openhands/app_server/user_auth/default_user_auth.py:24`
- `enterprise/server/routes/billing.py`、`enterprise/server/auth/`、`enterprise/integrations/`
- `Makefile:6`、`docker-compose.yml:15`、`containers/app/Dockerfile:105`
