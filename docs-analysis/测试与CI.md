# 测试与 CI（测试与CI.md）

## 分析快照

- 分支：`main`；HEAD：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- 工作区状态：clean；子模块：无
- 分析范围：`tests/`、`enterprise/tests/`、`frontend/__tests__/`、`frontend/tests/`、`openhands-ui/`、`.github/workflows/`（26 个）、`pytest.ini`、`pyproject.toml`、`dev_config/`、`frontend/.husky`
- 未覆盖范围：未实际运行测试（只读约束）

## 证据分类

严格区分四层：①测试文件存在 ②测试命令已配置 ③CI 实际执行 ④被跳过/禁用/条件。

---

## 1. 测试类别总览

| 类别 | 文件数 | 配置位置 | 执行它的 CI | CI 实际执行？ | Evidence |
| -- | -- | -- | -- | -- | -- |
| 后端 pytest（OSS `openhands/`） | 109（`tests/unit/`） | `pytest.ini:1-4` | `py-tests.yml` job `test-on-linux` | ✅ | `py-tests.yml:60`（`poetry run pytest --forked -n auto -s ./tests/unit --cov=openhands --cov-branch`） |
| 后端 pytest（Enterprise） | 156（`enterprise/tests/unit/`） | `enterprise/pyproject.toml:93-95` | `py-tests.yml` job `test-enterprise` | ✅ | `py-tests.yml:92`（`poetry run --project=enterprise pytest --forked -n auto -s -p no:ddtrace ... ./enterprise/tests/unit --cov=enterprise`） |
| 前端 vitest（unit） | 283（`frontend/__tests__/`） | `vite.config.ts:107-115`、`package.json:56,58` | `fe-unit-tests.yml` job `fe-test` | ✅ | `fe-unit-tests.yml:46`（`npm run test:coverage`） |
| 前端 Playwright（E2E） | 2（`frontend/tests/`） | `playwright.config.ts`、`package.json:57` | `fe-e2e-tests.yml` job `fe-e2e-test` | ⚠️ 部分（仅 chromium；firefox/webkit 声明但 CI 不跑） | `fe-e2e-tests.yml:42`（`npx playwright test --project=chromium`） |
| openhands-ui（storybook/vitest） | 0 真实测试，19 stories | `openhands-ui/vitest.config.ts:15-37` | `ui-build.yml` | ❌ 不跑测试（仅 `bun run build`） | `ui-build.yml:30-34`；`package.json` 无 `test` 脚本 |

> [Evidence] 无独立 `integration/` 测试树；`tests/` 仅 `unit/`。Makefile `test:` 仅 `test-frontend`（`Makefile:252-253`），无 `make test-python`。

## 2. 后端 pytest 配置与标记

- OSS `pytest.ini`：`addopts=-p no:warnings`、`asyncio_mode=auto`、`asyncio_default_fixture_loop_scope=function`。
- 覆盖率：`pyproject.toml:346-349`（`concurrency=["geent"]`、`omit=["enterprise/tests/*","**/test_*"]`）。
- Enterprise：`enterprise/pyproject.toml:93-99`（asyncio auto；coverage `omit=["tests/*"]`）。
- 标记：大量 `@pytest.mark.asyncio`（auto）；无 CI `-m` 过滤；无自定义 markers 块。
- 跳过（稀有）：
  - `enterprise/tests/unit/test_org_store.py:643,1083,1158,1223`（4 处，PostgreSQL-only SQL，SQLite 不支持）。
  - `enterprise/tests/unit/storage/test_auth_token_store.py:207,213`（2 处，SQLite 不支持 `lock_timeout`）。
  - `tests/unit/` 内无 skip。
- 发现路径：CI 显式传路径（`./tests/unit`、`./enterprise/tests/unit`），故 `pytest.ini` 无 testpaths。`openhands/` 源码树内无 `test_*.py`。

## 3. 前端测试

- vitest：`vite.config.ts:107-116`（jsdom、setup `vitest.setup.ts`、exclude `tests`、coverage include `src/**/*`）。setup 文件 mock react-i18next/zustand/MSW/ResizeObserver。283 文件，按 api/components/hooks/routes/services/stores/utils/i18n/ui 组织。
- Playwright：`playwright.config.ts`（testDir `./tests`，3 浏览器 chromium/firefox/webkit；`webServer.command=npm run dev:mock:saas --port 3001`）。
  - `placeholder.spec.ts`：`test("placeholder",()=>{})`（空操作，注释："确保 CI 通过直到加真实 E2E"）。
  - `avatar-menu.spec.ts`：1 个真实测试（issue #11933）。
  - CI 仅 chromium（`fe-e2e-tests.yml:42`）。

## 4. openhands-ui 测试

- `vitest.config.ts` 声明 `storybook` 测试项目（`@storybook/addon-vitest`，chromium headless）。
- `setupFiles: [".storybook/vitest.setup.ts"]` **该文件不存在**（仅 preview.tsx/prompt.txt/main.ts）→ 即便本地也跑不起来。
- `package.json:80-86`：无 `test` 脚本。
- CI `ui-build.yml:30-34`：仅 `bun install --frozen-lockfile` + `bun run build`。
- 结论：openhands-ui 的测试**配置存在但从未执行**（且当前不可执行）。

## 5. CI/CD 全量 workflow（26 个）

| 文件 | 用途 | 触发 | 跑测试？ |
| -- | -- | -- | -- |
| `py-tests.yml` | Python 单测+覆盖率评论 | push main / PR | ✅ OSS + enterprise |
| `fe-unit-tests.yml` | 前端 vitest | push main / PR(frontend/**) | ✅ |
| `fe-e2e-tests.yml` | Playwright | push main / PR(frontend/**) | ⚠️ 仅 chromium |
| `lint.yml` | lint(FE+Python) | push main / PR | 否（仅 lint：eslint/tsc/pre-commit/mypy/ruff/翻译完整性） |
| `lint-fix.yml` | 自动修复 lint | PR 标 `lint-fix` 标签 | 否 |
| `ui-build.yml` | 构建 @openhands/ui | push main / PR(openhands-ui/**) | ❌ 仅 build |
| `npm-publish-ui.yml` | 发布 @openhands/ui 到 npm | push main 触及 openhands-ui/package.json | 否 |
| `ghcr-build.yml` | Docker 镜像（app+enterprise） | push main/`*-rel-*` / PR / dispatch | 否（构建+推送多架构） |
| `_build-image.yml` | 可复用构建 | workflow_call | 否 |
| `enterprise-check-migrations.yml` | Alembic 完整性 | PR 触及 migrations/alembic.ini | ⚠️ 非 unit test，但跑真实 DB 迁移（postgres:16，upgrade→downgrade→upgrade，pg8000 与 psycopg2 矩阵） |
| `check-package-versions.yml` | 禁止 `rev=` 依赖 | push/PR/dispatch | 否 |
| `check-version-consistency.yml` | pyproject↔前端包↔compose 镜像 tag 一致 | push/PR/dispatch | 否 |
| `release.yml` | release-please（gui/cloud 两线） | push main/release/** | 否 |
| `release-ready.yml` | 发布线门控 | pull_request_target(ready_for_review) | 否 |
| `pypi-release.yml` | 发布 openhands-ai 到 PyPI | dispatch / tag | 否 |
| `bump-chart.yml` | bump Helm chart | workflow_run(tag-image) | 否 |
| `tag-image.yml` | 重 tag 镜像 semver 别名 | push tag | 否 |
| `promote-to-development.yml` | 推 enterprise-server 到 dev | workflow_run / dispatch | 否 |
| `pr.yml` | PR 标题 lint（conventional） | pull_request_target | 否 |
| `pr-readiness-confirm.yml` | 强制 HUMAN 段+首贡献者确认 | PR / schedule | 否 |
| `pr-artifacts.yml` | 审批后删 .pr/ | PR / review / dispatch | 否 |
| `issue-opened.yml` | issue triage | issues(opened)/schedule/dispatch | 否 |
| `welcome-good-first-issue.yml` | 欢迎评论 | issues(labeled) | 否 |
| `remove-duplicate-candidate-label.yml` | 去标签 | issue_comment(created) | 否 |
| `qa-changes-by-openhands.yml` | AI QA | PR 标 `qa-this` | 否 |
| `stale.yml` | 关闭陈旧 issue/PR(50天) | schedule `30 1 * * *` | 否 |

> [Evidence] 无 schedule 触发的测试 workflow。无 workflow 依赖 `openhands-sdk` 测试（跨仓库，见 `.agents/skills/cross-repo-testing/SKILL.md`）。

## 6. Lint / 类型检查 / 格式化

| 工具 | 配置位置 | CI 调用 |
| -- | -- | -- |
| ruff（OSS） | `pyproject.toml:308-344` | `lint.yml:57`(pre-commit) |
| ruff（enterprise） | `enterprise/dev_config/python/ruff.toml` | `lint.yml:75` |
| mypy（OSS） | `dev_config/python/mypy.ini`，hook `dev_config/python/.pre-commit-config.yaml:46-67`（跑 `openhands/`） | `lint.yml:57` |
| mypy（enterprise） | `enterprise/dev_config/python/mypy.ini`，跑 `-p integrations -p server -p storage -p sync` | `lint.yml:75` |
| eslint（FE） | `frontend/.eslintrc`（airbnb-typescript+prettier+i18next） | `lint.yml:37`（`npm run lint`） |
| tsc / react-router typegen | `package.json:66-67` | `lint.yml:38`、`fe-unit-tests.yml:43` |
| **flake8** | `pyproject.toml` 声明 | **未调用**（僵尸配置） |
| **black / autopep8** | `[tool.*]` 块存在 | **未调用**（已被 ruff format 取代） |

## 7. 覆盖率

- 工具：`pytest-cov==7.1.0`（`pyproject.toml:129`）；前端 `@vitest/coverage-v8`。
- OSS：`--cov=openhands --cov-branch`；enterprise：`--cov=enterprise`；前端：`vitest --coverage`（lcov/html/json/text-summary）。
- 合并/发布：`py-tests.yml:102-126` job `coverage-comment`（仅 PR，`py-cov-action/python-coverage-comment-action@v3`，`MERGE_COVERAGE_FILES=true`）。
- **阈值：无**。无 `--cov-fail-under`，无 `fail_under`。覆盖率仅评论，不阻断合并。

## 8. pre-commit / husky / lint-staged

- 前端 husky：`frontend/.husky/pre-commit`（唯一 hook，无 pre-push/commit-msg）：先 `npx lint-staged`（frontend/）再根 `poetry run pre-commit run ...`。`prepare` 脚本 `cd .. && husky frontend/.husky`。
- lint-staged（`package.json:70-81`）：`eslint --fix`+`prettier --write`、`typecheck:staged`、`check-translation-completeness`。
- Python pre-commit：根 `dev_config/python/.pre-commit-config.yaml`（7 hook：trailing-ws/eof-fixer/check-yaml/debug-statements/自定义 `warn-appmode-oss`/pyproject-fmt/validate-pygraph/ruff+mypy）；enterprise 镜像同结构（`files:^enterprise/`）。
- mypy additional_dependencies：根 pin `openhands-sdk==1.36.0`；enterprise pin `openhands-sdk==1.33.0`（**版本漂移**）。

## 9. 发布

- 两条 release-please 线：`release-please-config.json`（gui，`exclude-paths:["enterprise"]`）与 `release-please-config.cloud.json`（cloud，前缀 `cloud-`）。
- manifest：`.release-please-manifest.json`（`".": "1.11.0"`）。
- 触发 `release.yml`（push main/release/**）→ `OpenHands/release-actions`。
- 后续：`tag-image.yml`（重 tag）、`pypi-release.yml`（`./build.sh`=`poetry build`）、`bump-chart.yml`（bump OpenHands-Cloud Helm）。
- Makefile 无 release 目标。

## 10. 测试覆盖矩阵（按模块）

| 模块/功能 | 单元 | 集成 | E2E | CI 执行 | 主要缺口 |
| -- | -- | -- | -- | -- | -- |
| OSS app_server（router/service/sandbox/webhook） | ✅ 64 文件 | ❌ | ❌ | ✅ | 无集成测试 |
| OSS storage/settings/file_store | ✅ 7 | ❌ | — | ✅ | — |
| OSS analytics | ✅ 3 | — | — | ✅ | — |
| OSS integrations | ✅ 17 | ❌ | — | ✅ | — |
| OSS mcp | ✅ 1 | — | — | ✅ | 单文件 |
| OSS db（ssl） | ❌（间接） | — | — | — | 无直接测试 |
| openhands/runtime、openhands/agent | ❌ | ❌ | — | — | 实现在外部 SDK，本仓库无测试 |
| enterprise server/storage/integrations/sync/routes | ✅ 156 | ❌ | — | ✅ | 无集成测试 |
| enterprise migrations | ❌（smoke） | ✅(DB) | — | ✅(migrations workflow) | 仅迁移往返 |
| frontend components/hooks/routes/services/stores/utils | ✅ 283 | — | ⚠️ | ✅(vitest) | E2E 仅 chromium |
| openhands-ui | ❌(stories) | — | — | ❌ | 测试未执行且不可执行 |

## 11. 显式缺口

### 存在但 CI 不执行
1. **openhands-ui 测试**：vitest storybook 项目声明，但无 `test` 脚本、CI 仅 build、`vitest.setup.ts` 缺失。
2. **Playwright firefox/webkit**：声明但 CI 仅 chromium。
3. **flake8/black/autopep8**：僵尸配置。

### 被跳过
4. enterprise `test_org_store.py` 4 处 skip（PG-only SQL）。
5. enterprise `test_auth_token_store.py` 2 处 skip（SQLite `lock_timeout`）。
6. `frontend/tests/placeholder.spec.ts` 空操作占位。

### 本仓库无测试
7. `openhands/runtime`、`openhands/agent`（外部 SDK）。
8. `openhands/db`（仅间接）。
9. 无后端集成测试（`tests/` 仅 unit）；唯一真实 DB 执行是 `enterprise-check-migrations.yml`。

### 其他观察
10. 覆盖率无阈值，不阻断合并。
11. mypy SDK 版本漂移（1.36.0 vs 1.33.0）。
12. Makefile 无 `test-python`。
13. `pyproject.toml` 双声明（PEP621 + `[tool.poetry]`）。
14. `pytest-rerunfailures` 仅 CI 装（`py-tests.yml:55-56`），不在 `pyproject.toml`；本地跑无 rerun。
15. enterprise CI 显式 `-p no:ddtrace ...` 禁用 ddtrace 插件。

---

## 已确认事实

- OSS/enterprise/前端 vitest 均在 CI 执行。
- openhands-ui 测试从不执行（且不可执行）。
- 仅 `enterprise-check-migrations.yml` 跑真实 DB。
- 无覆盖率阈值；无后端集成测试；无 runtime/agent 测试（外部）。
- flake8/black/autopep8/python-socketio 为僵尸配置/依赖。

## 合理推断

- 测试以单元为主，跨边界（app-server↔agent-server）行为无本仓库集成覆盖；E2E 近乎缺失。

## Unknown 与待验证事项

- [Unknown] CI 中 fork-PR 的密钥可用性（`ghcr-build.yml:27` 有 fork 守卫）。
- [Unknown] 跨仓库 SDK 测试的实际覆盖（在外部仓库）。

## 批判性评估

- openhands-ui 测试"配置存在但不可执行"是最明显的虚假测试信号。
- E2E 几乎缺失（2 spec，1 占位）。
- 覆盖率不阻断 → 覆盖率可能长期下滑而无告警。
- mypy 版本漂移使 enterprise 类型检查针对过期 SDK。

## 建设性改善建议

- [Recommendation] 修复 openhands-ui 测试（补 `vitest.setup.ts`、加 `test` 脚本、CI 跑测试）或移除虚假配置。优先级：高；难度：低。
- [Recommendation] Playwright CI 跑 firefox/webkit，或移除声明。优先级：中；难度：低。
- [Recommendation] 设覆盖率阈值（`--cov-fail-under`）或至少 PR 评论趋势门控。优先级：中。
- [Recommendation] 统一 mypy 的 SDK pin（消除 1.36 vs 1.33 漂移）。优先级：中；难度：低。
- [Recommendation] 清理 flake8/black/autopep8/python-socketio 僵尸配置。优先级：低。
- [Recommendation] 补 app-server↔agent-server 集成测试（用 mock agent-server）。优先级：中；难度：高。
- [Recommendation] Makefile 加 `test-python` 目标。优先级：低。

## 主要证据索引

- `.github/workflows/py-tests.yml:60,92,102-126`
- `.github/workflows/fe-unit-tests.yml:43,46`、`fe-e2e-tests.yml:42`、`ui-build.yml:30-34`、`enterprise-check-migrations.yml:82-139`、`lint.yml:37,57,75`
- `pytest.ini:1-4`、`pyproject.toml:129,308-349`、`enterprise/pyproject.toml:93-99`
- `frontend/vite.config.ts:107-116`、`vitest.setup.ts`、`playwright.config.ts:39-81`
- `frontend/tests/placeholder.spec.ts`、`frontend/.husky/pre-commit`、`frontend/package.json:56-81`
- `openhands-ui/vitest.config.ts:15-37`、`openhands-ui/package.json:80-86`
- `dev_config/python/.pre-commit-config.yaml:46-67`、`enterprise/dev_config/python/.pre-commit-config.yaml:38-62`
- `Makefile:252-253`
- `tests/unit/README.md`
