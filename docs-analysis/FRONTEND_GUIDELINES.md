# 前端开发指南（FRONTEND_GUIDELINES）

## 分析快照

- 分支：`main`；HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean；子模块：无
- 分析范围：`frontend/src/`、`frontend/package.json`、`frontend/vite.config.ts`、`frontend/playwright.config.ts`、`openhands-ui/`
- 未覆盖范围：`package-lock.json` 未逐行阅读

## 证据分类

- Evidence：源码/配置直接证明
- Inference：从重复代码模式推断
- Recommendation：建议（非当前事实）

> 本文严格区分：①源码中明确存在的规范；②可推断的约定；③仓库无证据的规范；④推荐规范。

## 核心结论

> [Evidence] 前端为 React 19 SPA（react-router 7，`ssr:false`），Vite 7 构建，Tailwind 4，状态 = Zustand（17 store）+ TanStack Query，实时 = 原生 WebSocket（主）+ Socket.IO（后台会话）。
> [Evidence] 前端**不依赖** `@openhands/ui`；存在三套互不统一的 Design Token。

---

## 1. 前端入口 / 启动

- [Evidence] SPA shell 由 react-router 7 在构建期生成（仓库无 `index.html`）。
- `entry.client.tsx:1-43`：`prepareApp()`（`VITE_MOCK_API==='true'` 时启动 MSW）→ `hydrateRoot`，包 `QueryClientProvider → PostHogWrapper → HydratedRouter`，import `./i18n` 副作用初始化。
- `root.tsx:1-46`：`Layout`（HTML shell）、`meta`、默认 `App` 渲染 `<Outlet/>`，全局 `useInvitation()`，`<Toaster/>` 与 `#modal-portal-exit`。
- Provider 包裹顺序：QueryClient → PostHog → HydratedRouter。

## 2. 路由

[Evidence] `frontend/src/routes.ts:8-53`（显式 `route(...)`，非文件式）：
- 公开：`/login`、`/onboarding`、`/information-request`、`/automations/*`、`/shared/conversations/:id`。
- 受 `layout("routes/root-layout.tsx")` 保护（含 Sidebar）：`/`、`/accept-tos`、`/launch`、`/settings`（17 子页）、`/conversations/:conversationId`、`/oauth/device/verify`。
- 会话内 tab（非顶层路由）：browser/vscode/changes/planner/task-list。

## 3. 布局 / 组件层级

- `routes/root-layout.tsx:70-278`：认证/格式门，触发 `useAutoLogin`/`useAuthCallback`/`usePostHogIdentify`/`useAutoSelectOrganization`/`useReoTracking`，SaaS 未登录跳 `/login?returnTo=`，`<Outlet/>` 包 `OnboardingGuard → EmailVerificationGuard`。
- `routes/conversation.tsx:113-137`：`WebSocketProviderWrapper → ConversationSubscriptionsProvider → EventHandler → 内容`。

## 4. 组件目录组织

```
components/
├── features/      ~30 领域文件夹（chat/conversation/settings/payment/org/...）
├── providers/     PostHogWrapper
├── shared/        跨特性原语（buttons/modals/toggle/icons/inputs/badges）
├── ui/            本地 UI kit（typography/dropdown/context-menu/...）
└── v1/chat/       V1 专用聊天渲染（messages/event-*/task-tracking/subagent）
ui/                本地原语，经 "#/ui/*" 别名引用（非 @openhands/ui）
```

## 5. 状态管理

### 5.1 Zustand store（`frontend/src/stores/`，每文件一个）

| store | 文件 | 持有 |
| -- | -- | -- |
| `useAgentStore` | `agent-store.ts:17` | `curAgentState` |
| `useBrowserStore` | `browser-store.ts:21` | 浏览器 URL + 截图 |
| `useBtwStore` | `btw-store.ts:41` | 每会话 BTW 问题 |
| `useCommandStore` | `command-store.ts:15` | 终端命令日志 |
| `useConversationStore` | `conversation-store.ts:126` | 右栏可见性/tab/附件/待发消息/模式(plan|code)/plan 内容；devtools + 每会话 localStorage |
| `useErrorMessageStore` | `error-message-store.ts:18` | 当前错误 banner |
| `useEventMessageStore` | `event-message-store.ts:15` | 已提交事件 id（防闪烁） |
| `useEventStore` | `use-event-store.ts:64` | 追加事件日志 + 去重 + 流式 delta 合并 + UI 投影 |
| `useHomeStore` | `home-store.ts:26` | 最近仓库 + provider；persist |
| `useInitialQueryStore` | `initial-query-store.ts:36` | `/launch` 的待发 prompt/文件/repo/replay |
| `useMetricsStore` | `metrics-store.ts:20` | LLM cost/budget/token |
| `useModelStore` | `model-store.ts:44` | 每会话模型列表 + 当前 profile |
| `useOptimisticUserMessageStore` | `optimistic-user-message-store.ts:20` | 乐观消息占位 |
| `useSecurityAnalyzerStore` | `security-analyzer-store.ts:39` | 安全风险日志 + 确认态 |
| `useSelectedOrganizationStore` | `selected-organization-store.ts:19` | 当前 org id |
| `useStatusStore` | `status-store.ts:16` | 状态消息 |
| `useV1ConversationStateStore` | `v1-conversation-state-store.ts:18` | V1 执行状态 |

### 5.2 Context

- `contexts/conversation-websocket-context.tsx`（主 WS provider，维持主 agent + planning 两条 WS）。
- `contexts/websocket-provider-wrapper.tsx`（v0/v1 选择 + sandbox 恢复）。
- `context/conversation-subscriptions-provider.tsx`（Socket.IO，后台会话）。

### 5.3 TanStack Query

- [Evidence] `query-client-config.ts:19-63`：全局 `QueryCache`/`MutationCache` `onError` 弹 toast（`meta.disableToast` 除外）；401 失效 `["user","authenticated"]`；429 重试 2 次（`utils/rate-limit-retry.ts`）。
- hooks：`hooks/query/*`（60+ useQuery）、`hooks/mutation/*`（70+ useMutation）；key 常量 `hooks/query/query-keys.ts`。

## 6. 数据获取 / API 封装

- [Evidence] `api/open-hands-axios.ts:3-5`：单一 `openHands` axios 实例，`baseURL` 由 `window.location` + `VITE_BACKEND_BASE_URL` 构造；**cookie 同源，无 Authorization 拦截器**；响应拦截器（`:44-60`）监听 403 EmailNotVerifiedError 强制刷新。
- 服务模块（`src/api/*-service/`）：auth、option、conversation（v0+v1）、event、billing、git（v0+v1）、integration、onboarding、organization、pending-message、sandbox、secrets、settings、skills、suggestions、user、email、analytics、config、invariant、shared-conversation、api-keys。
- [Evidence] `api/README.md:1-103`：约定——服务为对象/类字面量；**组件不得直接调用服务**，必须经 TanStack Query hook。
- V1 运行时调用（`event-service.api.ts`）用裸 axios，目标为 runtime host（`buildHttpBaseUrl(conversationUrl)`），附 `X-Session-API-Key`。

## 7. 实时通信（WebSocket）

- 主路径：原生 WS，`utils/websocket-url.ts:78-95`（`ws(s)://<host>[/prefix]/sockets/events/<conversationId>`）。
- `hooks/use-websocket.ts:15-193`：重连感知通用 hook。
- `contexts/conversation-websocket-context.tsx:80-1002`：解析 JSON、v1 类型守卫、派发副作用（写 event/metrics/terminal/browser store、失效 TanStack cache、记录模型切换、错误 toast）；WS 关闭时回退 REST `PendingMessageService.queueMessage`（`:878-923`）；`resend_all=true` 触发历史回放，`EventService.getEventCount` 判断回放完成。
- Socket.IO（后台会话）：`context/conversation-subscriptions-provider.tsx:206-263`，`io(baseUrl,{transports:["websocket"],path:...,query:{conversation_id,session_api_key,providers_set}})`，仅订阅 `oh_event`。
- 副作用 reducer：`services/actions.ts`、`services/observations.ts`、`services/chat-service.ts`、`services/agent-state-service.ts`。

## 8. 表单 / 校验 / 错误处理 / 加载 / 空状态

- [Inference] 表单多用受控组件 + downshift + HeroUI Input；校验多为内联（未发现统一 schema 校验库如 zod/react-hook-form 全局采用）。
- 加载/空状态：`components/shared/`（loader）与各 feature 内的空态；`useErrorMessageStore` 全局错误 banner；react-hot-toast。
- [Unknown] 是否有统一表单校验约定——未见集中证据。

## 9. 样式系统 / Design Token / 主题

- [Evidence] Tailwind 4 经 `@tailwindcss/vite`（`vite.config.ts:8,32`）；`tailwind.config.js:1-27`（`darkMode:"class"`，扩展 `modal.*`/`org.*`，`@tailwindcss/typography`）。
- `tailwind.css:1-10`：`@import "tailwindcss"`、`@plugin '../hero.ts'`、`@config`、`@plugin tailwind-scrollbar`；`:13-25` 定义遗留 `@theme` token。
- `index.css:1-69`：Google Fonts（Outfit、IBM Plex Mono）、`:4-17` 另一套 `:root` CSS 变量（`--bg-dark` 等）、markdown/xterm 样式。
- `hero.ts:1-18`：HeroUI 暗色主题，primary `#4465DB`。
- `cn()` = `twMerge(clsx(...))`（`utils/utils.ts:12-14`）。

> [Evidence] **统一 Design Token 系统不存在**。三套独立 token：
> 1. `frontend/src/tailwind.css:13-25`（9 个 `@theme` 颜色）；
> 2. `frontend/src/index.css:4-17`（10 个 `:root` CSS 变量，仅被手写 CSS 用）；
> 3. `openhands-ui/tokens.css:1-176`（完整 6 色族×13 阶 + 排版，但前端不消费）。
> 前端无 `@openhands/ui` 依赖、无 `tokens.css` import、无任何 `@openhands/ui/*` import。

- [Recommendation] 收敛为单一 Design Token 来源（让前端消费 `@openhands/ui/tokens.css` 或反向合并）。优先级：中；难度：中。

## 10. 响应式 / 移动端 / 可访问性 / 国际化 / 图标 / 静态资源

- 国际化：`i18n/index.ts:24-51`，15 语言；`translation.json` 为源；`scripts/make-i18n-translations.cjs` 反向生成 `public/locales/*`（gitignored）与 `declaration.ts`（`I18nKey` enum，提交）；ESLint `eslint-plugin-i18next` 强制显示串来自 `I18nKey`。
- 图标：`lucide-react` + `react-icons`。
- 资源：`assets/`（品牌 SVG + `notification.mp3`）。
- [Inference] 响应式/可访问性多为组件级处理，未见集中规范文档。
- [Unknown] 是否有统一 a11y 规范——无集中证据。

## 11. 前端测试 / 构建

- vitest（unit，283 文件，jsdom + MSW，`vite.config.ts:107-116`、`vitest.setup.ts`）。
- playwright（E2E，2 spec，CI 仅 chromium，`playwright.config.ts`）。
- 构建：`npm run build` = `make-i18n && react-router build`（`react-router.config.ts:31-35`，`buildEnd: unpackClientDirectory`）。
- lint：`npm run lint` = `typecheck && eslint && prettier --check`；husky + lint-staged（`package.json:70-81`）。

## 12. 桌面/移动/浏览器差异

- [Evidence] 无桌面（Electron）/移动原生代码；纯 Web SPA。
- WSL：`dev_wsl` 脚本用 `VITE_WATCH_USE_POLLING=true`（`package.json`）。

## 13. 代码组织约定

- [Evidence] 路径别名：`#/ui/*` → `frontend/src/ui/*`（`tsconfig.json:39-43`）。
- [Evidence] `api/README.md` 规定组件不直接调服务。
- [Inference] store/服务/hook 分层清晰（stores=本地状态、query/mutation=服务端状态、services=纯 reducer）。

## 14. 前端安全边界

- [Evidence] 认证依赖 http-only cookie（同源），客户端不存 JWT/token；仅 `LOGIN_METHOD`/`SELECTED_ORG`/表单标志入 localStorage（`utils/local-storage.ts:1-46`）。
- [Evidence] 运行时调用用每会话 `session_api_key`（`X-Session-API-Key` header + WS query 参数）。
- [Evidence] axios 拦截器处理 403 邮箱未验证。

## 15. UI 技术债务

- `frontend/README.md` 过期（提 Redux、`state/` 目录不存在）。
- 三套 Design Token 不统一。
- `openhands-ui` 未接入，存在重复实现（toggle/typography 等）。
- Mock WS 仅覆盖 Socket.IO，未覆盖 V1 原生 WS（`mocks/handlers.ws.ts`）。

---

## 已确认事实

- React 19 SPA；Zustand+TanStack；原生 WS 主路径。
- 三套 Token；前端不消费 `@openhands/ui`。
- cookie 认证；`session_api_key` 运行时鉴权。

## 合理推断

- store/hook/service 分层为有意约束。

## Unknown 与待验证事项

- [Unknown] 是否有统一表单校验/a11y/响应式规范（无集中证据）。

## 批判性评估

- README 过期 + 三套 Token + UI 库未接入，增加新成员上手成本。

## 建设性改善建议

- [Recommendation] 更新 `frontend/README.md`（移除 Redux 提法，反映 Zustand）。优先级：低。
- [Recommendation] 统一 Design Token。优先级：中。
- [Recommendation] 接入 `@openhands/ui` 或明确废弃它。优先级：中。
- [Recommendation] 补齐 V1 原生 WS 的 MSW mock。优先级：低。

## 主要证据索引

- `frontend/src/entry.client.tsx:1-43`、`root.tsx:1-46`、`routes.ts:8-53`
- `frontend/src/routes/root-layout.tsx:70-278`、`routes/conversation.tsx:113-137`
- `frontend/src/contexts/conversation-websocket-context.tsx:80-1002`
- `frontend/src/utils/websocket-url.ts:78-95`、`hooks/use-websocket.ts:15-193`
- `frontend/src/context/conversation-subscriptions-provider.tsx:206-263`
- `frontend/src/api/open-hands-axios.ts:3-60`、`api/README.md:1-103`
- `frontend/src/query-client-config.ts:19-63`
- `frontend/src/stores/*.ts`（17 个）
- `frontend/src/tailwind.css:1-25`、`index.css:1-69`、`hero.ts:1-18`
- `frontend/src/i18n/index.ts:24-51`、`scripts/make-i18n-translations.cjs`
- `frontend/vite.config.ts:107-116`、`playwright.config.ts`
- `frontend/tsconfig.json:39-43`、`frontend/README.md`
- `openhands-ui/tokens.css:1-176`、`openhands-ui/index.ts:1-19`
